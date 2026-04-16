# AIOS 架构设计 v2.0
## 基于中央Agent的层级式编排体系

**设计理念：** 模仿 Linux 内核的 PID 0 (swapper/init) 架构 —— 所有进程由 init 派生，中央 Agent 是所有领域 Agent 的父进程，负责任务分发与调度。

---

## 一、设计原则

1. **中央Agent = PID 0** — 所有任务的发起者、调度者、资源分配者
2. **领域Agent = 子进程** — 各司其职，执行具体任务
3. **层级通信** — 中央Agent ↔ 领域Agent 通过 IPC 通信
4. **单一职责** — 每个领域Agent只做一个领域的事
5. **可扩展** — 新增领域只需新增Agent，无需修改中央Agent

---

## 二、架构图

```
                    ┌─────────────────────────────────────┐
                    │        User (Human)                 │
                    │   自然语言 / 命令行 / API            │
                    └─────────────────┬───────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │     🌟 中央Agent (Agent-0)          │
                    │  ─────────────────────────────────  │
                    │  • 意图解析 (Intent Parsing)        │
                    │  • 任务分解 (Task Decomposition)    │
                    │  • 资源调度 (Resource Scheduling)  │
                    │  • 结果聚合 (Result Aggregation)    │
                    │  • 故障恢复 (Fault Recovery)        │
                    │  • 进程管理 (类似PID 0)             │
                    └─────────────────┬───────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  📁 FileAgent   │       │  💻 SysAgent    │       │  🌐 NetAgent    │
│  文件整理Agent   │       │  系统软件Agent   │       │  网络管理Agent   │
│                 │       │                 │       │                 │
│  • 文件搜索      │       │  • 进程管理      │       │  • HTTP请求     │
│  • 文件分类      │       │  • 服务启停      │       │  • API调用      │
│  • 批量重命名    │       │  • 系统监控      │       │  • WebSocket    │
│  • 目录清理      │       │  • 注册表/配置   │       │  • DNS查询      │
│  • 权限管理      │       │                 │       │                 │
└────────┬────────┘       └────────┬────────┘       └────────┬────────┘
         │                         │                         │
         ▼                         ▼                         ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  📧 MailAgent   │       │  📊 DataAgent   │       │  🔧 ToolAgent   │
│  邮件Agent      │       │  数据处理Agent   │       │  工具调用Agent   │
│                 │       │                 │       │                 │
│  • 邮件读取      │       │  • 数据清洗      │       │  • Python调用   │
│  • 邮件发送      │       │  • 格式转换      │       │  • Shell执行    │
│  • 附件管理      │       │  • 统计分析      │       │  • API封装      │
│  • 规则过滤      │       │  • 可视化        │       │  • 插件系统     │
└─────────────────┘       └─────────────────┘       └─────────────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │   🔌 Plugin Agents (可扩展)   │
                    │   • DBAgent (数据库)          │
                    │   • ContainerAgent (容器)     │
                    │   • MonitorAgent (监控)        │
                    │   • SecurityAgent (安全)      │
                    └──────────────────────────────┘
```

---

## 三、中央Agent (Agent-0) 职责

### 3.1 核心功能

```cpp
class CentralAgent {
    // ==================== PID 0 角色 ====================
    
    // 1. 进程管理 - 派生/杀死领域Agent
    pid_t spawn(const DomainAgent& agent);
    bool kill(pid_t agent_pid);
    
    // 2. 任务分发 - 将任务分配给合适的Agent
    TaskDistributionResult distribute(const Task& task);
    
    // 3. 意图解析 - 理解用户想要什么
    Intent parse(const std::string& user_input);
    
    // 4. 任务分解 - 大任务拆成小任务
    std::vector<SubTask> decompose(const Intent& intent);
    
    // 5. 结果聚合 - 收集各Agent结果汇总
    std::string aggregate(const std::vector<AgentResult>& results);
    
    // 6. 故障恢复 - Agent挂了怎么办
    bool recover(pid_t failed_agent);
    
    // ==================== O(1) 调度器 ====================
    
    // O(1) 调度核心 (借鉴 Linux O(1) 调度器)
    ScheduleResult schedule();  // 调度复杂度 O(1)，与Agent数量无关
    
    // 资源限制 (cgroup-like)
    ResourceLimit getResourceLimit(pid_t agent_pid);
    
    // 进程状态监控
    AgentState getState(pid_t agent_pid);
};
```

### 3.2 O(1) 调度器设计

借鉴 Linux 2.6 O(1) 调度器的**优先级数组 + 位图扫描**机制：

```cpp
// ==================== Linux O(1) 调度器核心思想 ====================
// 
// Linux O(1) 调度器关键设计：
// 1. 两个优先级数组 (active[] 和 expired[]) - 避免遍历
// 2. 140 个优先级队列 (0-99 实时, 100-139 普通)
// 3. 位图标记哪些队列有任务 - 找最高优先级只需扫描位图
// 4. 数组切换只需指针交换 (swap) - O(1)
//
// AIOS 适配：
// - 每个领域Agent一个"优先级队列"
// - 任务按优先级入队
// - 调度时用位图找最高优先级非空队列
// - 切换只需指针交换
//

// 优先级定义 (共8级，类似Linux的140级)
enum AgentPriority {
    PRIO_CRITICAL = 0,  // 关键任务 (安全/故障恢复)
    PRIO_HIGH    = 1,  // 高优先级 (用户交互)
    PRIO_MEDIUM  = 2,  // 中优先级 (文件操作)
    PRIO_LOW     = 3,  // 低优先级 (后台任务)
    PRIO_IDLE    = 4,  // 空闲任务 (整理/备份)
    PRIO_COUNT   = 5
};

// 任务结构
struct SchedulableTask {
    uint64_t task_id;
    pid_t    agent_pid;      // 由哪个Agent执行
    AgentPriority priority;  // 优先级
    uint32_t time_slice;    // 时间片 (ms)
    uint32_t remaining;      // 剩余时间
    bool     is_batch;      // 是否批任务
};

// 优先级队列 (每个优先级一个队列)
class PriorityQueue {
    std::deque<SchedulableTask> queue_;
    uint64_t bitmask_;  // 位图标记 (用于O(1)查找)
};

// O(1) 调度器
class O1Scheduler {
public:
    // 入队 - O(1)
    void enqueue(SchedulableTask task) {
        int p = task.priority;
        queues_[p].push_back(task);
        bitmask_ |= (1ULL << p);  // 设置位图
    }
    
    // 出队 - O(1)  (找最高优先级非空队列)
    std::optional<SchedulableTask> dequeue() {
        if (bitmask_ == 0) return std::nullopt;  // 无任务
        
        // 找到最低优先级的1 (最高优先级任务)
        int p = __builtin_ctzll(bitmask_);  // O(1) 位图扫描
        
        auto& q = queues_[p];
        auto task = q.front();
        q.pop_front();
        
        // 如果队列空了，清除位图
        if (q.empty()) {
            bitmask_ &= ~(1ULL << p);
        }
        
        return task;
    }
    
    // 优先级提升 (抢占式调度用)
    void promote(uint64_t task_id, AgentPriority new_prio) {
        // 从原队列移除，加入新优先级队列
    }
    
private:
    PriorityQueue queues_[PRIO_COUNT];  // 5个优先级队列
    uint64_t bitmask_;  // 位图：bit n=1 表示 queues_[n] 非空
};
```

### 调度流程

```
                    ┌─────────────────────────────────────────┐
                    │         O(1) 调度器状态                  │
                    │                                          │
                    │   bitmask_: 0b00101                      │
                    │                  ↑  ↑                    │
                    │                  │  └── PRIO_LOW (3) 有任务
                    │                  └──── PRIO_MEDIUM (2) 有任务
                    │                                          │
                    │   queues_:                               │
                    │   [0] PRIO_CRITICAL: empty             │
                    │   [1] PRIO_HIGH:    empty               │
                    │   [2] PRIO_MEDIUM:  [Task-A, Task-B]    │
                    │   [3] PRIO_LOW:     [Task-C]           │
                    │   [4] PRIO_IDLE:    empty              │
                    └─────────────────────────────────────────┘
                              │
                              │ dequeue()
                              │ 1. __builtin_ctzll(bitmask_) = 2  // O(1)
                              │ 2. queues_[2].front() = Task-A   // O(1)
                              ▼
                         Task-A 出队执行
```

### 3.3 任务分发流程

```
用户输入: "帮我整理桌面，把所有PDF移到Documents/PDF文件夹，然后发邮件给张总"

                    ┌──────────────────┐
                    │   中央Agent (0)   │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   意图解析        │
                    │  Intent:        │
                    │  - action: multi │
                    │  - sub_tasks:   │
                    │    1. file_move │
                    │    2. send_mail │
                    └────────┬─────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                         │
┌─────────▼─────────┐                   ┌─────────▼─────────┐
│   FileAgent       │                   │   MailAgent       │
│   (子Agent-1)     │                   │   (子Agent-2)     │
│                   │                   │                   │
│  1. 扫描桌面       │                   │  1. 登录邮箱      │
│  2. 筛选PDF       │                   │  2. 撰写邮件      │
│  3. 移动到目标    │                   │  3. 发送附件      │
│  4. 返回结果      │                   │  4. 返回发送状态  │
└─────────┬─────────┘                   └─────────┬─────────┘
          │                                         │
          └──────────────────┬──────────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   结果聚合        │
                    │   "已整理12个PDF │
                    │    邮件已发送给   │
                    │    张总"          │
                    └──────────────────┘
```

---

## 四、领域Agent (Domain Agents)

### 4.1 通用接口

```cpp
class DomainAgent {
public:
    virtual ~DomainAgent() = default;
    
    // Agent标识 (类似PID)
    pid_t pid() const { return pid_; }
    std::string name() const { return name_; }
    
    // 核心生命周期
    virtual Result execute(const Task& task) = 0;
    virtual bool init() = 0;
    virtual void shutdown() = 0;
    
    // 状态
    enum State { INIT, IDLE, BUSY, WAITING, DEAD };
    State state() const { return state_; }
    
protected:
    pid_t pid_;           // 由中央Agent分配
    std::string name_;
    State state_;
    CentralAgent* parent_; // 指向中央Agent
};
```

### 4.2 各领域Agent详细设计

#### 📁 FileAgent (文件整理专家)

```cpp
class FileAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // 核心能力
    std::vector<FileInfo> scan(const ScanRequest& req);
    bool move(const std::string& from, const std::string& to);
    bool copy(const std::string& from, const std::string& to);
    bool delete(const std::string& path);
    std::vector<FileInfo> search(const SearchQuery& query);
    
    // 整理能力
    bool organizeByType(const std::string& dir);
    bool organizeByDate(const std::string& dir);
    bool batchRename(const BatchRenameRequest& req);
    bool cleanDuplicates(const std::string& dir);
    
    // Windows专有
    bool manageShortcuts(bool create, const std::string& target);
    bool clearRecycleBin();
    
    // Linux专有
    bool setPermissions(const std::string& path, mode_t mode);
    bool createSymlink(const std::string& target, const std::string& link);
    
private:
    // 文件系统抽象 (兼容Windows/Linux)
    IFileSystem* fs_;
};
```

#### 💻 SysAgent (系统软件调用专家)

```cpp
class SysAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // 进程管理
    std::vector<ProcessInfo> listProcesses();
    bool startProcess(const std::string& app, const std::vector<std::string>& args);
    bool killProcess(int pid);
    bool suspendProcess(int pid);
    bool resumeProcess(int pid);
    
    // 服务管理
    // Windows: services.msc / sc.exe
    // Linux: systemctl / service
    std::vector<ServiceInfo> listServices();
    bool startService(const std::string& name);
    bool stopService(const std::string& name);
    bool restartService(const std::string& name);
    
    // 系统信息
    SystemInfo getSystemInfo();
    std::vector<WindowInfo> listWindows(); // Windows GUI
    
    // 注册表/配置
    // Windows: reg query / reg add
    // Linux: /etc files, sysctl
    bool setConfig(const ConfigItem& item);
    std::string getConfig(const std::string& key);
};
```

#### 🌐 NetAgent (网络管理专家)

```cpp
class NetAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // HTTP 请求
    HttpResponse httpGet(const std::string& url, const Headers& headers);
    HttpResponse httpPost(const std::string& url, const std::string& body);
    
    // WebSocket
    bool wsConnect(const std::string& url);
    bool wsSend(const std::string& message);
    std::string wsReceive();
    void wsClose();
    
    // 网络诊断
    bool ping(const std::string& host);
    std::vector<RouteHop> traceroute(const std::string& host);
    std::string dnsLookup(const std::string& domain);
    
    // 网络配置
    // Windows: netsh / ipconfig
    // Linux: ip / ifconfig / ss
    std::vector<NetworkInterface> listInterfaces();
    bool setInterfaceIP(const std::string& iface, const std::string& ip);
};
```

#### 📧 MailAgent (邮件专家)

```cpp
class MailAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // 邮件操作 (IMAP/SMTP)
    bool connect(const MailConfig& config);
    std::vector<MailInfo> listInbox(const ListFilter& filter);
    std::string readMail(int mail_id);
    bool sendMail(const MailMessage& msg);
    bool deleteMail(int mail_id);
    
    // 附件处理
    std::vector<std::string> extractAttachments(int mail_id, const std::string& saveDir);
    bool attachFiles(int mail_id, const std::vector<std::string>& files);
    
    // 规则过滤
    bool createRule(const MailRule& rule);
    bool applyRules(const std::string& folder);
};
```

#### 📊 DataAgent (数据处理专家)

```cpp
class DataAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // 数据读取
    DataFrame readCSV(const std::string& path);
    DataFrame readExcel(const std::string& path);
    DataFrame readJSON(const std::string& path);
    DataFrame readSQL(const std::string& query);
    
    // 数据清洗
    DataFrame clean(const DataFrame& df, const CleanRule& rule);
    bool handleMissing(const DataFrame& df, const MissingStrategy& strategy);
    DataFrame deduplicate(const DataFrame& df);
    
    // 数据转换
    DataFrame transform(const DataFrame& df, const TransformRule& rule);
    DataFrame join(const DataFrame& left, const DataFrame& right, const JoinCondition& cond);
    DataFrame aggregate(const DataFrame& df, const GroupByRule& rule);
    
    // 输出
    bool toCSV(const DataFrame& df, const std::string& path);
    bool toExcel(const DataFrame& df, const std::string& path);
};
```

#### 🔧 ToolAgent (工具调用专家)

```cpp
class ToolAgent : public DomainAgent {
public:
    Result execute(const Task& task) override;
    
    // Shell命令
    ShellResult exec(const std::string& cmd, int timeout_sec = 30);
    
    // Python脚本
    PyResult runPython(const std::string& script, const std::vector<std::string>& args);
    
    // API封装
    bool registerTool(const ToolDef& tool);
    bool unregisterTool(const std::string& tool_name);
    ToolResult callTool(const std::string& tool_name, const Json::Value& params);
    
    // 插件管理
    bool loadPlugin(const std::string& plugin_path);
    bool unloadPlugin(const std::string& plugin_name);
    std::vector<std::string> listPlugins();
};
```

---

## 五、IPC 通信机制

### 5.1 消息格式

```cpp
// 中央Agent ↔ 领域Agent 通信协议
struct AgentMessage {
    Header header;      // 消息头
    Payload payload;    // 消息体
};

struct Header {
    uint32_t magic;         // 魔数 0x41494F53 ('AIOS')
    uint16_t version;       // 协议版本
    uint16_t type;          // 消息类型
    uint32_t seq;           // 序列号
    pid_t from;            // 发送者PID
    pid_t to;              // 接收者PID
    uint64_t timestamp;     // 时间戳
};

enum MessageType {
    TASK_SUBMIT = 1,      // 任务提交
    TASK_RESULT = 2,      // 任务结果
    TASK_PROGRESS = 3,    // 进度报告
    HEARTBEAT = 4,        // 心跳
    SHUTDOWN = 5,          // 关闭指令
    SPAWN = 6,             // 派生新Agent
    KILL = 7,              // 杀死Agent
};
```

### 5.2 通信方式选择

| 场景 | 推荐方式 | 理由 |
|------|---------|------|
| 同主机 | 共享内存 + RingBuffer | 最低延迟 |
| 同主机备选 | Unix Domain Socket | 简单可靠 |
| 跨主机 | gRPC + Protobuf | 跨语言、成熟 |
| 跨主机备选 | ZeroMQ | 高性能 |

### 5.3 共享内存 RingBuffer 设计

```cpp
// 用于同主机高速通信
class SharedRingBuffer {
public:
    // 写入数据 (写Agent)
    size_t write(const void* data, size_t len);
    
    // 读取数据 (读Agent)
    size_t read(void* buf, size_t len);
    
    // 等待数据可用
    void waitForData(int timeout_ms);
    
private:
    void* shm_addr_;           // 共享内存地址
    size_t shm_size_;          // 共享内存大小
    volatile uint32_t* head_;  // 头指针 (原子操作)
    volatile uint32_t* tail_;   // 尾指针 (原子操作)
};
```

---

## 六、生命周期管理

### 6.1 Agent 启动流程

```
中央Agent启动
      │
      ▼
加载配置文件 (agents.yaml)
      │
      ▼
创建 IPC 通道 (共享内存/Socket)
      │
      ▼
派生 FileAgent ──► 初始化 ──► 注册到中央Agent
      │
      ├── SysAgent ──► 初始化 ──► 注册到中央Agent
      │
      ├── NetAgent ──► 初始化 ──► 注册到中央Agent
      │
      └── ... (按需派生)
```

### 6.2 任务执行流程

```
用户请求
    │
    ▼
中央Agent: 意图解析
    │
    ▼
中央Agent: 任务分解
    │
    ▼
选择目标Agent集合
    │
    ├──▶ FileAgent ──► 执行子任务 ──► 返回结果
    │
    └──▶ SysAgent ──► 执行子任务 ──► 返回结果
    │
    ▼
中央Agent: 结果聚合
    │
    ▼
返回给用户
```

### 6.3 故障恢复

```
Agent崩溃检测 (心跳超时)
    │
    ▼
记录错误日志
    │
    ▼
尝试重启Agent
    │
    ├── 成功 ──► 重新注册 ──► 继续服务
    │
    └── 失败 ──► 告警通知用户
    │
    ▼
返回部分结果 (如果其他Agent有结果)
```

---

## 七、安全机制

### 7.1 权限分级

```cpp
enum AgentPrivilege {
    PRIVILEGE_NONE = 0,      // 无权限
    PRIVILEGE_READ = 1,      // 只读
    PRIVILEGE_WRITE = 2,     // 写入
    PRIVILEGE_EXEC = 4,      // 执行
    PRIVILEGE_ADMIN = 7,     // 完全权限
};

// 每个Agent分配最小权限
FileAgent: PRIVILEGE_READ | PRIVILEGE_WRITE | PRIVILEGE_EXEC
MailAgent: PRIVILEGE_READ | PRIVILEGE_WRITE
SysAgent:  PRIVILEGE_EXEC  (最高权限中央Agent单独控制)
```

### 7.2 操作审计

```cpp
struct AuditLog {
    pid_t agent_pid;      // 操作者
    std::string action;    // 操作类型
    std::string target;   // 操作对象
    int result;           // 结果码
    uint64_t timestamp;  // 时间戳
};
```

---

## 八、与 Linux PID 0 的类比

| Linux | AIOS | 说明 |
|-------|------|------|
| PID 0 (swapper) | 中央Agent | 所有进程的祖先 |
| PID 1 (init) | 系统初始化Agent | 用户态第一个进程 |
| 普通进程 | 领域Agent | 执行具体任务 |
| fork()/exec() | spawn() | 创建子Agent |
| kill() | kill() | 终止子Agent |
| wait() | result_collect() | 收集子进程状态 |
| cgroups | resource_limits | 资源限制 |
| /proc/[pid]/ | 中央Agent状态查询 | 查看Agent状态 |

---

## 九、技术实现要点

### 9.1 C++ 实现

```cpp
// main.cpp
int main() {
    // 1. 初始化日志
    Logger::init("/var/log/aios.log");
    
    // 2. 加载配置
    Config config = Config::load("config/agents.yaml");
    
    // 3. 初始化IPC
    IPCContext ipc(config.ipc_type);
    
    // 4. 创建中央Agent
    CentralAgent agent0(config);
    agent0.registerSignalHandlers();
    
    // 5. 派生领域Agents
    agent0.spawn<FileAgent>("file");
    agent0.spawn<SysAgent>("sys");
    agent0.spawn<NetAgent>("net");
    
    // 6. 进入主循环
    agent0.run();
    
    return 0;
}
```

### 9.2 Python 绑定

```python
# pyaios/core.py
import pybind11_aios as aios

class CentralAgent:
    def __init__(self, config_path: str):
        self._impl = aios.CentralAgent(config_path)
    
    def submit_task(self, description: str) -> str:
        """提交自然语言任务"""
        return self._impl.submit(description)
    
    def get_agent_status(self) -> dict:
        """查看所有Agent状态"""
        return self._impl.status()
    
    def kill_agent(self, name: str) -> bool:
        """杀死某个Agent"""
        return self._impl.kill(name)

# 使用示例
agent = CentralAgent("config/aios.yaml")
result = agent.submit_task("帮我整理桌面上的PDF文件")
print(result)
```

---

## 十、下一步行动

1. **Phase 1.1**: 实现中央Agent骨架 (Agent-0)
2. **Phase 1.2**: 实现基础的 IPC 机制
3. **Phase 1.3**: 实现 2-3 个领域Agent (FileAgent优先)
4. **Phase 1.4**: 实现任务分发和结果聚合
5. **Phase 1.5**: MVP - "帮我整理桌面" 端到端测试

---

*文档版本: v2.0*  
*更新: 推倒重来，重新设计架构*  
*创建时间: 2026-04-06*
