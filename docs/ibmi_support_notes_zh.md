# IBM i 支持说明（中文研究笔记）

> 本文是针对本仓库（ansible-automation-for-lmt，playbooks 版本 9.2.44.0）的代码走读笔记，
> 不属于 IBM 随包文档。结论以仓库代码为准，标注「未验证」的部分请自行核实。

---

## 1. 支持现状

官方文档明确声明 **不支持 IBM i**：

- `README.md:20`："They are not supported on IBM i."
- `docs/doc_playbooks_list.md:3`："...except for **IBM i**."

但代码中存在完整的 IBM i 分支，统一以 `ansible_facts['os_family'] == 'OS400'` 判断：

| Playbook | IBM i 分支 | 对应任务文件 |
|---|---|---|
| `lmt_install_or_upgrade_scanner.yml` | ✅ | `install_scanner_ibmi.yml` → `setup_scanner_ibmi.yml` / `upgrade_scanner_ibmi.yml` |
| `lmt_uninstall_scanner.yml` | ✅ | `uninstall_scanner_ibmi.yml` |
| `lmt_collect_status.yml` | ✅ | `check_endpoint_status_ibmi.yml` |
| `lmt_collect_results.yml` | ✅ | `collect_scan_results_ibmi.yml` |
| `lmt_collect_troubleshooting_data.yml` | ✅ | `collect_troubleshooting_data_ibmi.yml` |
| `lmt_reconfigure_scanner.yml` | ❌ | 无 IBM i 分支，对 IBM i 节点不做任何事 |
| `lmt_run_scan_now.yml`（本地新增） | ✅ | `run_scan_now_ibmi.yml`，手动执行软件扫描并打包，见第 4.3 节 |

结论：**功能可用，但官方不提供支持保证**。

> 第 4、5 节的调度信息已在一台 IBM i 7.5 实机上（扫描器 9.2.44.0，`LMT-DisconnectedScanner-ibm_i-9.2.44.0-20260622-1708`）
> 于 2026-10-16 实测确认。

---

## 2. 安装位置

- 默认目录：**`/cit`**（IFS 路径），定义在 `roles/ilmt_scanner/vars/main.yml:9`：
  ```yaml
  lmt_scanner_path_ibmi: "/cit"
  ```
- ⚠️ 该变量位于 role 的 `vars/`（而非 `defaults/`），**优先级高于 inventory 变量**，
  在 `lmt_inventory.yml` 中修改不会生效。如需更换目录：
  - 命令行 `-e lmt_scanner_path_ibmi=/xxx`（extra vars 优先级最高），或
  - 直接修改 `roles/ilmt_scanner/vars/main.yml`。
- 关键文件：

| 路径 | 用途 |
|---|---|
| `/cit/config/endpoint_id.txt` | 端点 ID，用于判断“是否已安装” |
| `/cit/config/sw_config.xml`、`computer_properties.yml` | 升级时备份/还原的配置 |
| `/cit/config/successful_setup.info` | setup 成功标记（9.2.26.0+） |
| `/cit/iso-swid/ibm.com_IBM_License_Metric_Tool-<版本>.swidtag` | 用于识别扫描器版本 |
| `/cit/<12位数字>-*.zip` | 扫描结果包（位于根目录，非 `output/`；UNIX 为 `output/*.tar.gz`） |
| `/cit/support_data/` | 排错数据脚本输出 |

- 最低磁盘空间：120 MB（`vars/main.yml:19`，`lmt_scanner_ibmi_scanner_minimum_disk_space`）。

---

## 3. 安装 / 升级流程（`install_scanner_ibmi.yml`）

1. **兼容处理**：旧版本（< 9.2.26.0）若只有 `computer.yml`，从中提取 `endpointID` 生成 `endpoint_id.txt`。
2. **前置检查**
   - `/cit` 已存在、非空且未安装扫描器 → 失败，提示先清理（例如运行卸载 playbook）。
   - `df -m` 检查安装根目录剩余空间是否 ≥ 120 MB。
3. **版本判断**：仅当未安装，或控制节点缓存版本 > 已安装版本时才执行后续步骤。
   缓存由同一 playbook 第一段 play 从 `lmt_server` 刷新到 `./lmt_file_storage/` 下。
4. **备份配置**（已安装时）：`endpoint_id.txt`、`sw_config.xml`、`computer_properties.yml` → `/cit/config_backup/`。
5. **上传并解包**：上传 `LMT-DisconnectedScanner-ibm_i-<版本>-<构建日期>.zip`，
   用 **`jar -xvf`** 解压（依赖 Java），之后删除 zip。
6. **还原配置**：从 `config_backup/` 拷回 `config/`。
7. **执行脚本**
   - 全新安装：`qsh setup_all.sh`（超时 `lmt_scanner_setup_timeout`，默认 300 秒），
     随后以 `async: 518400, poll: 0` 后台启动首次扫描 `qsh run_sw_and_pack.sh`，不等待完成。
   - 升级：`qsh update.sh`。
8. **收尾**：`/cit` 递归改属主为 `ansible_user`；删除 `config_backup/`；
   以 `config/endpoint_id.txt` 是否存在作为成功判定。

卸载（`uninstall_scanner_ibmi.yml`）：执行 `qsh uninstall.sh`，然后 `rm -rf /cit`。

---

## 4. 定时运行

定时分为两层，分别由不同的组件负责。

### 4.1 端点上的扫描定时：只有硬件扫描，且周期写死

IBM i 上**没有 cron**（也没有 `crontab` 命令），扫描器使用的是 IBM i 的**作业调度表**（job schedule）。

与 UNIX / Windows 的区别：

- UNIX / Windows 的 `setup_scanner_*.yml` 会把 inventory 参数写入 `config/setup_config.ini`
  （`SW_SCAN_SCHEDULE_ENABLED`、`SW_SCAN_FREQUENCY` 等），再由扫描器脚本落地成 cron / 计划任务。
- IBM i 版扫描器**没有 `setup_config.ini`**（`/cit/config/` 下不存在该文件），
  `bin/tools.sh` 中也没有任何调度参数。`setup_scanner_ibmi.yml` 只执行 `qsh setup_all.sh`。
- 因此 inventory 中的 `lmt_scanner_software_scans_*`、`lmt_scanner_daily_pack_results_enabled`
  等参数对 IBM i **全部无效**，`lmt_reconfigure_scanner.yml` 也不适用。

`setup_all.sh` 调用 `/cit/bin/schedule_scans.sh`，其核心逻辑：

```sh
SYSTEM "RMVJOBSCDE JOB(LMT_HW) ENTRYNBR(*ALL)"          # 先删除全部旧条目
SYSTEM "CRTJOBD JOBD(QSYS/QLMTJOBD) LOG(4 00 *NOLIST) TEXT('ILMT Disconnected Scans')"
# Scheduling HW Scan every 30 minutes
while [[ $HOUR -lt 24 ]] ; do
  ADDJOBSCDE JOB(LMT_HW) CMD(QSH CMD('/cit/run_hw.sh')) SCDDATE(*NONE) SCDDAY(*ALL) +
             SCDTIME('${HJB}00') FRQ(*WEEKLY) RCYACN(*NOSBM) USER(QSECOFR) JOBD(QLMTJOBD)
  ADDJOBSCDE JOB(LMT_HW) ... SCDTIME('${HJB}30') ...
done
```

实机查询 `QSYS2.SCHEDULED_JOB_INFO` 的结果：

| 作业名 | 条目数 | 状态 | 频率 | 日期 | 时间 | 命令 |
|---|---|---|---|---|---|---|
| `LMT_HW` | 48 | SCHEDULED | `*WEEKLY` | `*ALL` | 00:00 到 23:30，每 30 分钟 | `QSH CMD('/cit/run_hw.sh')` |

要点：

- **只调度了硬件（容量）扫描**，48 条 × 每天 = 每 30 分钟一次。
- **软件扫描和打包没有任何定时任务。** `run_sw_and_pack.sh` 只在安装时由 Ansible 后台触发一次。
  之后除非手动执行，否则**不会再生成新的结果包**，ILMT 服务器上的软件清单会停留在安装那天。
- 调度作业以 `USER(QSECOFR)` 身份运行，提交者是安装时的 `ansible_user`。
- 结果包由 `pack_results.sh` 写入 `PACKAGE_OUTPUT_DIR="/cit"`（在 `bin/tools.sh` 中写死），
  文件名形如 `/cit/202610160553-7892A61-3.zip`。`/cit/output/` 是工作目录，不放结果包。
- 实测耗时：软件扫描约 28 分钟，打包约 1 分钟。

### 4.2 周期是否可配置

**没有官方配置项。** 可选的做法：

| 目标 | 做法 | 注意 |
|---|---|---|
| 修改硬件扫描周期 | ① 修改 `/cit/bin/schedule_scans.sh` 后重新执行；② 用 `RMVJOBSCDE` / `ADDJOBSCDE` 直接修改调度表 | 都不是官方支持的方式。①升级时会被安装包覆盖；②在重新执行 `setup_all.sh` 时会被 `RMVJOBSCDE JOB(LMT_HW) ENTRYNBR(*ALL)` 清掉 |
| 定期执行软件扫描和打包 | 手动执行 `lmt_run_scan_now.yml`（第 4.3 节），或在端点上自行添加调度条目（见下方示例） | 用独立的作业名（如 `LMT_SW`），不会被 `schedule_scans.sh` 删掉；但 `uninstall.sh` 也不会清理它，卸载时需手动执行 `RMVJOBSCDE JOB(LMT_SW) ENTRYNBR(*ALL)` |

每周日 02:00 执行软件扫描和打包的示例：

```
ADDJOBSCDE JOB(LMT_SW) CMD(QSH CMD('/cit/run_sw_and_pack.sh')) SCDDATE(*NONE) +
           SCDDAY(*SUN) SCDTIME('020000') FRQ(*WEEKLY) RCYACN(*NOSBM) +
           USER(QSECOFR) JOBD(QLMTJOBD)
```

### 4.3 手动执行软件扫描：`lmt_run_scan_now.yml`

该 playbook 在所有平台上触发一次"软件扫描 + 打包"，路径统一引用 role 变量
（IBM i 为 `lmt_scanner_path_ibmi`，即 `/cit`），不再写死：

| 平台 | 任务文件 | 执行的命令 |
|---|---|---|
| IBM i | `run_scan_now_ibmi.yml` | 在 `/cit` 下执行 `qsh run_sw_and_pack.sh`，不提权 |
| UNIX/Linux | `run_scan_now_unix.yml` | 在 `<lmt_scanner_path_unix>/bin` 下执行 `./run_sw_and_pack_CRON.sh`，`become: true` |
| Windows | `run_scan_now_windows.yml` | 在 `<lmt_scanner_path_windows>` 下执行 `run_sw_and_pack.bat`，`runas system` |

执行前会先运行 `verify_scanner_installation`，未安装扫描器的节点直接失败。

参数（在 `roles/ilmt_scanner/defaults/main.yml` 中定义，可在 inventory 中或用 `-e` 覆盖）：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `lmt_scan_now_wait` | `true` | `true`：等待扫描完成（IBM i 约 30 分钟），并列出 `/cit` 下的结果包；`false`：后台启动后立即返回 |
| `lmt_scan_now_timeout` | `7200` | 扫描超时（秒） |

用法：

```sh
# 只对 IBM i 节点执行，并等待完成
ansible-playbook lmt_run_scan_now.yml -i lmt_inventory.yml -l localhost,ibmi1

# 后台触发，不等待
ansible-playbook lmt_run_scan_now.yml -i lmt_inventory.yml -l localhost,ibmi1 -e lmt_scan_now_wait=false

# 扫描完成后收集结果
ansible-playbook lmt_collect_results.yml -i lmt_inventory.yml -l lmt_server,localhost,ibmi1
```

实测（2026-10-16，等待模式）：`ok=5 changed=1 failed=0`，扫描加打包约 9 分钟，
生成 `/cit/202610160704-7892A61-3.zip`。首次扫描较慢（约 28 分钟），后续扫描明显更快。

注意：用 `-e lmt_scan_now_wait=false` 传入的是字符串，任务中已用 `| bool` 转换，可以正常生效。

也可以在控制节点上用 cron 定时执行这个 playbook，作为 IBM i 软件扫描的定时方案。
这样就不需要在端点上维护 `LMT_SW` 调度条目：

```
0 1 * * 0  ansible-playbook <dir>/lmt_run_scan_now.yml -i <dir>/lmt_inventory.yml -l localhost,ibmi
30 2 * * * ansible-playbook <dir>/lmt_collect_results.yml -i <dir>/lmt_inventory.yml
```

### 4.4 控制节点上的结果收集定时 —— 与其他平台相同

在控制节点用 cron 定时运行 `lmt_collect_results.yml`，它会 fetch `/cit/*.zip` 回控制节点
并删除端点上的原文件（参见 `docs/doc_schedule_collection.md`）：

```
30 2 * * * ansible-playbook <dir>/lmt_collect_results.yml -i <dir>/lmt_inventory.yml
```

---

## 5. 查看端点上的调度条目（WRKJOBSCDE）

`WRKJOBSCDE`（Work with Job Schedule Entries）是查看作业调度表的 CL 命令。

### 5.1 5250 终端交互

```
WRKJOBSCDE
WRKJOBSCDE JOB(LMT*)      /* 按作业名通配过滤 */
```

条目前选项：`5` 显示详情（可看到 `CMD` 参数，据此识别扫描器条目）、`2` 修改、
`3` 暂挂、`6` 释放、`4` 删除。

### 5.2 SQL（IBM i 7.2+，推荐）

按作业名汇总（实测可用）：

```sql
SELECT SCHEDULED_JOB_NAME, COUNT(*) AS ENTRIES, MIN(STATUS) AS STATUS,
       MIN(FREQUENCY) AS FRQ, MIN(SCHEDULED_DAYS) AS DAYS,
       MIN(SCHEDULED_TIME) AS FIRST_TIME, MAX(SCHEDULED_TIME) AS LAST_TIME,
       MIN(SCHEDULED_BY) AS BY_USER, MIN(COMMAND_STRING) AS CMD
  FROM QSYS2.SCHEDULED_JOB_INFO
 WHERE SCHEDULED_JOB_NAME LIKE 'LMT%'
 GROUP BY SCHEDULED_JOB_NAME
```

注意：该视图中没有 `JOB_DESCRIPTION` 列，引用会报 SQLSTATE 42703。

### 5.3 SSH / QSH 命令行

`WRKJOBSCDE` 是交互式命令，SSH 下无法显示界面。SQL 里的单引号和 `%` 嵌在多层引号中
很容易转义出错（实测报 SQLSTATE 42601），**推荐把 SQL 写入文件再用 `db2 -f` 执行**：

```sh
echo "SELECT SCHEDULED_JOB_NAME, SCHEDULED_TIME, STATUS FROM QSYS2.SCHEDULED_JOB_INFO WHERE SCHEDULED_JOB_NAME LIKE 'LMT%'" > /tmp/lmt.sql
qsh -c "db2 -f /tmp/lmt.sql"
rm -f /tmp/lmt.sql

system "WRKJOBSCDE OUTPUT(*PRINT)"   # 输出到假脱机文件，用 WRKSPLF 查看
```

### 5.4 Ansible ad-hoc

```sh
ansible <ibmi_host> -i lmt_inventory.yml -m shell -a \
  "echo \"SELECT SCHEDULED_JOB_NAME, SCHEDULED_TIME, STATUS FROM QSYS2.SCHEDULED_JOB_INFO WHERE SCHEDULED_JOB_NAME LIKE 'LMT%'\" > /tmp/lmt.sql && qsh -c 'db2 -f /tmp/lmt.sql'; rm -f /tmp/lmt.sql"
```

注意：

- 查看其他用户提交的条目通常需要 `*JOBCTL` 特殊权限。
- 安装后应能看到 48 条 `LMT_HW` 条目。如果数量不对，查看 `/cit/logs/schedule_scans_std.log`。

---

## 6. SSH 连接用户的权限要求

> 本仓库没有 IBM i 专门的权限文档，以下由代码行为与 IBM i 平台通用要求推导。

### 6.1 关键事实：IBM i 分支不提权

UNIX / Windows 分支均有 `become: true`，IBM i 分支没有（例如 `lmt_install_or_upgrade_scanner.yml:51-56`）。
所有操作都直接以 `ansible_user` 执行，**该用户本身必须具备足够权限**。

### 6.2 连接前提

| 项 | 要求 |
|---|---|
| SSH 服务 | `STRTCPSVR SERVER(*SSHD)` 已启动，用户可 SSH 登录 |
| Python | 通过 IBM i 开源包（yum）安装 Python 3，建议显式设置 `ansible_python_interpreter: /QOpenSys/pkgs/bin/python3` |
| Home 目录 | 用户配置文件的 `HOMEDIR` 在 IFS 中真实存在（Ansible 需写 `~/.ansible/tmp`） |
| Java | 安装 5770JV1，且 `jar` 在 PASE PATH 中（用于解包安装包） |
| `qsh` | 安装、升级、卸载、排错数据收集均通过 `qsh xxx.sh` 执行 |
| 控制节点 collection | `community.general`（排错数据收集使用 `community.general.archive`）。本仓库未使用 `ibm.power_ibmi` |

### 6.3 各操作的权限需求

| 操作 | 涉及动作 | 权限需求 |
|---|---|---|
| 安装 / 升级 | 在 `/` 下创建 `/cit`、上传、解包、执行 `setup_all.sh` / `update.sh`、`chown` | 对 `/` 有 `*W` + `*X`；对 `/cit` 有 `*RWX`；另加脚本内部所需权限 |
| 软件扫描 | `run_sw_and_pack.sh` 读取系统产品与对象信息 | **`*ALLOBJ`（IBM ILMT 文档要求）** |
| 注册调度 | `schedule_scans.sh` 执行 `CRTJOBD JOBD(QSYS/QLMTJOBD)`，并以 `USER(QSECOFR)` 执行 `ADDJOBSCDE` | 要在 QSYS 中创建对象，并对 QSECOFR 用户配置文件有 `*USE` 权限，`*ALLOBJ` 都已覆盖 |
| 收集结果 | 在 `/cit` 下 find、fetch、删除 zip | 对 `/cit` 有 `*RWX`（安装时属主已改为 `ansible_user`） |
| 查看状态 | 读取 `/cit/config`、`/cit/iso-swid` | 读权限 |
| 卸载 | `qsh uninstall.sh`、`rm -rf /cit` | 能删除 `/cit` 全部内容，并能撤销扫描器注册的调度条目 |
| 查看调度 | `WRKJOBSCDE` 查看其他用户的条目 | `*JOBCTL`（可选，仅排查时需要；查看本用户提交的条目不需要） |

### 6.4 建议的用户配置

按 IBM ILMT 文档的要求，连接用户需要 **`*ALLOBJ`** 特殊权限：

```
CRTUSRPRF USRPRF(ANSIBLE) PASSWORD(...) USRCLS(*USER) +
          SPCAUT(*ALLOBJ) +
          HOMEDIR('/home/ANSIBLE')
```

已有用户可以直接修改：

```
CHGUSRPRF USRPRF(ANSIBLE) SPCAUT(*ALLOBJ)
```

注意：`SPCAUT` 是整体替换，不是追加。如果该用户原来就有其他特殊权限，要一并列出，
例如 `SPCAUT(*ALLOBJ *JOBCTL)`。

然后在 PASE 中执行 `mkdir -p /home/ANSIBLE`，并建议设置 `LANG` 或 `LOCALE` 以避免 CCSID 编码问题。

说明：

- `*ALLOBJ` 是 IBM i 上范围最大的对象访问权限。安装、扫描、收集结果、卸载所需的 IFS 和对象访问都已覆盖。
- `*JOBCTL` 与 `*ALLOBJ` 互不包含，只在需要查看或管理其他用户的调度条目和作业时才用得到，可按需添加。
- `*SECADM`、`*IOSYSCFG` 不是扫描器所需的权限，不建议授予。
- 不要用 `USRCLS(*SECOFR)` 加 `SPCAUT(*USRCLS)` 的组合，那样会默认授予全部 8 种特殊权限，超出需要。

### 6.5 Inventory 示例

```yaml
ibmi:
  hosts:
    ibmi1:
      ansible_host: <ip>
      ansible_user: ANSIBLE
      ansible_python_interpreter: /QOpenSys/pkgs/bin/python3
```

安装、日常收集和卸载应使用 **同一个用户**：安装后 `/cit` 的属主会被改为该用户，
换用其他用户收集结果可能遇到权限问题。

---

## 7. 已知缺口与改进方向

1. **IBM i 没有软件扫描定时**，这是扫描器本身的设计，不是 playbook 的遗漏。
   IBM i 版扫描器没有 `setup_config.ini`，所以无法像 UNIX 那样通过 `lineinfile` 补齐参数。
   可行的方案：
   - 在控制节点用 cron 定时执行 `lmt_run_scan_now.yml`（已实现，见第 4.3 节）；
   - 或者在 `setup_scanner_ibmi.yml` 中增加可配置的 `LMT_SW` 调度条目（`ADDJOBSCDE`），
     同时在 `uninstall_scanner_ibmi.yml` 中对应地执行 `RMVJOBSCDE JOB(LMT_SW)`。
2. `lmt_reconfigure_scanner.yml` 缺少 IBM i 分支。由于上一条的原因，该 playbook 对 IBM i 也没有可调的配置项。
3. `lmt_scanner_path_ibmi` 放在 `vars/` 中，无法通过 inventory 覆盖，建议移到 `defaults/`。
   不过扫描器的 `bin/tools.sh` 中也把 `PACKAGE_OUTPUT_DIR="/cit"` 写死了，单改变量可能不够，
   需要实测确认。
4. 没有 IBM i 的官方权限说明，也没有调度检查工具。可以增加一个 playbook
   （例如 `lmt_check_ibmi_schedule.yml`），用第 5.3 节的 SQL 统一收集各节点的调度信息。
