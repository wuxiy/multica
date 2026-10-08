# Multica 懒猫微服（Lazycat）lpk 打包

本仓库以上游 [multica-ai/multica](https://github.com/multica-ai/multica) 为模板，
在其之上追加懒猫 lpk 打包配置，可跟随上游持续更新并重新打包。

## 新增文件（均为 lpk 专用，不与上游文件冲突）

| 文件 | 作用 |
| ---- | ---- |
| `package.yml` | 应用元数据与权限（LPK V2） |
| `lzc-manifest.yml` | 运行结构：postgres / backend / frontend 三服务 + 网关路由 |
| `lzc-deploy-params.yml` | 安装时参数：JWT 密钥（随机）、Resend Key 与发件地址（可选）、允许注册、GitHub App Slug 与 Webhook Secret（可选） |
| `lzc-build.yml` | 打包配置 |
| `lzc/icon.png` | 应用图标（取自 `apps/web/public/icons/icon-512.png`） |

## 架构

直接复用上游自部署 Docker 镜像（版本号 pinned 在 `lzc-manifest.yml`）：

- `pgvector/pgvector:pg17` — 数据库，数据落盘 `/lzcapp/var/data/postgres`
- `ghcr.io/multica-ai/multica-backend:vX.Y.Z` — Go API + WebSocket（:8080），
  启动时自动跑数据库迁移，上传文件落盘 `/lzcapp/var/data/uploads`
- `ghcr.io/multica-ai/multica-web:vX.Y.Z` — Next.js 前端（:3000）

路由与上游 `apps/web/proxy.ts` 的分流保持一致：

- `/api` `/v1` `/uploads` `/ws` `/health` → 网关直达 backend（保留前缀，
  WebSocket 由懒猫网关代理；这些路径声明为 `public_path`，走 Multica 自身鉴权，
  便于 `multica` CLI / agent daemon 从其他设备接入）
- 其余（含 `/auth/*`）→ frontend:3000（Next 运行时代理会把其中的后端接口
  转发到 `REMOTE_API_URL=http://backend:8080`，`/auth/callback` 等前端页面不受影响）

外部地址通过 `{{ .S.AppDomain }}` 模板注入 `FRONTEND_ORIGIN` /
`MULTICA_APP_URL` / `MULTICA_PUBLIC_URL`，CORS 与 WebSocket origin 校验随之生效。

默认 `DO_NOT_TRACK=1`（关闭上游的匿名部署遥测）；如需开启请删除该环境变量。

## GitHub 集成

安装向导里的 `GitHub App Slug` + `GitHub Webhook Secret` 两者都填写后，后端的
`isGitHubConfigured()` 即开启 GitHub 集成（PR 镜像到卡片、安装流程）。在 GitHub
App 设置里把 Webhook URL 填 `https://<应用域名>/api/webhooks/github`（该路径已
路由到后端且免懒猫登录），secret 与向导里填的一致。

后续补配/修改这两个值：浏览器打开 `https://<应用域名>/_lzc/sys/setup/index`
重走参数向导；若盒子不再弹出表单，把 `lzc-deploy-params.yml` 里对应参数的
`optional: true` 临时去掉重新 deploy，会强制重弹向导（填完改回）。

浏览仓库/PR 富化还需要 `GITHUB_APP_ID` + `GITHUB_APP_PRIVATE_KEY`（完整 PEM）。
PEM 是多行的，向导参数和 YAML env 都装不下，lpk 也不允许 setup_script 与
entrypoint/command 并存——所以 backend 的 setup_script 会在每次启动前给镜像
的 `/app/entrypoint.sh` 幂等地前置一段 source 逻辑，从持久化文件
`/lzcapp/var/config/github-app.env` 注入（首次启动自动生成模板）。填法：

```bash
# 本地编辑好 github-app.env，格式：
#   GITHUB_APP_ID=123456
#   GITHUB_APP_PRIVATE_KEY="
#   -----BEGIN RSA PRIVATE KEY-----
#   ...（GitHub App 设置页下载的 PEM 原样粘贴）...
#   -----END RSA PRIVATE KEY-----
#   "
B64=$(base64 < github-app.env | tr -d '\n')
lzc-cli project exec --release -s backend -- sh -c "echo $B64 | base64 -d > /config/github-app.env"
lzc-cli project start --release --restart
```

注意：manifest 里所有 `.U.*` 引用都带 `{{ if }}` 守卫——未存储的参数不守卫会
渲染成字面量 `<no value>`，它是非空字符串，会把 `isGitHubConfigured()` 之类
的开关悄悄打开。

## 打包与部署

```bash
lzc-cli project build -o release/multica.lpk   # 打包
lzc-cli project deploy --release               # 部署到默认盒子（开发验证）
lzc-cli app install release/multica.lpk        # 安装 lpk
```

登录方式：安装时填写 Resend API Key 则验证码发邮箱；未配置时验证码打印在
backend 容器日志（`[DEV] Verification code for ...`）。

## 跟随上游更新

```bash
git fetch upstream
git merge upstream/main        # lpk 文件不在上游，不会冲突；
                               # 若上游改了 .gitignore 等再手动处理
# 按需升级镜像 tag（lzc-manifest.yml 中三处 image 与 package.yml 的 version）
lzc-cli project build -o release/multica.lpk
lzc-cli app install release/multica.lpk   # 覆盖升级，数据保留
```

上游镜像 tag 与 Git tag 对应（如 `v0.6.1`），见
<https://github.com/multica-ai/multica/pkgs/container/multica-backend>。

## 上架商店说明

`lzc-cli` 的 lint 会提示镜像需转存到 `registry.lazycat.cloud` 才能上架商店；
个人安装使用 GHCR / Docker Hub 镜像不受影响。若要上架，把三个 image 转存到
懒猫官方仓库后替换前缀即可。
