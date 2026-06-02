# Better Harness - 评估驱动的 Agent 框架自进化与动态热补丁调优深度剖析

`better-harness` 是 Deep Agents monorepo 中技术思想极具颠覆性的**元智能（Meta-Intelligence）**设计示例。该系统参考了 LangChain 官方的“Harness 工程化”（Harness Engineering）思想与 Karpathy 的学术级 Auto-Research。它展现了如何通过“套娃（Meta-Harness）”的设计：**让一个高阶的“外部 Agent”（Outer Agent）通过多轮的 Baseline 测试与测试集跑分，自主且动态地去修改、重构另一个“内部 Agent”（Inner Agent）的提示词（Prompt）、工具文件（Tools）以及中间件（Middleware）源码，从而实现框架层级的自进化调优**。

---

## 🎯 核心使用场景与设计目的

在调优一个 Agent 应用（如客服机器人、代码生成器）时，提示词工程与中间件开发常常陷入盲目的“人工试错（Trial and Error）”深渊：
- **微小改动引发全局崩塌**：修改了 System Prompt 中的一个单词，可能修好了 Case A，但却导致先前跑通的 Case B 发生大面积 Regression 报错。
- **高昂的人工评估成本**：开发者每次改完代码，需要手动点几十次测试并肉眼比对输出，工作极其枯燥低效。

`better-harness` 提出了**评估跑分驱动的自编辑环（Eval-driven Self-optimization Loop）**：
1. **Outer & Inner Agent Split (双级分工)**：外部 Agent 是优化器，能看到训练集（Train Split）的报错日志和当前 Agent 源码；内部 Agent 是真实上线的业务 Agent。外部 Agent 通过自编辑生成一个“Proposer (候选方案)”。
2. **Dynamic Hot Patching (动态热补丁机制)**：通过 `module_attr`（内存属性拦截替换）和 `workspace_file`（物理工作区文件动态临时替换），在不破坏主开发分支的前提下，在测试运行环境中动态“打上”候选补丁。只有当测试用例通过率（Pass Count）纯增时，才保留该修改，否则直接 Discard 回滚。

---

## 🏗️ 架构与控制流

```mermaid
graph TD
    User([启动调优任务]) --> Optimizer[better-harness 调优主控]
    
    subgraph "元优化双环 (Meta-Optimization Loop)"
        Optimizer -->|1. 运行 Baseline| Runner[Pytest 评估器]
        Runner -->|收集失败用例| Logs[报错日志与 Trace 痕迹]
        Logs -->|2. 提供给外部模型| Outer[外部 Agent<br>Claude-Sonnet]
        Outer -->|3. 自行修改内部 Agent 源码| Proposed[自编辑候选方案<br>Proposer]
        
        Proposed -->|4. 应用内存/物理热补丁| Patch[patch_from_env() 动态热补丁拦截]
        Patch -->|5. 跑分测试集| Runner
    end

    Runner -->|测试通过率提升| Accept[6. Accept: 固化修改, 生成最终 Agent]
    Runner -->|通过率未提升| Discard[7. Discard: 回滚方案, 进入下一轮探索]
```

---

## 💻 核心控制机制剖析

系统支持两类极为高超的**热补丁（Patching）**加载方式，使得外部 Agent 可以随心所欲地控制内部 Agent 的运行时：

### 1. `module_attr` (内存级别属性热拦截)
外部 Agent 直接编辑纯文本形式的 Prompt 表面，Better Harness 底层会在测试启动前动态将其 Patch 到指定的内存属性上（例如 `deepagents.graph:BASE_AGENT_PROMPT`）：
```toml
[surfaces.prompt]
kind = "module_attr"
target = "my_agent.graph:BASE_PROMPT" # 运行时自动替换 my_agent.graph 模块下的 BASE_PROMPT 属性
filename = "prompt.txt"
base_value = """
You are a helpful assistant.
"""
```

### 2. `workspace_file` (物理工作区临时文件覆盖)
当外部 Agent 修改了复杂的 `middleware.py` 时，系统在跑测试前，会将生成的临时文件瞬时覆盖拷贝至目标路径下，执行完毕后再还原，完美保障了主开发区代码的安全：
```toml
[surfaces.middleware_impl]
kind = "workspace_file"
target = "my_agent/middleware.py"
filename = "middleware.py"
base_file = "middleware.py"
```

### 3. Pytest 插件端点对接机制 (`better_harness_plugin.py`)
为了在 `pytest` 运行中物理拦截并覆盖环境，Better Harness 在 `conftest.py` 中挂载了如下插件：
```python
# conftest.py
# 导入 Better Harness 提供的 patch_from_env
from better_harness import patch_from_env

# 跑任何 pytest 案例之前，自动提取 env 中的 Proposer 补丁并强行注入内存
patch_from_env()
```

---

## 🛠️ Project Showcase & 实战复用指南

如果您在为您的企业开发**高精度、强容错的金融/医疗 Agent 系统，需要让 AI 每天深夜自我跑跑分、自主优化 System Prompt 以逼近 100% 正确率**，可以直接复用以下元优化器跑分控制脚本：

### 1. 简易 Auto-Prompt 优化器骨架

```python
# file: custom_meta_optimizer.py
import os
import subprocess
import shutil
from langchain_anthropic import ChatAnthropic

class AgentHarnessOptimizer:
    def __init__(self, target_workspace: str, test_suite_path: str):
        self.workspace = target_workspace
        self.test_suite = test_suite_path
        self.outer_model = ChatAnthropic(model="claude-sonnet-4-6", temperature=0.2)
        
        # 约定可编辑的 Prompt 表面物理路径
        self.prompt_surface_path = os.path.join(target_workspace, "prompt.txt")
        self.backup_path = os.path.join(target_workspace, "prompt.txt.bak")

    def _run_evals(self) -> int:
        """物理执行 pytest 测试，并返回通过的测试用例个数"""
        result = subprocess.run(
            ["pytest", self.test_suite, "-q", "--tb=short"],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        # 从末尾统计通过数
        output = result.stdout
        print(f"[Eval Output]: {output.splitlines()[-1] if output.splitlines() else 'Error'}")
        
        # 极简解析: 寻找 'passed' 的用例个数
        passed_count = 0
        if "passed" in output:
            try:
                # 解析例如 '3 passed, 1 failed in 2.3s'
                passed_count = int(output.split("passed")[0].split()[-1])
            except Exception:
                passed_count = 0
        return passed_count

    def start_optimize_loop(self, iterations: int = 3):
        # 1. 备份当前的初始 Prompt
        shutil.copyfile(self.prompt_surface_path, self.backup_path)
        
        # 2. 跑 Baseline 拿到起始分数
        baseline_score = self._run_evals()
        print(f"[Baseline] 初始通过测试用例数: {baseline_score}")
        
        best_score = baseline_score
        
        for i in range(1, iterations + 1):
            print(f"\n--- [Iteration {i} / {iterations}] 启动自动调优 ---")
            
            # 读取当前 Prompt 和最近一次 pytest 的报错（在真实系统里可以抓取 pytest stdout 作为报错输入）
            with open(self.prompt_surface_path) as pf:
                current_prompt = pf.read()
                
            # 3. 询问外部“教练”大模型，针对报错提出更优的 Prompt 候选方案
            coach_query = (
                f"你是一个高级 Agent 调优教练。当前我们的内部 Agent 的 System Prompt 如下：\n"
                f"```\n{current_prompt}\n```\n"
                f"在上一轮测试中，我们通过的测试用例数为 {best_score} 个。\n"
                f"请分析如何微调、补充或限制这个 Prompt 才能进一步提升通过率？\n"
                f"请直接给出经过修改后的、完整的全新 System Prompt 文本，严禁夹带任何多余废话！"
            )
            
            response = self.outer_model.invoke([("user", coach_query)])
            proposed_prompt = response.content.strip()
            
            # 4. 动态应用候选方案（物理写入 prompt.txt）
            with open(self.prompt_surface_path, "w") as pf:
                pf.write(proposed_prompt)
                
            # 5. 重新跑分评估
            new_score = self._run_evals()
            print(f"[Optimizer] 候选 Prompt 跑分结果: {new_score}")
            
            # 6. 决策评估：纯增通过率则 Accept 固化；否则 Discard 回滚
            if new_score > best_score:
                print(f"[Accept] 成功！通过率由 {best_score} 提升至 {new_score}。固化修改！")
                best_score = new_score
            else:
                print(f"[Discard] 失败！候选方案并未提升正确率（当前: {new_score}，历史最佳: {best_score}）。执行回滚。")
                shutil.copyfile(self.backup_path, self.prompt_surface_path)

if __name__ == "__main__":
    # 使用指南
    workspace = "./target_bot"
    tests = "./target_bot/tests"
    
    os.makedirs(tests, exist_ok=True)
    
    # 模拟放置初始 prompt.txt
    with open(os.path.join(workspace, "prompt.txt"), "w") as f:
        f.write("You are an assistant. Answer user question in lowercase.")
        
    # 只要您在 tests 目录下建有标准的 pytest 案例，即可立刻启动元优化循环！
    # optimizer = AgentHarnessOptimizer(workspace, tests)
    # optimizer.start_optimize_loop(max_iterations=3)
```

**复用提示**：
- **工业级自闭环价值**：很多企业在上线 Agent 机器人后，会配置专门的标注人员（Labeler）去收集用户投诉和 Failed Case。利用 Better Harness 的工程思想，您可以把每天收集到的真实 Bug Case 放入 `tests/` 目录作为评估集，由后台的 `better-harness` 每天深夜自主微调、评估、比对，自动打出高得分的 Prompt 和中间件热补丁上线。这让您的 Agent 具备了类似于软件 CI/CD 自动更新的、无止境自我进化的超级生命力。
