# 密码管理器 Pro · Cloudflare 单文件版（xiaozhou 二开）

> 基于 [`ilikeeu/spassword`](https://github.com/ilikeeu/spassword) 二次开发。把密码管理器的前端与 API 合并在**一个 `_worker.js`** 里，数据存进你自己的 **Cloudflare KV**，登录走 **GitHub OAuth**，全站托管在 Cloudflare，不依赖任何服务器。
>
> 本项目站点：<https://mima.fuxicun.top>
>
> **二开改动（相对上游）**
>
> 1. 修复 **GitHub OAuth 端点不兼容**：上游写死 `/oauth2/authorize`、`/oauth2/token`、`/api/user`，GitHub 实际使用 `/login/oauth/authorize`、`/login/oauth/access_token`、`api.github.com/user`，直连会 404。本版新增 `resolveOAuthEndpoints()` 自动识别 GitHub 并适配令牌交换与用户字段映射。
> 2. 补齐工程化部署文件：`wrangler.toml`、`package.json`、GitHub Actions（可选），支持控制台接 Git 自动部署或命令行部署。
> 3. 补齐本详细部署文档（控制台 + 命令行两种路径）。

---

## 目录

- [核心功能](#核心功能)
- [部署前须知](#部署前须知)
- [控制台部署（推荐，全程网页操作）](#控制台部署推荐全程网页操作)
- [命令行部署（进阶）](#命令行部署进阶)
- [wrangler.toml 配置说明](#wranglertoml-配置说明)
- [环境变量一览](#环境变量一览)
- [OAuth 应用配置（GitHub / Gitee）](#oauth-应用配置github--gitee)
- [自动构建（二选一）](#自动构建二选一)
- [配套客户端（油猴脚本 / 浏览器扩展）](#配套客户端油猴脚本--浏览器扩展)
- [日常使用](#日常使用)
- [常见问题](#常见问题)
- [版权与致谢](#版权与致谢)

---

## 核心功能

- **密码管理**：增删改查、分类、批量导入导出、密码强度生成、重复检测、分页浏览。
- **密码历史**：修改密码自动留痕，误改可一键恢复，历史记录可删。
- **自动填充**（油猴脚本 / Chrome 扩展）：登录页自动识别表单、智能匹配账户、一键填充、密码变更检测。
- **WebDAV 云备份**：支持坚果云 / NextCloud 等，AES-GCM 加密备份，自动去重恢复。
- **GitHub OAuth 登录 + 用户隔离**：每个 GitHub 账户的数据按 userId 隔离；可选 `OAUTH_ID` 锁定单一账号访问。

## 部署前须知

- 需要：一个 **Cloudflare 账号**（免费版够用）+ 一个 **GitHub 账号**（创建 OAuth App 用）。
- 本项目为**单文件 Worker**：`_worker.js` 同时包含前端 HTML 与后端 API，**无需本地构建前端**，`npm run build` 是空脚本。
- **KV 绑定名固定为 `PASSWORD_KV`**：Worker 代码里写死 `env.PASSWORD_KV`，创建 KV 时命名空间标题可任意，但 `wrangler.toml` 中 `binding` 必须保持 `PASSWORD_KV`。
- **密钥不入仓库**：`OAUTH_CLIENT_SECRET`、`OAUTH_ID` 等敏感值一律在 Cloudflare 后台以「加密 Secret」设置，不写在任何代码/配置/文档里。`OAUTH_CLIENT_ID` 是公开标识（非密钥），可放 `wrangler.toml [vars]`。

---

## 控制台部署（推荐，全程网页操作）

### 第 1 步：创建 KV 命名空间

1. 打开 <https://dash.cloudflare.com> → 左侧 **Workers 和 Pages → KV**（顶部搜索框输入 `KV` 也可）。
2. **创建命名空间**，名称填 `xiaozhou-cloudflare-spassword-kv`（名称随意，便于识别即可）。
3. 创建完成后复制 **ID**（32 位十六进制串），下一步要用。

> 本项目已预置的 KV ID 为 `74c4b27365f44b1887c201660f6f117b`，如你重新创建，请在第 2 步换成你自己的 ID。

### 第 2 步：Fork 仓库并填入 KV ID

1. 在本仓库 GitHub 页面点右上角 **Fork** 到你自己的账号。
2. 在你 fork 的仓库里打开 `wrangler.toml`（网页点铅笔图标编辑），确认 `[[kv_namespaces]]` 段的 `id` 是第 1 步的 KV ID：

   ```toml
   name = "xiaozhou-cloudflare-spassword-web"

   [[kv_namespaces]]
   binding = "PASSWORD_KV"                        # 不可改
   id = "74c4b27365f44b1887c201660f6f117b"        # ← 换成你的 KV ID

   [vars]
   OAUTH_PROVIDER = "github"
   OAUTH_BASE_URL = "https://github.com"
   OAUTH_REDIRECT_URI = "https://你的域名/api/oauth/callback"
   OAUTH_CLIENT_ID = "你的 GitHub OAuth App Client ID"
   ```

   > 先填你自己的域名；域名在第 5 步绑定，回调地址必须与 GitHub OAuth App 中登记的完全一致。
   > **必须先提交再接第 4 步的 Git 自动部署**：`wrangler.toml` 的 `name` 与 Worker 名必须一致。

3. 点 **Commit changes** 提交。

### 第 3 步：创建 GitHub OAuth App

1. 打开 <https://github.com/settings/developers> → **OAuth Apps → New OAuth App**。
2. 填写：
   - **Application name**：`密码管理器 Pro`（随意）
   - **Homepage URL**：`https://mima.fuxicun.top`（你的站点域名）
   - **Authorization callback URL**：`https://mima.fuxicun.top/api/oauth/callback`（**必须与 `OAUTH_REDIRECT_URI` 完全一致**）
3. 创建后复制 **Client ID**（公开标识），并 **Generate a new client secret** 生成 **Client Secret**（只显示一次，立即保存）。
4. 可选：若要把访问锁定到你一个账号，访问 <https://api.github.com/user> 拿到你的数字 `id`，作为 `OAUTH_ID`。

### 第 4 步：创建 Worker 并配置环境变量与密钥

1. 控制台 → **Workers 和 Pages → 创建 → Worker**。
2. 名称填 `xiaozhou-cloudflare-spassword-web`（与 `wrangler.toml` 的 `name` 一致）。初始代码随便填，接 Git 后会被仓库代码覆盖。
3. 进入该 Worker → **Settings → Variables and Secrets**：
   - **明文变量（Variables）**：`wrangler.toml [vars]` 里的 4 项（`OAUTH_PROVIDER`、`OAUTH_BASE_URL`、`OAUTH_REDIRECT_URI`、`OAUTH_CLIENT_ID`）已在配置文件中，**接 Git 部署后会自动写入**，无需手动加。
   - **加密 Secret（Encrypted Secrets）**：点 **Add** 逐个添加：

     | 名称 | 值 | 必填 | 说明 |
     | --- | --- | --- | --- |
     | `OAUTH_CLIENT_SECRET` | 第 3 步生成的 Client Secret | **必填** | GitHub OAuth 应用密钥 |
     | `OAUTH_ID` | 你的 GitHub 数字 id | 可选 | 设了只允许该账号登录；不设则任何授权用户都可访问 |

     > 加密 Secret 只在添加那一刻显示，之后无法回看——**自己留一份私人备忘**，忘了就重设新值。

4. ⚠️ 改完密钥后**必须触发一次部署**才生效：往下接 Git 后 push 一次，或本地 `npx wrangler deploy`。仅在后台改值不部署，运行的仍是旧值。

### 第 5 步：接 GitHub 自动构建（Workers Builds）

Worker → **Settings → Builds → Connect to Git**：

**① 连接到仓库**

| 界面项 | 填什么 |
| --- | --- |
| Git 帐户 | 你刚授权的 GitHub 账号 |
| 存储库 | 你 fork 的仓库（如 `xiaozhou-cloudflare-spassword`） |
| 分支 | `main` |

**② 构建配置**

| 界面项 | 本项目怎么处理 |
| --- | --- |
| 构建命令（Build command，可选） | **留空**；本项目是单文件 Worker，无需构建。保留默认 `npm run build` 也没问题（空脚本，不会报错）。 |
| 部署命令（Deploy command） | **保持 `npx wrangler deploy`** |

> 「启用预览（Enable preview deployments）」默认开启，会用 `npx wrangler preview` 给 PR 建预览，本项目不需要，可**关掉**避免预览任务报错；开着也不影响 main 分支正式部署。

点保存后 CF 立即触发首次构建（几分钟）。以后往 `main` push 任何改动都会自动重新部署。

### 第 6 步：绑定自定义域名

Worker → **Settings → Domains & Routes → Add → Custom Domain** → 填 `mima.fuxicun.top`。

- 域名已托管在 Cloudflare：直接填，CF 自动配 DNS（CNAME）。
- 域名不在 CF：按提示在你 DNS 服务商加一条 `CNAME` 指向 CF 给出的地址。

### 第 7 步：验证

```bash
curl -I https://mima.fuxicun.top
# HTTP/2 200 即站点可访问
```

浏览器打开 <https://mima.fuxicun.top>，点「开始使用 GitHub 登录」，能跳转 GitHub 授权并回到站点显示登录成功，即全部配置正确。

---

## 命令行部署（进阶）

习惯 CLI 可跳过大部分网页步骤：

```bash
# 1. 克隆你 fork 的仓库
git clone https://github.com/你的用户名/xiaozhou-cloudflare-spassword.git
cd xiaozhou-cloudflare-spassword
npm install

# 2. 登录 Cloudflare（浏览器授权）
npx wrangler login

# 3. 创建 KV，把输出的 id 填进 wrangler.toml 的 [[kv_namespaces]].id
npx wrangler kv namespace create "PASSWORD_KV"

# 4. 设置敏感密钥（交互式输入，不会进仓库）
npx wrangler secret put OAUTH_CLIENT_SECRET
npx wrangler secret put OAUTH_ID        # 可选，锁定单一账号

# 5. 确认 wrangler.toml 的 vars 里域名、Client ID 已改成你的，然后部署
npx wrangler deploy

# 6. 查看实时日志（排错用）
npx wrangler tail
```

绑定域名也可以在命令行完成：

```bash
# 域名已托管在该 CF 账户时，用 route 绑定
npx wrangler domains add mima.fuxicun.top
```

> CLI 与 Workers Builds 二选一即可；两者都配也可，但一次 push 会触发两次部署。

---

## wrangler.toml 配置说明

```toml
name = "xiaozhou-cloudflare-spassword-web"   # ① Worker 名，全局唯一，与控制台一致
main = "_worker.js"                          # ② 唯一入口，单文件 Worker
compatibility_date = "2025-01-01"            # ③ 运行时兼容日期
compatibility_flags = ["nodejs_compat"]      # ④ Node 兼容标志

[[kv_namespaces]]
binding = "PASSWORD_KV"                      # ⑤ 固定，代码里 env.PASSWORD_KV
id = "74c4b27365f44b1887c201660f6f117b"      # ⑥ 你的 KV 命名空间 ID

[vars]                                       # ⑦ 非敏感明文变量（会入库、会随部署同步）
OAUTH_PROVIDER = "github"
OAUTH_BASE_URL = "https://github.com"
OAUTH_REDIRECT_URI = "https://mima.fuxicun.top/api/oauth/callback"
OAUTH_CLIENT_ID = "Iv1.ce420dca9b679885"

[observability]
enabled = true                               # ⑧ 打开 Worker 日志
```

要点：

- ③ `compatibility_date` 影响 Workers 运行时行为，升级前建议先 `wrangler dev` 本地验证。
- ⑤ 绑定名**不可改**，改了代码里所有 `env.PASSWORD_KV` 都要同步改。
- ⑦ `[vars]` 里的值会写进公开仓库，**只能放非敏感信息**；Client Secret、OAUTH_ID 用 Secret。
- 明文 Variable 会在每次 `wrangler deploy` 时按 toml 全量同步；Secret 不受影响、永久保留。

---

## 环境变量一览

| 变量名 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `PASSWORD_KV` | KV 绑定 | ✅ | 数据库；binding 名固定，id 填你的 KV ID |
| `OAUTH_PROVIDER` | Variable | 推荐 | `github` 或 `gitee`（二开新增，触发对应端点适配；不填时靠 `OAUTH_BASE_URL` 自动识别） |
| `OAUTH_BASE_URL` | Variable | ✅ | GitHub 填 `https://github.com`；Gitee 填 `https://gitee.com` |
| `OAUTH_CLIENT_ID` | Variable | ✅ | GitHub OAuth App 的 Client ID（公开标识） |
| `OAUTH_CLIENT_SECRET` | **Secret** | ✅ | GitHub OAuth App 的 Client Secret |
| `OAUTH_REDIRECT_URI` | Variable | ✅ | `https://你的域名/api/oauth/callback`，与 OAuth App 登记值一致 |
| `OAUTH_ID` | **Secret** | 可选 | 你的 GitHub 数字 id，设置后仅允许该账号登录 |

---

## OAuth 应用配置（GitHub / Gitee）

本项目二开已适配 **GitHub** 与 **Gitee（码云）** 两种 OAuth 流程，通过 `OAUTH_PROVIDER` 切换：

| 提供商 | `OAUTH_PROVIDER` | `OAUTH_BASE_URL` | 授权端点 | Token 端点 | 用户端点 | 国内访问 |
| --- | --- | --- | --- | --- | --- | --- |
| GitHub | `github` | `https://github.com` | `/login/oauth/authorize` | `/login/oauth/access_token` | `api.github.com/user` | 间歇性，不稳定 |
| Gitee | `gitee` | `https://gitee.com` | `/oauth/authorize` | `/oauth/token` | `gitee.com/api/v5/user` | **稳定可直连** |

> OAuth 流程中只有「跳转授权页」由用户浏览器访问提供商，换 token 与查用户信息都在 Cloudflare 海外节点完成。因此若你所在网络访问 GitHub 授权页不稳定，**推荐改用 Gitee**——Gitee 的 API V5 参照 GitHub 设计，用户字段（`login`/`name`/`avatar_url`）与本项目映射逻辑完全兼容。

标准流程（以 GitHub 为例）：

1. `GET /api/oauth/login` → 返回授权 URL，并把 `state` 存入 KV（10 分钟有效）。
2. 用户在提供商授权后回调 `GET /api/oauth/callback?code=...&state=...` → 校验 state → 换取 token。
3. 带 token 取用户信息（自动映射 `login→username`、`name→nickname`、`avatar_url→avatar_template`）。
4. 按 `userId` 隔离数据，会话存 KV，7 天有效。

### 切换到 Gitee（国内推荐）

1. 打开 <https://gitee.com/oauth/applications/new> 创建应用，回调地址填 `https://mima.fuxicun.top/api/oauth/callback`，拿到 Client ID 与 Client Secret。
2. 修改 `wrangler.toml [vars]`：
   ```toml
   OAUTH_PROVIDER = "gitee"
   OAUTH_BASE_URL = "https://gitee.com"
   OAUTH_CLIENT_ID = "你的 Gitee Client ID"
   ```
3. 把 `OAUTH_CLIENT_SECRET` 加密 Secret 更新为 Gitee 的 Client Secret（CF 后台或 `wrangler secret put`）。
4. push 触发自动部署。域名与回调地址不变。

若登录报 404 或 token 交换失败，优先核对：`OAUTH_PROVIDER` 与 `OAUTH_BASE_URL` 匹配、回调 URL 与 OAuth App 登记值完全一致、Client Secret 已作为 Secret 设置并重新部署。

---

## 自动构建（二选一）

本项目提供两种自动部署方式，**任选其一**：

- **Cloudflare Workers Builds（推荐）**：控制台 **Worker → Settings → Builds → Connect to Git**，见[第 5 步](#第-5-步接-github-自动构建workers-builds)。无需在仓库放密钥，CF 侧托管授权。
- **GitHub Actions**：使用仓库中的 `.github/workflows/deploy.yml`。需在 **仓库 Settings → Secrets and variables → Actions** 添加：
  - `CLOUDFLARE_API_TOKEN`：具有 Workers Scripts 编辑权限的 API Token
  - `CLOUDFLARE_ACCOUNT_ID`：CF 账户 ID

  添加后 push 到 `main` 或手动触发 workflow 即部署。

> 同时开启两者会导致一次 push 触发两次部署，建议只保留一种。

---

## 配套客户端（油猴脚本 / 浏览器扩展）

仓库内提供两个客户端：

- `tampermonkey脚本20250614.js`：油猴自动填充脚本。
- `chrome扩展20250611.zip`：Chrome 扩展。

**使用前必须改地址**：把脚本/扩展里的 `www.deno.dev`（或旧域名）全部替换为你的站点 `mima.fuxicun.top`（油猴脚本约 3 处）。扩展装好后，在系统管理页可能要多刷新几次令牌才显示。

---

## 日常使用

1. 打开 <https://mima.fuxicun.top> → 点「开始使用 GitHub 登录」→ 完成授权。
2. **添加密码**：「添加密码」页填写站点、账号、密码，可用内置生成器；保存后自动分类。
3. **管理密码**：搜索、分类筛选、分页浏览；支持查看/复制/编辑/删除，修改留历史可恢复。
4. **云备份**：「云备份」页填 WebDAV 信息（坚果云/NextCloud），测试连接后创建加密备份；恢复时选备份文件+密码，自动去重。
5. **自动填充**：登录任意站点时油猴脚本/扩展自动检测并浮出匹配账户，点击填充；新账户自动保存，密码变更会提示。

---

## 常见问题

1. **登录跳 GitHub 后 404 / token 交换失败**：检查 `OAUTH_PROVIDER=github`、`OAUTH_BASE_URL=https://github.com`、回调 URL 与 OAuth App 一致、Client Secret 已设且已重新部署。
2. **页面空白 / 报 `PASSWORD_KV is not defined`**：`wrangler.toml` 的 KV `binding` 必须是 `PASSWORD_KV`，且 id 正确；改完要重新部署。
3. **改了后台 Secret 不生效**：Secret 改完必须重新部署（push 或 `wrangler deploy`）。
4. **扩展检测不到账户**：确认扩展里的地址已改为 `mima.fuxicun.top`；在密码管理器页面先登录；多刷新几次。
5. **想只允许自己访问**：把你的 GitHub 数字 id 设为 `OAUTH_ID` Secret，其他账号登录会被拒绝。

排错命令：

```bash
npx wrangler tail                              # 实时日志
curl https://mima.fuxicun.top/api/oauth/login  # 应返回 authUrl（含 github.com/login/oauth/authorize）
```

---

## 版权与致谢

- 上游项目：[ilikeeu/spassword](https://github.com/ilikeeu/spassword)（MIT）。
- 本仓库在其基础上做 GitHub OAuth 端点适配与工程化部署改造，保留上游许可声明。
- 密码数据采用 AES-GCM 加密存储；请妥善保管 OAuth 账号与备份密码。
