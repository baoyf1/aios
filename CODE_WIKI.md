# AIOS 项目 Code Wiki

## 1. 项目概述

AIOS（AI Agent Operating System）是一个面向商业化垂直领域的智能操作系统，旨在实现自然语言驱动的意图执行、跨应用自动化、多Agent协作以及本地LLM部署。

### 1.1 核心目标
- 自然语言驱动的意图执行
- 跨应用自动化
- 多Agent协作
- 本地LLM部署（Ollama/国内厂商）

### 1.2 技术栈

| 层级 | 技术 |
|------|------|
| 系统层 | C/C++ (多线程/进程/IPC) |
| Agent开发 | Python + LangChain + LangGraph |
| LLM | Ollama / 国内LLM / OpenAI兼容 |
| OS集成 | Win32 API / Linux D-Bus |

## 2. 架构设计

AIOS采用5层架构设计，参考Linux操作系统的分层思想，实现高内聚低耦合的系统架构。

### 2.1 5层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 5: Application Layer     (应用层)                              │
│  ─────────────────────────────────────────────────────────────────  │
│  • Central Agent (PID 0) — 任务调度 + Agent Registry               │
│  • Domain Agents — 领域专家 (FileAgent, SysAgent, NetAgent...)     │
│  • Function Modules — 功能模块 (文件/系统/网络/记忆/LLM)           │
│  • LangChain/LangGraph — 工作流编排 (内嵌在Domain Agent中)         │
└─────────────────────────────────┬─────────────────────────────────────┘
                                  │ ▲ 向上解包 / 向下封装
┌─────────────────────────────────┴─────────────────────────────────────┐
│  Layer 4: Middleware Layer      (中间件层)                            │
│  ─────────────────────────────────────────────────────────────────  │
│  • 消息序列化 (JSON/Protobuf)                                        │
│  • 会话管理 (Session Manager)                                        │
│  • Python/Go 绑定 (如有需要)                                         │
│  注意: LangChain/LangGraph 已移入 Domain Agent 内部                  │
└─────────────────────────────────┬─────────────────────────────────────┘
                                  │ ▲
┌─────────────────────────────────┴─────────────────────────────────────┐
│  Layer 3: System Call / Driver  (系统调用/驱动层)                      │
│  ─────────────────────────────────────────────────────────────────  │
│  • 驱动模块 (Drivers) — FS/Net/LLM/Msg/Mem/Proc                    │
│  • 权限守卫 (PathGuard + CapChecker + AuditLog)                     │
│  • 系统调用分发 (DriverManager)                                       │
│  • 协议转换 (上层请求 → 底层硬件操作)                                 │
└─────────────────────────────────┬─────────────────────────────────────┘
                                  │ ▲
┌─────────────────────────────────┴─────────────────────────────────────┐
│  Layer 2: Kernel Layer           (内核层)                              │
│  ─────────────────────────────────────────────────────────────────  │
│  • 中央Agent (Agent-0, PID 0角色)                                     │
│  • O(1)调度器 (优先级数组 + 位图)                                    │
│  • 内存管理 (per-agent memory system)                                 │
│  • 进程管理 (spawn/kill/wait)                                        │
│  • IPC总线 (进程间通信)                                               │
│  • 安全沙箱 (权限控制, 审计日志)                                       │
└─────────────────────────────────┬─────────────────────────────────────┘
                                  │ ▲
┌─────────────────────────────────┴─────────────────────────────────────┐
│  Layer 1: Hardware Layer        (硬件层)                                │
│  ─────────────────────────────────────────────────────────────────  │
│  • 外部LLM API (OpenAI/Claude/MiniMax) — 依赖外部API Key            │
│  • 本地向量嵌入 (Ollama + nomic-embed-text) — 记忆系统专用           │
│  • 消息网关 (飞书开放平台API) — 接收/发送消息                        │
│  • 文件系统 (本地磁盘, 网络存储)                                      │
│  • HTTP Client (外部API调用)                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心设计原则

1. **底层不关心上层**
   - Layer 1-3 不感知 Domain Agent 的业务逻辑
   - 通过系统调用和 Driver 接口进行交互

2. **领域自治**
   - Domain Agent 独立管理自己的 LangGraph 和 Function Modules
   - Central Agent 只负责调度，不管具体实现

3. **严格分层**
   - 跨层通信必须通过邻层接口
   - 禁止跨层直接调用

4. **可追溯**
   - 每个请求都有 trace_id
   - 贯穿整个调用链

5. **权限最小化**
   - Agent 只能访问自己工作目录
   - 共享资源需要显式授权

## 3. 核心组件

### 3.1 中央Agent (Central Agent)

中央Agent是整个系统的核心，负责：
- 意图分析与解析
- 任务调度与分发
- Agent注册与管理
- 结果整合与返回

```python
# 中央Agent核心实现
class CentralAgent:
    def __init__(self, config: AgentConfig):
        self.config = config
        self.llm = LLMFactory.create(
            provider=config.provider,
            model=config.model,
            api_key=config.api_key
        )
        self.registry = AgentRegistry()
        self.intent_parser = IntentParser()
        self.memory = MemorySystem()
        
    async def process(self, user_input: str) -> Response:
        # 1. 解析意图
        intent = await self.intent_parser.parse(user_input)
        
        # 2. 规划任务
        tasks = self.planner.plan(intent)
        
        # 3. 分发执行
        if len(tasks) == 1:
            result = await self._execute_single(tasks[0])
        else:
            result = await self._execute_multi(tasks)
        
        # 4. 整合结果
        return self.integrator.format(result)
```

### 3.2 Domain Agents

Domain Agents是领域专家，负责特定领域的任务处理：

| Agent名称 | 领域 | 能力 |
|---------|------|------|
| FileAgent | file | list/read/write/move/search |
| SysAgent | system | run_cmd/env/proc/list |
| NetAgent | network | http_get/post/dns/... |
| MailAgent | mail | send/read/list/search |
| DataAgent | data | query/transform/report |

每个Domain Agent都包含：
- LangChain/LangGraph工作流
- 相关Function Modules
- 领域特定逻辑

### 3.3 Function Modules

Function Modules是具体的功能实现模块：

| 模块名称 | 功能 | 方法 |
|---------|------|------|
| file_ops | 文件操作 | list_directory, read_file, write_file, move_file, delete_file, search_files, get_file_info |
| system_ops | 系统操作 | run_command, get_env, list_processes, get_system_info |
| network_ops | 网络操作 | http_get, http_post, dns_lookup |
| memory_ops | 记忆操作 | memory_search, memory_write, memory_read |
| llm_ops | LLM操作 | chat, embed, stream_chat |

### 3.4 C++ 核心运行时

C++核心运行时负责：
- Agent生命周期管理
- IPC通信
- 内存管理
- 安全控制
- Python桥接

```cpp
// C++核心类
class AIOS {
public:
    AIOS(const Config& config);
    ~AIOS();
    
    // 启动
    void start();
    void stop();
    
    // Agent管理
    void register_agent(const std::string& name, py::object agent);
    void unregister_agent(const std::string& name);
    py::object get_agent(const std::string& name);
    
    // 任务提交
    std::string submit_task(const Task& task);
    TaskResult get_result(const std::string& task_id);
    
    // 状态
    AIOSStatus status() const;
    
private:
    std::unique_ptr<IPCBus> ipc_bus_;
    std::unique_ptr<Scheduler> scheduler_;
    std::unique_ptr<AgentRegistry> registry_;
    std::unique_ptr<Security> security_;
    std::unique_ptr<PyBridge> py_bridge_;
};
```

## 4. 项目结构

```
aios/
├── aios_core/                    # C++ 核心运行时
│   ├── include/
│   │   ├── aios.h               # 主头文件
│   │   ├── ipc_bus.h            # IPC总线
│   │   ├── scheduler.h          # 调度器
│   │   ├── agent_registry.h     # Agent注册表
│   │   ├── security.h           # 安全模块
│   │   └── py_bridge.h         # Python桥接
│   │
│   ├── src/
│   │   ├── main.cc             # 入口
│   │   ├── ipc_bus.cc
│   │   ├── scheduler.cc
│   │   ├── agent_registry.cc
│   │   ├── security.cc
│   │   └── py_bridge.cc        # Python调用C++
│   │
│   └── CMakeLists.txt
│
├── aios_langchain/             # Python LangChain模块
│   ├── __init__.py
│   │
│   ├── core/                   # 核心Agent
│   │   ├── central_agent.py    # 中央Agent
│   │   ├── base_agent.py        # Agent基类
│   │   └── intent_parser.py     # 意图解析
│   │
│   ├── agents/                  # Domain Agents
│   │   ├── doc_agent.py
│   │   ├── mail_agent.py
│   │   ├── code_agent.py
│   │   ├── note_agent.py
│   │   ├── media_agent.py
│   │   ├── data_agent.py
│   │   └── search_agent.py
│   │
│   ├── sys_agents/             # 系统Agents
│   │   ├── file_agent.py
│   │   ├── sys_agent.py
│   │   ├── net_monitor_agent.py
│   │   ├── mem_agent.py
│   │   ├── log_agent.py
│   │   └── security_agent.py
│   │
│   ├── tools/                  # 工具
│   │   ├── __init__.py
│   │   ├── file_ops.py
│   │   ├── system_ops.py
│   │   ├── network_ops.py
│   │   ├── memory_ops.py
│   │   ├── llm_ops.py
│   │   └── registry.py          # 工具注册表
│   │
│   ├── langgraph/              # LangGraph工作流
│   │   ├── workflow.py
│   │   ├── nodes.py
│   │   └── executor.py
│   │
│   └── pyproject.toml
│
├── cli/                        # 命令行界面
│   ├── main.py                 # CLI入口
│   ├── commands.py              # 命令定义
│   ├── interactive.py           # 交互模式
│   └── output.py               # 输出格式化
│
├── interfaces/                 # 界面适配层
│   ├── api_server.py           # API Server
│   ├── feishu_adapter.py      # 飞书适配器
│   └── web_adapter.py          # Web界面适配器
│
├── config/
│   ├── config.yaml             # 主配置
│   └── agents/                 # Agent配置
│       ├── doc_agent.yaml
│       ├── mail_agent.yaml
│       └── ...
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── examples/
│   └── quickstart.py
│
├── docs/
│   ├── design.md
│   └── api.md
│
├── pyproject.toml
├── requirements.txt
└── README.md
```

## 5. 开发计划

### 5.1 Phase 1: CLI核心 (v0.2.x)

| 阶段 | 周数 | 任务 |
|------|------|------|
| Phase 1.1: C++ Runtime | Week 1 | CMake项目搭建、IPC Bus实现、Agent Registry、Python Bridge |
| | Week 2 | CentralAgent Python端、Intent Parser、基本Agent调用、CLI入口 |
| Phase 1.2: Domain Agents | Week 3 | DocAgent (LangGraph)、file_ops工具、memory_ops工具 |
| | Week 4 | MailAgent、CodeAgent、SearchAgent |
| Phase 1.3: CLI完善 | Week 5 | 交互模式、输出格式化、帮助系统、配置文件加载 |

### 5.2 Phase 2: API Server (v0.3.x)

| 阶段 | 周数 | 任务 |
|------|------|------|
| Phase 2 | Week 6 | FastAPI/Gradio Server、REST API定义、WebSocket支持 |
| | Week 7 | 认证/授权、API文档、SDK封装 |

### 5.3 Phase 3: 界面适配 (v0.4.x)

| 阶段 | 周数 | 任务 |
|------|------|------|
| Phase 3 | Week 8 | 飞书适配器、消息收发、回调处理 |
| | Week 9-10 | Web界面、VSCode Plugin |

## 6. 核心 API

### 6.1 AIOS 核心 API

| 方法 | 描述 | 参数 | 返回值 |
|------|------|------|-------|
| start() | 启动AIOS | 无 | 无 |
| stop() | 停止AIOS | 无 | 无 |
| register_agent() | 注册Agent | name: str, agent: py::object | 无 |
| unregister_agent() | 注销Agent | name: str | 无 |
| get_agent() | 获取Agent | name: str | py::object |
| submit_task() | 提交任务 | task: Task | task_id: str |
| get_result() | 获取任务结果 | task_id: str | TaskResult |
| status() | 获取状态 | 无 | AIOSStatus |

### 6.2 CentralAgent API

| 方法 | 描述 | 参数 | 返回值 |
|------|------|------|-------|
| process() | 处理用户输入 | user_input: str | Response |
| _execute_single() | 执行单个任务 | task: Task | Result |
| _execute_multi() | 执行多个任务 | tasks: List[Task] | List[Result] |

### 6.3 BaseAgent API

| 方法 | 描述 | 参数 | 返回值 |
|------|------|------|-------|
| _build_graph() | 构建工作流图 | 无 | StateGraph |
| execute() | 执行任务 | task: Task | Result |
| _prepare_state() | 准备状态 | task: Task | dict |

### 6.4 CLI 命令

| 命令 | 描述 | 参数 |
|------|------|------|
| ask | 向AIOS提问 | prompt: str, --stream |
| chat | 启动交互式对话 | --interactive, -i |
| agents | 列出所有Agent | 无 |
| info | 查看Agent详情 | agent_name: str |

## 7. 配置系统

### 7.1 主配置文件

```yaml
# config/config.yaml

# ==================== LLM配置 ====================
llm:
  provider: "openai"
  api_key: "${OPENAI_API_KEY}"
  model: "gpt-4o"
  temperature: 0.7
  max_tokens: 2000

# ==================== Agent默认配置 ====================
agents:
  defaults:
    provider: "${llm.provider}"
    model: "${llm.model}"
    api_key: "${llm.api_key}"
    timeout: 120

# ==================== 应用层Agent ====================
app_agents:
  DocAgent:
    enabled: true
    tools: [file_ops, memory_ops]
    
  MailAgent:
    enabled: true
    tools: [network_ops]
    
  CodeAgent:
    enabled: true
    tools: [file_ops, system_ops]

# ==================== 系统层Agent ====================
sys_agents:
  FileAgent:
    enabled: true
    schedule: "0 3 * * *"  # 每天凌晨3点
    
  MemAgent:
    enabled: true
    schedule: "0 3 * * *"

# ==================== CLI配置 ====================
cli:
  prompt: "[AIOS]> "
  stream: true
  history_size: 100
```

### 7.2 Agent配置文件

每个Agent可以有自己的配置文件，位于 `config/agents/` 目录下。

## 8. 运行方式

### 8.1 构建

```bash
# 克隆仓库
git clone https://github.com/your-org/aios.git
cd aios

# 构建C++核心
mkdir build && cd build
cmake .. && make -j$(nproc)

# 安装Python依赖
cd ..
pip install -e ".[dev]"

# 运行
aios run --config config/config.yaml
```

### 8.2 CLI使用

```bash
# 交互模式
aios chat -i

# 单次问答
aios ask "帮我整理桌面文件"

# 列出Agent
aios agents

# 查看Agent信息
aios info DocAgent
```

### 8.3 API Server

```bash
# 启动API服务器
aios api --host 0.0.0.0 --port 8000

# 访问API文档
http://localhost:8000/docs
```

## 9. 多Agent协作

AIOS支持多Agent协作，通过Central Agent协调不同Domain Agent之间的工作。

### 9.1 协作流程

1. 用户输入："把桌面PDF发到张总邮箱"
2. Central Agent分析意图，识别需要FileAgent和MailAgent
3. 调用FileAgent列出桌面PDF文件
4. FileAgent处理完成后，将结果传递给MailAgent
5. MailAgent发送邮件（带附件）
6. Central Agent整合结果并返回给用户

### 9.2 上下文链

每个请求都有一个唯一的trace_id，贯穿整个调用链，确保可追溯性。

## 10. 第三方Agent开发

AIOS支持第三方Agent开发，开发者可以按照以下步骤创建自定义Agent：

### 10.1 Agent模板

```python
# my_agent/agent.py
from aios_langchain import BaseAgent

class MyAgent(BaseAgent):
    def __init__(self, config: AgentConfig):
        super().__init__("MyAgent", "custom", config)
    
    def _build_graph(self):
        # 定义工作流
        graph = StateGraph(dict)
        graph.add_node("process", self._process)
        graph.add_edge("process", "__end__")
        graph.set_entry_point("process")
        return graph.compile()
    
    async def _process(self, state: dict) -> dict:
        # 处理逻辑
        result = f"Processed: {state['task'].input}"
        state['result'] = result
        return state
```

### 10.2 配置文件

```yaml
# my_agent/config.yaml
name: MyAgent
domain: custom
provider: "claude"           # 可覆盖默认
api_key: "${MY_API_KEY}"
model: "claude-3-opus"

tools:
  - file_ops
  - network_ops

workflow: "my_agent/workflow.yaml"
```

## 11. 测试计划

### 11.1 单元测试

```python
# tests/unit/test_doc_agent.py
import pytest
from aios_langchain.agents import DocAgent

@pytest.fixture
def doc_agent():
    return DocAgent.from_config("config/agents/doc_agent.yaml")

@pytest.mark.asyncio
async def test_organize(doc_agent):
    task = Task(input={"path": "/tmp", "action": "organize"})
    result = await doc_agent.execute(task)
    assert result.status == "success"
```

### 11.2 集成测试

```bash
# tests/integration/test_cli.py
def test_ask_command():
    result = subprocess.run(
        ["aios", "ask", "hello"],
        capture_output=True,
        text=True
    )
    assert result.returncode == 0
    assert "hello" in result.stdout.lower()
```

## 12. 项目状态

| 版本 | 状态 | 完成情况 |
|------|------|----------|
| v0.2.0 | ✅ | C++ Runtime完成、IPC Bus实现、Python Bridge、基本CLI |
| v0.2.1 | ✅ | DocAgent、FileAgent系统工具 |
| v0.2.2 | ✅ | MailAgent、CodeAgent、SearchAgent |
| v0.2.3 | ✅ | 交互模式完善、配置系统、帮助系统 |
| v0.3.0 | 🔲 | API Server、REST API、WebSocket |
| v0.4.0 | 🔲 | 飞书适配器、Web界面 |

## 13. 未来规划

1. **企业级特性**：用户认证、权限管理、审计日志
2. **扩展集成**：更多第三方服务集成、插件系统
3. **性能优化**：缓存系统、并行处理、资源管理
4. **多语言支持**：多语言Agent、国际化
5. **边缘部署**：轻量级部署、离线运行

## 14. 总结

AIOS是一个创新的AI操作系统项目，通过5层架构设计实现了高内聚低耦合的系统结构。它结合了C++的性能优势和Python的开发效率，使用LangChain/LangGraph实现了灵活的工作流编排，支持多Agent协作和本地LLM部署。

项目目前处于开发阶段，已经完成了核心功能的实现，包括C++运行时、基本Agent和CLI界面。未来将继续完善API服务器和多界面支持，为企业用户提供更加全面的AI操作系统解决方案。