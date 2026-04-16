# 文件操作工具功能文档

## 1. 基本文件操作

### 1.1 目录操作
- **list_directory**: 列出目录内容，支持递归列出
  - 参数: path (路径), recursive (是否递归), pattern (文件模式)
  - 返回: 目录内容列表

- **create_directory**: 创建目录，支持递归创建
  - 参数: path (路径), recursive (是否递归)
  - 返回: 是否成功

- **remove_directory**: 删除目录，支持递归删除
  - 参数: path (路径), recursive (是否递归)
  - 返回: 是否成功

### 1.2 文件操作
- **read_file**: 读取文件内容，支持不同编码
  - 参数: path (路径), encoding (编码)
  - 返回: 文件内容

- **write_file**: 写入文件内容，支持覆盖和追加
  - 参数: path (路径), content (内容), mode (模式: 'w'/'a')
  - 返回: 是否成功

- **copy_file**: 复制文件
  - 参数: source (源路径), destination (目标路径)
  - 返回: 是否成功

- **move_file**: 移动或重命名文件
  - 参数: source (源路径), destination (目标路径)
  - 返回: 是否成功

- **delete_file**: 删除文件
  - 参数: path (路径)
  - 返回: 是否成功

## 2. 高级文件操作

### 2.1 文件搜索
- **search_files**: 搜索文件，支持通配符和正则表达式
  - 参数: path (搜索路径), pattern (搜索模式), recursive (是否递归)
  - 返回: 匹配的文件列表

- **find_files_by_extension**: 按扩展名查找文件
  - 参数: path (搜索路径), extension (扩展名), recursive (是否递归)
  - 返回: 匹配的文件列表

- **find_files_by_size**: 按文件大小查找文件
  - 参数: path (搜索路径), min_size (最小大小), max_size (最大大小), recursive (是否递归)
  - 返回: 匹配的文件列表

### 2.2 文件信息
- **get_file_info**: 获取文件信息
  - 参数: path (路径)
  - 返回: 文件信息字典 (大小、修改时间、权限等)

- **get_file_size**: 获取文件大小
  - 参数: path (路径)
  - 返回: 文件大小 (字节)

- **get_file_modified_time**: 获取文件修改时间
  - 参数: path (路径)
  - 返回: 修改时间 (timestamp)

- **get_file_permissions**: 获取文件权限
  - 参数: path (路径)
  - 返回: 权限信息

### 2.3 文件监控
- **start_file_watcher**: 启动文件监控
  - 参数: path (监控路径), recursive (是否递归), callback (回调函数)
  - 返回: 监控器实例

- **stop_file_watcher**: 停止文件监控
  - 参数: watcher (监控器实例)
  - 返回: 是否成功

- **get_file_changes**: 获取文件变更
  - 参数: watcher (监控器实例)
  - 返回: 变更列表

### 2.4 文件比较
- **compare_files**: 比较两个文件的内容
  - 参数: file1 (文件1路径), file2 (文件2路径)
  - 返回: 差异列表

- **get_file_hash**: 计算文件哈希值
  - 参数: path (路径), algorithm (算法: 'md5'/'sha1'/'sha256')
  - 返回: 哈希值

- **check_file_integrity**: 检查文件完整性
  - 参数: path (路径), expected_hash (期望哈希值), algorithm (算法)
  - 返回: 是否完整

## 3. Windows特有功能

### 3.1 特殊文件夹
- **get_special_folder**: 获取Windows特殊文件夹路径
  - 参数: folder_type (文件夹类型: 'desktop'/'documents'/'downloads'等)
  - 返回: 文件夹路径

- **list_special_folders**: 列出所有Windows特殊文件夹
  - 参数: 无
  - 返回: 特殊文件夹字典

### 3.2 快捷方式
- **create_shortcut**: 创建快捷方式
  - 参数: target (目标路径), shortcut_path (快捷方式路径), arguments (参数), working_dir (工作目录), icon (图标)
  - 返回: 是否成功

- **read_shortcut**: 读取快捷方式信息
  - 参数: shortcut_path (快捷方式路径)
  - 返回: 快捷方式信息字典

### 3.3 文件关联
- **get_file_association**: 获取文件关联
  - 参数: extension (扩展名)
  - 返回: 关联信息

- **set_file_association**: 设置文件关联
  - 参数: extension (扩展名), program (程序路径)
  - 返回: 是否成功

### 3.4 资源管理器集成
- **open_in_explorer**: 在资源管理器中打开路径
  - 参数: path (路径)
  - 返回: 是否成功

- **select_in_explorer**: 在资源管理器中选择文件
  - 参数: path (路径)
  - 返回: 是否成功

## 4. 批量操作

### 4.1 批量文件操作
- **batch_copy**: 批量复制文件
  - 参数: files (文件列表), destination (目标目录)
  - 返回: 成功/失败文件列表

- **batch_move**: 批量移动文件
  - 参数: files (文件列表), destination (目标目录)
  - 返回: 成功/失败文件列表

- **batch_delete**: 批量删除文件
  - 参数: files (文件列表)
  - 返回: 成功/失败文件列表

### 4.2 批量重命名
- **batch_rename**: 批量重命名文件
  - 参数: files (文件列表), pattern (重命名模式), replacement (替换字符串)
  - 返回: 重命名后的文件列表

- **rename_with_sequence**: 按序列重命名文件
  - 参数: files (文件列表), prefix (前缀), start (起始序号)
  - 返回: 重命名后的文件列表

## 5. 错误处理

### 5.1 异常处理
- **handle_file_errors**: 处理文件操作异常
  - 参数: function (要执行的函数), *args, **kwargs
  - 返回: 执行结果或错误信息

- **validate_path**: 验证路径合法性
  - 参数: path (路径)
  - 返回: 是否合法

- **check_permissions**: 检查文件访问权限
  - 参数: path (路径), mode (访问模式: 'r'/'w'/'x')
  - 返回: 是否有权限

### 5.2 错误恢复
- **retry_operation**: 重试文件操作
  - 参数: function (要执行的函数), max_attempts (最大尝试次数), *args, **kwargs
  - 返回: 执行结果

- **rollback_operation**: 回滚文件操作
  - 参数: operation (操作类型), *args, **kwargs
  - 返回: 是否成功

## 6. 性能优化

### 6.1 缓存机制
- **file_cache**: 文件内容缓存
  - 缓存频繁访问的文件内容
  - 支持缓存过期和清理

- **path_cache**: 路径信息缓存
  - 缓存路径存在性和属性
  - 提高路径相关操作的性能

### 6.2 并行处理
- **parallel_file_operations**: 并行执行文件操作
  - 参数: operations (操作列表)
  - 返回: 操作结果列表

- **batch_processing**: 批量处理文件
  - 参数: files (文件列表), process_func (处理函数), batch_size (批量大小)
  - 返回: 处理结果列表

## 7. 安全性

### 7.1 路径安全
- **sanitize_path**: 清理路径，防止路径遍历攻击
  - 参数: path (路径)
  - 返回: 清理后的路径

- **validate_path_access**: 验证路径访问权限
  - 参数: path (路径), mode (访问模式)
  - 返回: 是否允许访问

### 7.2 敏感文件保护
- **is_sensitive_file**: 检测敏感文件
  - 参数: path (路径)
  - 返回: 是否敏感

- **protect_sensitive_files**: 保护敏感文件
  - 参数: paths (文件路径列表)
  - 返回: 保护结果

### 7.3 审计日志
- **log_file_operation**: 记录文件操作日志
  - 参数: operation (操作类型), path (路径), status (状态)
  - 返回: 无

- **get_operation_history**: 获取操作历史
  - 参数: path (路径), limit (限制数量)
  - 返回: 操作历史列表

## 8. 可扩展性

### 8.1 插件系统
- **register_file_operation**: 注册自定义文件操作
  - 参数: name (操作名称), function (操作函数)
  - 返回: 是否成功

- **get_file_operations**: 获取所有可用的文件操作
  - 参数: 无
  - 返回: 操作列表

### 8.2 扩展接口
- **FileOperation**: 文件操作基类
  - 提供统一的操作接口
  - 支持自定义操作实现

- **FileOperationManager**: 文件操作管理器
  - 管理所有文件操作
  - 提供操作调度和执行