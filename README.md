# JADX MCP Server（客户端侧）

本 README 只覆盖 **客户端本地桥接服务**（如何连接云端插件、如何接入 Claude Code）。

端到端流程请先看：
- `<workspace-root>/README.md`

云端插件侧请看：
- `<workspace-root>/jadx-ai-mcp/README.md`

## 1. 作用与边界

这个项目负责：

- 把云端 `jadx-ai-mcp` HTTP 能力桥接成 MCP tools
- 处理 token 鉴权 header
- 给 Claude Code 提供 stdio MCP server

这个项目不负责：

- 云端 JADX 进程与插件生命周期
- 云端端口开放策略

---

## 2. 启动前准备

### 2.1 本地依赖

- Python 3.10+
- `uv`

### 2.2 云端前置

你需要先拿到：

- 云端插件 URL（通常经隧道后为 `http://127.0.0.1:<local-port>`）
- 一次性 token

---

## 3. 启动本地 MCP Server

### 3.1 推荐：token-file 方式

```bash
mkdir -p ~/.secrets && chmod 700 ~/.secrets
printf '%s\n' '<ONE_TIME_TOKEN>' > ~/.secrets/jadx_cloud.token
chmod 600 ~/.secrets/jadx_cloud.token

cd <workspace-root>/jadx-mcp-server
uv run jadx_mcp_server.py \
  --jadx-url http://127.0.0.1:18650 \
  --token-file ~/.secrets/jadx_cloud.token
```

### 3.2 可选：命令行 token

```bash
uv run jadx_mcp_server.py --jadx-url http://127.0.0.1:18650 --token <ONE_TIME_TOKEN>
```

### 3.3 可选：环境变量 token

```bash
export JADX_AUTH_TOKEN=<ONE_TIME_TOKEN>
uv run jadx_mcp_server.py --jadx-url http://127.0.0.1:18650
```

---

## 4. 隧道建议（避免和本地冲突）

推荐把云端映射到 `18650`，不要复用本地 `8650`：

```bash
ssh -f -N -L 18650:127.0.0.1:8650 user@<cloud-ip>
```

说明：

- `ssh -N -L ...` 前台看起来“卡住”是正常的
- 使用 `-f` 可后台运行

检查隧道是否在监听：

```bash
lsof -nP -iTCP:18650 -sTCP:LISTEN
```

---

## 5. Claude Code MCP 配置（避免冲突）

推荐同时保留两个 server 名称：

- `jadx-local`（你本地 APK 分析）
- `jadx-cloud`（云端 APK 分析）

示例：

```json
{
  "mcpServers": {
    "jadx-cloud": {
      "type": "stdio",
      "command": "uv",
      "args": [
        "--directory",
        "<workspace-root>/jadx-mcp-server",
        "run",
        "jadx_mcp_server.py",
        "--jadx-url",
        "http://127.0.0.1:18650",
        "--token-file",
        "~/.secrets/jadx_cloud.token"
      ],
      "env": {}
    }
  }
}
```

---

## 6. 关键坑：`/mcp` 显示 No MCP servers configured

这个问题在本次排障中已复现，根因通常是 **scope 不一致**。

### 6.1 在当前项目目录注册 local scope（最稳）

进入你正在工作的目录后执行：

```bash
claude mcp add -s local jadx-cloud -- \
  uv --directory <workspace-root>/jadx-mcp-server \
  run jadx_mcp_server.py \
  --jadx-url http://127.0.0.1:18650 \
  --token-file ~/.secrets/jadx_cloud.token
```

验证：

```bash
claude mcp list
claude mcp get jadx-cloud
```

### 6.2 删除旧配置（推荐）

```bash
claude mcp remove -s local jadx-mcp
```

说明：仅设置 `disabled=true` 有时不能达到你预期，直接移除最干净。

### 6.3 如果你改了 server 名称

比如从 `jadx-mcp` 改到 `jadx-cloud`，记得同步更新项目白名单：

- 文件：`<project>/.claude/settings.local.json`
- 把 `mcp__jadx-mcp__*` 改为 `mcp__jadx-cloud__*`

否则工具调用会被权限策略拦截。

---

## 7. 常见问题

### Q1. 启动日志里 health check 401

remote mode 必须带 token；未带 token 是预期失败。

### Q2. 明明配置了还是连接不上

按顺序检查：

1. `claude mcp list` 在当前目录是否能看到 `jadx-cloud`
2. `lsof` 看本地隧道端口是否监听
3. token 文件是否存在、是否最新
4. 云端插件是否仍在运行

### Q3. 怎样区分本地与云端 MCP

用不同 server 名称 + 不同本地端口：

- `jadx-local` -> `127.0.0.1:8650`
- `jadx-cloud` -> `127.0.0.1:18650`

---

## 8. 快速自检命令

```bash
# 1) MCP 配置是否生效
claude mcp list

# 2) 服务详情
claude mcp get jadx-cloud

# 3) 直接跑桥接服务
uv run jadx_mcp_server.py --jadx-url http://127.0.0.1:18650 --token-file ~/.secrets/jadx_cloud.token
```
