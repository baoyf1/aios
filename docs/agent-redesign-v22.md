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
