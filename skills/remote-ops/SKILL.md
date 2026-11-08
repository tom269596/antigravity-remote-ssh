---
name: remote-ops
description: Linux 远程服务器运维排障与管理技能。指导 Agent 如何高效收集系统运行指标（CPU/内存/磁盘/端口）、排查服务与容器异常、安全分析日志、通过 SFTP 读写和调试远程文件，以及严格遵循“备份-修改-验证-征询用户-清理”的标准配置变更流程。
---

# Remote Ops Linux 运维与排障指南

本技能指导 Agent 如何通过 `remote-ssh` 插件连接远程 Linux 服务器，执行系统诊断、日常巡检、容器与服务管理、非交互式文件读写及排障运维。

---

## 1. 连接模式与凭据传递

底层通过 `mcp-ssh-session` 提供持久会话连接池，支持以下两种连接方式：

### 模式 A：SSH Config 别名（首选，私钥连接）
若本地 `~/.ssh/config` 中已配置主机（例如 `Host web-prod`），直接使用别名调用：
```json
{
  "host": "web-prod",
  "command": "uptime"
}
```

### 模式 B：临时主机与密码连接
对于未预先配置的主机，调用工具时传入连接凭据：
```json
{
  "host": "192.168.1.100",
  "username": "ubuntu",
  "password": "your_password",
  "port": 22,
  "command": "df -h"
}
```
* **Sudo 提权支持**：当需要执行需要 root 权限的命令且该用户需要输入 sudo 密码时，传入 `sudo_password`：
```json
{
  "host": "web-prod",
  "sudo_password": "your_sudo_password",
  "command": "systemctl restart nginx"
}
```
* **长连接特性**：连接成功后会保持会话连接池，后续同一主机的命令执行无需频繁重新握手。

---

## 2. 系统健康速查与排障工作流

在需要查看远程服务器相关信息时，按需调用以下标准组合命令，避免零散请求：

### 2.1 整体概览（一步到位）
通过一条组合命令快速收集服务器核心负载概览：
```bash
echo "=== UPTIME & LOAD ===" && uptime && \
echo "=== MEMORY ===" && free -h && \
echo "=== DISK ===" && df -h / /var /data 2>/dev/null || df -h && \
echo "=== TOP CPU PROCESSES ===" && ps aux --sort=-%cpu | head -n 6 && \
echo "=== TOP MEM PROCESSES ===" && ps aux --sort=-%mem | head -n 6
```

### 2.2 核心指标深度排查
* **CPU 排查**：
  * 使用单次非交互模式：`top -b -n 1 | head -n 25`
  * 查看多核分布：`mpstat -P ALL 1 1 2>/dev/null || lscpu`
* **内存排查**：
  * 查看缓存与 Swap：`free -m`
  * 内存虚存统计：`vmstat 1 2`
* **磁盘与 I/O 排查**：
  * 磁盘使用率：`df -Th`
  * 查找大目录：`du -ahx /var/log 2>/dev/null | sort -rh | head -n 10`
  * 磁盘 I/O 压力：`iostat -xz 1 2 2>/dev/null || vmstat 1 3`
* **网络与监听端口**：
  * 监听端口列表：`ss -tulpn` 或 `netstat -tulpn`
  * 连接状态汇总：`ss -s`
  * 外部连通性探测：`curl -Iv --connect-timeout 5 https://api.example.com`

---

## 3. 服务与容器排障

### 3.1 Systemd 服务排查
* 查看异常失败的服务：`systemctl list-units --failed --no-pager`
* 查看特定服务状态与简要日志：`systemctl status <service_name> -l --no-pager`
* 服务控制：`systemctl restart <service_name>` 或 `systemctl reload <service_name>`

### 3.2 Docker 容器排查
* 容器运行状态：`docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Image}}"`
* 容器资源实时占用（单次快照）：`docker stats --no-stream`
* 容器日志最后 100 行：`docker logs --tail 100 <container_name_or_id>`
* 容器异常退出排查：`docker inspect <container_name> --format '{{json .State}}'`

---

## 4. 日志提取规范（防截断与防爆流）

**严禁**直接对大型日志文件执行无限制的 `cat` 操作，防止日志爆炸冲垮上下文。

* **使用 journalctl**：
  * 查看指定服务最近 100 行日志：`journalctl -u <service> -n 100 --no-pager`
  * 带有时间范围查询：`journalctl -u <service> --since "1 hour ago" --no-pager`
  * 仅看错误级别及以上日志：`journalctl -u <service> -p err..emerg -n 50 --no-pager`
* **普通文件日志**：
  * 尾部读取：`tail -n 100 /var/log/nginx/error.log`
  * 错误关键词筛选：`grep -E "ERROR|CRITICAL|FATAL" /var/log/app.log | tail -n 50`

---

## 5. 远程文件编辑与生命周期规范（备份 -> 修改 -> 自检 -> 征询用户 -> 清理）

当需要修改远程配置文件（如 Nginx、MySQL、Redis、应用配置、环境变量）时，**必须**严格遵循五步作业流程：

```
Step 1: 建立备份 (cp <file> <file>.bak.$(date +%Y%m%d%H%M%S))
Step 2: 读取源文件 (read_file)并在本地规划精确修改
Step 3: 写回目标文件 (write_file 或 sed / Heredoc)
Step 4: 运行服务自检与状态验证 (如 nginx -t, sshd -t, systemctl status)
Step 5: 主动征询用户确认 -> 用户同意后执行清理 rm <backup_file>
```

### 示例操作序列：
1. **备份**：
   ```bash
   cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak.$(date +%Y%m%d%H%M%S)
   ```
2. **读取与修改**：
   * 调用 `read_file(host="web-prod", remote_path="/etc/nginx/nginx.conf")`
   * 修改本地内容
   * 调用 `write_file(host="web-prod", remote_path="/etc/nginx/nginx.conf", content="...")`
3. **自检**：
   ```bash
   nginx -t
   ```
4. **向用户汇报并征询确认**：
   * 向用户说明已完成的改动、`nginx -t` 的成功输出。
   * 给出备份文件路径：`/etc/nginx/nginx.conf.bak.20261107153022`。
   * 发起询问：“*当前配置已成功验证通过。请确认配置无误，是否允许我删除本次的临时备份文件？*”
5. **依据用户授权处理备份**：
   * 仅在用户回复确认清理后，执行 `rm /etc/nginx/nginx.conf.bak.20261107153022`。

---

## 6. 多服务器协同与对比

当排查涉及多台服务器时（如集群环境、主从数据库、前后端节点）：
1. 明确标注当前命令作用的 `host`。
2. 可平行提取各节点的配置或状态指标，并以 Markdown 表格形式直观对比各节点差异。
3. 关键节点配置变更建议分批次执行，先在单台节点验证通过并征询确认后，再推送到下一台节点。

---

## 7. 用户沟通与信息回显规范（双层呈现：精炼摘要 + 折叠原生数据）

在执行远程命令后，Agent 既不能将海量原生控制台输出无脑倾泻到聊天窗口，也不能过度抽象导致用户缺失关键排障证据。必须严格执行“**双层呈现规范**”：

### 7.1 双层呈现结构标准
1. **第一层：业务结论与指标摘要（用户主视区）**
   - 给出明确结论（如：“配置自检通过”、“发现 2 个失败的服务”、“磁盘根目录使用率达到 92%”）。
   - 提取关键指标（以 Markdown 表格、加粗清单形式展示）。
   - 异常风险警示：使用 GitHub-style Alert 块醒目高亮（如 `> [!WARNING]` / `> [!CAUTION]`）。

2. **第二层：可折叠的原始控制台数据（证据核实区）**
   - 凡涉及以下场景，**必须**使用 HTML `<details><summary>` 标签包裹原始输出，供用户随时点击核实：
     - ① 服务自检命令输出（如 `nginx -t`, `sshd -t`）。
     - ② 核心错误/警告日志（如 `journalctl` 报错片段、Nginx/Docker error log）。
     - ③ 系统关键配置修改前后的对比（Diff 片段）。
     - ④ 包含具体参数的复杂命令执行原始返回。

### 7.2 回显模板示例

#### 示例 A：配置文件自检与修改汇报
```markdown
✅ **Nginx 配置文件修改自检通过**
* 目标主机：`web-prod`
* 涉及文件：`/etc/nginx/conf.d/api.conf`
* 关键变更：将 `client_max_body_size` 调整为 `50m`
* 临时备份路径：`/etc/nginx/conf.d/api.conf.bak.20261107160530`

<details>
<summary><b>🔍 点击展开 nginx -t 原始控制台输出 (exit_code: 0)</b></summary>

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
</details>

*当前修改已自检通过。请您确认服务运行是否正常，是否允许我清理上述临时备份文件？*
```

#### 示例 B：系统巡检与告警汇报
```markdown
### 🖥️ 主机 `web-prod` 状态巡检结果

| 监控项 | 当前值 | 状态 | 说明 |
| :--- | :--- | :--- | :--- |
| **系统负载** | `0.45, 0.32, 0.28` | 🟢 正常 | 4 核 CPU，负载较低 |
| **内存使用** | 5.2G / 16G (32%) | 🟢 正常 | Swap 使用 0B |
| **磁盘使用 (/)** | 92% (85G/95G) | 🔴 **告警** | 剩余空间不足 10G |

> [!WARNING]
> 根目录 `/` 磁盘空间占用已达 **92%**，存在空间耗尽风险！建议重点清理 `/var/log` 或清理无用 Docker 镜像。

<details>
<summary><b>🔍 点击展开 df -h 与 top 进程原始控制台输出</b></summary>

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        95G   85G  9.8G  92% /
tmpfs           7.8G     0  7.8G   0% /dev/shm

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
www-data  1423  4.5 12.0 852100 196800 ?       S    10:20   2:15 node /app/server.js
root       890  1.2  3.4 345000  55600 ?       Ssl  09:00   0:45 /usr/bin/dockerd
```
</details>
```

