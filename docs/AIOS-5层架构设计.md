# AIOS 5层架构设计
## 操作系统级高内聚低耦合超级架构

---

## 一、Linux 5层架构参考

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: Applications    (应用层)    — 用户程序           │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Libraries/Shell  (库/Shell层) — glibc, API      │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: System Calls     (系统调用层) — 内核接口         │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Kernel           (内核层)    — 进程/内存/文件系统│
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Hardware         (硬件层)    — CPU/内存/设备     │
└─────────────────────────────────────────────────────────────┘

通信规则：
• 向下：封装 (Encapsulation) — 添加每层的控制信息
• 向上：解包 (Decapsulation) — 剥离控制信息，还原数据
• 跨层禁止直接通信 — 只允许邻层交互（低耦合）
```

---

## 二、AIOS 5层架构 (v2.0)

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

---

## 三、Domain Agent 架构 (Layer 5 核心)

### 3.1 设计理念

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Domain Agent = 领域专家                       │
│                                                                      │
│   旧设计: Layer 4 有 Skills，Layer 5 有 Agents，Agents调用Skills   │
│   新设计: Layer 5 统一，Domain Agent = 领域 + 工作流 + 功能模块    │
│                                                                      │
│   就像: 医院里，专科医生 = 看某科病 + 知道怎么治 + 有治疗工具        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 整体架构

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
│  │   └──────────────────────────────────────────────────────┘     │ │
│  │                         │                                     │ │
│  │                         ▼                                     │ │
│  │   ┌──────────────────────────────────────────────────────┐     │ │
│  │   │              Task Dispatcher (任务分发器)              │     │ │
│  │   │   1.分析意图 → 2.查找Agent → 3.分发 → 4.收集结果   │     │ │
│  │   └──────────────────────────────────────────────────────┘     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│                              ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │               Domain Agents (领域Agent)                          │ │
│  │                                                                │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │         LangChain / LangGraph (工作流)              │      │ │
│  │   │    (节点: LLM决策 / 函数调用 / 条件分支 / 循环)     │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                              │                                 │ │
│  │                              ▼                                 │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │         Function Modules (功能模块)                  │      │ │
│  │   │                                                      │      │ │
│  │   │   ┌─────────┐ ┌─────────┐ ┌─────────┐              │      │ │
│  │   │   │文件操作 │ │系统命令 │ │网络请求 │  ...         │      │ │
│  │   │   └─────────┘ └─────────┘ └─────────┘              │      │ │
│  │   │                                                      │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                                                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 预定义 Domain Agents

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Domain Agents                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  FileAgent (文件Agent)                                        │  │
│  │  domain: "file" | capabilities: list/read/write/move/search │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  SysAgent (系统Agent)                                        │  │
│  │  domain: "system" | capabilities: run_cmd/env/proc/list     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  NetAgent (网络Agent)                                        │  │
│  │  domain: "network" | capabilities: http_get/post/dns/...    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  MailAgent (邮件Agent)                                       │  │
│  │  domain: "mail" | capabilities: send/read/list/search       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  DataAgent (数据Agent)                                       │  │
│  │  domain: "data" | capabilities: query/transform/report     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 预定义 Function Modules

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Function Modules                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  file_ops (文件操作模块)                                       │  │
│  │  list_directory, read_file, write_file, move_file,           │  │
│  │  delete_file, search_files, get_file_info                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  system_ops (系统操作模块)                                      │  │
│  │  run_command, get_env, list_processes, get_system_info       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  network_ops (网络操作模块)                                     │  │
│  │  http_get, http_post, dns_lookup                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  memory_ops (记忆操作模块)                                      │  │
│  │  memory_search, memory_write, memory_read                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  llm_ops (LLM操作模块)                                        │  │
│  │  chat, embed, stream_chat                                   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 LangGraph 工作流示例

```
用户: "把桌面的PDF整理到Documents"

┌─────────────────────────────────────────────────────────────────────┐
│                    LangGraph 工作流                                  │
│                                                                      │
│    ┌──────────┐                                                     │
│    │  Start   │                                                     │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │ ParseDir │ 列出桌面文件                                        │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │FilterPDF │ 筛选PDF                                            │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │MoveFiles │ 移动到Documents/PDF                                 │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │  Report  │ 返回结果                                            │
│    └────┬─────┘                                                     │
│         ▼                                                           │
│    ┌──────────┐                                                     │
│    │   End    │                                                     │
│    └──────────┘                                                     │
│                                                                      │
│  节点类型:                                                          │
│  - 函数节点: 调用 Function Module                                    │
│  - LLM决策节点: 用 LLM 决定下一步                                   │
│  - 条件节点: if/else 分支                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.6 多Agent协作

```
用户: "把桌面PDF发到张总邮箱"

┌─────────────────────────────────────────────────────────────────────┐
│  协作流程                                                            │
│                                                                      │
│  Central Agent → 分析意图 → 需要 FileAgent + MailAgent             │
│                       │                                            │
│                       ▼                                            │
│           ┌──────────────────────┐     ┌──────────────────────┐   │
│           │     FileAgent        │     │     MailAgent        │   │
│           │                      │     │                      │   │
│           │ 1.列出桌面文件       │     │                      │   │
│           │ 2.筛选PDF           │────▶│ 4.发送邮件(带附件)   │   │
│           │ 3.准备附件          │     │                      │   │
│           └──────────────────────┘     └──────────────────────┘   │
│                      │                          ▲                  │
│                      └──────────────────────────┘                  │
│                                 IPC 调用                            │
│                                                                      │
│  Context Chain: "0/1/2" (Central/File/Mail)                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 四、层间通信协议

### 4.1 封装/解包流程

```
                    ▲ 向上解包 (Decapsulation)                    
                    │                                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        用户输入                                     │
│                   "帮我整理桌面PDF"                                  │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                    Layer 5: Central Agent
                    ┌─────────────────────────────────────────────┐
                    │ 1. 意图分析 → domain="file"                 │
                    │ 2. 查注册表 → FileAgent                    │
                    │ 3. 分发任务                                 │
                    └─────────────────────────────────────────────┘
                                  │ 封装: 添加任务上下文
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
                    Layer 5: Domain Agent (FileAgent)
                    ┌─────────────────────────────────────────────┐
                    │ 1. LangGraph 工作流执行                     │
                    │ 2. 调用 Function Module (file_ops)        │
                    │ 3. TaskResult{status, output, trace_id}   │
                    └─────────────────────────────────────────────┘
                                  │ 封装: 添加Agent响应头
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
                    Layer 4: 中间件层
                    ┌─────────────────────────────────────────────┐
                    │ 1. 序列化结果 → JSON                       │
                    │ 2. 添加中间件头 MiddlewareHeader{          │
                    │      trace_id, agent_id, timestamp         │
                    │    }                                       │
                    └─────────────────────────────────────────────┘
                                  │ 封装: 添加协议头
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
                    Layer 3: 系统调用/驱动层
                    ┌─────────────────────────────────────────────┐
                    │ 1. 权限检查 (PathGuard + CapChecker)       │
                    │ 2. 调用对应 Driver                         │
                    │ 3. 添加 SysCallHeader                      │
                    └─────────────────────────────────────────────┘
                                  │ 封装: 添加内核请求
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
                    Layer 2: 内核层
                    ┌─────────────────────────────────────────────┐
                    │ 1. O(1)调度选择下一个Agent                  │
                    │ 2. 分配时间片                               │
                    │ 3. 记录审计日志                             │
                    │ 4. 上下文切换                              │
                    └─────────────────────────────────────────────┘
                                  │ 封装: 添加硬件请求
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
                    Layer 1: 硬件层
                    ┌─────────────────────────────────────────────┐
                    │ 1. HTTP调用外部LLM API (如有)              │
                    │ 2. 本地Ollama向量嵌入 (记忆检索)            │
                    │ 3. 文件系统操作 (读取/移动PDF)              │
                    │ 4. 飞书消息发送                             │
                    └─────────────────────────────────────────────┘
                                  │
                                  │ 结果返回 (逆向解包)
                                  ▼
                    ... 逐层向上返回结果 ...
```

### 4.2 协议头设计

```cpp
// ==================== Layer 1: Hardware Request ====================
struct HardwareRequest {
    uint64_t request_id;
    std::string operation;      // "http_call", "file_op", "embed"
    void* raw_result;         // 原始结果
};

// ==================== Layer 2: Kernel Request ====================
struct KernelRequest {
    uint64_t request_id;
    pid_t from_pid;            // 来源Agent
    pid_t to_pid;             // 目标Agent (0=中央Agent)
    AgentPriority priority;    // 调度优先级
    uint32_t time_slice_ms;   // 分配的时间片
    HardwareRequest hardware;  // 下层请求
};

// ==================== Layer 3: System Call ====================
struct SysCallRequest {
    uint64_t request_id;
    pid_t caller_pid;
    uint32_t syscall_id;       // 系统调用号
    ResourceLimits limits;    // 资源限制
    std::vector<std::string> params;
    KernelRequest kernel;
};

// ==================== Layer 4: Middleware ====================
struct MiddlewareMessage {
    uint64_t trace_id;        // 追踪ID
    pid_t agent_id;           // 来源Agent
    uint64_t timestamp;
    std::string session_id;
    std::string payload;      // JSON序列化数据
    SysCallRequest syscall;
};

// ==================== Layer 5: Application ====================
struct AppRequest {
    std::string raw_input;
    Intent parsed_intent;      // 解析后的意图 {domain, action, entities}
    std::vector<SubTask> tasks;
    MiddlewareMessage middleware;
};

// ==================== Task Result ====================
struct TaskResult {
    bool success;
    std::string output;        // 执行结果
    std::string error;         // 错误信息
    std::string trace_id;     // 追踪ID
    uint64_t duration_ms;     // 执行耗时
};
```

---

## 五、与Linux对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AIOS vs Linux 对比                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┬──────────────────────────────────────────────┐ │
│  │     Linux       │              AIOS                          │ │
│  ├─────────────────┼──────────────────────────────────────────────┤ │
│  │ Layer 5 应用     │ Layer 5 Domain Agent (Central + 领域Agent) │ │
│  │ Layer 4 库/Shell │ Layer 4 中间件 (序列化/会话)                 │ │
│  │ Layer 3 系统调用  │ Layer 3 驱动 (FS/Net/LLM/Msg/Mem/Proc)    │ │
│  │ Layer 2 内核     │ Layer 2 内核 (调度器/内存/IPC)              │ │
│  │ Layer 1 硬件     │ Layer 1 硬件 (LLM API/向量DB/文件系统)      │ │
│  └─────────────────┴──────────────────────────────────────────────┘ │
│                                                                      │
│  关键类比:                                                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Linux PID 0 (init)     →  AIOS 中央Agent (PID 0)            │ │
│  │  Linux 进程             →  AIOS Domain Agent                  │ │
│  │  Linux 系统调用表       →  AIOS DriverManager                  │ │
│  │  Linux 进程调度器       →  AIOS O(1) Scheduler                 │ │
│  │  Linux 文件描述符       →  AIOS EventChannel                   │ │
│  │  Linux wait()/exit()   →  AIOS wait_queue/wake_up            │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、核心设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                         核心设计原则                                  │
│                                                                      │
│  1. 底层不关心上层                                                   │
│     Layer 1-3 不感知 Domain Agent 的业务逻辑                       │
│     通过系统调用和 Driver 接口进行交互                               │
│                                                                      │
│  2. 领域自治                                                        │
│     Domain Agent 独立管理自己的 LangGraph 和 Function Modules       │
│     Central Agent 只负责调度，不管具体实现                           │
│                                                                      │
│  3. 严格分层                                                        │
│     跨层通信必须通过邻层接口                                         │
│     禁止跨层直接调用                                                 │
│                                                                      │
│  4. 可追溯                                                          │
│     每个请求都有 trace_id                                            │
│     贯穿整个调用链                                                   │
│                                                                      │
│  5. 权限最小化                                                      │
│     Agent 只能访问自己工作目录                                       │
│     共享资源需要显式授权                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 七、信息流：向上解包 / 向下封装

### 场景：用户说 "把桌面PDF整理到Documents"

```
┌─ 用户输入 ──────────────────────────────────────────────────────────────┐
│ "把桌面上的PDF文件整理到Documents/PDF文件夹，然后发邮件给张总"          │
└───────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ 【封装】
                            ┌───────────────────┐
                            │  Layer 5 应用层    │
                            │ Intent = MULTI     │
                            │ Tasks = [file, mail]│
                            └───────────────────┘
                                    │ 【封装: 添加元数据】
                                    ▼
                            ┌───────────────────┐
                            │  Layer 4 中间件    │
                            │ trace_id, timestamp│
                            │ session_id        │
                            └───────────────────┘
                                    │ 【封装: 资源限制】
                                    ▼
                            ┌───────────────────┐
                            │  Layer 3 系统调用  │
                            │ resource_limits    │
                            │ tool_id           │
                            └───────────────────┘
                                    │ 【封装: PID, 优先级】
                                    ▼
                            ┌───────────────────┐
                            │  Layer 2 内核      │
                            │ from=0, to=file   │
                            │ priority=HIGH     │
                            └───────────────────┘
                                    │ 【封装: 硬件操作码】
                                    ▼
                            ┌───────────────────┐
                            │  Layer 1 硬件      │
                            │ op=file_move     │
                            │ src=/Desktop/*.pdf│
                            │ dst=/Documents/PDF│
                            └───────────────────┘
                                    │
                                    ▼ 【执行】
                            ┌───────────────────┐
                            │  硬件执行结果       │
                            │ success, 12 files │
                            └───────────────────┘
                                    │
                                    ◀ 【反向逐层解包返回】
```

---

## 八、独立性保证

### 8.1 层间隔离机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                           隔离墙 (Isolation Wall)                      │
│                                                                      │
│   每个Layer运行在独立的上下文/进程中                                   │
│                                                                      │
│   Layer 5: 用户空间 (Domain Agent进程)                               │
│          ↓ IPC (只通过接口)                                         │
│   Layer 4: 中间件空间 (序列化/会话管理)                              │
│          ↓ IPC                                                      │
│   Layer 3: 驱动空间 (OS适配器)                                       │
│          ↓ 系统调用                                                  │
│   Layer 2: 内核空间 (中央Agent) ← 最高权限                           │
│          ↓ 硬件抽象                                                  │
│   Layer 1: 硬件空间 (LLM/DB/FS) ← 受限访问                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 依赖注入（Dependency Injection）

```cpp
// 每一层通过接口注入，不直接依赖实现
class AgentApplication {
    // 注入的不是具体类，而是接口
    IKernelLayer* kernel_;      // 指向内核层的指针
    IMiddlewareLayer* middleware_;
    
    // 通过构造函数注入
    AgentApplication(IKernelLayer* kernel, IMiddlewareLayer* mw)
        : kernel_(kernel), middleware_(mw) {}
    
    // 不关心底层实现，只通过接口调用
    void execute(Task t) {
        kernel_->send(t.to_pid, t.msg);  // 通过接口
    }
};
```

---

## 九、异常处理与传播

```
                        ▲ 异常向上传播 (类似异常穿透)
                        │
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Layer 1 (硬件) ──发现异常──▶ throw HwException                   │
│        ▲                                                               │
│        │ 捕获                                                      │
│  Layer 2 (内核) ──捕获──▶ 记录日志 ──▶ throw KernelException        │
│        ▲                                                               │
│        │ 捕获                                                      │
│  Layer 3 (系统调用) ──捕获──▶ 转换错误码 ──▶ throw SysCallException │
│        ▲                                                               │
│        │ 捕获                                                      │
│  Layer 4 (中间件) ──捕获──▶ 格式化错误 ──▶ throw MiddlewareException │
│        ▲                                                               │
│        │ 捕获                                                      │
│  Layer 5 (应用) ──捕获──▶ 用户友好的错误消息                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十一、上下文生命周期管理（Context Lifecycle）

### 11.1 设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                         上层 (Context Layer)                         │
│   业务上下文 ← 组装/销毁/传递 ← 需要额外处理                         │
├─────────────────────────────────────────────────────────────────────┤
│                      上下文桥接层 (Bridge)                           │
│   负责将业务上下文绑定到IO事件                                       │
├─────────────────────────────────────────────────────────────────────┤
│                      底层 (Event Engine)                             │
│   epoll ← 异步IO ← fd/事件 ← 不关心上层                             │
└─────────────────────────────────────────────────────────────────────┘

核心原则：底层不关心上层上下文，上层通过桥接层附加额外处理
```

### 11.2 上下文结构设计

```cpp
// ==================== 底层IO上下文 (不感知业务) ====================
struct IoContext {
    void* user_context;              // 上层业务上下文指针 (透明)
    std::function<void()> cleanup;   // 卸载时额外处理回调
    int fd;                          // 文件描述符
    uint32_t events;                 // EPOLLIN/EPOLLOUT/EPOLLET
};

// ==================== 上层业务上下文 (额外处理) ====================
struct BusinessContext {
    std::string session_id;
    std::string agent_name;
    std::unordered_map<std::string, std::string> attrs;
    
    // 额外处理：装载时 (Attach)
    void on_attach(IoContext* io_ctx) {
        // 业务特定的初始化逻辑
        // 例如：加载记忆、初始化状态、分配资源
    }
    
    // 额外处理：卸载时 (Detach)
    void on_detach() {
        // 业务特定的清理逻辑
        // 例如：保存状态、释放资源、持久化数据
    }
};
```

### 11.3 装载卸载流程

```
装载 (Attach) 流程:
┌─────────────────────────────────────────────────────────────────────┐
│  1. 上层创建 BusinessContext                                        │
│  2. 调用底层: engine.attach(fd, business_ctx, cleanup_cb)          │
│  3. 底层创建 IoContext，绑定 business_ctx                           │
│  4. 上层 on_attach() 执行额外组装逻辑                                │
│  5. 注册 fd 到 epoll                                                │
│  6. 返回 IoContext 句柄给上层                                       │
└─────────────────────────────────────────────────────────────────────┘

卸载 (Detach) 流程:
┌─────────────────────────────────────────────────────────────────────┐
│  1. epoll 触发关闭事件 或 上层主动关闭                               │
│  2. 底层调用 cleanup_cb() — 通知上层执行额外处理                     │
│  3. 底层从 epoll 移除 fd                                            │
│  4. 销毁 IoContext                                                  │
│  5. 上层 on_detach() 执行额外清理                                   │
│  6. 销毁 BusinessContext                                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.4 关键实现代码

```cpp
// ==================== EventEngine (底层，不关心上层) ====================
class EventEngine {
private:
    int epfd_;
    std::unordered_map<int, IoContext*> ctx_map_;  // fd → context
    
public:
    // 装载：底层只负责注册，不感知业务
    int attach(int fd, void* user_ctx, std::function<void()> cleanup) {
        IoContext* ctx = new IoContext{
            .user_context = user_ctx,
            .cleanup = cleanup,
            .fd = fd,
            .events = EPOLLIN | EPOLLOUT | EPOLLET
        };
        
        epoll_event ev = { .events = ctx->events, .data = { .ptr = ctx } };
        epoll_ctl(epfd_, EPOLL_CTL_ADD, fd, &ev);
        ctx_map_[fd] = ctx;
        return 0;
    }
    
    // 卸载：先调用cleanup通知上层，再销毁底层资源
    int detach(int fd) {
        auto it = ctx_map_.find(fd);
        if (it == ctx_map_.end()) return -1;
        
        IoContext* ctx = it->second;
        
        // 关键：先通知上层执行额外处理
        if (ctx->cleanup) {
            ctx->cleanup();
        }
        
        epoll_ctl(epfd_, EPOLL_CTL_DEL, fd, nullptr);
        ctx_map_.erase(it);
        delete ctx;
        return 0;
    }
    
    void* get_user_context(int fd) {
        auto it = ctx_map_.find(fd);
        return it != ctx_map_.end() ? it->second->user_context : nullptr;
    }
};

// ==================== 上层业务使用 ====================
class FileAgent {
    EventEngine engine_;
    
    void handle_connection(int client_fd) {
        auto biz_ctx = new BusinessContext{
            .session_id = generate_session_id(),
            .agent_name = "FileAgent"
        };
        
        // 额外组装逻辑 (在attach之前)
        biz_ctx->on_attach();
        
        // 装载到底层
        engine_.attach(client_fd, biz_ctx, [biz_ctx]() {
            // 卸载时的额外处理 (由底层在适当时候调用)
            biz_ctx->on_detach();
            delete biz_ctx;
        });
    }
};
```

### 11.5 多层嵌套上下文

对于复杂场景，支持多层嵌套的上下文装载：

```cpp
// 例如：FileAgent 处理请求时，需要同时访问 SysAgent 的能力
struct NestedContext {
    BusinessContext file_ctx;   // 自身业务
    void* sysagent_ctx;         // 依赖的SysAgent上下文
    std::function<void()> nested_cleanup;  // 嵌套清理
    
    void on_detach() {
        // 先执行自身的清理
        file_ctx.on_detach();
        // 再执行嵌套上下文的清理
        if (nested_cleanup) nested_cleanup();
    }
};
```

### 11.6 与层间通信的结合

上下文生命周期与五层架构的结合点：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 5 (应用层)  ─── 业务上下文 (BusinessContext)                │
│         │                                                         │
│         │ on_attach() / on_detach()                              │
│         ▼                                                         │
│  Layer 3 (系统调用) ─── IoContext 桥接                             │
│         │                                                         │
│         │ 底层不感知上层，只负责 fd 事件                            │
│         ▼                                                         │
│  Layer 2 (内核层)  ─── epoll 事件循环                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.7 设计要点总结

| 要点 | 说明 |
|------|------|
| **分离关注** | 底层只管IO事件，上层管业务逻辑 |
| **额外处理入口** | cleanup回调 + on_attach/on_detach |
| **生命周期安全** | detach时cleanup在底层删除前执行 |
| **内存管理** | 上层负责user_context的分配和释放 |
| **多线程安全** | epoll在单一线程，事件分发到业务线程池 |

---

## 十二、何时开始下一层

```
Phase 1: 专注Layer 5 (应用层)
         先做 FileAgent, SysAgent (纯应用逻辑)
         用 Mock 模拟下层接口

Phase 2: 加入Layer 4 (中间件层)
         加入 LangChain/LangGraph
         IPC 消息队列

Phase 3: 加入Layer 3 (系统调用层)
         实现 OS适配器 (Windows/Linux)
         工具抽象层

Phase 4: 实现Layer 2 (内核层)
         中央Agent骨架
         O(1)调度器
         进程管理

Phase 5: 实现Layer 1 (硬件层)
         LLM集成 (Ollama)
         向量数据库
         文件/网络操作
```

---

*文档版本: v2.0**2*  
*创建时间: 2026-04-07*  
*更新: 2026-04-07 17:05 - 添加第十一章：上下文生命周期管理*  
*理念: 操作系统级架构，高内聚低耦合*
## 十三、多Agent协作上下文（Multi-Agent Context）

### 13.1 协作场景分类

```
┌─────────────────────────────────────────────────────────────────────┐
│                        多Agent协作场景                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  【场景A】父子调用                                                    │
│  中央Agent (PID 0) ──spawn──▶ FileAgent (PID 1)                    │
│        │                         │                                  │
│        │ 上下文传递               │ 独立上下文                       │
│        │                         │                                  │
│        ◀──结果返回───────────────┘                                  │
│                                                                      │
│  【场景B】同级协作                                                    │
│  FileAgent ──IPC调用──▶ SysAgent                                   │
│        │                         │                                  │
│        │ 借用上下文               │ 独立执行                         │
│        │                         │                                  │
│        ◀──结果返回───────────────┘                                  │
│                                                                      │
│  【场景C】嵌套委托                                                    │
│  FileAgent ──委托──▶ SysAgent ──委托──▶ NetAgent                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.2 上下文传递机制

#### 13.2.1 Context Chain（上下文链）

```cpp
// 上下文链节点 - 追踪跨Agent调用
struct ContextNode {
    pid_t agent_id;                  // 当前Agent ID
    std::string agent_name;          // Agent名称
    void* local_context;            // 当前Agent的本地上下文
    ContextNode* parent;             // 父节点 (谁触发的)
    std::vector<ContextNode*> children;  // 子节点 (派生的Agent)
    std::string trace_id;            // 追踪ID (串联整个调用链)
    
    // 上下文传递
    void pass_to(ContextNode* child) {
        child->parent = this;
        child->trace_id = this->trace_id + "/" + std::to_string(child->agent_id);
        this->children.push_back(child);
    }
};

// 调用栈追踪
class ContextChain {
    ContextNode* root_;              // 中央Agent (PID 0)
    std::unordered_map<pid_t, ContextNode*> agents_;  // pid → node
    
public:
    ContextNode* createNode(pid_t agent_id, const std::string& name, void* ctx) {
        auto node = new ContextNode{
            .agent_id = agent_id,
            .agent_name = name,
            .local_context = ctx,
            .parent = nullptr,
            .trace_id = std::to_string(agent_id)
        };
        agents_[agent_id] = node;
        return node;
    }
    
    ContextNode* spawn(pid_t parent_id, pid_t child_id, const std::string& name) {
        auto parent = agents_[parent_id];
        auto child = createNode(child_id, name, nullptr);
        parent->pass_to(child);
        return child;
    }
    
    std::vector<ContextNode*> getTrace(pid_t agent_id) {
        std::vector<ContextNode*> trace;
        auto node = agents_[agent_id];
        while (node) {
            trace.push_back(node);
            node = node->parent;
        }
        std::reverse(trace.begin(), trace.end());
        return trace;
    }
};
```

#### 13.2.2 上下文传递协议

```
┌─────────────────────────────────────────────────────────────────────┐
│                    上下文传递流程                                    │
│                                                                      │
│  FileAgent 处理任务时需要调用 SysAgent:                              │
│                                                                      │
│  1. FileAgent 准备调用 SysAgent                                      │
│     └─> 打包当前上下文: {trace_id, parent_id, delegation_token}    │
│                                                                      │
│  2. 通过IPC发送调用请求到SysAgent                                    │
│     └─> 消息格式: {from, to, context_bundle, payload}               │
│                                                                      │
│  3. SysAgent 接收请求                                                │
│     └─> 解析context_bundle，创建新ContextNode                       │
│     └─> parent指向FileAgent，trace_id贯穿全程                        │
│                                                                      │
│  4. SysAgent 执行任务 (独立上下文执行)                               │
│                                                                      │
│  5. 结果返回 (携带trace_id)                                         │
│                                                                      │
│  6. 上下文清理 (按创建逆序: 叶子先清理，父后清理)                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.3 上下文所有权（Ownership）模型

```cpp
// 上下文所有权类型
enum class ContextOwnership {
    OWNED,          // 完全拥有，独立生命周期
    BORROWED,       // 借用，临时使用后归还
    SHARED,         // 共享，多个Agent共同持有
    DELEGATED       // 委托，所有权转移
};

// 委托令牌 - 用于所有权转移
struct DelegationToken {
    std::string token_id;
    pid_t from_agent;        // 原所有者
    pid_t to_agent;          // 新所有者
    ContextOwnership ownership;
    std::function<void()> revoke_callback;  // 归还回调
    
    void revoke() {
        if (revoke_callback) revoke_callback();
    }
};

// 带所有权的上下文
struct OwnedContext {
    void* context;
    ContextOwnership ownership;
    std::vector<DelegationToken> delegations;  // 委托出去的token
    
    bool can_cleanup() {
        return ownership == ContextOwnership::OWNED && delegations.empty();
    }
    
    DelegationToken delegate_to(pid_t to_agent) {
        ownership = ContextOwnership::DELEGATED;
        DelegationToken token{
            .token_id = generate_uuid(),
            .from_agent = current_agent_pid(),
            .to_agent = to_agent,
            .ownership = ContextOwnership::BORROWED,
        };
        delegations.push_back(token);
        return token;
    }
    
    void reclaim(const DelegationToken& token) {
        auto it = std::find_if(delegations.begin(), delegations.end(),
            [&token](const DelegationToken& t) { return t.token_id == token.token_id; });
        if (it != delegations.end()) delegations.erase(it);
        if (delegations.empty() && ownership == ContextOwnership::DELEGATED) {
            ownership = ContextOwnership::OWNED;
        }
    }
};
```

### 13.4 协作调度中的上下文

```
┌─────────────────────────────────────────────────────────────────────┐
│                    O(1)调度器 + 上下文集成                           │
│                                                                      │
│  调度器维护:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  runqueue[5]  // 5级优先级队列                               │     │
│  │  bit_map      // 位图快速查找非空队列                        │     │
│  │  curr_agent   // 当前运行的Agent                            │     │
│  │  ctx_chain    // 上下文链 (追踪调用关系)                     │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                      │
│  调度流程:                                                            │
│  1. 按优先级选择下一个Agent                                          │
│  2. 加载Agent上下文 (ContextChain中找到对应节点)                    │
│  3. 执行Agent任务                                                   │
│  4. 如果需要IPC调用: 保存当前上下文，发送context_bundle            │
│  5. Agent完成，按调用链逆序卸载上下文                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.5 多Agent上下文代码示例

```cpp
// ==================== 中央Agent (PID 0) ====================
class CentralAgent {
    ContextChain ctx_chain_;
    EventEngine& engine_;
    
public:
    pid_t spawn_file_agent() {
        pid_t pid = allocate_pid();
        auto node = ctx_chain_.spawn(0, pid, "FileAgent");
        
        auto file_ctx = new FileAgentContext{ .working_dir = "/tmp" };
        node->local_context = file_ctx;
        
        engine_.attach_file(pid, file_ctx, [node, file_ctx]() {
            delete file_ctx;
            delete node;
        });
        return pid;
    }
};

// ==================== FileAgent (调用SysAgent) ====================
class FileAgent {
    pid_t pid_;
    ContextNode* my_node_;
    
public:
    void call_sys_agent(std::string task) {
        my_node_ = ContextChain::getInstance().getNode(pid_);
        auto delegation = current_ctx_.delegate_to(sys_agent_pid);
        
        IPCMessage msg{
            .from = pid_,
            .to = sys_agent_pid,
            .context_bundle = {
                .trace_id = my_node_->trace_id,
                .delegation_token = delegation.token_id,
                .request = task
            }
        };
        IPCBus::send(msg);
    }
};

// ==================== SysAgent (处理请求) ====================
class SysAgent {
    void handle_request(const IPCMessage& msg) {
        auto parent_node = ContextChain::getInstance().getNode(msg.context_bundle.parent_id);
        
        auto my_node = new ContextNode{
            .agent_id = pid_,
            .agent_name = "SysAgent",
            .local_context = new SysAgentContext{},
            .parent = parent_node,
            .trace_id = parent_node->trace_id + "/" + std::to_string(pid_)
        };
        
        // 加载委托的资源
        if (msg.context_bundle.delegation_token.valid) {
            my_node->local_context = ContextManager::reclaim(msg.context_bundle.delegation_token);
        }
        
        execute_task(task);
        cleanup_node(my_node);  // 叶子节点先清理
    }
};
```

### 13.6 上下文追溯（Traceability）

```cpp
struct TraceRecord {
    std::string trace_id;
    std::vector<TraceEntry> entries;
};

struct TraceEntry {
    pid_t agent_id;
    std::string agent_name;
    uint64_t timestamp;
    std::string action;  // spawn/call/return/error
    std::string details;
};

class TraceLogger {
    std::unordered_map<std::string, TraceRecord> traces_;
    
public:
    void log(const std::string& trace_id, const TraceEntry& entry) {
        traces_[trace_id].entries.push_back(entry);
    }
    
    std::string visualize(const std::string& trace_id) {
        auto trace = traces_[trace_id];
        std::string output = "Trace: " + trace_id + "\n";
        for (auto& e : trace.entries) {
            output += "  [" + e.agent_name + "] " + e.action + ": " + e.details + "\n";
        }
        return output;
    }
};
```

### 13.7 设计要点总结

| 要点 | 说明 |
|------|------|
| **Context Chain** | 通过父子节点关系维护完整的调用链 |
| **trace_id** | 贯穿整个调用链，支持跨Agent追溯 |
| **Ownership模型** | OWNED/BORROWED/SHARED/DELEGATED 四种模式 |
| **委托令牌** | DelegationToken管理临时借用和归还 |
| **清理顺序** | 叶子节点先清理，父节点后清理 |
| **O(1)调度集成** | 上下文链与调度器共享，减少加载开销 |

### 13.8 与现有方案对比

| 场景 | 单Agent方案 | 多Agent方案 |
|------|-----------|-------------|
| **Attach** | 直接attach | 通过ContextChain创建节点 |
| **Detach** | 单点cleanup | 按调用链逆序清理 |
| **嵌套** | 不支持 | ContextNode父子关系 |
| **追溯** | 无 | trace_id全程贯穿 |
| **资源借用** | 无 | DelegationToken |

---

*文档版本: v2.0**3*  
*创建时间: 2026-04-07*  
*更新: 2026-04-07 21:30 - 添加第十三章：多Agent协作上下文*  
*理念: 操作系统级架构，高内聚低耦合*

## 十四、Agent事件驱动架构（Event Channel + WaitQueue）

### 14.1 核心洞察

```
传统OS:  等待 文件fd  →  磁盘/网络IO完成
AIOS:    等待 Agent   →  LLM输出/任务完成

AIOS等待的是 Agent的输出，而不是文件描述符
```

### 14.2 混合架构：Event Channel + WaitQueue

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Global Event Loop                                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           epoll (IO事件) + WaitQueue (Agent事件)            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│              ┌───────────────┴───────────────┐                     │
│              ▼                               ▼                      │
│  ┌─────────────────────────┐   ┌─────────────────────────────┐     │
│  │  Event Channel (IO型)   │   │  AgentWaitQueue (Agent型)  │     │
│  │  - 网络 fd               │   │  - 等待某个Agent完成        │     │
│  │  - 文件 fd              │   │  - 等待LLM输出              │     │
│  │  - 定时器              │   │  - 等待子Agent结果          │     │
│  │                         │   │                             │     │
│  │  epoll 监听             │   │  回调机制                   │     │
│  └─────────────────────────┘   └─────────────────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.3 Event Channel（事件通道）

#### 14.3.1 Channel 结构

```cpp
// Channel 类似管道，但专门用于事件传递
struct EventChannel {
    int read_fd;      // 读端 (Global Event Loop持有)
    int write_fd;     // 写端 (Agent持有)
    EventType type;   // AGENT_OUTPUT / IO_COMPLETE / TIMEOUT
    pid_t owner;      // 谁拥有这个channel
};

// 事件消息格式
struct ChannelEvent {
    pid_t source_agent;      // 事件来源Agent
    EventType type;          // 事件类型
    std::string payload;     // 输出内容 / 错误信息
    uint64_t timestamp;
};
```

#### 14.3.2 Channel 创建与分发

```cpp
class ChannelManager {
    std::unordered_map<pid_t, EventChannel*> agent_channels_;
    std::unordered_map<int, EventChannel*> fd_channels_;  // fd → channel
    
public:
    // Agent创建自己的output channel
    EventChannel* create_agent_channel(pid_t agent_id) {
        int fds[2];
        pipe(fds);  // 或 eventfd
        
        auto channel = new EventChannel{
            .read_fd = fds[0],
            .write_fd = fds[1],
            .type = EventType::AGENT_OUTPUT,
            .owner = agent_id
        };
        
        // 注册到全局epoll
        epoll_ctl(epfd_, EPOLL_CTL_ADD, fds[0], &event);
        
        agent_channels_[agent_id] = channel;
        fd_channels_[fds[0]] = channel;
        
        return channel;
    }
    
    // 向channel写入事件
    void write_event(pid_t agent_id, const ChannelEvent& event) {
        auto channel = agent_channels_[agent_id];
        write(channel->write_fd, &event, sizeof(event));
    }
    
    // Global Event Loop 读取事件
    ChannelEvent read_event(int fd) {
        ChannelEvent event;
        read(fd, &event, sizeof(event));
        return event;
    }
};
```

### 14.4 AgentWaitQueue（Agent等待队列）

#### 14.4.1 WaitEntry 结构

```cpp
// 等待条目
struct WaitEntry {
    pid_t waiter_id;           // 等待者Agent
    pid_t target_agent;        // 等待哪个Agent完成
    std::function<void(const Output&)> callback;  // 回调
    uint64_t timeout_ms;       // 超时时间 (可选)
    uint64_t enqueue_time;     // 入队时间
};

// 等待原因
enum class WaitReason {
    DEPENDENCY,     // 等待依赖的Agent完成
    LLM_OUTPUT,     // 等待LLM输出
    TIMEOUT,        // 超时
    CANCEL          // 被取消
};
```

#### 14.4.2 WaitQueue 实现

```cpp
class AgentWaitQueue {
    std::deque<WaitEntry> waiters_;
    std::unordered_map<pid_t, std::deque<WaitEntry*>> target_index_;  // target → 等待它的条目
    
public:
    // Agent等待某个目标Agent完成
    void wait(pid_t waiter, pid_t target, Callback cb, uint64_t timeout_ms = 0) {
        auto entry = new WaitEntry{
            .waiter_id = waiter,
            .target_agent = target,
            .callback = cb,
            .timeout_ms = timeout_ms,
            .enqueue_time = now_ms()
        };
        
        waiters_.push_back(*entry);
        target_index_[target].push_back(entry);
    }
    
    // 目标Agent完成时，唤醒所有等待者
    void wake_up(pid_t target, const Output& output) {
        auto it = target_index_.find(target);
        if (it == target_index_.end()) return;
        
        for (auto entry : it->second) {
            // 调用回调
            entry->callback(output);
        }
        
        // 清空该target的所有等待条目
        target_index_.erase(it);
    }
    
    // 检查超时 (Global Event Loop定期调用)
    void check_timeout() {
        uint64_t now = now_ms();
        for (auto& entry : waiters_) {
            if (entry.timeout_ms > 0 && (now - entry.enqueue_time) > entry.timeout_ms) {
                entry.callback(Output{.error = "TIMEOUT"});
            }
        }
        
        // 移除已处理的超时条目
        std::erase_if(waiters_, [](auto& e) { return e.timeout_ms > 0 && 
            (now_ms() - e.enqueue_time) > e.timeout_ms; });
    }
};
```

### 14.5 完整的事件循环

```cpp
class GlobalEventLoop {
    int epfd_;
    AgentWaitQueue wait_queue_;
    ChannelManager channel_manager_;
    
public:
    void run() {
        while (running_) {
            // 1. 收集所有待监听的文件描述符
            std::vector<epoll_event> events = epoll_wait(epfd_, ...);
            
            // 2. 处理IO事件 (epoll)
            for (auto& ev : events) {
                auto channel = fd_to_channel(ev.data.fd);
                auto event = channel_manager_.read_event(ev.data.fd);
                
                // 根据事件类型分发
                if (event.type == EventType::AGENT_OUTPUT) {
                    // Agent输出事件 → 唤醒等待者
                    wait_queue_.wake_up(event.source_agent, event.payload);
                } else {
                    // IO事件 → 投递给对应Agent的pending队列
                    dispatch_to_agent(event);
                }
            }
            
            // 3. 处理超时
            wait_queue_.check_timeout();
            
            // 4. O(1)调度: 选择下一个ready的Agent执行
            schedule();
        }
    }
};
```

### 14.6 Agent间协作示例

```
场景: FileAgent 等待 SysAgent 完成

┌─────────────────────────────────────────────────────────────────────┐
│  FileAgent                                     SysAgent            │
│     │                                             │                │
│     │  1. channel = create_channel()              │                │
│     │  2. wait_queue_.wait(MY_PID, SYS_AGENT, cb)│               │
│     │  3. yield() → 进入等待                      │                │
│     │                                             │                │
│     │                          SysAgent执行中...  │                │
│     │                                             │                │
│     │                          完成!              │                │
│     │                          channel.write(OUT) │                │
│     │                                             │                │
│     ▼                                             ▼                │
│  Global Event Loop 收到 channel事件              │                │
│     │                                             │                │
│     │  wait_queue_.wake_up(SYS_AGENT, output)     │                │
│     │                                             │                │
│     ▼                                             ▼                │
│  FileAgent 被唤醒，callback执行                   │                │
│     │  处理SysAgent的输出                         │                │
│     ▼                                             │                │
│  FileAgent 继续执行...                           │                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.7 设计要点

| 组件 | 职责 | 核心机制 |
|------|------|----------|
| **EventChannel** | Agent输出通道 | pipe/eventfd + epoll |
| **ChannelManager** | 管道生命周期管理 | 创建/注册/分发 |
| **AgentWaitQueue** | Agent间依赖等待 | 回调 + 目标索引 |
| **GlobalEventLoop** | 统一事件循环 | epoll + wait_queue + schedule |

### 14.8 与原方案对比

| 方面 | 原方案(文件fd) | 新方案(Agent事件) |
|------|---------------|------------------|
| **等待对象** | 文件描述符 | Agent输出 |
| **事件源** | 磁盘/网络IO | LLM/子Agent |
| **分发机制** | fd→Context映射 | Channel + WaitQueue |
| **唤醒方式** | epoll返回 | 回调直接调用 |

---

*文档版本: v2.0**4*  
*创建时间: 2026-04-07*  
*更新: 2026-04-07 21:55 - 添加第十四章：Agent事件驱动架构*  
*理念: Event Channel + WaitQueue 混合方案*
## 十五、Layer 1 硬件层详解（简化版）

### 15.1 设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Layer 1 设计原则                            │
│                                                                      │
│  LLM推理 ──────────▶ 外部API (OpenAI/Claude/MiniMax)              │
│       ▲                                                           │
│       │ 原因: 无需本地GPU，成本低，模型更新灵活                      │
│                                                                      │
│  向量嵌入 ─────────▶ 本地Ollama (nomic-embed-text)                │
│       ▲                                                           │
│       │ 原因: 记忆系统需要频繁检索，本地延迟低                       │
│                                                                      │
│  消息通道 ─────────▶ 飞书开放平台API                               │
│       ▲                                                           │
│       │ 原因: 统一入口，便于扩展到其他IM平台                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.2 模块架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Layer 1: Hardware Layer                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                  External API Gateway                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │ OpenAI API  │  │ Claude API   │  │ MiniMax API  │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    Local Embedding                              │ │
│  │   Ollama + nomic-embed-text (本地向量嵌入)                    │ │
│  │   用途: 记忆系统向量存储与检索、文档相似度匹配                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    Message Gateway                              │ │
│  │   飞书开放平台 API (Webhook接收 + 消息发送)                    │ │
│  │   可扩展: 企业微信、Discord、Telegram                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    File System                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │   本地磁盘    │  │   网络存储    │  │  临时文件    │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.3 模块职责表

```
┌─────────────────────────────────────────────────────────────────────┐
│  模块              │  职责                      │  实现方式           │
├────────────────────┼────────────────────────────┼────────────────────┤
│  External LLM API  │  大语言模型对话             │  HTTP REST调用     │
│  Local Embedding   │  向量嵌入生成与检索         │  Ollama + LanceDB  │
│  Message Gateway   │  飞书消息收发               │  飞书开放平台SDK   │
│  File System       │  Agent持久化存储            │  标准文件系统API   │
│  HTTP Client       │  外部API调用                │  libcurl/reqwest   │
└────────────────────┴────────────────────────────┴────────────────────┘
```

### 15.4 与传统OS的类比

```
传统Linux Layer 1          │     AIOS Layer 1
────────────────────────────────────────────────────────────────
CPU/GPU 物理计算          │     CPU计算 (无本地LLM)
内存管理                  │     内存管理 (Agent内存分配)
磁盘驱动                  │     文件系统 (本地磁盘)
网卡驱动                  │     HTTP Client (外部通信)
声卡/显示器               │     消息网关 (飞书)
GPU CUDA驱动              │     本地向量嵌入 (Ollama)

关键差异:
- AIOS不管理本地GPU (LLM走外部API)
- 本地计算主要是向量嵌入 (记忆检索)
- 通信主要是HTTP REST (调用外部服务)
```

### 15.5 请求处理示例

```
用户通过飞书发送: "帮我整理桌面的PDF"

Layer 1 处理流程:

1. Message Gateway (飞书)
   └─> 接收用户消息，传递给上层

2. 上层处理: Layer 5 FileAgent 解析意图
   └─> 调用 Layer 3 文件工具

3. Layer 1 File System
   └─> list_files("/Desktop/*.pdf")
   └─> move_file(src, dst)

4. 结果返回给上层

5. Message Gateway (飞书)
   └─> 发送Agent响应给用户
```

### 15.6 Layer 1 接口定义

```cpp
class ILayer1 {
    // LLM接口 (代理到外部API)
    virtual LLMResponse chat(const ChatRequest& req) = 0;
    virtual Stream<LLMTokon> chat_stream(const ChatRequest& req) = 0;
    
    // 向量嵌入接口 (本地Ollama)
    virtual std::vector<float> embed(const std::string& text) = 0;
    virtual std::vector<SearchResult> search(const std::string& query, int top_k) = 0;
    
    // 消息网关接口 (飞书)
    virtual void send_message(const Message& msg) = 0;
    virtual Message receive_message() = 0;
    
    // 文件系统接口
    virtual FileHandle open(const std::string& path, int flags) = 0;
    virtual size_t read(FileHandle fh, void* buf, size_t len) = 0;
    virtual size_t write(FileHandle fh, const void* buf, size_t len) = 0;
    virtual int close(FileHandle fh) = 0;
    
    // HTTP Client接口
    virtual HttpResponse http_get(const std::string& url) = 0;
    virtual HttpResponse http_post(const std::string& url, const std::string& body) = 0;
};
```

### 15.7 简化后的优势

```
✅ 降低复杂度 - 无需本地LLM推理、GPU资源管理、模型更新维护
✅ 降低成本 - 使用外部API按量计费，本地只需跑向量嵌入模型
✅ 灵活扩展 - 轻松切换LLM provider，消息网关可扩展到多平台
✅ 专注核心 - 精力集中在Agent编排和调度，记忆系统作为核心竞争力
```

---

*文档版本: v2.0***  
*创建时间: 2026-04-07*  
*更新: 2026-04-07 23:10 - 添加第十五章：Layer 1 硬件层详解（简化版）*  
*理念: LLM走外部API，本地只做向量嵌入和消息网关*
# Layer 3 系统调用/驱动层 - 完整设计方案

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Layer 3: System Call / Driver Layer                 │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     System Call Interface                        │ │
│  │  ┌──────────────────────────────────────────────────────────┐ │ │
│  │  │  sys_open() | sys_read() | sys_write() | sys_close() │ │ │
│  │  │  sys_spawn() | sys_kill() | sys_wait()                │ │ │
│  │  │  sys_http_get() | sys_http_post()                     │ │ │
│  │  │  sys_llm_chat() | sys_llm_embed()                     │ │ │
│  │  │  sys_msg_send() | sys_msg_recv()                      │ │ │
│  │  │  sys_memory_read() | sys_memory_write()                │ │ │
│  │  │  sys_tool_call() | sys_skill_load() | sys_mcp_call()   │ │ │
│  │  └──────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Permission Guard                             │ │
│  │  ┌──────────────────────────────────────────────────────────┐ │ │
│  │  │  PathGuard    - 路径权限检查                             │ │ │
│  │  │  CapChecker   - Capability能力检查                       │ │ │
│  │  │  AuditLog     - 审计日志                                │ │ │
│  │  └──────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Driver Manager                               │ │
│  │  - 驱动加载/卸载                                              │ │
│  │  - 驱动注册表 (driver_registry_)                              │ │
│  │  - 健康检查                                                    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                      Drivers (可插拔模块)                         │ │
│  │                                                                  │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │ │
│  │  │   FS_Driver │ │  Net_Driver  │ │  LLM_Driver  │           │ │
│  │  │  文件系统    │ │  网络通信    │ │  LLM调用     │           │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘           │ │
│  │                                                                  │ │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │ │
│  │  │  Msg_Driver │ │ Mem_Driver  │ │ Skill_Driver │           │ │
│  │  │  消息网关    │ │  记忆存储    │ │  Skill加载   │           │ │
│  │  └──────────────┘ └──────────────┘ └──────────────┘           │ │
│  │                                                                  │ │
│  │  ┌──────────────┐ ┌──────────────┐                             │ │
│  │  │  MCP_Driver  │ │ Proc_Driver │                             │ │
│  │  │  MCP协议     │ │  进程管理    │                             │ │
│  │  └──────────────┘ └──────────────┘                             │ │
│  │                                                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、核心接口定义

### 2.1 驱动基类

```cpp
// ==================== IDriver 基类 ====================
class IDriver {
public:
    virtual ~IDriver() = default;
    
    // 驱动元信息
    virtual std::string name() const = 0;
    virtual std::string version() const = 0;
    virtual int priority() const = 0;  // 优先级，越高越先调用
    
    // 生命周期
    virtual bool init() = 0;           // 初始化
    virtual void shutdown() = 0;        // 关闭
    virtual bool health_check() = 0;   // 健康检查
    
    // 是否处理该请求
    virtual bool can_handle(const SysCallRequest& req) const = 0;
    
    // 处理请求
    virtual SysCallResult handle(const SysCallRequest& req) = 0;
};

// ==================== DriverManager ====================
class DriverManager {
private:
    std::vector<IDriver*> drivers_;  // 按priority排序
    std::unordered_map<std::string, IDriver*> name_index_;
    
public:
    // 注册驱动 (按priority自动排序)
    void register_driver(IDriver* driver) {
        drivers_.push_back(driver);
        std::sort(drivers_.begin(), drivers_.end(),
            [](IDriver* a, IDriver* b) { return a->priority() > b->priority(); });
        name_index_[driver->name()] = driver;
    }
    
    // 卸载驱动
    void unregister_driver(const std::string& name) {
        auto it = name_index_.find(name);
        if (it != name_index_.end()) {
            drivers_.erase(std::remove(drivers_.begin(), drivers_.end(), it->second));
            name_index_.erase(it);
        }
    }
    
    // 调度到第一个能处理该请求的驱动
    SysCallResult dispatch(const SysCallRequest& req) {
        for (auto* driver : drivers_) {
            if (driver->can_handle(req)) {
                return driver->handle(req);
            }
        }
        return SysCallResult::Error(E_NOT_FOUND, "No driver can handle this request");
    }
};
```

### 2.2 系统调用请求/响应

```cpp
// ==================== SysCallRequest ====================
struct SysCallRequest {
    uint64_t trace_id;           // 追踪ID
    pid_t caller_pid;           // 调用者Agent PID
    uint32_t syscall_id;        // 系统调用号
    std::vector<Value> args;    // 参数
    ResourceLimits limits;      // 资源限制
    PermissionContext perms;     // 权限上下文
};

// ==================== SysCallResult ====================
struct SysCallResult {
    enum class Status { OK, ERROR };
    Status status;
    int error_code;
    std::string error_msg;
    Value return_value;
    
    static SysCallResult Ok(Value v) {
        return {Status::OK, 0, "", v};
    }
    static SysCallResult Error(int code, const std::string& msg) {
        return {Status::ERROR, code, msg, {}};
    }
};
```

---

## 三、权限模块

### 3.1 Capability 定义

```cpp
// ==================== Capability 能力枚举 ====================
enum class Capability : uint32_t {
    NONE           = 0,
    // 文件操作
    FS_READ        = 1 << 0,   // 读文件
    FS_WRITE       = 1 << 1,   // 写文件
    FS_EXEC        = 1 << 2,   // 执行文件
    // 网络操作
    NET_HTTP_GET   = 1 << 3,   // HTTP GET
    NET_HTTP_POST  = 1 << 4,   // HTTP POST
    NET_ALL        = NET_HTTP_GET | NET_HTTP_POST,
    // LLM操作
    LLM_CHAT       = 1 << 5,   // LLM对话
    LLM_EMBED      = 1 << 6,   // 向量嵌入
    LLM_ALL        = LLM_CHAT | LLM_EMBED,
    // 记忆操作
    MEM_READ       = 1 << 7,   // 读记忆
    MEM_WRITE      = 1 << 8,   // 写记忆
    MEM_ALL        = MEM_READ | MEM_WRITE,
    // 工具操作
    TOOL_CALL      = 1 << 9,   // 调用工具
    SKILL_LOAD     = 1 << 10,  // 加载Skill
    MCP_CALL       = 1 << 11,  // 调用MCP
    // 系统操作
    AGENT_SPAWN    = 1 << 12,  // 派生Agent
    AGENT_KILL     = 1 << 13,  // 终止Agent
    // 共享资源
    SHARED_WRITE   = 1 << 14,  // 写共享目录
    SYSTEM_ACCESS  = 1 << 15,  // 访问系统目录
    ADMIN          = 1 << 31,  // 管理员权限
};

constexpr Capability operator|(Capability a, Capability b) {
    return static_cast<Capability>(static_cast<uint32_t>(a) | static_cast<uint32_t>(b));
}
constexpr bool has_cap(Capability all, Capability cap) {
    return (static_cast<uint32_t>(all) & static_cast<uint32_t>(cap)) != 0;
}
```

### 3.2 PathGuard 路径守卫

```cpp
// ==================== PathGuard ====================
class PathGuard {
public:
    enum class PathType {
        AGENT_WORKDIR,     // Agent工作目录 /root/aios/agents/{agent_id}/work/
        SHARED,            // 共享目录 /root/aios/shared/
        SYSTEM,            // 系统目录 /etc, /usr, /bin, /lib, /root
        OTHER_AGENT,        // 其他Agent目录 /root/aios/agents/{other_id}/
    };
    
    struct PathRule {
        PathType type;
        std::string pattern;     // glob模式
        Capability required;     // 需要的权限
        bool default_allow;      // 默认是否允许
    };
    
private:
    std::vector<PathRule> rules_;
    
public:
    PathGuard() {
        // 默认规则
        rules_ = {
            // Agent工作目录 - 自动允许全部权限
            {PathType::AGENT_WORKDIR, "/root/aios/agents/*/work/**", Capability::NONE, true},
            
            // 共享目录 - 读自动允许，写需要SHARED_WRITE
            {PathType::SHARED, "/root/aios/shared/**", Capability::NONE, true},
            
            // 系统目录 - 默认拒绝
            {PathType::SYSTEM, "/etc/**", Capability::SYSTEM_ACCESS, false},
            {PathType::SYSTEM, "/usr/**", Capability::SYSTEM_ACCESS, false},
            {PathType::SYSTEM, "/bin/**", Capability::SYSTEM_ACCESS, false},
            {PathType::SYSTEM, "/lib/**", Capability::SYSTEM_ACCESS, false},
            {PathType::SYSTEM, "/root(?!/aios)", Capability::SYSTEM_ACCESS, false},
            
            // 其他Agent目录 - 需要显式授权
            {PathType::OTHER_AGENT, "/root/aios/agents/*/**", Capability::NONE, false},
        };
    }
    
    // 检查路径权限
    CheckResult check(const std::string& path, Capability requested, pid_t agent_id) {
        for (auto& rule : rules_) {
            if (glob_match(rule.pattern, path)) {
                if (rule.default_allow) {
                    return CheckResult::ALLOW;
                } else {
                    // 检查额外权限
                    if (has_cap(requested, rule.required)) {
                        return CheckResult::ALLOW;
                    } else {
                        return CheckResult::DENY(rule.required);
                    }
                }
            }
        }
        // 默认拒绝未知路径
        return CheckResult::DENY(Capability::NONE);
    }
    
    // Agent特定工作目录
    std::string get_agent_workdir(pid_t agent_id) {
        return "/root/aios/agents/" + std::to_string(agent_id) + "/work";
    }
};
```

### 3.3 PermissionGuard 权限守卫

```cpp
// ==================== PermissionGuard ====================
class PermissionGuard {
private:
    std::unordered_map<pid_t, Capability> agent_caps_;
    PathGuard path_guard_;
    AuditLog audit_log_;
    
public:
    // 设置Agent权限
    void set_agent_capability(pid_t agent_id, Capability caps) {
        agent_caps_[agent_id] = caps;
    }
    
    // 检查Agent是否有某权限
    bool has_capability(pid_t agent_id, Capability cap) {
        auto it = agent_caps_.find(agent_id);
        if (it == agent_caps_.end()) return false;
        return has_cap(it->second, cap);
    }
    
    // 检查系统调用权限
    CheckResult check_syscall(pid_t agent_id, const SysCallRequest& req) {
        // 1. 检查Agent基础权限
        Capability required = get_required_capability(req.syscall_id);
        if (!has_capability(agent_id, required)) {
            audit_log_.log(agent_id, req, DenyReason::NO_CAPABILITY);
            return CheckResult::DENY(required);
        }
        
        // 2. 如果涉及文件路径，检查路径权限
        if (req.args.has_path()) {
            auto path_result = path_guard_.check(
                req.args.path(), 
                required, 
                agent_id
            );
            if (!path_result.allowed) {
                audit_log_.log(agent_id, req, DenyReason::PATH_DENIED);
                return path_result;
            }
        }
        
        // 3. 记录审计日志
        audit_log_.log(agent_id, req, DenyReason::ALLOWED);
        
        return CheckResult::ALLOW;
    }
    
private:
    // 根据syscall_id获取所需权限
    Capability get_required_capability(uint32_t syscall_id) {
        switch (syscall_id) {
            case SYS_FILE_OPEN:
            case SYS_FILE_READ:    return Capability::FS_READ;
            case SYS_FILE_WRITE:   return Capability::FS_WRITE;
            case SYS_FILE_EXEC:    return Capability::FS_EXEC;
            case SYS_NET_HTTP_GET: return Capability::NET_HTTP_GET;
            case SYS_NET_HTTP_POST:return Capability::NET_HTTP_POST;
            case SYS_LLM_CHAT:     return Capability::LLM_CHAT;
            case SYS_LLM_EMBED:    return Capability::LLM_EMBED;
            case SYS_MEM_READ:     return Capability::MEM_READ;
            case SYS_MEM_WRITE:    return Capability::MEM_WRITE;
            case SYS_TOOL_CALL:    return Capability::TOOL_CALL;
            case SYS_SKILL_LOAD:   return Capability::SKILL_LOAD;
            case SYS_MCP_CALL:     return Capability::MCP_CALL;
            case SYS_AGENT_SPAWN:  return Capability::AGENT_SPAWN;
            case SYS_AGENT_KILL:   return Capability::AGENT_KILL;
            default:               return Capability::NONE;
        }
    }
};
```

---

## 四、驱动模块详解

### 4.1 FS_Driver (文件系统驱动)

```cpp
// ==================== FS_Driver ====================
class FSDriver : public IDriver {
private:
    PathGuard* path_guard_;
    pid_t current_pid_;
    
public:
    std::string name() const override { return "fs_driver"; }
    int priority() const override { return 100; }  // 高优先级
    
    bool init() override { return true; }
    void shutdown() override {}
    bool health_check() override { return true; }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_FILE_BASE && 
               req.syscall_id <= SYS_FILE_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_FILE_OPEN:
                return do_open(req);
            case SYS_FILE_READ:
                return do_read(req);
            case SYS_FILE_WRITE:
                return do_write(req);
            case SYS_FILE_CLOSE:
                return do_close(req);
            case SYS_FILE_STAT:
                return do_stat(req);
            case SYS_DIR_LIST:
                return do_listdir(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown file syscall");
        }
    }
    
private:
    SysCallResult do_open(const SysCallRequest& req) {
        std::string path = req.args.get<std::string>(0);
        int flags = req.args.get<int>(1);
        
        // 路径权限检查
        Capability needed = (flags & O_WRONLY) ? Capability::FS_WRITE : Capability::FS_READ;
        auto result = path_guard_->check(path, needed, current_pid_);
        if (!result.allowed) {
            return SysCallResult::Error(E_PERMISSION_DENIED, 
                "Path access denied: " + result.deny_reason);
        }
        
        // 调用实际文件系统
        int fd = ::open(path.c_str(), flags, 0644);
        if (fd < 0) {
            return SysCallResult::Error(E_IO, strerror(errno));
        }
        
        return SysCallResult::Ok(Value::from_int(fd));
    }
    
    // ... 其他文件操作实现
};
```

### 4.2 Net_Driver (网络驱动)

```cpp
// ==================== Net_Driver ====================
class NetDriver : public IDriver {
private:
    HttpClient* http_client_;
    PermissionGuard* perm_guard_;
    
public:
    std::string name() const override { return "net_driver"; }
    int priority() const override { return 90; }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_NET_BASE && 
               req.syscall_id <= SYS_NET_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_NET_HTTP_GET:
                return do_http_get(req);
            case SYS_NET_HTTP_POST:
                return do_http_post(req);
            case SYS_NET_HTTP_REQUEST:
                return do_http_request(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown net syscall");
        }
    }
    
private:
    SysCallResult do_http_get(const SysCallRequest& req) {
        std::string url = req.args.get<std::string>(0);
        
        auto response = http_client_->get(url);
        return SysCallResult::Ok(Value::from_string(response.body));
    }
    
    SysCallResult do_http_post(const SysCallRequest& req) {
        std::string url = req.args.get<std::string>(0);
        std::string body = req.args.get<std::string>(1);
        
        auto response = http_client_->post(url, body);
        return SysCallResult::Ok(Value::from_string(response.body));
    }
};
```

### 4.3 LLM_Driver (LLM驱动)

```cpp
// ==================== LLM_Driver ====================
class LLMDriver : public IDriver {
public:
    // LLM Provider配置
    struct ProviderConfig {
        std::string name;           // "openai", "claude", "minimax"
        std::string endpoint;
        std::string api_key;
        std::string default_model;
    };
    
private:
    std::unordered_map<std::string, ProviderConfig> providers_;
    ProviderConfig* current_provider_;
    
public:
    std::string name() const override { return "llm_driver"; }
    int priority() const override { return 80; }
    
    void add_provider(const ProviderConfig& config) {
        providers_[config.name] = config;
        if (!current_provider_) {
            current_provider_ = &providers_[config.name];
        }
    }
    
    void set_provider(const std::string& name) {
        auto it = providers_.find(name);
        if (it != providers_.end()) {
            current_provider_ = &it->second;
        }
    }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_LLM_BASE && 
               req.syscall_id <= SYS_LLM_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_LLM_CHAT:
                return do_chat(req);
            case SYS_LLM_EMBED:
                return do_embed(req);
            case SYS_LLM_STREAM_CHAT:
                return do_stream_chat(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown LLM syscall");
        }
    }
    
private:
    SysCallResult do_chat(const SysCallRequest& req) {
        std::string model = req.args.get<std::string>("model", current_provider_->default_model);
        auto messages = req.args.get<std::vector<Message>>("messages");
        
        // 构建请求
        LLMRequest llm_req;
        llm_req.model = model;
        llm_req.messages = messages;
        
        // 发送到外部API
        auto response = http_post_json(
            current_provider_->endpoint + "/chat/completions",
            current_provider_->api_key,
            to_json(llm_req)
        );
        
        return SysCallResult::Ok(Value::from_string(response.body));
    }
    
    SysCallResult do_embed(const SysCallRequest& req) {
        std::string text = req.args.get<std::string>(0);
        
        // 本地Ollama调用
        auto response = http_post_json(
            "http://localhost:11434/api/embeddings",
            "",  // Ollama不需要API key
            R"({"model": "nomic-embed-text", "prompt": """ + text + """})"
        );
        
        return SysCallResult::Ok(Value::from_vector(parse_embeddings(response)));
    }
};
```

### 4.4 Msg_Driver (消息驱动)

```cpp
// ==================== Msg_Driver ====================
class MsgDriver : public IDriver {
public:
    // 消息通道接口
    struct IChannel {
        virtual ~IChannel() = default;
        virtual void send(const Message& msg) = 0;
        virtual std::optional<Message> recv() = 0;
    };
    
private:
    std::unordered_map<std::string, IChannel*> channels_;  // channel_name -> channel
    
public:
    std::string name() const override { return "msg_driver"; }
    int priority() const override { return 70; }
    
    // 注册消息通道 (飞书、企业微信等)
    void register_channel(const std::string& name, IChannel* channel) {
        channels_[name] = channel;
    }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_MSG_BASE && 
               req.syscall_id <= SYS_MSG_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_MSG_SEND:
                return do_send(req);
            case SYS_MSG_RECV:
                return do_recv(req);
            case SYS_MSG_BROADCAST:
                return do_broadcast(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown msg syscall");
        }
    }
    
private:
    SysCallResult do_send(const SysCallRequest& req) {
        std::string channel_name = req.args.get<std::string>("channel", "feishu");
        Message msg = req.args.get<Message>("message");
        
        auto it = channels_.find(channel_name);
        if (it == channels_.end()) {
            return SysCallResult::Error(E_NOT_FOUND, "Channel not found: " + channel_name);
        }
        
        it->second->send(msg);
        return SysCallResult::Ok(Value::empty());
    }
};
```

### 4.5 Mem_Driver (记忆驱动)

```cpp
// ==================== Mem_Driver ====================
class MemDriver : public IDriver {
public:
    // 记忆层
    enum class MemLayer {
        SHORT_TERM,    // 24小时短期记忆
        MEDIUM_TERM,   // 中期记忆
        LONG_TERM,     // 长期记忆
    };
    
private:
    VectorDB* vector_db_;        // 向量数据库
    std::unordered_map<pid_t, std::string> agent_mem_paths_;  // Agent -> 记忆目录
    
public:
    std::string name() const override { return "mem_driver"; }
    int priority() const override { return 60; }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_MEM_BASE && 
               req.syscall_id <= SYS_MEM_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_MEM_READ:
                return do_read(req);
            case SYS_MEM_WRITE:
                return do_write(req);
            case SYS_MEM_SEARCH:
                return do_search(req);
            case SYS_MEM_EMBED:
                return do_embed(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown mem syscall");
        }
    }
    
private:
    SysCallResult do_search(const SysCallRequest& req) {
        pid_t agent_id = req.args.get<pid_t>("agent_id");
        std::string query = req.args.get<std::string>("query");
        int top_k = req.args.get<int>("top_k", 5);
        
        // 1. 生成查询向量
        auto embedding = generate_embedding(query);
        
        // 2. 向量搜索
        auto results = vector_db_->search(embedding, top_k);
        
        // 3. 构造返回
        return SysCallResult::Ok(Value::from_vector(results));
    }
    
    SysCallResult do_write(const SysCallRequest& req) {
        pid_t agent_id = req.args.get<pid_t>("agent_id");
        std::string content = req.args.get<std::string>("content");
        MemLayer layer = req.args.get<MemLayer>("layer", MemLayer::SHORT_TERM);
        
        // 写入对应层
        // ...
        
        return SysCallResult::Ok(Value::empty());
    }
};
```

### 4.6 Skill_Driver (Skill加载驱动)

```cpp
// ==================== Skill_Driver ====================
class SkillDriver : public IDriver {
public:
    struct SkillHandle {
        std::string skill_id;
        void* module;                    // dlopen返回的句柄
        ISkill* instance;                // Skill实例
    };
    
private:
    std::unordered_map<std::string, SkillHandle> loaded_skills_;
    std::vector<std::string> skill_search_paths_;  // 搜索路径
    
public:
    std::string name() const override { return "skill_driver"; }
    int priority() const override { return 50; }
    
    void add_search_path(const std::string& path) {
        skill_search_paths_.push_back(path);
    }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_SKILL_BASE && 
               req.syscall_id <= SYS_SKILL_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_SKILL_LOAD:
                return do_load(req);
            case SYS_SKILL_INVOKE:
                return do_invoke(req);
            case SYS_SKILL_UNLOAD:
                return do_unload(req);
            case SYS_SKILL_LIST:
                return do_list(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown skill syscall");
        }
    }
    
private:
    SysCallResult do_load(const SysCallRequest& req) {
        std::string skill_id = req.args.get<std::string>(0);
        
        // 1. 查找Skill文件
        std::string path = find_skill(skill_id);
        if (path.empty()) {
            return SysCallResult::Error(E_NOT_FOUND, "Skill not found: " + skill_id);
        }
        
        // 2. dlopen加载
        void* handle = dlopen(path.c_str(), RTLD_LAZY);
        if (!handle) {
            return SysCallResult::Error(E_LOAD_FAILED, dlerror());
        }
        
        // 3. 获取create_instance函数
        auto create = (SkillFactory)dlsym(handle, "create_instance");
        if (!create) {
            dlclose(handle);
            return SysCallResult::Error(E_INIT_FAILED, "Invalid skill module");
        }
        
        // 4. 创建实例
        ISkill* instance = create();
        if (!instance->init()) {
            delete instance;
            dlclose(handle);
            return SysCallResult::Error(E_INIT_FAILED, "Skill init failed");
        }
        
        // 5. 注册
        loaded_skills_[skill_id] = {skill_id, handle, instance};
        
        return SysCallResult::Ok(Value::empty());
    }
    
    SysCallResult do_invoke(const SysCallRequest& req) {
        std::string skill_id = req.args.get<std::string>(0);
        std::string method = req.args.get<std::string>(1);
        Value args = req.args.get<Value>(2);
        
        auto it = loaded_skills_.find(skill_id);
        if (it == loaded_skills_.end()) {
            return SysCallResult::Error(E_NOT_FOUND, "Skill not loaded: " + skill_id);
        }
        
        // 调用Skill方法
        auto result = it->second.instance->invoke(method, args);
        return SysCallResult::Ok(result);
    }
};
```

### 4.7 MCP_Driver (MCP协议驱动)

```cpp
// ==================== MCP_Driver ====================
class MCDriver : public IDriver {
public:
    // MCP服务器连接
    struct MCPServer {
        std::string name;
        std::string endpoint;      // http://localhost:8080
        HttpClient* client;
        bool connected;
    };
    
private:
    std::unordered_map<std::string, MCPServer> servers_;
    
public:
    std::string name() const override { return "mcp_driver"; }
    int priority() const override { return 40; }
    
    // 连接MCP服务器
    bool connect_server(const std::string& name, const std::string& endpoint) {
        servers_[name] = {name, endpoint, new HttpClient(), false};
        return health_check_server(name);
    }
    
    void disconnect_server(const std::string& name) {
        servers_.erase(name);
    }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_MCP_BASE && 
               req.syscall_id <= SYS_MCP_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_MCP_CONNECT:
                return do_connect(req);
            case SYS_MCP_DISCONNECT:
                return do_disconnect(req);
            case SYS_MCP_CALL:
                return do_call(req);
            case SYS_MCP_LIST_TOOLS:
                return do_list_tools(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown MCP syscall");
        }
    }
    
private:
    SysCallResult do_call(const SysCallRequest& req) {
        std::string server_name = req.args.get<std::string>("server");
        std::string tool = req.args.get<std::string>("tool");
        Value args = req.args.get<Value>("args");
        
        auto it = servers_.find(server_name);
        if (it == servers_.end()) {
            return SysCallResult::Error(E_NOT_FOUND, "MCP server not found");
        }
        
        // 发送MCP请求
        auto response = it->second.client->post_json(
            it->second.endpoint + "/tools/" + tool,
            to_json(args)
        );
        
        return SysCallResult::Ok(Value::from_string(response.body));
    }
};
```

### 4.8 Proc_Driver (进程管理驱动)

```cpp
// ==================== Proc_Driver ====================
class ProcDriver : public IDriver {
public:
    // Agent进程信息
    struct AgentProcess {
        pid_t pid;
        std::string name;
        std::string agent_type;
        AgentState state;
        pid_t parent_pid;
        void* context;           // Agent上下文
        ContextNode* ctx_node;  // 上下文链节点
    };
    
private:
    std::unordered_map<pid_t, AgentProcess> processes_;
    pid_t next_pid_;
    pid_t central_agent_pid_;    // PID 0
    
public:
    ProcDriver() : next_pid_(1) {}  // PID 0 保留给中央Agent
    
    std::string name() const override { return "proc_driver"; }
    int priority() const override { return 95; }  // 高优先级，进程管理优先
    
    pid_t create_central_agent() {
        auto pid = next_pid_++;
        processes_[pid] = {pid, "CentralAgent", "central", AgentState::READY, 0, nullptr, nullptr};
        central_agent_pid_ = pid;
        return pid;
    }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_PROC_BASE && 
               req.syscall_id <= SYS_PROC_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_PROC_SPAWN:
                return do_spawn(req);
            case SYS_PROC_KILL:
                return do_kill(req);
            case SYS_PROC_WAIT:
                return do_wait(req);
            case SYS_PROC_YIELD:
                return do_yield(req);
            case SYS_PROC_STATUS:
                return do_status(req);
            default:
                return SysCallResult::Error(E_INVALID, "Unknown proc syscall");
        }
    }
    
private:
    SysCallResult do_spawn(const SysCallRequest& req) {
        std::string agent_type = req.args.get<std::string>("type");
        pid_t parent_pid = req.caller_pid;
        
        pid_t new_pid = next_pid_++;
        
        // 创建Agent进程
        auto& proc = processes_[new_pid] = {
            new_pid,
            agent_type + "_" + std::to_string(new_pid),
            agent_type,
            AgentState::READY,
            parent_pid,
            nullptr,  // 上下文后续设置
            nullptr
        };
        
        // 在上下文链中创建节点
        auto parent_node = processes_[parent_pid].ctx_node;
        auto child_node = ctx_chain_->spawn(parent_node, new_pid, proc.name);
        proc.ctx_node = child_node;
        
        return SysCallResult::Ok(Value::from_int(new_pid));
    }
};
```

---

## 五、系统调用号定义

```cpp
// ==================== 系统调用号定义 ====================
namespace SysCallNumbers {
    // 进程类 (95-99)
    constexpr uint32_t PROC_BASE = 9000;
    constexpr uint32_t PROC_SPAWN = PROC_BASE + 1;      // 9001
    constexpr uint32_t PROC_KILL = PROC_BASE + 2;        // 9002
    constexpr uint32_t PROC_WAIT = PROC_BASE + 3;        // 9003
    constexpr uint32_t PROC_YIELD = PROC_BASE + 4;       // 9004
    constexpr uint32_t PROC_STATUS = PROC_BASE + 5;      // 9005
    constexpr uint32_t PROC_END = PROC_BASE + 9;
    
    // 文件类 (9100-9199)
    constexpr uint32_t FILE_BASE = 9100;
    constexpr uint32_t FILE_OPEN = FILE_BASE + 1;
    constexpr uint32_t FILE_READ = FILE_BASE + 2;
    constexpr uint32_t FILE_WRITE = FILE_BASE + 3;
    constexpr uint32_t FILE_CLOSE = FILE_BASE + 4;
    constexpr uint32_t FILE_STAT = FILE_BASE + 5;
    constexpr uint32_t FILE_UNLINK = FILE_BASE + 6;
    constexpr uint32_t DIR_MKDIR = FILE_BASE + 10;
    constexpr uint32_t DIR_RMDIR = FILE_BASE + 11;
    constexpr uint32_t DIR_LIST = FILE_BASE + 12;
    constexpr uint32_t FILE_END = FILE_BASE + 99;
    
    // 网络类 (9200-9299)
    constexpr uint32_t NET_BASE = 9200;
    constexpr uint32_t NET_HTTP_GET = NET_BASE + 1;
    constexpr uint32_t NET_HTTP_POST = NET_BASE + 2;
    constexpr uint32_t NET_HTTP_REQUEST = NET_BASE + 3;
    constexpr uint32_t NET_END = NET_BASE + 99;
    
    // LLM类 (9300-9399)
    constexpr uint32_t LLM_BASE = 9300;
    constexpr uint32_t LLM_CHAT = LLM_BASE + 1;
    constexpr uint32_t LLM_EMBED = LLM_BASE + 2;
    constexpr uint32_t LLM_STREAM_CHAT = LLM_BASE + 3;
    constexpr uint32_t LLM_END = LLM_BASE + 99;
    
    // 记忆类 (9400-9499)
    constexpr uint32_t MEM_BASE = 9400;
    constexpr uint32_t MEM_READ = MEM_BASE + 1;
    constexpr uint32_t MEM_WRITE = MEM_BASE + 2;
    constexpr uint32_t MEM_SEARCH = MEM_BASE + 3;
    constexpr uint32_t MEM_EMBED = MEM_BASE + 4;
    constexpr uint32_t MEM_END = MEM_BASE + 99;
    
    // 消息类 (9500-9599)
    constexpr uint32_t MSG_BASE = 9500;
    constexpr uint32_t MSG_SEND = MSG_BASE + 1;
    constexpr uint32_t MSG_RECV = MSG_BASE + 2;
    constexpr uint32_t MSG_BROADCAST = MSG_BASE + 3;
    constexpr uint32_t MSG_END = MSG_BASE + 99;
    
    // 工具类 (9600-9699)
    constexpr uint32_t TOOL_BASE = 9600;
    constexpr uint32_t TOOL_CALL = TOOL_BASE + 1;
    constexpr uint32_t SKILL_LOAD = TOOL_BASE + 10;
    constexpr uint32_t SKILL_INVOKE = TOOL_BASE + 11;
    constexpr uint32_t SKILL_UNLOAD = TOOL_BASE + 12;
    constexpr uint32_t SKILL_LIST = TOOL_BASE + 13;
    constexpr uint32_t MCP_CONNECT = TOOL_BASE + 20;
    constexpr uint32_t MCP_DISCONNECT = TOOL_BASE + 21;
    constexpr uint32_t MCP_CALL = TOOL_BASE + 22;
    constexpr uint32_t MCP_LIST_TOOLS = TOOL_BASE + 23;
    constexpr uint32_t TOOL_END = TOOL_BASE + 99;
}
```

---

## 六、审计日志

```cpp
// ==================== AuditLog ====================
class AuditLog {
public:
    struct LogEntry {
        uint64_t timestamp;
        pid_t agent_id;
        std::string agent_name;
        uint32_t syscall_id;
        std::string syscall_name;
        std::string args_json;
        DenyReason deny_reason;
        bool allowed;
    };
    
    enum class DenyReason {
        ALLOWED,
        NO_CAPABILITY,
        PATH_DENIED,
        RATE_LIMITED,
        QUOTA_EXCEEDED,
    };
    
private:
    std::deque<LogEntry> logs_;
    size_t max_size_;
    
public:
    void log(pid_t agent_id, const SysCallRequest& req, DenyReason reason) {
        LogEntry entry{
            .timestamp = now_ms(),
            .agent_id = agent_id,
            .syscall_id = req.syscall_id,
            .syscall_name = get_syscall_name(req.syscall_id),
            .args_json = to_json(req.args),
            .deny_reason = reason,
            .allowed = (reason == DenyReason::ALLOWED),
        };
        
        logs_.push_back(entry);
        if (logs_.size() > max_size_) {
            logs_.pop_front();
        }
    }
    
    // 查询日志
    std::vector<LogEntry> query(pid_t agent_id = 0, uint64_t since = 0) {
        std::vector<LogEntry> result;
        for (auto& entry : logs_) {
            if ((agent_id == 0 || entry.agent_id == agent_id) &&
                entry.timestamp >= since) {
                result.push_back(entry);
            }
        }
        return result;
    }
};
```

---

## 七、初始化流程

```cpp
// ==================== Layer3 初始化 ====================
class Layer3 {
private:
    DriverManager driver_mgr_;
    PermissionGuard perm_guard_;
    AuditLog audit_log_;
    ContextChain* ctx_chain_;  // 上下文链引用
    
public:
    bool init() {
        // 1. 创建中央Agent (PID 0)
        pid_t central_pid = proc_driver_.create_central_agent();
        
        // 2. 注册驱动 (按priority排序)
        driver_mgr_.register_driver(&proc_driver_);   // priority 95
        driver_mgr_.register_driver(&fs_driver_);      // priority 100
        driver_mgr_.register_driver(&net_driver_);      // priority 90
        driver_mgr_.register_driver(&llm_driver_);      // priority 80
        driver_mgr_.register_driver(&msg_driver_);       // priority 70
        driver_mgr_.register_driver(&mem_driver_);      // priority 60
        driver_mgr_.register_driver(&skill_driver_);    // priority 50
        driver_mgr_.register_driver(&mcp_driver_);      // priority 40
        
        // 3. 配置LLM Providers
        llm_driver_.add_provider({"openai", "https://api.openai.com/v1", api_key});
        llm_driver_.add_provider({"claude", "https://api.anthropic.com", api_key});
        
        // 4. 配置消息通道
        msg_driver_.register_channel("feishu", &feishu_channel_);
        
        // 5. 配置Skill搜索路径
        skill_driver_.add_search_path("/root/aios/skills/");
        
        return true;
    }
    
    // 处理系统调用
    SysCallResult dispatch(const SysCallRequest& req) {
        // 1. 权限检查
        auto result = perm_guard_.check_syscall(req.caller_pid, req);
        if (!result.allowed) {
            return SysCallResult::Error(E_PERMISSION_DENIED, 
                "Permission denied: " + result.deny_reason);
        }
        
        // 2. 调度到驱动
        return driver_mgr_.dispatch(req);
    }
};
```

---

*文档版本: v2.0**0*  
*创建时间: 2026-04-07*  
*理念: 模块化驱动架构，高内聚低耦合*
# AIOS Layer 5 应用层重构 - Domain Agent架构

## 一、设计理念

```
┌─────────────────────────────────────────────────────────────────────┐
│                         设计理念                                      │
│                                                                      │
│  旧设计:                                                            │
│  Layer 4: Skills (独立模块) → Layer 5: Agents 调用 Skills          │
│                                                                      │
│  新设计:                                                            │
│  Layer 5: Domain Agent (领域专家)                                   │
│           = 领域知识 + LangChain/LangGraph + 功能模块              │
│           = 自带工具的专家，直接上岗                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、整体架构

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
│  │   └──────────────────────────────────────────────────────┘     │ │
│  │                         │                                     │ │
│  │                         ▼                                     │ │
│  │   ┌──────────────────────────────────────────────────────┐   │ │
│  │   │              Task Dispatcher (任务分发器)              │   │ │
│  │   │   1. 分析用户意图                                     │   │ │
│  │   │   2. 查询注册表找到对应Domain Agent                  │   │ │
│  │   │   3. 分发任务                                        │   │ │
│  │   │   4. 收集结果返回                                    │   │ │
│  │   └──────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│                              ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │               Domain Agents (领域Agent)                          │ │
│  │                                                                │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │              LangChain / LangGraph                   │      │ │
│  │   │    (工作流编排 + 决策树 + 工具调用)                │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                              │                                 │ │
│  │                              ▼                                 │ │
│  │   ┌─────────────────────────────────────────────────────┐      │ │
│  │   │          Function Modules (功能模块)                 │      │ │
│  │   │                                                      │      │ │
│  │   │   ┌─────────┐ ┌─────────┐ ┌─────────┐              │      │ │
│  │   │   │ 文件操作 │ │ 系统命令 │ │ 网络请求 │  ...       │      │ │
│  │   │   └─────────┘ └─────────┘ └─────────┘              │      │ │
│  │   │                                                      │      │ │
│  │   └─────────────────────────────────────────────────────┘      │ │
│  │                                                               │ │
│  │   领域: FileAgent | SysAgent | NetAgent | MailAgent | DataAgent │
│  │                                                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、Central Agent 详解

### 3.1 Agent Registry (注册表)

```cpp
// 注册表条目
struct AgentEntry {
    std::string name;              // "FileAgent"
    std::string domain;            // "file"
    std::string description;        // 描述
    std::vector<std::string> capabilities;  // 能力列表
    std::string langchain_config;  // LangChain配置路径
    std::string module_path;       // 功能模块路径
    bool enabled;                  // 是否启用
};

// 注册表
class AgentRegistry {
    std::unordered_map<std::string, AgentEntry> agents_;
    
public:
    // 注册Agent
    void register_agent(const AgentEntry& entry) {
        agents_[entry.name] = entry;
    }
    
    // 按领域查找
    DomainAgent* find_by_domain(const std::string& domain);
    
    // 按名称查找
    DomainAgent* find_by_name(const std::string& name);
    
    // 列出所有
    std::vector<AgentEntry> list_all();
};
```

### 3.2 预定义Domain Agents

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Domain Agents                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  FileAgent (文件Agent)                                        │  │
│  │  domain: "file"                                              │  │
│  │  capabilities: ["list_dir", "read_file", "write_file",       │  │
│  │                "move_file", "delete_file", "search_files"]   │  │
│  │  module: 文件操作模块 + 路径权限控制                          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  SysAgent (系统Agent)                                        │  │
│  │  domain: "system"                                           │  │
│  │  capabilities: ["run_command", "get_env", "process_list",   │  │
│  │                "service_control", "system_info"]            │  │
│  │  module: 系统命令模块                                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  NetAgent (网络Agent)                                        │  │
│  │  domain: "network"                                          │  │
│  │  capabilities: ["http_get", "http_post", "websocket",       │  │
│  │                "dns_lookup", "port_scan"]                   │  │
│  │  module: 网络请求模块                                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  MailAgent (邮件Agent)                                       │  │
│  │  domain: "mail"                                             │  │
│  │  capabilities: ["send_mail", "read_mail", "list_mail",       │  │
│  │                "attachment_upload", "mail_search"]           │  │
│  │  module: 邮件API模块                                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  DataAgent (数据Agent)                                       │  │
│  │  domain: "data"                                             │  │
│  │  capabilities: ["query_db", "data_transform", "generate_report", │
│  │                "data_analysis"]                              │  │
│  │  module: 数据库/分析模块                                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Task Dispatcher (任务分发器)

```cpp
// 任务分发流程
class TaskDispatcher {
    AgentRegistry* registry_;
    ContextChain* ctx_chain_;
    
public:
    // 分发任务
    DispatchResult dispatch(const UserTask& task) {
        // 1. 意图分析 (可能是LLM辅助)
        Intent intent = analyze_intent(task.raw_input);
        
        // 2. 查找合适的Agent
        auto agent = registry_->find_by_domain(intent.domain);
        if (!agent) {
            return DispatchResult::error("No agent for domain: " + intent.domain);
        }
        
        // 3. 检查权限
        if (!check_permission(task.caller_pid, agent->name())) {
            return DispatchResult::error("Permission denied");
        }
        
        // 4. 创建任务上下文
        auto task_ctx = create_task_context(task, agent);
        
        // 5. 加入调度队列
        schedule_agent(agent, task_ctx);
        
        return DispatchResult::ok(task_ctx->task_id);
    }
    
private:
    // 意图分析
    Intent analyze_intent(const std::string& input) {
        // 关键词匹配 / LLM辅助分析
        // 返回: domain, action, entities
    }
};
```

---

## 四、Domain Agent 详解

### 4.1 Domain Agent 结构

```cpp
// 领域Agent基类
class IDomainAgent {
public:
    virtual ~IDomainAgent() = default;
    
    // 元信息
    virtual std::string name() const = 0;
    virtual std::string domain() const = 0;
    virtual std::vector<std::string> capabilities() const = 0;
    
    // 生命周期
    virtual bool init() = 0;
    virtual void shutdown() = 0;
    
    // 处理任务
    virtual TaskResult execute(const Task& task) = 0;
    
    // LangChain/LangGraph集成
    virtual void load_workflow(const std::string& config_path) = 0;
};

// 具体领域Agent示例
class FileAgent : public IDomainAgent {
private:
    std::unique_ptr<LangGraph> workflow_;  // LangGraph工作流
    std::vector<FunctionModule*> modules_;   // 功能模块
    
public:
    std::string name() const override { return "FileAgent"; }
    std::string domain() const override { return "file"; }
    
    TaskResult execute(const Task& task) override {
        // 交给LangGraph处理
        return workflow_->run(task);
    }
};
```

### 4.2 LangChain / LangGraph 集成

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LangGraph 工作流示例 (FileAgent)                   │
│                                                                      │
│  用户输入: "把桌面的PDF整理到Documents"                              │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      LangGraph 工作流                        │   │
│  │                                                              │   │
│  │    ┌──────────┐                                             │   │
│  │    │  Start   │                                             │   │
│  │    └────┬─────┘                                             │   │
│  │         ▼                                                   │   │
│  │    ┌──────────┐                                             │   │
│  │    │ ParseDir │ 列出桌面文件                                 │   │
│  │    └────┬─────┘                                             │   │
│  │         ▼                                                   │   │
│  │    ┌──────────┐                                             │   │
│  │    │FilterPDF │ 筛选PDF                                     │   │
│  │    └────┬─────┘                                             │   │
│  │         ▼                                                   │   │
│  │    ┌──────────┐                                             │   │
│  │    │MoveFiles │ 移动到Documents/PDF                         │   │
│  │    └────┬─────┘                                             │   │
│  │         ▼                                                   │   │
│  │    ┌──────────┐                                             │   │
│  │    │  Report  │ 返回结果                                    │   │
│  │    └────┬─────┘                                             │   │
│  │         ▼                                                   │   │
│  │    ┌──────────┐                                             │   │
│  │    │  End     │                                             │   │
│  │    └──────────┘                                             │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  每个节点可能是:                                                     │
│  - LLM决策节点 (用LLM决定下一步)                                     │
│  - 函数节点 (调用Function Module)                                    │
│  - 条件节点 (if/else分支)                                           │
│  - 循环节点 (遍历处理)                                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 工作流配置文件示例 (YAML)

```yaml
# FileAgent 工作流配置
name: FileAgent
domain: file

nodes:
  - id: parse_dir
    type: function
    module: file_ops
    function: list_directory
    params:
      path: "{{ input.path }}"
      
  - id: filter_pdf
    type: function  
    module: file_ops
    function: filter_by_extension
    params:
      files: "{{ parse_dir.result }}"
      extension: ".pdf"
      
  - id: move_files
    type: function
    module: file_ops
    function: move_files
    params:
      files: "{{ filter_pdf.result }}"
      dest: "{{ input.dest }}"
      
  - id: report
    type: llm_decision
    prompt: "总结移动了哪些文件"

edges:
  - from: start
    to: parse_dir
  - from: parse_dir
    to: filter_pdf
  - from: filter_pdf
    to: move_files
  - from: move_files
    to: report
  - from: report
    to: end
```

---

## 五、Function Modules (功能模块)

### 5.1 模块接口

```cpp
// 功能模块接口
class IFunctionModule {
public:
    virtual ~IFunctionModule() = default;
    
    // 模块信息
    virtual std::string name() const = 0;
    virtual std::string version() const = 0;
    
    // 初始化/销毁
    virtual bool init() = 0;
    virtual void shutdown() = 0;
    
    // 获取提供的函数列表
    virtual std::vector<FunctionDef> list_functions() = 0;
    
    // 调用函数
    virtual FunctionResult call(const std::string& func_name, 
                                 const Value& args) = 0;
};

// 函数定义
struct FunctionDef {
    std::string name;           // "list_directory"
    std::string description;     // "列出目录下的文件"
    std::vector<ParamDef> params;  // 参数定义
    std::string return_type;    // 返回类型
};

// 函数调用结果
struct FunctionResult {
    bool success;
    Value result;
    std::string error;
};
```

### 5.2 预定义功能模块

```
┌─────────────────────────────────────────────────────────────────────┐
│                    预定义 Function Modules                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  file_ops (文件操作模块)                                       │  │
│  │  - list_directory(path) → [files]                            │  │
│  │  - read_file(path) → content                                 │  │
│  │  - write_file(path, content) → success                       │  │
│  │  - move_file(src, dest) → success                            │  │
│  │  - delete_file(path) → success                               │  │
│  │  - search_files(pattern) → [matches]                         │  │
│  │  - get_file_info(path) → FileInfo                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  system_ops (系统操作模块)                                    │  │
│  │  - run_command(cmd) → output                                │  │
│  │  - get_env(key) → value                                     │  │
│  │  - list_processes() → [processes]                           │  │
│  │  - get_system_info() → SystemInfo                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  network_ops (网络操作模块)                                    │  │
│  │  - http_get(url) → response                                 │  │
│  │  - http_post(url, body) → response                          │  │
│  │  - dns_lookup(domain) → ip                                   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  memory_ops (记忆操作模块)                                    │  │
│  │  - memory_search(query, top_k) → [results]                  │  │
│  │  - memory_write(content, layer) → success                    │  │
│  │  - memory_read(key) → value                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  llm_ops (LLM操作模块)                                        │  │
│  │  - chat(model, messages) → response                         │  │
│  │  - embed(text) → embedding                                  │  │
│  │  - stream_chat(model, messages) → stream                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 模块加载机制

```cpp
// Domain Agent 的模块加载
class DomainAgent {
    std::vector<std::unique_ptr<IFunctionModule>> modules_;
    
public:
    // 加载模块
    bool load_module(const std::string& module_path) {
        // 1. dlopen 动态加载
        void* handle = dlopen(module_path.c_str(), RTLD_LAZY);
        if (!handle) return false;
        
        // 2. 获取工厂函数
        auto factory = (ModuleFactory)dlsym(handle, "create_module");
        if (!factory) return false;
        
        // 3. 创建实例
        auto module = factory();
        if (!module->init()) return false;
        
        modules_.push_back(std::unique_ptr<IFunctionModule>(module));
        return true;
    }
    
    // 调用模块函数
    FunctionResult call_function(const std::string& func_name, 
                                   const Value& args) {
        // 1. 查找函数所在模块
        for (auto& module : modules_) {
            auto funcs = module->list_functions();
            if (std::find_if(funcs.begin(), funcs.end(),
                    [&func_name](auto& f) { return f.name == func_name; })
                    != funcs.end()) {
                // 2. 调用
                return module->call(func_name, args);
            }
        }
        return FunctionResult::error("Function not found: " + func_name);
    }
};
```

### 5.4 模块注册到Agent

```cpp
// Agent模块配置
struct AgentConfig {
    std::string name;
    std::string langgraph_config;  // 工作流配置路径
    std::vector<std::string> module_paths;  // 功能模块路径列表
};

// 初始化Agent时加载
void FileAgent::init() {
    // 加载LangGraph工作流
    load_workflow(langgraph_config_);
    
    // 加载功能模块
    for (auto& path : module_paths_) {
        load_module(path);
    }
}
```

---

## 六、协作流程

### 6.1 简单任务 (单Agent)

```
用户: "帮我列出桌面的文件"

┌─────────────────────────────────────────────────────────────────────┐
│  Central Agent                                                      │
│     │                                                                │
│     │ 1. 意图分析 → domain="file", action="list"                  │
│     │                                                                │
│     ▼                                                                │
│  AgentRegistry.find("file") → FileAgent                            │
│     │                                                                │
│     ▼                                                                │
│  FileAgent.execute(task)                                           │
│     │                                                                │
│     │ ┌─────────────────────────────────────────────────────┐     │
│     │ │ LangGraph: list_directory → return files            │     │
│     │ └─────────────────────────────────────────────────────┘     │
│     │                                                                │
│     ▼                                                                │
│  返回结果给用户                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 复杂任务 (多Agent协作)

```
用户: "把桌面的PDF发到张总邮箱"

┌─────────────────────────────────────────────────────────────────────┐
│  Central Agent                                                      │
│     │                                                                │
│     │ 意图分析 → 需要 FileAgent + MailAgent                        │
│     │                                                                │
│     ▼                                                                │
│  ┌──────────────────┐     ┌──────────────────┐                     │
│  │   FileAgent      │     │   MailAgent      │                     │
│  │                  │     │                  │                     │
│  │ 1.列出桌面文件   │     │                  │                     │
│  │ 2.筛选PDF       │────▶│ 4.发送邮件       │                     │
│  │ 3.准备附件      │     │   (附带PDF)      │                     │
│  └──────────────────┘     └──────────────────┘                     │
│         │                          ▲                                │
│         └──────────────────────────┘                                │
│                    IPC 调用                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Context Chain: FileAgent(1) → MailAgent(2)
Trace ID: "0/1/2"
```

---

## 七、与旧设计对比

| 项目 | 旧设计 (Skills) | 新设计 (Domain Agent) |
|------|-----------------|------------------------|
| **结构** | Layer 4 Skills → Layer 5 Agents | Layer 5 统一 |
| **功能载体** | Skills 是独立模块 | Domain Agent 自带功能 |
| **编排** | Agent 调用 Skills | LangChain/LangGraph 内嵌 |
| **复用性** | Skills 可被多Agent调用 | 功能模块绑定Agent |
| **复杂度** | 多层间接调用 | 直接调用 |
| **扩展方式** | 新增 Skill | 新增 Domain Agent |

---

## 八、优势总结

```
1. 更内聚
   Domain Agent = 领域专家 = 自己能干活的人

2. 更简单
   Central Agent 只做调度，不管具体实现

3. 更像操作系统
   Linux: 内核调度进程，进程自带功能
   AIOS: Central调度Agent，Agent自带功能模块

4. 易于扩展
   新领域 = 新Agent + 注册
   新功能 = 给Agent加模块
```

---

## 九、下一步

```
Phase 1: 实现 Central Agent + Registry + Dispatcher
Phase 2: 实现 2-3 个基础 Domain Agent (FileAgent, SysAgent)
Phase 3: 实现 LangGraph 工作流引擎
Phase 4: 实现 Function Module 加载机制
Phase 5: 添加更多 Domain Agent
```

---

*文档版本: v2.0*  
*创建时间: 2026-04-07*  
*理念: Domain Agent = 领域专家，自带LangChain编排 + 功能模块*

## 十八、Layer 3 安全审计层重构 (v2.1)

### 18.1 核心转型

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Layer 3 职责转型                                      │
│                                                                      │
│  旧职责: IO操作层 (所有操作过Layer 3)                              │
│  新职责: 安全审计层 (只做安全检查，不再做IO)                        │
│                                                                      │
│  转型原因:                                                          │
│  - 高频IO (如写代码) 走Layer 3开销大                              │
│  - 大文件下载没有流量控制 → 磁盘打满                               │
│  - Layer 3应该做安全守门员，不是IO中间件                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 18.2 新架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Layer 5: Application Layer                         │
│                                                                      │
│  Central Agent + Domain Agents                                       │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Domain Agent (内嵌Sandbox)                      │    │
│  │                                                              │    │
│  │   Sandbox (Agent私有工作区)                                  │    │
│  │   /root/aios/agents/{id}/sandbox/                          │    │
│  │       ├── code/     (自由写代码)                           │    │
│  │       ├── downloads/ (自由下载)                             │    │
│  │       └── output/   (结果输出)                             │    │
│  │                                                              │    │
│  │   ✅ 自由IO，无Layer 3开销                                  │    │
│  │   ✅ 高性能写入                                              │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                        │
│                              │ commit(result)                       │
│                              ▼                                        │
├─────────────────────────────────────────────────────────────────────┤
│                    Layer 3: 安全审计层                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Security Audit                             │    │
│  │                                                              │    │
│  │   1. PathGuard     - 路径审计 (不能写系统目录)             │    │
│  │   2. CapChecker    - 能力审计 (检查Capability)              │    │
│  │   3. QuotaGuard   - 配额审计 (磁盘/流量/API)              │    │
│  │   4. AuditLog     - 审计日志 (记录所有敏感操作)            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 18.3 目录结构

```
/root/aios/
├── agents/
│   ├── 1/                      # Agent PID 1
│   │   ├── sandbox/            # Agent私有工作区 (自由IO)
│   │   │   ├── code/           # 代码文件 (自由读写)
│   │   │   ├── downloads/      # 下载文件 (流式写入)
│   │   │   ├── temp/          # 临时文件
│   │   │   └── output/        # 输出结果
│   │   ├── work/               # 正式工作区 (commit后)
│   │   └── quota.json          # 配额信息
│   │
│   ├── 2/                      # Agent PID 2
│   │   └── ...
│   │
│   └── shared/                  # 共享目录 (需授权)
│
└── system/                     # 系统目录 (受保护)
    └── audit/                   # 审计日志
```

### 18.4 安全规则

```
允许的操作:
✅ /root/aios/agents/{agent_id}/sandbox/...  (Sandbox内自由)
✅ /root/aios/agents/{agent_id}/work/...    (工作区)
✅ /root/aios/shared/...           (共享目录，需SHARED_WRITE)

禁止的操作:
❌ /etc/**, /usr/**, /bin/**, /lib/**  (系统目录)
❌ /root/aios/agents/{other_id}/  (其他Agent目录)
❌ /root/**  (系统root，除非SYSTEM_ACCESS)
```

### 18.5 操作流程对比

```
旧流程 (所有操作过Layer 3):
Agent → Layer 3 (open) → Layer 3 (write) → Layer 3 (close) → 磁盘
问题: 每次write都过Layer 3，性能差

新流程 (Sandbox自由IO + 结果审计):
Agent → Sandbox (直接写) → 磁盘
                    ↓
                commit(result)
                    ↓
            Layer 3 (安全审计)

下载流程:
Agent → Sandbox (流式下载) → 磁盘
                    ↓
                commit(result)
                    ↓
            Layer 3 (磁盘配额检查)
```

### 18.6 安全审计接口

```cpp
// 安全审计检查
struct SecurityAudit {
    // 1. 路径检查
    bool check_path(pid_t agent_id, const std::string& path);
    
    // 2. 能力检查
    bool check_capability(pid_t agent_id, Capability cap);
    
    // 3. 配额检查
    bool check_quota(pid_t agent_id, const std::string& resource);
    
    // 4. 审计日志
    void audit(pid_t agent_id, const std::string& action, 
               bool allowed, const std::string& reason);
};

// commit时触发审计
struct CommitRequest {
    pid_t agent_id;
    std::vector<std::string> files;
    std::string dest_path;
};

SecurityResult audit_commit(const CommitRequest& req) {
    if (!security.check_path(req.agent_id, req.dest_path))
        return denied("Path not allowed");
    if (!security.check_quota(req.agent_id, "disk"))
        return denied("Disk quota exceeded");
    security.audit(req.agent_id, "commit", true, req.dest_path);
    return move_files(req.files, req.dest_path);
}
```

### 18.7 敏感操作列表 (必须过Layer 3审计)

```
系统级操作:
- agent_spawn    - 派生新Agent
- agent_kill     - 终止Agent
- system_access  - 访问系统目录
- cross_agent_access - 访问其他Agent

资源操作:
- commit         - 提交文件到正式位置
- shared_write   - 写共享目录
- quota_exceed    - 超出配额

API操作:
- llm_api_call   - LLM API调用
- external_api   - 外部API调用
```

### 18.8 配额系统

```cpp
struct DiskQuota {
    uint64_t max_total_bytes;      // Agent最大磁盘使用
    uint64_t max_single_file_bytes; // 单文件最大
    uint64_t warn_threshold;       // 告警阈值 (80%)
};

struct NetQuota {
    uint64_t max_download_bytes;   // 最大下载量
    uint64_t max_upload_bytes;     // 最大上传量
    uint64_t rate_limit;          // 速率限制 (MB/s)
};

struct ApiQuota {
    uint32_t max_llm_calls_per_min; // LLM调用频率
    uint32_t max_external_calls;    // 外部API调用次数
};

// 预定义配额
FileAgent:  {disk: 100GB, net: 50GB, api: 1000/min}
SysAgent:   {disk: 1GB,   net: 1GB,   api: 100/min}
NetAgent:   {disk: 10GB,  net: 100GB, api: 500/min}
```

### 18.9 实现计划

```
Phase 1: Sandbox + commit机制
- 每个Agent有独立sandbox目录
- IO操作在sandbox内自由执行
- commit时做安全审计

Phase 2: 流式下载支持
- 大文件流式写入sandbox
- 下载完成后再审计

Phase 3: 配额系统
- 磁盘配额
- 流量配额
- API调用配额

Phase 4: 告警系统
- 配额使用超过80%告警
- 敏感操作告警
```

---

*文档版本: v2.1*  
*创建时间: 2026-04-08*  
*更新: 2026-04-08 00:20 - Layer 3转型为安全审计层*  
*理念: Layer 3不做IO，只做安全守门员*

## 十九、Central Agent 交互流程优化

### 19.1 完整交互流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    用户与AIOS交互流程                                  │
│                                                                      │
│  1. 用户输入                                                         │
│     用户 ──▶ 飞书 ──▶ Central Agent                                │
│                        "帮我把桌面PDF整理到Documents"                  │
│                                                                      │
│  2. Central Agent 意图理解                                           │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  1. 意图解析: domain="file", action="organize"         │     │
│     │  2. 参数提取: src="/Desktop", ext=".pdf", dest="..."   │     │
│     │  3. 任务拆分: 列出→筛选→移动→报告                      │     │
│     │  4. 选择Agent: FileAgent                                 │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                       │
│  3. 分发给Domain Agent                                              │
│     Central Agent ──▶ FileAgent                                    │
│                              │                                       │
│  4. Domain Agent 执行                                                │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  LangGraph工作流:                                        │     │
│     │  Start → ParseDir → FilterPDF → MoveFiles → Report      │     │
│     │                                                          │     │
│     │  Sandbox内自由执行:                                       │     │
│     │  - 列出/Desktop/*                                       │     │
│     │  - 筛选.pdf                                             │     │
│     │  - 移动到/Documents/PDF/                               │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                       │
│  5. 结果返回                                                         │
│     FileAgent ──▶ Central Agent ──▶ 飞书 ──▶ 用户                  │
│                        "已整理X个PDF文件到Documents/PDF"           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.2 Central Agent 核心职责

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Central Agent 核心职责                              │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  1. 意图理解 (Intent Understanding)                           │  │
│  │     - 解析用户输入                                           │  │
│  │     - 提取关键信息 (domain, action, entities)               │  │
│  │     - LLM辅助理解模糊需求                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  2. 任务规划 (Task Planning)                                  │  │
│  │     - 拆解复杂任务为子任务                                    │  │
│  │     - 确定执行顺序                                            │  │
│  │     - 决定是否需要多Agent协作                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  3. Agent调度 (Agent Dispatch)                                │  │
│  │     - 查注册表找到对应Domain Agent                           │  │
│  │     - 传递任务上下文                                          │  │
│  │     - 监控执行状态                                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  4. 结果整合 (Result Integration)                             │  │
│  │     - 收集Domain Agent返回                                    │  │
│  │     - 多Agent结果合并                                         │  │
│  │     - 格式化输出                                              │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.3 Central Agent 决策逻辑

```cpp
class CentralAgent {
    AgentRegistry* registry_;
    
    // 处理用户输入
    Response process_user_input(const string& user_input) {
        // 1. 意图理解
        Intent intent = understand_intent(user_input);
        
        // 2. 简单任务直接处理
        if (intent.is_simple()) {
            return handle_simple(intent);
        }
        
        // 3. 复杂任务分发给Domain Agent
        vector<Task> tasks = plan_tasks(intent);
        
        if (tasks.size() == 1) {
            auto agent = registry_->find_by_domain(intent.domain);
            return agent->execute(tasks[0]);
        } else {
            return execute_multi_agent(tasks);
        }
    }
    
    // 意图理解
    Intent understand_intent(const string& input) {
        // 使用LLM辅助解析
        // 返回: domain, action, entities, complexity, urgency
    }
    
    // 任务规划
    vector<Task> plan_tasks(const Intent& intent) {
        if (intent.complexity < SIMPLE_THRESHOLD) {
            return single_agent_plan(intent);
        } else {
            return multi_agent_plan(intent);
        }
    }
};
```

### 19.4 多Agent协作示例

```
用户: "把桌面PDF发到张总邮箱，同时告诉我天气"

┌─────────────────────────────────────────────────────────────────────┐
│  Central Agent 决策                                                │
│                                                                      │
│  1. 识别两个独立任务:                                              │
│     - Task A: 发邮件 (domain=mail)                                │
│     - Task B: 查天气 (domain=network)                             │
│                                                                      │
│  2. 并行分发:                                                      │
│     ┌─────────────────┐    ┌─────────────────┐                  │
│     │ FileAgent       │    │ NetAgent       │                  │
│     │ (准备附件)      │    │ (查询天气)     │                  │
│     └────────┬────────┘    └────────┬────────┘                  │
│              │                      │                              │
│              └──────────┬───────────┘                              │
│                         │ MailAgent                               │
│                         │ (发送邮件)                              │
│                         ▼                                         │
│                    Central Agent                                   │
│                                                                      │
│  3. 结果整合:                                                      │
│     - 邮件发送成功                                                  │
│     - 张家界今天晴，25度                                           │
│                         │                                          │
│                         ▼                                          │
│                    用户看到完整回复                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.5 用户对话示例

```
用户: "帮我整理桌面"

    │
    ▼
Central Agent: [理解意图] "用户想整理桌面文件"
    │
    ▼
Central Agent: [规划任务] "需要FileAgent处理"
    │
    ▼
FileAgent: [执行] "正在列出桌面文件..."
    │
    ▼
Central Agent: [整合结果]
"桌面有3个PDF和2个图片，已整理到Documents对应文件夹"
```

### 19.6 与Domain Agent的关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Central Agent vs Domain Agent                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Central Agent (PID 0)                                              │
│  - 用户入口，所有请求都经过它                                       │
│  - 不做具体业务，只做调度和整合                                    │
│  - 类似Linux的init进程                                             │
│                                                                      │
│  Domain Agent                                                       │
│  - 执行具体任务                                                    │
│  - 在Sandbox内自由IO                                               │
│  - 返回结果给Central Agent                                          │
│                                                                      │
│  关系:                                                              │
│  Central Agent ──调度──▶ Domain Agent                              │
│       ◀───结果──────                                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

*文档版本: v2.1*  
*创建时间: 2026-04-08*  
*更新: 2026-04-08 00:26 - 添加第十九章：Central Agent交互流程优化*  
*理念: Central Agent是用户入口，负责意图理解、任务规划和结果整合*

## 二十、Central Agent 超级记忆系统

### 20.1 核心问题

```
主动加载 → 上下文爆炸 → 幻觉飙升
被动加载 → 记忆被忽视 → 遗忘重要信息

解决: 向量数据库 + 渐进披露 + 强制梯度加载
```

### 20.2 完整架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Central Agent 超级记忆系统                              │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    主Agent (Central Agent)                      │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  记忆钩子 (Memory Hooks)                           │  │   │
│  │  │  on_request: memory_search                          │  │   │
│  │  │  on_response: memory_store                         │  │   │
│  │  │  on_error: memory_log_error                        │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  提示词模板 (必须包含)                              │  │   │
│  │  │  "如果信息不存在于记忆，使用memory_search工具"    │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    记忆Agent (Memory Agent)                     │   │
│  │                                                              │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐│   │
│  │  │MemorySearch │  │ MemoryStore  │  │MemoryRefine ││   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘│   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  向量数据库 (LanceDB)                               │  │   │
│  │  │  - 语义搜索 + 时间过滤 + 分块处理                   │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    记忆存储层                               │   │
│  │                                                              │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │   │
│  │  │短期记忆 │  │中期记忆 │  │长期记忆 │  │向量存储 │  │   │
│  │  │(24h)   │  │(7d)    │  │(MEM.md) │  │(LanceDB)│  │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    自动整理 (Memory GC)                       │   │
│  │  每天凌晨3点执行                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.3 渐进式记忆披露流程

```
用户: "将昨天的文档整理发给我"

┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: 主Agent收到请求                                         │
│                                                                      │
│  - 意图理解: domain="file", action="organize"                 │
│  - 调用: memory_search("昨天的文档")                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 2: 记忆Agent返回                                            │
│                                                                      │
│  memory_search返回: [doc1路径, doc2路径, doc3路径]              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 3: 主Agent分发给FileAgent                                   │
│                                                                      │
│  主Agent不做具体处理，只分发:                                     │
│  - task: 整理文档                                                 │
│  - files: [doc1, doc2, doc3]                                     │
│                                                                      │
│  注意: 主Agent只做调度，业务处理交给Domain Agent                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 4: FileAgent执行 (Sandbox内)                               │
│                                                                      │
│  FileAgent做具体整理操作                                          │
│  返回: 整理结果给主Agent                                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Step 5: 主Agent整合结果 → 发给用户                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.4 记忆Agent系统指令格式

```cpp
// 记忆任务格式
struct MemorySearchTask {
    string query;           // "昨天的文档"
    string user_id;         // "bao"
    string time_range;      // "yesterday" / "7d" / "all"
    int max_results;        // 最多返回多少条
    int chunk_size;         // 防上下文爆炸的分块大小
};

struct MemoryStoreTask {
    string content;         // "AIOS架构讨论..."
    MemoryMetadata metadata; // {user_id, time, tags, importance}
};

struct MemoryRefineTask {
    string user_id;
    string date;            // "2026-04-07"
};
```

### 20.5 上下文防爆机制 (Chunking)

```
分块策略:

1. 时间分块
   - 最近1小时 / 今天 / 昨天 / 本周 / 更早
   - 优先级: 最近 > 久远

2. 相关度分块
   - top_k=5 (最多返回5条)
   - threshold=0.8 (低于阈值忽略)

3. 大小分块
   - 每块 max_token=2K
   - 超出就压缩摘要

4. 渐进披露
   - 第一轮: 返回top 3
   - 如果不够，主Agent再请求下一轮
```

### 20.6 主Agent记忆钩子

```
┌─────────────────────────────────────────────────────────────────────┐
│  主Agent提示词模板 (必须包含):                                     │
│                                                                      │
│  "你有一个memory_search工具，当你需要查找:                         │
│   - 用户提到的文档/文件信息                                        │
│   - 用户的偏好/习惯                                                │
│   - 之前的操作历史                                                  │
│   - 陌生的概念/术语                                                │
│   - 相关但不确定的事实                                              │
│   请立即调用memory_search，不要假设你知道"                        │
│                                                                      │
│  "你有一个memory_store工具，当你:                                  │
│   - 完成重要任务时                                                  │
│   - 用户确认某个决策时                                              │
│   - 产生值得记住的结果时                                            │
│   请调用memory_store保存"                                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  钩子触发点:                                                        │
│                                                                      │
│  on_request:                                                       │
│    - 检查是否需要memory_search                                     │
│    - 提取关键实体/时间/操作                                         │
│                                                                      │
│  on_response:                                                      │
│    - 检查是否需要memory_store                                     │
│    - 评估信息重要性                                                 │
│                                                                      │
│  on_error:                                                         │
│    - 记录错误上下文                                                 │
│    - 下次注意类似问题                                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.7 自动整理 (Memory GC)

```
每天凌晨3点自动执行:

1. 短期记忆整理 (short/ → medium/)
   - 将昨天的摘要转入中期记忆
   - 删除过期的临时信息
   - 合并同事件的记录

2. 向量数据库整理
   - 删除低相关度记录 (threshold < 0.3)
   - 合并相似记录
   - 补充缺失的时间戳

3. 长期记忆更新 (memory.md)
   - 提炼本周重要决策
   - 更新用户画像
   - 删除已过时的信息
```

### 20.8 记忆分级策略

```
必须保留:
- 用户明确确认的信息
- 重大决策

可以合并:
- 同话题的多条记录

可以删除:
- 临时任务
- 过时资讯
- 模糊引用

时间衰减:
- 最近7天: 完整保留
- 7-30天: 摘要保留
- 30天以上: 仅保留高重要性
```

### 20.9 时间在记忆中的重要性

```
向量数据库存储时间字段:
- 创建时间 (created_at)
- 访问时间 (accessed_at)
- 过期时间 (expires_at)

时间过滤查询:
- "昨天的文档" → time_range: yesterday
- "上周的文件" → time_range: 7d
- "最近的讨论" → time_range: 1d
- "所有记忆" → time_range: all

时间在整理中的作用:
- 优先保留近期记忆
- 久远记忆自动归档/删除
- 时间衰减相关度分数
```

---

*文档版本: v2.1*  
*创建时间: 2026-04-08*  
*更新: 2026-04-08 00:50 - 添加第二十章：Central Agent超级记忆系统*  
*理念: 向量数据库+渐进披露+强制梯度加载+记忆Agent+自动整理*
## 二十一、Agent架构重新定义 (v2.2)

### 21.1 核心变化

```
旧设计:
- FileAgent既是应用层又是系统层
- 所有任务都走同一个Agent

新设计:
- 应用层Agent: 处理用户任务
- 系统层Agent: 系统维护与安全 (定时任务调用)
```

### 21.2 新的Agent分类

```
Layer 5: 应用层 (Application Agents)
┌─────────────────────────────────────────────────────────────┐
│  应用Agents (处理用户任务)                               │
│                                                              │
│  DocAgent: 文档整理/搜索/发送
│  MailAgent: 邮件收发
│  SearchAgent: 信息检索/网络搜索
│  CodeAgent: 代码相关
│  NoteAgent: 笔记整理
│  MediaAgent: 图片/音视频处理
│  DataAgent: 数据分析/报表
└─────────────────────────────────────────────────────────────┘
                              ↓ 主Agent分发
Layer 3: 系统层 (System Agents)
┌─────────────────────────────────────────────────────────────┐
│  系统Agents (系统维护与安全)                             │
│                                                              │
│  FileAgent: 文件安全/磁盘监控/定时清理
│  SysAgent: 系统状态/进程管理
│  NetMonitorAgent: 网络流量监控
│  MemAgent: 记忆系统维护
│  LogAgent: 日志收集/分析
│  SecurityAgent: 安全扫描/异常检测
└─────────────────────────────────────────────────────────────┘
                              ↓ 主Agent定时任务
Layer 3: 安全审计层 (PathGuard/CapChecker/QuotaGuard/AuditLog)
```

### 21.3 应用层Agent (用户任务)

| Agent | 领域 | Capabilities |
|-------|------|--------------|
| DocAgent | 文档 | organize/search/send/convert/summarize |
| MailAgent | 邮件 | send/read/list/search/attachment |
| SearchAgent | 搜索 | web_search/knowledge_search/file_search |
| CodeAgent | 代码 | write/review/test/explain/refactor |
| NoteAgent | 笔记 | create/organize/link/search |
| MediaAgent | 媒体 | image_process/video_thumb/audio_transcribe |
| DataAgent | 数据 | analyze/visualize/report/transform |

### 21.4 系统层Agent (定时维护)

| Agent | 领域 | Capabilities | 调用方式 |
|-------|------|--------------|----------|
| FileAgent | 文件 | disk_monitor/security_scan/cleanup/quota_check | 定时任务 |
| SysAgent | 系统 | status_check/process_monitor/service_restart | 定时任务 |
| NetMonitorAgent | 网络 | traffic_monitor/connection_check/bandwidth_alert | 定时任务 |
| MemAgent | 记忆 | memory_gc/memory_compact/memory_backup | 每天凌晨 |
| LogAgent | 日志 | collect/analyze/alert/archive | 持续监控 |
| SecurityAgent | 安全 | vulnerability_scan/access_audit/threat_detect | 定时任务 |

### 21.5 交互流程

```
用户任务 → 主Agent → 应用层Agent (DocAgent等)
                         ↓
                   memory_search
                         ↓
                   整理文档 → 返回

系统维护 → 主Agent定时任务 → 系统层Agent (FileAgent等)
                                        ↓
                              磁盘监控/安全扫描/清理
```

### 21.6 主Agent调度逻辑

```cpp
// 用户任务 → 应用层Agent
Response handle_user_task(task) {
    if (task.type == "document_*") return doc_agent_->execute(task);
    if (task.type == "mail_*") return mail_agent_->execute(task);
    if (task.type == "code_*") return code_agent_->execute(task);
}

// 系统维护 → 系统层Agent (定时)
void run_periodic_maintenance() {
    // 每天凌晨3点
    file_agent_->disk_monitor();
    file_agent_->cleanup_expired();
    file_agent_->security_scan();
    
    // 每天凌晨4点
    sys_agent_->status_report();
    
    // 每天凌晨3点半
    mem_agent_->memory_gc();
}
```

---

*文档版本: v2.2*  
*更新: 2026-04-08 01:03 - Agent架构重新定义，应用层/系统层分离*
