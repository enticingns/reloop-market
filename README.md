# Reloop 伪装站 + 节点容器（Caddy 前门）

一个容器搞定：**sing-box（VLESS + WebSocket）藏在一个正经网站后面，前面再套 Cloudflare CDN**。

- 打开域名看到的是一个英文二手交易站 **Reloop**（首页 / 浏览 / 商品页 / 出售 / 关于 / 联系 / 隐私 / 条款 + 404），有 robots.txt、favicon、12 张商品插画，不是一张孤零零的模板页；
- 只有**同时满足「秘密路径」+「真实 WebSocket 升级」**的请求才会被转给 sing-box；
- 猜路径的扫描器、普通 GET、打错的链接，拿到的都是**和站点其它 404 完全一样**的页面，秘密路径无从分辨；
- 流量走 Cloudflare 边缘，客户端还可以填「优选 IP」进一步提速；
- UUID、路径、端口全部用环境变量配置，不写死在代码里。

> ⚠️ **开源免费声明**：本项目开源免费发布，**仅供学习、练习与参考，严禁任何商业使用**（包括但不限于收费部署、打包售卖、作为付费服务的一部分、商用引流）。详见 [LICENSE](LICENSE)。

## 工作原理

```
客户端 ──TLS──▶ Cloudflare（橙云代理）──▶ 容器 :3000（Caddy）
                                              ├─ 普通路径 / 无升级请求 → 伪装站（Reloop 静态站）
                                              ├─ 404 → 伪装站自己的 404 页
                                              └─ $WS_PATH + Upgrade: websocket
                                                      └─▶ 反代到 sing-box (127.0.0.1:10000)
                                                              VLESS + WS 出站直连
```

## 伪装加固（相对上游的改动）

伪装分两层：外面那层是「看起来像个正常网站」，里面那层是「探测不出来这里有个代理内核」。

**一、伪装站**

| 改动 | 说明 |
|---|---|
| 主题换成英文二手交易站 Reloop | 中文宠物救助站配境外 IP 太显眼；二手交易站的访问量、图片请求量都符合代理节点的真实流量特征 |
| 从单页扩成 8 个页面 | 首页、浏览（带筛选/排序/分页）、商品详情、出售、关于、联系、隐私、条款 |
| 站点级 404 | `site/404.html`，由 Caddy `handle_errors` 接管，不再暴露 Caddy 默认错误页 |
| 静态资源齐全 | `robots.txt`、`favicon.svg`、共享样式表、12 张商品插画（内联 SVG，无外链） |
| 取链接页重绘 | `/<UUID>` 页面改成站点内页风格，措辞中性（Saved link），不再出现 “Node access / Warm Paws” 这类一眼看穿的字样 |

**二、内核与反代**

| 改动 | 说明 |
|---|---|
| 路径 + WS 升级双重校验 | 上游只要 `GET /ray` 就会转发给 sing-box，sing-box 回 `400 Bad Request`，这个响应和伪装站的 404 不一致，是可被指纹识别的破绽。现在必须是 `Upgrade: websocket` + `Connection: *Upgrade*` 才转发 |
| 探测一律回落伪装页 | 不满足上面条件的请求走 `file_server` → 命中站点自己的 404，与随便打错一个路径的结果完全相同 |
| 统一错误页 | `handle_errors` 把 404 渲染成 `404.html`，其它状态码只回状态行，不泄露 Caddy 版本与反代信息 |
| 响应头收敛 | 去掉 `Server`、`X-Powered-By`，加 `nosniff` / `Referrer-Policy` |
| WS 缓冲参数 | `max_early_data: 2048` + `early_data_header_name: Sec-WebSocket-Protocol`，握手时把首包数据带在头部，少一次往返 |
| 日志降噪 | sing-box 日志级别从 `info` 降到 `warn`，正常流量不落盘 |
| 启动自检 | `entrypoint.sh` 先跑 `sing-box check`，配置错了立刻失败，不会出现「伪装站正常但节点不通」的迷惑状态 |
| 弱路径告警 | `WS_PATH` 短于 10 个字符时启动时打警告（短路径是唯一还能被爆破的东西） |
| 版本对齐 | sing-box `v1.14.4`、caddy `2.11.9`，均为当前最新稳定版（内核参数已按新版本写法给出） |

## 环境变量

| 变量 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `UUID` | ✅ | 无 | 节点 UUID，自己生成一个（`cat /proc/sys/kernel/random/uuid`） |
| `WS_PATH` | | `/ray` | 秘密路径，**务必改**成别人猜不到的，如 `/a8f3-media-stream-2` |
| `PORT` | | `3000` | 容器对外端口（平台要求多少就填多少） |

## 部署方式一：Docker 镜像（平台选「Deploy a Docker image」）

镜像用你自己的——本仓库自带 GitHub Actions 工作流，推送后自动打包到**你自己的** GHCR，全程不用装 Docker：

1. 打开仓库的 **Actions** 页（首次需要点一次启用）；
2. 选 **Build Docker image → Run workflow**（或随便推送一次提交），等它跑绿；
3. 镜像生成在 `ghcr.io/<你的用户名>/reloop-market:latest`。仓库是公开的，包默认也是公开的，部署平台可直接免密拉取（若把仓库改回私有，就要去 **Packages** 把这个包的可见性改成 **Public**）；
4. 在部署平台创建应用时填**你自己的**镜像地址，设置环境变量 `UUID`、`WS_PATH`、`PORT`。

（也可以本地 `docker build` 后推到任意公共仓库，效果一样。）

本地自检：

```bash
cp .env.example .env   # 填好 UUID
docker compose up -d
curl -I http://127.0.0.1:3000/          # 200，伪装站首页
curl -I http://127.0.0.1:3000/nope      # 404，但页面是伪装站自己的 404
```

## 部署方式二：上传源码（平台选 Upload / Compose）

把本仓库文件（Dockerfile、Caddyfile、entrypoint.sh、site/、sub-template.html、docker-compose.yml）打包上传，平台会按 Dockerfile 构建；同样设置好三个环境变量即可。

## Cloudflare 设置（关键）

1. 域名托管到 Cloudflare，给节点加一条 DNS 记录（如 `node.example.com`）指向平台给的公网地址，**开启橙云代理**。
2. SSL/TLS 模式：
   - 平台只给 HTTP 入口 → 用 **Flexible**（CF 到源站走 HTTP，先跑通）；
   - 平台给源站配了证书 → 用 **Full (Strict)**（更推荐）。
3. 确认所选套餐支持 WebSocket（免费版支持，无需额外设置）。

## 获取节点链接（推荐：取链接页面）

部署成功后，用浏览器打开（后缀就是你的 UUID，它本来就是节点的钥匙，不用再多记一个）：

```
https://<你的地址>/<你的 UUID>
```

页面会自动拼好完整的节点链接，点 **Copy link**，再到客户端里选「从剪贴板导入」即可。**不要把这个页面地址发给任何人、不要让它出现在截图和视频里**——打开它的人就能拿到完整链接。

手工拼链接（备用）：

```
vless://<UUID>@<你的域名>:443?encryption=none&security=tls&sni=<你的域名>&type=ws&host=<你的域名>&path=<WS_PATH 的 URL 编码>#Reloop-Node
```

例：WS_PATH 为 `/a8f3-media-stream-2` 时 path 填 `%2Fa8f3-media-stream-2`。

**优选加速**：把链接里的地址换成 Cloudflare 优选 IP（域名仍填在 sni/host 里），是这套方案常见的提速手法；稳定性因地区和运营商而异，建议与直连域名各存一条对比实测。

## 自检（伪装是否生效）

| 请求 | 期望结果 |
|---|---|
| `GET /` | 200，二手交易站首页 |
| `GET /browse/`、`/about/`、`/contact/` | 200，各自页面 |
| `GET /随便一个不存在的路径` | 404，页面是 `Reloop` 的「This page has moved on」 |
| `GET /<你的 WS_PATH>`（浏览器直接打开） | **同样是那个 404 页** —— 说明双重校验生效，秘密路径和错误路径无法区分 |
| `GET /<你的 UUID>` | 200，取链接页 |
| 客户端带正确 UUID 连接 | 握手成功，能正常上网 |

> 上游版本的这一步会返回 `400 Bad Request`（sing-box 直接应答），那正是被指纹识别的破绽，现在已被堵掉。

## 安全与边界

- `UUID` 和 `WS_PATH` 就是全部凭证，泄露等于节点公开；
- 伪装网页只是「看起来正常」，不提供额外的加密强度；真正的保密来自 TLS + UUID；
- 本仓库的加固只消除**应用层指纹**（错误页、响应头、路径探测），不改变协议特征，也不对抗主动探测流量分析；
- 请遵守部署平台的服务条款，本项目仅供个人学习与自用；
- 免费平台的流量/请求额度各不相同，注意控制台的用量提醒。

## 文件清单

| 文件 | 作用 |
|---|---|
| `Dockerfile` | 单容器镜像：Caddy（前门）+ sing-box（内核），版本已钉死 |
| `.github/workflows/` | 打包工作流：经 Actions 构建镜像并推到你自己的 GHCR |
| `Caddyfile` | 伪装站 + 路径/WS 双重校验 + 自定义错误页 |
| `entrypoint.sh` | 按环境变量生成并校验 sing-box 配置、渲染取链接页，拉起两个进程 |
| `sub-template.html` | 取链接页模板（运行时按 UUID/路径渲染到秘密后缀下） |
| `site/` | 伪装站：8 个页面 + 404 + robots.txt + favicon + 样式与商品插画 |
| `docker-compose.yml` | 本地/Compose 部署 |
