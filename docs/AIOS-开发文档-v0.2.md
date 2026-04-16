# AIOS 开发文档 v0.2

**需求文档**: AIOS 5层架构设计 v2.2
**开发版本**: v0.2
**技术栈**: LangChain/LangGraph + C++
**创建时间**: 2026-04-08

---

## 一、技术栈选择

### 1.1 技术栈

```
┌─────────────────────────────────────────────────────────────────────┐
│                    技术栈选择                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  核心框架:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  LangChain / LangGraph (Python)                           │   │
│  │  - 工作流编排                                              │   │
│  │  - 工具调用                                                │   │
│  │  - Agent抽象                                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  C++ (主进程)                                            │   │
│  │  - Agent生命周期管理                                       │   │
│  │  - IPC通信                                                │   │
│  │  - 内存管理                                                │   │
│  │  - 热加载                                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  CLI (首选界面)                                           │   │
│  │  - 快速验证                                                │   │
│  │  - 便于调试                                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  后续扩展:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  API Server → Web界面                                      │   │
│  │  Feishu适配 → 飞书                                        │   │
│  │  Plugin → VSCode/IDE                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AIOS 架构                                    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    C++ Runtime (aios-core)                   │   │
│  │                                                              │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│  │  │IPC Bus  │  │Scheduler│  │Registry │  │Security │   │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │ ▲                                     │
│                    IPC       │ │       Python/C++ Bridge          │
│                              ▼ │                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 LangChain/LangGraph (Python)                  │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  CentralAgent (主控)                                  │ │   │
│  │  │    - Intent Parser                                   │ │   │
│  │  │    - Task Planner                                    │ │   │
│  │  │    - Result Integrator                                │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                              │                              │   │
│  │                              ▼                              │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  Domain Agents (LangChain Agents)                   │ │   │
│  │  │    - DocAgent / MailAgent / CodeAgent / ...       │ │   │
│  │  │    - 每个Agent一个LangGraph工作流                   │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Tools (Python)                            │   │
│  │                                                              │   │
│  │  file_ops / system_ops / network_ops / memory_ops / llm_ops │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、项目结构

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

---

## 三、开发阶段规划

### 3.1 Phase 1: CLI核心 (v0.2.x)

```
目标: 命令行完整运行

┌─────────────────────────────────────────────────────────────┐
│  Phase 1.1: C++ Runtime                                   │
│                                                              │
│  Week 1:                                                    │
│  - [ ] CMake项目搭建                                        │
│  - [ ] IPC Bus实现                                         │
│  - [ ] Agent Registry                                      │
│  - [ ] Python Bridge                                      │
│                                                              │
│  Week 2:                                                    │
│  - [ ] CentralAgent Python端                               │
│  - [ ] Intent Parser                                       │
│  - [ ] 基本Agent调用                                        │
│  - [ ] CLI入口                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Phase 1.2: Domain Agents                                 │
│                                                              │
│  Week 3:                                                    │
│  - [ ] DocAgent (LangGraph)                               │
│  - [ ] file_ops工具                                        │
│  - [ ] memory_ops工具                                      │
│                                                              │
│  Week 4:                                                    │
│  - [ ] MailAgent                                           │
│  - [ ] CodeAgent                                           │
│  - [ ] SearchAgent                                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Phase 1.3: CLI完善                                        │
│                                                              │
│  Week 5:                                                    │
│  - [ ] 交互模式                                            │
│  - [ ] 输出格式化                                          │
│  - [ ] 帮助系统                                            │
│  - [ ] 配置文件加载                                        │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Phase 2: API Server (v0.3.x)

```
目标: 提供HTTP API

┌─────────────────────────────────────────────────────────────┐
│  Phase 2                                                   │
│                                                              │
│  Week 6:                                                    │
│  - [ ] FastAPI/Gradio Server                              │
│  - [ ] REST API定义                                        │
│  - [ ] WebSocket支持                                       │
│                                                              │
│  Week 7:                                                    │
│  - [ ] 认证/授权                                            │
│  - [ ] API文档                                             │
│  - [ ] SDK封装                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Phase 3: 界面适配 (v0.4.x)

```
目标: 多界面支持

┌─────────────────────────────────────────────────────────────┐
│  Phase 3                                                   │
│                                                              │
│  Week 8:                                                    │
│  - [ ] 飞书适配器                                          │
│  - [ ] 消息收发                                            │
│  - [ ] 回调处理                                            │
│                                                              │
│  Week 9-10:                                                 │
│  - [ ] Web界面                                             │
│  - [ ] VSCode Plugin                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、C++ Runtime 设计

### 4.1 核心组件

```cpp
// ==================== aios_core.h ====================
namespace aios {

// 主类
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

// IPC消息
struct IPCMessage {
    uint64_t id;
    std::string from;
    std::string to;
    MessageType type;
    std::string payload;
    uint64_t timeout_ms;
};

}
```

### 4.2 Python Bridge

```cpp
// ==================== py_bridge.h ====================
namespace aios {

class PyBridge {
public:
    PyBridge();
    ~PyBridge();
    
    // 初始化Python环境
    void init(const std::string& python_path);
    
    // 调用Python Agent
    py::object call_agent(const std::string& agent_name, 
                          const std::string& method,
                          const py::args& args,
                          const py::kwargs& kwargs);
    
    // 获取Python对象
    py::object get_attr(const std::string& module, 
                        const std::string& attr);
    
    // 执行Python代码
    py::object exec(const std::string& code);
    
private:
    py::object scope_;
};

// 模板调用
template<typename... Args>
py::object PyBridge::call(const std::string& agent,
                            const std::string& method,
                            Args&&... args) {
    return call_agent(agent, method, 
        py::make_tuple(std::forward<Args>(args)...),
        py::dict());
}

}
```

### 4.3 IPC Bus

```cpp
// ==================== ipc_bus.h ====================
class IPCBus {
public:
    IPCBus();
    
    // 发送消息
    void send(const IPCMessage& msg);
    
    // 注册处理器
    void register_handler(const std::string& to,
                         std::function<void(const IPCMessage&)> handler);
    
    // 启动
    void start();
    void stop();
    
private:
    int pipe_fd_[2];
    std::unordered_map<std::string, 
        std::function<void(const IPCMessage&)>> handlers_;
    std::thread loop_thread_;
    bool running_;
};
```

---

## 五、LangChain/LangGraph 设计

### 5.1 CentralAgent

```python
# ==================== central_agent.py ====================
from langchain import LLMChain
from langchain.agents import Agent
from langchain.prompts import PromptTemplate

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
    
    async def _execute_single(self, task: Task) -> Result:
        agent = self.registry.get(task.domain)
        return await agent.execute(task)
    
    async def _execute_multi(self, tasks: List[Task]) -> List[Result]:
        # 并行执行
        results = await asyncio.gather(*[
            self._execute_single(t) for t in tasks
        ])
        return list(results)
```

### 5.2 BaseAgent

```python
# ==================== base_agent.py ====================
from langgraph import StateGraph
from abc import ABC, abstractmethod

class BaseAgent(ABC):
    def __init__(self, name: str, domain: str, config: AgentConfig):
        self.name = name
        self.domain = domain
        self.config = config
        self.llm = LLMFactory.create(config)
        self.tools = ToolRegistry.get_tools(config.tools)
        self.graph = self._build_graph()
    
    @abstractmethod
    def _build_graph(self) -> StateGraph:
        """子类实现工作流图"""
        pass
    
    async def execute(self, task: Task) -> Result:
        initial_state = self._prepare_state(task)
        final_state = await self.graph.ainvoke(initial_state)
        return self._extract_result(final_state)
    
    def _prepare_state(self, task: Task) -> dict:
        return {
            "task": task,
            "context": {},
            "memory": self.memory,
            "tools": self.tools
        }
```

### 5.3 DocAgent 示例

```python
# ==================== doc_agent.py ====================
from .base_agent import BaseAgent

class DocAgent(BaseAgent):
    def __init__(self, config: AgentConfig):
        super().__init__("DocAgent", "document", config)
    
    def _build_graph(self) -> StateGraph:
        graph = StateGraph(dict)
        
        # 添加节点
        graph.add_node("parse_intent", self._parse_intent)
        graph.add_node("list_files", self._list_files)
        graph.add_node("filter_files", self._filter_files)
        graph.add_node("organize", self._organize)
        graph.add_node("report", self._report)
        
        # 添加边
        graph.add_edge("parse_intent", "list_files")
        graph.add_edge("list_files", "filter_files")
        graph.add_edge("filter_files", "organize")
        graph.add_edge("organize", "report")
        
        graph.set_entry_point("parse_intent")
        graph.set_finish_point("report")
        
        return graph.compile()
    
    async def _parse_intent(self, state: dict) -> dict:
        prompt = f"分析用户意图: {state['task'].input}"
        intent = await self.llm.agenerate([prompt])
        state['intent'] = intent
        return state
    
    async def _list_files(self, state: dict) -> dict:
        path = state['task'].input.get('path', '/')
        files = await self.tools['list_directory'].execute(path)
        state['files'] = files
        return state
    
    # ... 其他节点
```

---

## 六、CLI 设计

### 6.1 命令行接口

```python
# ==================== cli/main.py ====================
import click
from aios_langchain import AIOS

@click.group()
@click.pass_context
def cli(ctx):
    """AIOS - 智能操作系统级Agent系统"""
    ctx.obj = AIOS.from_config('config/config.yaml')

@cli.command()
@click.argument('prompt')
@click.option('--stream', is_flag=True, help='流式输出')
@click.pass_obj
def ask(ai, prompt, stream):
    """向AIOS提问"""
    result = ai.process(prompt, stream=stream)
    if stream:
        for chunk in result:
            print(chunk, end='', flush=True)
    else:
        print(result)

@cli.command()
@click.option('--interactive', '-i', is_flag=True, help='交互模式')
@click.pass_obj
def chat(ai, interactive):
    """启动交互式对话"""
    if interactive:
        from .interactive import InteractiveMode
        InteractiveMode(ai).run()
    else:
        while True:
            prompt = input("> ")
            if prompt.lower() in ['exit', 'quit']:
                break
            print(ai.process(prompt))

@cli.command()
@click.pass_obj
def agents(ai):
    """列出所有Agent"""
    for name, meta in ai.list_agents():
        print(f"{name}: {meta.domain}")

@cli.command()
@click.argument('agent_name')
@click.pass_obj
def info(ai, agent_name):
    """查看Agent详情"""
    meta = ai.get_agent_info(agent_name)
    print(f"Name: {meta.name}")
    print(f"Domain: {meta.domain}")
    print(f"Capabilities: {', '.join(meta.capabilities)}")

if __name__ == '__main__':
    cli()
```

### 6.2 交互模式

```python
# ==================== cli/interactive.py ====================
class InteractiveMode:
    def __init__(self, ai: AIOS):
        self.ai = ai
        self.history = []
    
    def run(self):
        print("AIOS Interactive Mode")
        print("Type 'help' for commands, 'exit' to quit\n")
        
        while True:
            try:
                prompt = input("\n[You]> ").strip()
                
                if not prompt:
                    continue
                    
                if prompt == 'exit':
                    break
                    
                if prompt == 'help':
                    self._show_help()
                    continue
                
                if prompt == 'history':
                    self._show_history()
                    continue
                
                # 处理命令
                if prompt.startswith('/'):
                    self._handle_command(prompt)
                else:
                    # 处理对话
                    response = self.ai.process(prompt)
                    print(f"\n[AI]> {response}")
                    self.history.append((prompt, response))
                    
            except KeyboardInterrupt:
                print("\n\nUse 'exit' to quit")
            except Exception as e:
                print(f"\nError: {e}")
    
    def _show_help(self):
        print("""
Commands:
  /help     - Show this help
  /history  - Show conversation history
  /agents   - List all agents
  /clear    - Clear screen
  /exit     - Exit
        """)
```

---

## 七、配置文件

### 7.1 主配置

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

---

## 八、接口定义

### 8.1 AIOS API

```python
# ==================== aios/api.py ====================
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List, Dict, Any

app = FastAPI(title="AIOS API")

class TaskRequest(BaseModel):
    input: str
    domain: Optional[str] = None
    stream: bool = False
    context: Optional[Dict[str, Any]] = None

class TaskResponse(BaseModel):
    task_id: str
    result: str
    status: str

@app.post("/api/v1/tasks", response_model=TaskResponse)
async def create_task(req: TaskRequest):
    """提交任务"""
    task_id = aios.submit(req.input, domain=req.domain)
    return TaskResponse(task_id=task_id, result="", status="pending")

@app.get("/api/v1/tasks/{task_id}", response_model=TaskResponse)
async def get_task(task_id: str):
    """获取任务结果"""
    result = aios.get_result(task_id)
    if not result:
        raise HTTPException(404, "Task not found")
    return TaskResponse(**result)

@app.get("/api/v1/agents")
async def list_agents():
    """列出所有Agent"""
    return {"agents": aios.list_agents()}

@app.get("/api/v1/health")
async def health():
    """健康检查"""
    return {"status": "ok"}
```

---

## 九、第三方Agent开发

### 9.1 Agent模板

```python
# ==================== my_agent/agent.py ====================
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

### 9.2 配置文件

```yaml
# ==================== my_agent/config.yaml ====================
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

---

## 十、构建与运行

### 10.1 构建

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

### 10.2 CLI使用

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

---

## 十一、测试计划

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

---

## 十二、Milestones

```
v0.2.0 (Week 1-2)
  ✅ C++ Runtime完成
  ✅ IPC Bus实现
  ✅ Python Bridge
  ✅ 基本CLI

v0.2.1 (Week 3)
  ✅ DocAgent
  ✅ FileAgent系统工具

v0.2.2 (Week 4)
  ✅ MailAgent
  ✅ CodeAgent
  ✅ SearchAgent

v0.2.3 (Week 5)
  ✅ 交互模式完善
  ✅ 配置系统
  ✅ 帮助系统

v0.3.0 (Week 6-7)
  🔲 API Server
  🔲 REST API
  🔲 WebSocket

v0.4.0 (Week 8-10)
  🔲 飞书适配器
  🔲 Web界面
```

---

*文档版本: v0.2*
*创建时间: 2026-04-08*
*状态: 开发中*
*技术栈: LangChain/LangGraph + C++*
