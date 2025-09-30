# IP DERP Server

自托管的 Tailscale DERP 中继服务器，支持使用 IP 地址（自签名证书）运行。

## 功能特性

- 支持使用 IP 地址直接访问（无需域名）
- 自动生成自签名 SSL 证书
- 内置 STUN 服务支持
- Docker 容器化部署
- 可配置的客户端验证选项

## 快速开始

### 构建镜像

```bash
docker build -t ip_derper .
```

### 运行容器

```bash
docker run -d \
  -e DERP_HOST=<你的服务器IP> \
  -e DERP_ADDR=:443 \
  -e DERP_HTTP_PORT=80 \
  -e DERP_STUN=true \
  -e DERP_VERIFY_CLIENTS=false \
  -p 80:80 \
  -p 443:443 \
  -p 3478:3478/udp \
  --name derper \
  ip_derper
```

## 配置参数

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `DERP_HOST` | `127.0.0.1` | 服务器 IP 地址 |
| `DERP_ADDR` | `:443` | DERP 服务监听地址 |
| `DERP_HTTP_PORT` | `80` | HTTP 端口 |
| `DERP_CERTS` | `/app/certs/` | SSL 证书目录 |
| `DERP_STUN` | `true` | 是否启用 STUN 服务 |
| `DERP_VERIFY_CLIENTS` | `false` | 是否验证客户端身份 |

## 端口要求

容器需要开放以下端口：

- **TCP 80**: HTTP 服务
- **TCP 443**: HTTPS/DERP 服务
- **UDP 3478**: STUN 服务

确保防火墙允许这些端口的流量通过。

## 工作原理

1. 容器启动时自动执行 `build_cert.sh` 脚本
2. 为指定的 IP 地址生成包含 SAN (Subject Alternative Name) 的自签名证书
3. 启动 derper 服务，使用生成的证书

## 证书生成

`build_cert.sh` 脚本会：
- 生成 OpenSSL 配置文件
- 创建 2048 位 RSA 密钥
- 生成有效期 730 天的自签名证书
- 将 IP 地址添加到证书的 SAN 字段

## 使用场景

- 内网环境部署 DERP 服务器
- 临时测试和开发环境
- 无法使用域名的场景
- 需要 IP 直连的网络架构

## 注意事项

⚠️ **重要提示**

- 使用自签名证书，客户端需要信任该证书
- 不推荐在生产环境使用自签名证书
- 如有域名，建议使用 Let's Encrypt 证书
- 跨 Tailnet 功能（如节点共享）在自定义 DERP 服务器上不可用
- 定期更新 derper 二进制文件以保持与 Tailscale 客户端兼容

## 客户端验证模式

如果启用 `DERP_VERIFY_CLIENTS=true`：
- 需要在同一主机上运行 `tailscaled`
- 所有客户端必须在 ACL 中对 derper 的 tailscaled 可见
- derper 和 tailscaled 二进制文件需要从相同的 git 版本构建

## 监控和诊断

- Debug 处理器: `http://localhost/debug/`（仅本地或 Tailscale IP 可访问）
- Prometheus 指标: `http://localhost/debug/varz`
- Go pprof: `http://localhost/debug/pprof/`

使用 `tailscale debug derp` 和 `tailscale netcheck` 进行客户端诊断。

## 项目来源

基于 [Tailscale DERP 服务器](https://tailscale.com/kb/1232/derp-servers) 实现。

## License

本项目基于 Tailscale 开源代码，遵循相应的开源协议。