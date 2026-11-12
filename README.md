# Peter 的知识库（VitePress + Cloudflare Pages）

## 这是什么

一个用 VitePress 搭的个人知识库，部署在 Cloudflare Pages 上，域名是
`kb.mynavidro.ccwu.cc`。文章是 Markdown 文件，改完推送到 GitHub 就自动发布。

## 本地预览（可选，不必须）

```bash
npm install
npm run docs:dev
```

然后打开 http://localhost:5173 看效果。

## 加一篇新文章

1. 在 `docs/` 下面新建一个 `.md` 文件，比如 `docs/vps.md`，用 Markdown 写内容。
2. 打开 `docs/.vitepress/config.mjs`，在 `sidebar` 的 `items` 里加一行：

```js
{ text: '文章标题', link: '/vps' }
```

（link 是文件名去掉 `.md`，前面加 `/`。）
3. 把改动推送到 GitHub，Cloudflare Pages 会自动重新部署，1–2 分钟后上线。

## 安全提醒

**不要把密码、密钥、验证码写进任何文章里。** 这个仓库是公开的，
所有内容任何人都能看到。写流程时，密钥类东西只写“填你自己的值”，
不要贴真实值。
