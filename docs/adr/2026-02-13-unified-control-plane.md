# ADR: Unified Control Plane with LLM-First CLI + Web + Wizard

Date: 2026-02-13
Status: Accepted

## Context

ShellCrash currently relies on numeric interactive menus. This is usable for humans but unstable for automation and LLM agents.

## Decision

Adopt a unified control plane with three frontends:
- Web control panel
- Wizard CLI (human guided)
- Non-interactive CLI (LLM/automation friendly)

All three frontends MUST call one shared action layer. Frontends MUST NOT directly mutate system runtime state.

## Consequences

Positive:
- One business path, lower divergence risk
- Stable machine-readable outputs for automation
- Easier parity testing between CLI and API

Tradeoffs:
- Initial refactor cost
- Need a stricter contract and test gate

## Compatibility

Legacy numeric menu remains temporarily as compatibility mode only.
