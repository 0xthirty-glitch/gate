# VPNGate JP/KR 自动订阅个人实验

个人学习测试版。部署与验收步骤见 [DEPLOYMENT.md](./DEPLOYMENT.md)。请勿把静态打包或模拟检测当成真实客户端可用性证明。

## 数据流

1. GitHub Actions 每天 00:17 UTC 抓取 VPN Gate 公共节点，默认只考虑源列表中的 JP/KR 且带 TCP 入口的候选。
2. 受 Bearer 令牌保护的 Cloudflare Checker 逐个测试 SSTP，并返回出口信息。
3. 仅发布检测成功且 **Checker 报告的出口国家代码**为 JP/KR 的候选；缺少出口国家的节点不发布。
4. GitHub Pages 公布 nodes.txt，edgetunnel 从该 URL 生成链式订阅。客户端仍须实测真实出口。

入口地址由 GitHub Actions variable EDGETUNNEL_HOST 指向自己的 Cloudflare 域名；没有默认第三方优选 IP。网络类型仅凭出口服务返回的 hosting 标记或 ASN 名称推断，标记为“疑似住宅”“机房”或“未分类”。公共 VPN 不能保证纯净、独享、隐私或长期稳定。

## 必要配置

- Cloudflare：在本地工作区单独部署 Checker，使用 auth-worker.mjs 作为入口，配置 Secret CHECK_TOKEN 和检测域名；edgetunnel 绑定 KV 命名空间 KV，配置独立的 Secret ADMIN、自有域名，并设置 OFF_LOG=1。
- GitHub：将本地修改应用于小号的 Fork；在 Actions variables 中设置 CHECK_WORKER_URL（以 /check?sstp=vpn:vpn@ 结尾）及 EDGETUNNEL_HOST（host:443）；在 Actions secret 中设置同一个 CHECK_TOKEN。Settings → Pages 的 Source 选 GitHub Actions。
- 首次手动执行工作流，检查 Pages nodes.txt 非空；随后确认定时工作流已启用，观察下次运行结果。
- 本部署禁用 edgetunnel 的 /login 与 /admin 页面，直接在 KV 中将 config.json 的“随机IP”设为 false，并把 nodes.txt URL 写入 ADD.txt；否则默认逻辑不会使用自定义节点。

不要把 CHECK_TOKEN、ADMIN 或 Cloudflare 凭据写进公开仓库、Pages、日志或聊天。Checker 端点无令牌返回 401 是预期现象，不能靠浏览器直接打开 /check 来验收。

## 已知风险与验收

- 上游 edgetunnel 从第三方 edt-pages.github.io 获取登录和后台页面；本部署用包装入口阻止 /login 与 /admin 访问，避免通过该页面修改配置。不要把 Cloudflare API Key 或其他账号凭据输入任何第三方页面。
- VPN Gate 为志愿者公共中继，有运营方日志策略。不要用于支付、重要账号或敏感数据。
- 上游 Checker 的自实现 TLS 未严格校验目标站证书链与域名，志愿节点可能伪造出口查询响应；JP/KR 只能视为候选标签，不能当作可信的地区证明。实际客户端、目标网站和时间点也可能不同。至少验证一个日本和一个韩国节点的出口 IP、国家、ASN 与重复连接稳定性。
- 本次筛选结果为零时工作流失败，保留上一次 Pages 产物；订阅用户应关注页面生成时间和最近一次 Actions 结果。

上游参考：[VPN Gate](https://www.vpngate.net/)、[gate](https://github.com/hezhanleiok/gate)、[Checker](https://github.com/lsh8848/cm-Workers-CheckSocks5)、[edgetunnel](https://github.com/cmliu/edgetunnel)。
