---
name: hermes-memory
description: 操作 hermes-agent-memories 仓（AX Graph 外部 base 实例）—— 与 axgraph 一致的多 Layer TOML 体系，字段可扩充但原则不可冲突（`number` 是本仓唯一的字段扩充示例）。触发词：hermes memory、写一条 AX 记忆、第 N 条记忆、number 是 X、查 hermes 记忆、AGENTS.md numbering。
version: 4.0
metadata:
  hermes:
    tags: [axgraph, memory, hermes-agent]
    related_skills: [axgraph-external-base-bootstrap]
---

# hermes-memory —— 操作 hermes-agent-memories 仓

> 命令、schema、铁律、base 多库体系、写入流程 —— **全部见 axgraph AGENTS.md**。本 skill 只讲本仓与 axgraph 默认 base 不同的地方。

## 本仓 base

- 仓根：`/root/projects/hermes-agent-memories/`
- base 目录：`.axgraph/base-dir-hermes-agent-memories/`
- 当前只有 `Layer-longlife.toml` —— **沿用 axgraph 惯例**：未来会自然分裂成多个 `Layer-*.toml`（参考 axgraph AGENTS §6「决定挂哪个文件」）
- `Layer-*.detail.toml` 平级记，**独立处理**（见 axgraph AGENTS §2）

## 字段与原则的关系

- **字段可以扩充** —— 本仓通过 `base.toml` 的 `local_kinds` 思路引入 axgraph 全局 schema 之外的字段（**只动本仓 `base.toml`，不动 axgraph 全局**）
- **原则不可冲突** —— 任何扩充都必须与 axgraph 现有 schema / 铁律 / 节点-边约定兼容

`number` 字段就是「字段可扩充、原则不冲突」的一个具体例子。

## 本仓唯一字段扩充：`number`

> axgraph 全局 schema 没有；本仓通过扩充方式引入。

- 用途：口语「第 N 条记忆」的短锚点
- 位置：`id` 之后、`kind` 之前，正整数
- 规则：全仓唯一、严格递增、严禁跳号 / 重复 / 回填改号
- 与 `id` 关系：`id` 机器字符串不可改 / `number` 口语编号谨慎新增，两者同级不替代
- 完整规则：`numbering-rule.txt`

**头号陷阱**（实测，2026-10-03）：

- `ax 31` 直接报错 → **CLI 只接受 id 字符串或关键词，不接受数字 number**
- 绕路：`grep -B2 -A5 '^number = N ' Layer-*.toml` 拿 id，再 `ax <id>`
- 「查第 N 条」命中节点 = 查 `number = N`，**不要**用 `[[nodes]]` 数组下标凑

## 反例

- 「找第 5 条记忆」→ **不要**用 `[[nodes]]` 数组下标第 5 个元素；用 `number = 5` 字段
- 「这条记忆过时要废弃」→ **不要**直接删节点；建 `SUPERSEDES` 边 + `status="deprecated"`
- 「新增节点 number = 31 但仓里最大 #34」→ **不要**回填改号；number 严格递增，跳号作废
- 「detail.toml 里找 number」→ **不要**；detail.toml 走独立流程，不补 number