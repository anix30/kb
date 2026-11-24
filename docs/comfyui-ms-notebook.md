# 魔搭 Notebook · 纯 CPU 学 ComfyUI 工作流

目标：在 ModelScope 免费 **CPU 实例（8 核 32G）** 上把 ComfyUI 工作流搞明白，
环境 + 模型全部落在 `/mnt/workspace`，之后开 **GPU 实例直接复用同一块盘**，零重装。

> 依据：官方文档《Notebook介绍 → 免费资源额度用尽的处理 → 路径一：切换至 CPU 环境继续开发》
> 明确写了 CPU/GPU 实例可互访持久化数据。

---

## 0. 开实例前

- 个人中心 → 我的 Notebook → 新建 **CPU 环境实例**（认准带"免费"标识的规格）
- 必须已绑定阿里云账号（授权页**全部勾选**，否则绑定失败）
- 进入后右上角可切 **Web-IDE（VS Code）**，有 Terminal，比 Jupyter 好用

---

## 1. 装 ComfyUI（全部落在持久化盘）

```bash
cd /mnt/workspace/
git clone --depth 1 https://github.com/Comfy-Org/ComfyUI.git
cd ComfyUI
# 万一 github 拉不动，换镜像：
# git clone --depth 1 https://gitee.com/hf_nlp/ComfyUI.git

# 关键：安装虚拟环境
```bash
Python-m env env
source /mnt/workspace/bin/activate
```

# 关键：依赖装到持久化目录，否则关实例就没了
```bash
cd /mnt/workspace/ComfyUI
pip install -r requirements.txt
```

每次开新 shell 都要：

```bash
export PYTHONPATH=/mnt/workspace/pylibs:$PYTHONPATH
export OMP_NUM_THREADS=8          # 不加这个 torch 经常只吃 1~2 核
```

建议直接追加进 `/mnt/workspace/myenv.sh`，之后 `source` 它。

---

## 2. 下载模型（走魔搭，不要碰 HuggingFace）

Notebook 环境 **外网访问受限，官方明写包括 huggingface**，所以下载一律走 modelscope：

```bash
mkdir -p /mnt/workspace/ComfyUI/models/checkpoints

# SD1.5 全功能底模（20 步档，约 2GB）
modelscope download --model AI-ModelScope/stable-diffusion-v1-5 \
  --local_dir /tmp/sd15
cp /tmp/sd15/v1-5-pruned-emaonly.safetensors /mnt/workspace/ComfyUI/models/checkpoints/

# SD-Turbo（1 步档，学参数专用，约 2GB）
modelscope download --model AI-ModelScope/sd-turbo \
  --local_dir /tmp/turbo
cp /tmp/turbo/sd_turbo.safetensors /mnt/workspace/ComfyUI/models/checkpoints/

# SDXL（等 GPU 阶段用，CPU 上别跑）
# modelscope download --model AI-ModelScope/stable-diffusion-xl-base-1.0 --local_dir /tmp/sdxl
```

不确定文件名的话：`ls /tmp/sd15`。

100G 空间记法：`du -sh /mnt/workspace`

---

## 3. 启动 ComfyUI（CPU 模式，脚本化）

```bash
cd /mnt/workspace/ComfyUI
python main.py --cpu --listen 127.0.0.1 --port 8188 \
               --disable-auto-launch > /mnt/workspace/comfy.log 2>&1 &
```

等它加载完：

```bash
sleep 25; curl -s http://127.0.0.1:8188/system_stats | head -c 400
# 有返回（cpu 字段，devices 里 gpu 为 0 正常）就是起来了
```

> WebUI 从公网基本打不到（DSW 的 `/proxy/<port>/` 在魔搭免费实例上不一定开放，
> 内网穿透又被"外网受限"卡）。**不需要它** —— 下面直接用 API 提交任务。

---

## 4. 提交 workflow + 出图（Jupyter cell）

新建 ipynb（放在 `/mnt/workspace` 下！），一个 cell 定义好提交器：

```python
import json, os, time, urllib.request, io, uuid
import IPython.display as ipd

BASE = "http://127.0.0.1:8188"
OUT  = "/mnt/workspace/ComfyUI/output"

def api(path, data=None):
    req = urllib.request.Request(
        BASE + path,
        data=json.dumps(data).encode() if data is not None else None,
        headers={"Content-Type": "application/json"})
    return json.loads(urllib.request.urlopen(req, timeout=60).read())

def run(workflow, timeout=1800):
    """提交一个 API 格式的 workflow，阻塞等结果，返回 output 文件名列表"""
    client_id = uuid.uuid4().hex
    before = set(__import__("os").listdir(OUT))
    wf = json.loads(json.dumps(workflow))
    for n in wf.values():                      # 给 SaveImage 换唯一前缀，方便抓文件
        if n["class_type"] == "SaveImage":
            n["inputs"]["filename_prefix"] = "learn/" + client_id[:8]
    pid = api("/prompt", {"prompt": wf, "client_id": client_id})["prompt_id"]
    t0 = time.time()
    while time.time() - t0 < timeout:
        time.sleep(3)
        h = api("/history/" + pid)
        if pid in h:
            outs = h[pid].get("outputs", {})
            if "error" in h[pid].get("status", {}).get("status_str", ""):
                raise RuntimeError(json.dumps(h[pid], ensure_ascii=False)[:2000])
            return [os.path.join(OUT, i["subfolder"], i["filename"])
                    for o in outs.values() for i in o.get("images", [])]
    raise TimeoutError("CPU 跑太慢或进程已死，看 /mnt/workspace/comfy.log")

def show(paths):
    for p in paths:
        print(p)
        ipd.display(ipd.Image(filename=p))
```

### 最小 txt2img workflow（API 格式）—— 这就是你要看懂的东西

```python
def txt2img(ckpt, prompt, neg="", seed=0, steps=20, cfg=7.0, w=512, h=512):
    return {
      "1": {"class_type": "CheckpointLoaderSimple",
            "inputs": {"ckpt_name": ckpt}},
      "2": {"class_type": "CLIPTextEncode",          # 正向：文本 → conditioning
            "inputs": {"text": prompt, "clip": ["1", 1]}},
      "3": {"class_type": "CLIPTextEncode",          # 负向
            "inputs": {"text": neg, "clip": ["1", 1]}},
      "4": {"class_type": "EmptyLatentImage",        # 空潜空间
            "inputs": {"width": w, "height": h, "batch_size": 1}},
      "5": {"class_type": "KSampler",                # 采样：整条链的心脏
            "inputs": {"model": ["1", 0],
                       "positive": ["2", 0], "negative": ["3", 0],
                       "latent_image": ["4", 0],
                       "seed": seed, "steps": steps, "cfg": cfg,
                       "sampler_name": "euler", "scheduler": "normal",
                       "denoise": 1.0}},
      "6": {"class_type": "VAEDecode",               # 潜空间 → 像素
            "inputs": {"samples": ["5", 0], "vae": ["1", 2]}},
      "7": {"class_type": "SaveImage",
            "inputs": {"images": ["6", 0], "filename_prefix": "learn/out"}},
    }
```

看懂 `["1", 0]` 这套语法 = 看懂了 ComfyUI：**"取 1 号节点的第 0 个输出"**。
UI 里那根蓝线/紫线/黄线，在数据层就是这个元组。

### 跑起来

```python
# SD-Turbo：1 步、CFG=1，CPU 上约 6 秒 —— 学参数主力
paths = run(txt2img("sd_turbo.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    steps=1, cfg=1.0, seed=42))
show(paths)
```

```python
# SD1.5：20 步，约 1~2 分钟 —— 看"正常流程"长什么样
paths = run(txt2img("v1-5-pruned-emaonly.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    "lowres, blurry, bad anatomy",
                    steps=20, cfg=7.0, seed=42))
show(paths)
```

### 一次跑一组对比（CPU 上唯一舒服的练法）

```python
for s in [0, 42, 123, 2024]:
    show(run(txt2img("sd-turbo.safetensors",
                     "a red fox sitting on a rock, cinematic light",
                     steps=1, cfg=1.0, seed=s)))
```

---

## 5. 学习路线（顺序别乱，每步都在 CPU 上够快）

| 步 | 学什么 | 为什么 CPU 够用 |
|---|---|---|
| 1 | txt2img 全链路 7 个节点，各自吃/吐什么类型 | 6 秒/次，随便试错 |
| 2 | `seed` 固定 → 只动 `cfg` / `steps` / `sampler_name`，肉眼对比 | 单变量对比是 1 步就够的事 |
| 3 | 加 `CLIPTextEncode` 拆成正/负/风格三段；`ConditioningCombine`、权重 `(word:1.3)` | 纯文本层，不吃算力 |
| 4 | `EmptyLatentImage` 换 `LoadImage` + `VAEEncode` → **img2img**，玩 `denoise` | 核心：理解 denoise = 改动幅度 |
| 5 | `CheckpointLoader` 换成 `LoraLoader` 链式加载 | 加载一次常驻内存，之后每张仍 6 秒 |
| 6 | 分辨率与显存/内存的关系、`VAEDecode` 为什么 OOM | 认知准备，不必真跑大图 |
| 7 | **ControlNet / IPAdapter / upscale refine** —— SDXL 也一并放到 GPU 再做 | CPU 上 SDXL 10~20 分钟/张，纯折磨 |

第 1~6 步是概念，跟算力无关。第 7 步开始是重节点，那才是 GPU 的时间。

---

## 6. 避坑清单

- **闲置 1 小时自动关机、单次最长 10 小时。** 跑批之前算一下时间；产出随时落盘。
- 只有 `/mnt/workspace/` 下的东西持久。模型、仓库、pylibs、output、workflow JSON 全放这里。
- 盘**不提供 SLA**，重要产出自已下载备份；账号 365 天无活动可能被清。
- **别开 WebUI 挂着等图** —— 挂机容易被判"空闲"（你没在 notebook 里操作），一次 SDXL 白跑。
- 关机不扣 GPU 额度（按运行时长扣），但正在跑的任务会断。
- 36 小时 GPU（A10 24G，一次性额度，用完不补）**只用来正式出图和跑 ControlNet**，别拿它装环境 —— 官方文档都是这个意思。
- 额度余量看：个人控制台 → 我的 Notebook 右上角倒计时。

---

## 7. 转 GPU 那天

1. 关闭 CPU 实例
2. 新建实例时选 GPU 规格（8 核 32G + 24G 显存）
3. `cd /mnt/workspace/ComfyUI`，`source /mnt/workspace/myenv.sh`
4. 启动去掉 `--cpu`：`python main.py --listen 127.0.0.1 --port 8188 &`
5. 上面那个 notebook **一行代码都不用改** —— 工作流和模型完全通用
6. 顺手把 SDXL 权重 `cp` 到 `models/checkpoints/`，正式出图
