# Spaceship API 与 NS/DNSSEC 切换

适用于用户自己的 `<GITHUB_OWNER>/<DEPLOY_REPO>`。域名仍注册在 Spaceship；迁移权威 DNS 不等于转移注册商。本文不表示凭据、NS 变更或 DNSSEC 操作已完成。

## 1. 用户创建凭据

1. 登录 [官方 API Manager](https://www.spaceship.com/application/api-manager/)，选择 New API key → Custom access。
2. 只读核对域名需要 `domains:read`；确实需要且已批准修改 NS 时才加 `domains:write`。
3. `domains:write` 能改域名设置，不是 NS-only 权限。默认不要启用 `domains:billing`、`domains:transfer`、联系人、SellerHub、Hyperlift 等无关权限。
4. API key/secret 由用户本人在安全界面保存；API secret 只显示一次，不发聊天、截图、仓库或日志。新建/扩展持续访问按当前平台的确认要求执行。

[官方 API 文档、scope 与 NS 端点](https://docs.spaceship.dev/)

## 2. 保存到 GitHub Secrets

在已核实的用户运维仓库打开 **Settings → Secrets and variables → Actions → New repository secret**，分别保存：

- `SPACESHIP_API_KEY`
- `SPACESHIP_API_SECRET`

两者必须来自同一组凭据。只有名称和脱敏结果进入文档；值不进普通 Variables，也不复制到网站源码仓库。Cloudflare token 不能代替 Spaceship 凭据。[GitHub 官方步骤](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)

## 3. 最小只读测试

查看当前 `Verify API credentials (read-only)` workflow 的真实实现，使用官方 `GET https://spaceship.dev/api/v1/domains?take=100&skip=0`，认证头为 `X-API-Key` 和 `X-API-Secret`。不输出这两个头或整个响应；报告请求结果、计数和是否还有分页。核对具体目标时继续分页或读取该域名，不把首屏没有目标判为不存在。

空列表可能是有效认证但无可见域名；保存 Secret 不等于跑过测试；只读成功不证明 NS 写权限。认证、scope、目标身份、限流及网络故障分别说明。不会为了验证权限而改 NS。

修改 NS 使用官方 Update domain nameservers 端点及 Cloudflare 实际分配的完整主机列表；不硬编码模板作者的 NS，不混入旧 NS，不用此接口购买、续费或转移域名。它成功只表示注册商接受变更，仍须验证父区委派及公共解析。

## 4. NS/DNSSEC 切换顺序

以下为普通迁移的保守操作方案。需要保持连续 DNSSEC 的 multi-signer 迁移是另一套高级流程，不能与本顺序混用。

1. **先备份和准备**：记录原 NS、完整 DNS、旧服务、DNSSEC 设置及回退依据；迁移邮件和验证记录，不能只依赖自动扫描。验证新 Worker/站点版本已部署，新 Cloudflare zone、记录和实际 NS 正确。先不要改父区 DS 或 NS。
2. **查父区 DS**：向父区权威服务器查询该域 DS，记录 key tag、algorithm、digest type、digest、原始 TTL、查询时间和服务器；检查多个权威响应是否一致。递归解析器里的剩余 TTL 不等于原始 TTL。未知/空响应要区分无记录、查询失败及尚未一致。
3. **确认安全变更**：向用户说明移除旧 DS 会暂时取消 DNSSEC 信任链，以及错误顺序可能造成 SERVFAIL、网站或邮件中断。取得本次移除/停用与后续恢复方案的相应确认。当前公开 Spaceship API 文档未列 DNSSEC/DS 管理端点；使用已获准的官方 UI，不虚构 REST API。
4. **移除旧 DS、保留旧服务**：用户或获准的官方流程在注册商处理 DS。尽量维持旧权威服务及旧签名，直到旧 DS 缓存过期；若提供商按钮同时停止签名，先解释它对仍缓存 DS 的验证解析器也可能有影响。不得仅凭开关显示 Off 就当作父区已更新。
5. **确认权威移除并等待**：复查父区权威，记录确认旧 DS 全部消失的时间。从该确认时间起至少等待记录的一个完整旧 DS TTL，再检查父区和多个公共验证解析器。采用更保守余量时明确说明。这是本仓库的安全等待方案，不是所有域统一“24 小时”；注册商 NS 更新时间与 DS 缓存等待是两个时钟。
6. **正常切换关卡**：仍有旧 DS、TTL 不明、权威不一致或 SERVFAIL 时停止变更并继续排查。关卡通过后，在批准范围内把 NS 改为新 zone 实际分配的列表。保留旧 DNS/站点，避免缓存命中旧 NS 时失效；不要反复来回切 NS。
7. **提前切换例外**：只有用户明确要求提前，且已说明旧 DS 缓存剩余窗口、可能影响的解析/邮件、回退不能即时清空缓存，并取得针对本域和此次安全变更的明确当次确认，才评估执行。至少重新确认父区不再发布旧 DS及新 DNS 完整；仍有 DS 或状态不清不能被笼统“直接切”覆盖。记录为风险接受的提前操作，不报告“DNSSEC 风险已解除”，并继续观察至原等待窗口结束及解析稳定。平台要求当次确认的安全设置不能靠本指南预批跳过。
8. **验证新权威稳定**：核对父区 NS、Cloudflare Active、多个公共解析器、邮件相关记录与可信 HTTPS；绑定/检查 Custom Domain、证书和跳转。NS API 200、zone Active 和单个 HTTP 200 各自只证明一部分。
9. **最后启用新 DNSSEC**：新权威与服务稳定、迁移缓存窗口结束后，按获准方案在 Cloudflare 启用 DNSSEC，读取该 zone 实际 DS。用户在注册商准确添加这组新 DS，核对 key tag、algorithm、digest type 和完整 digest；绝不复用旧 DS。随后验证父区新 DS、DNSKEY/RRSIG 与验证解析器无 SERVFAIL。没有完成核对就标记等待；不要把启用按钮成功当作全链路已签名验证。

回退也先看父区 DS、NS 与缓存状态。已经添加新 DS 时，不能直接退回不持有匹配密钥的旧 NS。涉及新的安全配置变更时重新按风险与授权处理。

依据：[Cloudflare Full setup](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/) · [旧 DS 与 NS 排错](https://developers.cloudflare.com/dns/zone-setups/full-setup/troubleshooting/) · [DNSSEC 概览](https://developers.cloudflare.com/dns/dnssec/) · [需要持续签名时的高级迁移](https://developers.cloudflare.com/dns/dnssec/dnssec-active-migration/)

## 5. 交付给用户

分别写明：凭据只读结果、目标域名、NS 提交结果、父区委派、DS/TTL 等待截止时间（含时区）、公共解析、旧服务保留情况、新 DNSSEC 是否恢复、HTTP/证书结果和仍需决定的事项。不要因某一项通过就写成域名已全部完成。
