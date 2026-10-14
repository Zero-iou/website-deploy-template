# Cloudflare：管理 API 与原生 Builds 配置

适用于任意用户。先确认 `<GITHUB_OWNER>/<DEPLOY_REPO>`、`<DOMAIN>`、目标 Cloudflare account/zone 和站点源码仓库；所有示例都要替换成用户的实际目标。本指南不表示任何凭据、权限或部署已经完成。官方依据核对日期：2026-10-13；执行时仍检查当前页面与 API。

## 1. 先分清三件事

1. **管理 API token**：由 `website-deploy` 的可信运维流程使用，保存为 `CLOUDFLARE_API_TOKEN`，用于获准的查询、Builds 配置、DNS/域名操作。
2. **build token**：由 Cloudflare Builds 部署站点时使用，在 Cloudflare 的 Worker → Settings → Builds → API token 中选择；管理 API 只保存/使用它的 UUID，不需要把其秘密复制到网站仓库。
3. **Cloudflare GitHub App**：从 GitHub 读取私有站点源码。API token 并不授予 GitHub 仓库访问；初次 App 授权需在官方界面完成，优先只选必要仓库。

[官方 Builds API 指南](https://developers.cloudflare.com/workers/ci-cd/builds/api-reference/)明确区分管理 token 与 build token；目前 Builds API/Builds 使用 user token，不能把不兼容的 account-owned token 当作等价替代。

## 2. 先列操作，再勾权限

### 管理 token：按本次实际动作组合

| 目的 | 权限/角色和范围 |
| --- | --- |
| 查账号、zone | 需要列账号时 Account → Account Settings → Read；目标 zone 的 Zone → Zone → Read / DNS → Read，按实际 GET 端点核对 |
| 读取 Builds、trigger、build token 元数据 | API 权限目录为 Workers CI Read；端点也接受 Workers CI Write。仅只读时不主动申请写入 |
| 管理 Builds 连接/trigger、触发 build | 官方接入指南称 Workers Builds Configuration → Edit；API 端点称 Workers CI Write。另需 Workers Scripts Read 来查 Worker tag。按当前 UI/新角色映射核实，不盲目叠加所有权限 |
| 新建 zone | Create Zone 接受 Zone Zone Edit **或** Zone DNS Edit；资源范围必须覆盖待创建 zone，必要时经批准选“指定账号的所有 zones”，不是全部账号 |
| 更新 DNS | Zone → DNS → Edit（API：DNS Write），限定目标 zone |
| 创建 Worker / 更新已有 Worker | 当前角色模型：新建需要目标账号的 Workers 产品级 Admin；已有 Worker 更新为 Editor。可细分时只选目标 Worker |
| 添加/改变 Custom Domain、Route | Workers Editor 加目标 zone 的 Workers Routes Write（UI 常显示 Zone → Workers Routes → Edit）；当前 Custom Domains 尚不支持 per-Worker roles，须按官方限制核对所需产品范围 |
| www/HTTP Single Redirect | Zone → Single Redirect → Edit；API 名 Dynamic URL Redirects Write，限定目标 zone |

Cloudflare 当前官方文档同时存在新角色和旧 API 权限名称：Workers Scripts Edit 对应 Workers Editor，Workers CI Edit 也映射到产品级 Editor；旧名称不是另一组必须重复授予的权限。查看实际 token 界面和调用端点，记录权限与范围；若支持范围不一致，先解释并取得最小必要扩权确认。

依据：[Workers 角色、范围和旧权限映射](https://developers.cloudflare.com/workers/authorization/workers/) · [权限目录](https://developers.cloudflare.com/fundamentals/api/reference/permissions/) · [Builds 读取权限](https://developers.cloudflare.com/api/resources/workers_builds/subresources/triggers/methods/list/) · [Create Zone](https://developers.cloudflare.com/api/resources/zones/methods/create/) · [Single Redirect](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/create-api/)

### build token：先满足静态站最小需求

纯静态 assets、部署到**已存在** Worker、构建不改变路由/域绑定时，只需要该部署所需的 Workers Editor 能力；旧 UI 名为 Account → Workers Scripts → Edit。不要默认加数据库、KV、R2、Hyperdrive 或所有 zone 的 Routes 权限。若需要新建 Worker或改路由，按上表另行授权。沿用已审查且可用的 build token，避免随手创建第二个。

Builds 的自动“Create new token”流程可能预置 Account Settings、Workers Scripts、KV、R2、所有 zones 的 Routes 等较宽权限，不能把它当成静态站最小权限模板。先审查摘要，创建新凭据必须取得相应当次确认。[Builds token 配置](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/#api-token)

### 可选后端资源权限

只有工作流确实要直接创建、迁移、读取或修改这些资源时才增加。仅部署引用已有绑定的 Worker，不自动意味着要这些产品的写权限。

| 实际操作 | 自定义 token UI / API 权限名 | 资源范围 |
| --- | --- | --- |
| 创建或修改 D1、执行写迁移 | Account → D1 → Edit / D1 Write | 指定目标账号；脚本固定实际数据库 ID |
| 创建 KV namespace、写 key | Account → Workers KV Storage → Edit / Workers KV Storage Write | 指定目标账号；脚本固定实际 namespace ID |
| 创建 R2 bucket、管理对象 | Account → Workers R2 Storage → Edit / Workers R2 Storage Write | 指定目标账号；脚本固定实际 bucket；S3/单 bucket 凭据另行配置 |
| 创建/改 Hyperdrive 配置 | Account → Hyperdrive → Edit / Hyperdrive Write | 指定目标账号；脚本固定实际 Hyperdrive ID |

以上账号级权限的 Account Resources 只选目标账号，不选 All accounts。脚本校验具体资源 ID 是额外防护，不等于 token 原生具备相同粒度；只有当前平台实际支持时才进一步收窄 token。只读操作采用对应 Read；产品开通、费用、数据库权限与 API token 权限是不同事情。Hyperdrive origin 数据库密码必须安全输入，不能写进示例、命令日志或聊天。

依据：[D1](https://developers.cloudflare.com/api/resources/d1/subresources/database/methods/create/) · [KV](https://developers.cloudflare.com/api/resources/kv/subresources/namespaces/methods/create/) · [R2](https://developers.cloudflare.com/api/resources/r2/subresources/buckets/methods/create/) · [Hyperdrive](https://developers.cloudflare.com/api/resources/hyperdrive/subresources/configs/methods/create/)

## 3. 用户取得并保存管理 token

1. 打开 [Cloudflare 官方 token 页面](https://dash.cloudflare.com/profile/api-tokens)，选择 Create Token → Custom Token；普通原生 Builds 管理使用 user token。
2. AI 先给用户列出本次需要的权限、账号/zone 范围和用途。用户按上表逐项选择；不用 Global API Key，不申请创建其他 token 的权限。
3. Account Resources 指定目标账号；Zone Resources 指定目标 zone。创建尚不存在的 zone 时，解释并批准临时覆盖该账号 zones 的范围，创建后按需要收窄。
4. 检查有效期、权限摘要及 IP 限制；不为 GitHub 托管 runner 猜一个固定 IP。用户本人确认创建并安全保存秘密。
5. 在已核实的用户运维仓库，打开 Settings → Secrets and variables → Actions → New repository secret，名称填写 `CLOUDFLARE_API_TOKEN`，由用户本人直接粘贴值。
6. 不在聊天索取/回传 token，不放普通 Variables，不复制到每个站点仓库。若要创建新 build token，让用户在官方安全界面完成，之后只核对它的名称/UUID/适用目标。

[Cloudflare 创建 token 步骤](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) · [GitHub Secrets 步骤](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)

## 4. 只读验证与失败分类

先读仓库当前验证 workflow。若存在 `Verify API credentials (read-only)`，按实际输入运行；不要假设可以只选一个平台。分 job 报告结果：

- `GET /user/tokens/verify` 返回 active：user token 有效；不是写权限验收。
- 读取指定 account/zone/Worker：核对目标身份、状态、资源范围。列表分页须读完整，空第一页不证明目标不存在。
- 原生 Builds：GET 目标 Worker 的 triggers、选中 build token 元数据、指定 build UUID 的状态；Worker API 里的 tag 才是 Builds 的 `external_script_id`，不要用 Worker 名代替。
- 只输出 HTTP 状态、允许的错误码、预先约定的目标 ID/状态及脱敏诊断；不打印原始 environment_variables、请求头或完整日志。

失败分类：缺 Secret；401/无效/过期 token；403 权限或目标范围待核对；404 ID/端点/可见性待核对；429 限流；5xx/超时/网络；成功但业务数据为空。单个 HTTP 码不能确定根因。先检查官方端点、token 类型、目标和安全错误摘要，再请求具体最小补充；不盲目换 token、扩权或试写。

## 5. 接上原生 Git 构建

获准后，在 Cloudflare Worker → Settings → Builds → Connect 选择 GitHub 和用户的站点仓库；必要时由用户完成 App 授权。核对项目实际根目录、build/deploy 命令、main 生产分支及已选 build token。默认不建预览分支 trigger。版本文件、依赖锁和自定义安装行为在站点仓库维护；中央运维仓库只配置/查询 API。

连接后检查是否已经自动构建，避免重复触发。验证真实 commit → build UUID → outcome → Worker version/deployment；再独立检查正式域。Cloudflare 构建成功不表示 DNS/证书/GSC 已完成。[Builds 配置](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/) · [构建镜像](https://developers.cloudflare.com/workers/ci-cd/builds/build-image/)
