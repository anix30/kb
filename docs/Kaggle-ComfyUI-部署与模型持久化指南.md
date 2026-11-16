# Kaggle 免费 GPU 安装 ComfyUI 完整流程（图片 / 视频生成 + 模型持久化）

> 适用对象：在 Kaggle Notebook 免费 GPU 上跑 ComfyUI，生成图片或短视频，并让模型在会话重启 / 新建 Notebook 后不丢失。
> 所有代码块均可直接粘贴到 Kaggle 的 Code Cell 中按顺序运行。

---

## 0. 先搞懂 Kaggle 的存储规则（这是"重启后模型丢失"的根源）

Kaggle 每次会话（Session）跑在一个临时容器里，目录分三类：

| 路径 | 性质 | 会话重启 / 崩溃后 | 用途 |
|---|---|---|---|
| `/kaggle/working/` | 可写，最多约 20 GB，会自动归档 | **只有点了 Save Version（保存版本）才保留**；中途断连、崩溃、超时被回收时，这里也可能被清空 | 放代码、依赖安装产物、本次生成的图片/视频 |
| `/kaggle/input/<数据集名>/` | 挂载进来的 Dataset，**只读** | 永久保留，下次 Add Input 重新挂载即可 | **放大模型、LoRA、VAE 等需要长期保留的文件（推荐）** |
| 其他临时目录（如 `/kaggle/ComfyUI`、`/tmp`、`/root`） | 可写，scratch 空间 | **会话结束即全部丢失** | 只放可重装的程序本体、缓存 |

**核心结论：**
- ComfyUI 程序本体可以每次重装（几分钟），不用持久化。
- **大模型文件（几个 GB 到几十 GB）绝不能只放在 `/kaggle/working` 或临时目录**，必须放进 Kaggle Dataset，挂载到 `/kaggle/input/` 下长期保留。
- 本次生成的图片 / 视频默认在 `ComfyUI/output`，需要复制到 `/kaggle/working` 才能被保存和下载。

---

## 1. 前置准备

1. 注册并登录 Kaggle：https://www.kaggle.com
2. 准备一个反向代理工具账号（二选一）：
   - **Cloudflared（推荐，免注册、免 Token）**：直接下载二进制即可用，会给一个 `https://*.trycloudflare.com` 临时地址。
   - **ngrok（备选）**：到 https://dashboard.ngrok.com/get-started/your-authtoken 复制你的 Authtoken，在 Notebook 右侧 **Add-ons → Secrets** 里新增 Secret，Name 填 `NGROK_TOKEN`，Value 填 Token。
3. 网络：Notebook 右侧面板 **Internet 必须设为 On**（否则无法 clone 仓库、下载模型）。

### 1.1 Kaggle 免费 GPU 规格与限制（先有预期）

- 可选加速器：**T4 ×2**（每张 16 GB）、**单 T4**、**P100**（16 GB）。在右侧 **Notebook options → Accelerator** 选择。
- 单会话有时长上限（GPU 会话通常约 9 小时），且受每周免费 GPU 配额约束；会话断开后容器回收。
- **显存预期：**
  - 图片：SD1.5 随便跑；SDXL 建议用量化版（fp8 / Lightning 加速 LoRA），16 GB 可跑。
  - 视频：T4/P100 16 GB 只能跑**低分辨率、短时长、fp8 量化**的小视频模型（如 Wan2.1/2.2 量化版、HunyuanVideo 量化版）；14B 全量模型基本装不下，长视频 / 高清视频不要指望免费卡。

---

## 2. 新建 Notebook 并做基础设置

1. Kaggle 首页 → **Create → New Notebook**。
2. 右侧面板逐项设置：
   - **Accelerator（加速器）**：选 `GPU T4 x2`（出图更快；跑不动可换 `GPU P100`）。
   - **Language**：`Python`。
   - **Persistence（持久化）**：选 `Files only`（或 `Variables and Files`）。作用是把 `/kaggle/working` 的内容在保存版本时归档。
   - **Internet**：`On`。
3. 第一个 Cell 先检查环境：

```python
import os, subprocess, torch
print("Python 工作目录:", os.getcwd())
print("CUDA 可用:", torch.cuda.is_available())
print("GPU 数量:", torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(f"  GPU {i}: {torch.cuda.get_device_name(i)}, 显存 {torch.cuda.get_device_properties(i).total_memory/1e9:.1f} GB")
!nvidia-smi
```

---

## 3. 安装 ComfyUI 程序本体（每次新会话重装，无需持久化）

> 程序本体放在临时目录即可，重装成本低；模型走第 4 节的 Dataset 持久化方案。

### 3.1 克隆仓库并安装依赖

```python
import os
os.chdir("/kaggle/working")

# 克隆 ComfyUI 官方仓库
!git clone https://github.com/comfyanonymous/ComfyUI.git

%cd /kaggle/working/ComfyUI
!pip install -r requirements.txt
```

### 3.2（可选）安装常用自定义节点（ComfyUI-Manager）

```python
import os
os.makedirs("/kaggle/working/ComfyUI/custom_nodes", exist_ok=True)
%cd /kaggle/working/ComfyUI/custom_nodes
!git clone https://github.com/ltdrdata/ComfyUI-Manager.git
!pip install -r ComfyUI-Manager/requirements.txt
```

> 视频工作流常用的节点（如视频加载/合并、Wan/Hunyuan 相关自定义节点）可在 ComfyUI-Manager 界面内按需安装；装完重启服务生效。

---

## 4. 模型持久化方案（重点：让模型重启不丢）

### 4.1 推荐架构：模型放 Kaggle Dataset，用配置文件映射进 ComfyUI

目录规划（Dataset 内部结构，自己上传或复用别人公开的同名结构即可）：

```
/kaggle/input/我的模型库/                 # 挂载后的 Dataset 路径（只读）
├── checkpoints/      # 大底模（SD1.5 / SDXL / Wan 等主模型）
├── loras/            # LoRA
├── vae/              # VAE
├── unet/             # 扩散模型（新版 ComfyUI 也叫 diffusion_models）
├── clip/             # CLIP / text encoder（含 umt5 等视频模型编码器）
├── clip_vision/
├── controlnet/
└── upscale_models/
```

ComfyUI 支持用 `extra_model_paths.yaml` 额外扫描外部模型目录，**不用把模型复制进程序目录，也不用软链接**，重启后只要重新挂载 Dataset 就能识别。

### 4.2 创建自己的模型 Dataset（首次，二选一）

**方式 A：直接复用 Kaggle 上别人公开的模型 Dataset（最省事）**
- Notebook 右侧 **Input → Add Input → Datasets**，搜索你要的模型（如 `SDXL`、`Wan2.1`、`HunyuanVideo`），挂载后在 `/kaggle/input/<数据集名>/` 下找到对应文件即可。

**方式 B：把自己下载的模型上传成私有 Dataset**
1. Kaggle 顶部 **Datasets → New Dataset**。
2. 按上面 4.1 的子目录结构上传模型文件（checkpoints / loras / vae …），命名例如 `my-comfyui-models`，设为 Private。
3. 回到 Notebook，**Add Input** 挂载这个 Dataset。

### 4.3 在 Notebook 里生成 `extra_model_paths.yaml` 指向 Dataset

把下面代码里的 `你的数据集名` 改成你在 4.2 挂载的 Dataset 目录名（先用 `!ls /kaggle/input/` 确认实际目录名）。

```python
import os, yaml

# 先确认挂载进来的 Dataset 目录名
!ls /kaggle/input/
```

```python
dataset_name = "你的数据集名"   # 例如 my-comfyui-models
base = f"/kaggle/input/{dataset_name}"

model_dirs = ["checkpoints", "loras", "vae", "unet", "diffusion_models",
              "clip", "clip_vision", "controlnet", "upscale_models"]

config = {}
for d in model_dirs:
    p = os.path.join(base, d)
    if os.path.isdir(p):
        config[d] = p

yaml.safe_dump(config, open("/kaggle/working/ComfyUI/extra_model_paths.yaml", "w"),
               allow_unicode=True, default_flow_style=False)
print(open("/kaggle/working/ComfyUI/extra_model_paths.yaml").read())
```

> 说明：`/kaggle/input` 是只读挂载，所以这里只做"路径映射"，不在 Dataset 目录里写文件。ComfyUI 启动时会同时扫描自带 `models/` 目录和 yaml 里列出的外部目录。

### 4.4 首次需要下载新模型时的临时落地办法

如果某个模型还没进 Dataset，先临时下载到 `/kaggle/working` 用完，**结束前务必转存到 Dataset**，否则下次丢失。

**从 HuggingFace 下载（推荐，用 hf 命令，支持断点续传）：**

```python
# !pip install -U "huggingface_hub[cli]"
!huggingface-cli download 仓库ID/文件名 \
  --local-dir /kaggle/working/tmp_models/checkpoints
# 示例：
# !huggingface-cli download stabilityai/stable-diffusion-xl-base-1.0 sd_xl_base_1.0.safetensors \
#   --local-dir /kaggle/working/tmp_models/checkpoints
```

**从直链 wget 下载（Civitai 公开直链等）：**

```python
!mkdir -p /kaggle/working/tmp_models/checkpoints
!wget -O "/kaggle/working/tmp_models/checkpoints/模型名.safetensors" "https://模型直链地址"
```

**把临时目录里的模型转存为 Dataset（二选一）：**
- 网页方式：Notebook 右上角 **Save Version → Save & Run All**，输出归档后在该 Notebook 的 Output 里找到文件，再 **New Dataset** 收录；或直接在 Dataset 页面上传本地下载好的模型。
- API 方式（已配 Kaggle API Token）：

```bash
# 先在 Kaggle 账号页新建 API Token 并上传到 Notebook Secret，再执行：
# kaggle datasets create -p /kaggle/working/tmp_models --dir-mode zip
```

### 4.5 生成的图片 / 视频如何保存

ComfyUI 默认输出在 `/kaggle/working/ComfyUI/output/`，已经在 working 目录内：
- 会话中可直接在右侧 **Output** 面板预览、下载单张 / 单个视频。
- 要跨会话保留：**Save Version** 时会一并归档；长期留存的成品建议同样下载到本地，或转存进自己的 Dataset。
- 若你把输出目录改到了别处，记得在结束前 `cp` 回 `/kaggle/working`。

---

## 5. 启动 ComfyUI 并建立公网访问

Kaggle 容器**不直接对外暴露端口**，需要一条反向代理隧道把本地 `8188` 端口暴露成公网网址。

### 5.1 先启动 ComfyUI 后台进程

```python
import os, subprocess, threading, time, glob

os.chdir("/kaggle/working/ComfyUI")

# --listen 0.0.0.0 允许隧道访问；--force-fp16 降低显存占用
cmd = [
    "python", "main.py",
    "--listen", "0.0.0.0",
    "--port", "8188",
    "--force-fp16",
]

proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                         universal_newlines=True, bufsize=1)

def log_reader(stream):
    for line in stream:
        print(line.rstrip())

threading.Thread(target=log_reader, args=(proc.stdout,), daemon=True).start()

# 等待服务就绪（最长约 3 分钟）
import urllib.request
ready = False
for _ in range(90):
    time.sleep(2)
    try:
        urllib.request.urlopen("http://127.0.0.1:8188/system_stats", timeout=3)
        ready = True
        break
    except Exception:
        pass
print("ComfyUI 已就绪:" if ready else "ComfyUI 尚未就绪，请看上方日志")
```

### 5.2 方式一：Cloudflared 隧道（免 Token，推荐）

```python
import os, subprocess, time

if not os.path.exists("/kaggle/working/cloudflared"):
    !wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O /kaggle/working/cloudflared
    !chmod +x /kaggle/working/cloudflared

# 后台启动隧道，日志写到文件
tunnel = subprocess.Popen(
    ["/kaggle/working/cloudflared", "tunnel", "--url", "http://localhost:8188", "--no-autoupdate"],
    stdout=subprocess.PIPE, stderr=subprocess.STDOUT, universal_newlines=True, bufsize=1
)

# 从日志里解析公网地址
public_url = None
for _ in range(40):
    time.sleep(2)
    out = tunnel.stdout.readline()
    if out and "trycloudflare.com" in out:
        import re
        m = re.search(r"https://[\w-]+\.trycloudflare\.com", out)
        if m:
            public_url = m.group(0)
            break

print("公网访问地址:", public_url)
print("在浏览器打开该地址即可进入 ComfyUI 界面")
```

### 5.3 方式二：ngrok 隧道（备选，需 Secret）

```python
from kaggle_secrets import UserSecretsClient
from pyngrok import conf, ngrok

user_secrets = UserSecretsClient()
NGROK_TOKEN = user_secrets.get_secret("NGROK_TOKEN")   # 对应右侧 Secret 名
conf.get_default().auth_token = NGROK_TOKEN
conf.get_default().monitor_thread = False

tunnels = ngrok.get_tunnels(conf.get_default())
if not tunnels:
    url = ngrok.connect(8188).public_url
else:
    url = tunnels[0].public_url
print("公网访问地址:", url)
```

> 隧道地址每次会话都会变；关闭 Notebook 或会话结束后隧道自动失效。

---

## 6. 在界面里生成图片 / 视频

1. 浏览器打开第 5 节给出的公网地址，进入 ComfyUI 界面。
2. **加载工作流：**
   - 图片：菜单 **Workflow → Browse Templates** 选 SD1.5 / SDXL 文生图模板；或拖入自己的 JSON 工作流。
   - 视频：在 Manager 里安装对应视频模型的自定义节点，加载 Wan / Hunyuan 官方示例工作流（ComfyUI 官网模板或 GitHub 示例 JSON）。
3. **选模型：** 在 Checkpoint Loader / UNET Loader 节点的下拉框里选 Dataset 中挂载进来的模型；若看不到，按 `R` 键刷新，或重启服务后再看。
4. **出图 / 出视频：** 填提示词、采样步数、分辨率、批量，点击 **Queue Prompt**。
5. **取成品：** 结果显示在界面右侧；同时文件落在 `/kaggle/working/ComfyUI/output/`，可在 Output 面板下载。

### 6.1 视频生成的务实建议（免费 16 GB 显存）

- 分辨率先从 `480P` 起步，帧数控制在 **20–49 帧（约 2–5 秒）**，跑通后再加大。
- 优先选 **fp8 / 量化版**权重；开启 `--force-fp16`；节点里必要时开 `vae tiling`、分块采样。
- 一次只加载一个视频模型，出完再换；避免同时驻留多个大编码器爆显存（OOM）。
- 报 CUDA OOM 时：降分辨率 / 减帧数 / 换更小的量化模型 / 关闭其他占显存进程。

---

## 7. 下次重启或新建 Notebook 后的复用流程

ComfyUI 程序本体不持久化，每次按下面顺序重建（模型走 Dataset，不用重新下载）：

1. 新建 / 打开 Notebook，右侧选 GPU、开 Internet、Persistence 设 `Files only`。
2. **Add Input** 挂载你的模型 Dataset（`/kaggle/input/<数据集名>`）。
3. 重跑第 3 节：clone ComfyUI + 装依赖 + Manager。
4. 重跑第 4.3 节：生成 `extra_model_paths.yaml` 指向 Dataset。
5. 重跑第 5 节：启动服务 + 开隧道，拿新公网地址使用。
6. 本次新生成的成品，结束前确认已下载本地或归档进 Dataset。

> 省事技巧：把上述 Cell 保存成自己的 Notebook 版本（**Save Version**），以后 **Fork** 这份 Notebook，每次只需挂载 Dataset、按顺序 Run All 即可，不用重写代码。

---

## 8. 常见坑与排查

| 现象 | 原因 / 处理 |
|---|---|
| 重启后模型全没了 | 模型只放在了 `/kaggle/working` 或临时目录；必须进 `/kaggle/input` 挂载的 Dataset |
| 界面下拉框看不到模型 | 路径没写进 `extra_model_paths.yaml`；按 `R` 刷新；确认 Dataset 子目录名与 yaml 键名对应 |
| `/kaggle/input` 下找不到文件 | 右侧 Input 面板没挂载成功；用 `!ls /kaggle/input/` 核对实际目录名 |
| CUDA out of memory | 降分辨率 / 减帧数 / 换 fp8 量化模型 / 加 `--force-fp16` / 清理已加载模型 |
| 隧道地址打不开 | 等 ComfyUI 真正就绪后再开隧道；cloudflared 偶发抽风可重跑 5.2；换 ngrok 方式 |
| 下载模型极慢或失败 | 确认 Internet 已开；HF 大文件用 `huggingface-cli` 断点续传；直链失效就换源 |
| 会话中途断连，working 被清空 | 这是 Kaggle 已知行为；重要模型与成品务必及时进 Dataset / 下载本地，不要只依赖 working |
| 磁盘写满 | `/kaggle/working` 约 20 GB；临时下载的模型用完及时 `rm -rf /kaggle/working/tmp_models` 再转 Dataset |

---

## 9. 一页速查清单

- [ ] 右侧：GPU 加速、Internet On、Persistence = Files only
- [ ] 挂载模型 Dataset 到 `/kaggle/input/`
- [ ] clone ComfyUI → `pip install -r requirements.txt`
- [ ] 写 `extra_model_paths.yaml` 映射 Dataset 各模型子目录
- [ ] 后台 `python main.py --listen 0.0.0.0 --port 8188 --force-fp16`
- [ ] Cloudflared / ngrok 暴露 8188，浏览器开公网地址
- [ ] 成品下载本地 / 归档 Dataset；临时文件及时清理

> （注：部分内容由豆包 AI 生成）