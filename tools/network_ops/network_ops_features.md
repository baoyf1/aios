# 网络操作工具功能文档

## 1. HTTP/HTTPS请求

### 1.1 基本HTTP请求
- **http_get**: 发送HTTP GET请求
  - 参数: url (URL), headers (请求头), params (查询参数), timeout (超时时间)
  - 返回: 响应对象 (状态码、响应头、响应体)

- **http_post**: 发送HTTP POST请求
  - 参数: url (URL), data (请求数据), json (JSON数据), headers (请求头), timeout (超时时间)
  - 返回: 响应对象

- **http_put**: 发送HTTP PUT请求
  - 参数: url (URL), data (请求数据), json (JSON数据), headers (请求头), timeout (超时时间)
  - 返回: 响应对象

- **http_delete**: 发送HTTP DELETE请求
  - 参数: url (URL), headers (请求头), timeout (超时时间)
  - 返回: 响应对象

- **http_head**: 发送HTTP HEAD请求
  - 参数: url (URL), headers (请求头), timeout (超时时间)
  - 返回: 响应对象 (仅响应头)

### 1.2 请求配置
- **set_request_headers**: 设置请求头
  - 参数: headers (请求头字典)
  - 返回: 配置对象

- **set_request_timeout**: 设置请求超时
  - 参数: timeout (超时时间，秒)
  - 返回: 配置对象

- **set_request_proxy**: 设置代理
  - 参数: proxy (代理URL)
  - 返回: 配置对象

- **set_request_auth**: 设置认证信息
  - 参数: username (用户名), password (密码)
  - 返回: 配置对象

### 1.3 响应处理
- **get_response_status**: 获取响应状态码
  - 参数: response (响应对象)
  - 返回: 状态码

- **get_response_headers**: 获取响应头
  - 参数: response (响应对象)
  - 返回: 响应头字典

- **get_response_content**: 获取响应内容
  - 参数: response (响应对象), encoding (编码)
  - 返回: 响应内容

- **get_response_json**: 获取响应JSON数据
  - 参数: response (响应对象)
  - 返回: JSON字典

## 2. 网络工具

### 2.1 DNS查询
- **dns_lookup**: 执行DNS查询
  - 参数: hostname (主机名), record_type (记录类型: 'A'/'AAAA'/'CNAME'/'MX'/'NS'/'TXT')
  - 返回: 查询结果列表

- **reverse_dns**: 执行反向DNS查询
  - 参数: ip_address (IP地址)
  - 返回: 主机名

- **dns_resolve**: 解析主机名为IP地址
  - 参数: hostname (主机名)
  - 返回: IP地址列表

### 2.2 网络测试
- **ping**: 执行网络ping测试
  - 参数: host (主机名或IP地址), count (ping次数), timeout (超时时间)
  - 返回: ping结果 (延迟、丢包率等)

- **traceroute**: 执行网络路由跟踪
  - 参数: host (主机名或IP地址), max_hops (最大跳数)
  - 返回: 路由路径列表

- **port_scan**: 执行端口扫描
  - 参数: host (主机名或IP地址), ports (端口范围), timeout (超时时间)
  - 返回: 开放端口列表

- **network_diagnostics**: 网络诊断
  - 参数: host (主机名或IP地址)
  - 返回: 诊断结果

### 2.3 网络状态
- **check_network_status**: 检查网络连接状态
  - 参数: host (主机名或IP地址)
  - 返回: 网络状态 (在线/离线)

- **get_public_ip**: 获取公网IP地址
  - 参数: 无
  - 返回: 公网IP地址

- **get_local_ip**: 获取本地IP地址
  - 参数: 无
  - 返回: 本地IP地址列表

## 3. 文件传输

### 3.1 HTTP文件传输
- **download_file**: 下载文件
  - 参数: url (文件URL), output_path (输出路径), timeout (超时时间), chunk_size (块大小)
  - 返回: 下载结果 (成功/失败, 文件路径)

- **upload_file**: 上传文件
  - 参数: url (上传URL), file_path (文件路径), headers (请求头), timeout (超时时间)
  - 返回: 上传结果

- **resume_download**: 断点续传下载
  - 参数: url (文件URL), output_path (输出路径), resume_from (续传位置)
  - 返回: 下载结果

### 3.2 FTP客户端
- **ftp_connect**: 连接FTP服务器
  - 参数: host (主机名), port (端口), username (用户名), password (密码)
  - 返回: FTP客户端实例

- **ftp_download**: 从FTP服务器下载文件
  - 参数: ftp (FTP客户端实例), remote_path (远程路径), local_path (本地路径)
  - 返回: 下载结果

- **ftp_upload**: 上传文件到FTP服务器
  - 参数: ftp (FTP客户端实例), local_path (本地路径), remote_path (远程路径)
  - 返回: 上传结果

- **ftp_list**: 列出FTP服务器文件
  - 参数: ftp (FTP客户端实例), path (路径)
  - 返回: 文件列表

- **ftp_disconnect**: 断开FTP连接
  - 参数: ftp (FTP客户端实例)
  - 返回: 是否成功

### 3.3 SFTP客户端
- **sftp_connect**: 连接SFTP服务器
  - 参数: host (主机名), port (端口), username (用户名), password (密码) 或 key_file (密钥文件)
  - 返回: SFTP客户端实例

- **sftp_download**: 从SFTP服务器下载文件
  - 参数: sftp (SFTP客户端实例), remote_path (远程路径), local_path (本地路径)
  - 返回: 下载结果

- **sftp_upload**: 上传文件到SFTP服务器
  - 参数: sftp (SFTP客户端实例), local_path (本地路径), remote_path (远程路径)
  - 返回: 上传结果

- **sftp_list**: 列出SFTP服务器文件
  - 参数: sftp (SFTP客户端实例), path (路径)
  - 返回: 文件列表

- **sftp_disconnect**: 断开SFTP连接
  - 参数: sftp (SFTP客户端实例)
  - 返回: 是否成功

## 4. 网络监控

### 4.1 网络连接
- **list_connections**: 列出网络连接
  - 参数: protocol (协议: 'TCP'/'UDP'), state (状态: 'ESTABLISHED'/'LISTEN'等)
  - 返回: 网络连接列表

- **monitor_connections**: 监控网络连接
  - 参数: interval (采样间隔), callback (回调函数)
  - 返回: 监控器实例

- **stop_connection_monitor**: 停止网络连接监控
  - 参数: monitor (监控器实例)
  - 返回: 是否成功

### 4.2 网络统计
- **get_network_stats**: 获取网络统计信息
  - 参数: interface (网络接口)
  - 返回: 网络统计信息 (发送/接收字节数、包数等)

- **monitor_network_stats**: 监控网络统计信息
  - 参数: interface (网络接口), interval (采样间隔), callback (回调函数)
  - 返回: 监控器实例

- **stop_stats_monitor**: 停止网络统计监控
  - 参数: monitor (监控器实例)
  - 返回: 是否成功

### 4.3 网络流量
- **get_network_usage**: 获取网络流量使用情况
  - 参数: interface (网络接口)
  - 返回: 网络流量使用情况 (上传/下载速度)

- **monitor_network_usage**: 监控网络流量
  - 参数: interface (网络接口), interval (采样间隔), callback (回调函数)
  - 返回: 监控器实例

- **stop_usage_monitor**: 停止网络流量监控
  - 参数: monitor (监控器实例)
  - 返回: 是否成功

### 4.4 网络接口
- **list_network_interfaces**: 列出网络接口
  - 参数: 无
  - 返回: 网络接口列表

- **get_interface_info**: 获取网络接口信息
  - 参数: interface (网络接口名称)
  - 返回: 接口信息 (IP地址、MAC地址等)

- **get_interface_status**: 获取网络接口状态
  - 参数: interface (网络接口名称)
  - 返回: 接口状态 (启用/禁用)

## 5. 错误处理

### 5.1 异常处理
- **handle_network_errors**: 处理网络操作异常
  - 参数: function (要执行的函数), *args, **kwargs
  - 返回: 执行结果或错误信息

- **validate_network_operation**: 验证网络操作的合法性
  - 参数: operation (操作类型), *args, **kwargs
  - 返回: 是否合法

- **retry_network_operation**: 重试网络操作
  - 参数: function (要执行的函数), max_attempts (最大尝试次数), *args, **kwargs
  - 返回: 执行结果

### 5.2 网络诊断
- **diagnose_network_issue**: 诊断网络问题
  - 参数: host (主机名或IP地址)
  - 返回: 诊断结果

- **test_network_connectivity**: 测试网络连接性
  - 参数: hosts (主机列表)
  - 返回: 连接测试结果

## 6. 性能优化

### 6.1 连接池
- **create_connection_pool**: 创建HTTP连接池
  - 参数: size (池大小), timeout (超时时间)
  - 返回: 连接池实例

- **use_connection_pool**: 使用连接池发送请求
  - 参数: pool (连接池实例), request_func (请求函数), *args, **kwargs
  - 返回: 响应对象

- **close_connection_pool**: 关闭连接池
  - 参数: pool (连接池实例)
  - 返回: 是否成功

### 6.2 并行请求
- **parallel_requests**: 并行执行HTTP请求
  - 参数: requests (请求列表)
  - 返回: 响应列表

- **batch_download**: 批量下载文件
  - 参数: urls (URL列表), output_dir (输出目录), max_concurrent (最大并发数)
  - 返回: 下载结果列表

### 6.3 缓存
- **set_request_cache**: 设置请求缓存
  - 参数: cache_dir (缓存目录), max_size (最大缓存大小)
  - 返回: 缓存实例

- **clear_request_cache**: 清除请求缓存
  - 参数: cache (缓存实例)
  - 返回: 是否成功

## 7. 安全性

### 7.1 HTTPS安全
- **verify_ssl**: 验证SSL证书
  - 参数: url (URL), verify (是否验证)
  - 返回: 验证结果

- **set_ssl_context**: 设置SSL上下文
  - 参数: cert (证书路径), key (密钥路径)
  - 返回: SSL上下文

### 7.2 网络安全
- **scan_security_vulnerabilities**: 扫描网络安全漏洞
  - 参数: host (主机名或IP地址)
  - 返回: 漏洞列表

- **secure_network_request**: 安全网络请求
  - 参数: url (URL), **kwargs
  - 返回: 响应对象

### 7.3 数据保护
- **encrypt_network_data**: 加密网络数据
  - 参数: data (数据), key (密钥)
  - 返回: 加密数据

- **decrypt_network_data**: 解密网络数据
  - 参数: encrypted_data (加密数据), key (密钥)
  - 返回: 解密数据

## 8. 可扩展性

### 8.1 插件系统
- **register_network_operation**: 注册自定义网络操作
  - 参数: name (操作名称), function (操作函数)
  - 返回: 是否成功

- **get_network_operations**: 获取所有可用的网络操作
  - 参数: 无
  - 返回: 操作列表

### 8.2 扩展接口
- **NetworkOperation**: 网络操作基类
  - 提供统一的操作接口
  - 支持自定义操作实现

- **NetworkOperationManager**: 网络操作管理器
  - 管理所有网络操作
  - 提供操作调度和执行