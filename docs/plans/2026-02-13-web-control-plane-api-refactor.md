# Web-First Control Plane Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将 ShellCrash 的入口交互层从 CLI 多级数字菜单重构为“纯 HTTP API + Web 控制面板”，用户连接路由器 Wi-Fi 后通过浏览器完成初始化与后续管理。

**Architecture:** 通过新增常驻 API 服务进程接管交互语义，`scripts/menu.sh` 不再承载业务交互逻辑，仅保留兼容跳转与维护入口。核心执行链（`start.sh` + `starts/*` + `libs/*`）继续复用，通过 API Handler 调用“命令编排层”实现配置读写、核心管理、服务启停与状态查询。前端控制面板调用本地 API 完成初始化向导与运行态管理。

**Tech Stack:** POSIX Shell（核心执行）、轻量 HTTP 服务（uhttpd 或内置 busybox httpd + CGI/FastCGI）、JSON API、静态 Web 前端（Vanilla/轻量框架）、ShellCheck、端到端冒烟测试（curl + sh）。

---

### Task 1: 定义目标架构与兼容边界

**Files:**
- Create: `docs/adr/2026-02-13-web-first-control-plane.md`
- Modify: `docs/项目架构与核心原理报告.md`
- Modify: `README.md`
- Modify: `README_CN.md`

**Step 1: 写出架构决策记录（ADR）**

- 明确“入口交互层弃用数字菜单”的决策与范围。
- 定义保留项：`start.sh`、`starts/*`、`libs/*` 不做行为回归。
- 定义替换项：`menu.sh` 的业务入口替换为 API + Web。

**Step 2: 明确对外兼容策略**

- CLI 兼容命令仅保留：`start/stop/restart/status/log/debug`。
- 所有原“菜单配置项”必须有对应 API 路径。
- 标注废弃时间线：vNext 兼容，vNext+1 移除交互菜单。

**Step 3: 文档化 API 作为唯一交互层**

- 在 README 中更新“初始化方式”为浏览器控制面板。
- 明确默认访问地址、端口、首次初始化入口。

**Step 4: 评审通过后冻结范围**

- 冻结本次不做项：核心代理逻辑重写、内核替换策略大改。

**Step 5: Commit**

```bash
git add docs/adr/2026-02-13-web-first-control-plane.md docs/项目架构与核心原理报告.md README.md README_CN.md
git commit -m "docs: define web-first control plane architecture and compatibility"
```

### Task 2: 抽离“命令编排层”并建立 API 可调用边界

**Files:**
- Create: `scripts/api/commands.sh`
- Create: `scripts/api/validators.sh`
- Modify: `scripts/menu.sh`
- Modify: `scripts/menus/*.sh`（按需迁移纯业务函数）
- Test: `tests/api/commands_spec.sh`

**Step 1: 写失败测试（命令编排函数契约）**

- 设计函数：`api_get_status`、`api_set_config`、`api_start_service`、`api_stop_service`、`api_switch_core`。
- 断言输出为 JSON，错误返回非 0，并包含标准错误码。

**Step 2: 运行测试确认失败**

Run: `sh tests/api/commands_spec.sh`
Expected: FAIL，提示函数或输出格式不存在。

**Step 3: 最小实现命令编排层**

- 在 `scripts/api/commands.sh` 中封装对 `start.sh`、`libs/set_config.sh`、`libs/get_config.sh` 的调用。
- 不允许直接拼接用户输入到 shell 命令，全部经 `validators.sh` 白名单校验。

**Step 4: 再跑测试确保通过**

Run: `sh tests/api/commands_spec.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add scripts/api/commands.sh scripts/api/validators.sh tests/api/commands_spec.sh scripts/menu.sh
git commit -m "refactor(api): extract command orchestration layer from cli menus"
```

### Task 3: 实现 HTTP API 服务骨架（控制平面后端）

**Files:**
- Create: `scripts/api/server.sh`
- Create: `scripts/api/router.sh`
- Create: `scripts/api/handlers/system.sh`
- Create: `scripts/api/handlers/config.sh`
- Create: `scripts/api/handlers/core.sh`
- Create: `scripts/api/handlers/tasks.sh`
- Modify: `scripts/start.sh`
- Modify: `scripts/starts/general_init.sh`
- Test: `tests/api/http_smoke.sh`

**Step 1: 写失败测试（HTTP 路由可达性）**

- 设计冒烟：`GET /api/v1/health`、`GET /api/v1/system/status`。
- 验证 HTTP 200 + JSON Content-Type。

**Step 2: 运行测试确认失败**

Run: `sh tests/api/http_smoke.sh`
Expected: FAIL（端口未监听或路由不存在）。

**Step 3: 实现最小 API Server 与 Router**

- 支持路由分发、方法校验、统一 JSON 响应。
- 统一错误模型：`{"code":"...","message":"...","details":...}`。

**Step 4: 集成到生命周期管理**

- 在 `start.sh` 中新增 `api-start/api-stop` 并纳入 `start`/`stop` 链路。
- `general_init.sh` 保证开机后 API 面板可达。

**Step 5: 再跑测试确保通过**

Run: `sh tests/api/http_smoke.sh`
Expected: PASS。

**Step 6: Commit**

```bash
git add scripts/api/server.sh scripts/api/router.sh scripts/api/handlers scripts/start.sh scripts/starts/general_init.sh tests/api/http_smoke.sh
git commit -m "feat(api): add http control-plane server and lifecycle integration"
```

### Task 4: 覆盖初始化向导 API（替代首次 CLI 配置）

**Files:**
- Create: `scripts/api/handlers/onboarding.sh`
- Modify: `scripts/init.sh`
- Modify: `scripts/libs/set_profile.sh`
- Modify: `scripts/libs/set_config.sh`
- Test: `tests/api/onboarding_spec.sh`

**Step 1: 写失败测试（初始化向导）**

- `POST /api/v1/onboarding/start`：生成会话与最小默认配置。
- `POST /api/v1/onboarding/network`：写入运行模式、端口与 LAN 访问策略。
- `POST /api/v1/onboarding/complete`：触发配置校验并启动服务。

**Step 2: 运行测试确认失败**

Run: `sh tests/api/onboarding_spec.sh`
Expected: FAIL。

**Step 3: 实现最小向导处理器**

- 将原 `init.sh` 里“交互输入决策”改为“参数化函数”。
- API 层只做参数接收与校验，业务执行下沉到可复用函数。

**Step 4: 再跑测试确保通过**

Run: `sh tests/api/onboarding_spec.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add scripts/api/handlers/onboarding.sh scripts/init.sh scripts/libs/set_profile.sh scripts/libs/set_config.sh tests/api/onboarding_spec.sh
git commit -m "feat(onboarding): provide web-first initialization APIs"
```

### Task 5: 构建 Web 控制面板（初始化 + 常规管理）

**Files:**
- Create: `public/control-panel/index.html`
- Create: `public/control-panel/app.js`
- Create: `public/control-panel/styles.css`
- Create: `public/control-panel/views/onboarding.js`
- Create: `public/control-panel/views/dashboard.js`
- Modify: `scripts/starts/bfstart.sh`
- Test: `tests/web/panel_smoke.sh`

**Step 1: 写失败测试（静态资源与 API 联通）**

- 验证面板 URL 可访问，且能调用 `GET /api/v1/health`。

**Step 2: 运行测试确认失败**

Run: `sh tests/web/panel_smoke.sh`
Expected: FAIL。

**Step 3: 最小实现前端页面**

- 初始化向导：网络模式、核心选择、订阅导入、启动确认。
- 管理面板：状态、启停、日志、配置更新、核心切换。
- 所有操作均走 HTTP API，不再依赖 CLI 菜单语义。

**Step 4: 再跑测试确保通过**

Run: `sh tests/web/panel_smoke.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add public/control-panel scripts/starts/bfstart.sh tests/web/panel_smoke.sh
git commit -m "feat(web): add browser control panel for onboarding and management"
```

### Task 6: 安全与认证基线（路由器局域网场景）

**Files:**
- Create: `scripts/api/auth.sh`
- Create: `scripts/api/csrf.sh`
- Modify: `scripts/api/server.sh`
- Modify: `scripts/api/router.sh`
- Modify: `scripts/configs/ShellCrash.cfg`（默认鉴权策略）
- Test: `tests/api/security_spec.sh`

**Step 1: 写失败测试（未授权访问必须拒绝）**

- 未登录访问写操作返回 401/403。
- 非法 origin/referer 写操作拒绝。
- 初始化完成后强制要求设置管理口令。

**Step 2: 运行测试确认失败**

Run: `sh tests/api/security_spec.sh`
Expected: FAIL。

**Step 3: 实现最小安全机制**

- 会话 token + HttpOnly Cookie。
- 写操作 CSRF Token 校验。
- 失败次数限制与基础日志。

**Step 4: 再跑测试确保通过**

Run: `sh tests/api/security_spec.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add scripts/api/auth.sh scripts/api/csrf.sh scripts/api/server.sh scripts/api/router.sh tests/api/security_spec.sh
git commit -m "feat(security): add auth and csrf protection for web control plane"
```

### Task 7: 下线数字菜单交互，保留最小 CLI 兼容层

**Files:**
- Modify: `scripts/menu.sh`
- Modify: `scripts/menus/*.sh`
- Create: `scripts/cli/compat.sh`
- Test: `tests/cli/compat_spec.sh`

**Step 1: 写失败测试（CLI 最小命令仍可用）**

- `crash -s start|stop|restart` 仍有效。
- `crash` 默认行为不再进入数字菜单，而是输出面板访问提示。

**Step 2: 运行测试确认失败**

Run: `sh tests/cli/compat_spec.sh`
Expected: FAIL。

**Step 3: 实现兼容层**

- 删除/禁用菜单循环入口。
- 保留调试与服务控制命令。
- 输出面板地址、初始化状态和故障排查提示。

**Step 4: 再跑测试确保通过**

Run: `sh tests/cli/compat_spec.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add scripts/menu.sh scripts/menus scripts/cli/compat.sh tests/cli/compat_spec.sh
git commit -m "refactor(cli): remove numeric interactive menu and keep compatibility commands"
```

### Task 8: 发布迁移与回滚机制

**Files:**
- Create: `docs/migration/web-control-plane-migration.md`
- Modify: `scripts/starts/start_error.sh`
- Modify: `.github/workflows/test.yaml`
- Create: `.github/workflows/web-control-plane-ci.yaml`

**Step 1: 写失败测试（迁移脚本与回滚）**

- 老配置升级后 API 模式可启动。
- API 启动失败时可回滚到“仅服务控制 CLI”。

**Step 2: 运行测试确认失败**

Run: `sh tests/migration/migration_spec.sh`
Expected: FAIL。

**Step 3: 实现迁移逻辑与 CI**

- 增加迁移脚本检查旧字段并自动补齐新字段。
- 在 CI 中加入 API 与 Web 面板冒烟。

**Step 4: 再跑测试确保通过**

Run: `sh tests/migration/migration_spec.sh`
Expected: PASS。

**Step 5: Commit**

```bash
git add docs/migration/web-control-plane-migration.md scripts/starts/start_error.sh .github/workflows/test.yaml .github/workflows/web-control-plane-ci.yaml tests/migration/migration_spec.sh
git commit -m "chore(release): add migration, rollback, and web control-plane CI gates"
```

## API 最小清单（v1）

- `GET /api/v1/health`
- `GET /api/v1/system/status`
- `POST /api/v1/system/start`
- `POST /api/v1/system/stop`
- `POST /api/v1/system/restart`
- `GET /api/v1/config`
- `PUT /api/v1/config`
- `POST /api/v1/core/switch`
- `GET /api/v1/logs`
- `POST /api/v1/onboarding/start`
- `POST /api/v1/onboarding/network`
- `POST /api/v1/onboarding/complete`

## 非功能要求

- 局域网首次打开面板 <= 2 秒。
- API 写操作默认需要认证与 CSRF 保护。
- 启停链路与旧版本相比不增加行为回归。
- 关键 API 错误必须结构化返回，便于前端提示。

## 风险与应对

- 风险 1：路由器环境 HTTP 运行时差异大。
  - 应对：抽象 server adapter（uhttpd/httpd 两套实现）。
- 风险 2：旧脚本流程耦合菜单变量。
  - 应对：先抽离命令编排层再替换入口，不直接在 handler 中写业务。
- 风险 3：安全暴露面增加。
  - 应对：默认启用口令认证、CSRF、写接口限流与访问日志。

## 里程碑建议

- M1（1 周）：API 骨架 + 初始化向导 + CLI 兼容。
- M2（1 周）：Web 面板 + 安全基线 + CI。
- M3（3-5 天）：迁移发布与灰度观察。
