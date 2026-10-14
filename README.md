# website-deploy-template

把网站上线流程整理成可复用的 AI 操作指南：GitHub 保存站点源码，Cloudflare Workers Builds 负责原生构建和部署，按需接入 Spaceship 域名管理与 Google Search Console。

**这是部署流程模板，不是一键全自动上线工具。** 当前包含通用指南、AI 入口和手动触发的只读凭据检查；不包含网站源码模板、个人站点脚本、真实凭据或历史部署记录。

## 工作方式

```text
你的站点源码仓库 → Cloudflare GitHub App → Workers Builds → Worker
你的私有运维仓库 → 管理 API → 查询 / 获准的配置变更
域名与 HTTPS 验收 → GSC 网页验证与 sitemap 提交
```

站点自动部署由 Cloudflare 原生 Git 集成提供。完成首次配置并启用生产分支自动发布后，后续代码更新沿用该集成；域名切换和 GSC 设置需单独完成，不在每次更新时重复执行。

## 快速开始

1. 点击 **Use this template → Create a new repository**，在自己的账号下创建私有运维仓库，建议命名为 `website-deploy`。也可以 fork 学习和修改；公开仓库的 fork 不适合作为私有运维仓库。
2. 将站点源码保存在自己的独立仓库中，确认目标账号、域名、Worker 和认可的代码版本。
3. 让 AI 读取 [AGENT.md](AGENT.md)，根据实际目标逐步接入服务。例如：

   > 请先读取 AGENT.md，检查当前仓库和我的目标，再指导我配置 Cloudflare 原生自动部署。真实凭据由我在官方安全表单填写。

4. 根据 [Cloudflare 配置指南](docs/CLOUDFLARE_SETUP.md) 创建最小必要权限的管理 token。需要 Spaceship 时再按 [Spaceship 指南](docs/SPACESHIP_SETUP.md) 配置相应凭据。
5. 在自己的运维仓库 **Settings → Secrets and variables → Actions** 中填写 Secrets，勿将真实值写进代码或聊天。
6. 在 **Actions → Verify API credentials (read-only) → Run workflow** 选择 `cloudflare`、`spaceship` 或 `both`。仅检查所选平台，不修改 DNS、域名或部署资源。
7. 按 Cloudflare 指南连接自己的站点源码仓库，配置实际构建/部署命令、生产分支和 build token。首次连接可能自动构建，先检查结果再决定是否重复触发。
8. 独立完成域名、DNSSEC、HTTPS 和网页验收，再按 [GSC 指南](docs/GSC_SETUP.md) 验证所有权、提交 sitemap。

## 凭据属于每位使用者自己

| Secret | 用途 | 何时需要 |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | Cloudflare 管理 API | 使用 Cloudflare 查询或运维时 |
| `SPACESHIP_API_KEY` | Spaceship API key | 使用 Spaceship 时 |
| `SPACESHIP_API_SECRET` | 与 API key 配对的 secret | 使用 Spaceship 时 |

复制模板或 fork **不会复制作者的 Secrets、账号授权、域名或部署状态**。所有用户使用自己的账号和密钥。

Cloudflare 管理 token、Cloudflare Builds 使用的 build token、GitHub App 授权是三件不同的事。原生 Builds 不要求给运维仓库额外配置跨仓库读取源码 token。GSC 使用官方网页，不需要 GitHub Secret。

## 已提供与未提供

- 已提供：Cloudflare、Spaceship、GSC 配置说明；DNSSEC 切换步骤；AI 操作约束；只读凭据检查 workflow。
- 未提供：可直接运行的网站源码模板、自动修改 DNS/NS 的通用脚本、全链路一键部署。`template/` 目前只有目录说明。
- 只读检查通过仅表示对应认证或读取检查通过，不证明写权限、网站上线或搜索收录。

## 安全与验收

只读 workflow 仅支持手动触发，使用空的 GitHub token 权限，不检出或执行外部 PR 代码，也不打印密钥或完整 API 响应。Cloudflare 可选可见性检查与 Spaceship 域名读取只统计第一页，不用它判定某个目标不存在。

个人运维仓库应保持私有，使用最小权限凭据。DNS 迁移前备份必要记录，按指南处理 DS/TTL 等待；部署、DNS、证书、HTTP 和 GSC 分别验收。

## 文档

- [AI 操作入口](AGENT.md)
- [Cloudflare 管理 API 与原生 Builds](docs/CLOUDFLARE_SETUP.md)
- [Spaceship API 与 NS/DNSSEC](docs/SPACESHIP_SETUP.md)
- [Google Search Console](docs/GSC_SETUP.md)
- [网站模板目录说明](template/README.md)
