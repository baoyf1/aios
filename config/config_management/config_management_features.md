# 配置管理功能文档

## 功能概述
配置管理模块是Windows版本AIOS系统的重要组成部分，负责配置的加载、解析、验证和热重载等功能，为系统提供统一的配置管理能力。

## 核心功能

### 1. YAML配置解析

#### 1.1 配置加载
```python
class ConfigManager:
    def __init__(self, config_path):
        self.config_path = config_path
        self.config = {}
        self.load_config()
    
    def load_config(self):
        """
        加载配置文件
        """
        try:
            with open(self.config_path, 'r', encoding='utf-8') as f:
                self.config = yaml.safe_load(f)
        except Exception as e:
            logger.error(f"加载配置文件失败: {e}")
            # 使用默认配置
            self.config = self.get_default_config()
    
    def get_default_config(self):
        """
        获取默认配置
        """
        return {
            'api_server': {
                'host': '0.0.0.0',
                'port': 8000
            },
            'llm': {
                'model': 'llama3',
                'temperature': 0.7
            }
        }
```

#### 1.2 配置访问
```python
def get(self, key, default=None):
    """
    获取配置值
    
    参数:
        key: 配置键，支持点号分隔的嵌套键
        default: 默认值
    
    返回:
        配置值或默认值
    """
    keys = key.split('.')
    value = self.config
    
    for k in keys:
        if isinstance(value, dict) and k in value:
            value = value[k]
        else:
            return default
    
    return value

def set(self, key, value):
    """
    设置配置值
    
    参数:
        key: 配置键，支持点号分隔的嵌套键
        value: 配置值
    """
    keys = key.split('.')
    config = self.config
    
    for k in keys[:-1]:
        if k not in config:
            config[k] = {}
        config = config[k]
    
    config[keys[-1]] = value
    self.save_config()

def save_config(self):
    """
    保存配置到文件
    """
    try:
        with open(self.config_path, 'w', encoding='utf-8') as f:
            yaml.dump(self.config, f, default_flow_style=False, allow_unicode=True)
    except Exception as e:
        logger.error(f"保存配置文件失败: {e}")
```

### 2. 环境变量支持

#### 2.1 环境变量加载
```python
def load_from_env(self, prefix="AIOS_"):
    """
    从环境变量加载配置
    
    参数:
        prefix: 环境变量前缀
    """
    import os
    
    for key, value in os.environ.items():
        if key.startswith(prefix):
            # 移除前缀并转换为小写
            config_key = key[len(prefix):].lower()
            # 将下划线转换为点号
            config_key = config_key.replace('_', '.')
            # 转换值类型
            value = self._convert_env_value(value)
            # 设置配置
            self.set(config_key, value)
    
    logger.info("从环境变量加载配置完成")

@staticmethod
def _convert_env_value(value):
    """
    转换环境变量值类型
    
    参数:
        value: 环境变量值
    
    返回:
        转换后的值
    """
    # 尝试转换为布尔值
    if value.lower() == 'true':
        return True
    if value.lower() == 'false':
        return False
    
    # 尝试转换为整数
    try:
        return int(value)
    except ValueError:
        pass
    
    # 尝试转换为浮点数
    try:
        return float(value)
    except ValueError:
        pass
    
    # 尝试转换为列表
    if value.startswith('[') and value.endswith(']'):
        try:
            return eval(value)
        except:
            pass
    
    # 默认为字符串
    return value
```

### 3. 配置验证

#### 3.1 配置模式定义
```python
class ConfigSchema:
    """
    配置模式
    """
    def __init__(self):
        self.schema = {
            'api_server': {
                'type': 'dict',
                'required': True,
                'schema': {
                    'host': {'type': 'string', 'required': True},
                    'port': {'type': 'integer', 'required': True, 'min': 1, 'max': 65535}
                }
            },
            'llm': {
                'type': 'dict',
                'required': True,
                'schema': {
                    'model': {'type': 'string', 'required': True},
                    'temperature': {'type': 'number', 'required': True, 'min': 0, 'max': 1}
                }
            }
        }
    
    def validate(self, config):
        """
        验证配置
        
        参数:
            config: 配置对象
        
        返回:
            (bool, list): (是否有效, 错误信息列表)
        """
        errors = []
        self._validate(config, self.schema, '', errors)
        return len(errors) == 0, errors
    
    def _validate(self, config, schema, path, errors):
        """
        递归验证配置
        """
        # 实现验证逻辑
        pass
```

#### 3.2 配置验证
```python
def validate(self, schema=None):
    """
    验证配置
    
    参数:
        schema: 配置模式
    
    返回:
        (bool, list): (是否有效, 错误信息列表)
    """
    if schema is None:
        schema = ConfigSchema()
    
    return schema.validate(self.config)

def validate_and_fix(self, schema=None):
    """
    验证并修复配置
    
    参数:
        schema: 配置模式
    
    返回:
        bool: 是否成功修复
    """
    is_valid, errors = self.validate(schema)
    if is_valid:
        return True
    
    # 尝试修复配置
    for error in errors:
        # 实现修复逻辑
        pass
    
    return self.validate(schema)[0]
```

### 4. 配置热重载

#### 4.1 配置监控
```python
class ConfigWatcher:
    def __init__(self, config_manager, interval=1):
        self.config_manager = config_manager
        self.interval = interval
        self.running = False
        self.thread = None
    
    def start(self):
        """
        开始监控配置文件
        """
        self.running = True
        self.thread = threading.Thread(target=self._watch)
        self.thread.daemon = True
        self.thread.start()
    
    def stop(self):
        """
        停止监控配置文件
        """
        self.running = False
        if self.thread:
            self.thread.join()
    
    def _watch(self):
        """
        监控配置文件变更
        """
        last_mtime = os.path.getmtime(self.config_manager.config_path)
        
        while self.running:
            try:
                current_mtime = os.path.getmtime(self.config_manager.config_path)
                if current_mtime > last_mtime:
                    logger.info("配置文件已变更，重新加载")
                    self.config_manager.load_config()
                    self.config_manager.load_from_env()
                    # 触发配置变更事件
                    self._on_config_change()
                    last_mtime = current_mtime
            except Exception as e:
                logger.error(f"监控配置文件失败: {e}")
            
            time.sleep(self.interval)
    
    def _on_config_change(self):
        """
        配置变更回调
        """
        # 可以通过事件系统通知其他模块
        pass
```

#### 4.2 热重载管理
```python
def enable_hot_reload(self, interval=1):
    """
    启用配置热重载
    
    参数:
        interval: 监控间隔（秒）
    """
    self.watcher = ConfigWatcher(self, interval)
    self.watcher.start()
    logger.info("配置热重载已启用")

def disable_hot_reload(self):
    """
    禁用配置热重载
    """
    if hasattr(self, 'watcher'):
        self.watcher.stop()
        del self.watcher
        logger.info("配置热重载已禁用")
```

## 技术实现

### 1. 架构设计
- **分层设计**：配置加载层、解析层、验证层、管理层
- **模块化**：配置源作为独立模块
- **事件驱动**：基于事件的配置变更通知

### 2. 核心组件
- **ConfigManager**：配置管理
- **ConfigSchema**：配置模式
- **ConfigWatcher**：配置监控
- **EnvLoader**：环境变量加载

### 3. 数据流
1. **配置加载**：从文件或环境变量加载配置
2. **配置解析**：解析配置文件格式
3. **配置验证**：验证配置的有效性
4. **配置访问**：提供配置访问接口
5. **配置监控**：监控配置文件变更
6. **配置重载**：在配置变更时重载配置

## 配置选项

### 1. 配置文件结构
```yaml
# AIOS配置文件

# API服务器配置
api_server:
  host: "0.0.0.0"
  port: 8000
  debug: false

# LLM配置
llm:
  model: "llama3"
  temperature: 0.7
  max_tokens: 1000

# 工具配置
tools:
  file_ops:
    enabled: true
  system_ops:
    enabled: true
  network_ops:
    enabled: true
  memory_ops:
    enabled: true
  llm_ops:
    enabled: true

# 日志配置
logging:
  level: "info"
  file: "aios.log"
```

### 2. 环境变量格式
```bash
# API服务器配置
AIOS_API_SERVER_HOST=0.0.0.0
AIOS_API_SERVER_PORT=8000

# LLM配置
AIOS_LLM_MODEL=llama3
AIOS_LLM_TEMPERATURE=0.7

# 工具配置
AIOS_TOOLS_FILE_OPS_ENABLED=true
AIOS_TOOLS_SYSTEM_OPS_ENABLED=true
```

## 错误处理

### 1. 异常处理
```python
try:
    config_manager = ConfigManager("config.yaml")
except Exception as e:
    logger.error(f"初始化配置管理器失败: {e}")
    # 使用默认配置
    config_manager = ConfigManager(None)
```

### 2. 错误恢复
```python
def load_config_with_retry(self, max_retries=3):
    """
    尝试加载配置文件，失败时重试
    
    参数:
        max_retries: 最大重试次数
    
    返回:
        bool: 是否成功加载
    """
    for i in range(max_retries):
        try:
            self.load_config()
            return True
        except Exception as e:
            logger.warning(f"加载配置文件失败 (尝试 {i+1}/{max_retries}): {e}")
            time.sleep(1)
    
    logger.error("加载配置文件失败，已达到最大重试次数")
    return False
```

## 性能优化

### 1. 配置缓存
- 缓存配置对象，减少I/O操作
- 使用内存映射文件，提高读取速度
- 延迟加载非关键配置

### 2. 并发处理
- 使用线程安全的配置访问
- 异步加载配置文件
- 批量处理配置变更

### 3. 资源使用
- 优化内存使用，避免内存泄漏
- 减少磁盘I/O，提高性能
- 合理使用缓存策略

## 安全考虑

### 1. 敏感配置
- 加密存储敏感配置（如API密钥）
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
- 与API服务器集成，提供配置管理API
- 与Agent系统集成，提供Agent配置管理
- 与工具系统集成，提供工具配置管理

### 2. 扩展点
- 自定义配置源
- 自定义配置验证规则
- 配置变更通知机制
- 配置版本管理