# 部署步骤

> 本文档基于本人在黑群晖 DS920+（DSM 7.1）上的一次真实部署整理。
> 文中命令标注「示意」的，是需要你按自己环境调整参数的**参考命令**，不是唯一做法；标注「实测」的，是本次部署中实际执行并验证过的命令。

## 0. 前置条件

- 一台黑群晖 / Synology NAS（本次为 DS920+，DSM 7.1.1-42962）
- 已开启 Docker（群晖叫 **Container Manager**），能 SSH 登录
- 准备两个宿主机目录：`config`（配置主目录）与 `workspace`（智能体工作区）

本次部署目录约定：

| 宿主机路径 | 容器内路径 | 用途 |
|---|---|---|
| `/volume2/docker/openclaw/config` | `/home/node/.openclaw` | 配置主目录（openclaw.json、插件、状态、凭据） |
| `/volume2/docker/openclaw/workspace` | `/home/node/.openclaw/workspace` | 智能体工作区（身份文件、自定义技能、记忆） |

## 1. 启动容器（gateway 模式）

容器以 **host 网络**运行，进程为 `node openclaw.mjs gateway`，入口 `tini`。

```bash
# 拉取镜像（版本号按需更换）
docker pull ghcr.io/openclaw/openclaw:2026.8.2

# 创建并启动容器（示意：请按你实际的 volume / 参数调整）
docker run -d \
  --name openclaw \
  --network host \
  --restart unless-stopped \
  -v /volume2/docker/openclaw/config:/home/node/.openclaw \
  -v /volume2/docker/openclaw/workspace:/home/node/.openclaw/workspace \
  ghcr.io/openclaw/openclaw:2026.8.2
```

- **网络模式必须是 host**：gateway 直接监听宿主机网络栈。此时 `docker ps` 的 PORTS 一列为空是**正常**的（不是没端口）。
- 本次实际监听端口：
  - `0.0.0.0:18789` —— 网关主端口（`gateway.port=18789`, `bind=lan`）
  - `127.0.0.1:18799` —— 仅本机回环的管理端口
- 健康检查每 180s 跑一次，容器状态应保持 `Up (healthy)`。

## 2. 接入 DeepSeek 大模型

DeepSeek 通过 OpenClaw 的 **deepseek 插件**接入，provider 协议为 `openai-completions`。

```bash
# 安装插件（实测命令：网关停止时用一次性容器执行）
openclaw plugins install @openclaw/deepseek --accept-capabilities --force
```

配置结构（openclaw.json，示意）：

```jsonc
{
  "models": {
    "providers": {
      "deepseek": {
        "base_url": "https://api.deepseek.com",
        "protocol": "openai-completions",
        "models": [
          { "name": "deepseek-v4-flash",       "reasoning": true },
          { "name": "deepseek-v4-pro",         "reasoning": true },
          { "name": "deepseek-v4-flash-vision-exp" }   // 视觉版，可读图
        ]
      }
    }
  }
}
```

要点：
- **API key 不写进 openclaw.json**：走 OpenClaw 的 auth profile（`auth.profiles` 里的 `deepseek:setup-…`，`mode: api_key`），密钥存在 agent 级凭据库，相对更安全。容器环境变量里也**不需要**注入 `DEEPSEEK_API_KEY`。
- DeepSeek V4 是推理模型（`reasoning: true`），本次配置上下文窗口 1,000,000、max tokens 384,000。
- 在 `agents.defaults.models` 给常用模型起别名（如 `DeepSeek`），UI 下拉里就能直接选。

## 3. 接入飞书（聊天入口）

飞书**不是普通 skill，而是一个完整渠道插件** `@openclaw/feishu`，自带 4 组飞书工具技能（文档 / 云盘 / 权限 / 知识库）。

### 3.1 安装并启用插件

```bash
# 实测命令（网关停止时执行；版本与网关保持同版本）
openclaw plugins install @openclaw/feishu@2026.8.2 --accept-capabilities --force
```

安装后插件自带的技能目录会被软链到 `config/plugin-skills/`，agent 启动时自动发现：

```
config/plugin-skills/feishu-doc    → 飞书文档
config/plugin-skills/feishu-drive  → 飞书云盘
config/plugin-skills/feishu-perm   → 权限/成员
config/plugin-skills/feishu-wiki   → 知识库
```

### 3.2 在飞书开放平台创建应用

1. 打开 [飞书开放平台](https://open.feishu.cn)，创建企业自建应用。
2. 记下 `App ID` 与 `App Secret`。
3. 按需开启机器人能力与权限（⚠️ 见 troubleshooting：**不勾权限，机器人会「不能发图/发不了某些消息」**）。

### 3.3 配置 channel（openclaw.json，脱敏示意）

```jsonc
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxxxxx",
      "appSecret": "<你的 appSecret>",
      "domain": "feishu",
      "groupPolicy": "open",
      "connectionMode": "websocket"   // ★ 出站长连接
    }
  }
}
```

- **`connectionMode: "websocket"` 是核心**：由 OpenClaw **主动**连飞书服务器，因此 **NAS 不需要给飞书开任何入站端口/端口映射**，内网穿透压力也小很多。
- `channels.feishu.domain="feishu"` 是 OpenClaw 内部的消息域标识，**不是**域名白名单。

## 4. 网页控制台（Control UI）+ 反代

Control UI 经群晖自带 **nginx 反向代理**暴露：外网走 dynv6 IPv6 域名、内网走局域网 IP，统一转发到 `127.0.0.1:18789`。

```jsonc
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "auth": { "mode": "token", "token": "<已脱敏>" },
    "controlUi": {
      "allowedOrigins": [
        "https://<你的域名>:18888",
        "http://<内网IP>:18888"
      ]
    },
    "trustedProxies": ["127.0.0.1", "::1"]
  }
}
```

> ⚠️ **`allowedOrigins` 白名单是逐字比较**「协议 + host + 端口」的。每新增一个访问入口（域名/IP/端口变了），必须同步加进白名单，否则浏览器报 `origin not allowed` —— 详见 troubleshooting 坑 1。

## 5. 验证与日常维护

```bash
# 容器健康状态
docker ps --filter name=openclaw

# 最近活动（含模型调用 / 飞书消息）
docker logs --tail 50 openclaw

# 看/切换默认模型
docker exec openclaw openclaw models list
docker exec openclaw openclaw models get      # models set <模型> 可切换默认

# 健康检查
curl -s http://127.0.0.1:18789/healthz        # => {"ok":true,"status":"live"}
```

> ⚠️ 改了 `openclaw.json` 后**必须 `docker restart openclaw` 才生效**；重启后配置文件属主保持 `1000:1000`、权限 `600`。
> ⚠️ **切换默认模型不会立刻全局生效**：存量会话会记住自己用过的模型。切完最好 `docker restart openclaw` 并新开会话验证。

## 6. 备份与回滚

OpenClaw 的所有数据都在 `config` 与 `workspace` 两个挂载目录，**备份时拷这两处即可**。本次部署保留了升级前的完整备份：

```bash
# 示意：打包配置与工作区
tar czf config-backup-$(date +%Y%m%d).tar.gz -C /volume2/docker/openclaw config workspace
```

需要回滚时：停容器 → 用备份还原目录（属主 `1000:1000`、权限 `600`）→ 重启容器。

## 同机还跑了什么（供参考）

同机 Container Manager 里还运行着 homeassistant、jellyfin、mihomo、cloudflared、ddns-go 等容器，与本项目共用 Docker 目录但不互相依赖。
