---
title: "S009 · DeepSeek Harness 架构解析：从 Coding Agent 到 Agent OS"
type: source
publish: true
article-number: S009
tags:
  - AI 与智能系统
source-type: 文章
author:
url: https://blog.anionex.me/archives/deepseek-harness-agent-os
created: 2026-08-23
updated: 2026-08-25
---

# DeepSeek Harness 架构解析：从 Coding Agent 到 Agent OS

> 编号：S009 · 主题：AI 与智能系统

## 摘要

DeepSeek Harness（DSH）把模型、工具、策略、存储、上下文、界面乃至 Agent Loop 都纳入统一插件运行时，并通过 Profile、Preset、Capability Provider、Session Event Log 和动态插件机制组织 Agent。它展示了 Coding Agent 从固定工具外壳走向可声明、可替换、可分层、可观测且可局部自修改的 Agent Runtime，但插件契约、分发、隔离和长会话重组语义仍未成熟。

## 要点

> Everything is a plugin，包括Loop。

最近鄙人一直在研究DeepSeek Harness，也就是DSH。

一开始看到“一切皆插件”的时候，我以为它大概就是插件系统做得比较彻底，工具、MCP、Skill这些东西都可以往里面装。继续翻了一圈仓库文档，发现它连Agent Loop也放进了插件系统。

Model、Tool、Policy、Storage、Context、UI，甚至Agent Loop本身，都被它放进了同一套插件运行时里面。到这里已经超出“工具很多的Coding Agent”这个范围了。

再加上Profile、Agent Preset、Session Log和运行时自修改这些东西，它已经有一点从Coding Agent走向Agent Runtime，甚至Agent OS-like架构的味道了。

![DeepSeek Harness架构解析封面](https://blog.web-of-anion.top/upload/dsh-slide-01.png)

先说明一下，“Agent OS”只是我用来理解DSH的一个架构类比，官方目前没有这样命名它。

> 注：DSH正在公测并持续迭代，细节或许会有细微出入

### Model很重要，但是Harness能改变很多

平时讨论Coding Agent，大家首先比较的往往是模型。GPT、Claude、Gemini、DeepSeek，哪个模型代码能力更强，哪个推理更好，哪个上下文更长。

这些当然重要。但是同一个模型装进不同Coding Agent里面，最终表现出来的能力差异可能很大。为什么？

因为模型主要决定基础智能，Harness决定这个模型实际能看到什么、能做什么、能记住什么，以及什么时候必须停下来。

![Model与Harness的区别](https://blog.web-of-anion.top/upload/dsh-slide-02.png)

具体一点：

- System Prompt决定它用什么身份理解任务。
- Tool Catalog决定它能不能读取文件、修改代码、运行Shell、浏览网页。
- Context Manager决定哪些历史信息还能继续进入模型。
- Session和Memory决定它能记住什么。
- Sandbox、Approval和Policy决定哪些操作可以执行。
- Agent Loop决定模型、工具与用户之间如何不断往返。

外围Harness也是Coding Agent产品壁垒的一部分。

### 传统Coding Agent大概是什么样的

传统Coding Agent通常有一条相对固定的控制流程：

```text
用户输入
→ 固定Agent Loop
→ 固定System Prompt和Tool Catalog
→ 模型调用工具
→ 文件、Shell、Git、浏览器
→ 工具结果返回模型
```

![传统Coding Agent的固定Harness](https://blog.web-of-anion.top/upload/dsh-slide-03.png)

它也可以扩展工具，例如接入MCP、增加Skill、安装插件。

但是很多时候，扩展范围集中在外围工具。Agent Loop怎么运行、权限怎么插入、状态怎么保存、UI怎么显示，仍然属于固定核心。

这不一定是坏事。固定核心更容易保证体验，也更容易维护。

但是如果想把同一个Harness改造成只读调研Agent、完整开发Agent、视觉Agent、自动化Agent，或者一个可以调用其他Coding Agent的编排器，改起来就容易碰到核心边界。

DSH把这些部分也逐步拆进了插件系统。

### 一切皆插件，包括Loop

DSH的架构文档开头直接写了：

> everything is a plugin, including the loop

Tool可以做成插件，Agent运行所需的其他部分也按照相同方式组织。

![DSH的一切皆插件](https://blog.web-of-anion.top/upload/dsh-slide-04.png)

它底层使用的是Cordis。可以先重点理解四个东西：

- `Context`：插件获取和注册能力的上下文。
- `Service`：插件之间共享的能力接口。
- `Event`：插件插入现有流程的扩展点。
- `Effect`：注册行为的生命周期和清理方式。

例如一个插件注册了Tool、Event Listener、Service或者Timer，这些注册会归属于当前插件的Fiber。插件卸载时，对应效果也能跟着清理。

这件事看起来比较底层，但后面的热重载、Agent Scope和 `cordis_unmount`，其实都依赖这个生命周期。

我觉得这里挺有趣。插件已经成了DSH组织系统的基本方式。

### 把DSH拆成六个平面

为了方便理解，我把当前DSH拆成六个平面。

![DSH六个架构平面](https://blog.web-of-anion.top/upload/dsh-slide-05.png)

#### 交互层

用户或者其他程序从这里进入DSH。

当前可以看到Web UI、Headless、ACP、Python SDK和JSON-RPC等入口。

#### 编排层

决定一个DSH进程装载什么，以及某个Agent具体拥有什么能力。

这里主要包括Bundle、Patch、Profile、Agent Preset和Scope。

#### 运行主干

也就是Agent真正开始工作的地方。

Session、Turn、Step和Agent Loop负责不断组装请求、调用模型、执行工具、记录结果，然后进入下一步。

#### 能力层

LLM、Filesystem、Bash、Subprocess、PTY、LSP、Web、Skill、Compaction、Subagent等能力都在这里。

#### 数据平面

Session Event Log、Persistence、Query、Projection、Telemetry都属于这一层。

#### 动态运行时

负责检查当前插件树、动态挂载临时插件、卸载临时插件，以及运行中的扩展。

这六层是我根据当前文档整理的理解方式，仓库没有给出同名的固定分层。用它解释DSH的各部分还算顺手。

### Agent Loop退回到运行主干

DSH用Session、Turn、Step组织一次Agent运行。

![Session、Turn与Step](https://blog.web-of-anion.top/upload/dsh-slide-06.png)

它们大概是这样的：

- `Session`：完整会话，也是可以持久化、恢复和分叉的单位。
- `Turn`：一次用户输入或者系统触发形成的工作轮次。
- `Step`：一次模型请求，以及随后产生的工具执行。

一个Turn里面可以有很多Step。

例如模型先读取文件，这是一个Step。工具结果回来后，模型又执行搜索或者修改代码，再进入下一个Step。一直到模型给出最终回答，这个Turn才结束。

每个Step中，Agent Loop主要做这些事情：

1. 组装System Prompt、Tools和Variables。
2. 记录本次模型请求的Header。
3. 调用LLM Adapter并接收流式输出。
4. 执行模型生成的Tool Calls。
5. 把Assistant内容、`tool/call`和 `tool/result`写入Session。

这里有一个设计方向我比较喜欢：新增功能尽量不要直接修改Agent Loop。

权限、沙箱、压缩、重试、子Agent、Plan Mode这些东西，优先通过Service、Event和Plugin接进去。Loop保持相对稳定，外面的能力继续生长。

有点像把复杂功能从内核里面拿出来。

### Capability Seam

DSH文档里有一个术语叫Capability Seam。

文档把一项完整能力分成三种角色：

1. `Service Definition`：定义能力接口和语义。
2. `Service Provider`：提供具体实现。
3. `Consumer`：使用这个能力，或者把它暴露给模型。

![Capability Seam](https://blog.web-of-anion.top/upload/dsh-slide-07.png)

拿Bash举例。

上层Tool只需要知道怎么调用 `ctx.bash`，至于命令最终通过本机Provider执行，还是经过Sandbox包装，或者它下面的文件与子进程能力已经换成E2B世界，上层Consumer不需要跟着重写。

这个区别挺重要。

如果插件系统只能增加Tool，那么我们只是不断给模型增加新按钮。

有了Service Definition和Provider之后，插件还可以替换按钮下面的实现。

例如：

- 替换模型Provider。
- 替换Session Persistence。
- 替换Filesystem和Subprocess。
- 替换Web Search Provider。
- 替换Compaction策略。
- 替换Subagent Provider。
- 替换Code Runtime。

Tool负责给模型提供调用入口。具体执行方式由Provider决定。

### 工具执行之前还有一条流水线

模型生成一次Tool Call之后，还要经过一条统一的Tool Pipeline。

![DSH工具调用流水线](https://blog.web-of-anion.top/upload/dsh-slide-08.png)

当前文档描述的流程大概包括：

```text
tool/call
→ pre-execute
→ guards
→ approval
→ execute middleware
→ tool body
→ post-execute
→ normalize
→ finalizer
→ tool/result
→ UI presenter
```

这样设计之后，Approval、Sandbox、Hook、Timeout、Metrics和结果改写，可以插入统一执行链，而不需要每个Tool自己重复实现一套。

比如文件修改工具和Shell工具都需要审批，Approval Capability可以统一进入执行流水线，两个Tool不用分别编写一遍交互逻辑。

工具结果还会经过归一化、日志记录和UI Presenter，最终可以在Web界面显示成终端卡片、Diff卡片或者普通结果卡片。

这已经有一点系统调用和中间件管线结合的感觉了。

### Session Log记录完整运行事实

DSH把Session设计成append-only Event Log。

![Session Log事实平面](https://blog.web-of-anion.top/upload/dsh-slide-09.png)

除了用户和Assistant最后看到的消息，里面还会记录：

- 用户输入。
- 模型请求Header。
- 流式Text和Reasoning Block。
- Tool Call和Tool Result。
- Plan、Goal、Todo等状态变化。
- Approval以及其他需要恢复的运行事实。

仓库里面有一条很关键的规则：

> Model-visible ⟺ logged

凡是模型能够看到的内容，都应该可以通过Session Log重建。

为什么要这么麻烦？

因为只有最后一条回答，根本无法恢复一个真实Agent。

当时模型看到的是哪些Tool？System Prompt有没有变化？中间执行过什么命令？哪个结果进入了上下文？这些都会影响模型后续行为。

有了统一事件日志之后，很多能力都能从同一份事实派生出来：

- Resume。
- Fork。
- Query和全文搜索。
- Replay。
- UI Projection。
- Telemetry。

Persistence本身也是可替换Provider，目前文档里面有JSONL和SQLite实现。

我感觉这部分是DSH最容易被“一切皆插件”抢走风头，但实际上非常重要的一层。没有统一日志，后面的动态组合很容易变成不可恢复的玄学状态。

### Profile定义进程

DSH通过Profile装配一个进程的启动配置。

![Profile进程级组合](https://blog.web-of-anion.top/upload/dsh-slide-10.png)

Profile由多层Patch叠加形成，可以粗略理解为：

```text
Bundle Patch
+ Profile Patch
+ 用户目录覆盖
+ --patch参数
+ 启动时Patch
= 最终Cordis配置
```

它适合决定整个进程级别的东西，例如：

- 启动Web UI还是Headless。
- 使用哪些Host Service。
- 装载哪种Persistence和API Gateway。
- 增加外部Plugin Bundle。
- 使用什么安全和环境配置。

所以Profile作用在进程级，决定这一个DSH进程怎么启动。Session里面临时选择工作模式属于另一层。

### Preset定义Agent

只做到进程级Profile其实还不够。

如果一个DSH进程中同时存在调研Agent、开发Agent和自指Agent，总不能为了换一套Tool就重新启动三个服务。

当前仓库已经增加了Agent Preset。

![Agent级Preset与Scope](https://blog.web-of-anion.top/upload/dsh-slide-11.png)

它的查找关系可以概括成：

```text
agent → preset → global
```

近层覆盖远层。

也就是说，Host可以继续提供共享的模型路由、Persistence、Sandbox Policy和基础Registry。但是某个Agent能够看到的Persona、Tool、Skill和Prompt，可以由它挂载的Preset决定。

当前内置了四个Preset：

- `minimal`：最小能力组合。
- `standard`：标准Coding Agent能力。
- `code`：使用Code Mode呈现工具。
- `cordis`：开放运行时检查和临时插件挂载能力。

所以现在更准确的说法是：

大概可以记成：Profile管进程启动，Preset给Agent挂载能力，Scope再处理实际可见范围。

一个Process里面可以同时运行继承不同Preset的Agent，这一点比全局插件开关灵活很多。

### 那么，能不能从只读调研Agent切成开发Agent？

答案是有机会，而且现在已经能看到两条路线。

![调研Agent切换开发Agent的两条路径](https://blog.web-of-anion.top/upload/dsh-slide-12.png)

#### 路线一：Preset Recompose

受信任的Preset可以定义一整套能力组合。

例如research Preset只开放只读文件、Web Search和Session Query，development Preset再加入文件修改、Shell、LSP、测试和Subagent。

当前源码里面已经有 `recompose()`，它会先保证新的Preset成功Mount，然后重新绑定Agent Scope的Parent。

不过目前还不能直接把它理解成“任意长历史Session都能无缝切换”。

当前实现把blank-session语义检查交给调用方。已有Tool history、正在运行的Task、PTY、Approval或者Subagent应该怎么迁移，还需要调用层明确处理。

#### 路线二：临时Plugin Mount

另一条路线是让Agent在运行中临时增加能力。

缺什么就Mount什么，用完再Unmount。

这条路比较快，也比较好玩，但是当前临时插件属于进程内共享状态，不天然等于Session私有能力。

所以我更赞成PPT里面这个分工：

我更倾向于让Preset管理整套可信组合，自修改只处理任务中的临时增量。

如果把几十种开发工具全部注册成临时插件，理论上当然可以模拟整套Harness切换。但是依赖顺序、失败回滚、并发Session干扰、持久化和权限问题，都需要自己接住。

做实验很爽，拿来当稳定配置可能有点猛。

### Agent开始修改自己的外围运行时

`cordis`Preset里面最有意思的是三个工具：

- `cordis_inspect`。
- `cordis_mount`。
- `cordis_unmount`。

![DSH Self-modification](https://blog.web-of-anion.top/upload/dsh-slide-13.png)

Agent可以先检查当前进程里面有哪些Service、Plugin Fiber、Tool、API和Event。

发现能力不够之后，它可以生成一段JavaScript，返回一个临时Cordis Plugin，再把这个Plugin挂进当前运行时。

这个临时插件可以注册新的Tool、Service、Prompt Contribution或者Listener。后面的Turn就能使用这些新能力。

任务结束后，再通过 `cordis_unmount`释放它拥有的注册和效果。

流程大概是：

```text
inspect
→ generate
→ mount
→ use
→ unmount
```

热加载插件本身不算特别稀奇。

更好玩的是，检查和插件管理能力也暴露给了模型。这样Agent也能参与调整自己的部分运行时组成。

当然，这里有几条限制必须摆出来：

- Tool Cordis是opt-in能力。
- 临时插件只存在于当前进程内存，重启就消失。
- 它不会自动写入Profile，也不会自动发布成正常插件。
- `cordis_unmount`只能卸载 `cordis_mount`创建的临时插件。
- 临时插件可能影响同一进程中的其他Session。
- VM只提供有限隔离，官方文档建议按照Bash权限看待。

最后一条非常关键。

所以这个功能适合当成一套受信任的运行时实验工具。它没有给不可信模型代码提供完整的安全沙箱。

### E2B替换的是执行世界

Capability Provider可以替换某个工具的实现，也可以替换一整组上层能力实际作用的环境。

![Host Plane、Agent Plane与Execution World](https://blog.web-of-anion.top/upload/dsh-slide-14.png)

当前仓库中的E2B POC提供了远程 `ctx.fs`和 `ctx.subprocess`实现。

上层Bash、PTY和LSP依赖这些抽象Service，所以不用分别维护一份E2B专用Fork。替换底层Provider之后，它们的可变工作就能进入同一个E2B Sandbox。

整个DSH仍然留在Host侧，包括Harness进程、Cordis对象、模型调用、Agent和Session状态、Session Persistence。

当前E2B POC替换的是Agent执行操作所依赖的FS和Subprocess环境。PPT里面把它简称为Execution World。

ps：PPT里面为了方便理解，把Execution World画成FS、Subprocess、Network和I/O的完整区域。当前仓库的E2B POC明确实现的是FS与Subprocess这两个基础适配器，不能直接推断所有网络能力也已经跟着迁移。

### Code Mode：让模型编写程序调用工具

普通Tool Calling通常是模型和工具不断往返。

模型调用Tool A，拿到结果之后再次推理，然后调用Tool B，再拿结果，再调用Tool C。

![DSH Code Mode](https://blog.web-of-anion.top/upload/dsh-slide-15.png)

Code Mode把工具目录通过生成的TypeScript SDK暴露给模型，模型主要调用一个保留工具 `run_code`，在程序里面连续调用多个工具。

```ts
const a = await tool_a(...)
const b = await tool_b(a)
const c = await tool_c(b)
return c
```

这样做的好处比较直接：

- 循环、条件和数据处理可以在程序里面完成。
- 多次模型与工具往返可以压缩成一次程序运行。
- 中间结果不用全部重新塞进模型上下文。
- 只有程序输出回到模型。

内部工具调用仍然经过原来的Policy和Log路径，没有绕过Harness。

这个设计有点像让模型编写一段小程序，再由程序连续调用系统能力。

### 一个Runtime可以服务很多界面

同一个DSH Runtime可以接入Web UI、CLI和其他调用端。

![DSH的多种交互界面](https://blog.web-of-anion.top/upload/dsh-slide-16.png)

当前可以看到这些Surface：

- Web UI。
- Headless运行模式。
- ACP自动化协议。
- Python SDK和JSON-RPC。
- 浏览器Client Modules。
- TypeRT和API Gateway。

其中Client Module比较有意思。

一个插件可以给Host增加Service和Tool，也可以携带浏览器侧Bundle来扩展Web UI。需要前后端联动的插件，就不用把所有交互压成纯文本Tool Result。

例如会话回退、可视化工具、设置页面、自定义Tool Card，这些功能天然需要Host和Browser两边一起扩展。

不同Surface可以复用同一个Agent Runtime和Session事实，省掉每种界面各自实现Agent的工作。

这也是我觉得它开始有平台味道的原因之一。

### 为什么我觉得它像Agent OS

把DSH与传统OS的几个概念放在一起，会出现一组挺工整的对应关系。

![DSH与传统OS概念映射](https://blog.web-of-anion.top/upload/dsh-slide-17.png)

|传统OS概念|DSH里面可以类比的东西|
|---|---|
|Kernel|Runtime Spine、Agent Loop|
|Driver|Capability Provider|
|User Space|Preset、Tool、Skill、Persona|
|Process|Agent、Subagent、Workflow Worker、Task|
|Journal / Log|Session Event Log|
|Dynamic Module|Cordis Plugin、mount、unmount|

当然，这里只是借用OS概念帮助理解。DSH不负责硬件、内存和CPU调度，和Linux这种通用操作系统差得很远。

这里说的Agent OS，是指它开始系统化管理Agent运行所需的东西：

- 执行主干。
- 能力Provider。
- 进程级与Agent级组合。
- 权限和审批。
- Session持久化。
- 子Agent与工作单元。
- 多种交互Surface。
- 动态插件装载。

传统Coding Agent主要让模型在一个既定Harness里面工作。

DSH进一步把Harness本身变成可声明、可替换、可分层、可观测，甚至可以局部自修改的对象。

我感觉“从Coding Agent到Agent OS”的核心，大概就在这里。

### 目前还不能吹太早

架构确实很有意思，但是当前现实也很明确。

![DSH的结论与风险](https://blog.web-of-anion.top/upload/dsh-slide-18.png)

#### 还在Internal testing

仓库README现在仍然标注Internal testing，功能和接口可能变化。现在看到的Package边界、CLI参数和Preset内容，都不一定是最终版本。

#### Plugin Contract还需要稳定

一切皆插件的上限由插件Contract决定。

如果Service、Event、配置格式和Client Module接口频繁变化，外部生态很难积累。插件开发者不可能每周把仓库全部跟着重写一次。

#### 分发链路还在形成

插件怎么安装、怎么发布、怎么被Hub发现、依赖包如何获取，这些东西会直接影响生态能不能长起来。

架构再灵活，没有低摩擦的分发也很难形成规模。

#### 动态组合需要更清楚的隔离语义

Preset已经提供Agent级组合，但是Recompose仍有blank-session约束。

临时Plugin Mount很灵活，但是当前有shared process state和Bash级信任问题。

如果以后要实现一个Agent在长Session中不停切换完整Harness，还需要继续定义：

- 正在运行的Tool和Task怎么处理。
- PTY与Subagent归谁。
- Tool history如何保持可解释。
- Prompt和Tool Schema改变后如何恢复。
- 动态能力是否能做到Session私有。
- 权限边界由谁保证。

这些问题都不算小。

### 写在最后

鄙人研究完这一圈之后，印象最深的是它对外围Harness的处理。原来固定在Coding Agent里面的很多部分，现在可以参与组合和扩展了。

Profile、Preset、Capability Provider、Session Log和Cordis工具各自处理了一部分。组合、执行、记录和临时扩展都能找到对应位置。

如果这套Plugin Contract后面能够稳定下来，再长出插件分发和一批高质量生态，DSH也许可以拿来构建各种不同的Coding Agent。至于最后会不会走到这一步，现在还不太好说。

目前还早，后面见分晓吧。

如果文中哪里理解错了，也欢迎佬友们不吝赐教。

## 我的思考

DSH 最值得复用的不是“一切皆插件”这句口号，而是围绕稳定运行主干建立清晰扩展缝隙：能力接口与实现分离，策略统一进入工具流水线，模型可见事实全部写入事件日志，进程配置、Agent 能力和执行环境分别管理。这样既能扩展 Harness，又能保留恢复、审计和权限控制所需的确定性。

动态自修改只有在生命周期、隔离范围和信任边界明确时才适合作为生产能力；否则灵活性会转化为共享状态干扰、恢复困难和权限风险。

## 相关笔记

- [[可扩展 Agent Harness 的架构原则]]
- [[工具无法替代知识内化]]
- [[从运行轨迹构建 AI 评估体系]]

