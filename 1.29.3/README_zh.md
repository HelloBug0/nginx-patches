# Nginx 补丁: 支持带端口的 SSL Server Name 匹配

## 配置指令
Syntax:  ssl_server_name_with_port on | off;
Default: ssl_server_name_with_port off;
Context: http

## 简介
此补丁 (`01-ssl_server_name_with_port.patch`) 旨在增强 Nginx 的 SSL 握手处理能力，使其支持匹配包含端口号的 Server Name (SNI)。

在标准的 Nginx 逻辑中，SNI 匹配通常仅基于域名。然而，某些客户端或代理环境在发送 ClientHello 时，可能会在 ServerName 字段中包含端口号（例如 `example.com:443`）。如果不做处理，标准 Nginx 往往无法正确匹配到配置了相应 `server_name` 的 server 块。

本补丁引入了 `ssl_server_name_with_port` 指令。开启后，Nginx 将利用 Proxy Protocol 传递的目标端口信息（若 Proxy Protocol 缺失则使用本地监听端口），尝试构造 `host:port` 格式的 Server Name 进行二次匹配。

## 功能特性
- **新指令**: `ssl_server_name_with_port` (on/off)
- **动态匹配**: 结合 Proxy Protocol 获取的目标端口（或本地监听端口），支持识别 `Host:Port` 格式的 SNI 请求。

## 适用环境
- **开发基准**: Nginx 1.29.3
- **测试系统**: CentOS 7 / 8
- **模块依赖**: 必须启用 HTTP SSL 模块 (`--with-http_ssl_module`)

## 使用指南

### 1. 应用补丁
将补丁文件放置于 Nginx 源码目录外，进入源码目录执行：

```bash
# 假设补丁路径为 ../01-ssl_server_name_with_port.patch
patch -p1 < ../01-ssl_server_name_with_port.patch
```

### 2. 编译安装 (关键步骤)
**注意**：本补丁被 `NGX_SSL_SERVER_NAME_WITH_PORT` 宏包围。**必须**在配置编译参数时通过 `--with-cc-opt` 显式定义该宏，否则功能将不会被编译。

```bash
# 配置编译参数，务必添加 -DNGX_SSL_SERVER_NAME_WITH_PORT
./configure \
    --with-http_ssl_module \
    --with-cc-opt="-DNGX_SSL_SERVER_NAME_WITH_PORT" \
    ... (其他参数)

# 编译并安装
make
sudo make install
```

### 3. 配置示例

在 `nginx.conf` 中配置使用：

```nginx
http {
    # 【核心配置】开启带端口的 SNI 匹配功能
    ssl_server_name_with_port on;
    server {
        # 通常配合 Proxy Protocol 使用，以便获取真实目标端口
        listen 443 ssl proxy_protocol;
        
        # 必须在 server_name 中包含带端口的域名，以便匹配成功
        server_name example.com example.com:443;

        ssl_certificate     /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;

        location / {
            return 200 "SNI Matched with Port!\n";
        }
    }
}
```

## 实现原理
1.  **配置解析**: 在 `ngx_http_ssl_module` 中增加了新的配置项解析逻辑。
2.  **握手回调**: 修改了 `ngx_http_request.c` 中的 `ngx_http_ssl_servername` 回调函数。
3.  **逻辑判断**: 当 SSL 握手发生时，如果 `ssl_server_name_with_port` 为 `on`，代码会将原有的 Host 重新格式化为 `Host:Port`（例如 `example.com` -> `example.com:443`），然后重新在哈希表中查找对应的 Server 配置。端口来源优先从 Proxy Protocol 头部获取；如果不存在，则使用服务器的本地监听端口。

## 注意事项
1.  **宏定义**: 再次强调，编译时若遗漏 `-DNGX_SSL_SERVER_NAME_WITH_PORT`，补丁代码将不起任何作用，且不会报错。
2.  **端口解析 (Port Resolution)**: 此功能优先使用 Proxy Protocol 中的目标端口。如果 Proxy Protocol 信息缺失，则回退使用 Nginx 服务器的监听端口。
3.  **Server Name 配置**: 务必在 `server_name` 指令中显式添加带端口的域名（如 `example.com:443`），否则即使 Nginx 构造了带端口的 Host，也找不到对应的 Server 块。

