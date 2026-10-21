# 约 4000 个 ILMT 端点：`lmt_collect_results.yml` 在 AWX / AAP 中的周期作业设计（中文研究笔记）

> 本文是调研笔记，不属于 IBM 随包文档；标注「未验证」的部分请自行核实。
> 调研日期：2026-10-20。所依据的 playbooks 版本：9.2.45.0（`roles/ilmt_scanner/vars/main.yml:21`）。
>
> 来源约定：
> - **AWX 源码**：`ansible/awx` tag **24.6.1**（commit `94e5795`，2024-07-02，调研时 `git ls-remote` 列出的最新 tag）。
>   行号以该 tag 为准，`devel` 分支可能已变化。AAP 的行为以 Red Hat 文档为准，本文未对 AAP 内部源码做核对。
> - **ansible-core 源码**：tag **v2.13.13**（commit `c2b0ea5`）。本仓库 `execution-environment.yml` 固定的是 2.13.10，
>   同一 2.13 系列。
> - **实测**：本机 ansible-core **2.15.13**（与 EE 的 2.13 不同），全部用 `ansible_connection: local` 的假端点，
>   未连接任何外部主机，未修改仓库已有文件。实测只证明 Ansible 语义，**不代表** SSH / WinRM 下的耗时。
>   本机是共享机器（负载 1–2.5），耗时数字只能当量级参考。
> - 「**未验证**」= 没有一手来源或没有做实验；「**未实测**」= 只读了源码，没有跑。
> - 备注：Red Hat docs.redhat.com 页面是用 WebFetch 摘录的（摘要模型转述 + 短引文），措辞以页面原文为准。

---

## 0. 结论速览

### 0.1 七条设计风险的判定

| # | 你提出的风险 | 判定 | 一句话 |
|---|---|---|---|
| 1 | 切片后只有一个 slice 含 `lmt_server`，其它 slice 的第④段不上传 | **成立** | 其它 slice 第④段 `skipping: no hosts matched`，退出码 0，**不报错**；包留在各自 job 的暂存目录里 |
| 2 | 暂存目录是临时的 + 端点包已删 ⇒ 数据丢失 | **成立，且比你说的更宽**：不限于 container group，也不限于切片 | AWX 默认 `./lmt_file_storage` 落在每个 job 的临时 private data dir 里，job 结束即删；"第③段删端点包"与"第④段上传"之间任何失败都会丢包 |
| 3 | 第④段 `gather_facts: false` 却用 `ansible_facts['os_family']`，切片后变量未定义而失败 | **按你描述的场景不成立；换了触发条件则成立** | lmt_server 不在 slice ⇒ 第④段无主机，不会求值；但只要 lmt_server 进了第④段而本次没采集过它的 facts（拆分 JT、`add_host`、facts 缓存过期），**必然 fatal** |
| 4 | forks / `strategy: free` 瓶颈 | **部分成立，且瓶颈常不在 forks** | 默认 forks=5；主进程单线程调度，本机实测 4000 台 × 1 个空任务约 172 s，且 forks 5/20/100 结果相同；第④段 `copy` 逐文件执行，约 0.5 s/文件（本机），4000 个包 ≈ 30+ 分钟 |
| 5 | AWX 容量模型 | 已核实 | task impact = `min(inventory 主机数（切片后按份数折算）, forks 或 5) + 1`，**不看 Limit**；container group 不使用实例容量算法，只受 `max_forks` / `max_concurrent_jobs` 约束 |
| 6 | 周期作业设置 | 已核实 | 调度器不检查重叠，靠 `allow_simultaneous=false` 让后到的 job 排队；"relaunch failed hosts" 生成的 `limit` **不含 lmt_server**（会再次触发风险 2） |
| 7 | 解耦 / 按 group 错峰优于切片 | **成立（对本 playbook）** | 见第 7、8 节；零代码改动的可行方案是 B 与 C |

### 0.2 额外发现（你没列出，但影响设计）

1. **lmt_server 不可达时静默丢包**（实测）：lmt_server 在第③段被标 UNREACHABLE，会被移出活动主机，第④段无主机可跑；
   此时端点上的包已被删，退出码 4。
2. **`Prevent Instance Group Fallback` 关闭时可能静默落到没挂载共享卷的节点**：第②段 `file: state=directory` 会在任何路径
   上"创建目录"，不检查它是不是挂载点，于是暂存目录悄悄变成临时目录。
3. **共享暂存目录 + 并发 job = 丢包**：第④段 `copy` 拷整个目录后 `file: state=absent` 删整个目录；`fetch` 非原子写入。
4. 第④段 `copy` 是逐文件执行（`copy.py:509` 循环），官方模块文档自己写明"不适合数百个以上文件"。

### 0.3 推荐

**首选 B（按 inventory group 拆成多个 JT，每个都带 `lmt_server,localhost`，每个 JT 独立的暂存目录，错峰）**，
规模继续增大时升级到 **C（并行收集 → 汇聚 → 单次上传）**，C 的最小形态**不需要改 playbook**（见 8.3）。
**不要对本 playbook 直接使用 `job_slice_count`。** 理由与量化见第 8 节。

---

## 1. 本 playbook 的结构与耦合点（先读仓库得到的事实）

`lmt_collect_results.yml` 四段（`lmt_collect_results.yml:1-55`）：

| 段 | hosts | 做什么 | 与本文相关的点 |
|---|---|---|---|
| ① | `localhost` | 打印版本 | — |
| ② | `localhost` | `file state=directory` 建 `{{ lmt_local_file_storage_path }}/scan_result_packages/` | 默认 `./lmt_file_storage`（`roles/ilmt_scanner/defaults/main.yml:7`）；`any_errors_fatal: true` |
| ③ | `all`，`gather_facts: true`，`gather_subset: "!all,!min"`，`strategy: free` | 按 `os_family` 分流到 windows/unix/ibmi 收包任务：`find` → `fetch(flat)` → `when: fetched_files is succeeded` 才删端点原包（`ignore_errors: true`） | 删除发生在 fetch 成功之后；**删除时上传尚未发生** |
| ④ | `lmt_server`，`gather_facts: false` | `copy` 整个暂存目录到 datasource；成功后 `delegate_to: localhost` 删暂存目录 | `when` 使用 `ansible_facts['os_family']`，依赖第③段已采集的 facts |

关键耦合：**①②④ 依赖"暂存目录对本次 job 内所有 play 可见"**，**④ 依赖"lmt_server 出现在本次 job 的 inventory 里"**。
`lmt_inventory_template.yml:5-7` 里 lmt_server 是 `all` 里的普通 host（`ansible_connection: local`）。

---

## 2. 风险 1：Job slicing 与第④段

### 2.1 AWX 如何分配 hosts 给 slice

- `Inventory.get_script_data()` 先按主机名排序，再取 `[offset::slice_count]` 的等距子集：
  `host_queryset = self.hosts.filter(**hosts_kw).order_by('name')...`，`hosts = self.get_sliced_hosts(...)`
  （`awx/main/models/inventory.py:306-315`，`get_sliced_hosts` 在 `:287-304`，核心为 `host_queryset[offset::slice_count]` 第 303 行）。
- 分组关系只是在已选出的 host 集合上过滤：`if host_name not in all_hostnames: continue  # host might not be in current shard`（`:336`）。
  **没有"固定某个 host / group 到每个 slice"的机制。** slice 与 group 无关。
- slice 的 inventory 文件由 `build_inventory` 以 `slice_number` / `slice_count` 生成（`awx/main/tasks/jobs.py:324-330`）。
- 切片份数：`min(job_slice_count, inventory.hosts.count())`（`awx/main/models/jobs.py:336-346`）；
  `job_slice_count > 1` 时模板启动的是 **WorkflowJob**（`create_unified_job`，`jobs.py:378-405`），每个 slice 是一个 workflow node。
- 每个 slice job 单独启动 `ansible-playbook`，`allow_simultaneous` 被强制为 True、`job_slice_number/count` 被设置
  （`awx/main/models/workflow.py:382-386`）。
- 官方说明："The inventory is distributed evenly amongst the sliced jobs"（`docs/job_slicing.md`）；"the inventory fed into ansible-playbook is a pared-down version of the original inventory that only contains the hosts in that particular slice"
  （`docs/docsite/rst/userguide/job_slices.rst`）。
  AAP 2.5："Job slicing" 章同义，并明确 "Any job that intends to orchestrate across hosts (rather than just applying changes to individual hosts)" 不应配置为切片作业
  （https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-job-slicing ）。
  本 playbook 恰好是跨 host 编排（端点 → 控制节点 → lmt_server），属于被警告的类型。
- 带 Limit 的切片：若 Limit 使某 slice 没有 host，该 slice **失败**（`job_slices.rst`："those slices will fail, causing the overall job to fail"）。

### 2.2 实测：lmt_server 不在 slice 时第④段的行为

方法：用上述 AWX 算法（按名排序、`[offset::3]`）把 7 台 + `lmt_server` 切成 3 份，各自生成 inventory，对 playbook 逐份运行
（实验目录 `/tmp/lmt_exp/exp`）。

| slice | 含的 host | 第③段 | 第④段 | 退出码 | 结果 |
|---|---|---|---|---|---|
| 1 | h01 h04 h07 | 3 台 fetch 并删除端点包 | `skipping: no hosts matched`（并有 `Could not match supplied host pattern, ignoring: lmt_server` 警告） | 0 | 包留在暂存目录，**datasource 为空** |
| 3 | h03 h06 | 同上 | 同上 | 0 | 同上 |
| 2 | h02 h05 lmt_server | 3 台 fetch 并删除 | 上传 | 0 | 只上传本份 |

补充（`--list-hosts`，同一实验）：
- slice 的 inventory 里没有 `localhost` 条目，但第①②段依然在 `localhost` 上执行 —— Ansible 的隐式 localhost 在每次运行中都存在，
  切片对它没有影响。`hosts: all` 不包含隐式 localhost（`-l lmt_server,localhost,h01` 时第③段只有 h01、lmt_server）。
- 在**同一个共享目录**下（本机 CLI，我是按 slice 1、3、2 的顺序依次运行的，不是并发），slice 2 把前面所有 slice 的包都上传了 —— 因为暂存目录是共享的。
  **这在 AWX 里不会发生**：每个 job 有自己的 private data dir（见第 3 节）。

结论：风险 1 成立。结果既不是"报错"也不是"阻塞"，而是**静默的数据滞留**。

### 2.3 能不能让 lmt_server 出现在每个 slice？

- 源码层面：没有（2.1）。
- Limit 无济于事：Limit 作用于切片之后的 inventory（`jobs.py:1006-1007`，`-l job.limit`），且会触发"空 slice 失败"。
- **playbook 层面**可行（需改 playbook）：不把 lmt_server 放进 inventory，改为在第①段用 `add_host` 动态创建，
  连接信息通过 extra_vars / 全局变量提供。我在 `/tmp/lmt_exp/design/collect_sliced.yml` 做了设计草图并实测：
  3 个 slice 并发运行，各用独立暂存目录，全部成功上传到同一个 datasource（7 个包齐全），暂存目录各自清空。
  **代价**：`add_host` 创建的主机没有 facts，第④段必须 `gather_facts: true`，否则 `when` 中的 `ansible_facts['os_family']` 报错
  （实测，见 4.2）。多 slice 并发写同一个 datasource 的行为见 3.4。

---

## 3. 风险 2：暂存目录的生命周期与数据丢失

### 3.1 AWX 中 `./lmt_file_storage` 在哪里、何时消失

1. `lmt_local_file_storage_path` 默认 `./lmt_file_storage`（`defaults/main.yml:7`）。相对路径由 `fetch` 用 `loader.path_dwim(dest)` 解析
   （`ansible-core v2.13.13 lib/ansible/plugins/action/fetch.py:153`），即**相对 playbook 所在目录**而不是进程 cwd。
   实测：在 `/tmp/lmt_exp/rel/cwd` 下运行，目录建在 playbook 目录（`repo_copy/lmt_file_storage`），cwd 里什么都没有。
2. AWX 每个 job 创建一次性目录：`build_private_data_dir` 用 `tempfile.mkdtemp(prefix=JOB_FOLDER_PREFIX % pk, dir=settings.AWX_ISOLATION_BASE_PATH)`，
   且 `if settings.AWX_CLEANUP_PATHS: self.cleanup_paths.append(path)`（`awx/main/tasks/jobs.py:209-216`），
   job 结束时 `with_path_cleanup` 对这些路径 `shutil.rmtree`（`:90-103`）。`AWX_CLEANUP_PATHS` 默认 True（`awx/settings/defaults.py:996`）。
3. 项目被**整体复制**到该目录的 `project/` 子目录（`RunProjectUpdate.make_local_copy`，`jobs.py:1363-1372`，`shutil.copytree(project_path, <private_data_dir>/project)`）。
   ansible-runner 把 `project` 目录作为 playbook 根与工作目录（https://github.com/ansible/ansible-runner/blob/devel/docs/intro.rst ，"the directory that will be set as the current working directory when launching the Ansible process"）。
   ⇒ `./lmt_file_storage` 实际是 `<private_data_dir>/project/lmt_file_storage`，**随 job 清理而消失**。
   在 EE 容器内，private data dir 是 `/runner`（`awx/main/tasks/receptor.py` 的 `--private-data-dir=/runner`；`awx/main/utils/execution_environments.py` 的 `CONTAINER_ROOT = '/runner'`）。
4. Container group 的 Pod 是一次性的："pods are created on-demand and only exist for the duration of the playbook run"
   （`docs/docsite/rst/administration/containers_instance_groups.rst`，"Container Groups" 节；`docs/container_groups.md` 同义）。
5. 执行节点（非 container group）上的残留文件另有 `ansible-runner worker cleanup` 兜底（`Instance.get_cleanup_task_kwargs`，
   `awx/main/models/ha.py`，`grace_period` 默认 60 分钟，仅匹配 `JOB_FOLDER_PREFIX` 目录）。具体在 execution node 上的清理时序**未验证**。

结论：**无论 hybrid/execution 节点还是 container group，默认暂存目录都是 job 级别的临时目录。**

### 3.2 丢包的窗口（逐项实测）

端点包在第③段 `fetch` 成功后立刻被删，而上传发生在第④段。以下任一情形，包只剩在临时暂存目录里，job 结束即消失：

| 情形 | 实测结果 |
|---|---|
| 切片，lmt_server 不在本 slice | 第④段跳过，退出码 0（2.2） |
| `-l`/Limit 漏掉 lmt_server | 第④段 `skipping: no hosts matched`；端点包已删，包滞留暂存目录 |
| **lmt_server 不可达**（把 lmt_server 配成连不上） | 第③段 `lmt_server` 被标 `UNREACHABLE`；第④段无主机；`ok1` 的端点包被 fetch 并删除，暂存目录里有包，退出码 4 |
| datasource 目录不存在 | 第④段 `stat` 触发 `failed_when`，`FAILED`；包滞留 |
| 其它端点不可达 | **不影响**：第④段仍会执行（实测 `PLAY4 REACHED despite unreachable endpoint`，退出码 4） |
| job `Timeout` 触发 / 手动取消 | 进程被杀；源码见 `runner_settings['job_timeout']`（`jobs.py:581`）；未专门实测，推断同上 |
| `copy` 中途失败 | `uploaded_files_unix is succeeded` 为假 ⇒ 暂存目录不删（CLI 下保留，AWX 下随 job 清理而丢） |
| **`Relaunch on failed hosts`** | 生成的 limit 是 `','.join(retry_host_list)`（`awx/api/views/__init__.py:3588-3607`），只有失败的端点，**不含 lmt_server** ⇒ 只收不传 |

在**持久且固定**的暂存目录下，上述情形是**自愈**的：下一次运行第④段会把目录里所有残留一并上传
（实测：`rel2` 实验中上一轮遗留的 `ha` 包被本轮上传到 datasource）。这是"拷贝整个目录"这个设计的好处，
但前提是目录跨 job 持久。

### 3.3 解法一：把暂存目录放到持久卷（配置即可）

只需设置 `lmt_local_file_storage_path`（inventory / JT extra_vars 均可）。注意**变量作用域**（实测）：
- inventory 的 **group vars** 只对该 group 的 host 生效；`localhost`（第②段）和 `lmt_server`（第④段）看不到，会得到默认值。
- **JT extra_vars（`-e`）对所有 play 的所有 host 生效**，包括隐式 localhost。
- 放在 `all:vars` 也对三者生效。
⇒ 用 JT extra_vars 或 `all:vars`，不要放在子 group vars。

挂载方式：

| 环境 | 方式 | 出处 / 注意 |
|---|---|---|
| 实例组（VM 上的 execution/hybrid 节点，EE 用 podman 运行） | **Settings → Jobs → "Paths to expose to isolated jobs"**（`AWX_ISOLATION_SHOW_PATHS`），格式 `HOST-DIR[:CONTAINER-DIR[:OPTIONS]]` | `awx/main/conf.py:277-290`、`awx/main/tasks/jobs.py:181-200`、`docs/docsite/rst/userguide/execution_environments.rst`。**这是全局设置**，所有 job 都挂载。若有多个 execution 节点，该路径必须在**每个**会跑此 JT 的节点上指向同一份共享存储（NFS 等），否则又回到"每节点一份"。多节点共享一致性：**未验证** |
| container group | 在 **Customize pod specification** 里加 `volumes` / `volumeMounts`（PVC） | `awx/main/tasks/receptor.py:600-686`；见下方 pod spec 合并规则 |
| container group（hostPath 路线） | 设置 `AWX_MOUNT_ISOLATED_PATHS_ON_K8S=True` + `AWX_ISOLATION_SHOW_PATHS` | `receptor.py:631-666`；`conf.py:377-` 文字警告 "HostPath volumes present many security risks"。hostPath 是**节点本地**，多节点调度会分叉，不适合共享暂存 |

**pod spec 合并规则（源码）**：自定义 spec 与默认 spec 用 `deepmerge` 合并（`receptor.py:611`），
`deepmerge`（`awx/main/utils/common.py:1113-1127`）对 dict 递归合并、对**列表整体替换**（`elif b is None: return a else: return b`）。
默认 spec 为 `containers: [{image, name:'worker', args, resources:{requests:{cpu:'250m', memory:'100Mi'}}}]`
（`awx/main/utils/execution_environments.py:32-52`）。因此在 override 里写 `containers:` 列表会**替换**默认列表：
`image` 和 `args` 之后会被 AWX 重新设置（`receptor.py:613-614`），但 `name` 和 `resources` 要自己写全。
`volumes` / `volumeMounts` 如果默认 spec 里没有，则不受此影响，AWX 还会把 isolated-path 的卷追加到已有列表上（`receptor.py:656-666`）。
override 写法示例（草案，**未在真实集群验证**）：

```yaml
spec:
  containers:
    - name: worker
      resources:
        requests: {cpu: "2", memory: 6Gi}
        limits:   {memory: 8Gi}
      volumeMounts:
        - {name: lmt-stage, mountPath: /mnt/lmt_stage}
  volumes:
    - name: lmt-stage
      persistentVolumeClaim: {claimName: lmt-stage-rwx}
```

PVC 访问模式（https://kubernetes.io/docs/concepts/storage/persistent-volumes/ ，Access Modes）：
- `ReadWriteOnce`："mounted as read-write by a single node"，同一节点上的多个 Pod 仍可同时读写；
- `ReadWriteMany`：多节点读写；
- `ReadWriteOncePod`：整个集群只允许一个 Pod 使用该 PVC。
并发 job 跨节点共享暂存，需要 RWX；`ReadWriteOncePod` 会让第二个 Pod 一直 Pending（AWX 默认 Pod pending 超时 2h，`defaults.py:62`）。
注意卷名叫"持久"并不保证写入原子性。

**同时必须防止"静默回退到未挂载的节点"**：第②段只是 `file state=directory`，对任何路径都成功，所以没挂载时它会在容器可写层里
新建一个目录，一切看起来正常，但数据是临时的。配置层面的防护：
- JT 上选中专用实例组 / container group，并勾选 **Prevent Instance Group Fallback**
  （`jobs.py:275-283`、`preferred_instance_groups` `jobs.py:799-812`：JT → inventory → organization 依次累加，遇到设置了该标志的对象即停止，
  没有任何实例组时落到 `default`（+ controlplane）全局组，`unified_jobs.py:1538`）；
- 或把实例组关联到 inventory（`containers_instance_groups.rst`"Control Where a Job Runs"："run only on the group associated with it"）。
playbook 层面的防护（需改 playbook，见 9.2）：断言暂存路径是挂载点。

### 3.4 解法二：每个 job 独立暂存目录（配置即可）

AWX 在 extra_vars 里注入 `awx_job_id`、`awx_job_template_id/name`、`awx_workflow_job_id` 等（及 `tower_` 同义变量，
`awx/main/models/unified_jobs.py:1554-1584`、`awx/main/constants.py:101`、`awx/main/tasks/jobs.py:1031`），
可把 `lmt_local_file_storage_path` 设为 `/mnt/lmt_stage/{{ awx_job_template_id }}`（按 JT 隔离）或 `/mnt/lmt_stage/{{ awx_job_id }}`（按 job 隔离）。
实测 `-e 'lmt_local_file_storage_path=/mnt/lmt/{{ awx_job_id }}'` 能被 localhost / lmt_server / 端点同时解析
（`/tmp/lmt_exp/tpl`，目录建为 `stage/4242/scan_result_packages`）。

- **按 JT 隔离**：同一个 JT 不并发（`allow_simultaneous=false`）时保留"失败自愈"（下次运行会补传该 JT 目录里的残留）；
  不同 JT 之间互不干扰。**推荐**。
- **按 job 隔离**：避免任何并发冲突，但失败后会留下"孤儿目录"，没有任何后续 job 会去上传它（需要一个清扫 job，要改 playbook / 新增 playbook）。

### 3.5 共享目录 + 并发写的风险（源码 + 推理）

- 第④段 `copy` 把 `src: <dir>/` 递归拷贝（`roles/ilmt_scanner/tasks/upload_to_server_unix.yml:17-21`；`copy` action 先 `_walk_dirs` 列出文件再逐个处理，
  `lib/ansible/plugins/action/copy.py:485,509`），成功后 `file state=absent` 删除**整个目录**（`:25-28`）。
- `fetch` 通过连接插件直接把文件写到 `dest`（`fetch.py:169-173`，`self._connection.fetch_file(source, dest)`，没有临时文件 + rename），
  所以另一个 job 的第④段可能看到**写了一半的包**并拷走；之后 `state=absent` 会把还没来得及被列举的、已从端点删除的包一并删掉。
- 这意味着：共享同一目录的并发 job，**既可能向 LMT 递交残包，也可能直接丢包**。LMT 对残包的处理（拒绝 / 报错 / 隔离）**未验证**。
- 上传侧：`ansible.builtin.copy` 用 `atomic_move` 落盘（`lib/ansible/modules/copy.py:709`，`module_utils/basic.py:1653`：同目录 `mkstemp(prefix='.ansible_tmp')` 再 rename），
  datasource 里会短暂出现 `.ansible_tmp*` 隐藏临时文件；LMT 是否会误读，**未验证**。

---

## 4. 风险 3：第④段的 facts

### 4.1 Ansible 的 facts 作用域

- facts 默认只存在内存里，"stored in memory for the duration of the current playbook run"，需要持久化要换 cache 插件
  （https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html ，"Caching facts"）。
- `gathering` 配置：`implicit`（默认，"the cache plugin will be ignored and facts will be gathered per play unless 'gather_facts: False' is set"）、
  `smart`（"if the same host is addressed in multiple plays it will not be contacted again in the run"）
  （`lib/ansible/config/base.yml:658-672`）。
- 因此单个 job 里第③段对 lmt_server 采集的 `os_family`，第④段（`gather_facts: false`）可以直接使用：实测 `PLAY [all]` 采集后，同一次运行里后面的 play 能读到。
- 本机验证：`gather_subset: "!all,!min"` 仍会返回 `os_family`（实测返回了 `os_family=RedHat`，facts 键含 `os_family`、`env`、`distribution` 等）。

### 4.2 实测：什么时候会炸

| 场景 | 结果（实测） |
|---|---|
| 单 job，lmt_server 在 `all` 里（现状） | 正常 |
| 切片，lmt_server 不在本 slice | 第④段无主机，**不求值，不报错**（2.2） |
| lmt_server 在第④段、但本次运行从未采集过它的 facts（模拟：只跑第④段） | `fatal: ... 'dict object' has no attribute 'os_family'`，**`!=` 和 `==` 两种条件都炸**；`include_role ... when:` 同样炸（`/tmp/lmt_exp/t_incl.yml`） |
| 第③段改成 `hosts: all:!lmt_server`，第④段不采集 | 同上 |
| `add_host` 动态创建 lmt_server，第④段不采集 | 同上（`ansible_facts` 为空，实测 `0 NOFACT`）；第④段加 `gather_facts: true` 后恢复正常 |
| 外部 jsonfile fact cache 已有 lmt_server 的 facts | 第④段正常（`skipping`，实测）；缓存过期（`ANSIBLE_CACHE_PLUGIN_TIMEOUT=1` 且等待 2 s）后又 fatal |

### 4.3 AWX 的 "Enable Fact Storage"（`use_fact_cache`）能不能兜底

- 开启时，AWX 把库里该 inventory 的 host facts 写成 jsonfile，启用 `jsonfile` 缓存插件，结束后把变化写回库
  （`docs/fact_cache.md`；`awx/main/tasks/jobs.py:565,1089`；`awx/main/tasks/facts.py:24-`）。
- 注入范围 = `job.get_hosts_for_fact_cache()`，**切片时只含本 slice 的 host**（`awx/main/models/jobs.py:843-863`，`get_sliced_hosts`）。
  ⇒ 不能让一个不在本 slice 的 lmt_server 凭空有 facts。
- 过期：`ANSIBLE_FACT_CACHE_TIMEOUT` 默认 0 = 不过期（`awx/main/conf.py:513-525`；`facts.py:40`）。
- 官方警告：不要用 ansible.cfg 自己配 fact cache，可能与 AWX 冲突（`docs/docsite/rst/userguide/job_templates.rst`，Fact Caching 节）。
- 结论：开启 fact cache 能缓解"第④段没 facts"，但它**不解决**风险 1/2；而且本 playbook 的第③段本来就每次采集 facts，不需要它。

---

## 5. 风险 4：forks、`strategy: free` 与控制节点瓶颈

### 5.1 forks 的含义（ansible-core 源码）

- 默认值 5：`DEFAULT_FORKS: default: 5`（`lib/ansible/config/base.yml:650-657`，env `ANSIBLE_FORKS`，ini `forks`）；`TaskQueueManager._forks = forks or 5`（`task_queue_manager.py:117`）。
  官方："Maximum number of forks Ansible will use to execute tasks on target hosts"；
  命令行 `-f 30`（https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html ，"Setting the number of forks"）。
- worker 数 = `min(forks, iterator.batch_size)`（`task_queue_manager.py:294`）。
- 机制：每个"(host, task)"启动一个 `WorkerProcess` 子进程，同时在飞的最多 forks 个
  （`lib/ansible/plugins/strategy/__init__.py:405-418`）。**forks 限制的是并发的 host-task 数，不是并发的主机数。**
- `strategy: free`："a host that is slow or stuck on a specific task won't hold up the rest of the hosts and tasks"（`strategy/free.py` DOCUMENTATION）。
  `free` 不支持 `max_fail_percentage`（`free.py:95-96` 会 warn）。
  `linear`（默认）每个 task 要等所有 host 做完，对 Windows/Linux/IBM i 混合、耗时差异大的环境不利；本 playbook 用 `free` 是合理的。
- AWX 传递：JT 的 forks 经 `--forks=N` 传给 ansible-playbook，若超过 `MAX_FORKS`（默认 **200**，`awx/main/conf.py:530-538`）则被钳到该值（`jobs.py:998-1003`）；
  保存 JT 时超限直接报错（`jobs.py:325-328`）。JT 里 forks=0 表示用 Ansible 默认（5）（`job_templates.rst` "Forks" 行）。

### 5.2 控制节点瓶颈（先量化，再推理）

**(a) 主进程是单线程调度循环**。实测（`inv_N.yml`：N 台 `local` 连接假主机，1 个 `debug` 任务，`strategy: free`，`/tmp/lmt_exp/scale.out`）：

| 主机数 | forks | 耗时 | 约每 host-task |
|---|---|---|---|
| 250 | 20 | 7.4 s | 30 ms |
| 500 | 20 | 13.3 s | 27 ms |
| 1000 | 20 | 27.5 s | 28 ms |
| 2000 | 20 | 60.6 s | 30 ms |
| 4000 | 5 / 20 / 100 | **171.2 / 172.3 / 173.3 s** | 43 ms |
| 1000（`linear`） | 20 | 27.5 s | 同 free |

- 4000 台时 forks 从 5 提到 100 **没有任何变化** ⇒ 对"极轻任务"，瓶颈在主进程调度/结果处理，不在 worker 并发。
- 4000 时单位成本明显高于 ≤2000（43 vs ~29 ms，超线性），说明主进程开销随主机数×任务数增长。
- 对真实的网络任务（SSH/WinRM 往返几百 ms ~ 数秒），forks 才是主要杠杆，主进程成本是下限（floor）。

**(b) 主进程内存随 host×task 增长**（`/proc/<pid>/status` 采样，`/tmp/lmt_exp/memsample.out`）：1 个 debug 任务时，
主进程峰值 RSS ≈ 55 MB（200 台）、≈ 100 MB（1000 台）、≈ **620 MB（4000 台）**。
本 playbook 每台约有 10 余个任务结果（`gather_facts`、3 个 `include_role` + 条件跳过、`find`、`fetch` 循环、清理循环），
所以 4000 台的真实内存会明显高于 620 MB，**未实测**，需要用试点外推（第 10 节）。

**(c) 每个 fork 的内存**：本机 local 连接下 worker RSS ≈ 40–70 MB（PSS ≈ 16 MB，fork 后与父进程共享页）。
AWX 容量算法按 **每个 fork 约 100 MB** 估算（`docs/capacity.md`："about 100MB per fork"，`SYSTEM_TASK_FORKS_MEM` 默认 100）。
SSH/WinRM 连接、模块 payload、更大的 `setup` 输出会让真实值更高，**应实测**。

**(d) 一次 job 内 fetch 的传输**（经控制节点）：
- `fetch` 先 `stat` 取远端 checksum（有 `become` 时改用 `slurp`，"effectively doubles the transfer size" 并可能 `MemoryError`，
  https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html ；源码 `fetch.py:73-129`）。
  本 playbook 的收集任务没有 `become`，没有触发这条。
- SSH：`ssh` 连接插件的 `fetch_file` 走 `sftp/scp/piped`（`ssh.py:1191-1220`）；默认 `ssh_args` 含 `ControlMaster=auto -o ControlPersist=60s`（`ssh.py:77`）。
  pipelining 默认关（`config/base.yml:98-113`），开启可减少往返；与 `become` 的 requiretty 有冲突说明（本 playbook 收集阶段不用 become）。
- WinRM：`winrm.py:680-725` 的 `fetch_file` 以 **0.5 MB（`2**19`）为块**，每块运行一次 PowerShell 脚本并 base64 编码返回，
  即每块一次 WinRM 往返；`psrp.py:614-` 的块大小由 `max_payload_size` 决定。
  官方 Windows 文档：NTLM "slower to authenticate because it requires more round trips"；Kerberos 支持消息加密；CredSSP "only when absolutely necessary"
  （https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html ）。
  ⇒ 包越大、越多 Windows 端点，WinRM 往返越多；包大小**未知（未验证）**，需要从真实端点取样。
- 目标端并发限额：每台端点只有一个连接（每台是不同 host），不会触到单机 WinRS 默认的 `MaxShellsPerUser = 5` /
  `MaxConcurrentUsers = 5`（https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management ）。
  **如果**你的端点要经跳板机 / 堡垒机（`ProxyJump`）访问，则堡垒机 `sshd` 的 `MaxStartups`（默认 `10:30:100`）、`MaxSessions`（默认 10）
  （https://man.openbsd.org/sshd_config ）会随 forks 提高而成为瓶颈。是否有堡垒机：**不知道（需要你确认）**。

**(e) 第④段 `copy` 逐文件，是 4000 台场景里最容易被忽略的串行瓶颈**。
- 源码：`copy` action 先列目录，然后对每个文件走 `_copy_file`（`copy.py:509-519`），每个文件至少一次远端 stat + 一次 `copy` 模块执行（`:263,341`）。
  官方模块文档："The ansible.builtin.copy module recursively copy facility does not scale to lots (>hundreds) of files"（`lib/ansible/modules/copy.py:125`）。
- 实测（lmt_server 用 `local` 连接，20 KB 文件，本机）：100 个文件首次拷贝 **51.2 s**，**完全相同内容再拷一次也要 54.7 s**，即约 **0.5 s/文件**（1000 个文件的测试超过 400 s 被我中止，未得出完整数字）。
  对比：同样 100 个文件，`copy` + `remote_src: true` 约 1.4 s，`tar` 打包 + `unarchive` 约 2.7 s。
- 外推（公式，不是实测）：上传时间 `T_up ≈ P × t_file`，其中 P = 本次包的个数，`t_file ≈ 0.5 s`（本机 local 连接；若 lmt_server 是远程 SSH/WinRM，只会更高）。
  P = 4000 → 约 33 分钟；P = 8000（例如包含周扫描日的双份包）→ 约 67 分钟。
  这是一个**单线程串行**步骤，加 forks 没用。

**(f) 事件量/数据库压力**：AWX 对每个任务结果写事件。官方样例估算：一个 `debug` 在 1 台上、verbosity 1 就产生 6 个事件
（`docs/docsite/rst/administration/performance.rst` 容量规划例子）。本 playbook 估算下限：
`4000 台 × 约 12 个任务 × 6 事件 ≈ 29 万事件/次`（计算值，任务数为我的估计，**未实测**）。控制节点处理事件的能力（callback receiver）因此也是约束；
AWX 建议在事件压力下调低 `capacity_adjustment`（`performance.rst`）。

### 5.3 forks 取值的估算框架（公式 + 假设，不是出处数字）

记：N = 端点数；F = forks；`t_host` = 单个端点在**一个 fork 上**的端到端耗时（gather + find + fetch + 清理，含网络往返和传输）；
P = 包数；`c` = 主进程每个 host-task 的固定成本（本机空任务 ≈ 0.03–0.045 s，见 5.2a）；`T` = 每台的任务数。

```
T_collect  ≈  max( N × t_host / F ,  N × T × c )          # 网络并发受 F 约束；调度有下限
T_upload   ≈  P × t_file                                  # 串行，t_file 本机实测约 0.5 s
T_total    ≈  T_collect + T_upload
控制节点内存 ≈ M_main(N × T) + F × m_fork                  # M_main 实测随 N×T 增长；m_fork 按 AWX 默认 100 MB 保守估
```

套数（纯演算，`t_host` 为假设）：

| `t_host` 假设 | F=25 | F=50 | F=100 |
|---|---|---|---|
| 10 s | 26.7 min | 13.3 min | 6.7 min |
| 20 s | 53.3 min | 26.7 min | 13.3 min |
| 40 s | 106.7 min | 53.3 min | 26.7 min |

（N = 4000。表中只是 `N × t_host / F`，不含 `N × T × c` 下限，本机实测下限量级为 `4000 × 12 × 0.03–0.045 s ≈ 24–36 min`，说明在较大 F 下**调度下限开始主导**。）

要得到真实 `t_host`：用试点（第 10 节）在真实网络上跑 20–50 台，取"每台总耗时 × F / N"的反推值。
**不要把上表的 `t_host` 当事实。**

> 实测补充（`/tmp/lmt_exp/bench`）：把完整的第①②③段对 200 台 local 假端点（每台 1 个 50 KB 包）运行，forks=10 → 46 s，forks=40 → 45 s，即 ≈ 0.23 s/台，
> 二者无差别。再次说明本机 local 连接下是 CPU/主进程受限；换成真实网络时 forks 的作用会显现，但"主进程有下限"这一点仍成立。

---

## 6. 风险 5：AWX / AAP 容量模型

### 6.1 实例的容量

- 内存口径：`(mem_MiB − 2048) // SYSTEM_TASK_FORKS_MEM`（默认 100）；在 K8s（`settings.IS_K8S`）下不扣 2 GiB
  （`awx/main/utils/common.py:824-855`）。
- CPU 口径：`cpus × SYSTEM_TASK_FORKS_CPU`（默认 4），`max(1, …)`（`common.py:750-763`）。
- `capacity = min + (max − min) × capacity_adjustment`，`capacity_adjustment` 默认 1.0（取大者）（`awx/main/models/ha.py:321-329`，`:163`）。
  `docs/capacity.md`："mem_capacity (the default)" 模式。
- 例（纯计算，非 K8s）：4 vCPU / 8 GiB → mem 61、cpu 16；8 vCPU / 16 GiB → mem 143、cpu 32；16 vCPU / 32 GiB → mem 307、cpu 64；
  `capacity_adjustment=0.5` 时分别为 38.5 / 87.5 / 185.5。这些只是 AWX 的"记账口径"，**不等于**真实可跑的 fork 数。
- 本 playbook 的真实瓶颈更接近"主进程成本 + 网络往返"，AWX 默认的 100 MB/fork 记账口径对它可能偏保守（内存）或偏乐观（CPU，见 5.2a），需要实测校准。

### 6.2 task impact

`Job._get_task_impact`（`awx/main/models/jobs.py:690-702`）：

```
count_hosts = inventory.total_hosts                      # 注意：用 inventory 总主机数，不看 Limit
if job_slice_count > 1:
    count_hosts = (count_hosts + job_slice_count - job_slice_number) // job_slice_count
impact = min(count_hosts, 5 if forks == 0 else forks) + 1
```

- `+1` 是 Ansible 父进程（`docs/capacity.md`："AWX will add 1 to the number of forks"）。
- **Limit 不降低 impact**：源码用的是 `inventory.total_hosts`，不看 Limit。例如 inventory 有 4000 台、Limit 只选 30 台、forks=100，impact 仍是 `min(4000,100)+1 = 101`，
  而不是 `min(30,100)+1 = 31`。对"按 Limit 拆 JT"的方案意味着：每个批次 JT 的 impact 取决于它的 forks 与整个 inventory 的规模，所以 forks 应按批大小单独设置，不要照搬。
- WorkflowJob（含切片的父 job）impact = 0（`awx/main/models/workflow.py:714-715`）。
- 控制开销：每个 job 在 control/hybrid 节点上另占 `AWX_CONTROL_NODE_TASK_IMPACT`（默认 **1**，`awx/settings/defaults.py:65`；
  `ha.py:206-217` `consumed_capacity`）。hybrid 节点上合计 `forks + 1 + 1`（`docs/capacity.md` 例子："forks set to 1 → total 3"）。
- 调度选择：`fit_task_to_most_remaining_capacity_instance` 要求 `remaining − impact ≥ 0`，找不到时回退到 **"最大的空闲实例"**
  （`awx/main/scheduler/task_manager_models.py:209-241`；`task_manager.py:610-612`）。
  含义：**forks 大于任何实例容量的 job，只要有实例空闲就会被运行**（官方也这么说："will allow that Job to run on the Instance even if it would push the Instance over capacity"），
  AWX 不会替你拦住"forks=200 跑在 4 GB 机器上"。

### 6.3 实例组与 container group

| 项 | 内容 | 出处 |
|---|---|---|
| `max_concurrent_jobs` / `max_forks` | 实例组级上限，0 = 不限制。`has_remaining_capacity`：`remaining_jobs = max_concurrent_jobs − 运行中 − 1`，`remaining_forks = max_forks − 已用 − task_impact`，任一为负则不放行 | `ha.py:452-453`、`task_manager_models.py:99-125` |
| container group 的"容量" | 没有实例；`get_capacity()` 返回 `max_forks`；`max_forks=0`/`max_concurrent_jobs=0` 即无限提交 Pod | `task_manager_models.py:81-94`；`docs/capacity.md`："Container Groups where previously, there was no limit" |
| 容器组 Pod 资源 | 默认只有 **requests**（cpu 250m / memory 100Mi），**所有 job 用同一份 pod spec，不论 forks 多大**；因此 Pod 可超额使用；用 `max_forks` 限制总量 | `execution_environments.py:49`；`containers_instance_groups.rst`"Instance group capacity limits"最后一段 |
| container group 不使用实例容量算法 | "You would need to explicitly set the number of forks at the job template level… that setting will be passed along to the container" | `containers_instance_groups.rst` 末尾 Note |
| control 面开销 | container group job 只在 controller 上记 control impact（默认 1） | `docs/capacity.md` |
| `policy_instance_percentage` / `policy_instance_minimum` / `policy_instance_list` | 仅对**普通实例组**有意义；container group 没有实例。percentage 与 minimum 取"分配得更多的那个"规则 | `docs/clustering.md:184-192`；`ha.py:511`（容器组不触发策略任务）；`containers_instance_groups.rst` |
| 节点类型 | `control`（不跑 job）、`hybrid`（控制+执行）、`execution`、`hop`（仅转发） | `ha.py:176-181`；`docs/clustering.md:64-69` |
| 哪些任务能跑在哪 | Job 的 capacity_type = `execution`，只会被安排到 `execution` 或 `hybrid` 节点；项目更新等是 `control` | `awx/main/utils/common.py:571-582`；`task_manager_models.py:216-217` |
| AAP 文档页的字段描述 | AAP 2.5 "Managing Instance Groups" 页中 "Max concurrent jobs" 与 "Max forks" 的描述**看起来被互换了**（摘录自页面原文）；**以 AWX 模型 help_text 为准**：`max_concurrent_jobs` = 并发 job 数，`max_forks` = 总 fork 数（`ha.py:452-453`） | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-instance-groups |
| `default_container_group` | **未找到**：在 AWX 24.6.1 源码里搜索 `default_container_group` / `DEFAULT_CONTAINER_GROUP` 无结果；AWX operator 部署时默认执行队列由 `DEFAULT_EXECUTION_QUEUE_POD_SPEC_OVERRIDE` 等控制（`defaults.py:1068-1076`）。AAP 中同名概念：**未验证** | — |

**可操作结论（容器组）**：把容器组的 pod spec 的 `resources.requests` 调成与 forks 匹配，并设置 `max_forks` 为"集群允许的总 fork 数"；
一次 Pod 的内存 ≈ `M_main + F × m_fork`（5.3），按此填 requests/limits。

---

## 7. 风险 6：周期作业设置

### 7.1 Schedule（rrule）

- 调度器 `awx_periodic_scheduler`：对到期的 schedule 逐个 `create_unified_job(**job_kwargs)` 再 `signal_start()`
  （`awx/main/tasks/system.py:717-770`）。**这里没有"上一个还没跑完就跳过"的检查**。
- RRULE 约束（`docs/schedules.md:108-120`）：必须以 `DTSTART` 开头且带时区（`DTSTART;TZID=...` 或 `Z`）；至少一条 RRULE；必须带 `INTERVAL`；
  禁用 `FREQ=SECONDLY`、`RDATE`/`EXDATE`；`COUNT` 与 `UNTIL` 不能同时出现。例如：
  `DTSTART;TZID=Asia/Shanghai:20261021T020000 RRULE:FREQ=DAILY;INTERVAL=1`。

### 7.2 重叠与排队

- `allow_simultaneous`（UI: **Concurrent Jobs**）默认 False（`jobs.py:149`）。`job_blocked_by`：JT 的 job 若 `allow_simultaneous is False`，
  会被同一 JT 已有的 pending/running job 阻塞（`awx/main/scheduler/dependency_graph.py:88-95`），任务管理器给出
  `waiting for job-<id> to finish`（`task_manager.py:554-560`）。
  ⇒ 前一次没跑完时，后一次**排队等待**，不是跳过、也不是并行；积压的排队 job 会按顺序执行（长期超时会形成队列增长）。
- 切片：切片父 WorkflowJob 同样受 `workflow_job_blocked_by` 约束（`dependency_graph.py:97-106`，
  注释 "Sliced jobs can be either Job or WorkflowJob type, and either should block a sliced WorkflowJob"）；
  文档说明"Slicing overrides this behavior and implies allow_simultaneous"，指的是 slice 子 job 之间并行。
  父 WorkflowJob 是否继承 JT 的 `allow_simultaneous`：按源码推断是的，**未实测**。
- 对本 playbook：**不要勾选 Concurrent Jobs**（尤其在共享暂存目录时，见 3.5）；若多个 JT 并发，各用独立暂存目录（3.4）。

### 7.3 JT 其它字段

| 字段 | 本场景建议 / 行为 | 出处 |
|---|---|---|
| **Timeout** | 正数 = 秒；0 = 用全局设置（默认不限）；负数 = 真·不限。超时会杀 job，若恰在第③段删包之后、第④段上传之前，等同风险 2 | `job_templates.rst`（Timeout 行）；AAP 2.5 `controller-job-templates` 同义；`jobs.py:341,581` |
| **Limit** | `-l` 参数；逗号/冒号分隔。要保证 `lmt_server,localhost` 在内，**漏 localhost 的后果**：第②段不执行，若同时含 lmt_server，则第④段 `copy` 因源目录不存在而 `fatal`（实测 `Could not find or access '.../scan_result_packages/'`）；漏 lmt_server 的后果：只收不传 | `jobs.py:1006-1007`；实测 |
| **Job Slicing** | 见 2/8 节，**本 playbook 不建议** | — |
| **Prevent Instance Group Fallback** | 与指定的实例组/容器组配合使用，防止回退到没有挂载共享卷的 default 组 | `jobs.py:275-283,799-812` |
| **Enable Fact Storage** | 对本 playbook 无必要（4.3） | — |
| **Prompt on launch（Limit / Forks / Slice count 等）** | 建议关闭 Limit 的 prompt：允许手工填 Limit 就能漏掉 lmt_server | `mixins.py:112`（`ask_limit_on_launch`） |
| Execution Environment | 自定义 EE（ansible-core 2.13.10 / CentOS Stream 9）需包含 `pywinrm` 及 Kerberos 依赖（本仓库 `requirements.txt` / `bindep.txt`） | 仓库 |

### 7.4 Workflow 与 `job_slice_count`

- `job_slice_count > 1` 时，AWX 会为 JT 生成一个 **WorkflowJob**，每个 slice 是它的一个节点（`jobs.py:378-405`）；
  这些 slice 节点是**并行的根节点**，没有"汇聚"节点。要做"并行收集 → 汇聚 → 单次上传"必须用自己建的 Workflow Job Template（8.3）。
- 可在 Workflow 节点间使用 `Always` / `On Success` / `On Failure` 边，以及汇聚节点的 Convergence 规则（`workflow_templates.rst` 第 357-377 行）。
  使用 `Always` 边可让上传在部分收集失败时也执行。
- 把**带切片的 JT**作为 Workflow 节点：源码里 `WorkflowJobNode.get_job_kwargs` 只对"切片根节点"设置 `_prevent_slicing`（`workflow.py:382-386`），
  普通节点调用 `create_unified_job` 时会因 `slice_ct > 1` 生成嵌套 WorkflowJob；是否按预期工作 **未实测**。

### 7.5 失败 slice / 失败主机的重跑

- **整体 relaunch**：对 WorkflowJob 重跑会重建整个流程（`docs/workflow.md:136-142`）。
- **单个 slice 重跑**：`Job.copy_unified_job` 设置 `_prevent_slicing=True` 并沿用原 `job_slice_number/count`，即只重跑同一份 slice，不会重新生成 Workflow
  （`jobs.py:652-659`）。
- **Relaunch on failed hosts**（`POST /api/v2/jobs/N/relaunch/ {"hosts":"failed"}`）：
  limit 被替换为失败主机的名字（`awx/api/views/__init__.py:3588-3607`，`copy_kwargs['limit'] = ','.join(retry_host_list)`；`docs/retry_by_status.md`：
  "failed hosts" 与"有失败任务的主机"口径不同，不可达主机可无失败任务）。
  **limit 不含 lmt_server** ⇒ 第④段无主机 ⇒ 风险 2 复现。**对本 playbook 禁止使用该功能**，除非 lmt_server 恰好也在失败主机里；
  正确做法是整体重跑（幂等：端点上没删掉的包会再次 fetch；已删的包依赖暂存目录持久，见 3.3）。
- `fetch` 对相同内容是幂等的（本地 checksum 与远端相同则不传，`fetch.py:164-167`），整体重跑代价较低。

---

## 8. 风险 7：打包与收包解耦；架构对比

### 8.1 解耦的事实基础

端点上的产包由扫描器自己的 cron 完成，Ansible 只收（`docs/ilmt_data_flow_notes_zh.md` 第 1、5、6 节）。
`defaults/main.yml` 的说明：每日打包在安装后 23 小时起、此后按天（`lmt_scanner_daily_pack_results_enabled`），
时刻取决于安装时刻 —— 所以 4000 台的出包时间天然分散，**不存在"必须在某一时刻同时收"的要求**。
收集频率只要 ≥ 打包频率即可（一天一次足够；LMT 侧数据新鲜度的容忍度由你们的合规要求决定，**未验证**）。
由此，错峰收集（分批在窗口内依次跑）不会降低数据质量，只是最长延迟最多多出一个窗口宽度。

### 8.2 架构对比表

假设：N = 4000，混合 Windows/Linux/IBM i，lmt_server 是 `ansible_connection: local`（模板默认）。

| | A：单 job，大 forks，不切片 | B：多个 JT 按 group 错峰，每个带 `lmt_server,localhost` | C：Workflow：并行收集 → 汇聚 → 单次上传 | D：container group + 共享 PVC（部署形态，可叠加 A/B/C） |
|---|---|---|---|---|
| 要改 playbook? | 不要 | 不要（推荐加断言，9.2） | **最小形态不要**（8.3），推荐加断言 | 不要 |
| 要配置 | 持久暂存卷、forks、Timeout | 多个 JT / inventory group、每个 JT 独立暂存路径、错峰 Schedule | 两个 inventory、Workflow、共享 RWX 暂存、可选 slice | pod spec(PVC/资源)、`max_forks`、Prevent Fallback |
| 风险 1（切片） | 无（不切片） | 无（每个 job 都含 lmt_server） | 无（收集 job 里不放 lmt_server，上传 job 独立） | 取决于叠加的方案 |
| 风险 2（丢包） | 需持久暂存 | 需持久暂存或 per-JT 目录 + 失败自愈 | 必须共享持久暂存（跨 job） | **PVC 是前提** |
| 并发写冲突 | 无（单 job） | per-JT 目录则无 | 收集阶段多写者、上传阶段单读者且 workflow 顺序保证 | PVC 访问模式（RWX） |
| 控制节点瓶颈 | 单 job 主进程承载 4000 台，内存/事件大，**最差** | 每个 job 仅 N/k 台，主进程压力线性下降 | 每个收集 job 小；上传 job 轻 | Pod 级隔离，资源需按 forks 填 |
| 上传串行时间 | 一次性 `P × t_file`（≈ 33 分钟+） | 分摊到每个 job（每个 job 只传自己的包），总和不变但相互错峰 | 单次上传，仍 `P × t_file` | 同左 |
| 失败爆炸半径 | 一次失败重跑全部 | 只影响一个 group | 单个收集 job 失败不影响其它；上传可单独重跑 | 同左 |
| 运维复杂度 | 最低 | 低~中（JT 个数 = 批数） | 中（Workflow + 两套 inventory） | 中~高（K8s 存储与资源） |
| 失败提示 | 单个 job 状态 | 每个批次一条 | Workflow 视图 | — |
| 适合 | ≤ 数百台，或先试跑 | **~1000–5000 台，默认推荐** | 需要最大并行度和"单点上传"，且有 RWX 存储 | 部署在 K8s/OCP 上时 |

### 8.3 C 的最小形态（零改 playbook，已实测语义）

1. **两个 inventory**（或一个 inventory + 两个 Limit，但见下）：
   - `inv-collect`：只有端点，**不含 lmt_server**，作为切片的对象（`job_slice_count = k`）。
     第④段 `skipping: no hosts matched`，退出码 0（2.2 实测）。第①②段仍在隐式 localhost 上跑。
   - `inv-upload`：只有 `lmt_server`（同样的变量）。
   这样可以避免"lmt_server 落入某个 slice → 该 slice 的第④段在别的 slice 还在写时就把整个共享目录拷走并删掉"。
   如果用同一个 inventory + Limit `!lmt_server`，仅含 lmt_server 的那一份切片会因为 "no hosts" 失败（`job_slices.rst`）。
2. **共享且持久**的暂存路径，两个 JT 用同一个 `lmt_local_file_storage_path`（JT extra_vars），RWX 卷或 NFS。
3. **Workflow**：`collect-JT(sliced)` →（`Always` 边）→ `upload-JT`（用 `lmt_collect_results.yml`，Limit `lmt_server,localhost`，inventory = `inv-upload`）。
   上传 JT 的第③段会对 lmt_server 本身再跑一次 `collect_scan_results_unix`（`find` 在不存在的路径只给 warning，`matched=0`，实测不报错）。
   这是无害的；如果 lmt_server 上本来装了扫描器，反而会顺带收它自己的包。
4. Workflow 本身 `Allow simultaneous` 保持关闭；schedule 设在 Workflow 上。
5. 其余约束：收集 JT 之间同时写同一目录，`fetch` 目标文件名含主机名和时间戳，不会冲突；上传 JT 运行时收集 JT 必然已结束，所以"拷整个目录再删目录"是安全的。
   注意：**不能**让上一次 Workflow 的上传与下一次 Workflow 的收集重叠（保持 `allow_simultaneous=false` 与足够的 Schedule 间隔）。

C 的优化版（**需改 playbook**）：把上传改为"打包后一次性传输再解包"（实测：本机 100 个文件，`copy` 51 s，`tar+unarchive` 2.7 s），
但 `unarchive` 会让 datasource 里一次性出现很多文件，LMT 是否会在中途扫描并读到半个文件：**未验证**。

### 8.4 推荐与理由

1. **对本 playbook，切片不适用**（风险 1、2、3 都与之相关；官方也警告跨 host 编排类 job 不要切片）。切片带来的是"在同一个 job 里加速"，
   而本 playbook 的第④段需要看到全部包；B/C 达到同样并行度而不破坏不变量。
2. **推荐 B 起步**：把 inventory 按站点/区域/OS 切成 k 个 group（建议每批约 500–1000 台，依试点定，见第 10 节），每批一个 JT，
   Limit = `lmt_server,localhost,<group>`，Schedule 错开 15–30 分钟。理由：
   - 控制节点主进程内存/调度成本随批大小线性下降（5.2：200→1000→4000 台，主进程 RSS 55 → 100 → 620 MB，单位 host-task 成本 27→43 ms）；
   - 单批失败只重跑该批；
   - 零代码改动；
   - 每个 JT 自己的暂存目录让"拷整个目录再删目录"安全且自愈。
3. **当收集阶段需要比 B 更高的并行度，或者要求"只有一次上传"时升级到 C**（8.3）。
4. **A 仅作为试点/小规模**：4000 台单 job 的主进程内存与事件量最大、一次失败代价最大。
5. **D 不是一个独立选项**：如果你们本来就跑在 K8s/OCP 上，D 就是 B 或 C 的运行底座，其核心是 PVC + pod spec + `max_forks`。

---

## 9. 哪些"配置即可"，哪些"需要改 playbook"

### 9.1 配置即可

| 目的 | 配置 |
|---|---|
| 不丢包 | 持久暂存卷（3.3）；`lmt_local_file_storage_path` 指向它（JT extra_vars）；Prevent Instance Group Fallback；专用实例组 |
| 避免并发冲突 | JT `allow_simultaneous=false`；按 JT 隔离暂存目录 `/…/{{ awx_job_template_id }}`（3.4） |
| 确保 lmt_server 在场 | 每个批次一个 JT，Limit 固定写死 `lmt_server,localhost,<group>`，**关闭 Limit 的 prompt on launch**；或每批一个自带 lmt_server 的 inventory（结构性保证） |
| 禁止的操作 | 对本 JT 使用 "Relaunch on failed hosts"、对本 JT 使用 `job_slice_count`、手工改 Limit |
| 控制节点保护 | 合适的 forks（公式见 5.3）；实例组/容器组的 `max_forks`、`max_concurrent_jobs`；容器组 pod spec 里按 forks 配 requests/limits |
| 错峰 | 多个 Schedule 的 DTSTART 间隔 |
| 失败可见性 | 为 JT 配置失败通知 |

### 9.2 需要改 playbook（建议，均为我在 `/tmp/lmt_exp` 下实测过的草图，**没有改动仓库**）

1. **fail-closed 预检**（实测可用，`/tmp/lmt_exp/wrap/repo/lmt_collect_results_safe.yml`）：在原 playbook 前加一个新的包装 playbook，
   用 `import_playbook: lmt_collect_results.yml` 复用原文件。预检做两件事：
   - 在 `lmt_server` 上 `stat` datasource 目录并 `set_fact: lmt_server_in_job: true`；
   - 在 `localhost` 上 `assert hostvars.get('lmt_server', {}).get('lmt_server_in_job', false)`，否则 `fail`。
   实测：Limit 不含 lmt_server 时，在**任何端点包被 fetch 并删除之前**就失败，端点包原封不动；完整运行则正常上传。
   同一个包装 playbook 可以再加"暂存目录是挂载点"的断言（例如在目标节点用 `command: findmnt` 检查），**未实现/未验证**。
2. **删除端点包推迟到上传成功之后**（实测可用，`/tmp/lmt_exp/dau`）：把收集任务文件里的 "Clean up" 拆出去，
   在第④段之后再对端点执行清理，条件是 `hostvars['lmt_server'].lmt_upload_ok`。实测：
   - 成功路径：datasource 收到包，端点包被清理；
   - 失败路径（datasource 目录不存在）：端点上的包**保留**，暂存目录里也有一份，下次重收。
   代价：端点上的包多留到上传之后；`fetch` 是幂等的所以重收不会重复传输。需要改 `roles/ilmt_scanner/tasks/collect_scan_results_*.yml`
   并新增一个清理任务文件，且 Windows / IBM i 各一份。这是**真正消除风险 2 的方案**（不依赖持久卷）。
3. **上传改为批量**：`copy` 逐文件 → `tar` + `unarchive` 或 `copy remote_src` + `synchronize`（后者在 `ansible.posix` 里，EE 需安装），见 5.2e 与 8.3 的未验证项。
4. **若坚持使用切片**：lmt_server 改为 `add_host` 动态主机，第④段 `gather_facts: true`，每个 slice 独立暂存并各自上传（2.3 实测）。
   仍然需要共享 datasource 的并发写入语义确认（3.5）。

---

## 10. 落地验证步骤（先小批量试跑）

0. **离线验证（不连端点）**：对准备好的 inventory，用 `ansible-playbook lmt_collect_results.yml -i <inv> -l <limit> --list-hosts`
   检查第①~④段命中的主机：第③段应含 lmt_server 与目标端点，第④段应命中 lmt_server（本文所有 `--list-hosts` 结果均可在 `/tmp/lmt_exp` 复现）。
1. **一个 20–50 台的试点 group**（Windows / Linux / IBM i 各占一部分），JT forks=10，使用最终的暂存配置：
   记录 AWX 作业详情里的总时长、各任务耗时；计算 `t_host ≈ 总时长 × F / N`；记录 EE/Pod 的峰值内存（`kubectl top pod` 或 `podman stats`）。
2. **放大**：200 → 500 → 1000 台，每次记录：主进程 RSS（对应 5.2b 的本机数字）、事件数、WinRM 端点耗时占比、包大小分布。
   用 5.3 的公式反推 F 与批大小，直到总时间落入窗口。
3. **故障演练**（每项都确认"端点包没丢"）：
   - 手工取消 job（第③段后、第④段前）；
   - 把 lmt_server 的 `ansible_host` 改错；
   - 把 `lmt_server_datasource_path` 改错；
   - 把 Limit 里的 lmt_server 去掉（应由 9.2-1 的断言拦下；没改 playbook 则观察到的是丢包 ⇒ 说明必须先落地 9.1 的防护）；
   - 关闭 Prevent Instance Group Fallback 并让主组满载，观察是否回退到没挂载的节点。
4. **重叠演练**：把 Schedule 间隔设得短于运行时长，确认第二次排队为 `waiting for job-…`，且没有并行写同一暂存目录。
5. **LMT 侧验证**：确认 LMT 对 datasource 里并发写入、`.ansible_tmp*` 隐藏文件、部分上传的行为（本文**未验证**）。
6. 通过后再上线全量，并在上线首周对比 LMT 侧"最近一次上传时间"分布，确认没有端点长期滞后。

---

## 11. 经验判断（非出处）

以下是我的判断，没有一手来源，仅供参考：

- 4000 台、一天一次的收集，我倾向于 **6–8 批、每批 500–700 台、forks 25–40、间隔 20–30 分钟**，总窗口 2–3 小时。起始数字只是试点前的占位值，不是推荐值。
- 对 Windows 端点，我预期 WinRM 往返会占 `t_host` 的大头（块大小 0.5 MB 的 base64 读取，每块一次 PowerShell）；如果 Windows 占比高，宁可降低 F 而不是增加控制节点 CPU。
- 控制节点 CPU 在这个负载下通常先于内存成为瓶颈（单线程主进程 + 每个 fork 一个短命子进程）；AWX 的 `SYSTEM_TASK_FORKS_CPU=4` 对它可能偏乐观。
- IBM i 官方不支持（README），本文未对 IBM i 的 `fetch` 行为做任何实验；它们的 SSH 连接很可能是整体吞吐的长尾。
- 数据量上，每台每天一个包的假设可能是对的，但周扫描日会有 2 个包；`P` 可能是 `N` 到 `2N`。

---

## 12. 本次做过的实验（均在 `/tmp/lmt_exp`，未修改仓库、未连外部主机、未 commit）

| 实验 | 目的 | 结论 |
|---|---|---|
| 按 AWX 算法切 3 份 inventory，逐份运行 playbook | 风险 1 | 不含 lmt_server 的 slice 第④段 `no hosts matched`，退出码 0，包滞留 |
| `--list-hosts` 对各 slice、各 `-l` 组合 | 风险 1、6 | `hosts: all` 不含隐式 localhost；`-l` 不含任何匹配时报 `leaves us with no hosts to target`（退出码 1） |
| 只跑第④段 / 删除缓存 / jsonfile cache 命中 | 风险 3 | 无 facts 则 `== / !=` 都 fatal；缓存命中则正常 |
| `add_host` 动态 lmt_server，3 slice 并发 | 切片替代方案 | 需 `gather_facts: true`；各自上传成功 |
| lmt_server 连不上 / datasource 目录缺失 | 风险 2 | 端点包已删，包留在暂存目录，退出码 4 / FAILED |
| 端点不可达时第④段是否执行 | 风险 2 | 仍执行，退出码 4 |
| 相对路径 `./lmt_file_storage` 的解析位置 | 风险 2 | 相对 playbook 目录，不是 cwd |
| 变量作用域（group vars / `-e`） | 3.3 | group vars 对 localhost / lmt_server 不生效；`-e` 全生效 |
| 4000 台 local 假主机，forks 5/20/100；250–2000 台 forks=20 | 风险 4 | 见 5.2a/b |
| 完整第①②③段对 200 台 local 假端点，forks 10/40 | 风险 4 | 46 s / 45 s，无差别 |
| `copy` 目录拷贝 100 个 20 KB 文件；`remote_src`；`tar+unarchive` | 风险 4 | 51 s / 54.7 s（重复）/ 1.4 s / 2.7 s；1000 个文件的测试因超时被中止 |
| fail-closed 包装 playbook、"上传后再删端点包" | 9.2 | 均可用 |
| 空闲路径 `find`、`fetch` 目标目录自动创建 | 辅助 | `find` 对不存在路径只 warning；`fetch` 会自建目录 |

未能做的：真实 SSH/WinRM 往返、真实 AWX 实例（切片、容量、Schedule、Relaunch 均为**源码阅读**，**未实测**）、K8s PVC 与 pod spec 合并（**未实测**）、
LMT 对 datasource 并发写入/隐藏临时文件/残包的行为（**未验证**）。
实验过程中有两次删除命令被沙箱的安全检查拦截（相对路径通配符），改为不删除、改用新目录继续，不影响结论。

---

## 13. 来源索引

**AWX 24.6.1（https://github.com/ansible/awx/tree/24.6.1 ，commit 94e5795）**
- 切片：`awx/main/models/jobs.py`（`get_effective_slice_ct` 336、`create_unified_job` 378、`copy_unified_job` 652、`_get_task_impact` 690、`preferred_instance_groups` 799、`get_hosts_for_fact_cache` 843）；
  `awx/main/models/inventory.py`（`get_sliced_hosts` 287、`get_script_data` 306）；`awx/main/models/workflow.py`（382-386、714）；`awx/main/tasks/jobs.py`（`build_inventory` 324）。
- 容量/调度：`awx/main/models/ha.py`（176、206、321、407、452-455）；`awx/main/scheduler/task_manager_models.py`（81、99、209）；
  `awx/main/scheduler/task_manager.py`（554、578、603-612）；`awx/main/scheduler/dependency_graph.py`（88-106）；
  `awx/main/utils/common.py`（571、750、824、1113）；`awx/settings/defaults.py`（57-65、996、1068-1076、1087）；`awx/main/conf.py`（277、377、513、530）。
- 运行时：`awx/main/tasks/jobs.py`（90、181、209、581、998-1007、1031、1363）；`awx/main/tasks/receptor.py`（600-686）；
  `awx/main/utils/execution_environments.py`（32-52）；`awx/main/tasks/facts.py`；`awx/main/models/unified_jobs.py`（1538、1554）。
- 调度/重跑：`awx/main/tasks/system.py`（717-770）；`awx/api/views/__init__.py`（3588-3607）。
- 文档（同仓库）：`docs/job_slicing.md`、`docs/capacity.md`、`docs/fact_cache.md`、`docs/clustering.md`、`docs/container_groups.md`、`docs/schedules.md`、`docs/retry_by_status.md`、`docs/workflow.md`、
  `docs/docsite/rst/userguide/{job_slices,job_templates,job_capacity,workflow_templates,execution_environments}.rst`、
  `docs/docsite/rst/administration/{containers_instance_groups,performance}.rst`。

**ansible-core v2.13.13（https://github.com/ansible/ansible/tree/v2.13.13 ，commit c2b0ea5）**
- `lib/ansible/config/base.yml`（98 pipelining、650 forks、658-672 gathering）；`lib/ansible/executor/task_queue_manager.py`（117、294、300-309）；
  `lib/ansible/executor/playbook_executor.py`（207）；`lib/ansible/plugins/strategy/__init__.py`（405-418）；`lib/ansible/plugins/strategy/free.py`；
  `lib/ansible/plugins/action/fetch.py`；`lib/ansible/plugins/action/copy.py`（485-519）；`lib/ansible/modules/copy.py`（125、709）；
  `lib/ansible/module_utils/basic.py`（1653）；`lib/ansible/plugins/connection/{ssh,winrm,psrp}.py`。

**Ansible 官方文档**
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html
- https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html
- https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html（`--limit`；**未找到** 关于"隐式 localhost 不被 `all` 匹配"的原文，该结论来自实测）
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html
- https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html

**Red Hat AAP 2.5（docs.redhat.com，WebFetch 摘录）**
- Job slicing：https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-job-slicing
- Job templates：…/controller-job-templates
- Instance groups：…/controller-instance-groups（描述互换，见 6.3）
- Jobs（容量公式）：…/controller-jobs

**其它一手来源**
- ansible-runner：https://github.com/ansible/ansible-runner/blob/devel/docs/intro.rst
- Kubernetes PV Access Modes：https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Microsoft WinRM 默认配置：https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management
- OpenSSH `sshd_config`：https://man.openbsd.org/sshd_config

**仓库内**
- `lmt_collect_results.yml`、`roles/ilmt_scanner/tasks/{collect_scan_results_*,upload_to_server_*,create_scan_result_packages_folder}.yml`、
  `roles/ilmt_scanner/defaults/main.yml`、`lmt_inventory_template.yml`、`execution-environment.yml`、
  `docs/{doc_schedule_collection,doc_automating_with_awx_tower,ilmt_data_flow_notes_zh}.md`。
