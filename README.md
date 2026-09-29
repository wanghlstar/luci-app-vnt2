# luci-app-vnt2

这是一个用于管理 VNT2 CLI / CTRL / Web / Server 服务的 OpenWrt LuCI 插件，基于 [weicaixian86/luci-app-vnt2](https://github.com/weicaixian86/luci-app-vnt2) v2.0.48 三模式版本，并修复了其中的安全与功能缺陷。

## 功能特性

三个可独立启用、互斥提示的服务模块：

| 模块 | 二进制 | 说明 |
|------|--------|------|
| `vnt2_cli` 客户端 | `vnt2_cli` + `vnt2_ctrl` | 命令行客户端，支持网络参数、STUN、端口映射、加密压缩、连接信息面板 |
| `vnt2_web` Web 客户端 | `vnt2_web` | Web 管理端（默认 19099 端口），与客户端共用同一份 TOML 配置 |
| `vnts2` 服务端 | `vnts2` | 服务端，监听 TCP/QUIC/WS，自带 Web 管理，支持集群与网段分配 |

通用能力：

- **配置生成**：从 UCI 自动导出 TOML 运行时配置，无需手写
- **程序自动下载**：缺二进制时从多镜像（gh-proxy / GitHub / Gitee / GitLab / Cloudflare R2）下载匹配架构的发行包
- **上传安装**：网页上传二进制或 `.tar.gz` 压缩包，校验 ELF/压缩包安全后安装到 `/usr/bin`
- **日志查看**：CLI / Web / 服务端 / 下载四套日志页，自动刷新与清除
- **连接信息面板**：通过 `vnt2_ctrl` 查看本机信息、IP 列表、设备列表、路由、启动参数
- **防火墙自动化**：按需创建 VNT2 区域、转发规则、端口放行

## 包名称说明

OpenWrt 软件包管理器中的实际包名固定为：

- `luci-app-vnt2`

无论是在 `系统 -> 软件包` 页面中搜索，还是使用 `opkg` / `apk` 查询，实际包名都应当使用 `luci-app-vnt2`。

## 发布文件说明

GitHub Release 中发布的安装文件，保留 OpenWrt 标准构建产物命名方式。
文件名中可能带有版本号、发布号、架构等后缀，但软件包名前缀始终为：

- `luci-app-vnt2`

例如：

- `luci-app-vnt2_2.1.0-r1_all.ipk`
- `luci-app-vnt2-2.1.0-r1.apk`

## 安装方法

### OpenWrt 24.10.x

将 `.ipk` 文件上传到路由器，例如上传到 `/tmp/`，然后执行：

```sh
opkg install /tmp/luci-app-vnt2*.ipk
opkg info luci-app-vnt2
opkg list-installed | grep luci-app-vnt2
```

### OpenWrt 25.12.0

将 `.apk` 文件上传到路由器，例如上传到 `/tmp/`，然后执行：

```sh
apk add --allow-untrusted /tmp/luci-app-vnt2*.apk
apk info luci-app-vnt2
```

## 在 OpenWrt 源码树中编译

可将本项目放入 OpenWrt 的 `package/` 目录，或放入自定义 feed 中，然后执行：

```sh
git clone <你的仓库地址> package/luci-app-vnt2
make menuconfig
make package/luci-app-vnt2/compile V=s
```

编译完成后：

- OpenWrt 24.10.x 生成 `.ipk`
- OpenWrt 25.12.0 生成 `.apk`

## LuCI 菜单位置

安装完成后，在 LuCI 中进入：

```text
VPN -> VNT2
```

包含五个页面：**基本设置**（三个模块的配置页签）、**CLI 日志**、**Web 日志**、**服务端日志**、**下载日志**。

## 配置文件说明

| 路径 | 用途 |
|------|------|
| `/etc/config/vnt2` | UCI 配置（三个 section：`vnt2_cli` / `vnt2_web` / `vnts2`） |
| `/etc/config/vnt2_cli_web.toml` | 客户端与 Web 共用的运行时 TOML（vnt2_web 与 vnt2_cli 读同一份） |
| `/etc/config/vnts2.toml` | 服务端运行时 TOML |
| `/usr/bin/vnt2_cli` `vnt2_ctrl` `vnt2_web` `vnts2` | 各模块二进制 |
| `/etc/config/network_control.db` `cert.pem` `key.pem` | 服务端设备数据库与自签名证书 |
| `/etc/vnt_config/` | vnt2_web 数据目录（内含托管配置的链接） |
| `/tmp/vnt_current_config.txt` | vnt2_web 自启配置记录（软链至 /tmp，不写 flash） |
| `/tmp/vnt2-cli.log` `/tmp/vnt2-web.log` `/tmp/vnts2.log` | 各模块运行日志 |
| `/tmp/vnt2-download.log` | 下载/安装日志 |

## 升级说明

从 2.1.0 之前的版本升级时，旧路径 `/vnt_config/vnt2_cli_web.toml` 中的配置会在首次启动时自动迁移到 `/etc/config/vnt2_cli_web.toml`（新路径已存在则不覆盖）。

## 防火墙说明

启动时会自动创建以下规则（按需）：

- **客户端**：`VNT2` 网络区域 + lan/wan 双向 forwarding 规则（可在“访问控制”多选里调整）；指定了固定 `tunnel_port` 时会自动放行该端口的 WAN 入站（tcp+udp），否则使用随机端口无法静态放行
- **服务端**：`tcp/quic/ws/web` 四个端口各自受“允许 WAN 访问”开关控制；配置了 `server_quic_bind`（集群）时自动放行该 UDP 端口
- **Web**：19099 端口受“允许 WAN 访问”控制（默认关闭）

## 使用提示

- 三个模块默认均为**未启用**，需要在基本设置中勾选启用并保存应用
- 首次启用时若二进制不存在，会自动从镜像源下载；也可在各自页签的“上传程序”中手动上传
- 客户端页签的“连接信息”可查看本机信息、设备列表、路由等（需要 `vnt2_ctrl`）
- 服务端 Web 管理默认端口 29871，客户端 Web 管理默认端口 19099

## 自动构建（GitHub Actions）

仓库内置 `.github/workflows/build.yml`：

| 触发方式 | 行为 |
|----------|------|
| 推送 `main` 分支 | 构建 x86_64 的 ipk（24.10）+ apk（25.12），上传为 Actions artifacts |
| 推送 `v*` tag | 构建并自动发布 Release |
| 手动触发 | Actions → Build workflow → Run workflow |

CI 仅构建 **x86_64**；其他架构请在对应 SDK 中编译。

## 相对上游 v2.0.48 的修复

- 修复 IP 转发逻辑写反导致防火墙 forwarding 规则失效
- 修复三个启动命令的 shell 注入（二进制路径白名单校验）
- 修复三个“上传程序”页签完全不可用（被 LuCI 框架覆盖），重做为框架原生流程并补齐安装逻辑
- 修复六个日志页的存储型 XSS（pcdata 转义）
- 修复清日志端点 CSRF（改为 POST + token）
- 修复 TOML 转义不处理换行/控制字符导致服务无法启动
- 修复 `get_router_host` 采信 HTTP_HOST 导致的开放重定向
- 修复 Web 面板默认放行 WAN（`web_wan` 默认改为关闭）
- 修复 /tmp 下载缓存权限、`unzip -t` 非法用法、URL 候选遍历被 glob 破坏等一批中低危问题

## 截图

### 状态总览

![状态总览](jpg/1.jpg)

### `vnt2_cli` 客户端配置

![vnt2_cli 客户端配置](jpg/2.jpg)

### `vnt2_web` 客户端配置

![vnt2_web 客户端配置](jpg/3.jpg)

### `vnts2` 服务端配置

![vnts2 服务端配置](jpg/4.jpg)

## 致谢

- [vnt-dev/vnt](https://github.com/vnt-dev/vnt) — VNT 本体
- [weicaixian86/luci-app-vnt2](https://github.com/weicaixian86/luci-app-vnt2) — 上游
