# Agent架构重新定义需求文档

**版本**: v1.0  
**章节**: 第十二章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**状态**: 需求中  
**优先级**: P1

---

## 一、核心变化

```
旧设计:
- FileAgent既是应用层又是系统层
- 所有任务都走同一个Agent

新设计:
- 应用层Agent: 处理用户任务
- 系统层Agent: 系统维护与安全 (定时任务调用)
```

---

## 二、新的Agent分类

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

---

## 三、应用层Agent (用户任务)

| Agent | 领域 | Capabilities |
|-------|------|--------------|
| DocAgent | 文档 | organize/search/send/convert/summarize |
| MailAgent | 邮件 | send/read/list/search/attachment |
| SearchAgent | 搜索 | web_search/knowledge_search/file_search |
| CodeAgent | 代码 | write/review/test/explain/refactor |
| NoteAgent | 笔记 | create/organize/link/search |
| MediaAgent | 媒体 | image_process/video_thumb/audio_transcribe |
| DataAgent | 数据 | analyze/visualize/report/transform |

### 3.1 DocAgent 详细设计

```python
class DocAgent(BaseAgent):
    """文档Agent - 处理用户文档相关任务"""
    
    def __init__(self, config: AgentConfig):
        super().__init__("DocAgent", "document", config)
        self.capabilities = [
            "organize",   # 文档整理
            "search",     # 文档搜索
            "send",       # 文档发送
            "convert",    # 格式转换
            "summarize"   # 文档摘要
        ]
    
    def _build_graph(self) -> StateGraph:
        graph = StateGraph(dict)
        graph.add_node("parse_intent", self._parse_intent)
        graph.add_node("execute_action", self._execute_action)
        graph.add_node("report", self._report)
        graph.set_entry_point("parse_intent")
        graph.add_edge("parse_intent", "execute_action")
        graph.add_edge("execute_action", "report")
        graph.set_finish_point("report")
        return graph.compile()
```

### 3.2 MailAgent 详细设计

```python
class MailAgent(BaseAgent):
    """邮件Agent - 处理用户邮件相关任务"""
    
    def __init__(self, config: AgentConfig):
        super().__init__("MailAgent", "mail", config)
        self.capabilities = [
            "send",       # 发送邮件
            "read",       # 阅读邮件
            "list",       # 列出邮件
            "search",     # 搜索邮件
            "attachment"  # 处理附件
        ]
```

---

## 四、系统层Agent (定时维护)

| Agent | 领域 | Capabilities | 调用方式 |
|-------|------|--------------|----------|
| FileAgent | 文件 | disk_monitor/security_scan/cleanup/quota_check | 定时任务 |
| SysAgent | 系统 | status_check/process_monitor/service_restart | 定时任务 |
| NetMonitorAgent | 网络 | traffic_monitor/connection_check/bandwidth_alert | 定时任务 |
| MemAgent | 记忆 | memory_gc/memory_compact/memory_backup | 每天凌晨 |
| LogAgent | 日志 | collect/analyze/alert/archive | 持续监控 |
| SecurityAgent | 安全 | vulnerability_scan/access_audit/threat_detect | 定时任务 |

### 4.1 FileAgent 系统维护

```python
class FileAgent(BaseAgent):
    """文件Agent - 系统层，负责文件安全与维护"""
    
    def __init__(self, config: AgentConfig):
        super().__init__("FileAgent", "file", config)
        self.capabilities = [
            "disk_monitor",    # 磁盘监控
            "security_scan",   # 安全扫描
            "cleanup",         # 定时清理
            "quota_check"      # 配额检查
        ]
    
    def run_periodic_maintenance(self):
        """定时任务 - 每天凌晨执行"""
        self.disk_monitor()
        self.cleanup_expired()
        self.security_scan()
    
    def disk_monitor(self):
        """监控磁盘使用情况"""
        pass
    
    def cleanup_expired(self):
        """清理过期文件"""
        pass
    
    def security_scan(self):
        """安全扫描"""
        pass
```

### 4.2 MemAgent 记忆维护

```python
class MemAgent(BaseAgent):
    """记忆Agent - 系统层，负责记忆系统维护"""
    
    def __init__(self, config: AgentConfig):
        super().__init__("MemAgent", "memory", config)
    
    def run_periodic_maintenance(self):
        """定时任务 - 每天凌晨3点执行"""
        self.memory_gc()
        self.memory_compact()
        self.memory_backup()
    
    def memory_gc(self):
        """记忆垃圾回收"""
        # 从向量数据库删除低相关度记录
        # 合并相似记录
        pass
    
    def memory_compact(self):
        """记忆压缩"""
        pass
    
    def memory_backup(self):
        """记忆备份"""
        pass
```

---

## 五、交互流程

### 5.1 用户任务流程

```
用户任务 → 主Agent → 应用层Agent (DocAgent等)
                         ↓
                   memory_search
                         ↓
                   整理文档 → 返回
```

### 5.2 系统维护流程

```
主Agent定时任务 → 系统层Agent (FileAgent等)
                              ↓
                    磁盘监控/安全扫描/清理
```

---

## 六、主Agent调度逻辑

```cpp
// 用户任务 → 应用层Agent
Response handle_user_task(task) {
    if (task.type == "document_*") return doc_agent_->execute(task);
    if (task.type == "mail_*") return mail_agent_->execute(task);
    if (task.type == "code_*") return code_agent_->execute(task);
    // ...
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

## 七、Agent注册表

```python
class AgentRegistry:
    """Agent注册表"""
    
    def __init__(self):
        self.app_agents = {}   # 应用层Agent
        self.sys_agents = {}   # 系统层Agent
    
    def register_app_agent(self, name: str, agent: BaseAgent):
        self.app_agents[name] = agent
    
    def register_sys_agent(self, name: str, agent: BaseAgent):
        self.sys_agents[name] = agent
    
    def get_app_agent(self, domain: str) -> BaseAgent:
        return self.app_agents.get(domain)
    
    def get_sys_agent(self, name: str) -> BaseAgent:
        return self.sys_agents.get(name)
    
    def list_app_agents(self) -> List[str]:
        return list(self.app_agents.keys())
    
    def list_sys_agents(self) -> List[str]:
        return list(self.sys_agents.keys())
```

---

## 八、验收标准

- [ ] 应用层Agent与应用层Agent清晰分离
- [ ] 应用层Agent处理用户任务
- [ ] 系统层Agent处理定时维护任务
- [ ] 主Agent调度逻辑正确
- [ ] Agent注册表正确管理两类Agent
- [ ] 定时任务正确调用系统层Agent
- [ ] 内存GC每天凌晨执行

---

*文档版本: v1.0*
*创建时间: 2026-04-10*
