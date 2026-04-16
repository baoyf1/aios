# 多Agent协作上下文需求文档

**版本**: v1.0  
**章节**: 第六章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**状态**: 需求中  
**优先级**: P1

---

## 一、协作场景分类

### 1.1 场景分类

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

---

## 二、上下文传递机制

### 2.1 Context Chain（上下文链）

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

---

## 三、上下文传递流程

### 3.1 流程图

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

---

## 四、上下文所有权模型

### 4.1 所有权类型

```cpp
// 上下文所有权类型
enum class ContextOwnership {
    OWNED,          // 完全拥有，独立生命周期
    BORROWED,       // 借用，临时使用后归还
    SHARED,         // 共享，多个Agent共同持有
    DELEGATED,      // 委托，所有权转移
};
```

### 4.2 委托令牌

```cpp
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

---

## 五、协作调度中的上下文

### 5.1 调度器结构

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

---

## 六、上下文追溯

### 6.1 追溯结构

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

---

## 七、设计要点总结

| 要点 | 说明 |
|------|------|
| **Context Chain** | 通过父子节点关系维护完整的调用链 |
| **trace_id** | 贯穿整个调用链，支持跨Agent追溯 |
| **Ownership模型** | OWNED/BORROWED/SHARED/DELEGATED 四种模式 |
| **委托令牌** | DelegationToken管理临时借用和归还 |
| **清理顺序** | 叶子节点先清理，父节点后清理 |
| **O(1)调度集成** | 上下文链与调度器共享，减少加载开销 |

---

## 八、与现有方案对比

| 场景 | 单Agent方案 | 多Agent方案 |
|------|-----------|-------------|
| **Attach** | 直接attach | 通过ContextChain创建节点 |
| **Detach** | 单点cleanup | 按调用链逆序清理 |
| **嵌套** | 不支持 | ContextNode父子关系 |
| **追溯** | 无 | trace_id全程贯穿 |
| **资源借用** | 无 | DelegationToken |

---

## 九、验收标准

- [ ] ContextNode 结构定义完整
- [ ] ContextChain 实现调用链追踪
- [ ] 上下文传递协议正确实现
- [ ] Ownership模型四种模式完整
- [ ] DelegationToken 借用归还机制可用
- [ ] trace_id 贯穿整个调用链
- [ ] 上下文清理按逆序执行

---

*文档版本: v1.0*
*创建时间: 2026-04-10*
