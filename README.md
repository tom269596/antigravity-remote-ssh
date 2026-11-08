# Remote SSH Linux Ops 插件

基于 **Antigravity Plugin 规范** 与 **`devnullvoid/mcp-ssh-session`** 构建的专业远程 Linux 运维与排障插件。

支持安全连接远程 Linux 服务器、执行 Shell 命令、监控系统资源、管理服务与容器、非交互式读写配置文件，并严格遵循全流程配置变更安全生命周期。

---

## 🌟 核心特性

* **🚀 持久会话与防挂起机制（Persistent Session & Smart Execution）**：
  * 基于长连接池设计，跨多轮对话复用连接，避免重复频繁建立握手。
  * 具备智能超时后台异步切换与命令中断机制，彻底避免因长耗时命令导致 Agent 挂死。
* **🔑 灵活认证与多主机管理**：
  * **私钥登录**：原生集成并解析本地 `~/.ssh/config`，通过主机别名一键连接。
  * **临时密码**：支持对话中临时指定 `username`、`password` 与 `port` 动态连接。
  * **Sudo 权限无缝注入**：支持 `sudo_password` 自动处理提权提示，无需人工介入终端。
* **🛡️ 严格的安全红线与防破坏护栏**：
  * 内置 `rules/AGENTS.md`：严禁自主执行 `rm -rf`、`mkfs`、`reboot`、`shutdown`、`iptables -F` 等高危破坏性命令，必须明确提示风险并经用户二次确认。
  * 严禁调用交互式 TUI 命令（如 `vim`、`nano`、裸 `top`、裸 `less`），杜绝终端挂起。
* **📝 配置文件变更安全闭环（SOP）**：
  * **先备份**：修改前强制备份 `*.bak.$(date +%Y%m%d%H%M%S)`。
  * **非交互修改**：通过 SFTP 原生 `read_file` / `write_file` 内存修改并写回，零字符转义风险。
  * **配置自检**：修改后自动执行服务自检（如 `nginx -t`, `sshd -t`）。
  * **确认与清理**：自检通过后**主动询问用户确认**，得到用户明确授权后才执行备份文件清理。

---

## 📁 插件结构

```text
remote-ssh/
├── plugin.json                 # 插件元信息与 Starter Prompts
├── README.md                   # 插件使用手册
├── mcp_config.json             # 声明底层的 SSH MCP Server (基于 uvx)
├── assets/
│   └── logo.svg                # 插件高对比度矢量图标
├── rules/
│   └── AGENTS.md               # 安全红线、非交互规范与备份清理闭环
├── skills/
│   └── remote-ops/
│       └── SKILL.md            # 系统巡检、容器管理、日志提取与排障指南
└── agents/
    └── server-admin.md         # 专属 Linux 运维与排障专家 Subagent
```

---

## 🛠️ 安装与启用

### 1. 复制到 Antigravity 插件目录
将本插件目录放置到 `~/.gemini/config/plugins/remote-ssh/`：
* Windows: `C:\Users\<用户名>\.gemini\config\plugins\remote-ssh\`
* Linux/macOS: `~/.gemini/config/plugins/remote-ssh/`

### 2. 重启 Antigravity
重启 Antigravity IDE 或重新加载语言服务，系统将自动加载并激活该插件。

---

## ⚙️ 认证与主机配置指南

### 方式 1：使用 `~/.ssh/config`（强烈推荐，私钥免密）
在本地用户的 `~/.ssh/config` 文件中配置您的服务器别名与密钥：
```ssh-config
Host web-prod
    HostName 192.168.1.10
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host db-stage
    HostName 10.0.0.5
    User root
    IdentityFile ~/.ssh/id_rsa
```
在与 AI 对话时，只需直接提及别名：
> *“查看 `web-prod` 上的内存和磁盘使用率”*

### 方式 2：临时密码登录
如果是一台新测试机，无需配置任何本地文件，直接在对话中告知 AI：
> *“连接 192.168.1.50，用户 root，密码 xxx，帮我看看为什么 8080 端口无法访问”*

### 方式 3：隐藏凭据别名覆盖（环境变量配置）
如果您既想用别名，又不想在对话中明文透露密码，可以在插件的 `mcp_config.json` 的 `env` 字段中配置环境变量映射：
```json
{
  "mcpServers": {
    "remote-ssh": {
      "command": "uvx",
      "args": ["mcp-ssh-session"],
      "env": {
        "OVRD_mytest_HOST": "192.168.1.88",
        "OVRD_mytest_USER": "admin",
        "OVRD_mytest_PASSWORD": "your_secure_password"
      }
    }
  }
}
```
此时只要传入 `host: "mytest"`，底层会自动安全注入对应配置。

---

## 💡 典型使用示例

### 1. 系统负载与健康巡检
> *“帮我全面巡检一下 `web-prod` 的负载，看看有没有异常占用”*

AI 将自动聚合执行负载速查命令：
* CPU 与 Load Average (`uptime`)
* 内存使用 (`free -h`)
* 磁盘与根分区 (`df -h`)
* CPU / 内存占用前 5 的进程快照

### 2. 服务故障排查与日志诊断
> *“`web-prod` 上的 Nginx 报 502 错误，帮我排查一下具体原因”*

AI 将有序执行：
1. 检查 Nginx 运行状态：`systemctl status nginx --no-pager`
2. 提取 Nginx 最近错误日志：`tail -n 50 /var/log/nginx/error.log`
3. 检查后端上游（Upstream）监听端口存活情况：`ss -tulpn`
4. 综合定位根因并向您汇报。

### 3. 配置文件修改安全闭环（SOP）
> *“帮我将 `web-prod` 上的 Nginx 配置文件 `/etc/nginx/conf.d/api.conf` 中的 client_max_body_size 改为 50m”*

AI 的标准执行流程：
1. **备份**：`cp /etc/nginx/conf.d/api.conf /etc/nginx/conf.d/api.conf.bak.20261107...`
2. **修改**：调用 `read_file` 读取内容，精准修改为 `50m`，调用 `write_file` 安全写回。
3. **自检**：执行 `nginx -t` 验证配置文件语法。
4. **征询**：向您汇报修改结果与备份文件路径，并询问：“*已自检通过。请您确认配置是否符合预期，是否允许我清理上述临时备份文件？*”
5. **授权清理**：得到您的明确确认后，执行清理操作。

---

## 🔒 常见问题与说明

1. **依赖环境要求**：
   * 本机需具备 Python 与 `uvx`（执行 `uvx --version` 确认）。
   * 本机网络支持拉取 `mcp-ssh-session` 包。
2. **如何中断长时间运行的命令？**
   * 底层 MCP 提供了 `interrupt_command_by_id` 工具，当命令超时或陷入死循环时，AI 会自动调用中断。
