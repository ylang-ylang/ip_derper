# 在 OpenWrt 上配置自定义 DERP 服务器

本文档说明如何在 OpenWrt 路由器上配置 Tailscale 使用自定义 DERP 服务器。

## 前置条件

- 已在公网服务器上部署 DERP 服务器
- OpenWrt 路由器已安装 Tailscale
- OpenWrt 能访问公网 DERP 服务器

## 配置步骤

### 1. 创建 DERP 配置文件

SSH 登录到 OpenWrt：

```bash
ssh root@<openwrt-ip>
```

创建配置目录和 DERP map 文件：

```bash
# 创建配置目录
mkdir -p /etc/tailscale

# 创建 DERP map 配置文件
cat > /etc/tailscale/derp.json <<'EOF'
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "custom",
      "RegionName": "Custom DERP",
      "Nodes": [
        {
          "Name": "custom-derp",
          "RegionID": 900,
          "HostName": "<你的公网IP>",
          "IPv4": "<你的公网IP>",
          "InsecureForTests": true,
          "DERPPort": 443,
          "STUNPort": 3478
        }
      ]
    }
  }
}
EOF
```

### 2. 配置参数说明

| 参数 | 说明 | 注意事项 |
|------|------|----------|
| `RegionID` | DERP 区域 ID | 使用 900+ 避免与官方冲突 |
| `RegionCode` | 区域代码 | 自定义标识符 |
| `RegionName` | 区域名称 | 便于识别的名称 |
| `HostName` | 服务器地址 | 公网 IP 或域名 |
| `IPv4` | IPv4 地址 | 服务器的 IPv4 地址 |
| `InsecureForTests` | 跳过证书验证 | 自签名证书必须设为 `true` |
| `DERPPort` | DERP 端口 | 默认 443 |
| `STUNPort` | STUN 端口 | 默认 3478 |

### 3. 修改 Tailscale 启动脚本

编辑 OpenWrt 的 Tailscale init 脚本：

```bash
vi /etc/init.d/tailscale
```

在 `start_service()` 函数中添加 `--derp-map` 参数：

```bash
start_service() {
    procd_open_instance
    procd_set_param command /usr/sbin/tailscaled \
        --state=/var/lib/tailscale/tailscaled.state \
        --socket=/var/run/tailscale/tailscaled.sock \
        --port=41641 \
        --derp-map=/etc/tailscale/derp.json
    procd_set_param respawn
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

### 4. 重启 Tailscale 服务

```bash
/etc/init.d/tailscale restart
```

### 5. 验证配置

检查 DERP 连接状态：

```bash
tailscale netcheck
```

预期输出：

```
Report:
  * UDP: true
  * IPv4: yes, <your-ip>
  * Nearest DERP: Custom DERP (custom)
  * DERP latency:
    - custom: 5ms     ← 自定义 DERP
    - lax: 45ms       ← 官方洛杉矶
    - sfo: 50ms       ← 官方旧金山
    - tok: 120ms      ← 官方东京
```

查看 Tailscale 状态：

```bash
tailscale status
```

## 工作模式

### 混合模式（推荐）

**配置特点**：不添加 `OmitDefaultRegions` 参数

```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      ...
    }
  }
}
```

**行为**：
- ✅ 保留所有官方 DERP 服务器（Region 1-99）
- ✅ 添加自定义 DERP 服务器（Region 900+）
- ✅ 自动选择延迟最低的服务器
- ✅ 自定义 DERP 故障时自动回退到官方

**优势**：
- 局域网或同区域节点优先使用自定义 DERP（低延迟）
- 连接官方网络节点时使用官方 DERP（兼容性好）
- 高可用性（自动故障转移）

### 纯自定义模式

**配置特点**：添加 `"OmitDefaultRegions": true`

```json
{
  "OmitDefaultRegions": true,
  "Regions": {
    "900": {
      "RegionID": 900,
      ...
    }
  }
}
```

**行为**：
- ❌ 禁用所有官方 DERP 服务器
- ✅ 仅使用自定义 DERP 服务器
- ⚠️ 自定义 DERP 故障时无法中继

**适用场景**：
- 无法访问官方 DERP（网络限制）
- 安全合规要求（禁止第三方中继）
- 所有节点都在私有网络内

### 模式对比

| 模式 | 官方 DERP | 自定义 DERP | 故障转移 | 推荐度 |
|------|-----------|-------------|----------|--------|
| 混合模式 | ✅ 启用 | ✅ 启用 | ✅ 自动 | ⭐⭐⭐⭐⭐ |
| 纯自定义 | ❌ 禁用 | ✅ 启用 | ❌ 无 | ⭐⭐⭐ |
| 默认模式 | ✅ 启用 | ❌ 禁用 | ✅ 官方 | ⭐⭐⭐⭐ |

## IPv6 支持

如果 DERP 服务器支持 IPv6，在配置中添加：

```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "custom",
      "RegionName": "Custom DERP",
      "Nodes": [
        {
          "Name": "custom-derp",
          "RegionID": 900,
          "HostName": "<你的公网IP>",
          "IPv4": "<你的IPv4地址>",
          "IPv6": "<你的IPv6地址>",
          "InsecureForTests": true,
          "DERPPort": 443,
          "STUNPort": 3478
        }
      ]
    }
  }
}
```

## 使用域名（可选）

如果 DERP 服务器有域名和正式证书：

```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "custom",
      "RegionName": "Custom DERP",
      "Nodes": [
        {
          "Name": "custom-derp",
          "RegionID": 900,
          "HostName": "derp.example.com",
          "IPv4": "<服务器IPv4>",
          "DERPPort": 443,
          "STUNPort": 3478
        }
      ]
    }
  }
}
```

**注意**：使用正式证书时**移除** `"InsecureForTests": true`

## 故障排查

### 检查服务状态

```bash
# 查看 Tailscale 日志
logread | grep tailscale

# 检查进程
ps | grep tailscaled
```

### 测试网络连通性

```bash
# 测试 DERP 端口
nc -zv <公网IP> 443
nc -zv <公网IP> 80

# 测试 STUN 端口
nc -zu <公网IP> 3478
```

### 检查 DERP 服务器状态

```bash
# 查看服务器指标
curl -k https://<公网IP>/debug/varz

# 从 OpenWrt 测试
curl -k https://<公网IP>
```

### 常见问题

#### 1. 连接失败

**症状**：`tailscale netcheck` 看不到自定义 DERP

**排查**：
- 检查配置文件路径是否正确
- 检查 JSON 格式是否有效
- 查看 Tailscale 日志错误信息

```bash
# 验证 JSON 格式
cat /etc/tailscale/derp.json | jsonfilter -e '@'
```

#### 2. 证书错误

**症状**：日志显示 TLS 证书验证失败

**解决**：确保设置了 `"InsecureForTests": true`

#### 3. 端口不通

**症状**：无法连接到 DERP 服务器

**排查**：
- 公网服务器防火墙是否开放端口
- OpenWrt 防火墙是否允许出站连接
- 网络运营商是否限制端口

```bash
# OpenWrt 上测试
telnet <公网IP> 443
```

#### 4. 配置未生效

**症状**：仍然使用官方 DERP

**解决**：
- 确认 init 脚本已修改
- 重启服务后检查进程参数

```bash
ps | grep tailscaled
# 应该看到 --derp-map=/etc/tailscale/derp.json
```

## 性能优化建议

1. **多区域部署**：如果有多个地理位置，部署多个 DERP 服务器

```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "cn-east",
      "RegionName": "China East",
      "Nodes": [...]
    },
    "901": {
      "RegionID": 901,
      "RegionCode": "cn-west",
      "RegionName": "China West",
      "Nodes": [...]
    }
  }
}
```

2. **使用固定 IP**：避免 DNS 解析增加延迟

3. **启用 STUN**：提高 NAT 穿透成功率

4. **监控服务器**：定期检查 DERP 服务器健康状态

## 安全建议

1. ⚠️ **生产环境使用正式证书**：避免 `InsecureForTests`
2. 🔒 **启用客户端验证**：`DERP_VERIFY_CLIENTS=true`
3. 🔐 **限制访问 IP**：配置防火墙规则
4. 📊 **监控异常流量**：定期检查 `/debug/varz` 指标

## 持久化配置

将配置添加到 OpenWrt 的 UCI 系统：

```bash
# 创建自定义配置文件
cat > /etc/config/tailscale_custom <<EOF
config tailscale 'settings'
    option derp_map '/etc/tailscale/derp.json'
EOF
```

## 相关资源

- [Tailscale DERP 官方文档](https://tailscale.com/kb/1232/derp-servers)
- [OpenWrt Tailscale 包](https://openwrt.org/packages/pkgdata/tailscale)
- 项目主 README: [../README.md](../README.md)
- DERP 服务器构建脚本: [../build_cert.sh](../build_cert.sh)

## 更新日志

- 2025-09-30: 初始版本，支持混合模式和纯自定义模式