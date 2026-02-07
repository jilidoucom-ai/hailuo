# 使用 Decodo 住宅IP 代理出口流量指南

本指南说明如何在使用 [v2boardz](https://github.com/wyx2685/v2boardz)（前端面板）+ [v2node](https://github.com/wyx2685/v2node)（后端节点）架构下，将出口流量通过 [Decodo](https://decodo.com)（原 Smartproxy）购买的住宅 IP 进行路由。

## 前提条件

- 已部署 v2boardz 前端面板并可正常使用
- 已部署 v2node 后端节点并与面板成功对接
- 已购买 Decodo 住宅代理服务，并获取到代理凭据（用户名和密码）

## Decodo 住宅代理信息

| 项目 | 值 |
|------|------|
| 代理地址 | `gate.decodo.com` |
| 代理端口 | `7000` |
| 协议 | HTTP / SOCKS5 |
| 认证方式 | 用户名 + 密码 |

> **注意**: SOCKS5 仅支持主网关 `gate.decodo.com:7000`，国家专用端点（如 `us.decodo.com`）不支持 SOCKS5。如需指定国家，请使用用户名中的高级参数。

## 配置方法

### 方法一：通过 v2boardz 面板配置自定义出站规则

在 v2boardz 管理面板中，找到对应节点的配置，添加自定义出站（outbound）配置。

#### 使用 SOCKS5 协议

```json
{
  "outbounds": [
    {
      "tag": "decodo_residential",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "gate.decodo.com",
            "port": 7000,
            "users": [
              {
                "user": "你的Decodo用户名",
                "pass": "你的Decodo密码"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

#### 使用 HTTP 协议

```json
{
  "outbounds": [
    {
      "tag": "decodo_residential",
      "protocol": "http",
      "settings": {
        "servers": [
          {
            "address": "gate.decodo.com",
            "port": 7000,
            "users": [
              {
                "user": "你的Decodo用户名",
                "pass": "你的Decodo密码"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

### 方法二：在 v2node 服务器上直接修改 XRay 配置

如果需要在节点服务器上直接配置，可以修改 XRay 核心的配置文件，添加出站代理链。

#### 完整配置示例（SOCKS5）

在 XRay 配置文件中添加以下出站和路由规则：

```json
{
  "outbounds": [
    {
      "tag": "proxy_via_decodo",
      "protocol": "freedom",
      "settings": {},
      "proxySettings": {
        "tag": "decodo_socks"
      }
    },
    {
      "tag": "decodo_socks",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "gate.decodo.com",
            "port": 7000,
            "users": [
              {
                "user": "你的Decodo用户名",
                "pass": "你的Decodo密码"
              }
            ]
          }
        ]
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "outboundTag": "proxy_via_decodo",
        "network": "tcp,udp"
      }
    ]
  }
}
```

**说明**：
- `decodo_socks`：定义到 Decodo SOCKS5 代理的出站连接
- `proxy_via_decodo`：通过 `proxySettings` 将 freedom 出站链接到 Decodo 代理，实现代理链
- `routing` 中的规则将所有流量路由到 `proxy_via_decodo`，从而经过 Decodo 住宅 IP 出站

## 高级用法：指定地区

Decodo 支持通过用户名参数指定代理所在地区。将用户名格式修改为：

```
user-你的用户名-country-国家代码
```

### 示例

| 目标地区 | 用户名格式 |
|---------|-----------|
| 美国 | `user-你的用户名-country-us` |
| 日本 | `user-你的用户名-country-jp` |
| 英国 | `user-你的用户名-country-gb` |
| 香港 | `user-你的用户名-country-hk` |

### 固定 IP（粘性会话）

如需在一段时间内保持相同的出口 IP，可添加 `session` 参数：

```
user-你的用户名-country-us-session-my_session_1
```

## 验证配置

配置完成后，可在 v2node 服务器上使用以下命令验证出口 IP 是否已切换为 Decodo 住宅 IP：

```bash
# 通过 SOCKS5 代理测试
curl -x socks5h://你的用户名:你的密码@gate.decodo.com:7000 https://ip.decodo.com/json

# 通过 HTTP 代理测试
curl -x http://你的用户名:你的密码@gate.decodo.com:7000 https://ip.decodo.com/json
```

如果返回的 IP 地址为住宅 IP 而非服务器 IP，则说明配置成功。

## 注意事项

1. **流量消耗**：住宅代理按流量计费，请注意监控流量使用情况
2. **延迟影响**：经过住宅代理转发会增加一定延迟，建议选择就近地区的代理节点
3. **协议选择**：SOCKS5 通常比 HTTP 代理性能更好，推荐优先使用
4. **安全性**：请妥善保管 Decodo 代理凭据，不要将其提交到代码仓库中
5. **IP 轮换**：Decodo 住宅代理默认会轮换 IP，如需固定 IP 请使用粘性会话参数

## 参考链接

- [Decodo 住宅代理快速开始](https://help.decodo.com/docs/residential-proxy-quick-start)
- [Decodo 高级参数](https://help.decodo.com/docs/residential-proxy-advanced-parameters)
- [XRay 出站代理配置](https://xtls.github.io/en/config/outbound.html)
- [v2node 项目](https://github.com/wyx2685/v2node)
- [v2boardz 项目](https://github.com/wyx2685/v2boardz)
