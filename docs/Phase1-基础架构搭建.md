# AIOS Phase 1: 基础架构搭建

**时间：** Month 1-2（第1-8周）  
**目标：** 完成项目骨架、核心抽象层、OS适配层、IPC机制

---

## 1.1 项目初始化

### 目标
- 建立 C++/Python 混合项目结构
- 配置 CMake + venv 环境
- Git 仓库初始化

### 目录结构
```
/root/aios/
├── src/
│   ├── core/           # C++ 核心库
│   │   ├── agent/      # Agent 抽象
│   │   ├── scheduler/  # 调度器
│   │   ├── memory/     # 内存管理
│   │   └── ipc/       # 进程间通信
│   ├── bindings/       # Python 绑定
│   │   └── pycore/    # pybind11 绑定
│   └── os/             # OS 适配层
│       ├── windows/    # Win32/WMI
│       └── linux/      # D-Bus/proc
├── tests/
│   ├── unit/          # 单元测试
│   └── integration/   # 集成测试
├── scripts/           # 构建脚本
└── examples/          # 示例
```

### 依赖清单
| 组件 | 工具 | 版本 |
|------|------|------|
| C++ 编译器 | GCC/Clang | 11+ |
| CMake | cmake | 3.20+ |
| Python | Python | 3.10+ |
| 绑定工具 | pybind11 | 2.11+ |
| 单元测试 | GoogleTest | 1.12+ |

### 验收标准
- [ ] `make` 能编译整个项目
- [ ] `python -c "import aios_core"` 能导入模块
- [ ] 单元测试覆盖率 > 70%

---

## 1.2 Agent 抽象层设计

### 目标
- 定义 Agent 接口
- 实现状态机
- 设计消息传递机制

### 核心接口
```cpp
// agent.h
class Agent {
public:
    enum State { IDLE, PLANNING, EXECUTING, WAITING, COMPLETED };
    
    virtual ~Agent() = default;
    virtual void receive(const Message& msg) = 0;
    virtual void send(const Message& msg) = 0;
    virtual State state() const = 0;
    virtual std::string plan() = 0;
};
```

### 状态机设计
```
       ┌──────────────────────────────────────┐
       │                                      │
       ▼                                      │
    IDLE ──────► PLANNING ──────► EXECUTING ──┤
       ▲              │               │       │
       │              │               ▼       │
       │              │           WAITING ────┤
       │              │               │       │
       │              ▼               ▼       │
       └──────── COMPLETED ◄────────────┘       │
```

### 验收标准
- [ ] Agent 能收发消息
- [ ] 状态转换正确
- [ ] 单元测试通过

---

## 1.3 消息队列 / IPC 机制选型

### 候选方案
| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| POSIX 队列 | 跨平台 | 速度慢 | ⭐⭐ |
| 共享内存 + 信号量 | 速度快 | 编程复杂 | ⭐⭐⭐ |
| gRPC | 跨语言、成熟 | 延迟略高 | ⭐⭐⭐⭐ |
| ZeroMQ | 轻量、高性能 | 文档少 | ⭐⭐⭐⭐ |
| 自定义 RingBuffer | 最灵活 | 需手写 | ⭐⭐ |

### 推荐架构
```
┌─────────────────────────────────────────┐
│            Agent A (Python)             │
│         LangChain Agent                 │
└────────────────┬────────────────────────┘
                 │ IPC (gRPC / ZeroMQ)
┌────────────────┴────────────────────────┐
│            C++ Core                     │
│  ┌─────────┐  ┌─────────┐  ┌────────┐  │
│  │Scheduler│  │ Memory  │  │ Tools  │  │
│  └─────────┘  └─────────┘  └────────┘  │
└────────────────┬────────────────────────┘
                 │ IPC
┌────────────────┴────────────────────────┐
│            OS Layer                     │
│  Windows API / Linux D-Bus              │
└─────────────────────────────────────────┘
```

### 验收标准
- [ ] 支持多进程通信
- [ ] 延迟 < 10ms（本地）
- [ ] 支持心跳/超时检测

---

## 1.4 配置管理系统

### 设计
```yaml
# config.yaml
aios:
  version: "1.0.0"
  
llm:
  provider: "ollama"  # ollama | openai | zhipu | qwen
  model: "qwen2.5-7b"
  base_url: "http://localhost:11434"
  
agent:
  max_concurrent: 4
  timeout_seconds: 300
  
security:
  enable_sandbox: true
  allow_file_write: false
  
logging:
  level: "INFO"
  file: "/var/log/aios.log"
```

### 验收标准
- [ ] 支持 YAML 配置
- [ ] 支持环境变量覆盖
- [ ] 配置热加载

---

## 1.5 Windows OS 适配层

### Win32 核心功能
```cpp
// windows/process.h
class WindowsProcessManager {
public:
    // 进程枚举
    std::vector<ProcessInfo> listProcesses();
    
    // 进程控制
    bool kill(DWORD pid);
    bool suspend(DWORD pid);
    bool resume(DWORD pid);
    
    // 资源监控
    ProcessStats getStats(DWORD pid);
    
private:
    HANDLE snapshot_;
};

// WMI 查询
class WMIQuery {
public:
    std::vector<WMIRecord> query(const std::wstring& sql);
};
```

### 能力清单
| 功能 | Win32 API | 优先级 |
|------|-----------|--------|
| 进程枚举 | `CreateToolhelp32Snapshot` | P0 |
| 进程杀/起 | `TerminateProcess`/`CreateProcess` | P0 |
| 内存使用 | `GlobalMemoryStatusEx` | P1 |
| CPU占用 | `GetProcessTimes` | P1 |
| 窗口枚举 | `EnumWindows` | P2 |

### 验收标准
- [ ] 能枚举所有进程
- [ ] 能启动/杀死进程
- [ ] 能读取进程资源占用

---

## 1.6 Linux OS 适配层

### 核心功能
```cpp
// linux/process.h
class LinuxProcessManager {
public:
    // /proc 读取
    std::vector<ProcessInfo> listProcesses();
    
    // 进程控制
    bool kill(int pid);
    bool sendSignal(int pid, int sig);
    
    // 资源监控
    ProcessStats getStats(int pid);
    
    // cgroup 支持
    void setCgroupLimit(int pid, const CgroupConfig& config);
};
```

### D-Bus 集成
```cpp
// linux/dbus.h
class DBusConnection {
public:
    bool connect(DBusBusType type = DBUS_BUS_SESSION);
    std::optional<DBusVariant> callMethod(
        const std::string& service,
        const std::string& path,
        const std::string& interface,
        const std::string& method
    );
};
```

### 能力清单
| 功能 | API/文件 | 优先级 |
|------|---------|--------|
| 进程枚举 | `/proc/[pid]/stat` | P0 |
| 进程杀/起 | `kill()`/`fork()` | P0 |
| 内存使用 | `/proc/meminfo` | P1 |
| 网络连接 | `/proc/net/tcp` | P1 |
| D-Bus通信 | libdbus | P2 |

### 验收标准
- [ ] 能枚举所有进程
- [ ] 能启动/杀死进程
- [ ] 能通过 D-Bus 与系统通信

---

## 1.7 最小可行Agent（MVP）

### 功能定义
```
用户: "帮我打开记事本"
Agent: 
  1. 解析意图 → 启动应用
  2. 调用 WindowsProcessManager.start("notepad.exe")
  3. 返回结果
```

### 技术实现
```python
# mvp_agent.py
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个Windows/Linux助手。
    识别用户意图，返回JSON格式：
    {
        "action": "launch_app|kill_process|list_processes|...",
        "params": {"name": "xxx", "pid": 123}
    }"""),
    ("human", "{user_input}")
])

# 通过 pybind11 调用 C++ 核心
from aios_core import ProcessManager
pm = ProcessManager()
```

### 验收标准
- [ ] 自然语言启动应用（Windows）
- [ ] 自然语言杀死进程
- [ ] 自然语言查询进程列表

---

## 1.8 第一阶段复盘

### 完成度检查
| 模块 | 状态 | 完成度 |
|------|------|--------|
| 项目骨架 | 🟡 | 80% |
| Agent抽象 | 🟡 | 60% |
| IPC机制 | 🟡 | 50% |
| 配置管理 | 🟡 | 70% |
| Windows适配 | 🔴 | 30% |
| Linux适配 | 🔴 | 30% |
| MVP验证 | 🔴 | 0% |

### 风险评估
| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| pybind11 绑定复杂 | 中 | 预留2周缓冲 |
| Windows API 兼容性 | 高 | 重点测试Win10/11 |
| gRPC 性能 | 低 | 考虑 ZeroMQ 备选 |

### 下阶段准备
- Phase 2 需要：LangGraph 深入、向量数据库选型
- 资源需求：GPU 推理测试

---

*文档版本: v1.0*  
*创建时间: 2026-04-06*  
*负责人: AIOS Team*
