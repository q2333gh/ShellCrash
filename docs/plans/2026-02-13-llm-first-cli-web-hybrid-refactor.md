# LLM-First CLI + Web Hybrid Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将 ShellCrash 重构为“LLM 友好 CLI 为一等入口，同时支持 Web 控制面板”的双入口控制平面，替代多轮数字菜单交互。

**Architecture:** 建立统一 `control-plane` 服务层，CLI 与 Web 仅作为两种 Frontend，均调用同一动作接口（Action API）。CLI 采用非交互命令、稳定 JSON 输出与标准退出码；Web 通过 HTTP API 调同一动作，避免双套业务逻辑分叉。

**Tech Stack:** POSIX Shell（执行引擎）、Action 层（Shell/Go 过渡可选）、HTTP API、静态 Web 前端、JSON Schema、ShellCheck、契约测试（CLI/API）。

---

### Task 1: 定义 LLM 友好 CLI 规范（替代数字菜单）

**Files:**
- Create: `docs/adr/2026-02-13-llm-first-cli-contract.md`
- Create: `docs/specs/cli-command-contract.md`
- Modify: `README.md`
- Modify: `README_CN.md`

**Step 1: 定义命令模型**

- 一级命令按资源域分组：`system`/`config`/`core`/`task`/`onboarding`。
- 统一形式：`crash <resource> <action> [flags]`。

**Step 2: 定义输出与退出码**

- 默认机器可读：`--output json`（建议默认 json，`--output text` 仅人读）。
- 退出码分层：`0` 成功，`2` 参数错误，`3` 业务校验失败，`4` 运行时故障。

**Step 3: 定义“无交互优先”原则**

- 禁止多轮 `read` 交互作为主路径。
- 所有原菜单输入项必须有等价 flags。

**Step 4: Commit**

```bash
git add docs/adr/2026-02-13-llm-first-cli-contract.md docs/specs/cli-command-contract.md README.md README_CN.md
git commit -m "docs(cli): define llm-first command contract and output/exit-code rules"
```

### Task 2: 抽离统一 Action 层（CLI 与 Web 共用）

**Files:**
- Create: `scripts/control/actions.sh`
- Create: `scripts/control/validators.sh`
- Create: `scripts/control/renderers.sh`
- Modify: `scripts/menu.sh`
- Test: `tests/control/actions_spec.sh`

**Step 1: 写失败测试（动作契约）**

- 覆盖：`system.status/start/stop/restart`、`config.get/set`、`core.switch`。
- 断言返回结构：`{"ok":true|false,"code":"...","data":...}`。

**Step 2: 实现最小 Action 层**

- 封装现有 `start.sh`、`libs/get_config.sh`、`libs/set_config.sh`。
- 输入统一先过 `validators.sh`。

**Step 3: 输出渲染器**

- `render_json` 与 `render_text` 两种输出模式。
- 保证字段顺序稳定，便于 LLM agent 解析。

**Step 4: Commit**

```bash
git add scripts/control/actions.sh scripts/control/validators.sh scripts/control/renderers.sh scripts/menu.sh tests/control/actions_spec.sh
git commit -m "refactor(control): extract shared action layer for cli and web"
```

### Task 3: 实现 LLM-First CLI（非交互、可组合）

**Files:**
- Create: `scripts/cli/main.sh`
- Create: `scripts/cli/commands/system.sh`
- Create: `scripts/cli/commands/config.sh`
- Create: `scripts/cli/commands/core.sh`
- Create: `scripts/cli/commands/onboarding.sh`
- Modify: `scripts/menu.sh`
- Test: `tests/cli/contract_spec.sh`

**Step 1: 写失败测试（CLI 契约）**

- 示例：
  - `crash system status --output json`
  - `crash config set --key db_port --value 9999 --output json`
  - `crash onboarding apply --profile quick --yes --output json`

**Step 2: 实现命令解析与分发**

- 子命令 + flags 模式，支持 `--help`、`--output`、`--timeout`。
- 禁止进入旧数字菜单循环。

**Step 3: 增加批处理能力**

- 支持 `--from-json <file>` 一次执行动作（LLM agent 常用）。
- 支持 `--dry-run` 输出预期变更。

**Step 4: Commit**

```bash
git add scripts/cli scripts/menu.sh tests/cli/contract_spec.sh
git commit -m "feat(cli): add llm-first non-interactive command interface"
```

### Task 4: Web 接口与 CLI 语义对齐（双入口同构）

**Files:**
- Modify: `scripts/api/router.sh`
- Modify: `scripts/api/handlers/*.sh`
- Create: `docs/specs/cli-api-parity-matrix.md`
- Test: `tests/api/parity_spec.sh`

**Step 1: 建立语义映射表**

- 每个 CLI 命令必须映射到一个 API endpoint（反向亦然）。
- 统一错误码与错误消息。

**Step 2: 适配 API 调用 Action 层**

- API handler 不直接写业务，改调 `scripts/control/actions.sh`。

**Step 3: 契约测试**

- 同一操作经 CLI 与 API 触发，比较返回码与副作用一致。

**Step 4: Commit**

```bash
git add scripts/api/router.sh scripts/api/handlers docs/specs/cli-api-parity-matrix.md tests/api/parity_spec.sh
git commit -m "refactor(api): align api semantics with llm-first cli via shared actions"
```

### Task 5: 初始化流程重构为“一次命令/一次表单”

**Files:**
- Modify: `scripts/init.sh`
- Modify: `scripts/control/actions.sh`
- Create: `scripts/control/profiles/quick.json`
- Create: `scripts/control/profiles/router.json`
- Test: `tests/onboarding/non_interactive_spec.sh`

**Step 1: 写失败测试（无交互初始化）**

- `crash onboarding apply --profile quick --yes` 可一次完成最小可用配置。
- 与 Web onboarding API 的效果一致。

**Step 2: 实现 profile 化初始化**

- 把“多次数字键选择”转换为参数/模板驱动。
- 提供默认 profile + 自定义 profile 文件。

**Step 3: Commit**

```bash
git add scripts/init.sh scripts/control/actions.sh scripts/control/profiles tests/onboarding/non_interactive_spec.sh
git commit -m "feat(onboarding): support one-shot non-interactive initialization for cli and web"
```

### Task 6: 保留 Web，同时降级旧菜单为兼容层

**Files:**
- Modify: `scripts/menu.sh`
- Create: `scripts/cli/compat_notice.sh`
- Modify: `public/control-panel/*`
- Test: `tests/cli/compat_notice_spec.sh`

**Step 1: 兼容策略**

- `crash` 无参数：输出状态 + 推荐命令 + Web 地址。
- `crash legacy-menu`：仅调试期保留，默认隐藏。

**Step 2: Web 继续作为 GUI 入口**

- Web 使用同一 API，不绕过 CLI/Action 契约。

**Step 3: Commit**

```bash
git add scripts/menu.sh scripts/cli/compat_notice.sh public/control-panel tests/cli/compat_notice_spec.sh
git commit -m "chore(compat): keep web support and demote legacy menu to compatibility mode"
```

### Task 7: 安全、审计与可观测性

**Files:**
- Modify: `scripts/api/auth.sh`
- Create: `scripts/control/audit.sh`
- Create: `docs/specs/audit-events.md`
- Test: `tests/security/cli_api_auth_spec.sh`

**Step 1: 动作级审计日志**

- 记录来源（cli/api）、命令、结果码、耗时、调用者。

**Step 2: 统一认证策略**

- Web/API 写操作鉴权。
- 本地 CLI 默认可执行，远程执行需显式授权策略。

**Step 3: Commit**

```bash
git add scripts/api/auth.sh scripts/control/audit.sh docs/specs/audit-events.md tests/security/cli_api_auth_spec.sh
git commit -m "feat(security): add unified auth and action-level audit for cli/web"
```

### Task 8: 发布迁移、CI 与回滚

**Files:**
- Create: `docs/migration/llm-first-cli-web-hybrid.md`
- Create: `.github/workflows/cli-api-contract-ci.yaml`
- Modify: `.github/workflows/test.yaml`
- Test: `tests/migration/hybrid_rollout_spec.sh`

**Step 1: 新增 CI 质量门禁**

- CLI 契约测试、API 契约测试、Parity 测试必须通过。

**Step 2: 迁移与回滚**

- 灰度开关：`control_plane_mode=hybrid|web|legacy`。
- 失败自动回滚到 `hybrid` 安全模式。

**Step 3: Commit**

```bash
git add docs/migration/llm-first-cli-web-hybrid.md .github/workflows/cli-api-contract-ci.yaml .github/workflows/test.yaml tests/migration/hybrid_rollout_spec.sh
git commit -m "release(hybrid): add rollout, rollback, and contract ci for llm-first cli + web"
```

## CLI 设计要点（必须落地）

- `--output json` 机器可读优先。
- `--yes`/`--non-interactive` 消除确认阻塞。
- `--from-json` 支持 agent 批量调用。
- `--dry-run` 输出变更计划。
- 稳定错误码 + 稳定字段名 + 稳定 stdout/stderr 语义。

## 推荐命令形态（示例）

- `crash system status --output json`
- `crash system restart --yes --output json`
- `crash config get --key db_port --output json`
- `crash config set --key dns_mod --value mix --yes --output json`
- `crash core switch --to meta --yes --output json`
- `crash onboarding apply --profile router --from-json ./init.json --yes --output json`

## 里程碑建议

- M1（1 周）：CLI 契约 + Action 层 + 基础命令可用。
- M2（1 周）：CLI/API 同构 + Web 对齐 + 初始化一键化。
- M3（3-5 天）：迁移开关、CI 门禁、灰度发布。
