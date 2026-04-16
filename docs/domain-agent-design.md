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
