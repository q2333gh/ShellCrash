# Unified Control Plane Master Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 在 ShellCrash 中建立统一控制平面，支持 Web 配置、Wizard 配置和 LLM Agent 友好 CLI，三入口共享一套动作语义与执行引擎，实现高复用、低耦合、可迁移。

**Architecture:** 采用“多入口 + 单动作层 + 单执行引擎”架构。Web UI、Wizard CLI、Non-interactive CLI 仅负责交互与参数采集；所有业务动作统一进入 Control Action Layer，再调用现有 `start.sh`/`starts/*`/`libs/*`。通过统一契约（JSON 输出、错误码、参数 schema）保证人类和 LLM 的一致体验。

**Tech Stack:** POSIX Shell（执行引擎）、Control Action Layer（Shell 起步，Go 可演进）、HTTP API、静态 Web、CLI 子命令体系、JSON Schema、ShellCheck、契约测试（CLI/API/Parity）。

---

## 一、总分层设计（高复用核心）

### Layer 1: Interaction Frontends（多入口）

- Web Control Panel：浏览器初始化与日常管理。
- Wizard CLI：人类友好多步骤向导（`start/next/back/review/apply`）。
- LLM agnet CLI：非交互命令（`--output json --yes --from-json --dry-run`）。

职责：只负责输入收集、展示、会话引导，不直接写系统状态。

### Layer 2: Control API/CLI Adapters（协议适配）

- CLI Parser + Router
- HTTP Router + Handlers
- Wizard Session Manager

职责：把前端请求转换为统一 Action 调用；统一错误与返回格式。
我
### Layer 3: Control Action Layer（唯一业务入口）

- `system.*` / `config.*` / `core.*` / `task.*` / `onboarding.*`
- 参数校验、幂等检查、变更计划生成、审计埋点

职责：唯一业务语义层，Web/Wizard/CLI 共用，防止三套逻辑分叉。

### Layer 4: Execution Engine（复用现有能力）

- 复用：`scripts/start.sh`、`scripts/starts/*`、`scripts/libs/*`
- 包含：内核启动、配置改写、防火墙注入、策略路由、任务调度

职责：稳定执行，不承载交互逻辑。

### Layer 5: State & Contract（状态与契约）

- `ShellCrash.cfg` / `command.env` / Wizard session state
- JSON Schema（输入输出）
- Error Code Registry

职责：保证可恢复、可回放、可测试。

---

## 二、统一命令与接口契约

### CLI 命令域（建议）

- `crash system status|start|stop|restart`
- `crash config get|set|list`
- `crash core switch|info`
- `crash onboarding apply|validate`
- `crash wizard start|next|back|set|review|apply`

### API 命令域（与 CLI 一一映射）

- `/api/v1/system/*`
- `/api/v1/config/*`
- `/api/v1/core/*`
- `/api/v1/onboarding/*`
- `/api/v1/wizard/*`

### 输出和退出码（强约束）

- 默认 `--output json`
- 固定结构：`{ ok, code, message, data, hints }`
- 退出码：`0/2/3/4`

---

## 三、实施路线图（整合版）

### Phase 0: 架构冻结与契约先行（3-5 天）

**目标：** 固化三入口共用契约，避免后续返工。

**Deliverables：**
- ADR：统一控制平面与双入口策略
- CLI Contract / API Contract / Error Code Registry
- CLI-API parity matrix（命令与接口映射）

### Phase 1: 抽离共享 Action 层（1 周）

**目标：** 建立唯一业务入口，旧菜单先“壳保留、核替换”。

**Deliverables：**
- `scripts/control/actions.sh`
- `scripts/control/validators.sh`
- `scripts/control/renderers.sh`
- 基础动作：status/start/stop/config get/set/core switch

### Phase 2: LLM 友好 CLI 上线（1 周）

**目标：** 提供可脚本化、可批处理 CLI。

**Deliverables：**
- 新 CLI 子命令框架（resource/action）
- `--from-json`、`--dry-run`、`--yes`、`--output json`
- CLI 契约测试

### Phase 3: Wizard CLI 重构（4-6 天）

**目标：** 人类多步骤体验保留，但语义显式化（替代 `0 返回`）。

**Deliverables：**
- `wizard back/cancel/resume/reset-step`
- Wizard session state 持久化
- 与 onboarding apply 共享动作层

### Phase 4: Web 控制面板与 API 对齐（1 周）

**目标：** Web 完整接入同一动作层，覆盖初始化与运维。

**Deliverables：**
- API Router/Handlers 调 Action 层
- Web onboarding + dashboard
- API/CLI parity 测试通过

### Phase 5: 安全、审计、可观测（4-6 天）

**目标：** 在多入口统一后补齐治理能力。

**Deliverables：**
- 认证 + CSRF + 基础限流
- Action 级审计日志（来源、动作、结果码、耗时）
- 错误聚合与故障排查指引

### Phase 6: 迁移发布与回滚（3-5 天）

**目标：** 平滑替换旧数字菜单，不中断用户。

**Deliverables：**
- `control_plane_mode=legacy|hybrid|web_first|cli_first`
- 灰度发布与自动回滚策略
- 迁移文档与变更公告

---

## 四、目录与模块建议

- `scripts/control/`：动作层、校验器、渲染器、审计
- `scripts/cli/`：CLI 解析与命令实现
- `scripts/api/`：HTTP 适配层
- `public/control-panel/`：Web 前端
- `docs/specs/`：CLI/API/schema/错误码契约
- `tests/`：`cli/`、`api/`、`parity/`、`migration/`

---

## 五、关键复用点（必须坚持）

1. 三入口只做交互，不做业务
2. Action 层是唯一业务编排入口
3. 执行层优先复用现有 `start.sh` + `starts/*` + `libs/*`
4. 契约优先（先定 schema，再写实现）
5. CLI 与 API 语义同构（任何动作均可互换调用）

---

## 六、风险与对策

- 风险：旧流程耦合菜单状态
  - 对策：先抽 Action，再替换入口

- 风险：多入口导致行为不一致
  - 对策：Parity 测试作为 CI 强门禁

- 风险：路由器环境差异大
  - 对策：HTTP server adapter + capability probe

- 风险：安全面扩大
  - 对策：统一认证层、写操作防护、审计日志默认开启

---

## 七、验收标准（Definition of Done）

1. Web/Wizard/LLM CLI 三入口均可完成初始化与核心运维
2. 所有关键动作具备 JSON 输出和稳定退出码
3. CLI 与 API parity 测试通过
4. 启停、防火墙、透明代理行为无回归
5. 迁移模式可灰度切换并可回滚

---

## 八、最终推荐发布策略

- vNext：`hybrid` 默认（新 CLI + Web 可用，legacy 菜单隐藏保留）
- vNext+1：`cli_first` 或 `web_first`（按用户群选择）
- vNext+2：移除 legacy 数字菜单主路径，仅保留 `legacy-menu` 调试入口

该计划能同时满足：
- 人类用户：向导与 Web 易用性
- LLM agent：稳定、可脚本化 CLI
- 工程团队：高复用、层级清晰、低维护分叉
