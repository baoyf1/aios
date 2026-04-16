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

*文档版本: v1.0*  
*创建时间: 2026-04-07*  
*理念: 模块化驱动架构，高内聚低耦合*
