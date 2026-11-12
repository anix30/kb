# Cloud Mail 部署全流程

> 公开版：根据 2026-11-11 实际走通的一次部署整理。本文以免费域名 `zsqxl.de5.net` 为例，
> 换成你自己的域名，步骤完全一样。
> **已去掉密钥、账号和各类 ID**，照着做时填你自己的值。密钥类东西只记在自己的密码本里，
> 不要写进公开文章。
>
> 每一步都写了：**去哪 → 点什么 → 填什么 → 怎么确认做对了**。

---

## 先看懂全局：我们到底在搭什么

Cloud Mail 不是一个网页文件，而是一个跑在 Cloudflare 上的小程序（Worker）。一封邮件到达你的邮箱，要经过这条链：

```
别人发邮件
→ 域名的 MX 记录把邮件指给 Cloudflare
→ Cloudflare 的 Email Routing（邮件路由）接住
→ Catch-all 规则把邮件交给 Worker 程序 cloud-mail
→ 程序把邮件存进 D1 数据库，网页端从 D1 里读出来给你看
```

所以整个部署就是把这条链一节一节接起来：

| 部件 | 作用 | 在哪 |
|---|---|---|
| 域名 | 邮箱地址的后缀，如 `@你的域名` | 域名商申请，托管到 Cloudflare |
| Worker | 邮箱程序本体（网页端 + 收信处理） | Cloudflare Workers，代码来自 GitHub |
| D1 | 存邮件和账号的数据库 | Cloudflare，绑定名必须叫 `db` |
| KV | 存登录状态等临时数据 | Cloudflare，绑定名必须叫 `kv` |
| Email Routing | 收信的传送带 | 域名 zone 里开启 |

## 准备清单（开工前先备齐）

- [ ] 一个 Cloudflare 账号（免费）
- [ ] 一个 GitHub 账号（免费）
- [ ] 一个**能修改 NS（域名服务器）的域名**，且没有在做别的用途
- [ ] 一点耐心：有几个环节是“等生效”，几分钟到几小时不等，不是卡住了

⚠️ 一个域名只能托管一处、根域只能绑一个应用。已经在跑网站/节点的域名不要混用，
另申请一个新域名专做邮箱（这次就是这么做的，原来的节点域名完全没动）。

---

## 第 1 步：申请域名

1. 打开你的域名商后台，在免费域名里搜一个想要的前缀，选后缀，注册。
2. 拿到域名后，在“我的域名”里能看到它，状态“正常”，解析记录为空。

**完成标志：** 域名出现在你的域名列表里。

> 如果你用的是自己买的域名（比如在 Cloudflare 或其他注册商买的），跳过这步，
> 从第 2 步开始，第 3 步改 NS 的地点换成你的注册商后台，原理完全一样。

## 第 2 步：把域名添加进 Cloudflare，拿到 NS 地址

1. 登录 Cloudflare 控制台（dash.cloudflare.com）。
2. 点 **Add a domain / 添加域名**，输入你的域名，继续。
3. 套餐选最下面的 **Free（免费，$0）**，继续。
4. 它会扫描已有 DNS 记录，新域名是空的，扫不到东西是正常的，直接继续/确认。
5. 走到 **“更换域名服务器（Change your nameservers）”** 页面，Cloudflare 会分配给你**两个 NS 地址**，形如：
   - `xxxx.ns.cloudflare.com`
   - `yyyy.ns.cloudflare.com`
6. **把这两个地址原样抄下来**（一个字母都不能错），先别关这个页面。

**完成标志：** 域名出现在 Cloudflare 的域名列表里，状态显示 “Waiting for your registrar to propagate your new nameservers（等待注册商传播）”。

## 第 3 步：回域名商后台，把 NS 改成 Cloudflare 的

这步是告诉全世界：“这个域名以后归 Cloudflare 管。”

以 dnshe 为例：

1. 打开 my.dnshe.com → 你的域名 → **“DNS服务器”**。
2. 弹窗里把默认的两行（`ns1.dnshe.com` / `ns2.dnshe.com`）删掉，改为第 2 步抄下的两个 Cloudflare 地址，**每行一个**。
3. **勾选“强制替换冲突记录”**。
4. 点 **保存设置**。

**⚠️ 真实踩过的坑（重点看）：**

- 第二个地址 `elisabeth.ns.cloudflare.com` 比较长，手机输入框里可能显示不全（只看到 `...cloudflare.co`），**很容易以为填对了、其实末尾少了个 `m`**。填完点进输入框、把光标移到最后，确认结尾是完整的 `.com`。
- 第一次保存报了 **“DNS 操作失败”**，原因就是上面两点之一（拼写/没勾强制替换）。改对 + 勾选后保存成功。
- 保存后弹窗上方“当前 NS”那行会显示你填的两个地址，以那行为准核对一遍。

**完成标志：** 域名商后台显示该域名已使用“外部 DNS”；“当前 NS”是 Cloudflare 的两个地址。

## 第 4 步：等域名在 Cloudflare 变为 Active

1. 什么都不用做，等。Cloudflare 会自己去检查 NS 改好没有。
2. 想催一下：域名概览页有 **“Check nameservers now（立即检查）”** 按钮，可以点一次。
3. 公网其实十几分钟就通了，但控制台状态晚了约 50 分钟才翻绿。**控制台显示“等待传播”不等于没生效**，两者是两回事，别急着回去乱改。

**完成标志：** 域名概览页显示绿色对勾 **“Your domain is now protected by Cloudflare”**，激活邮件也会发到你的邮箱。

> 小知识：免费后缀域名（如 de5.net）的父级 DNS 偶尔会有 DNSSEC 小毛病，个别解析器（如 Google 的）会间歇性查不到这个域名，导致个别邮件迟到几分钟到几小时。这是平台问题，不是你配错了，自用场景可以接受；重要邮箱请用自己买的正经域名。

## 第 5 步：在 GitHub 上 Fork 一份 cloud-mail 代码

先解释 Fork：**它发生在 GitHub 云端**——把原作者的仓库复制一份到你自己的 GitHub 账号名下，得到 `你的用户名/cloud-mail`。不是下载到手机/电脑，手机只是你点按钮的工具。

1. 浏览器打开 https://github.com/maillab/cloud-mail ，用你的 GitHub 账号登录。
2. 点右上角 **Fork**。
3. Owner 选你自己，仓库名保持 `cloud-mail` 不变，点 **Create fork**。

**完成标志：** 页面跳到 `你的用户名/cloud-mail`，标题下方有一行小字 “forked from maillab/cloud-mail”。

## 第 6 步：在 Cloudflare 创建 D1 数据库和 KV（先备好，部署时要用）

1. 控制台左侧 **Storage & Databases（存储和数据库）→ D1** → **Create database**，名字填 `cloud-mail`，其余默认，创建。
2. 创建完进入详情页，把 **Database ID** 抄下来（一串 UUID），自己记好。
3. 再去 **Storage & Databases → Workers KV** → **Create a KV namespace**，名字也填 `cloud-mail`，创建。
4. 在 KV 列表里把这个命名空间的 **ID** 抄下来，自己记好。

**完成标志：** D1 列表和 KV 列表里各有一个叫 `cloud-mail` 的资源，ID 都记下来了。

> 注意：名字叫什么其实不关键（叫 cloud-mail 只是好记），关键是第 9 步绑定时**变量名**必须叫 `db` 和 `kv`。

## 第 7 步：部署 Worker（从 GitHub 导入）

1. 控制台左侧 **Workers & Pages** → **Create application（创建应用）**。
2. 选 **从 GitHub 导入 / Connect GitHub**。
3. 第一次用会跳到 GitHub 让你**登录并授权 “Cloudflare Workers and Pages”** 应用：登录你的 GitHub 账号 → 点同意/Install。安装范围建议选 **Only select repositories**，只勾 `cloud-mail` 一个仓库，权限最小够用。
4. 授权后回到 Cloudflare，选中仓库 **`你的用户名/cloud-mail`**。
5. 配置这样填：
   - 项目名（Worker 名）：`cloud-mail`
   - 生产分支：`main`（默认即可）
   - **根目录（Root directory / Path）：`mail-worker`** ← 最关键的一项，填错必失败
   - 构建命令（Build command）：留空（无）
   - 部署命令（Deploy command）：`npx wrangler deploy`（页面默认就是它）
6. 点部署，等构建跑完（约 1 分多钟：克隆 → 安装依赖 → 部署）。

**完成标志：** 部署列表里最新版本显示 **100% Ready**；Worker 概览页能看到一个地址，形如 `cloud-mail.你的子域.workers.dev`。先用这个地址往下走，正式域名第 12 步再绑。

## 第 8 步：给 Worker 填三个环境变量

1. 进入这个 Worker → **Settings（设置）→ Variables and Secrets（变量与机密）**。
2. 添加三个普通文本变量（Production 环境）：

| 变量名 | 填什么 | 说明 |
|---|---|---|
| `domain` | `["你的域名"]` | JSON 数组格式，引号、方括号照抄，只换域名 |
| `admin` | `admin@你的域名` | 这个邮箱注册出来自动是管理员 |
| `jwt_secret` | 一串随机字符 | 登录令牌的签名密钥。自己生成一串 32 位以上的随机字母数字（不要用特殊字符），记在自己的密码本里 |

3. 保存，页面会提示重新部署，点确认（Add variables and deploy）。

**完成标志：** 变量列表里三项都在，最新部署 Ready。

> `jwt_secret` 相当于这个邮箱系统的总钥匙之一：第 10 步初始化要用它，**不要发到公开场合**，记在自己的密码本里。

## 第 9 步：绑定 D1 和 KV

1. 同一个 Worker → **Settings → Bindings（绑定）** → Add：
   - 选 **D1 database**，**变量名填 `db`**，数据库选第 6 步建的 `cloud-mail`。
   - 再 Add 一次：选 **KV namespace**，**变量名填 `kv`**，命名空间选 `cloud-mail`。
2. 每次添加后控制台会自动重新部署一次，等它 Ready。

**⚠️ 变量名必须是 `db` 和 `kv`，一个字母都不能改**——这是程序里写死的，填别的名程序找不到数据库，会报错或白屏。

**完成标志：** Bindings 列表里有 `db`（D1）和 `kv`（KV）两条，最新部署 100% Ready。

> 附带认识一下：列表里可能还有程序自带的 `ai`（Workers AI）和 `forward`（邮件转发）两个绑定，不用管、不要删。

## 第 10 步：初始化系统（建数据库表）

1. 在浏览器地址栏访问（把地址换成你的）：

```
https://cloud-mail.你的子域.workers.dev/api/init/你的jwt_secret
```

2. 页面返回 **`success`** 就是成功了——这一步是在 D1 里把账号、邮件等数据表建出来。只做一次即可。

**完成标志：** 页面显示 `success`；打开 Worker 首页地址能看到 Cloud Mail 登录界面。

## 第 11 步：注册管理员和日常账号，然后关掉公开注册

顺序很重要：**先把自己要用的账号都注册好，最后再关注册。**

1. 打开邮箱站点（先用 workers.dev 地址也行），点 **创建账号/注册**。
2. 第一个注册管理员：邮箱前缀填 `admin`（后缀选你的域名），自己设一个密码。注册成功后它自动就是管理员。
3. 退出登录，再注册你的日常账号，想注册几个注册几个。
4. 用 admin 登录 → 左侧菜单 **系统设置** → “网站设置”卡片里第一个开关 **“用户注册”** → **关掉**并保存。
   - 关掉之后，外人无法再自行注册；以后要加账号，由管理员在 **用户列表** 里手动建。

**⚠️ 真实踩过的坑：**
- 登录时报 **“输入的邮箱不存在”**，八成是**前缀拼错了**（比如把 `zhengsq` 打成 `zhensq`，少个 g）。这个报错是登录接口的，意思是“这个邮箱还没注册过”，先核对拼写、再确认你点的是“登录”还是“注册”。
- 登录页底下“还没有账号？创建账号”才是注册入口；在已登录界面里的“添加账号”是把已有账号挂到一起，不是注册新账号的地方，别走错。

**完成标志：** admin 后台的用户列表里能看到你注册的全部账号；退出后打开站点，注册入口已关闭/注册会提示“注册功能已关闭”。

## 第 12 步：把正式域名绑到 Worker

（前提：第 4 步的域名状态已 Active。）

1. Worker → **Settings → Domains & Routes（域名与路由）** → **Add Domain（添加域）**。
2. 选你的域名，子域名**留空**（表示绑根域本身），Enable for 选 **Production**，确认。
3. 等 **HTTPS 证书签发**：刚绑完打开 `https://你的域名` 可能报 SSL 错误（如 `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`），这是证书还在签发（控制台 SSL/TLS → Edge Certificates 里状态是 Pending Validation），**正常等就行**，一般 10–30 分钟自动好。

**完成标志：** 浏览器打开 `https://你的域名` 能看到和 workers.dev 一样的登录界面。

## 第 13 步：开启 Email Routing（收信传送带）

1. 进入你的域名 zone → 左侧 **Email（电子邮件）→ Email Routing**。
2. 若提示 DNS records 未配置：点 **Fix DNS records** → 进 Settings → **Add missing records**，它会自动添加 5 条记录（3 条 MX + SPF + DKIM），加完 DNS records 状态变 **Locked**（表示被 Email Routing 托管，别手动改）。
   - 自动加的 MX 指向 `route1/route2/route3.mx.cloudflare.net`，这是 Cloudflare 的收信服务器。
3. 回到 **Routing rules（路由规则）**，找到 **Catch-all（全部捕获）** 那条，编辑：
   - Action 选 **Send to a Worker**
   - Destination（目标 Worker）选 **cloud-mail**
   - 保存后把这条规则的**开关打开**。
4. 总览页确认 **Routing status: Enabled**。

**完成标志：** Catch-all 显示 Active、动作是 cloud-mail；Routing status 为 Enabled。

> 为什么要 Catch-all：它表示“发到这个域名下**任何前缀**的邮件都收”，这样你注册的每个账号、甚至没建过的前缀，邮件都会先进系统，不会漏。

## 第 14 步：验收（一定要做）

1. 用一个**外部邮箱**（Gmail、QQ 邮箱等，不能用新邮箱自己给自己发）给你的新地址发一封信，主题随便写“测试”。
2. 登录新邮箱看收件箱，**收到了 = 全线打通**。
3. 免费后缀域名的父区 DNSSEC 偶尔有小毛病，邮件可能迟到 10–15 分钟；Cloudflare 控制台的 Activity log（活动日志）刷出来更慢。**没立刻到先别慌，等 10–15 分钟再看**；超过半小时还没到，再去 Email Routing 的 Activity log 查这封信的状态（Delivered / Dropped / Bounced）。

---

## 参数对照表（照着填你自己的）

| 项目 | 填什么 |
|---|---|
| 邮箱域名 | 你的域名（在域名商申请，托管到 Cloudflare） |
| Cloudflare Zone ID | 在域名概览页右下角看，不用记，查得到就行 |
| 分配的 NS | 第 2 步 Cloudflare 给你的两个地址，原样抄 |
| GitHub 仓库 | `你的用户名/cloud-mail`（fork 自 maillab/cloud-mail），根目录 `mail-worker` |
| Worker 名 / 地址 | `cloud-mail` / `https://cloud-mail.你的子域.workers.dev` |
| 正式入口 | `https://你的域名` |
| D1 | 名称 `cloud-mail`，ID 在详情页抄，绑定名 `db` |
| KV | 名称 `cloud-mail`，ID 在列表里抄，绑定名 `kv` |
| 环境变量 | `domain=["你的域名"]`、`admin=admin@你的域名`、`jwt_secret=` 你自己生成的随机串 |
| MX 记录 | `route3.mx.cloudflare.net`（优先级 67）、`route2.mx.cloudflare.net`（75）、`route1.mx.cloudflare.net`（95），由 Email Routing 自动添加 |

## 故障速查（按现象找）

| 现象 | 大概率原因 | 怎么办 |
|---|---|---|
| 域名商保存 NS 报“DNS 操作失败” | NS 拼写不全（末尾少字母）或没勾强制替换 | 逐字核对两个地址 + 勾“强制替换冲突记录”再保存 |
| CF 控制台一直显示等待传播 | 它自家检测慢，公网可能早通了 | 点 “Check nameservers now”，或等几十分钟到几小时 |
| 部署失败 / 程序白屏 | 根目录没填 `mail-worker`，或绑定名不是 `db`/`kv` | 回第 7、9 步核对这两处 |
| 访问 /api/init 不返回 success | jwt_secret 与环境变量里填的不一致 | 核对地址里的密钥和第 8 步填的完全一致 |
| 登录报“输入的邮箱不存在” | 前缀拼错，或在登录页输了还没注册的邮箱 | 核对拼写；没注册的先走“创建账号” |
| `https://域名` 报 SSL 错误 | 边缘证书还在签发 | 等 10–30 分钟，别改配置 |
| 邮件迟迟不到 | 发送方解析受父区 DNSSEC 间歇问题影响；或 Catch-all 没开 | 先等 15 分钟；检查第 13 步 Catch-all 是否 Active |
| 改了代码重新部署后变量/绑定丢了 | 控制台里加的变量/绑定没同步进仓库的 wrangler 配置 | 控制台 Settings 页有提示 “Update your Wrangler configuration…” 并给出代码片段，把它同步提交到 GitHub 仓库的 `mail-worker` 配置文件里；或每次部署后回控制台补加 |

## 日常使用与可选进阶

- **日常登录**：打开 `https://你的域名` ，用任意已注册账号登录。加新账号：admin 登录后在用户列表里建（公开注册保持关闭）。
- **现在只能收信、不能往外发**（站内账号之间可以互发）。想往外发信，需要接第三方发信服务 **Resend**（免费额度）：在 resend.com 注册并添加域名 → 按它给的值在 Cloudflare DNS 里加几条验证记录 → 把 Resend 的 API Key 填到 Cloud Mail 后台的系统设置里。
- **附件存储**：默认存在 KV 里，够用即可；附件量大时可再开 Cloudflare R2 并在后台切换。
- **再次提醒**：免费共享后缀域名信誉有限，只适合收验证码、注册小号和自用；银行、主账号等重要用途请用自有域名（把本文流程里的域名换成你买的域名，其余步骤一模一样）。

---

*本文档根据 2026-11-11 的真实部署过程整理，界面名称以当时 Cloudflare 控制台为准；Cloudflare 改版后按钮位置可能微调，按意思找同名功能即可。*
