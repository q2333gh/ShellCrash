# 关于 `http://192.168.31.1:9999/ui` 的原理说明

## 你的疑问

当用户初次运行并完成配置后，为什么会出现 `http://192.168.31.1:9999/ui`？
是不是路由器上启动了一个 HTTP 服务器？

## 结论（先说答案）

是的，会有一个 HTTP 服务端口在监听；但通常不是额外再部署一个独立 Web 管理后端，而是由代理内核（mihomo/sing-box 的 Clash API）暴露控制面板与 REST 接口。

`/ui` 页面本质是一个静态目录映射，`9999` 是控制端口（`db_port`）。

## 代码层证据

1. 控制端口默认值
- `scripts/libs/get_config.sh` 中：`db_port` 默认 `9999`。

2. mihomo/clash 配置写入
- `scripts/starts/clash_modify.sh` 会写入：
  - `external-controller: :$db_port`
  - `external-ui: ui`
  - `external-ui-url: ...`

这表示内核在 `db_port` 上开放控制器，并把 `ui` 目录作为面板路径。

3. sing-box 配置写入
- `scripts/starts/singbox_modify.sh` 会写入：
  - `experimental.clash_api.external_controller: 0.0.0.0:$db_port`
  - `experimental.clash_api.external_ui: "ui"`

同样是在内核的 Clash API 端口提供 UI。

4. UI 文件准备
- `scripts/starts/bfstart.sh` 会确保 `ui` 目录存在并生成/补齐 `index.html`（没有本地面板时会放一个跳转提示页）。

## 运行机制（初始化后）

1. ShellCrash 生成最终配置（YAML/JSON），其中包含 `external-controller` 与 `external-ui`。
2. 启动 `CrashCore` 进程。
3. `CrashCore` 在 `db_port`（默认 9999）监听 HTTP 控制接口。
4. 浏览器访问 `http://<路由器IP>:9999/ui`，由内核返回 UI 静态资源。

因此，`/ui` 不是 CLI 菜单渲染出来的页面，而是内核 HTTP 控制面提供的。

## 为什么有时不是 `:9999/ui`？

代码里有两种展示方式：

- 如果系统存在 `/www/clash/index.html`（常见 OpenWrt Web 根目录），可能提示使用 `/clash`。
- 否则通常提示 `:$db_port/ui`（即 `:9999/ui`）。

也就是说：
- `:9999/ui`：内核控制端口直出。
- `/clash`：由路由器现有 Web 服务（如 uhttpd）托管静态文件。

## 对你重构方向的意义

如果要改成“纯 HTTP API + Web 面板”，当前机制已经具备基础：

- 现有核心已经提供了 HTTP 控制面入口。
- 你需要做的是把原 CLI 菜单配置流程改造成 API 化流程，并把初始化向导前移到浏览器页面。
- 保留 `start.sh`/`starts/*` 的执行能力，逐步替换 `menu.sh` 的交互职责即可。

## 为什么小路由器也能跑这个 HTTP 控制面？（原理补充）

核心原因：这里通常不是额外部署一个“重型 Web 服务器”，而是代理内核进程内嵌了轻量 HTTP 控制接口。

1. 同进程复用事件循环
- 代理内核本来就是常驻进程，已经在处理大量网络连接。
- 额外监听一个管理端口（如 `9999`）只是在同一进程增加一个 socket 监听点。

2. HTTP 只是控制通道，不是重业务系统
- 主要返回控制 API（JSON）和少量静态资源（`/ui`）。
- 不涉及数据库、模板引擎、大量并发业务逻辑，资源开销较小。

3. 路由器运行的是完整 Linux 网络栈
- 具备 TCP/IP、`listen/accept`、文件系统等基础能力。
- 从操作系统角度看，“开启 HTTP 端口”与普通 Linux 主机没有本质差异。

4. 为什么资源上可承受
- 局域网管理请求频率低，通常只在配置/查看状态时访问。
- 控制面请求量远小于代理转发主业务流量。
- 内核使用的是轻量实现（常见为 Go/Rust 生态内置 HTTP 能力）。

结论：`/ui` 能在路由器上工作，不是因为额外跑了一个很重的 Web 平台，而是代理内核顺带提供了轻量管理接口，这在资源受限设备上是常见设计。
