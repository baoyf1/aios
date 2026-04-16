# Domain Agent 架构设计需求文档

**版本**: v1.1  
**章节**: 第二章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**更新**: 2026-04-10 18:10  
**状态**: 需求中  
**优先级**: P0

---

## 一、Domain Agent 概述

### 1.1 设计理念

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Domain Agent = 领域专家                       │
│                                                                      │
│   旧设计: Layer 4 有 Skills，Layer 5 有 Agents，Agents调用Skills   │
│   新设计: Layer 5 统一，Domain Agent = 领域 + 工作流 + 功能模块    │
│                                                                      │
│   就像: 医院里，专科医生 = 看某科病 + 知道怎么治 + 有治疗工具        │
│                                                                      │
│   三位一体:                                                         │
│   - 领域知识: 该领域的专业理解和判断能力                            │
│   - 工作流: 完成任务的标准流程和决策树                              │
│   - 功能模块: 执行具体操作的工具集合                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Layer 5: Application Layer                         │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    Central Agent (PID 0)                          │ │
│  │                                                                │ │
│  │   ┌──────────────────────────────────────────────────────┐     │ │
│  │   │              Agent Registry (注册表)                  │     │ │
│  │   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │     │ │
│  │   │  │FileAgent│ │SysAgent│ │NetAgent│ │MailAgent│ │     │ │
│  │   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ │     │ │
│  │   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │     │ │
│  │   │  │CodeAgent│ │DataAgent│ │NoteAgent│ │SearchAgent│   │ │
│  │   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ │     │ │
│  │   └──────────────────────────────────────────────────────┘     │ │
│  │                         │                                     │ │
│  │                         ▼                                     │ │
│  │   ┌──────────────────────────────────────────────────────┐     │ │
│  │   │              Task Dispatcher (任务分发器)              │     │ │
│  │   │   1. 分析用户意图 (Intent Parser)                   │     │ │
│  │   │   2. 查询注册表找到对应Domain Agent                 │     │ │
│  │   │   3. 分发任务到目标Agent                            │     │ │
│  │   │   4. 收集结果并整合返回                            │     │ │
│  │   └──────────────────────────────────────────────────────┘     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│                              ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │               Domain Agents (领域Agent)                          │ │
│  │                                                                │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │         LangChain / LangGraph (工作流)              │      │ │
│  │   │                                                      │      │ │
│  │   │    ┌─────────┐  ┌─────────┐  ┌─────────┐            │      │ │
│  │   │    │LLM决策  │  │函数调用 │  │条件分支 │            │      │ │
│  │   │    │ 节点    │  │  节点   │  │  节点   │            │      │ │
│  │   │    └─────────┘  └─────────┘  └─────────┘            │      │ │
│  │   │                                                      │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                              │                                 │ │
│  │                              ▼                                 │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │         Function Modules (功能模块)                  │      │ │
│  │   │                                                      │      │ │
│  │   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │      │ │
│  │   │   │文件操作 │ │系统命令 │ │网络请求 │ │记忆操作 │ │      │ │
│  │   │   └─────────┘ └─────────┘ └─────────┘ └─────────┘ │      │ │
│  │   │                                                      │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                                                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、Central Agent (中央Agent) 详细设计

### 2.1 角色定义

| 属性 | 说明 | 详细定义 |
|-----|------|---------|
| PID | 进程ID | 0 (类似Linux init进程，整个系统唯一) |
| 名称 | 名称 | "CentralAgent" |
| 职责 | 职责 | 任务调度 + Agent Registry + 意图解析 |
| 启动 | 启动顺序 | 系统第一个启动的Agent |
| 生命周期 | 生命周期 | 与系统共存亡，不可销毁 |

### 2.2 核心组件

```python
# ==================== central_agent.py ====================
class CentralAgent:
    """中央Agent - 系统总调度"""
    
    def __init__(self, config: AgentConfig):
        # 1. LLM配置
        self.llm = LLMFactory.create(
            provider=config.provider,
            model=config.model,
            api_key=config.api_key
        )
        
        # 2. Agent注册表
        self.registry = AgentRegistry()
        
        # 3. 意图解析器
        self.intent_parser = IntentParser(
            llm=self.llm,
            patterns=[
                IntentPattern("file_*", "file"),
                IntentPattern("mail_*", "mail"),
                IntentPattern("code_*", "code"),
                # ... 更多模式
            ]
        )
        
        # 4. 任务规划器
        self.planner = TaskPlanner(llm=self.llm)
        
        # 5. 结果整合器
        self.integrator = ResultIntegrator(llm=self.llm)
        
        # 6. 记忆系统
        self.memory = MemorySystem()
        
        # 7. 事件循环
        self.event_loop = GlobalEventLoop()
    
    async def process(self, user_input: str, session_id: str = None) -> Response:
        """
        处理用户输入的主流程
        
        Args:
            user_input: 用户原始输入
            session_id: 会话ID (可选)
            
        Returns:
            Response: 包含结果和状态的响应
        """
        # 1. 创建追踪上下文
        trace_id = self._create_trace_id()
        
        # 2. 意图解析
        intent = await self.intent_parser.parse(user_input, trace_id)
        
        # 3. 任务规划
        tasks = await self.planner.plan(intent, trace_id)
        
        # 4. 记忆搜索
        related_memories = await self.memory.search(
            query=user_input,
            user_id=intent.user_id,
            limit=5
        )
        
        # 5. 任务执行
        if len(tasks) == 1:
            result = await self._execute_single(tasks[0], trace_id)
        else:
            result = await self._execute_multi(tasks, trace_id)
        
        # 6. 结果整合
        response = await self.integrator.format(
            result=result,
            intent=intent,
            memories=related_memories
        )
        
        # 7. 记忆存储
        await self.memory.store(
            content=f"用户: {user_input}\n响应: {response.text}",
            metadata={
                "user_id": intent.user_id,
                "intent": intent.domain,
                "trace_id": trace_id
            }
        )
        
        return response
    
    async def _execute_single(self, task: SubTask, trace_id: str) -> Result:
        """执行单个任务"""
        agent = self.registry.get_agent(task.domain)
        if not agent:
            raise AgentNotFoundException(f"Agent not found: {task.domain}")
        
        # 设置trace_id传递给Agent
        task.trace_id = trace_id
        
        return await agent.execute(task)
    
    async def _execute_multi(self, tasks: List[SubTask], trace_id: str) -> List[Result]:
        """执行多个任务 (并行或串行)"""
        # 按依赖关系分组
        independent_tasks = [t for t in tasks if not t.dependencies]
        dependent_tasks = {t.task_id: t for t in tasks if t.dependencies}
        
        results = []
        
        # 执行无依赖的任务 (并行)
        if independent_tasks:
            results = await asyncio.gather(*[
                self._execute_single(t, trace_id) for t in independent_tasks
            ])
        
        # 处理有依赖的任务 (按依赖顺序)
        while dependent_tasks:
            # 找出依赖已满足的任务
            completed_ids = {r.task_id for r in results}
            ready = [
                t for t in dependent_tasks.values()
                if all(dep in completed_ids for dep in t.dependencies)
            ]
            
            if not ready:
                break  # 死锁检测
            
            for task in ready:
                result = await self._execute_single(task, trace_id)
                results.append(result)
                del dependent_tasks[task.task_id]
        
        return results
```

### 2.3 IntentParser 详细设计

```python
class IntentParser:
    """意图解析器"""
    
    def __init__(self, llm, patterns: List[IntentPattern]):
        self.llm = llm
        self.patterns = patterns  # 快速匹配模式
        self.llm_prompt_template = """
你是一个意图解析器。分析用户输入，提取：
1. domain: 领域 (file/mail/code/search/note/data/media)
2. action: 动作 (organize/send/search/create/analyze/...)
3. entities: 实体列表 [{type, value}]
4. user_id: 用户标识
5. confidence: 置信度 (0-1)

用户输入: {user_input}

请以JSON格式返回结果。
"""
    
    async def parse(self, user_input: str, trace_id: str) -> Intent:
        # 1. 快速模式匹配
        for pattern in self.patterns:
            if pattern.matches(user_input):
                return Intent(
                    raw_text=user_input,
                    domain=pattern.domain,
                    confidence=0.9,
                    trace_id=trace_id
                )
        
        # 2. LLM深度解析
        prompt = self.llm_prompt_template.format(user_input=user_input)
        response = await self.llm.agenerate([prompt])
        
        return Intent.from_json(response.text, trace_id=trace_id)
    
    def add_pattern(self, pattern: IntentPattern):
        """添加新的匹配模式"""
        self.patterns.append(pattern)


class IntentPattern:
    """意图匹配模式"""
    
    def __init__(self, keyword: str, domain: str):
        self.keyword = keyword.lower()
        self.domain = domain
    
    def matches(self, text: str) -> bool:
        return self.keyword in text.lower()
```

### 2.4 TaskPlanner 详细设计

```python
class TaskPlanner:
    """任务规划器"""
    
    def __init__(self, llm):
        self.llm = llm
        self.planning_prompt = """
分析以下意图，分解为可执行的任务：

意图: {intent}

任务类型:
- file: list/read/write/move/delete/search/organize
- mail: send/read/list/search/attachment
- code: write/read/search/test/explain
- search: web/knowledge/file/internal
- note: create/read/update/delete/organize
- data: query/transform/analyze/report
- media: process/thumb/transcribe

请返回任务列表，格式:
[
    {{
        "task_id": "1",
        "domain": "file",
        "action": "list",
        "input": {{"path": "/path"}},
        "dependencies": []
    }}
]
"""
    
    async def plan(self, intent: Intent, trace_id: str) -> List[SubTask]:
        # 单意图简单任务
        if len(intent.entities) <= 2 and self._is_simple_action(intent.action):
            return [SubTask(
                task_id="1",
                domain=intent.domain,
                action=intent.action,
                input={"entities": intent.entities},
                dependencies=[],
                priority=Priority.NORMAL,
                trace_id=trace_id
            )]
        
        # 复杂意图需要LLM规划
        prompt = self.planning_prompt.format(intent=intent)
        response = await self.llm.agenerate([prompt])
        
        tasks = self._parse_task_list(response.text)
        
        # 补充trace_id
        for task in tasks:
            task.trace_id = trace_id
        
        return tasks
    
    def _is_simple_action(self, action: str) -> bool:
        simple_actions = {"list", "read", "send", "search", "get", "show"}
        return action.lower() in simple_actions
```

### 2.5 Central Agent 功能需求

| 功能 | 描述 | 输入 | 输出 | 优先级 |
|-----|------|------|------|-------|
| 意图解析 | 解析用户输入确定领域和动作 | "整理桌面的PDF" | Intent{domain="file", action="organize"} | P0 |
| 任务规划 | 将复杂任务分解为子任务 | Intent | List[SubTask] | P0 |
| Agent调度 | 根据domain选择合适Agent | SubTask | Result | P0 |
| 结果整合 | 整合多Agent返回结果 | List[Result] | Response | P1 |
| 记忆搜索 | 搜索相关历史记忆 | query | List[Memory] | P1 |
| 记忆存储 | 存储重要交互记忆 | content, metadata | bool | P1 |
| 会话管理 | 管理多用户会话 | session_id | Session | P2 |

---

## 三、Agent Registry (注册表) 详细设计

### 3.1 数据结构

```python
class AgentRegistry:
    """Agent注册表"""
    
    def __init__(self):
        # name -> AgentEntry
        self._agents_by_name: Dict[str, AgentEntry] = {}
        
        # domain -> AgentEntry
        self._agents_by_domain: Dict[str, AgentEntry] = {}
        
        # 读写锁
        self._lock = RWLock()
    
    def register(self, agent: 'BaseAgent'):
        """
        注册Agent
        
        Args:
            agent: Agent实例，必须是BaseAgent子类
        """
        with self._lock.write():
            entry = AgentEntry(
                name=agent.name,
                domain=agent.domain,
                description=agent.description,
                capabilities=agent.capabilities,
                instance=agent,
                enabled=True
            )
            
            self._agents_by_name[agent.name] = entry
            self._agents_by_domain[agent.domain] = entry
    
    def unregister(self, name: str):
        """注销Agent"""
        with self._lock.write():
            if name in self._agents_by_name:
                entry = self._agents_by_name[name]
                del self._agents_by_domain[entry.domain]
                del self._agents_by_name[name]
    
    def get_by_name(self, name: str) -> Optional['BaseAgent']:
        """通过名称获取Agent"""
        with self._lock.read():
            entry = self._agents_by_name.get(name)
            return entry.instance if entry and entry.enabled else None
    
    def get_by_domain(self, domain: str) -> Optional['BaseAgent']:
        """通过领域获取Agent"""
        with self._lock.read():
            entry = self._agents_by_domain.get(domain)
            return entry.instance if entry and entry.enabled else None
    
    def list_all(self) -> List[AgentEntry]:
        """列出所有已注册Agent"""
        with self._lock.read():
            return list(self._agents_by_name.values())
    
    def list_by_capability(self, capability: str) -> List[AgentEntry]:
        """根据能力查找Agent"""
        with self._lock.read():
            return [
                entry for entry in self._agents_by_name.values()
                if capability in entry.capabilities and entry.enabled
            ]


@dataclass
class AgentEntry:
    """Agent注册表条目"""
    name: str                           # Agent名称 "FileAgent"
    domain: str                         # 领域 "file"
    description: str                    # 描述
    capabilities: List[str]            # 能力列表 ["list", "read", "organize"]
    instance: 'BaseAgent'              # Agent实例
    enabled: bool                      # 是否启用
    config_path: str = ""              # 配置文件路径
    module_path: str = ""              # 模块路径
    version: str = "1.0.0"             # 版本
    author: str = ""                    # 作者
    created_at: float = 0              # 创建时间
    updated_at: float = 0              # 更新时间
```

### 3.2 注册表操作接口

| 操作 | 方法 | 说明 |
|-----|------|------|
| 注册 | register(agent) | 添加新Agent到注册表 |
| 注销 | unregister(name) | 从注册表移除Agent |
| 按名获取 | get_by_name(name) | 通过名称获取Agent实例 |
| 按领域获取 | get_by_domain(domain) | 通过领域获取Agent实例 |
| 列出所有 | list_all() | 获取所有已注册Agent |
| 按能力查找 | list_by_capability(cap) | 查找具有特定能力的Agent |
| 启用/禁用 | enable(name) / disable(name) | 启用或禁用Agent |

---

## 四、预定义 Domain Agents

### 4.1 Agent列表与详细定义

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Domain Agents                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  【应用层Agent - 处理用户任务】                                      │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  FileAgent (文件Agent)                                        │  │
│  │  domain: "file" | capabilities: list/read/write/move/search  │  │
│  │  /organize/delete/copy                                        │  │
│  │  职责: 文档整理、文件搜索、文件操作                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  SysAgent (系统Agent)                                        │  │
│  │  domain: "system" | capabilities: run_cmd/env/proc/list      │  │
│  │  /service/status                                             │  │
│  │  职责: 执行系统命令、进程管理、环境变量                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  NetAgent (网络Agent)                                        │  │
│  │  domain: "network" | capabilities: http_get/post/dns/         │  │
│  │  api_call/webhook                                             │  │
│  │  职责: 网络请求、DNS查询、API调用                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  MailAgent (邮件Agent)                                       │  │
│  │  domain: "mail" | capabilities: send/read/list/search/       │  │
│  │  attachment/draft                                              │  │
│  │  职责: 邮件发送、接收、搜索、附件处理                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  CodeAgent (代码Agent)                                       │  │
│  │  domain: "code" | capabilities: read/write/search/test/      │  │
│  │  explain/refactor/deploy                                      │  │
│  │  职责: 代码阅读、编写、搜索、测试、部署                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  DataAgent (数据Agent)                                       │  │
│  │  domain: "data" | capabilities: query/transform/analyze/      │  │
│  │  report/visualize                                             │  │
│  │  职责: 数据查询、转换、分析、报表生成                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  NoteAgent (笔记Agent)                                       │  │
│  │  domain: "note" | capabilities: create/read/update/delete/   │  │
│  │  organize/search/link                                         │  │
│  │  职责: 笔记创建、阅读、更新、删除、链接                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  SearchAgent (搜索Agent)                                     │  │
│  │  domain: "search" | capabilities: web/knowledge/file/         │  │
│  │  desktop/internal                                            │  │
│  │  职责: 网页搜索、知识搜索、文件搜索、桌面搜索                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  MediaAgent (媒体Agent)                                      │  │
│  │  domain: "media" | capabilities: image_process/video_thumb/  │  │
│  │  audio_transcribe/format_convert                              │  │
│  │  职责: 图片处理、视频缩略图、音频转录、格式转换               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 BaseAgent 抽象基类

```python
# ==================== base_agent.py ====================
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from langgraph import StateGraph

@dataclass
class AgentConfig:
    """Agent配置"""
    name: str
    domain: str
    provider: str = "openai"
    model: str = "gpt-4o"
    api_key: str = ""
    temperature: float = 0.7
    timeout: int = 120
    max_retries: int = 3
    tools: List[str] = field(default_factory=list)


class BaseAgent(ABC):
    """
    Agent基类
    
    所有Domain Agent必须继承此类并实现抽象方法
    """
    
    def __init__(self, config: AgentConfig):
        # 基本属性
        self.name = config.name
        self.domain = config.domain
        self.config = config
        
        # LLM
        self.llm = LLMFactory.create(
            provider=config.provider,
            model=config.model,
            api_key=config.api_key
        )
        
        # 工具注册
        self.tool_registry = ToolRegistry()
        for tool_name in config.tools:
            self.tool_registry.register(self.tool_registry.get_tool(tool_name))
        
        # 记忆系统
        self.memory = AgentMemory(agent_id=self.name)
        
        # 状态
        self._status = AgentStatus.IDLE
        self._current_task: Optional[Task] = None
        
        # 构建工作流图
        self.graph = self._build_graph()
    
    @property
    def status(self) -> 'AgentStatus':
        return self._status
    
    @property
    def capabilities(self) -> List[str]:
        """返回Agent的能力列表"""
        return self._get_capabilities()
    
    @abstractmethod
    def _get_capabilities(self) -> List[str]:
        """
        返回Agent的能力列表
        子类必须实现
        """
        pass
    
    @abstractmethod
    def _build_graph(self) -> StateGraph:
        """
        构建LangGraph工作流
        
        子类必须实现，定义该Agent的工作流程
        
        Example:
            graph = StateGraph(dict)
            graph.add_node("start", self._start_node)
            graph.add_node("process", self._process_node)
            graph.add_node("end", self._end_node)
            graph.add_edge("start", "process")
            graph.add_edge("process", "end")
            graph.set_entry_point("start")
            graph.set_finish_point("end")
            return graph.compile()
        """
        pass
    
    async def execute(self, task: Task) -> Result:
        """
        执行任务
        
        Args:
            task: 任务对象
            
        Returns:
            Result: 执行结果
        """
        self._status = AgentStatus.RUNNING
        self._current_task = task
        
        try:
            # 1. 准备初始状态
            initial_state = self._prepare_state(task)
            
            # 2. 执行工作流
            final_state = await self.graph.ainvoke(initial_state)
            
            # 3. 提取结果
            result = self._extract_result(final_state)
            
            self._status = AgentStatus.COMPLETED
            return result
            
        except Exception as e:
            self._status = AgentStatus.ERROR
            return Result(
                success=False,
                output="",
                error=str(e),
                trace_id=task.trace_id
            )
        finally:
            self._current_task = None
    
    def _prepare_state(self, task: Task) -> dict:
        """
        准备工作流初始状态
        
        可以被子类重写以添加自定义状态
        """
        return {
            "task": task,
            "context": {},
            "memory": self.memory,
            "tools": self.tool_registry,
            "llm": self.llm,
            "result": None,
            "error": None
        }
    
    @abstractmethod
    def _extract_result(self, final_state: dict) -> Result:
        """
        从最终状态提取结果
        
        子类必须实现
        """
        pass
    
    async def health_check(self) -> bool:
        """健康检查"""
        try:
            # 检查LLM连接
            await self.llm.agenerate(["ping"])
            return True
        except:
            return False
    
    def get_info(self) -> AgentInfo:
        """获取Agent信息"""
        return AgentInfo(
            name=self.name,
            domain=self.domain,
            capabilities=self.capabilities,
            status=self._status.value,
            version=self.config.model
        )


class AgentStatus(Enum):
    IDLE = "idle"
    RUNNING = "running"
    WAITING = "waiting"
    COMPLETED = "completed"
    ERROR = "error"
```

### 4.3 FileAgent 完整实现示例

```python
# ==================== file_agent.py ====================
class FileAgent(BaseAgent):
    """文件Agent - 负责文件整理、搜索、操作"""
    
    def __init__(self, config: AgentConfig):
        super().__init__(config)
        self.supported_extensions = ['.pdf', '.doc', '.docx', '.txt', 
                                      '.xls', '.xlsx', '.ppt', '.pptx']
    
    def _get_capabilities(self) -> List[str]:
        return [
            "list",      # 列出目录文件
            "read",      # 读取文件内容
            "write",     # 写入文件
            "move",      # 移动文件
            "copy",      # 复制文件
            "delete",    # 删除文件
            "search",    # 搜索文件
            "organize",  # 整理文件
            "stat",      # 获取文件信息
            "mkdir",     # 创建目录
            "rmdir"      # 删除目录
        ]
    
    def _build_graph(self) -> StateGraph:
        """构建文件处理工作流"""
        graph = StateGraph(dict)
        
        # 添加节点
        graph.add_node("parse_intent", self._parse_intent)
        graph.add_node("validate_path", self._validate_path)
        graph.add_node("list_files", self._list_files)
        graph.add_node("filter_files", self._filter_files)
        graph.add_node("process_files", self._process_files)
        graph.add_node("report", self._report)
        
        # 添加边
        graph.add_edge("parse_intent", "validate_path")
        graph.add_edge("validate_path", "list_files")
        graph.add_edge("list_files", "filter_files")
        graph.add_edge("filter_files", "process_files")
        graph.add_edge("process_files", "report")
        graph.add_edge("report", "__end__")
        
        # 设置入口和结束点
        graph.set_entry_point("parse_intent")
        
        return graph.compile()
    
    async def _parse_intent(self, state: dict) -> dict:
        """解析文件操作意图"""
        task = state["task"]
        input_data = task.input
        
        # 使用LLM解析具体操作
        prompt = f"""
分析文件操作任务：
动作: {task.action}
输入: {input_data}

返回JSON:
{{
    "operation": "list|read|write|move|copy|delete|search|organize",
    "path": "文件路径",
    "filter": {{"type": "pdf|doc|...", "pattern": "*.pdf", "modified": "2024-01-01"}},
    "destination": "目标路径(用于move/copy)"
}}
"""
        response = await self.llm.agenerate([prompt])
        parsed = json.loads(response.text)
        
        state["operation"] = parsed.get("operation", task.action)
        state["path"] = parsed.get("path", input_data.get("path", "/"))
        state["filter"] = parsed.get("filter", {})
        state["destination"] = parsed.get("destination")
        
        return state
    
    async def _validate_path(self, state: dict) -> dict:
        """验证路径合法性"""
        path = state["path"]
        
        # 检查路径是否在允许范围内
        if not self._is_path_allowed(path):
            state["error"] = f"Path not allowed: {path}"
            state["operation"] = "error"
        
        return state
    
    async def _list_files(self, state: dict) -> dict:
        """列出文件"""
        path = state["path"]
        filter_opts = state.get("filter", {})
        
        files = await self.tool_registry.get_tool("file_ops").list_directory(
            path=path,
            include_hidden=False
        )
        
        state["files"] = files
        return state
    
    async def _filter_files(self, state: dict) -> dict:
        """过滤文件"""
        files = state.get("files", [])
        filter_opts = state.get("filter", {})
        
        if not filter_opts:
            state["filtered_files"] = files
            return state
        
        filtered = []
        for f in files:
            # 按扩展名过滤
            if "type" in filter_opts:
                if not f["name"].endswith(f".{filter_opts['type']}"):
                    continue
            
            # 按模式过滤
            if "pattern" in filter_opts:
                if not fnmatch.fnmatch(f["name"], filter_opts["pattern"]):
                    continue
            
            filtered.append(f)
        
        state["filtered_files"] = filtered
        return state
    
    async def _process_files(self, state: dict) -> dict:
        """处理文件"""
        operation = state["operation"]
        files = state.get("filtered_files", [])
        destination = state.get("destination")
        
        results = []
        
        if operation == "list":
            results = files
        elif operation == "organize":
            # 按类型分组
            organized = defaultdict(list)
            for f in files:
                ext = os.path.splitext(f["name"])[1]
                organized[ext].append(f)
            results = dict(organized)
        elif operation == "move" and destination:
            for f in files:
                src = os.path.join(state["path"], f["name"])
                dst = os.path.join(destination, f["name"])
                await self.tool_registry.get_tool("file_ops").move_file(src, dst)
                results.append({"from": src, "to": dst})
        # ... 其他操作
        
        state["results"] = results
        return state
    
    async def _report(self, state: dict) -> dict:
        """生成报告"""
        operation = state["operation"]
        results = state.get("results", [])
        error = state.get("error")
        
        if error:
            state["result"] = Result(
                success=False,
                output="",
                error=error,
                trace_id=state["task"].trace_id
            )
        else:
            summary = self._summarize_results(operation, results)
            state["result"] = Result(
                success=True,
                output=summary,
                error="",
                trace_id=state["task"].trace_id
            )
        
        return state
    
    def _extract_result(self, final_state: dict) -> Result:
        return final_state.get("result", Result(success=False, error="No result"))
    
    def _is_path_allowed(self, path: str) -> bool:
        """检查路径是否在允许范围内"""
        # 不允许系统目录
        forbidden = ["/etc", "/usr", "/bin", "/lib", "/sys", "/proc"]
        for d in forbidden:
            if path.startswith(d):
                return False
        return True
    
    def _summarize_results(self, operation: str, results: Any) -> str:
        """生成结果摘要"""
        if operation == "list":
            return f"找到 {len(results)} 个文件"
        elif operation == "organize":
            return f"已按类型整理，共 {len(results)} 个类别"
        elif operation == "move":
            return f"已移动 {len(results)} 个文件"
        return str(results)
```

---

## 五、Function Modules (功能模块) 详细设计

### 5.1 模块列表

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Function Modules                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  file_ops (文件操作模块)                                       │  │
│  │  ├── list_directory(path) -> List[FileInfo]                  │  │
│  │  ├── read_file(path) -> str                                  │  │
│  │  ├── write_file(path, content) -> bool                       │  │
│  │  ├── move_file(src, dst) -> bool                             │  │
│  │  ├── copy_file(src, dst) -> bool                             │  │
│  │  ├── delete_file(path) -> bool                               │  │
│  │  ├── search_files(path, pattern) -> List[FileInfo]           │  │
│  │  ├── get_file_info(path) -> FileInfo                         │  │
│  │  ├── create_directory(path) -> bool                          │  │
│  │  └── delete_directory(path) -> bool                          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  system_ops (系统操作模块)                                      │  │
│  │  ├── run_command(cmd) -> CommandResult                        │  │
│  │  ├── get_env(key) -> str                                     │  │
│  │  ├── set_env(key, value) -> bool                             │  │
│  │  ├── list_processes() -> List[ProcessInfo]                  │  │
│  │  ├── get_system_info() -> SystemInfo                        │  │
│  │  ├── get_cpu_usage() -> float                                │  │
│  │  ├── get_memory_usage() -> MemoryInfo                        │  │
│  │  ├── get_disk_usage() -> DiskInfo                            │  │
│  │  └── service_control(name, action) -> bool                   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  network_ops (网络操作模块)                                     │  │
│  │  ├── http_get(url, headers) -> HttpResponse                  │  │
│  │  ├── http_post(url, body, headers) -> HttpResponse           │  │
│  │  ├── dns_lookup(domain) -> str                                │  │
│  │  ├── check_connectivity(host) -> bool                        │  │
│  │  └── download_file(url, dest) -> bool                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  memory_ops (记忆操作模块)                                      │  │
│  │  ├── search(query, filters) -> List[MemoryEntry]             │  │
│  │  ├── write(content, metadata) -> str (id)                    │  │
│  │  ├── read(id) -> MemoryEntry                                 │  │
│  │  ├── update(id, content) -> bool                             │  │
│  │  ├── delete(id) -> bool                                      │  │
│  │  └── get_recent(limit) -> List[MemoryEntry]                  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  llm_ops (LLM操作模块)                                        │  │
│  │  ├── chat(messages) -> str                                  │  │
│  │  ├── embed(text) -> List[float]                             │  │
│  │  ├── stream_chat(messages) -> Generator                     │  │
│  │  └── count_tokens(text) -> int                              │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 BaseTool 抽象基类

```python
# ==================== base_tool.py ====================
from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional
from dataclasses import dataclass

@dataclass
class ToolDefinition:
    """工具定义"""
    name: str
    description: str
    parameters: Dict[str, Any]  # JSON Schema
    returns: Dict[str, Any]      # 返回值Schema


class BaseTool(ABC):
    """
    工具基类
    
    所有功能模块必须继承此类
    """
    
    def __init__(self):
        self._definition = self._get_definition()
    
    @property
    def name(self) -> str:
        return self._definition.name
    
    @property
    def description(self) -> str:
        return self._definition.description
    
    @property
    def parameters(self) -> Dict[str, Any]:
        return self._definition.parameters
    
    @abstractmethod
    def _get_definition(self) -> ToolDefinition:
        """返回工具定义"""
        pass
    
    @abstractmethod
    async def execute(self, **kwargs) -> Any:
        """
        执行工具
        
        Args:
            **kwargs: 工具参数
            
        Returns:
            工具执行结果
        """
        pass
    
    async def validate_params(self, **kwargs) -> bool:
        """验证参数"""
        required = self._definition.parameters.get("required", [])
        for param in required:
            if param not in kwargs:
                raise ValueError(f"Missing required parameter: {param}")
        return True
```

### 5.3 FileOpsTool 完整实现

```python
# ==================== file_ops.py ====================
class FileOpsTool(BaseTool):
    """文件操作工具"""
    
    def __init__(self, base_path: str = "/root/aios"):
        self.base_path = base_path
        self.allowed_paths = [base_path]
    
    def _get_definition(self) -> ToolDefinition:
        return ToolDefinition(
            name="file_ops",
            description="文件操作工具：列出、读取、写入、移动、复制、删除、搜索文件",
            parameters={
                "type": "object",
                "properties": {
                    "operation": {
                        "type": "string",
                        "enum": ["list", "read", "write", "move", "copy", "delete", "search", "stat", "mkdir", "rmdir"],
                        "description": "操作类型"
                    },
                    "path": {"type": "string", "description": "文件路径"},
                    "content": {"type": "string", "description": "写入内容 (用于write)"},
                    "destination": {"type": "string", "description": "目标路径 (用于move/copy)"},
                    "pattern": {"type": "string", "description": "搜索模式 (用于search)"}
                },
                "required": ["operation", "path"]
            },
            returns={
                "type": "object",
                "properties": {
                    "success": {"type": "boolean"},
                    "data": {"type": "any", "description": "操作返回数据"},
                    "error": {"type": "string"}
                }
            }
        )
    
    async def execute(self, **kwargs) -> Dict[str, Any]:
        await self.validate_params(**kwargs)
        
        operation = kwargs["operation"]
        path = self._safe_path(kwargs["path"])
        
        try:
            if operation == "list":
                return await self._list_directory(path)
            elif operation == "read":
                return await self._read_file(path)
            elif operation == "write":
                return await self._write_file(path, kwargs.get("content", ""))
            elif operation == "move":
                return await self._move_file(path, kwargs["destination"])
            elif operation == "copy":
                return await self._copy_file(path, kwargs["destination"])
            elif operation == "delete":
                return await self._delete_file(path)
            elif operation == "search":
                return await self._search_files(path, kwargs.get("pattern", "*"))
            elif operation == "stat":
                return await self._get_file_info(path)
            elif operation == "mkdir":
                return await self._create_directory(path)
            elif operation == "rmdir":
                return await self._delete_directory(path)
            else:
                return {"success": False, "error": f"Unknown operation: {operation}"}
        except Exception as e:
            return {"success": False, "error": str(e)}
    
    def _safe_path(self, path: str) -> str:
        """安全路径检查"""
        full_path = os.path.abspath(os.path.join(self.base_path, path))
        if not any(full_path.startswith(p) for p in self.allowed_paths):
            raise PermissionError(f"Path not allowed: {path}")
        return full_path
    
    async def _list_directory(self, path: str) -> Dict[str, Any]:
        """列出目录"""
        if not os.path.isdir(path):
            return {"success": False, "error": "Not a directory"}
        
        entries = os.listdir(path)
        files = []
        for name in entries:
            full_path = os.path.join(path, name)
            stat = os.stat(full_path)
            files.append({
                "name": name,
                "is_dir": os.path.isdir(full_path),
                "size": stat.st_size,
                "modified": datetime.fromtimestamp(stat.st_mtime).isoformat()
            })
        
        return {"success": True, "data": files}
    
    async def _read_file(self, path: str) -> Dict[str, Any]:
        """读取文件"""
        if not os.path.isfile(path):
            return {"success": False, "error": "Not a file"}
        
        with open(path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        return {"success": True, "data": content}
    
    async def _write_file(self, path: str, content: str) -> Dict[str, Any]:
        """写入文件"""
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'w', encoding='utf-8') as f:
            f.write(content)
        return {"success": True, "data": {"path": path}}
    
    # ... 其他方法类似
```

---

## 六、LangGraph 工作流

### 6.1 工作流节点类型

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| 函数节点 (Function Node) | 调用Function Module | list_files, move_files |
| LLM决策节点 (LLM Decision) | 用LLM决定下一步 | 分析意图、选择策略 |
| 条件节点 (Conditional) | if/else分支 | 检查结果决定走向 |
| 循环节点 (Loop) | 遍历处理 | 遍历文件列表 |
| 汇总节点 (Aggregation) | 汇总结果 | 统计、生成报告 |

### 6.2 工作流示例：文件整理

```
用户: "把桌面的PDF整理到Documents"

┌─────────────────────────────────────────────────────────────────────┐
│                    LangGraph 工作流: 文件整理                         │
│                                                                      │
│    ┌──────────┐                                                     │
│    │  Start   │ ← 入口节点                                          │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │ ParseDir │ 解析用户意图                                        │
│    │          │ 输入: "把桌面的PDF整理到Documents"                  │
│    │          │ 输出: {path: "/Desktop", type: "pdf", dest: "/Documents"}│
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │ValidatePath│ 验证路径合法性                                    │
│    │          │ 检查是否在允许范围内                                │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │ListFiles │ 列出源目录文件                                      │
│    │          │ 输出: [file1.pdf, file2.pdf, image.jpg, ...]        │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │FilterPDF │ 筛选PDF文件                                         │
│    │          │ 输入: 所有文件                                      │
│    │          │ 输出: [file1.pdf, file2.pdf]                        │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │CheckDest │ 检查目标目录是否存在                                │
│    │          │ 不存在则创建                                        │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │MoveFiles │ 移动文件到目标                                      │
│    │          │ file1.pdf → /Documents/PDF/file1.pdf               │
│    │          │ file2.pdf → /Documents/PDF/file2.pdf               │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │  Report  │ 生成整理报告                                        │
│    │          │ 成功: 2个文件已整理到 /Documents/PDF              │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │   End    │ ← 结束节点                                          │
│    └──────────┘                                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 工作流代码示例

```python
def _build_graph(self) -> StateGraph:
    """构建文件整理工作流"""
    graph = StateGraph(dict)
    
    # 添加节点
    graph.add_node("parse_intent", self._parse_intent)
    graph.add_node("validate_path", self._validate_path)
    graph.add_node("list_files", self._list_files)
    graph.add_node("filter_files", self._filter_files)
    graph.add_node("check_dest", self._check_destination)
    graph.add_node("move_files", self._move_files)
    graph.add_node("report", self._report)
    
    # 添加边
    graph.add_edge("parse_intent", "validate_path")
    
    # 条件边
    graph.add_conditional_edges(
        "validate_path",
        self._should_continue,  # 条件函数
        {
            "continue": "list_files",
            "error": "report"
        }
    )
    
    graph.add_edge("list_files", "filter_files")
    graph.add_edge("filter_files", "check_dest")
    graph.add_edge("check_dest", "move_files")
    graph.add_edge("move_files", "report")
    graph.add_edge("report", "__end__")
    
    graph.set_entry_point("parse_intent")
    
    return graph.compile()

def _should_continue(self, state: dict) -> str:
    """条件判断"""
    if state.get("error"):
        return "error"
    return "continue"
```

---

## 七、多Agent协作

### 7.1 协作场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                        多Agent协作场景                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  【场景A】父子调用 (主从关系)                                         │
│                                                                      │
│  Central Agent (PID 0) ──spawn──▶ FileAgent (PID 1)                │
│        │                         │                                  │
│        │ 任务分发                 │ 独立执行                          │
│        │ 等待结果                 │ 返回结果                          │
│        ▼                         ▼                                  │
│  主Agent整合结果 ←────────────────┘                                  │
│                                                                      │
│  【场景B】同级协作 (对等关系)                                         │
│                                                                      │
│  FileAgent ──IPC调用──▶ SysAgent                                   │
│        │                         │                                  │
│        │ 请求系统信息             │ 执行命令                          │
│        │ 等待响应                 │ 返回结果                          │
│        ▼                         ▼                                  │
│  处理文件时需要系统状态 ←──────────────────────────────────           │
│                                                                      │
│  【场景C】嵌套委托 (链式调用)                                         │
│                                                                      │
│  FileAgent ──委托──▶ SysAgent ──委托──▶ NetAgent                   │
│        │                │                │                          │
│        │ 嵌套调用        │ 嵌套调用        │ 嵌套调用                │
│        └────────────────┴────────────────┘                          │
│                          │                                          │
│                          ▼                                          │
│                    结果沿链返回                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 协作接口

```python
class Agent协作接口:
    """Agent间协作接口"""
    
    async def spawn_child(self, domain: str, task: Task) -> 'BaseAgent':
        """
        派生子Agent (父子调用)
        
        Args:
            domain: 子Agent领域
            task: 任务
            
        Returns:
            子Agent实例
        """
        child = self.registry.get_by_domain(domain)
        child.trace_id = f"{self.trace_id}/{child.pid}"
        return child
    
    async def call_other_agent(self, agent_name: str, task: Task) -> Result:
        """
        调用其他Agent (同级调用)
        
        Args:
            agent_name: 目标Agent名称
            task: 任务
            
        Returns:
            执行结果
        """
        agent = self.registry.get_by_name(agent_name)
        task.trace_id = f"{self.trace_id}/{agent.pid}"
        return await agent.execute(task)
    
    async def delegate(self, domain: str, task: Task, 
                       wait_result: bool = True) -> Optional[Result]:
        """
        委托任务给其他Agent
        
        Args:
            domain: 目标领域
            task: 任务
            wait_result: 是否等待结果
            
        Returns:
            结果 (如果wait_result=True)
        """
        agent = self.registry.get_by_domain(domain)
        task.trace_id = f"{self.trace_id}/delegate/{agent.pid}"
        
        if wait_result:
            return await agent.execute(task)
        else:
            # 异步执行，不等待结果
            asyncio.create_task(agent.execute(task))
            return None
```

---

## 八、第三方Agent开发

### 8.1 开发模板

```python
# ==================== my_agent/agent.py ====================
from aios_langchain import BaseAgent, AgentConfig

class MyAgent(BaseAgent):
    """
    自定义Agent
    
    这是一个示例，展示如何开发第三方Agent
    """
    
    def __init__(self, config: AgentConfig):
        super().__init__(config)
    
    def _get_capabilities(self) -> List[str]:
        """
        返回Agent的能力列表
        
        这里定义Agent能做什么
        """
        return [
            "custom_action_1",  # 自定义动作1
            "custom_action_2",  # 自定义动作2
        ]
    
    def _build_graph(self) -> StateGraph:
        """
        构建工作流图
        
        定义Agent的工作流程
        """
        graph = StateGraph(dict)
        
        # 添加节点
        graph.add_node("init", self._init_node)
        graph.add_node("process", self._process_node)
        graph.add_node("cleanup", self._cleanup_node)
        graph.add_node("end", self._end_node)
        
        # 添加边
        graph.add_edge("init", "process")
        graph.add_edge("process", "cleanup")
        graph.add_edge("cleanup", "end")
        graph.add_edge("end", "__end__")
        
        graph.set_entry_point("init")
        
        return graph.compile()
    
    async def _init_node(self, state: dict) -> dict:
        """初始化节点"""
        state["custom_data"] = {}
        return state
    
    async def _process_node(self, state: dict) -> dict:
        """处理节点"""
        # 实现处理逻辑
        result = f"Processed: {state['task'].input}"
        state["result"] = result
        return state
    
    async def _cleanup_node(self, state: dict) -> dict:
        """清理节点"""
        # 清理临时资源
        if "custom_data" in state:
            del state["custom_data"]
        return state
    
    async def _end_node(self, state: dict) -> dict:
        """结束节点"""
        state["final_result"] = state.get("result", "Completed")
        return state
    
    def _extract_result(self, final_state: dict) -> Result:
        return Result(
            success=not final_state.get("error"),
            output=final_state.get("final_result", ""),
            error=final_state.get("error", ""),
            trace_id=final_state["task"].trace_id
        )
```

### 8.2 配置文件

```yaml
# ==================== my_agent/config.yaml ====================
name: MyAgent
domain: custom
description: "我的自定义Agent"

# LLM配置 (可选，覆盖全局配置)
provider: "openai"
model: "gpt-4o"
api_key: "${MY_API_KEY}"

# Agent配置
timeout: 120
max_retries: 3

# 启用的工具
tools:
  - file_ops
  - network_ops

# 工作流配置
workflow:
  type: "langgraph"
  config: "my_agent/workflow.yaml"
```

### 8.3 工作流配置

```yaml
# ==================== my_agent/workflow.yaml ====================
name: MyAgent Workflow
version: "1.0"

nodes:
  - name: init
    type: function
    handler: _init_node
    
  - name: process
    type: llm_decision
    prompt: "分析任务并决定处理方式"
    
  - name: execute
    type: function
    handler: _execute_node
    
  - name: end
    type: aggregation
    handler: _end_node

edges:
  - from: init
    to: process
    
  - from: process
    to: execute
    condition: "success"
    
  - from: process
    to: end
    condition: "failure"
    
  - from: execute
    to: end

entry_point: init
finish_point: end
```

---

## 九、验收标准

### 9.1 Central Agent

- [ ] 意图解析准确率 > 90%
- [ ] 任务规划正确分解复杂任务
- [ ] Agent调度正确选择目标Agent
- [ ] 结果整合生成完整响应
- [ ] 记忆搜索返回相关结果
- [ ] 记忆存储正确保存

### 9.2 Agent Registry

- [ ] register/unregister 正常工作
- [ ] 按名称/领域查找正确
- [ ] 列出所有Agent正确
- [ ] 按能力查找正确
- [ ] 线程安全

### 9.3 Domain Agents

- [ ] FileAgent 实现完整文件操作
- [ ] 每个Agent独立LangGraph工作流
- [ ] BaseAgent 抽象基类定义完整
- [ ] 所有预定义Agent可实例化

### 9.4 Function Modules

- [ ] file_ops 实现所有操作
- [ ] system_ops 实现所有操作
- [ ] network_ops 实现所有操作
- [ ] BaseTool 接口定义规范

### 9.5 第三方开发

- [ ] Agent模板可用
- [ ] 配置文件正确解析
- [ ] 动态加载正常工作

---

## 十、相关文档

| 文档 | 位置 |
|-----|------|
| AIOS-5层架构设计.md | /root/aios/docs/AIOS-5层架构设计.md |
| AIOS-开发文档-v0.2.md | /root/aios/docs/AIOS-开发文档-v0.2.md |
| domain-agent-design.md | /root/aios/docs/domain-agent-design.md |

---

*文档版本: v1.1*
*创建时间: 2026-04-10*
*更新: 2026-04-10 18:10 - 增强功能需求详细设计*
