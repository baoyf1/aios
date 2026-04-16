# 记忆操作工具功能文档

## 1. 向量存储

### 1.1 向量数据管理
- **store_vector**: 存储向量数据
  - 参数: data (数据), embedding (向量), metadata (元数据), tags (标签)
  - 返回: 记忆ID

- **get_vector**: 获取向量数据
  - 参数: memory_id (记忆ID)
  - 返回: 向量数据 (数据, 向量, 元数据, 标签)

- **update_vector**: 更新向量数据
  - 参数: memory_id (记忆ID), data (数据), embedding (向量), metadata (元数据), tags (标签)
  - 返回: 是否成功

- **delete_vector**: 删除向量数据
  - 参数: memory_id (记忆ID)
  - 返回: 是否成功

### 1.2 向量嵌入
- **create_embedding**: 创建向量嵌入
  - 参数: data (数据), model (模型名称)
  - 返回: 向量

- **get_embedding_models**: 获取可用的嵌入模型
  - 参数: 无
  - 返回: 模型列表

- **set_default_embedding_model**: 设置默认嵌入模型
  - 参数: model_name (模型名称)
  - 返回: 是否成功

### 1.3 向量索引
- **create_index**: 创建向量索引
  - 参数: index_name (索引名称), dimension (向量维度)
  - 返回: 索引ID

- **add_to_index**: 向索引添加向量
  - 参数: index_id (索引ID), memory_id (记忆ID), embedding (向量)
  - 返回: 是否成功

- **search_index**: 搜索索引
  - 参数: index_id (索引ID), query_embedding (查询向量), top_k (返回数量)
  - 返回: 搜索结果

- **delete_index**: 删除索引
  - 参数: index_id (索引ID)
  - 返回: 是否成功

### 1.4 批量操作
- **batch_store_vectors**: 批量存储向量
  - 参数: vectors (向量列表)
  - 返回: 记忆ID列表

- **batch_get_vectors**: 批量获取向量
  - 参数: memory_ids (记忆ID列表)
  - 返回: 向量数据列表

- **batch_delete_vectors**: 批量删除向量
  - 参数: memory_ids (记忆ID列表)
  - 返回: 成功/失败列表

## 2. 记忆检索

### 2.1 向量搜索
- **search_by_vector**: 基于向量相似度搜索
  - 参数: query (查询文本), top_k (返回数量), model (嵌入模型)
  - 返回: 搜索结果 (记忆ID, 相似度, 数据, 元数据)

- **search_by_embedding**: 基于嵌入向量搜索
  - 参数: embedding (查询向量), top_k (返回数量)
  - 返回: 搜索结果

- **search_by_hybrid**: 混合搜索 (向量+关键词)
  - 参数: query (查询文本), top_k (返回数量), model (嵌入模型), keyword_weight (关键词权重)
  - 返回: 搜索结果

### 2.2 关键词搜索
- **search_by_keyword**: 基于关键词搜索
  - 参数: keyword (关键词), top_k (返回数量)
  - 返回: 搜索结果

- **search_by_tags**: 基于标签搜索
  - 参数: tags (标签列表), top_k (返回数量)
  - 返回: 搜索结果

- **search_by_metadata**: 基于元数据搜索
  - 参数: metadata (元数据条件), top_k (返回数量)
  - 返回: 搜索结果

### 2.3 搜索优化
- **set_search_params**: 设置搜索参数
  - 参数: params (搜索参数字典)
  - 返回: 配置对象

- **get_search_stats**: 获取搜索统计信息
  - 参数: 无
  - 返回: 统计信息 (搜索次数, 平均响应时间等)

- **optimize_search**: 优化搜索性能
  - 参数: 无
  - 返回: 优化结果

## 3. 记忆管理

### 3.1 记忆操作
- **create_memory**: 创建记忆
  - 参数: data (数据), metadata (元数据), tags (标签)
  - 返回: 记忆ID

- **read_memory**: 读取记忆
  - 参数: memory_id (记忆ID)
  - 返回: 记忆内容 (数据, 元数据, 标签, 创建时间, 更新时间)

- **update_memory**: 更新记忆
  - 参数: memory_id (记忆ID), data (数据), metadata (元数据), tags (标签)
  - 返回: 是否成功

- **delete_memory**: 删除记忆
  - 参数: memory_id (记忆ID)
  - 返回: 是否成功

### 3.2 标签和分类
- **add_tag**: 添加标签
  - 参数: memory_id (记忆ID), tag (标签)
  - 返回: 是否成功

- **remove_tag**: 移除标签
  - 参数: memory_id (记忆ID), tag (标签)
  - 返回: 是否成功

- **get_tags**: 获取所有标签
  - 参数: 无
  - 返回: 标签列表

- **get_memories_by_tag**: 获取指定标签的记忆
  - 参数: tag (标签), top_k (返回数量)
  - 返回: 记忆列表

### 3.3 过期和清理
- **set_expiry**: 设置记忆过期时间
  - 参数: memory_id (记忆ID), expiry_time (过期时间)
  - 返回: 是否成功

- **get_expired_memories**: 获取过期记忆
  - 参数: 无
  - 返回: 过期记忆列表

- **clean_expired_memories**: 清理过期记忆
  - 参数: 无
  - 返回: 清理结果

- **clean_duplicate_memories**: 清理重复记忆
  - 参数: 无
  - 返回: 清理结果

### 3.4 导入和导出
- **import_memories**: 导入记忆
  - 参数: file_path (文件路径), format (格式: 'json'/'csv')
  - 返回: 导入结果

- **export_memories**: 导出记忆
  - 参数: file_path (文件路径), format (格式: 'json'/'csv'), filters (过滤条件)
  - 返回: 导出结果

- **backup_memories**: 备份记忆
  - 参数: backup_path (备份路径)
  - 返回: 备份结果

- **restore_memories**: 恢复记忆
  - 参数: backup_path (备份路径)
  - 返回: 恢复结果

## 4. 记忆增强

### 4.1 自动总结
- **summarize_memory**: 总结记忆内容
  - 参数: memory_id (记忆ID), max_length (最大长度)
  - 返回: 总结内容

- **summarize_batch**: 批量总结记忆
  - 参数: memory_ids (记忆ID列表), max_length (最大长度)
  - 返回: 总结内容列表

- **auto_summarize**: 自动总结新记忆
  - 参数: enabled (是否启用), max_length (最大长度)
  - 返回: 是否成功

### 4.2 关联和链接
- **find_related_memories**: 查找相关记忆
  - 参数: memory_id (记忆ID), top_k (返回数量)
  - 返回: 相关记忆列表

- **link_memories**: 链接相关记忆
  - 参数: memory_id1 (记忆ID1), memory_id2 (记忆ID2), relationship (关系类型)
  - 返回: 是否成功

- **get_related_memories**: 获取链接的记忆
  - 参数: memory_id (记忆ID), relationship (关系类型)
  - 返回: 链接的记忆列表

### 4.3 上下文理解
- **add_context**: 添加上下文
  - 参数: memory_id (记忆ID), context (上下文信息)
  - 返回: 是否成功

- **get_context**: 获取上下文
  - 参数: memory_id (记忆ID)
  - 返回: 上下文信息

- **enhance_with_context**: 使用上下文增强记忆
  - 参数: memory_id (记忆ID)
  - 返回: 增强后的记忆

### 4.4 重要性评估
- **evaluate_importance**: 评估记忆重要性
  - 参数: memory_id (记忆ID)
  - 返回: 重要性评分 (0-100)

- **auto_evaluate_importance**: 自动评估新记忆重要性
  - 参数: enabled (是否启用)
  - 返回: 是否成功

- **get_important_memories**: 获取重要记忆
  - 参数: min_importance (最小重要性), top_k (返回数量)
  - 返回: 重要记忆列表

## 5. 错误处理

### 5.1 异常处理
- **handle_memory_errors**: 处理记忆操作异常
  - 参数: function (要执行的函数), *args, **kwargs
  - 返回: 执行结果或错误信息

- **validate_memory_operation**: 验证记忆操作的合法性
  - 参数: operation (操作类型), *args, **kwargs
  - 返回: 是否合法

- **retry_memory_operation**: 重试记忆操作
  - 参数: function (要执行的函数), max_attempts (最大尝试次数), *args, **kwargs
  - 返回: 执行结果

### 5.2 数据验证
- **validate_memory_data**: 验证记忆数据
  - 参数: data (数据), metadata (元数据), tags (标签)
  - 返回: 验证结果

- **validate_embedding**: 验证向量嵌入
  - 参数: embedding (向量)
  - 返回: 验证结果

## 6. 性能优化

### 6.1 缓存机制
- **set_memory_cache**: 设置记忆缓存
  - 参数: cache_size (缓存大小), expiry (过期时间)
  - 返回: 缓存实例

- **clear_memory_cache**: 清除记忆缓存
  - 参数: cache (缓存实例)
  - 返回: 是否成功

- **get_cached_memory**: 获取缓存的记忆
  - 参数: memory_id (记忆ID)
  - 返回: 记忆内容或None

### 6.2 索引优化
- **optimize_index**: 优化向量索引
  - 参数: index_id (索引ID)
  - 返回: 优化结果

- **rebuild_index**: 重建向量索引
  - 参数: index_id (索引ID)
  - 返回: 重建结果

- **compact_index**: 压缩向量索引
  - 参数: index_id (索引ID)
  - 返回: 压缩结果

### 6.3 批量处理
- **batch_process_memories**: 批量处理记忆
  - 参数: memories (记忆列表), process_func (处理函数)
  - 返回: 处理结果列表

- **parallel_processing**: 并行处理记忆
  - 参数: memories (记忆列表), process_func (处理函数), max_workers (最大工作线程数)
  - 返回: 处理结果列表

## 7. 安全性

### 7.1 数据加密
- **encrypt_memory**: 加密记忆
  - 参数: memory_id (记忆ID), key (密钥)
  - 返回: 是否成功

- **decrypt_memory**: 解密记忆
  - 参数: memory_id (记忆ID), key (密钥)
  - 返回: 解密后的记忆

- **set_encryption_key**: 设置加密密钥
  - 参数: key (密钥)
  - 返回: 是否成功

### 7.2 访问控制
- **set_memory_access**: 设置记忆访问权限
  - 参数: memory_id (记忆ID), permissions (权限设置)
  - 返回: 是否成功

- **check_memory_access**: 检查记忆访问权限
  - 参数: memory_id (记忆ID), user (用户)
  - 返回: 是否有权限

- **restrict_memory_access**: 限制记忆访问
  - 参数: memory_id (记忆ID), users (用户列表)
  - 返回: 是否成功

### 7.3 安全审计
- **log_memory_operation**: 记录记忆操作日志
  - 参数: operation (操作类型), memory_id (记忆ID), user (用户), status (状态)
  - 返回: 无

- **get_operation_history**: 获取操作历史
  - 参数: memory_id (记忆ID), limit (限制数量)
  - 返回: 操作历史列表

- **audit_memory_access**: 审计记忆访问
  - 参数: memory_id (记忆ID)
  - 返回: 访问审计报告

## 8. 可扩展性

### 8.1 插件系统
- **register_memory_operation**: 注册自定义记忆操作
  - 参数: name (操作名称), function (操作函数)
  - 返回: 是否成功

- **get_memory_operations**: 获取所有可用的记忆操作
  - 参数: 无
  - 返回: 操作列表

### 8.2 扩展接口
- **MemoryOperation**: 记忆操作基类
  - 提供统一的操作接口
  - 支持自定义操作实现

- **MemoryOperationManager**: 记忆操作管理器
  - 管理所有记忆操作
  - 提供操作调度和执行

### 8.3 存储后端
- **register_storage_backend**: 注册存储后端
  - 参数: name (后端名称), backend (后端实现)
  - 返回: 是否成功

- **set_default_storage_backend**: 设置默认存储后端
  - 参数: backend_name (后端名称)
  - 返回: 是否成功

- **get_storage_backends**: 获取所有可用的存储后端
  - 参数: 无
  - 返回: 后端列表