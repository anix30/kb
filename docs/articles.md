# 文章目录

## ComfyUI 提交器三函数逐行详解：api() / run() / show()

这份文档专门拆解 ModelScope（和 Kaggle）指南里那段 notebook 提交器代码。 三个函数的关系是分层的：

api()   —— 底层：发一个 HTTP 请求，把 ComfyUI 的响应解析成 Python 对象
  ↑
run()   —— 中层：用 api() 提交 workflow → 拿任务号 → 轮询直到出图 → 返回文件路径
  ↑
show()  —— 展示层：把 run() 返回的文件路径显示成图片

完整链路：txt2img() 造 workflow → run() 提交并等待 → show() 显示

[开始阅读](/comfyui-api-functions)

## Kaggle 免费资源跑 ComfyUI · 两阶段指南（先 CPU，后 GPU）

在 ModelScope Notebook 免费 CPU 上跑 ComfyUI的步骤搬到Kaggle上，不同点主要在notebook存储机制不同，其他流程基本相同，未使用cloudflared反代 。

[开始阅读](/comfyui-kaggle)

## 在ModelScope.cn 安装 ComfyUI，并用免费CPU学习）

在 ModelScope Notebook 免费 CPU 上跑 ComfyUI，生成图片，只是学习ComfyUI的工作流。

[开始阅读](/comfyui-ms-notebook)

## Kaggle 免费 GPU 安装 ComfyUI 完整流程（图片 / 视频生成 + 模型持久化）

在 Kaggle Notebook 免费 GPU 上跑 ComfyUI，生成图片或短视频，并让模型在会话重启 / 新建 Notebook 后不丢失。 所有代码块均可直接粘贴到 Kaggle 的 Code Cell 中按顺序运行。

[开始阅读](/Kaggle-ComfyUI-部署与模型持久化指南)

## Free-for-Dev 中英对照

面向开发者与开源作者的免费套餐清单，1480 条，57 个分类，英文 + 中文对照版。

<a href="/free-for-dev-bilingual.html" target="_self">打开页面</a>

## Awesome-Selfhosted 中英对照

可自托管的自由软件网络服务与 Web 应用清单, 1267条，95个分类， 英文 + 中文对照版

<a href="/Awesome-Selfhosted .html" target="_self">打开页面</a>

## Cloud Mail 部署全流程

用免费域名 + Cloudflare Worker、D1、KV，搭一个自己的网页邮箱，收信全流程，2026-11-11 实测走通过。

[开始阅读](/cloud-mail)
