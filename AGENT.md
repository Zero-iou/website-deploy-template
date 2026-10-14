# AGENT：网站、API 接入与上线

本文件供 AI 读取并向当前用户解释。它适用于任何用户，不继承模板作者的账号、凭据、站点授权或成功记录。文件名保持 `AGENT.md`；客户端不会自动读取时，请用户明确指定它。先调查现状，复用正确资源，只执行获准的差异。

## 1. 先完成 GitHub 接入

1. 检查当前平台的 GitHub 连接，通过服务返回的身份确认用户/组织；网页登录不等于连接器授权。没有连接时提供平台支持的授权入口，让用户在官方页面授权，不索要 PAT、密码或验证码截图。
2. 确认 `GITHUB_OWNER`、私有运维仓库 `DEPLOY_REPO`（默认 `website-deploy`）、注册域名 `DOMAIN`、站点源码仓库和认可 commit。新独立站默认使用 `<GITHUB_OWNER>/<DOMAIN>`；已有站保持既有仓库，不擅自改名。
3. 获准后由 AI 创建或复用用户自己的私有仓库，提供实际链接。复制模板不能复制 Secrets、GitHub App 授权或已有部署状态；不要把别人的公开 fork 当作用户的私有仓库。工具不支持建仓/写 workflow 时，准确说明缺少哪一步，使用获准的官方路径。

## 2. 两类仓库、三条授权链

- `website-deploy`：保存通用模板、AI 指南和经过审查的 API 运维操作；通过现有 Secrets 管理 Cloudflare/Spaceship。不从这里检出、构建或打包每个站点的私有源码。
- 站点源码仓库：保存网站代码、锁定依赖、运行时版本、构建适配器和 Worker 配置。Cloudflare Workers Builds 通过 GitHub App 直接读取此仓库并构建部署；默认生产分支 `main`，开启自动部署前须确认 main 就是认可版本。
- GitHub 连接授权让 AI 操作获准仓库；Cloudflare GitHub App 授权让 Cloudflare 读取指定站点仓库；Cloudflare API token 让获准的 API 调用生效。三者不能互相替代。
- 管理 API token 与 Cloudflare build token 分工不同。前者存在运维仓库，后者由 Cloudflare Builds 使用。原生路径无需 `SITE_REPO_READ_TOKEN`，也无需给每个站点复制运维仓库的 Cloudflare/Spaceship Secrets。
- 本公开模板不包含个人站点脚本或历史部署 workflow。用户自己的仓库若有历史流程，先读实际代码，不因模板文档而授权运行。

## 3. 教用户取得并配置凭据

按 [Cloudflare 配置](docs/CLOUDFLARE_SETUP.md) 和 [Spaceship 配置](docs/SPACESHIP_SETUP.md) 给出“官方入口 → 权限 → 资源范围 → 填在哪里 → 怎么验证”的具体步骤。每次只要求本次功能需要的能力，解释持续访问范围及风险。

用户本人将真实值直接填入自己运维仓库的 **Settings → Secrets and variables → Actions → New repository secret**：

- `CLOUDFLARE_API_TOKEN`：管理 API token。
- `SPACESHIP_API_KEY`、`SPACESHIP_API_SECRET`：需要操作 Spaceship 注册域名时配置。

真实 token、数据库密码和私钥不进聊天、源码、普通 Variables、artifact 或日志；AI 不读回秘密。账户 ID、zone ID、域名、Worker tag、trigger UUID 等非凭据配置可用经过校验的输入/Variables。创建凭据、OAuth 授权、配置持久访问或扩权须按当前平台的确认要求执行，不能从“有现成 Secret”推导新授权。

GSC 只用 [官方网页流程](docs/GSC_SETUP.md)，不要求 GSC API、服务账号 JSON 或 GSC GitHub Secret。先核实用户 Google 身份；DNS 等待期间即可从验证页取得真实 TXT，经批准写入当前权威 DNS 并完成所有权验证，迁移时在新 DNS 保留该记录。正式 HTTPS 可用后再提交 sitemap、检查实际网址与进行获准的索引请求。

## 4. 先只读验证，再决定变更

读取实际 workflow 后运行最小只读检查。分别报告缺 Secret、token 有效、目标可见、权限/范围不符、网络故障、API 不兼容和业务数据为空。模板只读验证通过 provider 选择 Cloudflare、Spaceship 或 both；选择 both 时分两个 job，一个成功不等于整次 run 成功。未选中的平台标记为未执行。`active` 和 GET 200 都不证明写权限已经实测，更不证明网站上线。

API 运维代码应固定官方 HTTPS 主机、拒绝带凭据的跳转、限制响应大小/超时，只输出允许字段与脱敏错误分类。不得打印全环境、认证头、完整 API 响应或未经筛选的构建日志。不通过未经批准的创建/删除来测试写权限；写入结果不确定时先读取状态，避免重复执行。

## 5. 原生构建与自动发布

1. 先确认源码、认可 commit、构建命令、输出目录、固定 Worker、账号和生产分支。新 Worker 默认用注册域名去掉公共后缀后的名称，例如 `example.com → example`、`example.co.uk → example`；名称不合法或撞名时先确认，已有正确 Worker 直接复用。保留站点内容与设计；模板只能提供可审查的起点。
2. 在 Cloudflare 官方界面核实 GitHub App 可读当前私有站点仓库，只授权必要仓库。创建/扩大 App 持久权限时取得相应确认。
3. 复用正确的 Worker、repo connection、trigger 和已有 build token。没有合适 token 时说明所需权限，让用户通过安全界面配置；不自动点“Create new token”。
4. Cloudflare Builds 直接连接站点仓库；设置正确根目录、项目实际 build/deploy 命令及 `main` 生产分支。默认关闭未审查的预览/PR 分支。不要把旧 main 自动晋升；获准后核对祖先关系并正常推进，不强推。
5. 自定义依赖安装仅在项目适配器已实现时设置 `SKIP_DEPENDENCY_INSTALL=true`。版本文件和包锁要一致；隔离 HOME/环境时注意版本管理器 shim，解析真实运行时路径，不为修复构建恢复全部凭据环境。子进程环境过滤不是对同机恶意代码的完整安全边界。
6. 保存真实源 commit、Cloudflare build UUID、构建结果及部署版本。失败先读取该次 build 的安全诊断再修复；不能用缓存离线测试代替 Cloudflare fresh build。首次连接或推送可能自动触发构建，先列出现有 build，避免重复手动触发。
7. 普通代码更新由同一原生集成发布，不重复切 NS、不复制中央 Secrets。部署成功后仍需独立验收域名。

## 6. 域名切换与 DNSSEC

先备份原 NS、完整 DNS（含邮件、CAA、验证记录）、规则、旧服务与回退版本。Cloudflare 自动扫描可能不完整；补齐必需记录并核对实际分配的 NS。保持旧服务可用。采用免费方案时不自动升级或创建收费资源。

按 [NS/DNSSEC 顺序](docs/SPACESHIP_SETUP.md#4-nsdnssec-切换顺序) 执行：记录父区旧 DS 和原始 TTL；获准移除旧 DS 后，确认父区权威移除并等待旧缓存 TTL，重新核对后再切 NS。递归解析器的剩余 TTL、注册商按钮成功、单次空响应都不能替代这个关卡。TTL 未知、仍有旧 DS 或 SERVFAIL 时先停查。

用户要求提前切换时，先明确剩余等待时间、可能的部分解析/邮件中断与回退局限，取得针对本次安全变更和提前切换的明确当次确认；不得把普通“上线”请求或历史授权当作跳过等待的许可。提前切换不等于缓存已清空，必须记录风险接受及持续检查。新 DS 只在新权威稳定后，按另行确认的 DNSSEC 方案启用。

zone Active 后核对既有 Worker，再处理获准的 Custom Domain 与证书。保留不相关记录。默认规范地址 `https://<DOMAIN>`，`www to root` 用 301 并保留 path/query；HTTP 最终到 HTTPS 主域。Single Redirect 需要被代理的入口记录。不得顺手更改 TLS 模式、HSTS 或全 zone 设置；脚本若实际会改，应提前列明。

## 7. 验收和向用户说明

- 分开核实：源码/main、自动构建、Worker 部署、NS/DS、zone Active、域绑定、证书、跳转、HTTP、GSC。
- 实测首页与重点页 200、不存在页真实 404；四种协议/主机入口的 Location、最终 URL、path/query 正确；手机页面、链接和资源可用；canonical、robots、sitemap 正确。
- GSC 所有权验证可在 DNS 等待期间先完成，确认使用当前权威 DNS 并保留验证 TXT 和用户所有权；sitemap 提交、实际网址检查和索引请求等到正式 HTTPS 可用后再做。提交或进入队列不保证收录。
- 给出官方 URL、私有源码仓库、认可 commit、原生 build/部署证据、API 运维 run、回退依据和当前阻塞。每项写“已完成 / 等待 / 未验证 / 失败”，不给笼统“全部已好”。
- 文档、模板或 workflow 存在本身不证明实现完成。任何历史成功记录只属于其当时的账号、域名和 commit。
