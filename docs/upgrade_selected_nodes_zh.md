# 只对部分节点安装或升级扫描器（中文说明）

> 适用于 `lmt_install_or_upgrade_scanner.yml`。结论来自仓库代码走读，以及用 ansible-core 2.15.13
> 对临时 inventory 执行 `--list-hosts` 的实测（不连接任何端点）。
> 标注「未核对」的部分没有读过对应代码，请自行验证。

---

## 1. 一句话结论

用 `-l`（`--limit`）限定范围，并且**必须把 `lmt_server` 一起带上**：

```bash
ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml \
  -l lmt_server,unix1,windows1
```

---

## 2. 为什么必须带 `lmt_server`

playbook 由三个 play 组成，每个 play 的 `hosts` 不同，而 `-l` 对所有 play 同时生效：

| # | hosts | 作用 |
|---|---|---|
| ① | `localhost` | 打印 playbook 版本 |
| ② | `lmt_server` | 把 LMT 服务器上最新的扫描器安装包刷新到控制节点缓存 |
| ③ | `all` | 在端点上安装或升级 |

如果 `-l` 里没有 `lmt_server`，第②段匹配 0 台主机，缓存不会刷新。实测：

```
-l unix1
  play #2 (lmt_server): hosts (0)        ← 缓存没有刷新
  play #3 (all):        unix1
```

后果取决于缓存当时的状态：

- 缓存是旧的：端点被升级到缓存里的旧版本，或者认为无需升级而跳过。
- 缓存不存在：第③段拿不到 `version.txt` / `build_date.txt`，安装包文件名拼不出来，任务失败。
  （第③段对缓存文件用的是 `lookup('file', ..., errors='ignore')`，见 `install_scanner_unix.yml`。）

---

## 3. 常用写法

下表结果均为实测（`--list-hosts`）。示例 inventory：主机 `lmt_server`、`unix1`、`unix2`、`win1`，
组 `batch1` 含 `unix1`、`win1`。

| 目的 | `-l` 取值 | 第②段 | 第③段 |
|---|---|---|---|
| 指定几台 | `lmt_server,unix1` | lmt_server | lmt_server, unix1 |
| 指定一个组 | `lmt_server,batch1` | lmt_server | lmt_server, unix1, win1 |
| 通配符 | `lmt_server,unix*` | lmt_server | lmt_server, unix1, unix2 |
| 从文件读主机名 | `lmt_server,@hosts.txt` | lmt_server | lmt_server + 文件中的主机 |
| 排除某个组 | `all:!batch1` | lmt_server | lmt_server, unix2 |
| ❌ 漏掉 lmt_server | `unix1` | **（空）** | unix1 |

`hosts.txt` 每行一个主机名，例如：

```
unix2
win1
```

### 注意：不要混用 `,` 和 `:`

下面两种写法**不会得到想要的结果**：

| 写法 | 实际结果 |
|---|---|
| `lmt_server,all:!batch1` | 警告 `Could not match supplied host pattern`，只剩 lmt_server |
| `lmt_server,!batch1` | 排除不生效，只剩 lmt_server |

要排除某个组，直接写 `all:!batch1`。`lmt_server` 不在被排除的组里，会自动保留。
前提是 `lmt_server` 确实不属于该组；如果属于，排除后第②段会变空。

---

## 4. `lmt_server` 自己也会被处理

`lmt_server` 是 `all` 的成员，所以带上它之后，第③段也会在 `lmt_server` 节点上执行安装或升级。
这和不加 `-l` 的全量运行行为一致，不是 `-l` 引入的新行为。

inventory 模板里 `lmt_server` 是 `ansible_host: localhost`、`ansible_connection: local`，
也就是说控制节点本机会被当作一个端点处理。如果你不希望在这台机器上装扫描器，需要另外处理，
本文不展开。

---

## 5. 重复运行是安全的

Unix/Linux 的 `install_scanner_unix.yml` 只在以下条件之一成立时才会执行安装块：

- 端点上还没有安装扫描器；
- 控制节点缓存里的版本，高于端点上已有的 `scanner_version`。

已经是最新版的端点会被跳过。所以分批运行时，不必担心重复覆盖已升级的节点。

缓存刷新也是增量的：只有 LMT 服务器上的版本更高，或版本相同但 build 不同，才会清空并重新下载。
（见 `refresh_installers_cache_unix.yml`。）

Windows 和 IBM i 的对应任务文件没有逐行核对，行为预期类似，请以实际运行为准。

---

## 6. 建议的操作流程

1. **预览范围**，不连接端点：

   ```bash
   ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml \
     -l lmt_server,batch1 --list-hosts
   ```

   检查两点：第②段是否显示 `lmt_server`，第③段是否是预期的主机。

2. **先小范围试点**（一两台），确认版本、配置、首次扫描都正常。

3. **再分批铺开**。可以在 inventory 里建 `batch1`、`batch2` 之类的组：

   ```bash
   for g in batch1 batch2 batch3; do
     ansible-playbook lmt_install_or_upgrade_scanner.yml -i lmt_inventory.yml -l lmt_server,$g
   done
   ```

   分批的另一个原因：升级会立即触发一次扫描，需要靠分批控制并发。
   详见 [ilmt_data_flow_notes_zh.md](ilmt_data_flow_notes_zh.md) 中的「分批升级」一节。

---

## 7. AWX / Automation Controller

- 作业模板的 **Limit** 同样要写成 `lmt_server,...`。
- **不要使用 "Relaunch on failed hosts"**：它生成的 limit 只包含失败主机，不含 `lmt_server`，
  会让第②段变空。分析见 [ansible_scaling_collect_results_notes_zh.md](ansible_scaling_collect_results_notes_zh.md)。
- 建议关闭 Limit 的 "Prompt on launch"，避免手工填写时漏掉 `lmt_server`。
