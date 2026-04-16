# Agent事件驱动架构需求文档

**版本**: v1.0  
**章节**: 第七章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**状态**: 需求中  
**优先级**: P1

---

## 一、核心洞察

```
传统OS:  等待 文件fd  →  磁盘/网络IO完成
AIOS:    等待 Agent   →  LLM输出/任务完成

AIOS等待的是 Agent的输出，而不是文件描述符
```

---

## 二、混合架构：Event Channel + WaitQueue

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

---

## 三、Event Channel（事件通道）

### 3.1 Channel 结构

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

### 3.2 Channel 创建与分发

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

---

## 四、AgentWaitQueue（Agent等待队列）

### 4.1 WaitEntry 结构

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

### 4.2 WaitQueue 实现

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

---

## 五、完整的事件循环

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

---

## 六、Agent间协作示例

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

---

## 七、设计要点

| 组件 | 职责 | 核心机制 |
|------|------|----------|
| **EventChannel** | Agent输出通道 | pipe/eventfd + epoll |
| **ChannelManager** | 管道生命周期管理 | 创建/注册/分发 |
| **AgentWaitQueue** | Agent间依赖等待 | 回调 + 目标索引 |
| **GlobalEventLoop** | 统一事件循环 | epoll + wait_queue + schedule |

---

## 八、与原方案对比

| 方面 | 原方案(文件fd) | 新方案(Agent事件) |
|------|---------------|------------------|
| **等待对象** | 文件描述符 | Agent输出 |
| **事件源** | 磁盘/网络IO | LLM/子Agent |
| **分发机制** | fd→Context映射 | Channel + WaitQueue |
| **唤醒方式** | epoll返回 | 回调直接调用 |

---

## 九、验收标准

- [ ] EventChannel 结构定义完整
- [ ] ChannelManager 创建/注册/分发正确
- [ ] AgentWaitQueue 等待/唤醒机制正确
- [ ] GlobalEventLoop 事件循环正确
- [ ] Agent间协作示例测试通过
- [ ] 超时处理机制正确

---

*文档版本: v1.0*
*创建时间: 2026-04-10*
