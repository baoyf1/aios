# Layer 3 安全审计层重构需求文档

**版本**: v1.0  
**章节**: 第十章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**状态**: 需求中  
**优先级**: P1

---

## 一、核心转型

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

---

## 二、新架构图

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

---

## 三、目录结构

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

---

## 四、安全规则

### 4.1 允许的操作

| 路径 | 权限 | 说明 |
|-----|------|------|
| /root/aios/agents/{agent_id}/sandbox/... | 自由 | Sandbox内自由IO |
| /root/aios/agents/{agent_id}/work/... | 自由 | 工作区 |
| /root/aios/shared/... | 需SHARED_WRITE | 共享目录 |

### 4.2 禁止的操作

| 路径 | 说明 |
|-----|------|
| /etc/**, /usr/**, /bin/**, /lib/** | 系统目录 |
| /root/aios/agents/{other_id}/ | 其他Agent目录 |
| /root/** | 系统root，除非SYSTEM_ACCESS |

---

## 五、操作流程对比

### 5.1 旧流程

```
Agent → Layer 3 (open) → Layer 3 (write) → Layer 3 (close) → 磁盘
问题: 每次write都过Layer 3，性能差
```

### 5.2 新流程

```
Agent → Sandbox (直接写) → 磁盘
                    ↓
                commit(result)
                    ↓
            Layer 3 (安全审计)
```

### 5.3 下载流程

```
Agent → Sandbox (流式下载) → 磁盘
                    ↓
                commit(result)
                    ↓
            Layer 3 (磁盘配额检查)
```

---

## 六、安全审计接口

### 6.1 SecurityAudit 结构

```cpp
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
```

### 6.2 Commit接口

```cpp
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
    return allowed();
}
```

---

## 七、配额管理

### 7.1 配额类型

| 资源 | 配额 | 说明 |
|-----|------|------|
| disk | 10GB | 磁盘空间配额 |
| network | 1GB/hour | 网络流量配额 |
| api_calls | 1000/hour | API调用配额 |

### 7.2 quota.json格式

```json
{
    "agent_id": 1,
    "quotas": {
        "disk": {
            "limit": "10GB",
            "used": "2.3GB"
        },
        "network": {
            "limit": "1GB",
            "used": "150MB",
            "window": "1h"
        },
        "api_calls": {
            "limit": 1000,
            "used": 234,
            "window": "1h"
        }
    }
}
```

---

## 八、验收标准

- [ ] Layer 3职责转型完成
- [ ] Sandbox目录结构正确
- [ ] PathGuard路径检查正确
- [ ] CapChecker能力检查正确
- [ ] QuotaGuard配额检查正确
- [ ] AuditLog审计日志完整
- [ ] commit机制正确实现
- [ ] 新流程性能提升验证

---

*文档版本: v1.0*
*创建时间: 2026-04-10*
