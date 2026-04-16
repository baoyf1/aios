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

*文档版本: v1.3*  
*创建时间: 2026-04-07*  
*更新: 2026-04-07 21:30 - 添加第十三章：多Agent协作上下文*  
*理念: 操作系统级架构，高内聚低耦合*
