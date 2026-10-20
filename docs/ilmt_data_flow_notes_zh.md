# ILMT 断连模式数据流与运维要点（中文研究笔记）

> 本文是针对本仓库（ansible-automation-for-lmt，playbooks 版本 9.2.44.0）的代码走读笔记，
> 不属于 IBM 随包文档。结论以仓库代码为准，标注「未验证」的部分请自行核实。

---

## 1. 采集数据的整体流向

```
① 被管端点 (Managed Node)：扫描器按自身 cron 产生数据，与 Ansible 无关
   容量扫描 / 软件扫描 / UMS 扫描  →  work/
                                   ↓ pack results
   <scanner>/output/
     Linux/UNIX : 20250915T1200-host-1726400000.tar.gz
     Windows    : 同名 .zip
     IBM i      : /cit/....zip
                 │
                 │ ② ansible fetch（flat=true），成功后删除端点原包
                 ↓
② 控制节点 (Control Node)
   ./lmt_file_storage/scan_result_packages/
                 │
                 │ ③ copy / win_copy 推送，成功后清空控制节点暂存
                 ↓
③ lmt_server 节点
   /opt/ibm/LMT/datasource（Win: ...\ibm\LMT\datasource）
   LMT 服务器自行扫描 datasource → 导入 DB → 报表
```

### 各阶段代码位置

**① 端点侧产生数据**（Ansible 只负责“配置”，不负责触发）

`roles/ilmt_scanner/tasks/setup_scanner_unix.yml` 将参数写入
`<scanner>/config/setup_config.ini`，再执行扫描器自带的 `setup.sh` /
`automation/configure.sh`，由这些脚本落地成端点本地的 cron / 计划任务。

playbook 写入 `setup_config.ini` 的共 11 行：

| 参数 | 作用 |
|---|---|
| `SW_SCAN_SCHEDULE_ENABLED` | 是否启用软件扫描计划 |
| `SW_SCAN_FREQUENCY` | DAILY / WEEKLY（仅当上面为 true 时写入） |
| `SW_SCAN_DAY_OF_WEEK` | 周几（WEEKLY 时有效，空 = 从 playbook 运行时刻起算） |
| `SW_SCAN_LOCAL_TIME` | HH:MM，端点本地时区（空 = 从 playbook 运行时刻起算） |
| `SW_SCAN_CPU_THRESHOLD_PERCENTAGE` | 软件扫描 CPU 占用上限 |
| `UMS_SCAN_ENABLED` | 采集 adoption / 使用度量数据（嵌在软件扫描内） |
| `DAILY_PACK_RESULTS_CREATION_ENABLED` | 是否启用每日打包 |
| `PACKAGE_OUTPUT_DIR` | 包的输出目录，决定第 ② 步去哪里找 |
| `PUBLIC_CLOUD_TYPE` | 公有云类型，影响 PVU / vCore 计算 |
| `VIRTUALIZATION_HOST_SCAN_ENABLED` + `COLLECT_HOST_HOSTNAME` | 宿主机容量扫描 |
| `DOCKER_SCAN_ENABLED` | Docker 容器扫描 |

另有 s390 平台自动推导的 `FAIL_ON_MISSING_CAPACITY_SCAN`
（`setup_scanner_unix.yml:82`）和 zLinux 硬件描述环境变量
（`TLM_MACHINETYPE` 等）。

**② 拉取到控制节点** — `lmt_collect_results.yml` 第 2~3 段

- `create_scan_result_packages_folder.yml`：在 localhost 建
  `{{ lmt_local_file_storage_path }}/scan_result_packages/`（默认 `./lmt_file_storage`）
- `collect_scan_results_unix.yml` / `_windows.yml` / `_ibmi.yml`：
  按正则 `^\d{12}-.+-\d{10}\.(tar\.gz|zip)$` 查找 → `fetch` →
  **fetch 成功才删源文件**（`when: fetched_files is succeeded`，
  `ignore_errors: true`，删不掉不算失败）
- 三平台并行走 `strategy: free`，按 `ansible_facts['os_family']` 分流
  （Windows / OS400 / 其他）

**③ 推送到 LMT 服务器** — `upload_to_server_unix.yml` / `_windows.yml`

- datasource 路径优先级：`lmt_server_datasource_path` >
  `lmt_server_path + /datasource` > 默认 `/opt/ibm/LMT/datasource`
- 先 `stat` 校验目录存在（不存在直接 fail），再 `copy` 整个目录（mode 0644），
  成功后 `delegate_to: localhost` 删掉控制节点暂存目录

### 旁支数据流

| 流 | Playbook | 落点 | 是否上传服务器 |
|---|---|---|---|
| 扫描结果 | `lmt_collect_results.yml` | `scan_result_packages/` → server datasource | 是 |
| 扫描器状态 | `lmt_collect_status.yml` → `prepare_scanner_status_file.yml` | `lmt_file_storage/scanner_statuses.csv`（host、路径、endpoint_id、版本、安装状态） | 否，仅本地报表 |
| 故障诊断数据 | `lmt_collect_troubleshooting_data.yml` | 端点跑 `collect_support_data.sh` + 附加 `ansible_facts.txt` → 打包 `<host>_<endpoint_id>.tar.gz` → `lmt_file_storage/troubleshooting_data/` | 否，交付支持人员 |

### 实践要点

1. 端点产包与 Ansible 收包是**解耦**的，收集频率应 >= 打包频率。
2. 删源发生在 fetch 之后；部分失败则整批不删，下次重收，不会丢数据但可能重复搬运。
3. 用 `-l` 限定主机时**必须带上 `lmt_server,localhost`**，否则第 ①/③ 段被跳过，
   包只会堆在控制节点。
4. 非 root / 非 Administrator 运行时需要对 `output` 目录有读写删权限
   （Windows 还需目录启用继承），否则清理静默失败导致包重复累积。
5. `endpoint_id.txt` 是端点身份锚点，安装 / 配置后必须存在
   （`setup_scanner_unix.yml:172` 强校验）。

---

## 2. 上传是否有 API？—— 没有，必须走文件落盘

LMT 服务器对断连扫描包的摄取机制是**文件系统驱动**的：服务器周期性扫描
disconnected data source 目录、解析包、入库。包怎么到那个目录，LMT 不关心。

`api/sam/v2` 这套 REST API 的定位是**查询与管理**，不是数据摄取：

- `GET /api/sam/v2/computers`、`license_usage`、`software_installations`、报表导出
- `DELETE /api/sam/v2/computers` ← **本仓库唯一用到的 API**
  （`roles/ilmt_scanner/tasks/decommission_computer.yml:4`，卸载时下线计算机）
- 没有 “POST scan package / trigger datasource import” 这类端点

> 未验证：IBM 官方文档站为 JS 渲染，本次未能抓取正文核对完整端点清单。
> 请以你所用 LMT 版本的官方文档为最终确认。

### 绕开 Ansible copy 的三条替代路径

1. **让 datasource 成为共享存储**：NFS / SMB 导出，端点把 `PACKAGE_OUTPUT_DIR`
   指向挂载点。代价是网络与权限面变大，且要保证写入原子性
   （先写临时名再 rename），否则服务器可能读到半个包。
2. **改用连接模式（BigFix agent）**：通过 HTTP(S) 主动上传到 relay/server，
   只需出向 52311 端口。这才是真正“非文件 copy”的架构；断连模式本就是
   为装不了 agent / 网络不通的场景准备的降级方案。
3. **保留三段式但换搬运工**：只跑到第 ② 步，然后用对象存储 / 跳板机 /
   消息队列投递。`lmt_collect_results.yml` 的第 ③ 段（`hosts: lmt_server`）
   可整段替换。

---

## 3. 一份扫描结果能否被多个 LMT server 消费？—— 可以

### 为什么包是“服务器无关”的

`grep -E 'lmt_server|server_url|BES' setup_scanner_*.yml install_scanner_*.yml`
→ 零命中（仅匹配到两行 CentOS/Ubuntu 注释）。

- `setup_config.ini` 里全是**采集行为**参数，没有服务器地址、证书、token
- 端点唯一身份 `endpoint_id` 是**端点本地自生成**的
  （`create_endpoint_id_file_ibmi.yml:15` 显示它来自扫描器自己写的
  `config/computer.yml` 的 `endpointID` 字段），不是服务器下发的注册 ID
- `lmt_server_token` / `lmt_server_port` 只在卸载下线时用，与包内容无关

对比连接模式：BigFix agent 安装时要灌 `masthead`（含服务器地址 + 公钥），
那才是绑死服务器的。断连包没有这一层。

### 四个注意事项

1. **datasource 是“消费即销毁”的队列，不是共享只读仓库。**
   服务器导入后会移走 / 删除 / 归档。不能让两台服务器挂同一目录去抢。
   必须给每台服务器各复制一份。

   本 playbook 天然支持：把 `lmt_server` 做成含多台主机的组，`copy` 会各推一份，
   末尾的清理是 `delegate_to: localhost`（重复执行幂等）。
   ⚠️ 但 `decommission_computer.yml:4` 硬编码 `hostvars['lmt_server']`，
   引用**名为 `lmt_server` 的单个主机**。做 fan-out 时要么保留一台叫
   `lmt_server` 作为主服务器，要么改这个文件，否则卸载下线只会打到其中一台。

2. **版本兼容是单向的。** `vars/main.yml` 注释写明 `lmt_playbooks_version`
   “should be in-line with LMT server version”（当前 9.2.44.0）。
   多服务器消费时以**最低版本的服务器**为准。

3. **同 endpoint_id 进两台服务器 = 两份独立且都“合法”的报表。**
   - ✅ 合理：生产 + 测试/演练并行、灾备同步、迁移期双写过渡、
     按地域/业务单元切分（端点集合**不重叠**）
   - ❌ 不合理：两套都作为正式审计依据且端点集合重叠（重复计数）

4. **包名带时间戳，服务器按端点 + 时间做增量。**
   迁移场景若只喂最后一个包，当前容量对但历史趋势断裂。
   要么从头补包，要么走数据库迁移而不是包重放。

---

## 4. 升级扫描器的流程

```bash
# 前提：先升级 LMT 服务器，再跑此 playbook（docs/doc_maintain_environment.md）
ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml
```

安装与升级是同一入口，幂等。内部三段：

**第 1 段 `hosts: lmt_server` — 刷新控制节点安装包缓存**
（`refresh_installers_cache_unix.yml`）

1. 读缓存里的 `scanner_installers/version.txt` + `build_date.txt`
2. 在服务器 `{{lmt_server_path}}/disconnected-scanners/` 查找
   `LMT-DisconnectedScanner-<平台>-<版本>-<YYYYMMDD-HHMM>.(tar.gz|zip)`
3. 比较：`服务器版本 > 缓存版本` **或** `版本相同但 build 不同` → 才重新下载
4. 清空缓存目录 → `fetch` 全平台安装包 → 写新的 `version.txt` / `build_date.txt`

找不到包直接 `fail`（`any_errors_fatal: true`）。

**第 2 段 `hosts: all, strategy: free` — 端点升级**
（`install_scanner_unix.yml`）

1. `get_scanner_facts_unix` 从
   `iso-swid/ibm.com_IBM_License_Metric_Tool-<版本>.swidtag` 解析已装版本
2. 升级门槛：`not 已安装 or 缓存版本 > 已装版本`
3. 调 `uninstall_scanner_unix` 并传 `leave_config_backup: true` →
   备份 `config/` 关键文件到 `config_backup/` → 跑 `uninstall.sh` →
   删掉除 `config_backup` 外的一切
4. 磁盘空间检查（Linux/AIX/Solaris 各自算法，阈值 100MB）
5. `copy` 安装包 → 解包 → 删包
6. 恢复身份文件：`endpoint_id.txt`、`sw_config.xml`、`isotag_config.xml`、
   `slmtag_config.xml`（必需）+ `uuid.dat`、`vmman_host_id`、
   `computer_properties.yml`（存在则恢复）
7. 跑 `setup_scanner_unix` → 重写 `setup_config.ini` + `setup.sh` →
   **重建 cron**
8. 目录 owner 改回普通 ansible 用户 → 删 `config_backup`

`endpoint_id.txt` 被保留，因此升级后服务器上仍是同一台计算机，历史数据不断。

### ⚠️ 坑：build-only 更新不会被端点升级

| 位置 | 条件 |
|---|---|
| `refresh_installers_cache_unix.yml:105` | `版本 > 缓存` **或**（`版本 ==` 且 `build 不同`）→ 刷新缓存 |
| `install_scanner_unix.yml:203` | **仅** `cached_scanner_version is version(已装版本, '>')` |

端点侧只比版本号不比 build，且已装版本从 swidtag 文件名解析（**不含 build 信息**）。

后果：IBM 若发同版本号、不同 build 的修订包，控制节点会下载新包，
但**所有端点静默跳过升级**。

核对方法：

```bash
ansible-playbook lmt_collect_status.yml -i lmt_inventory.yml
cat lmt_file_storage/scanner_statuses.csv
```

### 升级时的其他注意事项

1. **`setup_config.ini` 不被备份。** 备份的只有 `config/` 里那 7 个文件；
   ini 每次由 `setup_scanner_unix.yml` 从 inventory 变量**重新生成**。
   所以所有 `lmt_scanner_*` 参数**必须写在 inventory 里**；
   手工改端点 ini 的做法升级后必丢。
2. **磁盘空间在卸载之后才检查**（`install_scanner_unix.yml` 的顺序）。
   空间不足会让端点停在“已卸载、未安装”状态。批量升级前先确认目标分区
   有 >100MB 富余。
3. **升级会立即触发一次软件扫描**（`SW_SCAN_SCHEDULE_ENABLED=TRUE` 时，
   setup 完成后约 2 分钟异步起）。升级当天 `output/` 会多出包，属正常。
4. **Windows 用 `become_user: system` + `runas`**，Linux 用 `become: true`；
   收集则不需要提权。
5. **只改扫描参数不换版本，用 `lmt_reconfigure_scanner.yml`**：
   它给 `setup_scanner_*` 传 `only_configure: true`，走 `configure.sh` 而非
   `setup.sh`，**不卸载不重装**，风险低得多。
6. **Windows 老版本行为不同**：`setup_scanner_windows.yml:71-78` 显示
   9.2.22 及更低版本首次计划扫描是“从现在起一周/一天后”，
   故显式设 `DISABLE_SW_SCAN_AFTER_SETUP=TRUE` 再单独异步拉起一次
   （`async: 518400`, `poll: 0`）。9.2.23 起已与 Unix 一致。
7. **已知缺陷 198641**（`docs/release_notes.md:52-58`）：改了每日打包计划后跑
   reconfigure playbook 不生效，9.2.23 修复。低于该版本时改此参数需走完整的
   install_or_upgrade。

### 升级不能靠收集 cron 顺带完成

`lmt_collect_results.yml` 只做 `find`/`fetch`/`copy`，**不碰任何安装逻辑**，
所以现有的收集 cron 永远不会升级扫描器。

也**不建议**把升级塞进同一条 daily cron —— 两者风险等级差一个量级：

- 收集：只读 + 搬文件，失败下次重试即可，无副作用
- 升级：卸载 → 删目录 → 重装 → 重建 cron。失败会留下**没有扫描器的端点**，
  期间产生的包会丢

建议分开，升级低频且紧跟服务器升级之后触发：

```cron
# 每天收集（时刻见第 6 节）
24 8 * * * ansible-playbook /opt/lmt-ansible/lmt_collect_results.yml -i /opt/lmt-ansible/lmt_inventory.yml >> /var/log/lmt_collect.log 2>&1

# 升级建议人工执行；若必须自动化，用低频 + 先跑试点组
30 3 1 * * ansible-playbook /opt/lmt-ansible/lmt_install_or_upgrade_scanner.yml -i /opt/lmt-ansible/lmt_inventory.yml -l lmt_server,pilot_group >> /var/log/lmt_upgrade.log 2>&1
```

若要合并成一条，用串行且**先升级后收集**（升级会重建 cron，先收集能把
升级前遗留的包捞走）：

```bash
ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml \
  && ansible-playbook lmt_collect_results.yml -i lmt_inventory.yml
```

---

## 5. 两个 crontab 不要混淆

| | 位置 | 内容 | 谁创建 | 作用 |
|---|---|---|---|---|
| **A. 扫描 cron** | 每个**端点** | 容量扫描、软件扫描、每日打包 | 扫描器自带的 `setup.sh` / `setup.bat`，由 `setup_config.ini` 驱动 | **产生**数据 |
| **B. 收集 cron** | **控制节点** | `ansible-playbook lmt_collect_results.yml` | 你手工写的（`docs/doc_schedule_collection.md:31`） | **搬运**已有数据 |

两个重要事实：

- 扫描器**不会自动扫描**，全靠 A 触发
  （`defaults/main.yml:181`：“Enables scheduling software scans **in cron**”）
- `lmt_collect_results.yml` **不触发任何扫描**，只捞 `output/` 里已存在的包
- **Ansible 从不直接操作 cron**（整个 role 零处使用 `cron` 模块），
  它只改 `setup_config.ini` 然后执行 `setup.sh`，由后者写 crontab。
  所以 A 的行为只能通过参数间接控制。

### 端点 cron 实例解读

某 Linux 端点上的真实内容：

```cron
09,39 * * * * /var/opt/ibm/LMTScanner/bin/run_hw_CRON.sh           >/dev/null 2>&1
11 08 * * 2   /var/opt/ibm/LMTScanner/bin/run_sw_and_pack_CRON.sh  >/dev/null 2>&1
54 7 * * *    /var/opt/ibm/LMTScanner/bin/pack_results_CRON.sh     >/dev/null 2>&1
```

| cron | 脚本 | 含义 | 对应参数 |
|---|---|---|---|
| `09,39 * * * *` | `run_hw_CRON.sh` | **容量扫描，每 30 分钟**（每小时第 9、39 分） | 无参数可配 |
| `11 08 * * 2` | `run_sw_and_pack_CRON.sh` | **软件扫描 + 打包，每周二 08:11** | `SW_SCAN_SCHEDULE_ENABLED=TRUE`、`SW_SCAN_FREQUENCY=WEEKLY` |
| `54 7 * * *` | `pack_results_CRON.sh` | **仅打包，每天 07:54** | `DAILY_PACK_RESULTS_CREATION_ENABLED=TRUE` |

`hw` = hardware/capacity。三点印证：

- 脚本名 `run_sw_and_pack_CRON.sh` 本身证明软件扫描与打包绑在一个脚本里，
  即 `defaults/main.yml:183`「Software scan after each execution will
  automatically trigger pack results」的实现方式；与 Windows 侧
  `setup_scanner_windows.yml:109` 的 `run_sw_and_pack.bat` 同源。
- **没有独立的 UMS cron** —— UMS 嵌在软件扫描内（`defaults/main.yml:249`），
  故其刷新频率 = 每周二一次。
- `>/dev/null 2>&1`：cron 层不留日志，排障只能靠 `work/logs/` 或
  `lmt_collect_troubleshooting_data.yml`。

**周二会产生两个包，属正常**：07:54 的 pack（容量数据）+ 08:11 的
sw_and_pack（容量 + 刷新后的软件清单 + UMS）。两者都会被收集正则匹配并一起搬走，
服务器按时间戳增量处理。

---

## 6. `SW_SCAN_SCHEDULE_ENABLED` 与 `DAILY_PACK_RESULTS_CREATION_ENABLED` 的关系

**配置层面完全独立**：`setup_scanner_unix.yml` 中只有 `SW_SCAN_FREQUENCY` 带
`when: lmt_scanner_software_scans_enabled`（第 12 行），
而 `DAILY_PACK_RESULTS_CREATION_ENABLED`（第 38-42 行）**无条件写入**。
Windows 侧同构。四种组合都能配出来，Ansible 不阻止也不警告。

**功能上单向重叠**：

```
软件扫描执行完 ──自动──> 打包（无条件）
每日打包定时任务 ─────X──> 不触发任何扫描，只打包“现有的”结果
```

打包 ≠ 扫描。打包只是把 `work/` 里已有的扫描输出压成 `output/` 下的包。

| SW_SCAN | DAILY_PACK | 端点定时任务 | 服务器看到什么 |
|---|---|---|---|
| TRUE | TRUE | 软件扫描 + 每日打包 | ✅ **推荐**。容量数据每天新鲜，软件清单按扫描频率刷新 |
| TRUE | FALSE | 只有软件扫描 | ⚠️ 包只在扫描后产生。WEEKLY 时一周才一个包 → 容量数据陈旧，`Delayed Data Upload` / `Outdated Scan` 告警 |
| FALSE | TRUE | 只有每日打包 | ⚠️ 每天有包、容量新鲜，但**软件清单永久冻结**在最后一次扫描；UMS 也随之停更 |
| FALSE | FALSE | 无 | ❌ `output/` 永远为空，收集 playbook 每天捞到 0 个文件（不报错，静默）。该机器在服务器上逐渐失联 |

**为什么两个都要开**：容量扫描（PVU/RVU 计量的核心依据）不受这两个参数控制，
它有自己的 30 分钟节奏，数据写进 `work/`，但**只有打包才能把它送出去**。
所以 `DAILY_PACK=TRUE` 的真正作用是保证容量数据每天至少出一次门，
即使软件扫描是 WEEKLY。这就是官方默认 `WEEKLY` + `DAILY_PACK` 的原因 ——
软件扫描重、可以慢；容量数据轻、必须勤。

一句话：`SW_SCAN_SCHEDULE_ENABLED` 管**生产什么内容**，
`DAILY_PACK_RESULTS_CREATION_ENABLED` 管**多久发一次货**。
前者会顺带发一次货，后者不会生产内容。两个都 `true` 是唯一没有新鲜度缺口的组合。

---

## 7. 容量扫描频率：30 分钟，但不可通过本 playbook 配置

由第 5 节的 `09,39 * * * *` 实测确认为每 30 分钟。

本 playbook **完全不暴露**容量扫描频率参数。`defaults/main.yml`（全部 15 个参数）
与 `docs/doc_lmt_parameters.md` 中与 capacity 相关的只有三类，无一是频率：

| 参数 | 作用 |
|---|---|
| `lmt_scanner_zlinux_machine_type` / `processor_type` / `shared_pool_capacity` / `system_active_processors` | System z 上自动容量识别不可用时手工描述硬件 |
| `FAIL_ON_MISSING_CAPACITY_SCAN`（仅 s390，自动推导） | 容量数据缺失时是否让配置失败 |
| `lmt_scanner_virt_host_scan_enabled` + `lmt_scanner_collect_host_hostname` | 是否**扩大**容量扫描范围到虚拟化宿主机（`defaults/main.yml:326`：“becomes a part of the **regular** capacity scan”） |

### 如果确实要改

**不要手工改端点上的 ini** —— 升级后必丢（见第 4 节注意事项 1）。

正确做法是扩展 role：在 `defaults/main.yml` 加参数，在
`setup_scanner_unix.yml` / `_windows.yml` 各加一个 `lineinfile`，
照现有 11 行的写法，这样它就纳入“从 inventory 重新生成”的轨道，升级幂等。
代价是 fork 了 IBM 的 role，后续要合并上游更新。

**但建议先别改**：容量扫描很轻（读 `/proc`、`lscpu`、`lparstat` 之类，
不遍历文件系统），高频不是性能问题；调低频率会直接影响 PVU 计量准确性和
`Capacity Scan Health` 的合规状态。真正重的是软件扫描，那个已有
`SW_SCAN_FREQUENCY` 和 `SW_SCAN_CPU_THRESHOLD_PERCENTAGE` 可调。

先摸清端点上完整的 `setup_config.ini`，看「扫描器支持哪些参数」vs
「Ansible 暴露了哪些参数」的差集：

```bash
ansible unix1 -i lmt_inventory.yml -b -m shell \
  -a 'cat /var/opt/ibm/LMTScanner/config/setup_config.ini'
```

---

## 8. ⚠️ 数据新鲜度：收集时刻要对齐打包时刻

以第 5 节那台端点为例，若收集 cron 沿用文档示例的 02:30：

```
周三 07:39  容量扫描 → 数据写入 work/
周三 07:54  pack_results → 生成包放进 output/     ← 包在这里诞生
     ...    包在端点上静躺 18.6 小时
周四 02:30  收集 cron → fetch 到控制节点 → 推到服务器 datasource
周四 03:xx  服务器导入 → 报表可见
```

**服务器看到的容量数据实际是约 19 小时前的。**
容量扫描每 30 分钟跑一次对服务器端新鲜度**毫无帮助** ——
瓶颈不在扫描频率，而在 `pack → collect` 的时间差。

修法：把收集 cron 挪到打包之后。

```cron
# 原来（文档示例值）：延迟 ~19h
30 2 * * * ansible-playbook .../lmt_collect_results.yml -i .../lmt_inventory.yml

# 改成打包后 30 分钟：延迟降到 ~0.6h
24 8 * * * ansible-playbook /opt/lmt-ansible/lmt_collect_results.yml -i /opt/lmt-ansible/lmt_inventory.yml >> /var/log/lmt_collect.log 2>&1
```

这直接改善 Dashboard 上的 `Delayed Data Upload` 状态
（`docs/doc_maintain_environment.md` 特意提醒要盯的指标）。

**前提**：`07:54` 只是**那一台**的打包时刻，其他端点大概率不同。
正确做法是先摸清全机群的打包时间分布，取**最晚的那个 + 缓冲**作为收集时刻。

---

## 9. 升级时如何处理扫描时间点

升级路径是 `uninstall.sh` → `rm -rf` → 解包 → `setup.sh`：

1. `uninstall.sh` 清掉旧 cron 条目
2. `setup_config.ini` 从 inventory **重新生成**（旧 ini 连同整个目录被删，
   `config_backup` 只保 7 个身份文件，不含 ini）
3. `setup.sh` 按新 ini **重建 cron**

因此当 `SW_SCAN_LOCAL_TIME` 为空时，**A 的时间锚点 = 跑升级 playbook 的那一刻**。
`strategy: free` + 几百台同时升级 → cron 条目集中在同一分钟附近 →
下个周期所有端点同时开始软件扫描（全文件系统遍历，CPU/IO 都重）。

**核心原则：把 A 的时间显式钉死，让它与“何时跑升级”解耦，升级即对扫描节奏幂等。**

### 方案 1：按组固定错峰（简单、够用）

```yaml
all:
  children:
    unix_batch1:
      hosts: {unix1: ..., unix2: ...}
      vars:
        lmt_scanner_software_scans_frequency: "WEEKLY"
        lmt_scanner_software_scan_day_of_week: "SAT"
        lmt_scanner_software_scan_local_time: "01:00"
    unix_batch2:
      hosts: {unix3: ..., unix4: ...}
      vars:
        lmt_scanner_software_scans_frequency: "WEEKLY"
        lmt_scanner_software_scan_day_of_week: "SAT"
        lmt_scanner_software_scan_local_time: "03:00"
    unix_batch3:
      vars:
        lmt_scanner_software_scan_day_of_week: "SUN"
        lmt_scanner_software_scan_local_time: "01:00"
```

### 方案 2：按主机名哈希自动散列（几百台时推荐）

写在 `group_vars/all.yml`，新增主机无需维护；哈希确定性 ——
同一台机器每次升级算出同一时间，**反复升级不改变扫描节奏**（这是关键收益）。

```yaml
lmt_scanner_software_scans_frequency: "WEEKLY"

# 按主机名哈希散到 周六/周日 的 00:00-05:59，每 10 分钟一个槽位
lmt_scanner_software_scan_day_of_week: >-
  {{ ['SAT','SUN'][ inventory_hostname | hash('md5') | truncate(8, true, '') | int(base=16) % 2 ] }}
lmt_scanner_software_scan_local_time: >-
  {{ '%02d:%02d' | format(
       inventory_hostname | hash('md5') | truncate(6, true, '') | int(base=16) % 6,
       (inventory_hostname | hash('md5') | truncate(6, true, '') | int(base=16) // 6) % 6 * 10
     ) }}
```

先验证分布：

```bash
ansible all -i lmt_inventory.yml -m debug \
  -a "msg={{ lmt_scanner_software_scan_day_of_week }} {{ lmt_scanner_software_scan_local_time }}" \
  | grep msg | sort | uniq -c
```

### 方案 3：分批升级（与 1/2 互补，非替代）

```bash
for g in batch1 batch2 batch3; do
  ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml -l lmt_server,$g
done
```

时间已钉死时，分批只是为了控制**升级本身**的并发（每台会立即跑一次首扫）。

### 三个约束

1. **每日打包时刻无法配置。** `DAILY_PACK_RESULTS_CREATION_ENABLED` 只有开关，
   没有对应时间参数（`defaults/main.yml:266`：“first pack results is initiated
   **23 hours after you install**”）。打包时刻永远锚定升级时刻 +23h。
   影响较小（压缩很轻），且 23h 这个奇数偏移本身会让各端点缓慢错开；
   但要知道升级后包的出现时刻会变 —— 连带影响第 8 节的收集时刻选择。
2. **升级会立即触发一次扫描，参数钉死也拦不住**，只能靠分批。
   若规模很大想完全避开首扫风暴，可两步走：
   ```bash
   # 第一步：不产生首扫
   ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml \
     -e lmt_scanner_software_scans_enabled=false
   # 第二步：reconfigure（不卸载重装，走 configure.sh，风险低）开启并落地散列时间
   ansible-playbook lmt_reconfigure_scanner.yml -i lmt_inventory.yml
   ```
3. **`-l` 限定时必须包含 `lmt_server`**，否则第 1 段缓存刷新被跳过，
   端点会拿旧缓存装。

### 待确认：扫描器是否自带随机化？

第 5 节那台的分钟值（:09/:39、:11、:54）无明显关联，`09,39` 这种非整点半小时
错位说明扫描器**会做一定偏移处理**。但单台样本无法判定这些值是
「按安装时刻推算」还是「扫描器随机散列」—— 这直接决定方案 2 是否必要。

用一条命令判定：

```bash
ansible all -i lmt_inventory.yml -b -m shell \
  -a 'crontab -l 2>/dev/null | grep -E "run_hw_CRON|run_sw_and_pack_CRON|pack_results_CRON"'
```

- 各端点时间**散得开** → 扫描器自带随机化，不需要做散列错峰
- **扎堆在相近时刻** → 确认安装时刻锚定，方案 2 有必要

同一条命令的输出也用于第 8 节：取最晚的打包时刻来定收集 cron。

---

## 10. 待办 / 待验证清单

- [ ] 跑第 9 节末尾那条命令，摸清全机群 cron 时间分布
- [ ] 据此重定收集 cron 时刻（第 8 节），压缩数据上报延迟
- [ ] 若分布扎堆，落地第 9 节方案 2 的哈希散列
- [ ] 导出一台端点完整的 `setup_config.ini`（第 7 节命令），
      对比扫描器支持参数与 role 暴露参数的差集
- [ ] 确认 IBM 是否会发同版本号、不同 build 的修订包
      （若会，需处理第 4 节的 build-only 升级坑）
- [ ] 核对官方文档：是否真无扫描包上传 API（第 2 节「未验证」项）
