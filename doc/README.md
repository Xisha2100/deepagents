# Deep Agents 示例指南与核心架构解析

欢迎来到 Deep Agents 核心示例与集成参考指南！本目录包含了对 `deepagents` 官方 monorepo 中所有 15 个代表性示例代码的深度剖析与实战复用模板。这些文档由浅入深，覆盖了从极简检索到企业级异构多模型沙盒、递归并行 REPL 等各种高级模式。

---

## 🗺️ Deep Agents 核心组件全景图

在深入具体示例之前，我们需要理解 `create_deep_agent` 的核心组件与执行链。Deep Agents 建立在 **LangGraph** 运行时之上，通过一套预置的“中间件（Middleware）”来组装出高水平的自主运行能力。

```mermaid
graph TD
    User([用户输入]) --> Orchestrator[主 Agent (create_deep_agent)]
    
    subgraph "Deep Agents 核心中间件栈 (Middleware Stack)"
        Orchestrator --> Todo[TodoListMiddleware<br>任务清单管理]
        Todo --> Skills[SkillsMiddleware<br>按需加载 SKILL.md]
        Skills --> FS[FilesystemMiddleware<br>读/写/改/Grep/Glob 授权操作]
        FS --> Sub[SubAgentMiddleware<br>通过 task 工具调用同步子 Agent]
        Sub --> AsyncSub[AsyncSubAgentMiddleware<br>通过 start_async_task 等工具调用异步子 Agent]
        AsyncSub --> Memory[MemoryMiddleware<br>加载 AGENTS.md 写入 System Prompt]
        Memory --> HITL[HumanInTheLoopMiddleware<br>审批中断机制]
    end
    
    HITL --> LLM{大语言模型调用<br>Claude / OpenAI / Nemotron}
    LLM --> |返回文本或工具调用| ToolEx[ToolExclusionMiddleware<br>隐藏被排除的工具]
    ToolEx --> |工具调用物理执行| Backend[存储与沙盒后端<br>StateBackend / FilesystemBackend / Modal]
    Backend --> |更新状态| Orchestrator
```

---

## 📂 示例分类与选型矩阵

我们已将 15 个官方示例按其设计模式与实际应用场景分为 5 大类别，您可以根据自己项目的具体需求，直接查阅对应的深度剖析文档。

### 1. 🔍 检索与数据库类 (Retrieval & DB)
通过专门的搜索或数据接入工具，提升 Agent 在垂直领域的幻觉抑制能力。
*   👉 **[01_deep_research.md](./01_deep_research.md) (深度网络检索与反思)**
    *   *技术重点*：Tavily web search、Think/Reflect 反思工具、Orchestrator + Researcher 双角色分工。
*   👉 **[02_text_to_sql_agent.md](./02_text_to_sql_agent.md) (自然语言转 SQL 专家)**
    *   *技术重点*：`SQLDatabaseToolkit`、多技能协同（表结构探索、SQL 生成、验证）、`FilesystemBackend` 数据持久化。
*   👉 **[06_deploy_mcp_docs_agent.md](./06_deploy_mcp_docs_agent.md) (MCP 实时知识库检索)**
    *   *技术重点*：Model Context Protocol (MCP) 服务集成、`tools.json` 全局调用、零幻觉知识问答。

### 2. 🚀 云端服务与商业部署 (Cloud & Deployable Services)
如何在云端生产环境部署 Deep Agents，包括多用户数据隔离、账户鉴权与混合编排。
*   👉 **[03_deploy_coding_agent.md](./03_deploy_coding_agent.md) (LangSmith 沙盒编程 Agent)**
    *   *技术重点*：`agent.json` 声明式部署、LangSmith Sandbox 隔离安全执行环境、编码/设计规约注入。
*   👉 **[04_deploy_content_writer.md](./04_deploy_content_writer.md) (Supabase Auth 与多用户隔离记忆)**
    *   *技术重点*：低代码集成 Supabase JWT 鉴权、基于路径 `/memories/user/` 的独立用户记忆隔离。
*   👉 **[05_deploy_gtm_agent.md](./05_deploy_gtm_agent.md) (GTM 多级同步/异步子级 Agent 级联)**
    *   *技术重点*：多层嵌套 Agent 协同、`subagents/` 目录结构自适应加载、同步/异步混合执行。

### 3. 📦 打包分发与知识协同 (Distribution & Collaboration)
如何将 Agent 打包为标准文件夹或利用 Context Hub 协同演进。
*   👉 **[07_downloading_agents.md](./07_downloading_agents.md) (以文件夹存在的打包分发模式)**
    *   *技术重点*：`AGENTS.md` + `skills/` 免代码打包设计、zip 解压即用、`deepagents-cli` TUI REPL 模式。
*   👉 **[08_llm_wiki.md](./08_llm_wiki.md) (Context Hub 增量知识维基)**
    *   *技术重点*：`langsmith hub` 命令深度集成、增量式 Ingest/Query/Lint 循环流水线、自维护 `log.md` 时间线。
*   👉 **[15_content_builder_agent.md](./15_content_builder_agent.md) (YAML 驱动的自组装多媒体生成器)**
    *   *技术重点*：外部 YAML 定义子 Agent、Gemini 绘图工具联动、Rich TUI 动态进度追踪。

### 4. 🔄 高级循环与多模型协同 (Heterogeneous Models & Loops)
利用更先进的模型分工与自主控制流，实现自我进化的超级 Agent。
*   👉 **[09_nvidia_deep_agent.md](./09_nvidia_deep_agent.md) (Nemotron 异构模型与 GPU 沙盒)**
    *   *技术重点*：Anthropic 编排器 + NVIDIA Nemotron 检索器、Modal GPU (RAPIDS cuDF/cuML) 物理沙盒、自我更新技能文件 (`SKILL.md`)。
*   👉 **[10_ralph_mode.md](./10_ralph_mode.md) (Ralph 极简无状态自主循环模式)**
    *   *技术重点*：Geoff Huntley 的 Ralph Looping 模式、利用外部文件系统与 Git 作为永久记忆以规避 Token 上限。

### 5. ⚡ 并行、递归 REPL 与微服务 (Parallel, Recursive & Microservices)
当任务可以进行树状分解、并且能通过代码解释器进行多路并发扇出时的终极解决方案。
*   👉 **[11_repl_swarm.md](./11_repl_swarm.md) (QuickJS 异步并发 Swarm 架构)**
    *   *技术重点*：`langchain_quickjs` 并行工具调用（PTC）、TypeScript 技能定义、快速多任务分发。
*   👉 **[12_rlm_agent.md](./12_rlm_agent.md) (递归多级 REPL 级联)**
    *   *技术重点*：PTC 并行执行 + `CompiledSubAgent` 深度级联、多层 REPL 运行时隔离。
*   👉 **[13_async_subagent_server.md](./13_async_subagent_server.md) (FastAPI Agent Protocol 服务化)**
    *   *技术重点*：遵循 Agent Protocol 规范的 FastAPI 服务、带认证请求头的自托管异步子 Agent 实现。
*   👉 **[14_better_harness.md](./14_better_harness.md) (Better Harness 控制器自动调优优化器)**
    *   *技术重点*：Agent 自主修改另一个 Agent 的提示词/中间件、`module_attr` / `workspace_file` 热补丁动态测试、train/holdout 自动评估循环。

---

## 🛠️ 项目实战复用与选型决策指南

当您准备在自己的项目中开发自主 Agent 时，可以参考以下选型表：

| 如果您的需求是... | 推荐参考的示例 | 推荐理由与核心复用组件 |
| :--- | :--- | :--- |
| **需要高安全性的复杂网络检索与反思** | **[01_deep_research](./01_deep_research.md)** | 可直接 copy `tavily_search` 工具以及 `think_tool` 决策流。 |
| **需要极强的数据查询、统计或报表生成** | **[02_text_to_sql_agent](./02_text_to_sql_agent.md)** | 整合 `SQLDatabaseToolkit` 并复用其 SQL 编写与表探索技能。 |
| **为公司内多用户部署服务，需要数据隔离** | **[04_deploy_content_writer](./04_deploy_content_writer.md)** | 复制其 Supabase Auth 的配置方式，以极低的成本解决用户隔离存储难题。 |
| **需要处理上万字的大文件或超长周期任务** | **[10_ralph_mode](./10_ralph_mode.md)** | Ralph Looping 的“单次清空 Context，靠文件保存状态”是规避大模型幻觉与 Token 瓶颈的最佳工业级方案。 |
| **希望通过代码编写与子 Agent 组合来大幅加速处理** | **[11_repl_swarm](./11_repl_swarm.md)** 与 **[12_rlm_agent](./12_rlm_agent.md)** | 引入 `CodeInterpreterMiddleware` 与 JavaScript/TypeScript 技能包，利用 PTC (Parallel Tool Calling) 扇出子任务。 |
| **需要自托管微服务化的子 Agent 算力** | **[13_async_subagent_server](./13_async_subagent_server.md)** | 复制其 FastAPI 的 Agent Protocol 实现，以标准微服务形式对外暴露您的 Agent 能力。 |

请直接点击上方的具体文档链接，开始您的 Deep Agents 复用开发之旅！
