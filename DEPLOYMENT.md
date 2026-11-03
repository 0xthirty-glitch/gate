# VPNGate 个人实验版部署说明

部署目标：GitHub 小号 `0xthirty-glitch`、Cloudflare 小号和域名 `newupip.site`。本方案使用公共志愿 VPN 节点，“疑似住宅”只是分类推断，不保证出口质量或隐私。实际进度以 GitHub Actions、Cloudflare Worker 和客户端验收结果为准。

## 固定范围

- 仅筛选 JP/KR，GitHub Actions 每天 00:17 UTC 运行一次，也可手动触发。定时任务可能延迟。
- 检测 Worker 需要 Bearer 令牌；未配置 `CHECK_TOKEN` 时拒绝所有请求。
- 只使用自己绑定的 edgetunnel 域名作为 Cloudflare 入口，不默认引用第三方优选地址。
- 本次没有可用节点时任务失败，不发布空的 `nodes.txt`。
- 只有检测成功且 Checker 报告的出口国家代码为 JP/KR 的节点才发布；缺失出口国家也不发布。该报告不是独立可信的地区证明。

## 账号准备后配置

1. 贰玖购买新域名后，将其加入新 Cloudflare 小号并完成 DNS 生效；规划 `check.<domain>` 和 `sub.<domain>`。开始任何远程操作前，先核对当前登录的 GitHub 与 Cloudflare 均为小号，不使用已有大号会话。不要传递密码或 Token。
2. 在 Cloudflare 部署 `../checker`，把 `check.<domain>` 绑定到检测 Worker，并用 Secret 设置 `CHECK_TOKEN`。该 Worker 的入口是 `auth-worker.mjs`；不能只复制上游 `_worker.js`，否则没有鉴权。
3. 在 Cloudflare 部署 `../edgetunnel` 的 `safe-worker.mjs` 包装入口，绑定名为 `KV` 的 KV 命名空间，设置独立的 Secret `ADMIN` 和 `UUID`，并设置 `OFF_LOG=1`，再绑定 `sub.<domain>`。包装入口屏蔽 `/login` 和 `/admin`。
4. 在新 GitHub 小号 Fork `hezhanleiok/gate`，将本地 `gate` 改动应用到该 Fork。进入仓库 Settings → Pages，把 Source 设为 GitHub Actions。Fork 后在 Actions 页面启用工作流，并确认默认分支包含定时 YAML；公开仓库长期无活动时定时运行可能被自动禁用。
5. 在仓库 Actions variables 中设置 `CHECK_WORKER_URL=https://check.<domain>/check?sstp=vpn:vpn@`、`EDGETUNNEL_HOST=sub.<domain>:443`；在 Actions secret 中设置与检测 Worker 相同的 `CHECK_TOKEN`。不要把令牌放进变量、仓库代码或公开页面。
6. 手动运行一次 `VPN Gate Node Check`，核对 Actions 成功、Pages 上 `nodes.txt` 非空、每行含 `$sstp://`。
7. 直接在 edgetunnel KV 的 `config.json` 中设置 `优选订阅生成.本地IP库.随机IP=false`，再将 Pages 的 `nodes.txt` URL 写入 KV 键 `ADD.txt`。默认配置的随机 IP 为 `true`；不关闭时会跳过自定义节点。包装入口的 `/login` 和 `/admin` 返回 404 是预期行为。
8. 刷新 edgetunnel 订阅，确认生成节点的链式路径含 `/video/`，而不是只有随机 Cloudflare IP。客户端再连接至少一个日本、一个韩国节点；分别核对真实出口 IP、国家、ASN 和重复连接稳定性。只有这一步通过，才算实际可用。

## 边界与排障

- `CHECK_WORKER_URL` 是完整的检测接口前缀；它不是鉴权凭据。实际访问控制由 `CHECK_TOKEN` 完成。
- GitHub Pages 的 `nodes.txt` 是公开的；不要在其中放管理密码、令牌或私有代理。
- 已移除原项目的主机名推断；剩余 hosting 标记及 ASN 名称仍只能作为粗分类，`疑似住宅` 不代表目标网站认可的纯净住宅 IP。
- 上游 Checker 的自实现 TLS 仅检查目标证书非空，未严格校验证书链与域名；节点运营方可能伪造出口查询响应。自动发布的 JP/KR 标签只是待复核候选，不可据此认定真实出口地区。
- edgetunnel 上游代码会从 `edt-pages.github.io` 获取登录和后台页面；本部署使用包装入口屏蔽这些管理路径，配置仅经 Wrangler/KV 更新。不要在第三方页面输入 Cloudflare API Key、其他账号密码或 Token。
- 公共 VPN 节点由志愿者运营，不用于重要账号、支付或敏感数据。
- 如果 Checker 打包成功而在线连接失败，继续检查 Cloudflare Worker 运行限额、SSTP 握手、源站节点、KV 绑定、订阅生成和客户端出口；本地 dry-run 不能替代这些验证。
