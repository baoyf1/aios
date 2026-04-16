# AIOS Agent 记忆系统设计
## 每个领域Agent的独立向量数据库记忆系统

---

## 一、设计背景

现有记忆系统 v3.0 存在问题：
- **单实例** — 所有记忆混在一起，无法按Agent隔离
- **类型简单** — 只有普通记忆，缺少错误记录、工作经验等
- **无优先级** — 所有记忆等权处理
- **加载粗糙** — 直接注入全部记忆，效率低

目标：**每个领域Agent拥有独立、全面、自管理的记忆系统**

---

## 二、总体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    中央Agent (Agent-0)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              🧠 Global Memory Hub                       │   │
│  │  • 跨Agent共享记忆 (跨域知识/通用常识)                     │   │
│  │  • Agent注册表 (谁上线/谁下线)                           │   │
│  │  • 全局经验池 (最佳实践/通用模式)                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  📁 FileAgent  │ │  💻 SysAgent  │ │  🌐 NetAgent  │
│   Memory       │ │   Memory      │ │   Memory      │
│               │ │               │ │               │
│ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
│ │CoreMemory │ │ │ │CoreMemory │ │ │ │CoreMemory │ │
│ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
│ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
│ │ErrorRecord│ │ │ │ErrorRecord│ │ │ │ErrorRecord│ │
│ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
│ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
│ │WorkExp    │ │ │ │WorkExp    │ │ │ │WorkExp    │ │
│ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
│ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
│ │SkillLib   │ │ │ │SkillLib   │ │ │ │SkillLib   │ │
│ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
│ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
│ │CacheMemory│ │ │ │CacheMemory│ │ │ │CacheMemory│ │
│ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
└───────────────┘ └───────────────┘ └───────────────┘
```

---

## 三、记忆类型体系

### 3.1 记忆分类

| 类型 | 名称 | TTL | 说明 |
|------|------|-----|------|
| **CoreMemory** | 核心记忆 | 90天 | Agent的主要知识库 |
| **ErrorRecord** | 错误记录 | 30天 | 失败经验，避免重复踩坑 |
| **WorkExperience** | 工作经验 | 180天 | 成功模式，可复用的经验 |
| **SkillLibrary** | 技能库 | 365天 | 学会的技能/工具使用 |
| **CacheMemory** | 缓存记忆 | 1天 | 短期上下文，高频访问 |
| **InteractionLog** | 交互日志 | 7天 | 完整对话记录 |

### 3.2 记忆优先级

```
优先级从高到低：
┌────────────────────────────────────────────────────────┐
│ P0: ErrorRecord    │ 关键！必须加载，影响决策安全    │
├────────────────────────────────────────────────────────┤
│ P1: WorkExperience│ 高优先级！成功经验，指导行动      │
├────────────────────────────────────────────────────────┤
│ P2: SkillLibrary  │ 中优先级！技能储备                │
├────────────────────────────────────────────────────────┤
│ P3: CoreMemory    │ 中优先级！基础知识                │
├────────────────────────────────────────────────────────┤
│ P4: InteractionLog│ 低优先级！仅参考                  │
├────────────────────────────────────────────────────────┤
│ P5: CacheMemory   │ 最低！随时间过期                 │
└────────────────────────────────────────────────────────┘
```

---

## 四、每个Agent的记忆子系统

### 4.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Memory Subsystem                      │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  Memory      │     │   Memory     │     │   Memory     │   │
│  │  Loader      │────▶│  Selector    │────▶│  Injector    │   │
│  │  (加载器)     │     │  (选择器)     │     │  (注入器)     │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│          │                   │                    │              │
│          ▼                   ▼                    ▼              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Vector Database                        │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌─────┐ │   │
│  │  │ Core   │ │ Error  │ │ Work   │ │ Skill  │ │Cache│ │   │
│  │  │Memory  │ │Record  │ │ Exp    │ │ Lib    │ │     │ │   │
│  │  │coll    │ │ coll   │ │ coll   │ │ coll   │ │ coll│ │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └─────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Memory Consolidator (整理器)                │   │
│  │  • 凌晨自动整理 (类似 NightConsolidator)                  │   │
│  │  • 过期清理                                              │   │
│  │  • 经验提炼                                              │   │
│  │  • 去重合并                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 核心组件

#### 4.2.1 MemoryLoader (记忆加载器)

```cpp
class MemoryLoader {
    // 根据上下文加载相关记忆
    // 支持多种加载策略
    
    // 策略1: 全量加载 (小数据集)
    std::vector<Memory> loadAll(AgentID agent_id);
    
    // 策略2: 按相关性加载 (语义搜索)
    std::vector<Memory> loadByRelevance(
        AgentID agent_id,
        const std::string& context,
        int max_count = 20
    );
    
    // 策略3: 按类型加载 (只加载某类记忆)
    std::vector<Memory> loadByType(
        AgentID agent_id,
        MemoryType type,
        int max_count = 50
    );
    
    // 策略4: 优先级加载 (先高优后低优)
    std::vector<Memory> loadByPriority(
        AgentID agent_id,
        int max_tokens = 4000
    );
    
    // 策略5: 混合加载 (综合多种策略)
    std::vector<Memory> loadHybrid(
        AgentID agent_id,
        const QueryContext& ctx
    );
};
```

#### 4.2.2 MemorySelector (记忆选择器)

```cpp
class MemorySelector {
    // 决定哪些记忆应该被激活
    
    // 遗忘曲线模型 (类似人类记忆)
    float calculateUrgency(const Memory& m, int current_hour);
    
    // 访问频率加权
    float calculateFrequency(const Memory& m);
    
    // 重要性衰减
    float calculateImportanceDecay(const Memory& m);
    
    // 综合评分
    float score(const Memory& m, const QueryContext& ctx) {
        return 
            calculateUrgency(m) * 0.3 +
            calculateFrequency(m) * 0.2 +
            calculateImportanceDecay(m) * 0.3 +
            calculateRelevance(m, ctx) * 0.2;
    }
    
    // 选择Top-K
    std::vector<Memory> selectTopK(
        const std::vector<Memory>& candidates,
        int k,
        int max_tokens
    );
};
```

#### 4.2.3 MemoryInjector (记忆注入器)

```cpp
class MemoryInjector {
    // 将记忆格式化成prompt片段
    
    // 格式化风格选项
    enum class FormatStyle {
        NARRATIVE,      // 叙述式: "我记得上次..."
        FACTUAL,        // 事实式: "事实: ..."
        INSTRUCTIONAL,  // 指令式: "应该..."
        MIXED           // 混合
    };
    
    // 单条记忆格式化
    std::string format(
        const Memory& m,
        FormatStyle style = FormatStyle::MIXED
    );
    
    // 批量格式化
    std::string formatBatch(
        const std::vector<Memory>& memories,
        FormatStyle style = FormatStyle::MIXED
    );
    
    // 注入到prompt
    Prompt inject(
        const Prompt& original,
        const std::vector<Memory>& memories,
        InjectPosition pos = InjectPosition::PREPEND
    );
};

enum class InjectPosition {
    PREPEND,   // 追加到开头
    APPEND,    // 追加到结尾
    INTERLEAVE // 穿插到各段落
};
```

### 4.3 记忆存储层

#### 4.3.1 分层存储

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Storage Layer                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Vector Database (LanceDB)               │    │
│  │                                                       │    │
│  │   Collection: agent_{agent_id}_core                 │    │
│  │   Collection: agent_{agent_id}_error                │    │
│  │   Collection: agent_{agent_id}_workexp              │    │
│  │   Collection: agent_{agent_id}_skill                │    │
│  │   Collection: agent_{agent_id}_cache                │    │
│  │                                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Local File Cache (LevelDB)            │    │
│  │   热数据缓存 / 写缓冲 / WAL                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│                              ▼                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Global Backup (S3/本地)               │    │
│  │   每日增量备份 / 每周全量                           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 4.3.2 Schema 设计

```cpp
// Core Memory Schema
struct CoreMemory {
    std::string id;           // UUID
    std::string agent_id;     // 属于哪个Agent
    std::string content;      // 记忆内容
    std::vector<float> embedding;  // 向量
    MemoryMetadata metadata;   // 元数据
    int priority;             // 优先级 0-4
    uint64_t created_at;
    uint64_t updated_at;
    uint64_t accessed_at;
    int access_count;
    uint64_t expires_at;
};

// Error Record Schema
struct ErrorRecord {
    std::string id;
    std::string agent_id;
    std::string error_type;       // 错误类型
    std::string error_message;    // 错误信息
    std::string context;          // 发生时的上下文
    std::string root_cause;      // 根本原因
    std::string solution;        // 解决方案
    std::vector<std::string> symptoms;  // 症状
    bool resolved;
    int occurrence_count;
    uint64_t first_seen_at;
    uint64_t last_seen_at;
};

// Work Experience Schema
struct WorkExperience {
    std::string id;
    std::string agent_id;
    std::string task_type;        // 任务类型
    std::string situation;        // 情况描述
    std::string action_taken;     // 采取的行动
    std::string outcome;          // 结果
    float success_rate;          // 成功率
    int times_used;
    std::vector<std::string> applicable_scenarios;  // 适用场景
    bool is_best_practice;
    uint64_t created_at;
    uint64_t last_used_at;
};

// Skill Library Schema
struct SkillEntry {
    std::string id;
    std::string agent_id;
    std::string skill_name;       // 技能名称
    std::string description;      // 技能描述
    std::string how_to_use;      // 使用方法
    std::vector<std::string> examples;  // 使用示例
    int mastery_level;           // 掌握程度 1-5
    std::vector<std::string> related_tools;
    bool is_enabled;
    uint64_t learned_at;
    uint64_t last_used_at;
};
```

---

## 五、Global Memory Hub (中央记忆中枢)

### 5.1 职责

```
┌─────────────────────────────────────────────────────────────┐
│                  Global Memory Hub                          │
│                                                              │
│  1. Agent注册表                                             │
│     • 记录所有Agent状态                                      │
│     • 心跳检测                                              │
│     • 故障告警                                              │
│                                                              │
│  2. 跨Agent共享记忆                                          │
│     • 通用常识 (所有Agent都需要)                            │
│     • 跨域知识 (需要多个Agent协作)                          │
│     • 共享经验 (最佳实践)                                    │
│                                                              │
│  3. 全局经验池                                              │
│     • 跨Agent的成功模式                                     │
│     • 跨Agent的失败教训                                      │
│     • 协调过的任务记录                                       │
│                                                              │
│  4. 记忆联邦                                                │
│     • 各Agent记忆的索引                                     │
│     • 跨Agent搜索                                           │
│     • 记忆同步协调                                           │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 API 设计

```cpp
class GlobalMemoryHub {
    // ==================== Agent管理 ====================
    
    // Agent注册
    bool registerAgent(AgentID id, const AgentInfo& info);
    
    // Agent注销
    bool unregisterAgent(AgentID id);
    
    // 心跳检测
    bool heartbeat(AgentID id);
    
    // 获取在线Agent
    std::vector<AgentInfo> getOnlineAgents();
    
    // ==================== 跨Agent记忆 ====================
    
    // 存储跨域记忆
    bool storeSharedMemory(const SharedMemory& mem);
    
    // 搜索跨域记忆 (可指定Agent)
    std::vector<SharedMemory> searchSharedMemory(
        const std::string& query,
        const std::vector<AgentID>& agent_ids = {}  // 空=搜索全部
    );
    
    // 获取全局最佳实践
    std::vector<BestPractice> getBestPractices(
        const std::string& domain
    );
    
    // ==================== 联邦查询 ====================
    
    // 跨多个Agent的记忆查询
    FederatedSearchResult federatedSearch(
        const std::string& query,
        const std::vector<AgentID>& agent_ids
    );
};
```

---

## 六、Memory Consolidator (记忆整理器)

### 6.1 整理任务

```
┌─────────────────────────────────────────────────────────────┐
│              Memory Consolidator (凌晨自动运行)              │
│                                                              │
│  1. 过期清理                                                │
│     • 删除 TTL 过期的记忆                                    │
│     • 归档重要记忆                                           │
│                                                              │
│  2. 去重合并                                                │
│     • 合并相似的记忆                                         │
│     • 删除完全重复                                           │
│                                                              │
│  3. 经验提炼                                                │
│     • 从多次成功提取最佳实践                                 │
│     • 从多次失败提取错误模式                                 │
│                                                              │
│  4. 优先级调整                                              │
│     • 根据访问频率调整                                       │
│     • 长期未访问降级                                         │
│                                                              │
│  5. 向量索引重建                                            │
│     • 优化 HNSW 索引                                        │
│     • 清理孤儿向量                                           │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 整理策略

```cpp
class MemoryConsolidator {
    // ==================== 夜间整理 ====================
    
    // 每日凌晨3点自动运行
    void nightlyConsolidate(AgentID agent_id) {
        // 1. 过期清理
        int expired = cleanupExpired(agent_id);
        
        // 2. 去重
        int duplicates = deduplicate(agent_id);
        
        // 3. 经验提炼
        int refined = refineExperiences(agent_id);
        
        // 4. 索引重建
        rebuildIndex(agent_id);
        
        // 5. 备份
        backup(agent_id);
        
        LOG("Consolidated:", expired, "expired,", 
            duplicates, "duplicates,", refined, "refined");
    }
    
    // ==================== 经验提炼 ====================
    
    // 从多个相似经验提炼最佳实践
    std::optional<BestPractice> distillBestPractice(
        const std::vector<WorkExperience>& experiences
    ) {
        // 1. 找成功率最高的
        // 2. 找使用次数最多的
        // 3. 综合评估
        // 4. 生成最佳实践描述
    }
    
    // 从多个相似错误提炼错误模式
    std::optional<ErrorPattern> distillErrorPattern(
        const std::vector<ErrorRecord>& errors
    ) {
        // 1. 按错误类型分组
        // 2. 找共同根因
        // 3. 生成错误模式描述
    }
    
    // ==================== 遗忘策略 ====================
    
    // 基于访问频率和时间的遗忘
    bool shouldForget(const Memory& m) {
        // 如果超过TTL且访问次数少 → 删除
        // 如果长期未访问且不重要 → 降级或删除
    }
};
```

---

## 七、错误处理与恢复

### 7.1 错误记忆流程

```
任务执行失败
      │
      ▼
┌─────────────────────────────────────┐
│  记录错误 (ErrorRecord)              │
│  • 错误类型                         │
│  • 完整上下文                        │
│  • 堆栈信息                         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  分析错误 (ErrorAnalyzer)            │
│  • 是否已知错误?                     │
│  • 找相似错误                        │
│  • 尝试自动归因                      │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  决策分支                            │
│                                      │
│  ├─→ 已知错误 ──→ 加载历史解决方案   │
│  │                                    │
│  ├─→ 新错误 ──→ 记录并告警           │
│  │                                    │
│  └─→ 反复错误 ──→ 提升为系统问题      │
└─────────────────────────────────────┘
```

### 7.2 错误加载策略

```cpp
class ErrorLoader {
    // 加载相关错误记录
    std::vector<ErrorRecord> loadRelevantErrors(
        AgentID agent_id,
        const std::string& context
    ) {
        // 1. 精确匹配错误类型
        // 2. 模糊匹配上下文
        // 3. 按频率加权
        // 4. 返回Top-5
    }
    
    // 获取错误解决方案
    std::optional<std::string> getSolution(
        AgentID agent_id,
        const std::string& error_type
    ) {
        // 查找相同错误类型的已解决记录
        // 返回解决方案
    }
};
```

---

## 八、工作经验管理

### 8.1 经验积累流程

```
任务成功完成
      │
      ▼
┌─────────────────────────────────────┐
│  提取经验 (ExperienceExtractor)      │
│                                      │
│  • 任务类型                         │
│  • 采取的行动                       │
│  • 结果评估                         │
│  • 关键决策点                       │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  存入工作经历 (WorkExperience)        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  下次类似任务时加载 (ExperienceLoader)│
│                                      │
│  • 找相似场景                        │
│  • 建议采取的行动                    │
│  • 提示注意事项                      │
└─────────────────────────────────────┘
```

### 8.2 经验检索

```cpp
class ExperienceLoader {
    // 按场景搜索经验
    std::vector<WorkExperience> loadByScenario(
        AgentID agent_id,
        const std::string& current_scenario
    ) {
        // 1. 语义搜索相似场景
        // 2. 过滤适用场景
        // 3. 按成功率排序
    }
    
    // 获取最佳实践
    std::optional<BestPractice> getBestPractice(
        AgentID agent_id,
        const std::string& task_type
    ) {
        // 1. 找同类任务
        // 2. 按成功率和使用次数综合评估
        // 3. 返回最佳
    }
};
```

---

## 九、与现有 v3.0 的关系

```
AIOS Agent Memory System
         │
         ├── 基于 v3.0 的核心架构
         │    ├── VectorStore (复用)
         │    ├── EmbeddingService (复用，支持 Ollama)
         │    ├── MemoryAgent (扩展)
         │    └── RefinementEngine (复用)
         │
         ├── 新增组件
         │    ├── MemoryLoader
         │    ├── MemorySelector
         │    ├── MemoryInjector
         │    ├── GlobalMemoryHub
         │    ├── MemoryConsolidator
         │    ├── ErrorLoader
         │    └── ExperienceLoader
         │
         └── 改造点
              ├── MemoryAgent → PerAgentMemorySystem
              ├── 单一数据库 → 分表 (按Agent+类型)
              ├── 简单TTL → 多级TTL + 遗忘策略
              └── 无 → GlobalMemoryHub 联邦
```

---

## 十、存储目录结构

```
/root/aios/memory/
├── global/                      # 全局记忆
│   ├── shared/                 # 跨Agent共享
│   ├── best_practices/         # 最佳实践
│   └── agent_registry/         # Agent注册表
│
├── agents/                     # 各Agent记忆
│   ├── file_agent/
│   │   ├── core.lance         # 核心记忆
│   │   ├── error.lance        # 错误记录
│   │   ├── workexp.lance       # 工作经验
│   │   ├── skill.lance         # 技能库
│   │   ├── cache.lance         # 缓存
│   │   └── wal/               # Write-Ahead Log
│   │
│   ├── sys_agent/
│   │   └── ...
│   │
│   └── net_agent/
│       └── ...
│
├── backups/                    # 备份
│   ├── daily/                  # 每日增量
│   └── weekly/                 # 每周全量
│
└── index/                      # 全局索引
    └── federated_search.idx    # 联邦搜索索引
```

---

## 十一、实施计划

### Phase 1: 基础框架
- [ ] PerAgentMemorySystem 骨架
- [ ] 分表存储 (按Agent+类型)
- [ ] 基础 CRUD

### Phase 2: 加载机制
- [ ] MemoryLoader 实现
- [ ] MemorySelector 实现
- [ ] MemoryInjector 实现

### Phase 3: 高级功能
- [ ] ErrorRecord 管理
- [ ] WorkExperience 积累
- [ ] SkillLibrary

### Phase 4: Global Hub
- [ ] GlobalMemoryHub 实现
- [ ] 联邦搜索
- [ ] 跨Agent记忆共享

### Phase 5: 自动整理
- [ ] MemoryConsolidator
- [ ] 凌晨自动整理
- [ ] 经验提炼

---

## 十二、技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| 向量数据库 | LanceDB | 轻量、嵌入简单、支持分表 |
| 嵌入服务 | Ollama (nomic-embed-text) | 本地部署、低延迟 |
| 本地缓存 | LevelDB | 高性能 KV 存储 |
| 备份 | S3/本地 | 灵活 |

---

*文档版本: v1.0*  
*创建时间: 2026-04-07*  
*备注: 基于现有 v3.0 架构扩展*
