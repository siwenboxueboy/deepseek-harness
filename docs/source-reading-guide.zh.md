# DeepSeek Harness 源码阅读路线

这份文档面向希望通过 DeepSeek Harness 学习 **Agent Runtime / Agent Harness 架构** 的开发者。

目标不是按目录把整个仓库读完，而是先建立一条稳定的主干认知：

> 一个用户消息进入 Harness 后，如何经过 Agent → LLM → Tool → 再回到 Agent，并在 Session 中形成可持久化、可恢复的多 Step Turn。

---

## 1. 先建立主调用链

第一轮阅读只追这一条链：

```text
User Message
    ↓
Agent.followup() / steer()
    ↓
ReactLoopAgent.wakeDriver()
    ↓
kick()
    ↓
turn()
    ↓
preStep()
    ↓
step()
    ↓
prepareRequest()
    ↓
buildRequest()
    ↓
LLM.stream()
    ↓
Assistant Message
    ↓
executeToolCalls()
    ↓
Tool Result
    ↓
下一次 step()
```

主实现位于：

```text
packages/core/agent-loop/src/agent.ts
```

第一遍不要陷入取消、重试、KV Cache、system prompt replacement 等分支，先把正常路径串通。

---

## 2. 阅读顺序

### 阶段一：只理解 Cordis 的最小模型

先读：

```text
docs/cordis-primer.zh.md
```

只需要先理解 5 个概念：

```text
Context = Service 容器
Plugin  = 向 Context 贡献能力
inject  = 声明插件依赖哪些 Service
event   = 模块之间的扩展 / 拦截点
effect  = 有生命周期、可撤销的注册
```

此阶段不要直接钻进 `vendor/cordis`。

学习 Harness 时，先知道 Cordis 如何被使用，比先理解 Cordis 内部 Fiber 如何实现更重要。

---

### 阶段二：阅读总架构

再读：

```text
docs/architecture.zh.md
```

重点只看两部分：

- Core packages
- Turn flow

核心主干可以先记成：

```text
session
   ↓
system-prompt
   ↓
agent
   ↓
agent-loop
   ↓
llm
   ↓
tools
```

其中非常重要的一条设计原则是：

> Session Log 是模型可见上下文的持久事实来源。

也就是说，Harness 不是简单维护一个内存里的 `history: Message[]`，而是从 durable session events 投影出模型当前能看到的 surface，再派生出请求历史。

---

### 阶段三：先看 Agent 接口，再看 Agent Loop

先读：

```text
packages/core/agent/README.zh.md
packages/core/agent/src/types.ts
packages/core/agent/src/index.ts
```

这一阶段回答这些问题：

- Agent 对外暴露什么能力？
- AgentRegistry 管什么？
- Agent 是谁创建的？
- Agent 生命周期由谁拥有？
- 为什么业务方依赖 `ctx.agents`，而不是直接依赖具体 Agent Loop？

需要特别关注这个解耦关系：

```text
AgentRegistry
     │
     │ setFactory()
     ↓
AgentFactory        ← 公共接口
     ↑
     │ 实现
AgentLoop
```

外部消费者通过：

```ts
ctx.agents.create(...)
```

创建 Agent，而不需要知道底层是否是 `ReactLoopAgent`。

---

### 阶段四：精读 Agent Loop 主干

然后进入最重要的文件：

```text
packages/core/agent-loop/README.zh.md
packages/core/agent-loop/src/agent.ts
```

第一遍只按下面的方法调用关系往下追：

```text
ReactLoopAgent
     ↓
followup()
     ↓
send()
     ↓
wakeDriver()
     ↓
kick()
     ↓
turn()
     ↓
preStep()
     ↓
step()
     ↓
prepareRequest()
     ↓
buildRequest()
```

可以这样理解这些方法：

| 方法 | 核心职责 |
| --- | --- |
| `kick()` | Agent driver 总循环 |
| `turn()` | 驱动一个完整 Turn |
| `preStep()` | Claim 输入、组装 prompt/runtime context、执行 pre-step 扩展点 |
| `step()` | 完成一次 LLM 请求、Assistant settle、Tool 调用 |
| `prepareRequest()` | 解析 provider/model，并绑定 LLM adapter |
| `buildRequest()` | 从 durable surface 派生最终模型请求 |

建议第一遍只回答：

> 一个普通用户消息，在没有异常、没有取消、没有压缩的情况下，到底如何完成一次 Tool Calling Loop？

---

## 3. 再拆 Session

主循环理解后，再读：

```text
packages/core/session/README.zh.md
docs/subsystems/session.zh.md
```

重点观察 Agent Loop 如何持续写入事件：

```text
turn/start
step/start
system/message
user/message
assistant/message
tool/call
tool/result
step/end
turn/end
```

这里的核心思想不是简单的：

```text
history.append(message)
```

而更接近：

```text
Append-only Session Event Log
            ↓
        Projection
            ↓
   Model-visible Surface
            ↓
      deriveMessages()
            ↓
          LLM
```

阅读时重点思考：

- 为什么模型可见信息必须落到 durable log？
- 为什么 UI 实时流和最终 durable assistant message 要分开？
- 为什么 Tool Call / Result 也必须形成稳定事件？
- Resume / Fork / Replay 为什么都依赖同一份 Session Log？

---

## 4. 再拆 Tool Pipeline

然后读：

```text
packages/core/tools/README.zh.md
docs/tool-execution-pipeline.md
```

Tool 执行不是简单调用 `tool.execute()`，而是一条完整 Pipeline：

```text
Tool Call
   ↓
tools/pre-execute
   ↓
guard
   ↓
tools/execute
   ↓
Tool.execute()
   ↓
tools/post-execute
   ↓
finalizeContent
   ↓
tools/result
```

这里重点学习：

- schema 与真正 execute 实现为什么分离
- allow / deny / ask policy 如何插入
- timeout / retry / metrics 为什么适合放在 wrapper/event 层
- 工具失败为什么通常不应该直接结束整个 Turn
- Tool Result 如何重新进入 Session 和下一次 Step

---

## 5. 最后再看 Context / Compaction

理解 Agent Loop、Session、Tools 后，再进入：

```text
packages/compaction/
docs/subsystems/compaction.zh.md
```

不要预期在 DSH 中找到一个单独承担全部上下文逻辑的 `ContextBuilder`。

它的模型上下文是多个能力共同形成的：

```text
System Prompt
      +
Session Surface
      +
Runtime Context
      +
Tool Schemas
      +
Compaction
      ↓
Model Request
```

尤其要注意：

> Compaction 是可插拔 Capability，而不是 Agent Loop 的硬编码主干。

它通过类似 `agent/pre-step` 的扩展点介入运行过程。

这种设计很适合拿来和“Agent 内部直接调用 ContextManager.prepare()”的架构做对比。

---

## 6. 第一轮可以先忽略的目录

为了避免被大型 monorepo 带偏，第一轮可以先跳过：

```text
apps/desktop
apps/web
website
native
benchmarks
experimental
python SDK
browser-use
computer-use
agent-team
大量 tests
```

这些模块后续都值得看，但不是理解 Agent Runtime 主干的前置条件。

---

## 7. 和典型自研 Agent 做对照

如果已经自己实现过一个基础 Agent，可以这样建立映射：

| 典型自研 Agent | DeepSeek Harness |
| --- | --- |
| `Agent.run()` | `ReactLoopAgent.kick() / turn() / step()` |
| `LLM.invoke()` | `ctx.llm.prepareCall() / stream()` |
| `ToolExecutor` | `executeToolCalls()` + `ctx.tools` Pipeline |
| `ContextBuilder` | system prompt + Session Surface + runtime context + request build |
| `ConversationHistory` | Session Event Log + Projection |
| `ConversationRepository` | Session Persistence |
| `SummaryManager` | `ctx.compaction` |
| `SessionManager` | `ctx.sessions` |
| 手工依赖组装 | Cordis Context + Service + inject |

最值得体会的结构变化是：

```text
常见简单 Agent：

Agent
 ├─ LLM
 ├─ ToolExecutor
 ├─ ContextManager
 └─ Parser
```

而 Harness 更接近：

```text
                    Cordis Context
                          │
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
      ctx.llm         ctx.tools       ctx.sessions
         ↑                ↑                ↑
         └──────── ReactLoopAgent ─────────┘
                         │
                  extension events
                         │
           ┌─────────────┼─────────────┐
           ↓             ↓             ↓
      compaction       policy      telemetry
```

这里最关键的变化是：

> Agent 不再拥有所有能力；Agent Loop 主要负责驱动状态机，其余能力通过 Service、Capability 和 Event 组合进 Runtime。

---

## 8. 推荐的第一轮学习范围

第一轮只读这几个入口即可：

```text
docs/cordis-primer.zh.md
        ↓
docs/architecture.zh.md
        ↓
packages/core/agent/README.zh.md
        ↓
packages/core/agent-loop/README.zh.md
        ↓
packages/core/agent-loop/src/agent.ts
```

完成标准不是“读完文件”，而是能自己画出：

```text
kick
 ↓
turn
 ↓
preStep
 ↓
step
 ↓
prepareRequest
 ↓
buildRequest
 ↓
llm.stream
 ↓
tool calls
 ↓
next step
```

当这条主链能够脱离源码讲清楚后，再进入 Session、Tools 和 Compaction，学习效率会高很多。
