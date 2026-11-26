# ComfyUI 提交器三函数逐行详解：`api()` / `run()` / `show()`

这份文档专门拆解 ModelScope（和 Kaggle）指南里那段 notebook 提交器代码。
三个函数的关系是**分层**的：

```
api()   —— 底层：发一个 HTTP 请求，把 ComfyUI 的响应解析成 Python 对象
  ↑
run()   —— 中层：用 api() 提交 workflow → 拿任务号 → 轮询直到出图 → 返回文件路径
  ↑
show()  —— 展示层：把 run() 返回的文件路径显示成图片

完整链路：txt2img() 造 workflow → run() 提交并等待 → show() 显示
```

---

## 0. 前置：import 和常量

```python
import json, os, time, urllib.request, io, uuid
import IPython.display as ipd

BASE = "http://127.0.0.1:8188"
OUT  = "/mnt/workspace/ComfyUI/output"
```

| 行 | 作用 |
|---|---|
| `import json` | 处理 JSON：把 Python 字典序列化成 JSON 字符串、把 JSON 响应反序列化成字典 |
| `import os` | 文件路径操作（`os.listdir`、`os.path.join`） |
| `import time` | 计时（`time.time()`）和等待（`time.sleep()`） |
| `import urllib.request` | 发 HTTP 请求的标准库（不用装第三方包） |
| `import io` | 内存中的字节流；这段提交器里其实**没用到**，是预留的 |
| `import uuid` | 生成全局唯一 ID，给每个任务一个不重复的 `client_id` |
| `import IPython.display as ipd` | Jupyter 专用：把图片、文本渲染到 notebook 输出里 |
| `BASE = "..."` | ComfyUI 服务的地址。API 模式的 ComfyUI 跑在本机 8188 端口，所有请求都发到这个前缀 |
| `OUT = "..."` | ComfyUI 保存图片的目录。`run()` 要据此拼出图片完整路径；**Kaggle 版要改成 `/kaggle/working/ComfyUI/output`** |

> 一个 Python 细节：`except` 里用到的 `urllib.error` 没有单独 import，但
> `import urllib.request` 时 Python 会自动把 `urllib.error` 一起加载进来，所以能用。
> 想写得严谨一点可以补一行 `import urllib.error`。

---

## 1. `api(path, data=None)` —— HTTP 请求封装

### 它解决什么问题

ComfyUI 启动后是一个 HTTP 服务器。你要让它干活，本质是发一个 HTTP 请求到
`http://127.0.0.1:8188/xxx`，带上 JSON 数据，再读它返回的 JSON。
`api()` 把这套"发请求 → 收响应 → 解析 JSON → 出错时打印原因"封装成一个函数，
后面 `run()` 里所有跟服务器通信都调它。

### 代码（最终版，含错误处理）

```python
def api(path, data=None):
    req = urllib.request.Request(
        BASE + path,
        data=json.dumps(data).encode() if data is not None else None,
        headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=60) as r:
            return json.loads(r.read())
    except urllib.error.HTTPError as e:
        # 400 的响应体里有 error / node_errors，直接打印出来，别只丢一个 "HTTP 400"
        body = e.read().decode("utf-8", "replace")
        print("HTTP", e.code, body[:4000])
        raise RuntimeError("ComfyUI 拒绝了请求，见上面打印的 error/node_errors") from None
```

### 逐行拆解

**第 1 行 `def api(path, data=None):`**
定义函数，两个参数：
- `path`：要请求的 API 路径，比如 `/prompt`、`/history/<任务号>`。函数内部会拼到 `BASE` 后面。
- `data=None`：要发送的数据（Python 字典）。默认 `None` 表示"这次是 GET 请求，不带数据"。

**第 2–5 行 构造 `Request` 对象**

```python
req = urllib.request.Request(
    BASE + path,
    data=json.dumps(data).encode() if data is not None else None,
    headers={"Content-Type": "application/json"})
```

- `BASE + path` —— 完整 URL。例如 `path="/prompt"` 时得到 `http://127.0.0.1:8188/prompt`。
- `data=json.dumps(data).encode() if data is not None else None` —— 这是关键的三段式：
  1. `json.dumps(data)`：把 Python 字典转成 JSON 字符串（如 `{"a": 1}` → `'{"a": 1}'`），因为 HTTP 传输的是文本；
  2. `.encode()`：把字符串编码成字节（UTF-8），因为 HTTP body 必须是 bytes；
  3. `if data is not None else None`：如果没传数据就给 `None`。**这一点很重要**：`urllib` 看到 `data=None` 就发 GET 请求，看到 `data=字节` 就自动发 POST 请求。所以这一个对象同时支持 GET（查状态）和 POST（提交任务）。
- `headers={"Content-Type": "application/json"}` —— 告诉服务器"我发的是 JSON 格式"，服务器才会按 JSON 解析 body。

**第 6–8 行 发请求并解析响应**

```python
try:
    with urllib.request.urlopen(req, timeout=60) as r:
        return json.loads(r.read())
```

- `try:` —— 因为服务器可能返回错误（400/500），要预备捕获。
- `with urllib.request.urlopen(req, timeout=60) as r:` —— 真正发出请求，等待服务器响应：
  - `timeout=60`：最多等 60 秒，防止服务器卡死时程序永远挂住；
  - `with ... as r`：`r` 是响应对象，`with` 保证用完后自动关闭连接（好习惯）。
- `return json.loads(r.read())` —— `r.read()` 读出服务器返回的字节，`json.loads()` 把 JSON 文本还原成 Python 字典，然后返回。
  比如提交成功后返回 `{"prompt_id": "...", "node_errors": {}}`，调用方就能用 `["prompt_id"]` 取值。

**第 9–13 行 错误处理（这段是后来补上的，很重要）**

```python
except urllib.error.HTTPError as e:
    body = e.read().decode("utf-8", "replace")
    print("HTTP", e.code, body[:4000])
    raise RuntimeError("ComfyUI 拒绝了请求，见上面打印的 error/node_errors") from None
```

- `except urllib.error.HTTPError as e:` —— 服务器返回 4xx/5xx 时，`urlopen` 会抛出 `HTTPError`。你之前遇到的 `HTTP 400: Bad Request` 就是它。
- `body = e.read().decode("utf-8", "replace")` —— **关键**：错误信息不在异常消息里，而在响应体里。`e.read()` 把响应体读出来，`decode()` 转成字符串。`"replace"` 表示遇到无法解码的字节就用替代字符，避免程序因编码问题再崩一次。
- `print("HTTP", e.code, body[:4000])` —— 打印状态码和错误体。ComfyUI 的 400 响应体形如
  `{"error": {...}, "node_errors": {...}}`，里面写明了到底是哪个节点、什么原因出错——这正是之前那个"HTTP 400 看不出原因"问题被解决的地方。`body[:4000]` 是截断，防止错误太长刷屏。
- `raise RuntimeError(...) from None` —— 抛出一个新的、更友好的异常，让调用者知道去读上面打印的信息；`from None` 是抑制"原始异常链"，让 traceback 更干净（否则会先打印一大段 HTTPError 栈，干扰视线）。

> 这正是我们之前排查的两个 bug 能定位的原因：模型文件名写错、KSampler 多传了 `clip`，
> 都是靠这里打印出的 `node_errors` 找到的。

---

## 2. `run(workflow, timeout=1800)` —— 提交任务 + 轮询等待

### 它解决什么问题

ComfyUI 出图是**异步**的：`POST /prompt` 只是把任务排进队列，立刻返回一个任务号，
图还没画出来。你得拿这个任务号反复去问"画好了吗？"，直到完成，再去输出目录取图。
`run()` 把这整个过程封装成**一个阻塞调用**——调用它就像调用一个普通函数，返回时图已经画好了。

### 代码

```python
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
```

### 逐行拆解

**第 1 行 `def run(workflow, timeout=1800):`**
- `workflow`：API 格式的 workflow 字典，由 `txt2img()` 生成（7 个节点的那套）。
- `timeout=1800`：最长等待 1800 秒 = 30 分钟。CPU 上 SD1.5 约 1–2 分钟，够用；超时就放弃。

**第 2 行 文档字符串**
说明这个函数的作用：提交 workflow、阻塞等结果、返回输出文件名列表。

**第 3 行 `client_id = uuid.uuid4().hex`**
生成一个 32 位十六进制的随机字符串（如 `a3f2...`），作为本次任务的客户端 ID。
作用：让每个任务有唯一标识，避免多个任务的结果混淆；后面还用它生成文件名前缀。

**第 4 行 `before = set(__import__("os").listdir(OUT))`**
列出输出目录里现有的文件，存成集合。**说明：这行是遗留代码，后面根本没用到**——
早期版本想用"前后文件差集"来找出新生成的图，后来改成用唯一文件名前缀 + 从 history 读文件名，
这个办法更可靠，于是这行就闲置了。你读代码时不必纠结它；
`__import__("os")` 只是 `import os` 的动态写法，顶部已经 import 过 `os` 了，写 `os.listdir` 即可。

**第 5 行 `wf = json.loads(json.dumps(workflow))`**
这是 Python 里常用的**深拷贝**技巧：先序列化成 JSON 字符串，再反序列化回来。
目的是得到一份完全独立的新字典。为什么要复制？因为下一步要修改 SaveImage 的 `filename_prefix`，
不能改动调用方传进来的原始 workflow（否则你循环跑不同 seed 时，前缀会被反复改写）。

**第 6–8 行 给 SaveImage 换唯一前缀**

```python
for n in wf.values():
    if n["class_type"] == "SaveImage":
        n["inputs"]["filename_prefix"] = "learn/" + client_id[:8]
```

- 遍历 workflow 里所有节点（`wf.values()` 是每个节点的字典）。
- 找到 `class_type == "SaveImage"` 的那个节点（就是 7 号节点）。
- 把它的 `filename_prefix` 改成 `"learn/" + client_id[:8]`，比如 `learn/a3f2c1b9`。
  这样本次任务生成的图片文件名都带这个独特前缀，存到 `output/learn/` 子目录下，
  下次任务前缀不同，不会互相覆盖，也不会误抓到别的任务的图。

**第 9 行 `pid = api("/prompt", {"prompt": wf, "client_id": client_id})["prompt_id"]`**
这是最核心的一行，做了三件事：
1. 调 `api("/prompt", {...})` 发一个 POST 请求，body 是 `{"prompt": wf, "client_id": client_id}`——
   把 workflow 提交给 ComfyUI；
2. ComfyUI 校验通过后返回一个字典：`{"prompt_id": "...", "number": 0, "node_errors": {}}`；
3. `["prompt_id"]` 取出任务号存进 `pid`，后面靠它查询结果。

> 回想一下：之前那个 `HTTP 400` 就是卡在这一行——服务器校验 workflow 失败，没返回 `prompt_id`，
> 而是返回了 400 错误。我们靠改进的 `api()` 打印错误体定位了问题。

**第 10 行 `t0 = time.time()`**
记下开始时间，用于后面计算是否已经超时。

**第 11 行 `while time.time() - t0 < timeout:`**
轮询循环：只要还没超过 `timeout` 秒，就一直检查。

**第 12 行 `time.sleep(3)`**
每轮先等 3 秒再查。因为出图需要时间，不能疯狂轮询刷爆服务器；3 秒是合理间隔。

**第 13 行 `h = api("/history/" + pid)`**
用 GET 请求查这个任务的历史记录。ComfyUI 会把已完成任务的详细信息（包括输出的图片）
存在 `/history/<任务号>` 里。比如 `pid="abc"` 时请求 `/history/abc`。

**第 14 行 `if pid in h:`**
`history` 接口返回的是一个以任务号为 key 的字典。任务还在跑时，这个字典里没有 `pid`；
任务完成后才会出现。所以这句话的意思是"如果结果出来了"。

**第 15 行 `outs = h[pid].get("outputs", {})`**
取出这个任务的输出信息，它是一个字典：key 是节点编号（如 `"7"`），value 是该节点的输出，
里面通常有 `"images": [{"filename": ..., "subfolder": ..., "type": ...}]`。
`.get("outputs", {})` 表示没有 `outputs` 字段就返回空字典，避免 KeyError。

**第 16–17 行 检查执行错误**

```python
if "error" in h[pid].get("status", {}).get("status_str", ""):
    raise RuntimeError(json.dumps(h[pid], ensure_ascii=False)[:2000])
```

- workflow 提交校验通过、但实际执行时出错（比如显存不足、节点运行报错）时，
  `status.status_str` 会包含 `"error"`。
- 这时把整个 history 记录转成 JSON 字符串打印出来（截断 2000 字符），
  `ensure_ascii=False` 保证中文错误信息不被转义成一串 `\uXXXX`，方便阅读。
- 你之前那个 `KSampler.sample() got an unexpected keyword argument 'clip'` 就是从这类错误里看到的。

**第 18–19 行 汇总图片路径并返回**

```python
return [os.path.join(OUT, i["subfolder"], i["filename"])
        for o in outs.values() for i in o.get("images", [])]
```

这是一个嵌套的列表推导式，展开写等价于：

```python
result = []
for o in outs.values():              # 遍历每个输出节点
    for i in o.get("images", []):    # 遍历该节点产出的每张图片
        result.append(os.path.join(OUT, i["subfolder"], i["filename"]))
return result
```

- `o` 是每个节点的输出；`i` 是其中一张图片的元信息。
- `i["subfolder"]` 是子目录（我们改的前缀里有 `learn/`，所以是 `learn`）；
  `i["filename"]` 是文件名（如 `a3f2c1b9_00001.png`）。
- `os.path.join(OUT, subfolder, filename)` 拼出完整绝对路径，
  如 `/mnt/workspace/ComfyUI/output/learn/a3f2c1b9_00001.png`。
- 返回的是一个路径列表，`show()` 就拿它来显示。

**第 20 行 `raise TimeoutError("CPU 跑太慢或进程已死，看 ...")`**
如果 `while` 循环跑满 `timeout` 秒都没等到结果（任务没完成），抛出超时异常，
提示去看 `/mnt/workspace/comfy.log` 排查——常见原因是 CPU 上跑了太重的任务，或 ComfyUI 进程崩了。

---

## 3. `show(paths)` —— 显示图片

### 它解决什么问题

`run()` 返回的是一串文件路径字符串（如 `['/mnt/workspace/ComfyUI/output/learn/xxx.png']`）。
`show()` 负责把每个路径指向的图片真正渲染到 notebook 输出里，让你肉眼看到结果。

### 代码

```python
def show(paths):
    for p in paths:
        print(p)
        ipd.display(ipd.Image(filename=p))
```

### 逐行拆解

**第 1 行 `def show(paths):`**
参数 `paths` 是 `run()` 返回的图片路径列表。

**第 2 行 `for p in paths:`**
逐个处理每张图。批量跑多个 seed 时，`paths` 里会有多张图，这个循环会把它们依次显示出来。

**第 3 行 `print(p)`**
先把文件路径打印成文字。作用：让你知道这张图存在哪里，方便之后下载、备份或在别处打开。

**第 4 行 `ipd.display(ipd.Image(filename=p))`**
分两层理解：
- `ipd.Image(filename=p)` —— 创建一个 IPython 的 Image 对象，从路径 `p` 读取图片文件内容；
- `ipd.display(...)` —— 把这个对象渲染到 notebook 的当前 cell 输出里。

> 为什么不能只 `print` 路径？因为 notebook 不会自动把路径对应的图片显示出来，
> 必须用 IPython.display 显式渲染。这也是为什么顶部要 `import IPython.display as ipd`。
> 在 Kaggle 里这一段完全通用。

---

## 4. 三者如何串起来：一次完整调用

以你跑过的那次为例：

```python
paths = run(txt2img("sd_turbo.safetensors",
                    "a red fox sitting on a rock, cinematic light",
                    steps=1, cfg=1.0, seed=42))
show(paths)
```

发生的事情依次是：

1. `txt2img(...)` —— 按参数造出一个 7 节点的 workflow 字典（还没提交）；
2. `run(workflow)` ——
   a. 复制 workflow，给 SaveImage 换上唯一前缀；
   b. 调 `api("/prompt", ...)` 提交任务，拿到 `prompt_id`；
   c. 每 3 秒调一次 `api("/history/" + pid)` 轮询；
   d. 任务完成后，从 history 里提取图片路径，返回给 `paths`；
3. `show(paths)` —— 打印路径并把图片显示出来。

**一句话记忆**：
`api` 是"怎么跟服务器说话"，`run` 是"提交任务并等到图出来"，`show` 是"把图给我看"。

---

## 5. 常见改动点

| 你想改什么 | 动哪个函数/变量 |
|---|---|
| 从 ModelScope 搬到 Kaggle | 只改常量 `OUT` → `/kaggle/working/ComfyUI/output` |
| 换 ComfyUI 端口 | 只改常量 `BASE`（要和启动命令的 `--port` 一致） |
| 等待时间不够（CPU 跑大图超时） | 调大 `run(..., timeout=1800)` 的 1800 |
| 想加快轮询频率 | 改 `run()` 里的 `time.sleep(3)`（太小会刷爆服务器） |
| 报错看不清 | 确认 `api()` 用的是带 try/except 打印 `body` 的改进版 |
| 想保存图片到别处 | 改 workflow 里 SaveImage 的 `filename_prefix`，或拿到 `paths` 后自己 `shutil.copy` |

---

## 6. 代码里两个可以顺手清理的小瑕疵

不影响运行，但读代码时别被绕进去：

1. `run()` 里的 `before = set(__import__("os").listdir(OUT))` 是**死代码**（定义后从未使用）。
   可以直接删掉，行为完全不变。
2. `__import__("os")` 等价于顶部已 import 的 `os`，直接写 `os.listdir(OUT)` 即可；
   `except` 里用到的 `urllib.error` 建议在顶部补一行 `import urllib.error` 更规范。

这两处是写指南时留下的历史痕迹，知道即可，不必照抄进自己的代码。
