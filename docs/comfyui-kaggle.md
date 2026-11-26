# Kaggle 免费资源跑 ComfyUI · 两阶段指南（先 CPU，后 GPU）

> 基于 2026-11 Kaggle 镜像与资源规则整理（Python 3.13、Colab 基础镜像、预装 CUDA torch、
> GPU 通常是 T4 16G，**未在本机实测**）。
>
> **核心事实（和 ModelScope 最大的不同）**：
> Kaggle 同一台 notebook 通过 Settings → Accelerator 选 None（CPU）或 GPU，
> `/kaggle/working`（可写）每次新 session / 切换 accelerator 后都会重置。
> **但 `/kaggle/input` 是 Dataset 挂载区，是持久的**——模型文件放这里就只需下载一次，
> CPU / GPU 阶段、新 session 都能直接读，不用重复下。
> 所以正确玩法是：**第一次把模型下到 `/kaggle/working`，提交成 Dataset；以后挂载它，
> 从 `/kaggle/input` 直接读（或软链进 ComfyUI 的 models 目录）**。
>
> 你在 ModelScope 上已经跑通的 workflow JSON、`api()`/`run()`/`show()`、
> seed / steps / cfg 的全部结论——**都通用，不用重写**。

---

## 0-1. 先把模型做成持久 Dataset（只做一次）

这一步解决"模型每次重下"的问题。之后 A、B 两阶段都从 `/kaggle/input` 读模型。

**第一次（任一阶段，比如就在 CPU 阶段）**：把模型下载到可写目录

```python
from modelscope import snapshot_download
snapshot_download("AI-ModelScope/sd-turbo", local_dir="/kaggle/working/sd-turbo")
snapshot_download("AI-ModelScope/stable-diffusion-v1-5", local_dir="/kaggle/working/sd15")
```

**然后把它变成 Dataset**（两选一）：

- **界面操作**：左侧 Datasets → New Dataset → 上传上面两个文件夹（可打包成 zip 上传，
  Kaggle 会自动解压）。单文件上限约 20GB，SD-Turbo 5.2GB / SD1.5 4.2GB 都没问题。
- **代码操作**：notebook 里 `kaggle datasets create -p /kaggle/working/<导出目录>`（需先配 `kaggle.json` token）。

**以后每次开 notebook**：右侧边栏 **Add Input**（新版界面；旧版叫 Add Data，
在 Settings 里）→ 搜到你的 Dataset 挂上，文件就出现在 `/kaggle/input/<数据集名>/...`，
跨 session、CPU/GPU 都在，不用再下。

> 注意：ComfyUI 只扫描自己的 `models/checkpoints/` 目录，不会自动读 `/kaggle/input`。
> 两种接法（A2 和 B1 里会用）：
> - **软链**（推荐，零复制）：`ln -s /kaggle/input/<数据集>/sd_turbo.safetensors /kaggle/working/ComfyUI/models/checkpoints/`
> - 或写 `extra_model_paths.yaml` 放在 ComfyUI 根目录，让 ComfyUI 直接认 `/kaggle/input`
>   （ComfyUI 启动时会读这个文件；格式见官方 example，本指南用软链更简单）

---

# 阶段 A：CPU（搭环境 + 学工作流）

**目标**：在 Kaggle 免费 CPU notebook 上把环境跑通、把 7 节点工作流玩熟。
CPU 上 SD-Turbo 几秒~几十秒一张，SD1.5 一两分钟，完全够学习用。

## A0. 开 notebook

- New Notebook → **Accelerator: None**（就是 CPU）
- 语言 Python 3，其他默认

## A1. 搭环境（一个 cell 跑完）

Kaggle CPU 镜像已预装 torch（CPU 版），只缺 ComfyUI 的依赖：

```python
import os, subprocess, sys

W = "/kaggle/working"

# 1) clone 仓库
if not os.path.exists(f"{W}/ComfyUI/main.py"):
    subprocess.run(["git", "clone", "--depth", "1",
                    "https://github.com/Comfy-Org/ComfyUI.git",
                    f"{W}/ComfyUI"], check=True)

# 2) 装依赖（Kaggle 的 torch 是 CPU 版，ComfyUI 的 requirements 不会去动它）
%pip install -q -r {W}/ComfyUI/requirements.txt
%pip install -q modelscope
```

> **和 ModelScope 版的关键差异**：
> - 不建 venv（session 结束就没了，建了也带不走；直接装到系统 Python）
> - `%pip` 是 notebook 专用写法，不是 `!pip`
> - 路径是 `/kaggle/working`，没有 `source myenv.sh` 这一步

装完执行一次 **Runtime → Restart session**（`%pip` 之后已 import 的模块要重启才刷新），
然后从 A2 开始重新跑（A1 的 clone 有 `if not exists` 判断，重跑很快）。

## A2. 把模型接进 ComfyUI（从持久 Dataset 读，不用重下）

开 notebook 时已在右侧边栏 **Add Input** 挂上模型 Dataset，文件在 `/kaggle/input/<数据集名>/...`。
用软链接进 ComfyUI 的 checkpoints 目录（零复制，最省事）：

```python
import os, subprocess

W = "/kaggle/working"
DS = "/kaggle/input/<你的数据集名>"     # ← 改成实际名字，可用 os.listdir('/kaggle/input') 查看
ckpt = f"{W}/ComfyUI/models/checkpoints"
os.makedirs(ckpt, exist_ok=True)

# 找到模型文件（注意实际文件名 sd_turbo.safetensors，下划线）
import glob
src = glob.glob(f"{DS}/**/sd_turbo.safetensors", recursive=True)[0]
dst = f"{ckpt}/sd_turbo.safetensors"
if not os.path.exists(dst):
    os.symlink(src, dst)
print(os.listdir(ckpt))
```

> **第一次还没有 Dataset 时**：先按前面 0-1 节把模型下到 `/kaggle/working` 并提交成
> Dataset，之后就能像上面这样从 `/kaggle/input` 秒接。CPU 和 GPU 阶段都走这条路，模型只下这一次。

## A3. 启动 ComfyUI（CPU 模式）

```python
import subprocess, sys, time, urllib.request

log = open(f"{W}/comfy.log", "w")
proc = subprocess.Popen(
    [sys.executable, "main.py", "--cpu", "--listen", "127.0.0.1", "--port", "8188"],
    cwd=f"{W}/ComfyUI", stdout=log, stderr=log,
)
time.sleep(25)
print(urllib.request.urlopen("http://127.0.0.1:8188/system_stats", timeout=10).read()[:500])
```

> **有 `--cpu`** —— 这是 CPU 阶段与 GPU 阶段唯一实质性的命令差异。
> Kaggle 不需要 `OMP_NUM_THREADS=8`（CPU 镜像默认能用多核）。
> WebUI 不用管，和 ModelScope 一样直接用 API 提交。

## A4. 提交任务 + 学参数

把 ModelScope 指南第 4 节的 cell 原样复制过来，只改一个常量：

```python
OUT = "/kaggle/working/ComfyUI/output"
```

其余 `api()` / `run()` / `txt2img()` / `show()` **一个字都不用改**（`api()` 要用能打印 HTTP 400 响应体的改进版）。

然后就是你已经练熟的那套：

```python
# SD-Turbo：1 步、CFG=1
paths = run(txt2img("sd_turbo.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    steps=1, cfg=1.0, seed=42))
show(paths)

# 对比不同 seed
for s in [0, 42, 123, 2024]:
    show(run(txt2img("sd_turbo.safetensors",
                     "a red fox sitting on a rock, cinematic light",
                     steps=1, cfg=1.0, seed=s)))

# SD1.5：20 步、CFG=7（看"正常流程"）
paths = run(txt2img("v1-5-pruned-emaonly.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    "lowres, blurry, bad anatomy",
                    steps=20, cfg=7.0, seed=42))
show(paths)
```

学习路线照 ModelScope 指南第 5 节走：seed → steps/cfg → 正负 prompt 拆分 →
img2img → LoRA。**第 7 步（ControlNet / SDXL / upscale）留给 GPU 阶段。**

---

# 阶段 B：GPU（正式出图）

**目标**：换 GPU 加速器，跑同样的代码，出高质量图 / SDXL / ControlNet。

## B0. 开 GPU notebook

- New Notebook → Settings → Accelerator → **GPU（通常 T4 16G）**
- 每周 GPU 配额有限，用完等下周——别拿它学习、试错参数

> **CPU 阶段的东西不会"自动跟过来"**：新 notebook 是全新的 `/kaggle/working`，
> ComfyUI 仓库和 pip 依赖要重装（A1 那段，1–2 分钟）。
> **但模型不用重下** —— 右侧边栏 **Add Input** 挂上同一个 Dataset，从 `/kaggle/input` 软链进
> `models/checkpoints/` 即可（同 A2），模型只下这一次，CPU/GPU 通用。

## B1. 搭环境（和 A1 几乎一样，两处不同）

```python
import os, subprocess, sys

W = "/kaggle/working"
if not os.path.exists(f"{W}/ComfyUI/main.py"):
    subprocess.run(["git", "clone", "--depth", "1",
                    "https://github.com/Comfy-Org/ComfyUI.git",
                    f"{W}/ComfyUI"], check=True)
%pip install -q -r {W}/ComfyUI/requirements.txt
%pip install -q modelscope
```

**两处不同：**

1. **torch 不用动** —— Kaggle GPU 镜像预装的是带 CUDA 的版本。CPU 阶段那句
   "`--cpu`、torch 是 CPU 版"在这里都不存在。仍可确认一次：

   ```python
   import torch
   print(torch.cuda.is_available(), torch.cuda.get_device_name(0))
   # 应打印 True ... Tesla T4
   ```

   若真打印 `False`，才需要 `%pip install -q --force-reinstall torch torchvision`。

2. **模型接法同 A2**：右侧边栏 **Add Input** 挂载 Dataset，`glob` 找到 `/kaggle/input` 里的
   `sd_turbo.safetensors`，软链进 `models/checkpoints/`。**不需要重新下载。**
   `modelscope` 这行只在 B3 要下新模型（SDXL）时才需要装。

模型下载/复制步骤同 A2。

## B2. 启动 ComfyUI（GPU 模式）

```python
import subprocess, sys, time, urllib.request

log = open(f"{W}/comfy.log", "w")
proc = subprocess.Popen(
    [sys.executable, "main.py", "--listen", "127.0.0.1", "--port", "8188"],
    cwd=f"{W}/ComfyUI", stdout=log, stderr=log,
)
time.sleep(25)
print(urllib.request.urlopen("http://127.0.0.1:8188/system_stats", timeout=10).read()[:500])
```

> **对比 A3：唯一的变化就是删掉了 `--cpu`。**
> `--listen` / `--port` / API 地址 / 输出目录全部一样。
> `system_stats` 里 devices 应显示 T4，不再是 cpu。
> 万一 T4 16G 跑大模型爆显存，启动参数加 `--lowvram` 重试。

## B3. 出图（代码零改动）

A4 那段提交器、workflow、所有参数**照搬**，只是快得多：

```python
# 同样的代码，GPU 上几十秒出 20 张都行
paths = run(txt2img("v1-5-pruned-emaonly.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    "lowres, blurry, bad anatomy",
                    steps=20, cfg=7.0, seed=42))
show(paths)
```

这时才上 SDXL（CPU 上 10~20 分钟一张，纯折磨）。同样走持久 Dataset：
先在任一 notebook 里把它下下来、追加进你的模型 Dataset，以后挂载即可：

```python
from modelscope import snapshot_download
import shutil, glob
snapshot_download("AI-ModelScope/stable-diffusion-xl-base-1.0",
                  local_dir=f"{W}/models/sdxl")
# 把 sdxl 的 safetensors 也提交进 Dataset（或导出后 New/Update Dataset）
print(glob.glob(f"{W}/models/sdxl/**/*.safetensors", recursive=True))
```

这次挂载后，用 glob 找到 SDXL 文件软链进 `models/checkpoints/`（同 A2 的做法），
**核对实际文件名**后用 checkpoint 名调用同一个 `txt2img()` 即可（分辨率可开 1024）。
ControlNet / IPAdapter / 放大精修也都在这一阶段做。

---

# CPU vs GPU 流程对照表（一页看清）

| 步骤 | CPU（阶段 A） | GPU（阶段 B） |
|---|---|---|
| Accelerator 设置 | None | GPU (T4) |
| clone 仓库 | 一样 | 一样 |
| 装依赖 | `%pip install -r requirements.txt` + modelscope | 一样（modelscope 按需） |
| venv | 不建 | 不建 |
| torch | 预装 CPU 版，直接用 | 预装 CUDA 版，确认 `is_available()` 为 True |
| 下模型 | **只下一次**：0-1 节下到 `/kaggle/working` → 提交 Dataset | **不重下**：Add Input 挂载，从 `/kaggle/input` 软链 |
| **启动命令** | `python main.py --cpu --listen ... --port ...` | `python main.py --listen ... --port ...`（**只差 `--cpu`**） |
| API 提交器 / workflow | 完全一样 | 完全一样 |
| 适合做什么 | 学 7 节点、seed/steps/cfg、img2img、LoRA | 正式出图、SDXL、ControlNet、upscale |
| 速度（SD1.5 20步） | 1~2 分钟 | 几十秒甚至更快 |
| 配额 | 免费、无限（但 session 有时长限制） | 每周配额，用完等下周 |

---

# 通用避坑（两阶段都适用）

- **`%pip install` 后重启 kernel**：装完依赖后 Runtime → Restart session，否则已 import 的模块是旧的。
- **`sd_turbo.safetensors`（下划线）** 不是 `sd-turbo.safetensors`。
- **KSampler 没有 `clip` 输入**：用 ModelScope 指南里修正后的 7 节点版本。
- **`api()` 用改进版**：能打印 HTTP 400 响应体，别用会吞错误的旧版。
- **别挂机等图**：长时间无操作 session 会被回收，正在跑的任务一起死。
- **输出要主动保存**：`show()` 只在当前 session 有效；要带走图片用 File → Save Output 或下载。
- **路径全在 `/kaggle/working`**：这是相对于 ModelScope（`/mnt/workspace`）最容易打错的地方。
