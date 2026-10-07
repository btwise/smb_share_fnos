# 远程共享（Remote Share Manager）

> 飞牛 fnOS 的 SMB / CIFS 共享管理应用 —— 补全 fnOS 缺失的传统「网络共享」控制台。
>
> 作者：**草原企鹅** ｜ 版本：**1.0.1** ｜ 应用标识：`sharemanager`

---

## 目录

- [1. 这是什么](#1-这是什么)
- [2. 功能特性](#2-功能特性)
- [3. 运行环境](#3-运行环境)
- [4. 目录结构](#4-目录结构)
- [5. 架构设计](#5-架构设计)
- [6. 后端实现要点](#6-后端实现要点)
- [7. HTTP API 文档](#7-http-api-文档)
- [8. 前端实现要点](#8-前端实现要点)
- [9. 打包方式](#9-打包方式)
- [10. 安装 / 卸载 / 升级](#10-安装--卸载--升级)
- [11. 本地开发与调试](#11-本地开发与调试)
- [12. Samba 配置约定](#12-samba-配置约定)
- [13. 常见问题（踩坑记录）](#13-常见问题踩坑记录)
- [14. 版本历史](#14-版本历史)

---

## 1. 这是什么

飞牛 fnOS 把 SMB 共享做成了「按用户自动分配」的隐式逻辑共享（配置动态生成在 `users/*.share.conf`），
**没有**传统 NAS 那种「共享列表 / 权限 / 匿名访问 / 只读」的可视化管理界面。

本应用在系统之上补一个独立的共享控制台：

- 为**任意目录**手动创建 SMB 共享；
- 把**已有的手工共享**（写在 `smb.custom.conf` 等文件里的段）纳管、可视化编辑；
- 修改后 **testparm 校验 + 热重载 + 主动断开旧连接**，不中断其它共享的现有会话；
- 顺带展示 fnOS 各文件服务协议（SMB / NFS / WebDAV / AFP / FTP / SFTP / DLNA / Rsync）的运行状态。

设计原则：**只做加法，不侵入系统**。应用自身维护一个独立的 Samba 配置文件
`/etc/samba/smb.sharemanager.conf`，通过一行 `include` 挂进主配置，卸载时移除该行即回到原生状态。

---

## 2. 功能特性

### 共享目录

| 功能 | 说明 |
|---|---|
| 创建共享 | 指定名称、路径、备注，配置访问方式 |
| 编辑共享 | 支持**重命名**（原位置原子替换，不留中间态）、改路径、改权限 |
| 删除共享 | 应用创建的删除；手工段就地删除；**飞牛逻辑共享禁止删除** |
| 匿名访问 | 允许免账号访问（`guest ok = yes`，自动以 `nobody` 身份运行） |
| 只读模式 | `read only` 控制只读 / 可写 |
| 可浏览开关 | `browseable` 控制是否在网络邻居中出现 |
| 用户授权 | 从系统账户中多选，写入 `valid users` |
| 强制访问身份 | `force user` 指定一律以某系统用户身份访问 |
| 目录权限自动修正 | 勾选后对目标目录执行 `chmod -R`（含子目录），确保真正可访问 |
| 来源标记 | 每行标注 `应用创建` / `手工创建` / `飞牛管理`，并据此决定能否编辑 |
| 概览统计 | 表格下方 6 格统计：总数 / 匿名 / 只读 / 应用 / 手动 / 飞牛 |

### 用户管理

- 列出 **Samba 账户**（`pdbedit -L`）与**系统账户**两部分；
- 新增 Samba 用户（需先存在于系统账户，自动设置 Samba 密码）；
- 删除 Samba 账户（不会删除系统账户）；
- 概览统计：Samba 账户数 / 可登录共享用户数 / 系统用户数。

### 服务状态

- 8 种文件服务协议的运行状态（横向卡片网格展示）：图标 + 状态点 + 端口 + 描述；
- 检测逻辑双信号：`ss -lnt` 端口监听 **或** `systemctl is-active` unit 活跃，任一命中即「运行中」；
- 同时展示 SMB 服务状态、Samba 版本、配置校验结果、include 挂载状态；
- **只读展示，不做启停控制**（这些服务由 fnOS 系统统一管理）。

### 目录选择器

- 默认打开飞牛「我的文件」（`/vol1/<当前用户uid>`），不暴露 Linux 文件系统；
- 顶部快捷定位：🏠 我的文件 / 👥 团队文件 / 💾 全部文件；
- 面包屑使用飞牛式友好名称（`我的文件 / Movies / …`），切到根目录才显示真实路径。

---

## 3. 运行环境

| 项目 | 要求 |
|---|---|
| 系统 | 飞牛 fnOS（Debian 12 bookworm 底子），`os_min_version = 1.2.0` |
| 架构 | `platform = all`（实测 x86_64） |
| 运行时 | 系统自带 `/usr/bin/python3`（实测 3.11.2） |
| 第三方依赖 | **无**。后端仅用 Python 标准库 |
| 依赖的服务 | `smbd`（Samba 4.x）、`smbcontrol`、`testparm`、`pdbedit`、`ss`、`systemctl` |
| 权限 | 应用以 `root` 运行（见 `config/privilege`），API 侧二次校验调用者属于 `Administrators` 组 |

前端为原生 HTML/CSS/JavaScript，**不使用任何框架、构建工具**，直接由后端静态托管。

---

## 4. 目录结构

```
sharemanager/
├── README.md                    # 本文件
├── .gitignore                   # 版本控制忽略规则
├── ICON.PNG                     # 应用图标 256×256（应用中心读这个）
├── ICON_256.PNG                 # 同上（fnpack 规范要求的副本）
├── manifest                     # 应用元数据（名称/版本/作者/发布者/依赖…）
├── app/
│   ├── ui/
│   │   ├── config               # 桌面入口定义（标题、图标路径、iframe 网关参数）
│   │   └── images/
│   │       ├── icon_256.png     # 桌面/应用中心图标 256px
│   │       ├── icon_64.png      # 小图标 64px
│   │       ├── icon_v3_256.png  # 带版本后缀（用于刷新图标缓存，见 §13）
│   │       └── icon_v3_64.png
│   └── server/
│       ├── server.py            # 后端全部逻辑（约 885 行，Python 标准库）
│       └── public/              # 静态前端
│           ├── index.html       # 页面骨架（三个 TAB + 两个模态窗）
│           ├── style.css        # 全部样式
│           └── app.js           # 前端逻辑
├── config/
│   ├── privilege                # 运行身份：root / 用户名 sharemanager
│   └── resource                 # 资源限制（空对象 = 默认）
├── cmd/                         # 生命周期脚本（由 appcenter 调用）
│   ├── main                     # start / stop / status（进程与 pid 管理）
│   ├── install_init             # 安装前：备份 smb.conf
│   ├── install_callback         # 安装后：创建托管配置 + 挂 include 行 + 热重载
│   ├── uninstall_init           # 卸载前：备份托管配置（保留用户数据）
│   ├── uninstall_callback       # 卸载后：摘掉 include 行 + 热重载
│   ├── upgrade_init             # 升级前（空）
│   ├── upgrade_callback         # 升级后：与 install_callback 相同的自愈逻辑
│   ├── config_init              # 配置前（空）
│   └── config_callback          # 配置后（空）
├── wizard/
│   └── README.txt               # 向导目录占位（本应用未使用）
└── sharemanager.fpk             # 打包产物（安装包）
```

> `sharemanager.fpk` 实际是 **tar 压缩包**，可以直接 `tar -tf` 查看内容。

---

## 5. 架构设计

```
┌──────────────────────────────────────────────────────────┐
│  飞牛桌面（浏览器）                                        │
│  ┌────────────────────────────────────────────────────┐  │
│  │ iframe: /app/sharemanager                          │  │
│  │   index.html + app.js + style.css                  │  │
│  └───────────────┬────────────────────────────────────┘  │
└──────────────────┼───────────────────────────────────────┘
                   │ HTTP（统一网关反向代理）
┌──────────────────▼───────────────────────────────────────┐
│  fnOS 统一网关  /app/sharemanager/  →  unix socket        │
│  注入请求头 X-Trim-Userid（当前登录用户 uid）              │
└──────────────────┬───────────────────────────────────────┘
                   │ /vol1/@appcenter/sharemanager/app.sock
┌──────────────────▼───────────────────────────────────────┐
│  server.py（root 运行，仅标准库）                          │
│   ├── 静态文件服务  public/                               │
│   ├── /api/*  鉴权：X-Trim-Userid ∈ Administrators        │
│   └── Samba 操作层                                        │
│        ├── 读：解析 smb.conf + include（不跑 testparm）    │
│        ├── 写：文本级段落替换 → testparm 校验 → 失败回滚    │
│        ├── 生效：smbcontrol reload-config（热重载）        │
│        └── 断连：smbcontrol close-share <名字>            │
└──────────────────────────────────────────────────────────┘
                   │
      /etc/samba/smb.conf ── include ──▶ smb.sharemanager.conf
```

**关键设计点**

1. **Unix Domain Socket + 统一网关**：应用不监听 TCP 端口，只监听 `app.sock`，
   由 fnOS 网关负责鉴权入口与路径前缀 `/app/sharemanager`。
2. **应用纯净性**：共享配置独立成文件，主配置只多一行 `include`，卸载可完整还原。
3. **不跑慢命令**：读路径直接解析配置文件（`testparm` 首次要 2~5 秒），只在写入时校验、
   状态页带 120 秒缓存。
4. **写操作原子性**：先写临时文件、`fsync`、`os.replace` 原子替换，再跑 `testparm`，
   校验失败自动回滚原文；进程内 `WRITE_LOCK` 串行化写操作。

---

## 6. 后端实现要点

文件：`app/server/server.py`

### 环境变量

| 变量 | 默认值 | 作用 |
|---|---|---|
| `SOCKET_PATH` | 空 | 设置则监听 Unix socket（生产），为空则监听 TCP |
| `PORT` | `5955` | 无 socket 时监听的本地端口（开发用） |
| `GATEWAY_PREFIX` | `/app/sharemanager` | 网关注入的 URL 前缀，路由时剥离 |
| `PUBLIC_DIR` | 脚本同级 `public/` | 静态资源目录 |
| `TRIM_DEV` | 未设 | 设为 `1` 时放宽鉴权，方便本地调试 |

### 关键常量

```python
SMB_CONF      = "/etc/samba/smb.conf"              # 主配置
OUR_CONF      = "/etc/samba/smb.sharemanager.conf" # 应用托管配置
INCLUDE_LINE  = "include = /etc/samba/smb.sharemanager.conf"
ADMIN_GROUP   = "Administrators"                   # 允许操作的用户组
```

### 核心函数一览

| 函数 | 职责 |
|---|---|
| `run(cmd, timeout)` | 统一执行外部命令，返回 `(rc, stdout, stderr)`，自带超时 |
| `is_admin(username)` / `require_admin(handler)` | 鉴权：从 `X-Trim-Userid` 取 uid → 用户名 → 检查组，结果带缓存 |
| `parse_conf(path)` / `serialize_conf()` | 解析 / 序列化 `ini` 风格 Samba 配置（保留段落顺序） |
| `parse_all_shares()` | 解析主配置 + 所有 `include` 文件，返回 `{段名: 内容}` 与**每个段的来源文件** |
| `share_to_json()` | 段 → JSON，并标记 `origin` = `app` / `custom` / `fnos` |
| `valid_share_name()` / `valid_path()` | 名称与路径校验（禁 `\ / : * ? " < > \| [ ]`；路径禁 `..`、禁关键系统目录） |
| `_rewrite_sections_text()` / `rewrite_conf_file_multi()` | 文本级段落替换（保留注释与空行），支持一次改多段（重命名场景） |
| `rewrite_conf_file()` | 单段写入 + `testparm` 校验 + 失败回滚 + 热重载 |
| `commit_sections()` | 提交托管文件，同上校验与回滚 |
| `reload_samba()` | `smbcontrol all reload-config`，失败降级到 `systemctl reload smbd` / `killall -HUP smbd` |
| `close_share_connections(names)` | 对每个共享名执行 `smbcontrol all close-share <名>`，踢掉旧客户端会话 |
| `list_all_testparm()` | `testparm` 结果带 120 秒缓存（首次慢，之后瞬时） |
| `get_file_services()` | 端口监听 ∪ unit 活跃 → 8 种协议运行状态 |
| `list_system_users()` / `list_samba_users()` | 系统账户 / `pdbedit -L` Samba 账户 |

### 权限模型（写入 Samba 段时）

```
guest ok = yes（匿名）
    ├── force user  = nobody
    ├── force group = nogroup
    ├── create mask    = 0777
    └── directory mask = 0777
    + 目录权限修正：chmod -R o+rwX

指定 force_user
    └── force user = <该用户>
    + 目录权限修正：chmod -R a+rwX

valid users = 用户1, 用户2      # 勾选的授权用户
read only  = yes|no            # 只读开关
browseable = yes|no            # 可浏览开关
hide unreadable = yes
hide special files = yes
```

> 匿名共享之所以强制 `nobody`，是为了对齐 fnOS 自带「公共区」的做法：
> 匿名客户端映射到 `nobody`，配合 `o+rwX` 目录权限即可正常读写，避免因目录属主导致的拒绝访问。

### 来源判定与可编辑性

| 来源 | 判定条件 | 可编辑 | 可删除 |
|---|---|---|---|
| `app` | 段位于 `smb.sharemanager.conf` | ✅ | ✅ |
| `custom` | 段位于其它非 `users/` 文件（如 `smb.custom.conf`） | ✅（就地写回原文件） | ✅ |
| `fnos` | 段路径含 `/users/`（fnOS 动态生成的逻辑共享） | ❌ 403 | ❌ 403 |

---

## 7. HTTP API 文档

所有 API 路径**不带**网关前缀时也可用（后端会自动剥离 `GATEWAY_PREFIX`）。
所有 `/api/*` 请求都需要请求头 `X-Trim-Userid: <管理员 uid>`（由网关注入），否则返回 `403`。

### `GET /api/status`

SMB 服务与配置状态。

```json
{
  "smbd_active": true,
  "samba_version": "4.22.x",
  "config_valid": true,
  "config_error": "",
  "include_attached": true,
  "managed_file": "/etc/samba/smb.sharemanager.conf"
}
```

### `GET /api/services`

各文件服务协议运行状态。

```json
{
  "services": [
    {"key": "smb", "name": "SMB/CIFS", "desc": "…", "running": true,
     "ports": ["445"], "open_ports": ["445"], "unit": "smbd"}
  ]
}
```

### `GET /api/shares`

共享列表（含来源标记）。

```json
{
  "shares": [
    {"name": "公共区", "path": "/vol1/1000/公共文件", "comment": "",
     "browseable": true, "read_only": false, "guest_ok": true,
     "valid_users": [], "force_user": "nobody",
     "origin": "custom", "editable": true}
  ]
}
```

> `origin` ∈ `app` / `custom` / `fnos`；`editable` 为 `false` 时前端只读展示（飞牛逻辑共享）。
> 列表会过滤掉 `global`、`homes`、`printers`、`print$`。

### `POST /api/shares`

创建或更新共享（`original_name` 存在即编辑，且可用于重命名）。

```json
{
  "name": "Documents",
  "original_name": "Docs",       // 可选：编辑时传原名称，与 name 不同则重命名
  "path": "/vol1/1000/Documents",
  "comment": "文档目录",
  "guest_ok": false,
  "read_only": false,
  "browseable": true,
  "valid_users": ["admin"],
  "force_user": "",
  "fix_perm": true
}
```

响应：

```json
{"ok": true, "warning": "目录权限调整失败: …"}   // warning 可选
```

状态码：`200` 成功 ｜ `400` 参数/校验失败 ｜ `403` 目标是飞牛逻辑共享。

### `DELETE /api/shares/<名称>`

删除指定共享。`403` 表示该共享由 fnOS 逻辑管理，`404` 表示不存在。

### `GET /api/users`

```json
{
  "samba":  [{"username": "admin", "uid": 1000}],
  "system": [{"username": "admin", "uid": 1000}]
}
```

### `POST /api/users`

新增 Samba 账户：`{"username": "xxx", "password": "***"}`
用户名规则：`^[a-z_][a-z0-9_-]{0,31}$`（小写字母/下划线开头），且须已存在于系统账户。

### `DELETE /api/users/<用户名>`

删除 Samba 账户（不影响系统账户）。`root` / `admin` / `nobody` 等关键用户会被拒绝（400）。

### `GET /api/browse?path=<绝对路径>`

目录浏览。`path` 省略时默认返回当前登录用户的「我的文件」目录。

```json
{
  "path": "/vol1/1000",
  "display": "我的文件",
  "parent": "/vol1",
  "home": "/vol1/1000",
  "roots": [
    {"key": "home", "label": "我的文件", "path": "/vol1/1000", "icon": "🏠"},
    {"key": "team", "label": "团队文件", "path": "/vol1/@team", "icon": "👥"},
    {"key": "all",  "label": "全部文件", "path": "/",         "icon": "💾"}
  ],
  "dirs": [{"name": "Photos", "path": "/vol1/1000/Photos"}]
}
```

> `path` 始终是真实绝对路径（写进 Samba 配置的就是它）；`display` 是给前端显示用的友好面包屑文本；
> `roots` 中「团队文件」仅在 `/vol1/@team` 存在时返回；默认目录取请求头 `X-Trim-Userid` 对应的 `/vol1/<uid>`，
> 若该目录不存在则回退到 `/vol1`。

---

## 8. 前端实现要点

文件：`app/server/public/`

| 文件 | 说明 |
|---|---|
| `index.html` | 骨架：顶部标题栏 + 三个 TAB（共享目录 / 用户管理 / 服务状态）+ 共享弹窗（三标签）+ 目录选择弹窗 |
| `style.css` | 全部样式，使用 CSS 变量做主题色；响应式卡片网格 |
| `app.js` | 全部逻辑，原生 JS（ES5 风格、无构建步骤） |

**两个必须注意的约定**

1. **`<base href="/app/sharemanager/">` + 绝对路径**
   fnOS 网关可能以**不带尾斜杠**的 URL 打开 iframe（`/app/sharemanager`），
   此时页面内所有相对路径都会解析错层级 → 404（表现为「无样式、点击无反应」）。
   因此 `<head>` 里固定了 `<base>`，前端所有 API 调用统一走：

   ```js
   var API_BASE = "/app/sharemanager/";
   api("POST", "api/shares", payload)   // 内部拼成 /app/sharemanager/api/shares
   ```

   后端的 SPA fallback 也做了限制：**只对无扩展名的路径**回退到 `index.html`，
   避免把 `.js` / `.css` 的 404 也吞成 HTML。

2. **页面填满窗口**
   应用嵌在桌面 iframe 里，外层窗口高度由系统控制。页面用 `min-height:100%` + flex 全高自适应，
   另外通过 `postMessage` 上报内容高度（系统支持则自动收缩窗口，不支持则被忽略，无副作用）。
   内容较少时用统计卡片填充版面。

---

## 9. 打包方式

### 工具

使用飞牛官方打包工具 **`fnpack`**（项目根目录的 `fnpack.exe`，Windows 版）。

```bash
cd sharemanager
fnpack build            # Windows:  MSYS_NO_PATHCONV=1 ../fnpack.exe build
```

产物：`sharemanager/sharemanager.fpk`

### 打包前检查清单

1. **manifest 字段**（`version` / `display_name` / `desc` / `maintainer` / `distributor` 等）
2. **版本号 +1**（例如 `1.0.0` → `1.0.1`）
3. **图标四件套同步替换**
   - `ICON.PNG`、`ICON_256.PNG`（256×256，应用中心读这两个）
   - `app/ui/images/icon_256.png`、`icon_64.png`（桌面/详情页）
   - 若需强制刷新客户端图标缓存，同时更新带后缀的副本并修改 `app/ui/config`（见 §13）
4. **语法自检**（无构建步骤，直接编译检查）

```bash
python -m py_compile app/server/server.py     # 后端
node --check app/server/public/app.js         # 前端
rm -rf app/server/__pycache__                 # 别把 .pyc 打进包
```

### manifest 字段说明

| 字段 | 含义 |
|---|---|
| `appname` | **应用唯一标识**，决定网关路径 `/app/<appname>` 与数据目录名。**发布后不要改** |
| `version` | 版本号，用于应用中心比较新旧 |
| `display_name` | 应用中心与桌面显示的名称 |
| `desc` | 应用介绍 |
| `maintainer` / `maintainer_url` | 开发者（应用中心「开发者」列读 `maintainer`） |
| `distributor` / `distributor_url` | 发布者（应用中心「发布者」列读 `distributor`；不填会显示 `-`） |
| `source` | 来源：`thirdparty` / `official` |
| `platform` | 架构：`all` / `x86` / `arm` |
| `desktop_uidir` | 桌面入口配置目录（本项目为 `ui`） |
| `desktop_applaunchname` | 启动项名，需与 `ui/config` 中的键一致（`sharemanager.main`） |
| `ctl_stop` | 停止应用时是否调用 `cmd/main stop` |
| `checkport` | 是否做端口冲突检查（本应用走 socket，设 `false`） |
| `disable_authorization_path` | 关闭授权路径校验 |
| `os_min_version` | 最低系统版本 |
| `changelog` | 更新日志 |

### 图标设计规范（与 fnOS 原生保持一致）

从系统自带图标（`/usr/trim/www/static/app/icons/<app>/icon.png`）采样得到：

- 底色：垂直渐变 `#4778FF → #3D5DFF`
- 形状：圆角约 25% 的**超椭圆（squircle）**轮廓，非普通圆角矩形
- 图形：纯白，占画布高度约 68%，可带极淡投影 / 高光

---

## 10. 安装 / 卸载 / 升级

### 在飞牛上安装

**方式一（推荐）**：应用中心 → 设置 → **手动安装** → 选择 `.fpk` 文件。

**方式二（命令行）**：

```bash
sudo appcenter-cli install-fpk /path/to/sharemanager.fpk --volume 1
sudo appcenter-cli start sharemanager
sudo appcenter-cli status sharemanager
```

> `--volume 1` 指定安装到「存储空间 1」。

### 卸载

```bash
sudo appcenter-cli uninstall sharemanager
```

卸载会：

1. 把托管配置备份为 `/etc/samba/smb.sharemanager.conf.uninstalled.<时间戳>.bak`；
2. 从 `smb.conf` 中摘掉 `include` 行；
3. 热重载 Samba，系统回到原生状态。

**已有共享不会丢**：托管文件本身与 `smb.custom.conf` 里的手工共享都保留，重装后自动重新挂载。

### 升级（重要）

**fnOS 应用中心不支持第三方应用的原地覆盖升级**，直接装新版本会提示：

```
[Info]Application [sharemanager] is installed.
```

正确流程是**先卸载、再安装**（配置会保留）：

```bash
sudo appcenter-cli uninstall sharemanager
sudo appcenter-cli install-fpk /path/to/sharemanager-新版本.fpk --volume 1
sudo appcenter-cli start sharemanager
```

`upgrade_init` / `upgrade_callback` 脚本已按同样的自愈逻辑实现，若将来应用中心开放升级通道可直接复用。

---

## 11. 本地开发与调试

### 后端本地跑（Windows / Linux 均可）

不设 `SOCKET_PATH` 即监听 TCP：

```bash
# Windows（注意用 Python 3.10+）
set TRIM_DEV=1
python app/server/server.py     # → http://127.0.0.1:5955
```

```bash
# Linux / macOS
TRIM_DEV=1 python3 app/server/server.py
```

注意：Windows 上没有 `/etc/samba/`，涉及 Samba 的接口会失败，但**静态页面与前端调试完全可用**。

### 直接调后端 API（在飞牛上）

经统一网关（需管理员 uid）：

```bash
sudo curl -sS --unix-socket /vol1/@appcenter/sharemanager/app.sock \
  -H 'X-Trim-Userid: 1000' -H 'Content-Type: application/json' \
  http://localhost/app/sharemanager/api/shares
```

### 查看应用日志

```bash
tail -f /vol1/@appdata/sharemanager/app.log
```

### 手动重启应用

```bash
sudo appcenter-cli stop sharemanager && sudo appcenter-cli start sharemanager
# 或
sudo /var/apps/sharemanager/cmd/main restart 2>/dev/null || \
  (sudo /var/apps/sharemanager/cmd/main stop; sudo /var/apps/sharemanager/cmd/main start)
```

---

## 12. Samba 配置约定

### 配置文件关系

```
/etc/samba/smb.conf
├── [global] …                       # 系统配置，本应用绝不修改
├── include = /etc/samba/smb.sharemanager.conf   ← 本应用添加的唯一一行
├── include = /etc/samba/smb.custom.conf        # 手工共享（历史遗留，可被本应用就地编辑）
└── include = /etc/samba/users/%Y.share.conf    # fnOS 动态逻辑共享（本应用只读）
```

托管文件示例：

```ini
# sharemanager managed shares
# 由「远程共享」应用维护，请勿手工编辑

[公共文件]
	path = /vol1/1000/公共文件
	comment = 
	browseable = yes
	read only = no
	guest ok = yes
	available = yes
	hide unreadable = yes
	hide special files = yes
	force user = nobody
	force group = nogroup
	create mask = 0777
	directory mask = 0777
```

### 写入流程（每次保存 / 删除都走这条链）

```
1. WRITE_LOCK 加锁（进程内串行化）
2. 读原文件 → 文本级段落替换（保留注释、顺序、空行）
3. 写 .tmp → flush + fsync → os.replace 原子落盘
4. testparm 校验
     ├── 失败 → 用原文回滚，返回 400
     └── 成功 → 继续
5. smbcontrol all reload-config（热重载，不清空现有会话）
6. 对该共享执行 smbcontrol all close-share <名>（改名时新旧名都断）
7. 清 testparm 缓存
```

> 第 6 步很关键：`reload-config` 只对**新连接**生效，
> 已连上的 Windows 客户端会一直沿用建连时的权限。
> 主动 `close-share` 等价于「删除重建」的效果，使新权限在下一次访问立即生效。

---

## 13. 常见问题（踩坑记录）

### ① 装完图标还是旧图

**不是安装的问题**——检查安装目录会发现图标文件其实已经是新的。
原因是应用中心 / 浏览器按**固定 URL**（`images/icon_256.png`）缓存图片，而版本号若一直不变，缓存 key 也不变。

**解决**：图标文件名带版本后缀，并同步修改 `app/ui/config`：

```json
{ "sharemanager.main": { "icon": "images/icon_v3_{0}.png", … } }
```

装完建议 `Ctrl+F5` 强刷浏览器；桌面图标可注销重登。

### ② 改了只读 / 匿名，客户端权限没变化

配置其实写对了，是**客户端 SMB 会话缓存**没刷新。见 §12 第 6 步，本应用已在保存/删除后主动
`smbcontrol close-share`。若客户端仍显示旧状态，关闭该共享窗口或断开映射的驱动器再重开即可。

### ③ 匿名共享点了提示「没有权限打开」

匿名访问映射到 `nobody`，需要目录对 `others` 有权限。勾选弹窗里的
**「自动调整目录权限」**（默认勾选）即可，它会对目录执行 `chmod -R o+rwX`。

### ④ Python `http.server` 在 Unix socket 下崩溃 / API 全部超时

Unix socket 连接下 `client_address` 是**空字符串**，`BaseHTTPRequestHandler` 默认的
`address_string()` 访问 `[0]` 会抛 `IndexError`。本应用覆写了 `log_message` 绕开：

```python
def log_message(self, fmt, *args):
    sys.stdout.write("%s - %s\n" % (time.strftime("%H:%M:%S"), fmt % args))
```

### ⑤ 列表页加载要等好几秒

`testparm` 首次启动需要 2~5 秒。读取路径已改为**直接解析配置文件**，
`testparm` 只用于写入校验与状态页，且状态页结果带 120 秒缓存。

### ⑥ 从高版本装回低版本被拒

应用中心会判定为降级。需**先卸载再安装**。

---

## 14. 版本历史

| 版本 | 说明 |
|---|---|
| **1.0.1** | 补全发布者信息（`distributor`）；图标文件名加版本后缀以刷新应用中心/桌面的图标缓存 |
| **1.0.0** | 首个正式版：更名「远程共享」、新图标（贴合飞牛原生风格）、目录选择器默认打开「我的文件」并支持快捷定位；共享创建/编辑/重命名/删除，匿名、只读、可浏览与用户权限控制，配置校验与热重载，保存后主动断开旧连接；服务状态页展示 8 种文件服务协议运行状态 |

---

## 15. 上传代码库前的准备

### 建议的 `.gitignore`

```gitignore
# Python 缓存
__pycache__/
*.py[cod]

# 打包产物（如需随仓库分发安装包，可删掉此行）
*.fpk

# 本地备份 / 编辑器杂项
*.bak
manifest.bak
*.tmp
.DS_Store
```

### 提交前检查

1. 删除 `manifest.bak`（本地备份，不属于源码）；
2. 确认没有 `__pycache__/` 被提交；
3. `ICON.PNG`、`ICON_256.PNG`、`app/ui/images/*.png` **必须提交**（打包依赖它们）；
4. `app/server/public/` 三个前端文件是源码，不要当成构建产物忽略；
5. 若仓库需要展示效果图，可把 `doc/screenshot-*.png` 单独放，不混在打包目录里
   （`fnpack build` 会把 `sharemanager/` 下**所有文件**打进包，多余文件会增大安装包）。

### 仓库建议结构

```
remote-share-manager/          # 仓库根
├── README.md                  # 本文件
├── LICENSE
├── .gitignore
├── docs/                      # 截图、流程图等（不参与打包）
└── src/                       # 即原来的 sharemanager/ 目录内容
    ├── manifest
    ├── ICON.PNG
    ├── app/ config/ cmd/ wizard/
    └── README.md（可软链或复制）
```

> 打包时在 `src/`（原 `sharemanager/`）目录内执行 `fnpack build`，
> 因此**任何不需要发布到安装包的文件都应放在该目录之外**。

---

## 作者与许可

- 作者：**草原企鹅**
- 应用名：远程共享（`sharemanager`）
- 本应用为第三方应用（`source = thirdparty`），与飞牛 fnOS 官方无关。
