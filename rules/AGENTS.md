# Remote SSH Linux Ops 安全与操作规则

当 `remote-ssh` 插件处于启用状态时，所有与远程 Linux 服务器交互的 Agent 必须严格遵守以下安全红线与操作规范：

---

## 1. 配置文件修改与生命周期闭环规则（强制 SOP）

在远程服务器上修改任何配置文件（如 `/etc/nginx/`, `/etc/ssh/`, `/etc/systemd/`, 应用配置文件等）时，必须严格执行以下五步闭环流程，严禁跳步：

1. **预检与备份（Pre-backup）**：
   - 修改文件前，**必须**在同目录下创建带精确时间戳的备份文件：
     ```bash
     cp /path/to/target.conf /path/to/target.conf.bak.$(date +%Y%m%d%H%M%S)
     ```
   - 记录该备份文件的绝对路径。

2. **安全修改（Safe Modification）**：
   - 严禁在远程服务器启动交互式文本编辑器（如 `vim`, `nano`）。
   - 优先使用 MCP 原生 SFTP 工具（`read_file` 配合 `write_file`）在本地内存中精准修改后写回远程。
   - 局部单行配置修改可使用 `sed -i`。
   - 新建脚本或整份新配置文件使用无头 Heredoc（`cat << 'EOF' > ...`）。

3. **语法与服务自检（Verification）**：
   - 修改完成后，必须优先执行对应服务或语法的自检命令，验证配置是否合法：
     - Nginx: `nginx -t`
     - SSHD: `sshd -t`
     - Systemd: `systemctl daemon-reload`
     - Apache: `apachectl configtest`
     - 脚本/程序语法检查（如 `sh -n`, `python -m py_compile`）。

4. **主动征询用户确认（Confirmation Inquiry）**：
   - 自检通过后，**必须在回复中明确告知用户**：
     - 修改的目标文件及具体变更内容。
     - 临时备份文件的完整路径（如 `/etc/nginx/nginx.conf.bak.20261107153022`）。
     - 自检通过的结果。
   - **主动发起确认询问**：“*当前修改已自检通过。请您确认服务运行是否正常，是否允许我清理上述临时备份文件？*”

5. **授权清理或保留（Authorized Cleanup）**：
   - **严禁擅自删除备份文件**！只有在用户明确回复确认无误（如“可以删除”、“确认正常”、“清理备份”等）后，方可执行 `rm` 命令清理备份。
   - 若用户表示需要保留观察，或未给出删除指令，必须保留备份文件以备回滚。

---

## 2. 严禁交互式 TUI 命令（Non-Interactive Execution）

由于 SSH MCP 工具基于非交互式标准输入输出通道运行，任何需要物理终端（TTY）按键交互的程序都会导致命令挂起超时甚至导致会话阻塞。

1. **严禁调用的交互式程序**：
   - 文本编辑器：严禁执行 `vim`, `vi`, `nano`, `emacs`, `pico` 等。
   - 分页器：严禁执行裸 `less`, `more`，查看文件内容请用 `read_file` 或 `cat` / `head` / `tail`。
   - 交互式监控：严禁执行无退出参数的 `top`, `htop`, `iotop`, `iftop`。

2. **必须使用的非交互式等效替代**：
   - 进程监控：使用 `top -b -n 1`（批处理单次模式）或 `ps aux --sort=-%cpu | head -n 15`。
   - 磁盘与 I/O：使用 `iostat -x 1 1` 或 `vmstat 1 2`。
   - 容器统计：使用 `docker stats --no-stream`。
   - 查看日志：使用 `tail -n 100` 或 `journalctl -u <unit> -n 100 --no-pager`。

---

## 3. 高危破坏性命令拦截与二次确认

执行具有不可逆破坏性、可能导致系统宕机、网络断连或关键数据丢失的命令前，必须向用户进行风险提示，并获得显式授权：

1. **高危命令清单**：
   - 递归强制删除：`rm -rf /`, `rm -rf /*`, 递归删除根目录或非临时目录。
   - 磁盘与文件系统格式化：`mkfs`, `fdisk`, `parted`, `dd if=... of=/dev/...`。
   - 系统电源操作：`reboot`, `shutdown`, `poweroff`, `init 0`, `init 6`。
   - 防火墙全局清空：`iptables -F`, `ufw reset`, `systemctl stop firewalld`（可能导致永久失联）。
   - 关键数据库破坏：`DROP DATABASE`, `TRUNCATE TABLE`。

2. **二次确认规范**：
   - 必须向用户呈现：
     1. 目标服务器与执行身份。
     2. 即将执行的具体命令字符串。
     3. 潜在风险说明。
   - 等待用户显式答复（例如“确认执行”）后方可调用。

---

## 4. 凭证脱敏与敏感信息保护

1. **凭证脱敏**：
   - 不得在与用户的直接对话输出中明文回显用户的私钥文本（`BEGIN RSA PRIVATE KEY` 等）、明文密码、数据库连接串或 `.env` 敏感密钥。
2. **连接推荐**：
   - 优先引导用户使用 `~/.ssh/config` 中的主机别名或环境变量别名，减少敏感连接信息在会话中的暴露频率。

---

## 5. 结果同步与用户回显规范（双层呈现原则）

在向用户汇报命令执行与系统状态时，必须遵守“精炼摘要 + 折叠原生数据”双层呈现规范：

1. **第一层（主视区）**：
   - 给出清晰的排障或操作结论，使用 Markdown 表格或清单提取关键性能指标。
   - 发现指标异常（如磁盘 > 85%、CPU/内存高负荷、服务挂死）必须使用 `> [!WARNING]` 或 `> [!CAUTION]` 显式标出。
2. **第二层（可折叠区）**：
   - 关键自检命令输出（如 `nginx -t`）、错误/告警日志片段（`error.log` / `journalctl`）、配置变更 Diff 等，必须使用 HTML `<details><summary>🔍 点击展开原始控制台输出 (exit_code: X)</summary>...</details>` 包裹呈现，确保既不污染主对话流，又保留完整原始证据供用户查验。

