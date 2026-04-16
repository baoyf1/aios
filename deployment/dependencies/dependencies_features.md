# 依赖管理功能文档

## 功能概述
依赖管理模块是Windows版本AIOS系统的重要组成部分，负责管理Python、C++和LLM模型等依赖项，确保系统能够正确安装和使用所需的依赖包和模型。

## 核心功能

### 1. Python依赖

#### 1.1 依赖配置
```python
# requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
yaml==6.0.1
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
aiohttp==3.9.1
watchdog==3.0.0
pydantic==2.5.0
```

#### 1.2 依赖安装
```python
def install_python_dependencies(requirements_file, virtual_env=None):
    """
    安装Python依赖
    
    参数:
        requirements_file: 依赖配置文件路径
        virtual_env: 虚拟环境路径
    
    返回:
        bool: 安装是否成功
    """
    import subprocess
    import os
    
    cmd = ["pip", "install", "-r", requirements_file]
    
    if virtual_env:
        pip_path = os.path.join(virtual_env, "Scripts", "pip.exe")
        if os.path.exists(pip_path):
            cmd[0] = pip_path
    
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, check=True)
        print(result.stdout)
        return True
    except subprocess.CalledProcessError as e:
        print(f"安装依赖失败: {e.stderr}")
        return False
```

#### 1.3 依赖锁定
```python
def lock_python_dependencies(requirements_file, lock_file):
    """
    锁定Python依赖版本
    
    参数:
        requirements_file: 依赖配置文件路径
        lock_file: 锁定文件路径
    
    返回:
        bool: 锁定是否成功
    """
    import subprocess
    
    cmd = ["pip", "freeze", "--local"]
    
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, check=True)
        with open(lock_file, "w") as f:
            f.write(result.stdout)
        return True
    except subprocess.CalledProcessError as e:
        print(f"锁定依赖失败: {e.stderr}")
        return False
```

### 2. C++依赖

#### 2.1 CMake配置
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(AIOS)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 依赖管理
find_package(Boost REQUIRED COMPONENTS system filesystem)
find_package(OpenSSL REQUIRED)
find_package(Threads REQUIRED)

# 包含目录
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${OPENSSL_INCLUDE_DIR})

# 链接库
target_link_libraries(AIOS PRIVATE
    ${Boost_LIBRARIES}
    ${OPENSSL_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT}
)
```

#### 2.2 依赖安装
```python
def install_cpp_dependencies(cmake_file):
    """
    安装C++依赖
    
    参数:
        cmake_file: CMake配置文件路径
    
    返回:
        bool: 安装是否成功
    """
    import subprocess
    import os
    
    build_dir = os.path.join(os.path.dirname(cmake_file), "build")
    if not os.path.exists(build_dir):
        os.makedirs(build_dir)
    
    # 运行CMake
    cmake_cmd = ["cmake", ".."]
    try:
        subprocess.run(cmake_cmd, cwd=build_dir, check=True)
    except subprocess.CalledProcessError as e:
        print(f"CMake配置失败: {e}")
        return False
    
    # 构建项目
    build_cmd = ["cmake", "--build", "."]
    try:
        subprocess.run(build_cmd, cwd=build_dir, check=True)
        return True
    except subprocess.CalledProcessError as e:
        print(f"构建失败: {e}")
        return False
```

### 3. LLM依赖

#### 3.1 模型管理
```python
class LLMModelManager:
    def __init__(self, models_dir):
        self.models_dir = models_dir
        if not os.path.exists(models_dir):
            os.makedirs(models_dir)
    
    def download_model(self, model_name, version="latest"):
        """
        下载LLM模型
        
        参数:
            model_name: 模型名称
            version: 模型版本
        
        返回:
            bool: 下载是否成功
        """
        # 实现模型下载逻辑
        pass
    
    def list_models(self):
        """
        列出可用的模型
        
        返回:
            List[str]: 模型名称列表
        """
        # 实现列出模型逻辑
        pass
    
    def get_model_path(self, model_name):
        """
        获取模型路径
        
        参数:
            model_name: 模型名称
        
        返回:
            str: 模型路径
        """
        # 实现获取模型路径逻辑
        pass
```

#### 3.2 模型验证
```python
def validate_model(model_path):
    """
    验证模型完整性
    
    参数:
        model_path: 模型路径
    
    返回:
        bool: 模型是否有效
    """
    # 实现模型验证逻辑
    pass
```

### 4. 版本管理

#### 4.1 版本锁定
```python
def create_version_lock(requirements_file, lock_file):
    """
    创建版本锁定文件
    
    参数:
        requirements_file: 依赖配置文件
        lock_file: 锁定文件路径
    
    返回:
        bool: 创建是否成功
    """
    # 实现版本锁定逻辑
    pass
```

#### 4.2 版本冲突检测
```python
def detect_version_conflicts(requirements_file):
    """
    检测版本冲突
    
    参数:
        requirements_file: 依赖配置文件
    
    返回:
        List[str]: 冲突列表
    """
    # 实现版本冲突检测逻辑
    pass
```

## 技术实现

### 1. 架构设计
- **分层设计**：配置层、安装层、管理层
- **模块化**：依赖类型作为独立模块
- **配置驱动**：基于配置文件管理依赖

### 2. 核心组件
- **PythonDependencyManager**：Python依赖管理
- **CppDependencyManager**：C++依赖管理
- **LLMDependencyManager**：LLM依赖管理
- **VersionManager**：版本管理

### 3. 数据流
1. **配置读取**：读取依赖配置文件
2. **依赖解析**：解析依赖关系和版本
3. **依赖安装**：安装所需的依赖包
4. **版本锁定**：锁定依赖版本
5. **依赖验证**：验证依赖安装结果
6. **依赖使用**：系统使用安装的依赖

## 配置选项

### 1. Python依赖配置
```txt
# requirements.txt
# Web框架
fastapi==0.104.1
uvicorn[standard]==0.24.0

# 数据处理
yaml==6.0.1
pydantic==2.5.0

# 安全
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4

# 其他
python-multipart==0.0.6
aiohttp==3.9.1
watchdog==3.0.0
```

### 2. C++依赖配置
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(AIOS)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 依赖管理
find_package(Boost REQUIRED COMPONENTS system filesystem)
find_package(OpenSSL REQUIRED)
find_package(Threads REQUIRED)

# 包含目录
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${OPENSSL_INCLUDE_DIR})

# 链接库
target_link_libraries(AIOS PRIVATE
    ${Boost_LIBRARIES}
    ${OPENSSL_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT}
)
```

### 3. LLM模型配置
```yaml
# llm_models.yaml
models:
  - name: "llama3"
    version: "1.0"
    url: "https://huggingface.co/meta-llama/Llama-3-8B/resolve/main/model.safetensors"
    size: "8GB"
  - name: "gemma"
    version: "1.0"
    url: "https://huggingface.co/google/gemma-7b/resolve/main/model.safetensors"
    size: "7GB"
```

## 错误处理

### 1. 依赖安装错误处理
```python
try:
    install_python_dependencies("requirements.txt")
except Exception as e:
    print(f"安装依赖失败: {e}")
    # 尝试使用备用源
    os.environ["PIP_INDEX_URL"] = "https://pypi.tuna.tsinghua.edu.cn/simple"
    install_python_dependencies("requirements.txt")
```

### 2. 版本冲突处理
```python
def resolve_version_conflicts(requirements_file):
    """
    解决版本冲突
    
    参数:
        requirements_file: 依赖配置文件
    
    返回:
        bool: 解决是否成功
    """
    conflicts = detect_version_conflicts(requirements_file)
    if not conflicts:
        return True
    
    # 实现冲突解决逻辑
    for conflict in conflicts:
        # 尝试解决冲突
        pass
    
    return len(detect_version_conflicts(requirements_file)) == 0
```

## 性能优化

### 1. 依赖下载优化
- 并行下载依赖
- 使用本地缓存
- 减少网络请求

### 2. 依赖解析优化
- 缓存依赖解析结果
- 优化依赖树构建
- 减少重复解析

### 3. 资源使用优化
- 优化内存使用
- 减少磁盘I/O
- 合理使用系统资源

## 安全考虑

### 1. 依赖验证
```python
def verify_dependency_integrity(dependency_path, checksum):
    """
    验证依赖完整性
    
    参数:
        dependency_path: 依赖路径
        checksum: 预期校验和
    
    返回:
        bool: 验证是否成功
    """
    import hashlib
    
    with open(dependency_path, "rb") as f:
        content = f.read()
        actual_checksum = hashlib.sha256(content).hexdigest()
    
    return actual_checksum == checksum
```

### 2. 安全漏洞检查
```python
def check_security_vulnerabilities(requirements_file):
    """
    检查依赖安全漏洞
    
    参数:
        requirements_file: 依赖配置文件
    
    返回:
        List[str]: 漏洞列表
    """
    # 实现安全漏洞检查逻辑
    pass
```

### 3. 依赖来源验证
- 验证依赖包来源
- 使用可信的依赖源
- 防止恶意依赖注入

## 集成与扩展

### 1. 与其他模块的集成
- 与安装包制作模块集成，确保所有依赖项被包含
- 与自动更新模块集成，支持依赖更新
- 与配置管理模块集成，管理依赖配置

### 2. 扩展点
- 自定义依赖源
- 自定义依赖验证规则
- 依赖分析工具
- 依赖监控和告警