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

如果需要在节点服务器上直接配置，可以通过 v2node 的 `custom_outbound.json` 和 `custom_route.json` 文件添加出站代理链，将所有出口流量经过 Decodo 住宅 IP。

> **适用场景**：面板不支持自定义出站配置，或需要更精细地控制节点层面的出站行为。

#### 第一步：SSH 登录到 v2node 服务器

```bash
ssh your_user@你的服务器IP
```

> **安全建议**：建议使用具有 sudo 权限的非 root 用户登录，避免直接以 root 身份操作。以下命令如需提升权限，请在前面加 `sudo`。

#### 第二步：确认 v2node 配置目录

v2node（基于 XrayR）的配置文件通常位于以下路径之一：

```bash
# 查看 v2node 服务文件，确认配置路径
cat /etc/systemd/system/v2node.service

# 常见配置目录
ls /etc/v2node/
# 或
ls /etc/XrayR/
```

以下步骤假设配置目录为 `/etc/v2node/`，请根据实际情况替换。

#### 第三步：创建自定义出站配置文件

创建 `custom_outbound.json`，定义通过 Decodo SOCKS5 代理的出站链：

```bash
cat > /etc/v2node/custom_outbound.json << 'EOF'
[
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
    "tag": "proxy_via_decodo",
    "protocol": "freedom",
    "settings": {},
    "proxySettings": {
      "tag": "decodo_socks"
    }
  }
]
EOF
```

> **说明**：
> - `decodo_socks`：定义到 Decodo SOCKS5 代理的出站连接
> - `proxy_via_decodo`：通过 `proxySettings` 将 freedom 出站链接到 `decodo_socks`，实现代理链
> - 请将 `你的Decodo用户名` 和 `你的Decodo密码` 替换为 Decodo 控制台中获取的实际凭据
>
> **安全提示**：配置文件中包含代理凭据，请设置正确的文件权限防止未授权访问：
> ```bash
> chmod 600 /etc/v2node/custom_outbound.json
> ```

#### 第四步：创建自定义路由配置文件

创建 `custom_route.json`，将所有流量路由到 Decodo 代理出站：

```bash
cat > /etc/v2node/custom_route.json << 'EOF'
{
  "domainStrategy": "AsIs",
  "rules": [
    {
      "type": "field",
      "outboundTag": "proxy_via_decodo",
      "network": "tcp,udp"
    }
  ]
}
EOF
```

> **说明**：此规则将所有 TCP 和 UDP 流量都通过 `proxy_via_decodo` 出站，从而经过 Decodo 住宅 IP。

#### 第五步：修改 v2node 主配置文件

编辑 `config.yml`，添加自定义出站和路由文件的路径引用：

```bash
vi /etc/v2node/config.yml
```

在配置文件中找到或添加以下字段：

```yaml
# 自定义出站配置文件路径
OutboundConfigPath: /etc/v2node/custom_outbound.json
# 自定义路由配置文件路径
RouteConfigPath: /etc/v2node/custom_route.json
```

完整的 `config.yml` 示例（仅展示关键部分）：

```yaml
Log:
  Level: warning

OutboundConfigPath: /etc/v2node/custom_outbound.json
RouteConfigPath: /etc/v2node/custom_route.json

Nodes:
  - PanelType: "NewV2board"
    ApiConfig:
      ApiHost: "https://你的面板地址"
      ApiKey: "你的API密钥"
      NodeID: 1
      NodeType: V2ray
      Timeout: 30
    ControllerConfig:
      ListenIP: 0.0.0.0
      SendIP: 0.0.0.0
      UpdatePeriodic: 60
```

#### 第六步：重启 v2node 服务

```bash
# 重启服务
systemctl restart v2node

# 检查服务状态，确认正常运行
systemctl status v2node

# 查看日志排查错误（如有）
journalctl -u v2node --no-pager -n 50
```

#### 第七步：验证出口 IP

在 v2node 服务器上测试出口 IP 是否已切换：

```bash
# 直接测试服务器出口 IP（应为服务器原始 IP）
curl https://ip.decodo.com/json

# 通过 Decodo 代理测试（应为住宅 IP）
# 注意：命令行中的凭据可能出现在进程列表和 shell 历史中
# 建议测试完成后运行 history -c 清除历史记录
curl -x socks5h://你的Decodo用户名:你的Decodo密码@gate.decodo.com:7000 https://ip.decodo.com/json
```

然后通过客户端连接 v2node 节点，访问 [https://ip.decodo.com/json](https://ip.decodo.com/json)，确认显示的 IP 为 Decodo 住宅 IP 而非服务器 IP。

#### 故障排查

| 问题 | 排查方法 |
|------|---------|
| 服务启动失败 | 运行 `journalctl -u v2node --no-pager -n 100` 查看日志 |
| JSON 格式错误 | 使用 `python3 -m json.tool /etc/v2node/custom_outbound.json` 验证 JSON |
| 路径不正确 | 确认 `config.yml` 中的路径与实际文件路径一致 |
| 出口 IP 未变化 | 检查路由规则是否生效，确认 `outboundTag` 名称与出站 `tag` 匹配 |
| Decodo 认证失败 | 在 Decodo 控制台确认用户名密码，用 curl 单独测试代理连通性 |

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
