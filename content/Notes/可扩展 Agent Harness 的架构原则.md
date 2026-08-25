---
title: "N010 · 可扩展 Agent Harness 的架构原则"
type: note
publish: true
article-number: N010
tags:
  - AI 与智能系统
created: 2026-08-23
updated: 2026-08-25
---

# 可扩展 Agent Harness 的架构原则

> 编号：N010 · 主题：AI 与智能系统

## 摘要

可扩展的 Agent Harness 不应只是不断增加工具，而应把运行主干、能力接口、策略管线、事实日志、配置作用域和执行环境分别管理。插件负责组合与替换能力，稳定内核负责维持运行语义，完整事件日志负责恢复与审计，隔离和权限边界负责约束动态扩展。

## 正文

### Harness 决定模型如何成为 Agent

模型提供基础智能，Harness 决定模型实际能看到什么、能做什么、能记住什么，以及何时必须停止。System Prompt、工具目录、上下文管理、Session、Memory、Sandbox、Approval、Policy 和 Agent Loop 共同决定最终行为，因此外围 Harness 本身也是 Agent 产品能力的一部分。

这与 [[工具无法替代知识内化]] 并不矛盾：Harness 可以扩大模型的信息和行动边界，但不能自动替代模型的理解、判断和推理能力。

### 保持运行主干稳定，把变化放到扩展缝隙

Agent Loop 只负责组织 Session、Turn 和 Step，完成请求组装、模型调用、工具执行与结果记录。权限、沙箱、压缩、重试、子 Agent 和计划模式等能力通过 Service、Event 和 Plugin 接入，避免每增加一项能力就修改主循环。

稳定主干降低了组合复杂度，但前提是扩展接口具有明确、长期稳定的语义。

### 分离能力定义、实现与使用者

一项完整能力可以拆成三种角色：

1. **Service Definition**：定义接口与语义。
2. **Service Provider**：提供本地、远程或沙箱中的具体实现。
3. **Consumer**：调用能力，或者把能力暴露给模型。

这种 Capability Seam 使上层工具不必绑定具体执行方式。替换模型、文件系统、子进程、持久化、搜索、压缩或子 Agent Provider 时，Consumer 可以保持不变。

### 让策略进入统一工具流水线

工具调用应统一经过执行前检查、Guard、审批、中间件、工具主体、执行后处理、结果归一化、收尾和展示。Approval、Sandbox、Timeout、Metrics 和结果改写都在同一条管线中实现，避免每个工具重复编写安全与治理逻辑。

统一管线也保证 Code Mode 或其他工具调用方式不会绕过原有 Policy 和日志路径。

### 用事件日志记录完整运行事实

只保存用户和模型最终看到的消息，无法恢复真实 Agent。模型请求时使用的 Prompt、Tools、Variables，中间的 Tool Call、Tool Result、Approval、Plan 和状态变化都会影响后续行为。

因此可以采用一条约束：模型可见的内容必须已经写入日志。以 append-only Session Event Log 作为事实平面，Resume、Fork、Replay、Query、UI Projection、Telemetry 和基于轨迹的评估才能从同一份记录派生。相关评估方法可参考 [[从运行轨迹构建 AI 评估体系]]。

### 区分进程、Agent 与执行环境

- **Profile** 决定进程如何启动以及装载哪些共享服务。
- **Preset** 决定某个 Agent 拥有的 Persona、Tool、Skill 和 Prompt 组合。
- **Scope** 决定能力的实际可见范围。
- **Execution World** 决定文件和子进程等操作最终发生在哪个环境中。

这些层次分开后，同一个 Runtime 可以承载不同能力的 Agent，也可以在不重写上层工具的情况下，把执行切换到远程沙箱。

### 动态自修改必须有清晰边界

让 Agent 检查、挂载和卸载临时插件，可以按任务临时增加能力，但需要同时定义：

- 临时能力属于进程、Agent 还是 Session。
- 并发 Session 是否会互相影响。
- 正在运行的工具、PTY 和子 Agent 如何迁移。
- Prompt 与 Tool Schema 改变后如何恢复历史。
- 插件失败如何回滚，重启后是否需要恢复。
- 动态代码采用什么信任等级和权限边界。

整套可信能力更适合由 Preset 管理，运行时自修改只承担受控的临时增量。灵活性不能以丢失隔离、恢复和审计能力为代价。

## 相关笔记

- [[DeepSeek Harness 架构解析：从 Coding Agent 到 Agent OS]]
- [[工具无法替代知识内化]]
- [[从运行轨迹构建 AI 评估体系]]
