[English](README.md) | **简体中文**

# Temp Share（临时分享）

把一个文件夹、一个 HTML 文件或一个 `.zip` 丢给它 → 得到一个公开链接。链接可以过期、可以加密码，也可以是**一次性**的（第一次访问后自动销毁）。

它只为解决一个需求：*"我现在就想把一个文件夹交给别人，但不想去搭建托管。"*

```
$ tshare ./hkdse-notes chemistry-rev --days 0
https://temp.example.com/chemistry-rev/
  1 files · 135672 bytes · never expires
```

---

## 功能

| | |
|---|---|
| **发布** | 一个目录、单个文件或 `.zip`（自动解压；zip 里只有一个顶层文件夹时会被展平） |
| **自定义名称** | `temp.example.com/<slug>/`，slug 由你决定 |
| **过期** | 默认 7 天，可设任意天数，`0` = 永不过期 |
| **一次性链接** | 只提供一次，然后从磁盘删除；之后访问返回 404 |
| **密码** | 每个链接可选密码（`sha256` + 每个分享独立的盐） |
| **访客记录** | 每个分享记录：IP、粗略的设备类别、路径、时间、独立 IP 数 |
| **静态托管规则** | 分享中只有一个 `.html` 时，在 `/<slug>/` 直接提供；没有 `index.html` 的文件夹会生成文件列表 |
| **自动清理** | 每 15 秒一次的清理器会删除已过期和已销毁的分享及其文件 |
| **两个入口** | 控制面板（密码 + 可选 TOTP 双重验证）和 `tshare` 命令行 |
| **一键下载** | 任何分享都可以直接在面板里打包成 zip 下载 |
| **管理设置** | 修改密码、开启 / 关闭双重验证、在所有设备退出登录 |
| **自带字体** | Lato + JetBrains Mono 放在 `static/fonts/`，不用 CDN，离线可用 |

## 界面设计系统（"Flat Design Corporativo"）

面板、登录页和公开的分享页使用同一套扁平化企业风格。所有样式都在 `static/style.css` 里：要改就在那里改，不要写内联样式。

| 变量 | 值 |
|---|---|
| 主色 / 强调色 | 企业蓝 `#007BFF` |
| 表面色（深色外框） | 深灰 `#343A40` |
| 页面背景 | 浅灰 `#F8F9FA` |
| 语义色 | 绿 `#28A745` · 琥珀 `#FFC107` · 红 `#DC3545` · 青 `#17A2B8` |
| 圆角 | 处处 `4px`（有意使用锐利的边角） |
| 字体 | Lato 400/700/900，元数据用 JetBrains Mono |
| 层级阴影 | 扁平；不超过 `0 2px 8px rgba(0,0,0,.08)` |
| 动效 | 只用 transform + opacity；遵守 `prefers-reduced-motion` |
| z-index | 导航 `100` · 遮罩 `200` · 弹窗 `300` · 提示 `500` |

规则：只用纯色（不用装饰性渐变），界面中不用 emoji，只用内联 SVG 图标，不用纯黑，不做三等分的功能行。基础变量（CSS 变量）只在 `:root` 中声明一次，其他地方都引用它们。

## 快速开始（本地，60 秒）

```bash
git clone <repo> && cd Temp_Share
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
PORT=19011 python server.py
# → http://127.0.0.1:19011
```

用密码 **`12345678`** 登录，然后立刻到**设置**里修改。面板在 `./.admin-password`（权限 600）里保存的是 `scrypt` 哈希，而不是密码本身。另有 `./.api-token` 供 `tshare` 命令行和自动化使用。

## 发布内容

```bash
./tshare ./my-site                       # 文件夹  → slug "my-site"，7 天
./tshare index.html demo                 # 单个文件 → slug "demo"
./tshare ./bundle.zip site   --days 3    # zip 会被解压
./tshare ./secret  drop     --days 0 --once --pw hunter2
./tshare --list                          # 当前有哪些在线，谁打开过
./tshare --del drop                      # 立刻删除
```

`tshare` 只是 HTTP API 的一层薄封装，所以它能做的事，你也可以用 `curl` 做到：

```bash
TOKEN=$(cat .api-token)
curl -X POST localhost:19011/api/shares \
  -H "X-Token: $TOKEN" \
  -F slug=demo -F days=7 -F local_path=/abs/path/to/folder
```

## 工作原理

```
            https://temp.example.com/<slug>/…
                        │
                  ┌─────▼──────┐
                  │   nginx    │  TLS · 请求体限制 · X-Forwarded-For
                  └─────┬──────┘
                        │ 127.0.0.1:19011
                  ┌─────▼──────┐
                  │  FastAPI   │  认证 · 发布 · 提供内容 · 过期
                  └─────┬──────┘
                        │
        data/shares.json │ storage/<slug>/…
```

应用只绑定到本机回环地址。nginx 负责终止 TLS 并转发访客 IP，这正是每个分享的访客记录能够实现的原因。除了通过 nginx，没有任何东西对外暴露。

### 一次性链接的语义

这是最微妙的部分。如果"一次性"链接在*第一个字节*之后就失效，所有需要自己加载 CSS、JS 或图片的页面都会坏掉。所以：

| 步骤 | 结果 |
|---|---|
| 第一次导航请求 | 正常提供；`burn_at = 现在 + 30 秒` |
| 该页面的子资源 | 在宽限期内正常提供 |
| 清理器（每 15 秒） | `burn_at` 已过 → 删除文件和元数据 |
| 之后的任何访问 | `404` |

只有**导航**请求才会消耗链接：先检查 `Sec-Fetch-Dest`，再退回检查 `Accept`；任何无法识别的请求（curl、机器人、API 客户端）都当作一次访问，宁可销毁也不会让文件永远泄露。

### 如实说明过期

过期由清理器执行，而不是文件系统的 TTL，所以如果没人访问，一个分享最多可能比截止时间多存在一个清理周期（15 秒）。这是有意的设计：清理失败不会让服务崩溃，而且每次读取时都会检查截止时间（`is_expired` / `is_burned`），所以过期的链接即使在清理器运行之前也不会提供内容。

## HTTP API

管理员接口需要 `X-Token: <token>` 或面板的 cookie。

| 方法 | 路径 | 用途 |
|---|---|---|
| `GET` | `/healthz` | 存活状态 + `shares` + `sweeper` 状态（无需认证） |
| `POST` | `/api/login` | 表单 `password` + 可选的 `code`（TOTP）；设置会话 cookie |
| `POST` | `/api/logout` | 结束当前会话 |
| `GET` | `/api/session` | 这个浏览器是否已登录？是否开启双重验证？ |
| `POST` | `/api/password` | 表单 `current`、`new`（至少 8 个字符） |
| `POST` | `/api/2fa/setup` | 生成 TOTP 密钥（`otpauth://` 网址） |
| `POST` | `/api/2fa/enable` | 表单 `secret`、`code`：验证通过后开启双重验证 |
| `POST` | `/api/2fa/disable` | 关闭双重验证 |
| `GET` | `/api/shares/{slug}/download` | 把分享（或单个文件）打包成 zip |
| `GET` | `/api/shares` | 列出分享、访客和已用存储 |
| `POST` | `/api/shares` | 发布（`slug?`、`title?`、`days`、`password?`、`once?`，以及 `local_path?` **或** `file`） |
| `DELETE` | `/api/shares/{slug}` | 删除分享**及其文件** |
| `POST` | `/api/shares/{slug}/extend` | 表单 `days`；延长截止时间 |
| `POST` | `/api/sweep` | 立刻运行清理 |
| `GET` | `/{slug}/{path}` | 公开：提供分享内容（设置了密码则需要密码） |

`POST /api/shares` 接受 `local_path`（服务器端路径，供命令行使用）或 multipart 的 `file` 上传（供面板使用）。`local_path` 是经过认证的，不是远程输入。

## 安全

- **路径穿越**已被阻止：解析后的目标必须位于 `storage/<slug>/` 内。
- **Zip-slip**已被阻止：解压前会解析并检查每个压缩包成员。
- **密码**使用 `sha256(盐 + ":" + 密码)`，每个分享有独立的随机盐，不会明文保存。
- **不安全的 slug** 会被拒绝（`^[a-z0-9][a-z0-9._-]{0,63}$`），重复的返回 `409`。
- **管理员密码**以 `scrypt(密码, 每次安装独立的盐)` 保存，不可逆，也不会写进日志。
- **会话**是保存在 `data/sessions.json` 中的随机 32 字节 ID，有效期 30 天，通过 HttpOnly cookie 发送；修改密码或在所有设备退出登录都会使其失效。
- **双重验证（可选）**是符合标准的 TOTP（RFC 6238，SHA-1，6 位数字，30 秒），已用 RFC 测试向量验证，并容忍一个步长的时钟偏差。只用标准库，没有额外依赖。
- `tshare` 命令行用 `.api-token`（权限 600）认证，所以自动化永远不需要密码。
- 上传有大小上限（`MAX_UPLOAD_MB`，默认 800）。

已知限制，直说：密码接口没有限流，没有按分享统计带宽，访客记录信任 `X-Forwarded-For`（在你自己的 nginx 后面没问题，在不可信的代理后面就不对）。文件分享网站也是滥用的重灾区：如果将来要把一个开放的发布入口暴露到公网，请*先*加上发布认证和滥用监控。

## 配置

| 环境变量 | 默认值 | 含义 |
|---|---|---|
| `PORT` | `19011` | 监听端口（回环地址） |
| `MAX_UPLOAD_MB` | `800` | 单次上传上限 |
| `ONCE_GRACE_SECONDS` | `30` | 一次性链接继续提供资源的时长 |

## 测试

```bash
pip install -r requirements-dev.txt
pytest -q
```

43 个测试，不需要网络。覆盖存储辅助函数、过期 / 销毁逻辑、一次性语义、导航与资源请求的分类、密码哈希与修改流程、TOTP（对照 RFC 测试向量）、路径穿越防护、zip-slip、下载和访客记录。

## 部署（systemd + nginx）

```bash
sudo cp deploy/temp-share.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now temp-share

sudo cp deploy/nginx.conf /etc/nginx/sites-available/temp-share
sudo ln -s /etc/nginx/sites-available/temp-share /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

把 DNS 的 `A` 记录指向这台机器，并把证书路径填进 nginx 文件。两个模板都在 [`deploy/`](deploy/) 里。

## 写进代码里的教训

1. **单文件分享必须在 `/<slug>/` 直接打开文档。** 第一版显示的是只有一项的文件列表，对一个"把文件丢给它就行"的工具来说，这是失败。
2. **后台任务需要强引用和心跳。** 清理器原来是 `asyncio.create_task(loop())`，没有保留引用；asyncio 只保留弱引用，所以它可能被垃圾回收，什么都不会过期，而 `/healthz` 还显示"ok"。现在它放在模块全局变量中，每次清理都会记录日志，并在 `/healthz` 里报告自己的状态。

## 许可证

MIT，见 [LICENSE](LICENSE)。
