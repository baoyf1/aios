# Agent配置功能文档

## 功能概述
Agent配置模块是Windows版本AIOS系统的重要组成部分，负责Agent配置的管理、模板、继承和覆盖等功能，为Agent系统提供统一的配置管理能力。

## 核心功能

### 1. Agent配置模板

#### 1.1 模板管理
```python
class AgentTemplateManager:
    def __init__(self, templates_dir):
        self.templates_dir = templates_dir
        self.templates = {}
        self.load_templates()
    
    def load_templates(self):
        """
        加载所有模板
        """
        for file in os.listdir(self.templates_dir):
            if file.endswith('.yaml') or file.endswith('.yml'):
                template_path = os.path.join(self.templates_dir, file)
                try:
                    with open(template_path, 'r', encoding='utf-8') as f:
                        template = yaml.safe_load(f)
                        template_name = template.get('name', os.path.splitext(file)[0])
                        self.templates[template_name] = template
                except Exception as e:
                    logger.error(f"加载模板 {file} 失败: {e}")
    
    def get_template(self, template_name):
        """
        获取模板
        
        参数:
            template_name: 模板名称
        
        返回:
            模板对象
        """
        return self.templates.get(template_name)
    
    def create_from_template(self, template_name, params=None):
        """
        从模板创建配置
        
        参数:
            template_name: 模板名称
            params: 模板参数
        
        返回:
            配置对象
        """
        template = self.get_template(template_name)
        if not template:
            raise ValueError(f"模板 {template_name} 不存在")
        
        # 应用参数
        config = self._apply_params(template, params or {})
        return config
    
    def _apply_params(self, template, params):
        """
        应用模板参数
        
        参数:
            template: 模板对象
            params: 模板参数
        
        返回:
            应用参数后的配置
        """
        # 实现参数应用逻辑
        pass
```

#### 1.2 模板继承
```python
def get_inherited_template(self, template_name):
    """
    获取继承的模板
    
    参数:
        template_name: 模板名称
    
    返回:
        继承链上的所有模板
    """
    template = self.get_template(template_name)
    if not template:
        return []
    
    inheritance_chain = [template]
    parent_template = template.get('extends')
    
    while parent_template:
        parent = self.get_template(parent_template)
        if not parent:
            break
        inheritance_chain.insert(0, parent)
        parent_template = parent.get('extends')
    
    return inheritance_chain
```

### 2. 配置继承

#### 2.1 配置合并
```python
def merge_configs(parent_config, child_config):
    """
    合并配置
    
    参数:
        parent_config: 父配置
        child_config: 子配置
    
    返回:
        合并后的配置
    """
    if not isinstance(parent_config, dict) or not isinstance(child_config, dict):
        return child_config
    
    merged = parent_config.copy()
    
    for key, value in child_config.items():
        if key in merged and isinstance(merged[key], dict) and isinstance(value, dict):
            merged[key] = merge_configs(merged[key], value)
        else:
            merged[key] = value
    
    return merged

def resolve_inheritance(config):
    """
    解析配置继承
    
    参数:
        config: 配置对象
    
    返回:
        解析继承后的配置
    """
    if not isinstance(config, dict):
        return config
    
    # 处理继承
    if 'extends' in config:
        parent_template = config['extends']
        parent_config = template_manager.get_template(parent_template)
        if parent_config:
            # 移除extends字段
            child_config = config.copy()
            del child_config['extends']
            # 合并配置
            config = merge_configs(parent_config, child_config)
    
    # 递归处理嵌套配置
    for key, value in config.items():
        if isinstance(value, dict):
            config[key] = resolve_inheritance(value)
    
    return config
```

### 3. 配置覆盖

#### 3.1 运行时覆盖
```python
class AgentConfig:
    def __init__(self, config):
        self.base_config = config
        self.overrides = {}
        self.effective_config = config.copy()
    
    def override(self, key, value, persistent=False):
        """
        覆盖配置
        
        参数:
            key: 配置键，支持点号分隔的嵌套键
            value: 配置值
            persistent: 是否持久化
        """
        # 应用覆盖
        self.overrides[key] = {'value': value, 'persistent': persistent}
        self._update_effective_config()
    
    def remove_override(self, key):
        """
        移除覆盖
        
        参数:
            key: 配置键
        """
        if key in self.overrides:
            del self.overrides[key]
            self._update_effective_config()
    
    def _update_effective_config(self):
        """
        更新有效配置
        """
        self.effective_config = self.base_config.copy()
        
        for key, override in self.overrides.items():
            # 应用覆盖
            keys = key.split('.')
            config = self.effective_config
            
            for k in keys[:-1]:
                if k not in config:
                    config[k] = {}
                config = config[k]
            
            config[keys[-1]] = override['value']
    
    def get(self, key, default=None):
        """
        获取配置值
        
        参数:
            key: 配置键
            default: 默认值
        
        返回:
            配置值或默认值
        """
        keys = key.split('.')
        value = self.effective_config
        
        for k in keys:
            if isinstance(value, dict) and k in value:
                value = value[k]
            else:
                return default
        
        return value
```

#### 3.2 覆盖管理
```python
def save_overrides(self, path):
    """
    保存持久化覆盖
    
    参数:
        path: 保存路径
    """
    persistent_overrides = {k: v['value'] for k, v in self.overrides.items() if v['persistent']}
    
    try:
        with open(path, 'w', encoding='utf-8') as f:
            yaml.dump(persistent_overrides, f, default_flow_style=False, allow_unicode=True)
    except Exception as e:
        logger.error(f"保存覆盖配置失败: {e}")

def load_overrides(self, path):
    """
    加载持久化覆盖
    
    参数:
        path: 加载路径
    """
    try:
        with open(path, 'r', encoding='utf-8') as f:
            overrides = yaml.safe_load(f)
            if overrides:
                for key, value in overrides.items():
                    self.override(key, value, persistent=True)
    except Exception as e:
        logger.error(f"加载覆盖配置失败: {e}")
```

## 技术实现

### 1. 架构设计
- **分层设计**：模板层、配置层、覆盖层
- **模块化**：配置管理作为独立模块
- **事件驱动**：基于事件的配置变更通知

### 2. 核心组件
- **AgentTemplateManager**：模板管理
- **AgentConfig**：配置管理
- **ConfigMerger**：配置合并
- **ConfigOverrideManager**：配置覆盖管理

### 3. 数据流
1. **模板加载**：加载Agent配置模板
2. **配置创建**：从模板创建Agent配置
3. **继承解析**：解析配置继承关系
4. **配置合并**：合并父配置和子配置
5. **覆盖应用**：应用运行时配置覆盖
6. **配置使用**：Agent使用有效配置

## 配置选项

### 1. 模板配置
```yaml
# Agent模板配置
name: "file_agent_template"
description: "文件操作Agent模板"
type: "file_agent"

# 基础配置
config:
  name: "file_agent"
  description: "文件操作Agent"
  capabilities:
    - "read_file"
    - "write_file"
    - "list_files"
  settings:
    timeout: 30
    max_file_size: 1048576

# 继承配置
extends: "base_agent_template"
```

### 2. Agent配置
```yaml
# Agent配置
name: "my_file_agent"
description: "我的文件操作Agent"
type: "file_agent"

# 继承配置
extends: "file_agent_template"

# 覆盖配置
config:
  settings:
    timeout: 60
    max_file_size: 2097152
```

### 3. 覆盖配置
```yaml
# 持久化覆盖配置
config.settings.timeout: 120
config.settings.max_file_size: 4194304
```

## 错误处理

### 1. 异常处理
```python
try:
    agent_config = AgentConfig(config)
except Exception as e:
    logger.error(f"初始化Agent配置失败: {e}")
    # 使用默认配置
    agent_config = AgentConfig(default_config)
```

### 2. 错误恢复
```python
def load_config_with_fallback(config_path, fallback_path):
    """
    加载配置，失败时使用 fallback
    
    参数:
        config_path: 配置路径
        fallback_path:  fallback路径
    
    返回:
        配置对象
    """
    try:
        with open(config_path, 'r', encoding='utf-8') as f:
            return yaml.safe_load(f)
    except Exception as e:
        logger.warning(f"加载配置失败，使用 fallback: {e}")
        try:
            with open(fallback_path, 'r', encoding='utf-8') as f:
                return yaml.safe_load(f)
        except Exception as e:
            logger.error(f"加载 fallback 配置失败: {e}")
            return {}
```

## 性能优化

### 1. 配置缓存
- 缓存模板和配置对象
- 减少I/O操作
- 延迟加载非关键配置

### 2. 并发处理
- 使用线程安全的配置访问
- 异步加载配置
- 批量处理配置变更

### 3. 资源使用
- 优化内存使用
- 减少磁盘I/O
- 合理使用缓存策略

## 安全考虑

### 1. 敏感配置
- 加密存储敏感配置
- 避免在日志中输出敏感信息
- 提供配置加密和解密接口

### 2. 配置访问控制
- 基于角色的配置访问控制
- 配置变更审计
- 防止配置注入攻击

### 3. 安全验证
- 验证配置来源的合法性
- 检查配置值的安全性
- 防止恶意配置导致的安全问题

## 集成与扩展

### 1. 与其他模块的集成
- 与Agent系统集成，提供Agent配置管理
- 与配置管理系统集成，共享配置基础设施
- 与API服务器集成，提供配置管理API

### 2. 扩展点
- 自定义模板类型
- 自定义配置验证规则
- 配置变更通知机制
- 配置版本管理