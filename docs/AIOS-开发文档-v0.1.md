# AIOS 开发文档 v0.1

**需求文档**: AIOS 5层架构设计 v2.2
**开发版本**: v0.1
**创建时间**: 2026-04-08

---

## 一、项目概述

### 1.1 目标
基于AIOS 5层架构设计文档，实现一个操作系统级的多Agent系统。

### 1.2 核心技术点

| 技术点 | 描述 | 优先级 |
|--------|------|--------|
| 模块化架构 | Agent可插拔、热加载 | P0 |
| 多Agent通信 | IPC + EventChannel | P0 |
| 应用层Agent架构 | 统一接口、统一配置 | P0 |
| 主Agent保护 | 安全沙箱、权限控制 | P0 |
| 统一配置 | APIkey/模型统一配置 | P0 |
| 第三方Agent | 自定义模型/apikey | P1 |
| 热加载 | 无需重启添加Agent | P1 |

---

## 二、模块化架构

### 2.1 设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                    模块化设计原则                                      │
│                                                                      │
│  1. 接口隔离 (Interface Segregation)                                 │
│     - 所有Agent实现统一接口 IAgent                                   │
│     - 驱动实现统一接口 IDriver                                     │
│                                                                      │
│  2. 依赖倒置 (Dependency Inversion)                                 │
│     - 上层依赖接口，不依赖具体实现                                   │
│     - 具体实现可插拔                                                 │
│                                                                      │
│  3. 单一职责 (Single Responsibility)                                │
│     - 每个模块只做一件事                                             │
│     - Agent只负责特定领域                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块层次

```
┌─────────────────────────────────────────────────────────────────────┐
│                    模块层次结构                                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Application Layer (应用层)                               │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │   │
│  │  │ DocAgent│ │MailAgent│ │CodeAgent│ │ ...     │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Kernel Layer (内核层)                                    │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐              │   │
│  │  │Scheduler │ │IPC Bus   │ │Security   │              │   │
│  │  └───────────┘ └───────────┘ └───────────┘              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Infrastructure Layer (基础设施层)                          │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐              │   │
│  │  │LLM Provider│ │VectorDB  │ │Message GW │              │   │
│  │  └───────────┘ └───────────┘ └───────────┘              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 核心接口定义

```cpp
// ==================== Agent基础接口 ====================
class IAgent {
public:
    virtual ~IAgent() = default;
    
    // 元信息
    virtual std::string name() const = 0;
    virtual std::string domain() const = 0;
    virtual std::string version() const = 0;
    
    // 生命周期
    virtual bool init(const AgentConfig& config) = 0;
    virtual void shutdown() = 0;
    
    // 执行任务
    virtual TaskResult execute(const Task& task) = 0;
    
    // 健康检查
    virtual bool health_check() = 0;
};

// ==================== 应用层Agent接口 ====================
class IAppAgent : public IAgent {
public:
    // 获取能力列表
    virtual std::vector<std::string> capabilities() const = 0;
    
    // 获取工作流配置
    virtual std::string workflow_config() const = 0;
    
    // 获取使用的模型
    virtual std::string model() const = 0;
    
    // 是否使用自定义模型
    virtual bool use_custom_model() const = 0;
};

// ==================== 系统层Agent接口 ====================
class ISysAgent : public IAgent {
public:
    // 执行系统维护
    virtual void run_maintenance() = 0;
    
    // 获取系统状态
    virtual SystemStatus get_status() const = 0;
};
```

### 2.4 模块注册机制

```cpp
// ==================== Agent注册表 ====================
class AgentRegistry {
private:
    std::unordered_map<std::string, IAgent*> agents_;
    std::unordered_map<std::string, AgentMetadata> metadata_;
    
public:
    // 注册Agent
    bool register_agent(IAgent* agent, const AgentMetadata& metadata) {
        if (!agent || agents_.count(metadata.name)) {
            return false;
        }
        agents_[metadata.name] = agent;
        metadata_[metadata.name] = metadata;
        return true;
    }
    
    // 注销Agent
    bool unregister_agent(const std::string& name) {
        auto it = agents_.find(name);
        if (it == agents_.end()) return false;
        
        it->second->shutdown();
        agents_.erase(it);
        metadata_.erase(name);
        return true;
    }
    
    // 查找Agent
    IAgent* find(const std::string& name) const {
        auto it = agents_.find(name);
        return it != agents_.end() ? it->second : nullptr;
    }
    
    // 按领域查找
    std::vector<IAgent*> find_by_domain(const std::string& domain) const {
        std::vector<IAgent*> result;
        for (auto& [name, agent] : agents_) {
            if (metadata_.at(name).domain == domain) {
                result.push_back(agent);
            }
        }
        return result;
    }
    
    // 列出所有Agent
    std::vector<AgentMetadata> list_all() const {
        std::vector<AgentMetadata> result;
        for (auto& [name, meta] : metadata_) {
            result.push_back(meta);
        }
        return result;
    }
};
```

---

## 三、多Agent通信机制

### 3.1 通信架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    多Agent通信架构                                      │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    IPC Bus (进程间通信总线)                      │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  CentralAgent (PID 0)                               │  │   │
│  │  │      │                   │                           │  │   │
│  │  │      │ dispatch          │ result                   │  │   │
│  │  │      ▼                   ▼                           │  │   │
│  │  │  ┌─────────┐       ┌─────────┐                    │  │   │
│  │  │  │ AppAgent │◀─────▶│ SysAgent │                    │  │   │
│  │  │  └─────────┘       └─────────┘                    │  │   │
│  │  │      │                   │                           │  │   │
│  │  │      │       EventChannel │                         │  │   │
│  │  │      │◀─────────────────▶│                           │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    WaitQueue (等待队列)                          │   │
│  │  Agent等待 ──▶ 目标Agent完成 ──▶ 唤醒等待者                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 IPC消息格式

```cpp
// ==================== IPC消息 ====================
struct IPCMessage {
    uint64_t msg_id;              // 消息ID
    pid_t from;                   // 发送者PID
    pid_t to;                     // 接收者PID
    MessageType type;             // 消息类型
    uint64_t trace_id;            // 追踪ID
    
    // 内容
    std::string payload;          // JSON payload
    std::vector<std::string> attachments;  // 附件
    
    // 时间戳
    uint64_t timestamp;
    uint64_t timeout_ms;          // 超时时间
};

// 消息类型
enum class MessageType {
    TASK_DISPATCH,      // 任务分发
    TASK_RESULT,       // 任务结果
    TASK_PROGRESS,     // 进度更新
    HEARTBEAT,         // 心跳
    SHUTDOWN,          // 关闭
    ERROR,             // 错误
};
```

### 3.3 EventChannel

```cpp
// ==================== EventChannel ====================
class EventChannel {
private:
    int read_fd_;
    int write_fd_;
    pid_t owner_;                // 所属Agent
    EventType type_;
    
public:
    // 创建管道
    static std::unique_ptr<EventChannel> create(pid_t owner) {
        int fds[2];
        if (pipe(fds) < 0) return nullptr;
        
        return std::unique_ptr<EventChannel>(
            new EventChannel(fds[0], fds[1], owner));
    }
    
    // 发送事件
    bool send(const ChannelEvent& event) {
        return write(write_fd_, &event, sizeof(event)) == sizeof(event);
    }
    
    // 接收事件
    std::optional<ChannelEvent> recv() {
        ChannelEvent event;
        if (read(read_fd_, &event, sizeof(event)) == sizeof(event)) {
            return event;
        }
        return std::nullopt;
    }
    
    // 获取文件描述符 (用于epoll)
    int fd() const { return read_fd_; }
};
```

### 3.4 AgentWaitQueue

```cpp
// ==================== AgentWaitQueue ====================
class AgentWaitQueue {
private:
    struct WaitEntry {
        pid_t waiter;
        pid_t target;
        std::function<void(const TaskResult&)> callback;
        uint64_t timeout_ms;
        uint64_t enqueue_time;
    };
    
    std::deque<WaitEntry> waiters_;
    std::unordered_map<pid_t, std::deque<size_t>> target_index_;  // target → waiters index
    
public:
    // 等待某个Agent完成
    void wait(pid_t waiter, pid_t target, 
               std::function<void(const TaskResult&)> callback,
               uint64_t timeout_ms = 0) {
        WaitEntry entry{
            .waiter = waiter,
            .target = target,
            .callback = std::move(callback),
            .timeout_ms = timeout_ms,
            .enqueue_time = now_ms()
        };
        
        waiters_.push_back(entry);
        target_index_[target].push_back(waiters_.size() - 1);
    }
    
    // 唤醒等待者
    void wake_up(pid_t target, const TaskResult& result) {
        auto it = target_index_.find(target);
        if (it == target_index_.end()) return;
        
        for (size_t idx : it->second) {
            waiters_[idx].callback(result);
        }
        target_index_.erase(it);
    }
    
    // 检查超时
    void check_timeout() {
        uint64_t now = now_ms();
        std::erase_if(waiters_, [&](auto& e) {
            if (e.timeout_ms > 0 && (now - e.enqueue_time) > e.timeout_ms) {
                e.callback(TaskResult{.error = "TIMEOUT"});
                return true;
            }
            return false;
        });
    }
};
```

---

## 四、热加载机制

### 4.1 热加载架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Agent热加载架构                                     │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HotReloadManager                           │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  Watcher (监控目录变化)                            │  │   │
│  │  │  - agents/ 目录监控                                │  │   │
│  │  │  - *.so / *.dylib 动态库监控                      │  │   │
│  │  │  - config/*.json 配置文件监控                      │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  Loader (动态加载)                                  │  │   │
│  │  │  - dlopen/dlsym 加载                               │  │   │
│  │  │  - 工厂函数创建实例                                 │  │   │
│  │  │  - 健康检查                                         │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  Registry (注册表)                                    │  │   │
│  │  │  - register_agent                                   │  │   │
│  │  │  - notify_central                                 │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 动态库接口

```cpp
// ==================== Agent动态库接口 ====================
// agent_plugin.h

extern "C" {

// 工厂函数
typedef IAgent* (*AgentFactory)(const AgentConfig&);

// 获取Agent元信息
struct AgentPluginInfo {
    std::string name;
    std::string domain;
    std::string version;
    std::string description;
};

// 导出函数
AIOS_PLUGIN_API const char* agent_get_name();
AIOS_PLUGIN_API const char* agent_get_domain();
AIOS_PLUGIN_API const char* agent_get_version();
AIOS_PLUGIN_API IAgent* agent_create(const AgentConfig& config);
AIOS_PLUGIN_API void agent_destroy(IAgent* agent);

}
```

### 4.3 加载器实现

```cpp
// ==================== AgentLoader ====================
class AgentLoader {
private:
    struct LoadedModule {
        void* handle;
        IAgent* instance;
        AgentPluginInfo info;
    };
    
    std::unordered_map<std::string, LoadedModule> modules_;
    
public:
    // 加载Agent
    std::pair<IAgent*, AgentPluginInfo> load(const std::string& path) {
        // 1. dlopen
        void* handle = dlopen(path.c_str(), RTLD_LAZY);
        if (!handle) {
            throw std::runtime_error("dlopen failed: " + std::string(dlerror()));
        }
        
        // 2. 获取工厂函数
        auto factory = (AgentFactory)dlsym(handle, "agent_create");
        if (!factory) {
            dlclose(handle);
            throw std::runtime_error("agent_create not found");
        }
        
        // 3. 获取元信息
        AgentPluginInfo info;
        info.name = ((decltype(agent_get_name)*)dlsym(handle, "agent_get_name"))();
        info.domain = ((decltype(agent_get_domain)*)dlsym(handle, "agent_get_domain"))();
        info.version = ((decltype(agent_get_version)*)dlsym(handle, "agent_get_version"))();
        
        // 4. 创建实例
        IAgent* instance = factory(default_config());
        
        // 5. 健康检查
        if (!instance->health_check()) {
            instance->shutdown();
            dlclose(handle);
            throw std::runtime_error("health check failed");
        }
        
        modules_[info.name] = {handle, instance, info};
        return {instance, info};
    }
    
    // 卸载Agent
    void unload(const std::string& name) {
        auto it = modules_.find(name);
        if (it == modules_.end()) return;
        
        it->second.instance->shutdown();
        auto destroy = (decltype(agent_destroy)*)dlsym(it->second.handle, "agent_destroy");
        destroy(it->second.instance);
        dlclose(it->second.handle);
        modules_.erase(it);
    }
    
    // 热重载
    void reload(const std::string& name, const std::string& new_path) {
        unload(name);
        load(new_path);
    }
};
```

### 4.4 目录监控

```cpp
// ==================== HotReloadManager ====================
class HotReloadManager {
private:
    AgentLoader loader_;
    AgentRegistry* registry_;
    int watch_fd_;
    std::string agents_dir_;
    
public:
    void start(const std::string& agents_dir) {
        agents_dir_ = agents_dir;
        
        // 1. 创建inotify监控
        watch_fd_ = inotify_init();
        inotify_add_watch(watch_fd_, agents_dir_.c_str(), 
            IN_CREATE | IN_DELETE | IN_MODIFY);
        
        // 2. 启动监控线程
        std::thread([this] { watch_loop(); }).detach();
        
        // 3. 加载已有Agent
        load_existing_agents();
    }
    
private:
    void watch_loop() {
        char buf[1024];
        while (true) {
            int len = read(watch_fd_, buf, sizeof(buf));
            for (int i = 0; i < len; ) {
                auto* event = (inotify_event*)&buf[i];
                
                if (event->mask & IN_CREATE) {
                    if (ends_with(event->name, ".so") || ends_with(event->name, ".dylib")) {
                        load_new_agent(event->name);
                    }
                }
                else if (event->mask & IN_DELETE) {
                    unload_agent(event->name);
                }
                
                i += sizeof(inotify_event) + event->len;
            }
        }
    }
    
    void load_new_agent(const std::string& name) {
        auto path = agents_dir_ + "/" + name;
        try {
            auto [agent, info] = loader_.load(path);
            registry_->register_agent(agent, to_metadata(info));
            notify_central("agent_loaded", info.name);
        } catch (std::exception& e) {
            LOG(ERROR) << "Failed to load agent: " << e.what();
        }
    }
};
```

---

## 五、应用层Agent统一架构

### 5.1 统一架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    应用层Agent统一架构                                  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    BaseAgent (模板基类)                         │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  Common:                                          │  │   │
│  │  │    - name(), domain(), version()                   │  │   │
│  │  │    - init(), shutdown(), health_check()           │  │   │
│  │  │    - execute(), get_status()                      │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                          │                                  │   │
│  │                          ▼                                  │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  LangGraphExecutor:                               │  │   │
│  │  │    - workflow_                                     │  │   │
│  │  │    - execute_workflow()                           │  │   │
│  │  │    - node_handlers_                               │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ConcreteAgent (具体Agent)                      │   │
│  │                                                              │   │
│  │  DocAgent:                                                    │   │
│  │    - workflow_config() → YAML/JSON                          │   │
│  │    - node_handlers: [list_files, filter, move, ...]        │   │
│  │                                                              │   │
│  │  MailAgent:                                                   │   │
│  │    - workflow_config() → YAML/JSON                          │   │
│  │    - node_handlers: [compose, attach, send, ...]            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 BaseAgent模板

```cpp
// ==================== BaseAgent ====================
template<typename Derived>
class BaseAgent : public IAppAgent {
public:
    std::string name() const override { return derived().name(); }
    std::string domain() const override { return derived().domain(); }
    std::string version() const override { return derived().version(); }
    
    bool init(const AgentConfig& config) override {
        config_ = config;
        
        // 初始化LLM
        llm_client_ = LLMClientFactory::create(
            config_.llm.provider,
            config_.llm.api_key,
            config_.llm.model
        );
        
        // 初始化工作流
        auto workflow_path = config_.get_string("workflow");
        workflow_ = LangGraph::load(workflow_path);
        
        // 注册节点处理器
        register_node_handlers();
        
        return derived().on_init();
    }
    
    TaskResult execute(const Task& task) override {
        return workflow_.execute(task, node_handlers_);
    }
    
protected:
    AgentConfig config_;
    std::unique_ptr<ILLMClient> llm_client_;
    LangGraph workflow_;
    std::unordered_map<std::string, NodeHandler> node_handlers_;
    
private:
    Derived& derived() { return static_cast<Derived&>(*this); }
    
    void register_node_handlers() {
        // 注册通用处理器
        node_handlers_["llm_decision"] = [this](const Node& node) {
            return handle_llm_decision(node);
        };
        
        node_handlers_["function"] = [this](const Node& node) {
            return handle_function_call(node);
        };
        
        // 子类注册特定处理器
        derived().register_custom_handlers(node_handlers_);
    }
};

// 使用示例
class DocAgent : public BaseAgent<DocAgent> {
public:
    std::string name() const { return "DocAgent"; }
    std::string domain() const { return "document"; }
    
    void register_custom_handlers(NodeHandlerMap& handlers) {
        handlers["list_files"] = [this](const Node& n) { return handle_list(n); };
        handlers["filter_files"] = [this](const Node& n) { return handle_filter(n); };
        handlers["move_files"] = [this](const Node& n) { return handle_move(n); };
    }
};
```

### 5.3 工作流配置格式

```yaml
# DocAgent工作流配置
name: DocAgent
domain: document

nodes:
  - id: start
    type: start
    
  - id: parse_intent
    type: llm_decision
    prompt: |
      分析用户意图:
      - organize: 整理文件
      - search: 搜索文件
      - send: 发送文件
    output: intent
    
  - id: list_files
    type: function
    function: list_directory
    params:
      path: "{{ input.path }}"
      
  - id: filter_files
    type: function  
    function: filter_by_extension
    params:
      files: "{{ list_files.result }}"
      extension: "{{ intent.ext }}"
      
  - id: organize
    type: function
    function: move_files
    params:
      files: "{{ filter_files.result }}"
      dest: "{{ input.dest }}"
      
  - id: report
    type: llm_decision
    prompt: "总结操作结果"
    
  - id: end
    type: end

edges:
  - from: start → to: parse_intent
  - from: parse_intent → to: list_files
  - from: list_files → to: filter_files
  - from: filter_files → to: organize
  - from: organize → to: report
  - from: report → to: end
```

---

## 六、应用层通信机制

### 6.1 CentralAgent统一入口

```
┌─────────────────────────────────────────────────────────────────────┐
│                    应用层通信流程                                      │
│                                                                      │
│  用户 ──▶ 飞书 ──▶ CentralAgent                                    │
│                          │                                           │
│                          ▼                                           │
│                 ┌─────────────────┐                                  │
│                 │ Intent解析      │                                  │
│                 │ - domain       │                                  │
│                 │ - action       │                                  │
│                 │ - entities     │                                  │
│                 └────────┬────────┘                                  │
│                          │                                           │
│                          ▼                                           │
│                 ┌─────────────────┐                                  │
│                 │ Agent调度       │                                  │
│                 │ - 查注册表     │                                  │
│                 │ - 分发任务     │                                  │
│                 └────────┬────────┘                                  │
│                          │                                           │
│         ┌───────────────┼───────────────┐                          │
│         ▼               ▼               ▼                          │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐                     │
│    │DocAgent │    │MailAgent│    │CodeAgent│                     │
│    └────┬────┘    └────┬────┘    └────┬────┘                     │
│         │               │               │                          │
│         └───────────────┼───────────────┘                          │
│                         │                                          │
│                         ▼                                          │
│                 ┌─────────────────┐                                  │
│                 │ 结果整合        │                                  │
│                 │ - 格式化输出   │                                  │
│                 └────────┬────────┘                                  │
│                          │                                           │
│                          ▼                                           │
│                      飞书 ──▶ 用户                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 消息格式化

```cpp
// ==================== 统一消息格式 ====================
struct AppMessage {
    // 消息头
    MessageHeader header;
    
    // 内容
    MessageBody body;
    
    // 附件
    std::vector<Attachment> attachments;
};

struct MessageHeader {
    uint64_t msg_id;
    std::string trace_id;
    pid_t from;
    pid_t to;
    MessageType type;
    uint64_t timestamp;
};

struct MessageBody {
    std::string content_type;      // "text", "json", "markdown"
    std::string content;
    
    // 格式化
    FormatType format;            // "plain", "rich", "card"
    std::unordered_map<std::string, std::string> metadata;
};
```

---

## 七、主Agent保护机制

### 7.1 保护架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    主Agent保护架构                                     │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    安全边界 (Security Boundary)               │   │
│  │                                                              │   │
│  │  CentralAgent (PID 0)                                        │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  隔离运行环境                                         │  │   │
│  │  │  - 独立进程空间                                      │  │   │
│  │  │  - 受限系统调用                                      │  │   │
│  │  │  - 内存保护                                          │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    权限控制 (Capability Guard)                  │   │
│  │                                                              │   │
│  │  CentralAgent权限:                                           │   │
│  │  - 可以调度所有Agent                                         │   │
│  │  - 可以配置系统                                              │   │
│  │  - 可以管理Agent注册表                                        │   │
│  │  - 无法直接访问文件/网络                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    审计日志 (Audit Log)                        │   │
│  │                                                              │   │
│  │  所有操作记录到审计日志:                                      │   │
│  │  - Agent注册/注销                                          │   │
│  │  - 任务分发                                                │   │
│  │  - 敏感操作                                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 沙箱隔离

```cpp
// ==================== CentralAgent沙箱 ====================
class CentralAgentSandbox {
private:
    // 允许的系统调用白名单
    static const std::set<int> ALLOWED_SYSCALLS;
    
    // 允许的操作
    struct AllowedOps {
        bool can_spawn_agent = true;
        bool can_register_agent = true;
        bool can_dispatch_task = true;
        bool can_config_system = true;
        bool can_access_file = false;    // 禁止直接文件访问
        bool can_access_network = false;  // 禁止直接网络访问
    };
    
    AllowedOps allowed_ops_;
    
public:
    // 检查操作是否允许
    bool check_operation(const Operation& op) {
        switch (op.type) {
            case OpType::SPAWN_AGENT:
                return allowed_ops_.can_spawn_agent;
            case OpType::REGISTER_AGENT:
                return allowed_ops_.can_register_agent;
            case OpType::DISPATCH_TASK:
                return allowed_ops_.can_dispatch_task;
            case OpType::ACCESS_FILE:
                return false;  // 始终禁止
            case OpType::ACCESS_NETWORK:
                return false;  // 始终禁止
            default:
                return false;
        }
    }
    
    // 执行前验证
    bool validate(const Operation& op) {
        // 1. 检查操作类型
        if (!check_operation(op)) {
            audit_log_.log(op, DeniedReason::NOT_ALLOWED);
            return false;
        }
        
        // 2. 检查参数
        if (!validate_params(op)) {
            audit_log_.log(op, DeniedReason::INVALID_PARAMS);
            return false;
        }
        
        // 3. 记录审计
        audit_log_.log(op, DeniedReason::ALLOWED);
        return true;
    }
};
```

---

## 八、统一配置系统

### 8.1 配置架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    统一配置架构                                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    config.yaml (主配置)                         │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  llm:                                               │  │   │
│  │  │    provider: "openai"                              │  │   │
│  │  │    api_key: "${LLM_API_KEY}"                       │  │   │
│  │  │    model: "gpt-4"                                 │  │   │
│  │  │    base_url: "https://api.openai.com/v1"          │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  agents:  # 默认13个Agent配置                      │  │   │
│  │  │    default_model: "${llm.model}"                  │  │   │
│  │  │    default_provider: "${llm.provider}"            │  │   │
│  │  │    DocAgent:                                      │  │   │
│  │  │      enabled: true                               │  │   │
│  │  │      capabilities: ["organize", "search"]       │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 配置文件

```yaml
# config.yaml - AIOS主配置文件

# ==================== LLM配置 ====================
llm:
  provider: "openai"           # openai / claude / minimax
  api_key: "${OPENAI_API_KEY}"  # 环境变量引用
  model: "gpt-4o"
  base_url: "https://api.openai.com/v1"
  timeout: 60
  max_retries: 3

# ==================== 默认Agent配置 ====================
agents:
  # 默认配置 (所有Agent继承)
  defaults:
    provider: "${llm.provider}"    # 继承LLM配置
    model: "${llm.model}"
    api_key: "${llm.api_key}"
    timeout: 120
    retry: 3
    
  # 内置应用层Agent (7个)
  DocAgent:
    enabled: true
    domain: document
    capabilities:
      - organize
      - search
      - send
      - convert
      - summarize
    workflow: "agents/doc_agent/workflow.yaml"
    
  MailAgent:
    enabled: true
    domain: mail
    capabilities:
      - send
      - read
      - list
      - search
    workflow: "agents/mail_agent/workflow.yaml"
    
  SearchAgent:
    enabled: true
    domain: search
    capabilities:
      - web_search
      - knowledge_search
      - file_search
    workflow: "agents/search_agent/workflow.yaml"
    
  CodeAgent:
    enabled: true
    domain: code
    capabilities:
      - write
      - review
      - test
      - explain
    workflow: "agents/code_agent/workflow.yaml"
    
  NoteAgent:
    enabled: true
    domain: note
    capabilities:
      - create
      - organize
      - link
      - search
    workflow: "agents/note_agent/workflow.yaml"
    
  MediaAgent:
    enabled: true
    domain: media
    capabilities:
      - image_process
      - video_thumb
      - audio_transcribe
    workflow: "agents/media_agent/workflow.yaml"
    
  DataAgent:
    enabled: true
    domain: data
    capabilities:
      - analyze
      - visualize
      - report
    workflow: "agents/data_agent/workflow.yaml"
    
  # 内置系统层Agent (6个)
  FileAgent:
    enabled: true
    domain: file_system
    capabilities:
      - disk_monitor
      - security_scan
      - cleanup
      - quota_check
    workflow: "agents/file_agent/workflow.yaml"
    
  SysAgent:
    enabled: true
    domain: system
    capabilities:
      - status_check
      - process_monitor
      - service_restart
    workflow: "agents/sys_agent/workflow.yaml"
    
  NetMonitorAgent:
    enabled: true
    domain: network_monitor
    capabilities:
      - traffic_monitor
      - connection_check
      - bandwidth_alert
    workflow: "agents/net_monitor_agent/workflow.yaml"
    
  MemAgent:
    enabled: true
    domain: memory_maintenance
    capabilities:
      - memory_gc
      - memory_compact
      - memory_backup
    workflow: "agents/mem_agent/workflow.yaml"
    
  LogAgent:
    enabled: true
    domain: log
    capabilities:
      - collect
      - analyze
      - alert
      - archive
    workflow: "agents/log_agent/workflow.yaml"
    
  SecurityAgent:
    enabled: true
    domain: security
    capabilities:
      - vulnerability_scan
      - access_audit
      - threat_detect
    workflow: "agents/security_agent/workflow.yaml"

# ==================== 第三方Agent配置 ====================
# 后添加的Agent可以覆盖默认配置
custom_agents:
  CustomAgent:
    # 自定义模型/apikey
    provider: "claude"
    api_key: "${CUSTOM_API_KEY}"
    model: "claude-3-opus"
    # 自定义配置
    enabled: true
    domain: custom
    capabilities:
      - custom_action
    workflow: "agents/custom_agent/workflow.yaml"
```

### 8.3 配置加载器

```cpp
// ==================== ConfigLoader ====================
class ConfigLoader {
private:
    YAML::Node root_;
    std::unordered_map<std::string, std::string> env_vars_;
    
public:
    void load(const std::string& path) {
        root_ = YAML::LoadFile(path);
        
        // 加载环境变量
        for (auto& node : root_) {
            collect_env_vars(node);
        }
    }
    
    // 获取配置值 (支持${}插值)
    std::string get(const std::string& path) {
        std::string value = root_[path].as<std::string>();
        return resolve_env_vars(value);
    }
    
    // 获取配置节点
    YAML::Node get_node(const std::string& path) {
        return root_[path];
    }
    
private:
    // 解析 ${VAR} 或 ${parent.field}
    std::string resolve_env_vars(const std::string& value) {
        std::regex env_regex(R"(\$\{([^}]+)\})");
        std::string result = value;
        
        std::smatch match;
        while (std::regex_search(result, match, env_regex)) {
            std::string ref = match[1].str();
            std::string replacement;
            
            if (ref.find('.') == std::string::npos) {
                // 环境变量
                replacement = getenv(ref.c_str()) ?: "";
            } else {
                // 引用其他配置
                replacement = get(ref);
            }
            
            result.replace(match[0].first, match[0].second, replacement);
        }
        
        return result;
    }
};
```

### 8.4 Agent配置传递

```cpp
// ==================== AgentConfig ====================
struct AgentConfig {
    std::string name;
    std::string domain;
    std::vector<std::string> capabilities;
    
    // LLM配置
    std::string provider;      // openai / claude / minimax
    std::string api_key;
    std::string model;
    std::string base_url;
    
    // 超时配置
    uint32_t timeout;
    uint32_t max_retries;
    
    // 自定义配置
    std::unordered_map<std::string, std::string> extra;
    
    // 从YAML加载
    static AgentConfig from_yaml(const ConfigLoader& loader, 
                                 const std::string& agent_name,
                                 const YAML::Node& defaults) {
        AgentConfig config;
        config.name = agent_name;
        
        auto node = loader.get_node("agents." + agent_name);
        
        // 继承默认配置
        config.provider = loader.get(node["provider"].as<std::string>(defaults["provider"].as<std::string>()));
        config.model = loader.get(node["model"].as<std::string>(defaults["model"].as<std::string>()));
        config.api_key = loader.get(node["api_key"].as<std::string>(defaults["api_key"].as<std::string>()));
        
        // 如果有自定义配置，覆盖默认
        if (node["custom_provider"]) {
            config.provider = loader.get(node["custom_provider"].as<std::string>());
        }
        if (node["custom_api_key"]) {
            config.api_key = loader.get(node["custom_api_key"].as<std::string>());
        }
        
        return config;
    }
};
```

---

## 九、项目结构

```
aios/
├── config/
│   ├── config.yaml              # 主配置文件
│   └── agents/                  # Agent配置目录
│       ├── doc_agent.yaml
│       ├── mail_agent.yaml
│       └── ...
│
├── src/
│   ├── main.cc                 # 入口
│   │
│   ├── core/                   # 核心模块
│   │   ├── central_agent.cc
│   │   ├── agent_registry.cc
│   │   ├── scheduler.cc
│   │   ├── ipc_bus.cc
│   │   └── security.cc
│   │
│   ├── agents/                 # Agent实现
│   │   ├── base_agent.h
│   │   ├── app_agents/
│   │   │   ├── doc_agent.cc
│   │   │   ├── mail_agent.cc
│   │   │   └── ...
│   │   └── sys_agents/
│   │       ├── file_agent.cc
│   │       ├── sys_agent.cc
│   │       └── ...
│   │
│   ├── infra/                  # 基础设施
│   │   ├── llm/
│   │   │   ├── llm_client.h
│   │   │   ├── openai_client.cc
│   │   │   ├── claude_client.cc
│   │   │   └── minimax_client.cc
│   │   ├── memory/
│   │   │   ├── vector_db.cc
│   │   │   └── memory_agent.cc
│   │   └── message/
│   │       └── feishu_gateway.cc
│   │
│   ├── kernel/                 # 内核
│   │   ├── event_channel.cc
│   │   ├── wait_queue.cc
│   │   └── hot_reload.cc
│   │
│   └── langgraph/              # 工作流引擎
│       ├── langgraph.h
│       ├── executor.cc
│       └── node_handler.cc
│
├── agents/                     # Agent动态库
│   ├── doc_agent/
│   │   ├── agent.cc
│   │   ├── workflow.yaml
│   │   └── config.yaml
│   ├── mail_agent/
│   │   └── ...
│   └── custom_agent/           # 第三方Agent示例
│       └── ...
│
├── include/                    # 公共头文件
│   ├── aios_agent.h
│   ├── ipc_message.h
│   └── config.h
│
├── tests/                      # 测试
│   ├── unit/
│   └── integration/
│
├── BUILD                       # 构建文件
├── CMakeLists.txt
└── README.md
```

---

## 十、关键技术难点与解决方案

### 10.1 技术难点

| 难点 | 描述 | 解决方案 |
|------|------|----------|
| 模块化 | Agent可插拔、独立版本 | 动态库 + 工厂模式 + 注册机制 |
| 多Agent通信 | 进程间通信、事件通知 | IPC Bus + EventChannel + WaitQueue |
| 热加载 | 无需重启添加Agent | inotify监控 + dlopen/dlsym |
| 统一架构 | 应用层Agent统一模板 | BaseAgent模板 + LangGraph工作流 |
| 应用层通信 | 统一消息格式、格式化 | AppMessage + CentralAgent调度 |
| 主Agent保护 | 防止恶意Agent攻击 | 沙箱隔离 + 权限白名单 + 审计日志 |
| 统一配置 | APIkey/模型统一管理 | config.yaml + 环境变量插值 |

### 10.2 第三方Agent接入

```cpp
// 第三方Agent只需要实现以下接口

extern "C" {

// 1. 元信息
const char* agent_get_name() { return "MyCustomAgent"; }
const char* agent_get_domain() { return "custom"; }
const char* agent_get_version() { return "1.0.0"; }

// 2. 创建实例
IAgent* agent_create(const AgentConfig& config) {
    return new MyCustomAgent(config);
}

// 3. 销毁实例
void agent_destroy(IAgent* agent) {
    delete agent;
}

}

// 4. 配置文件
# agents/custom_agent/config.yaml
custom_api_key: "${CUSTOM_AGENT_KEY}"  # 独立APIkey

// 5. 配置覆盖
# config.yaml
custom_agents:
  MyCustomAgent:
    provider: "claude"
    custom_api_key: "${MY_API_KEY}"
    model: "claude-3-opus"
```

---

*文档版本: v0.1*  
*创建时间: 2026-04-08*  
*状态: 开发中*
