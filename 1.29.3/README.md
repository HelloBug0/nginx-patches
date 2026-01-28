[中文文档](README_zh.md) | **English**
# Nginx Patch: Support SSL Server Name with Port

## Directive
Syntax:  ssl_server_name_with_port on | off;
Default: ssl_server_name_with_port off;
Context: http

## Introduction
This patch (`01-ssl_server_name_with_port.patch`) enhances Nginx's SSL handshake capabilities by adding support for matching Server Names (SNI) that include a port number.

In standard Nginx logic, SNI matching is typically based solely on the domain name. However, some clients or proxy environments may include the port number in the ServerName field of the ClientHello message (e.g., `example.com:443`). Without modification, standard Nginx often fails to match the correct `server` block configured with the corresponding `server_name`.

This patch introduces the `ssl_server_name_with_port` directive. When enabled, Nginx utilizes the destination port information from the Proxy Protocol (or the local listening port if Proxy Protocol is absent) to construct a `host:port` formatted Server Name for a secondary matching attempt.

## Features
- **New Directive**: `ssl_server_name_with_port` (on/off)
- **Dynamic Matching**: Supports identifying SNI requests in `Host:Port` format by combining the domain with the destination port (obtained via Proxy Protocol or local listener).

## Compatibility
- **Base Version**: Nginx 1.29.3
- **Tested OS**: CentOS 7 / 8
- **Dependencies**: Requires the HTTP SSL module (`--with-http_ssl_module`)

## Usage Guide

### 1. Apply the Patch
Place the patch file outside the Nginx source directory, navigate to the source directory, and execute:

```bash
# Assuming the patch file is located at ../01-ssl_server_name_with_port.patch
patch -p1 < ../01-ssl_server_name_with_port.patch
```

### 2. Compile and Install (Crucial Step)
**Note**: The patch code is guarded by the `NGX_SSL_SERVER_NAME_WITH_PORT` macro. You **MUST** explicitly define this macro using `--with-cc-opt` during configuration; otherwise, the functionality will not be compiled.

```bash
# Configure build options, ensuring -DNGX_SSL_SERVER_NAME_WITH_PORT is added
./configure \
    --with-http_ssl_module \
    --with-cc-opt="-DNGX_SSL_SERVER_NAME_WITH_PORT" \
    ... (other options)

# Compile and install
make
sudo make install
```

### 3. Configuration Example

Configure `nginx.conf` as follows:

```nginx
http {
    # [Core Setting] Enable SNI matching with port
    ssl_server_name_with_port on;
    server {
        # Typically used with Proxy Protocol to retrieve the real destination port
        listen 443 ssl proxy_protocol;
        
        # You must include the domain with the port in server_name for successful matching
        server_name example.com example.com:443;

        ssl_certificate     /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;

        location / {
            return 200 "SNI Matched with Port!\n";
        }
    }
}
```

## Technical Details
1.  **Configuration Parsing**: Adds logic to parse the new directive in `ngx_http_ssl_module`.
2.  **Handshake Callback**: Modifies the `ngx_http_ssl_servername` callback in `ngx_http_request.c`.
3.  **Logic Flow**: During an SSL handshake, if `ssl_server_name_with_port` is `on`, the code reformats the original Host to `Host:Port` (e.g., `example.com` -> `example.com:443`) and re-attempts to look up the corresponding Server configuration in the hash table. The port is derived from the Proxy Protocol header if present; otherwise, the server's local listening port is used.

## Important Notes
1.  **Macro Definition**: Once again, if `-DNGX_SSL_SERVER_NAME_WITH_PORT` is omitted during compilation, the patch code will be inactive, and no error will be raised.
2.  **Port Resolution**: The feature prioritizes the destination port from the Proxy Protocol. If Proxy Protocol information is missing, it falls back to using the Nginx server's listening port.
3.  **Server Name Configuration**: Be sure to explicitly add the domain with the port (e.g., `example.com:443`) to the `server_name` directive; otherwise, even if Nginx constructs the host with the port, it will not find the corresponding Server block.

