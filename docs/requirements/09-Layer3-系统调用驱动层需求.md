# Layer 3 系统调用/驱动层需求文档

**版本**: v1.0  
**章节**: 第九章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**状态**: 需求中  
**优先级**: P0

---

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

## 三、驱动模块

### 3.1 驱动列表

| 驱动 | 名称 | 优先级 | 说明 |
|-----|------|-------|------|
| FS_Driver | 文件系统驱动 | 100 | 文件读写、目录操作 |
| Net_Driver | 网络驱动 | 90 | HTTP请求、DNS查询 |
| LLM_Driver | LLM驱动 | 80 | 大语言模型调用 |
| Msg_Driver | 消息驱动 | 70 | 飞书、企业微信消息 |
| Mem_Driver | 记忆驱动 | 60 | 向量存储、检索 |
| Skill_Driver | Skill驱动 | 50 | Skill加载、调用 |
| MCP_Driver | MCP驱动 | 40 | MCP协议支持 |
| Proc_Driver | 进程驱动 | 30 | 进程管理 |

### 3.2 FS_Driver

```cpp
class FSDriver : public IDriver {
public:
    std::string name() const override { return "fs_driver"; }
    int priority() const override { return 100; }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_FILE_BASE && 
               req.syscall_id <= SYS_FILE_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_FILE_OPEN:  return do_open(req);
            case SYS_FILE_READ:  return do_read(req);
            case SYS_FILE_WRITE: return do_write(req);
            case SYS_FILE_CLOSE: return do_close(req);
            case SYS_FILE_STAT:  return do_stat(req);
            case SYS_DIR_LIST:   return do_listdir(req);
            default: return SysCallResult::Error(E_INVALID, "Unknown file syscall");
        }
    }
};
```

### 3.3 LLM_Driver

```cpp
class LLMDriver : public IDriver {
public:
    // LLM Provider配置
    struct ProviderConfig {
        std::string name;           // "openai", "claude", "minimax"
        std::string endpoint;
        std::string api_key;
        std::string default_model;
    };
    
    void add_provider(const ProviderConfig& config);
    void set_provider(const std::string& name);
    
    std::string name() const override { return "llm_driver"; }
    int priority() const override { return 80; }
    
    bool can_handle(const SysCallRequest& req) const override {
        return req.syscall_id >= SYS_LLM_BASE && 
               req.syscall_id <= SYS_LLM_END;
    }
    
    SysCallResult handle(const SysCallRequest& req) override {
        switch (req.syscall_id) {
            case SYS_LLM_CHAT:        return do_chat(req);
            case SYS_LLM_EMBED:       return do_embed(req);
            case SYS_LLM_STREAM_CHAT: return do_stream_chat(req);
            default: return SysCallResult::Error(E_INVALID, "Unknown LLM syscall");
        }
    }
};
```

---

## 四、系统调用号定义

### 4.1 调用号分配

```
┌─────────────────────────────────────────────────────────────────────┐
│                         系统调用号分配表                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SYS_FILE_BASE = 1000    文件操作                                    │
│    SYS_FILE_OPEN = 1001                                                │
│    SYS_FILE_READ = 1002                                               │
│    SYS_FILE_WRITE = 1003                                              │
│    SYS_FILE_CLOSE = 1004                                             │
│    SYS_FILE_STAT = 1005                                               │
│    SYS_DIR_LIST = 1006                                                │
│                                                                      │
│  SYS_NET_BASE = 2000    网络操作                                     │
│    SYS_NET_HTTP_GET = 2001                                            │
│    SYS_NET_HTTP_POST = 2002                                           │
│    SYS_NET_DNS_LOOKUP = 2003                                         │
│                                                                      │
│  SYS_LLM_BASE = 3000    LLM操作                                      │
│    SYS_LLM_CHAT = 3001                                                │
│    SYS_LLM_EMBED = 3002                                               │
│    SYS_LLM_STREAM_CHAT = 3003                                         │
│                                                                      │
│  SYS_MSG_BASE = 4000    消息操作                                     │
│    SYS_MSG_SEND = 4001                                                │
│    SYS_MSG_RECV = 4002                                                │
│    SYS_MSG_BROADCAST = 4003                                          │
│                                                                      │
│  SYS_MEM_BASE = 5000    记忆操作                                     │
│    SYS_MEM_READ = 5001                                                │
│    SYS_MEM_WRITE = 5002                                               │
│    SYS_MEM_SEARCH = 5003                                               │
│                                                                      │
│  SYS_PROC_BASE = 6000    进程操作                                    │
│    SYS_PROC_SPAWN = 6001                                              │
│    SYS_PROC_KILL = 6002                                               │
│    SYS_PROC_WAIT = 6003                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、验收标准

- [ ] IDriver基类定义完整
- [ ] DriverManager实现驱动注册、调度、卸载
- [ ] FS_Driver实现文件操作
- [ ] Net_Driver实现网络操作
- [ ] LLM_Driver实现LLM调用
- [ ] 系统调用号定义规范
- [ ] 驱动可插拔机制可用

---

*文档版本: v1.0*
*创建时间: 2026-04-10*
