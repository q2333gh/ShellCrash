# 多步骤配置 UX：同时让人类好用、让 CLI LLM Agent 好用

## 1. 问题背景

传统数字菜单（`1/2/3/0返回`）对人类在终端里是可用的，但对 LLM agent 不友好：

- 需要多轮交互，状态隐式存在于菜单层。
- 输出不稳定，难以机器解析。
- “0 返回”语义依赖当前菜单层级，自动化难推断。

目标不是删掉人类可用性，而是把“交互外壳”和“配置语义”解耦。

## 2. 设计原则（双友好）

1. 单一语义层
- 配置动作只定义一次（Action 层），人类向导和 CLI/API 都调用它。

2. 双入口
- 人类入口：向导式 step UX（可回退）。
- LLM 入口：非交互命令 + JSON 输入输出。

3. 状态显式化
- 不依赖“当前菜单层级”隐式状态。
- 所有步骤状态可查询、可恢复、可重放。

4. 可中断可继续
- 人类和 agent 都能 `save/resume`。

## 3. 推荐 UX 方案

## A. 人类模式（Wizard）

保留“分步骤”体验，但从数字菜单改为“步骤向导 + 命令提示”：

- `crash wizard start`
- `crash wizard next`
- `crash wizard back`
- `crash wizard set <key> <value>`
- `crash wizard review`
- `crash wizard apply`

等价于原来的“0 返回”，但语义更清晰：`back` 是显式动作，不再依赖数字键。

## B. LLM 模式（Non-interactive）

LLM 优先使用一次性命令：

- `crash onboarding apply --profile router --yes --output json`
- `crash config set --key dns_mod --value mix --yes --output json`

或批处理：

- `crash plan apply --from-json ./setup.json --output json`

这避免多轮问答，最适合 agent 编排。

## 4. 把“返回”重设计为可编程动作

把 `0` 的“返回”语义替换为统一动作：

- `back`: 回到上一步
- `cancel`: 取消当前会话
- `resume <session_id>`: 恢复会话
- `reset-step <step_id>`: 重置某一步

这样人类和 LLM 都能理解同一套控制语义。

## 5. 输出规范（关键）

建议默认输出 JSON，文本仅作为辅助：

```json
{
  "ok": true,
  "code": "WIZARD_STEP_UPDATED",
  "session_id": "wiz_01H...",
  "step": "network",
  "next_step": "core",
  "hints": ["run: crash wizard next"]
}
```

并固定退出码：

- `0` 成功
- `2` 参数错误
- `3` 业务校验失败
- `4` 系统执行失败

## 6. 人类与 LLM 的界面分工

1. 人类 UI（CLI/Web）
- Web：首选向导页（步骤条、表单校验、回退按钮）。
- CLI：保留 `wizard` 子命令，显示简短步骤提示。

2. LLM UI（CLI/API）
- CLI：`--output json --yes --non-interactive`。
- API：同一语义 endpoint，返回结构化错误码。

## 7. 多步骤配置推荐模型（状态机）

示例步骤：

1. `detect_env`
2. `network_mode`
3. `core_select`
4. `subscription_input`
5. `policy_options`
6. `security_setup`
7. `preview`
8. `apply`

每一步都有：

- 输入 schema
- 校验规则
- 可回退规则
- side effect（是否写文件/是否仅内存）

## 8. 落地建议（低风险）

阶段 1：先做“同语义双入口”
- 旧菜单继续可用，但内部改为调用 Action 层。

阶段 2：推出新 CLI
- 发布 `wizard` + `onboarding apply`。
- 将数字菜单标记为 legacy。

阶段 3：默认切换
- 默认命令改为新 CLI 提示。
- legacy 菜单通过 `crash legacy-menu` 进入。

## 9. 你这个场景的最终建议

- 人类 UX：保留“多步骤引导”，但把 `0` 改为显式 `back`。
- LLM UX：提供“一次命令完成”与“批处理 JSON”两条主路径。
- 工程实现：CLI/Web/API 共享同一个动作层与状态机，避免维护两套流程。

这样既不会牺牲人类可理解性，也能让 LLM agent 稳定自动化。
