---
title: "N015 · Agentic RAG 六步工作流"
type: note
publish: true
article-number: N015
tags:
  - AI 与智能系统
created: 2026-08-25
updated: 2026-08-25
---

# Agentic RAG 六步工作流

> 编号：N015 · 主题：AI 与智能系统

## 摘要

Agentic RAG 可以理解为一条带质检的研究流水线：先判断任务复杂度并规划检索，再通过查询改写和并行检索收集证据，由充分性检查决定是否继续搜索，最后只基于已验证的证据生成答案。

## 正文

### 1. Root / Orchestrator

它是入口控制器。

**输入：**用户原始问题。

它要先判断三件事：

1. 这是不是一个简单问题？
2. 是否需要查企业知识库？
3. 是否需要多步检索、多数据源或比较推理？

如果问题是“公司的报销政策是什么”，可能直接走普通 RAG。

如果问题是“上季度投诉最多的三个客户分别对应哪些产品缺陷，修复状态如何”，它会判断这是复杂任务，因为里面有多个实体、多个数据源、多个推理步骤。

输出通常是：

```json
{
  "task_type": "complex_rag",
  "needs_planning": true,
  "goal": "answer customer complaint/product defect/fix status question",
  "constraints": ["use enterprise sources", "cite evidence", "avoid unsupported claims"]
}
```

它本质上决定：要不要启动后面的 agentic RAG 流程。

### 2. Planner Agent

Planner 负责把“我要回答什么”变成“我应该怎么查”。

**输入：**Orchestrator 给出的任务目标。

它会做任务分解和数据源选择。

比如用户问：

> 患者出院后需要注意哪些药物、饮食限制，以及住院期间有没有过敏反应？

Planner 会拆成：

```text
子任务 A：查出院药物
子任务 B：查饮食限制
子任务 C：查住院期间不良反应或过敏记录
```

如果是企业跨库场景，它还会判断查哪些 corpus：

```json
{
  "subtasks": [
    {
      "name": "discharge medications",
      "target_corpus": "clinical_notes"
    },
    {
      "name": "dietary restrictions",
      "target_corpus": "discharge_summary"
    },
    {
      "name": "allergic reactions during stay",
      "target_corpus": "nursing_notes"
    }
  ]
}
```

Google 文中强调 cross-corpus retrieval：Planner 要从多个语料库里选正确来源，而不是盲搜全部。

### 3. Query Rewriter

Query Rewriter 负责把人的问题改写成适合检索的查询。

它不是简单换同义词，而是把每个子任务变成多个检索角度。

例如子任务是：

> 查住院期间有没有过敏反应。

它可能生成：

```text
patient allergic reaction during hospitalization
rash adverse event inpatient note
medication allergy nursing note
urticaria itching swelling hospital stay
```

为什么要这样做？

因为原始问题里的“过敏反应”可能不会在文档里直接写成 allergy，可能写成 rash、hives、itching、adverse reaction、reaction to medication。

输出：

```json
{
  "rewritten_queries": [
    "allergic reaction during admission",
    "rash adverse event nursing note",
    "medication reaction inpatient record"
  ]
}
```

这一步的质量直接影响召回率。

### 4. Search Fanout / RAG Agent

这一步是真正去查。

**输入：**改写后的多个 query，以及 Planner 选中的 corpus。

它会并行检索多个来源：

```text
query 1 -> clinical_notes
query 2 -> nursing_notes
query 3 -> discharge_summary
query 4 -> medication_admin_records
```

每个检索结果通常会带上：

```json
{
  "chunk": "Patient developed rash after cefazolin...",
  "source": "nursing_notes",
  "document_id": "note_123",
  "timestamp": "2025-04-16",
  "score": 0.84
}
```

这里有一个关键点：Search Fanout 不是只找 top-k 一次。它会把多个查询同时发出去，扩大覆盖面，然后把片段交给后面的判断器。

普通 RAG 常见问题是：只拿“最相似”的片段。但复杂问题里，最相似不一定最完整。

### 5. Sufficient Context Agent

这是整套系统最关键的一步。

它不是回答问题，而是判断：

> 当前证据是否足够回答原问题？

它会检查：

1. 原问题有几个要求？
2. 每个要求是否都有证据？
3. 证据之间是否冲突？
4. 有没有关键字段缺失？
5. 草稿答案是否有没被证据支持的内容？

例如当前找到了：

```text
出院药物：有
饮食限制：有
过敏反应：没有找到
```

它不会让系统直接生成答案，而是输出缺口：

```json
{
  "sufficient": false,
  "missing": [
    "Evidence about allergic or adverse reactions during hospitalization"
  ],
  "follow_up_queries": [
    "rash during hospital stay",
    "adverse medication reaction inpatient",
    "allergy noted in nursing notes"
  ]
}
```

然后流程回到 Query Rewriter 或 Search Fanout，继续查。

如果证据够了，它输出：

```json
{
  "sufficient": true,
  "covered_items": [
    "discharge medications",
    "dietary restrictions",
    "allergic reaction evidence"
  ]
}
```

Google 这篇文章真正强调的就是这个“充分性判断”。相关内容不等于充分内容。很多 RAG 幻觉就是因为系统拿到了一点相关材料，就急着回答。

### 6. Synthesis Agent

最后才生成答案。

**输入：**已经通过 sufficient context 检查的证据集。

它要做三件事：

1. 把多个来源的信息合并。
2. 保留证据边界。
3. 不补没有证据的内容。

例如输出不是：

> 患者没有过敏反应。

而是：

> 出院总结和护理记录中未发现明确过敏反应记录；护理记录显示患者在用药后出现皮疹，因此应视为可能的不良反应并进一步核实。

好的 Synthesis Agent 会避免把“没检索到”说成“事实不存在”。

工程上，它的 prompt 往往会要求：

```text
Only answer using provided evidence.
Cite source chunks.
If evidence is missing or conflicting, say so.
Do not infer beyond the retrieved context.
```

所以最终答案会更像：

```text
根据出院总结，患者需继续服用 A、B 两种药物。
根据营养师记录，患者需低钠饮食。
关于过敏反应，护理记录提到用药后出现皮疹，但未明确标记为药物过敏；建议由医生确认。
```

### 整体循环

更准确的流程不是 1 到 6 直线走完，而是这样：

```text
用户问题
  -> Orchestrator 判断复杂度
  -> Planner 拆任务、选数据源
  -> Query Rewriter 生成检索查询
  -> Search Fanout 并行检索
  -> Sufficient Context 判断够不够
      -> 不够：带着缺口继续改写和检索
      -> 够了：进入 Synthesis
  -> 最终回答
```

### 工程总结

Orchestrator 决定“要不要启动复杂流程”；Planner 决定“查什么、去哪里查”；Query Rewriter 决定“怎么查”；Search Fanout 执行“并行查”；Sufficient Context Agent 决定“证据够不够”；Synthesis Agent 只在证据足够时负责“组织成答案”。

## 参考资料

- Google Research: [Unlocking dependable responses with Gemini Enterprise Agent Platform's Agentic RAG](https://research.google/blog/unlocking-dependable-responses-with-gemini-enterprise-agent-platforms-agentic-rag/)
- Google Cloud: [Cross-corpus retrieval](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/rag-engine/cross-corpus-retrieval)

## 相关笔记

- [[从运行轨迹构建 AI 评估体系]]
