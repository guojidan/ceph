# Ceph OSD 灾难恢复工具使用指南

**版本**: Ceph 14.2.22 (Nautilus)
**工具**: ceph-objectstore-tool
**适用场景**: OSD 无法启动的各种灾难场景

---

## 目录

1. [功能概述](#功能概述)
2. [Superblock 修复](#superblock-修复)
3. [RocksDB SST 修复](#rocksdb-sst-修复)
4. [OSDMap 恢复](#osdmap-恢复)
5. [磁盘满恢复](#磁盘满恢复)
6. [故障诊断流程](#故障诊断流程)

---

## 功能概述

### 四大核心功能

| 功能 | 适用场景 | 恢复时间 | 数据风险 |
|------|----------|----------|----------|
| **Superblock 修复** | Superblock 损坏 | 5-10 秒 | 低 |
| **RocksDB SST 修复** | RocksDB 数据库损坏 | 5-60 分钟 | 中 |
| **OSDMap 恢复** | OSDMap 丢失 | 1-5 分钟 | 低 |
| **磁盘满恢复** | BlueFS 空间耗尽 | 1-10 分钟 | 低-中 |

### 工具位置

```bash
# 工具路径
/usr/bin/ceph-objectstore-tool

# 或编译后的路径
/path/to/ceph/build/bin/ceph-objectstore-tool
```

---

## Superblock 修复

### 什么是 Superblock？

Superblock 是 OSD 的元数据结构，包含：
- **cluster_fsid**: 集群 UUID
- **osd_fsid**: OSD UUID
- **whoami**: OSD 编号
- **current_epoch**: 当前 epoch
- **oldest_map**: 最旧的 OSDMap epoch
- **newest_map**: 最新的 OSDMap epoch
- **past_intervals**: 历史区间信息

### 损坏症状

OSD 启动失败，日志显示：
```
-1 osd.0 0 OSD::init : unable to read osd superblock
-1 osd.0 0 OSD::mkfs: ObjectStore::mkfs failed with error -5
```

### 使用方法

#### 1. 自动模式（推荐）

自动从 BlueStore 检测 OSD FSID：

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op repair-superblock \
  --osd-id 0 \
  --cluster-fsid <cluster-fsid> \
  --current-epoch <epoch> \
  --force
```

#### 2. 手动模式

fsid 文件也损坏时，手动指定：

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op repair-superblock \
  --osd-id 0 \
  --cluster-fsid <cluster-fsid> \
  --osd-fsid <osd-fsid> \
  --current-epoch <epoch> \
  --force
```

### 参数说明

| 参数 | 必需 | 说明 | 获取方式 |
|------|------|------|----------|
| `--data-path` | 是 | OSD 数据目录 | `/var/lib/ceph/osd/ceph-N` |
| `--osd-id` | 是 | OSD 编号 | 从路径获取 |
| `--cluster-fsid` | 是 | 集群 UUID | `ceph fsid` |
| `--osd-fsid` | 否 | OSD UUID | 自动检测或手动指定 |
| `--current-epoch` | 是 | 当前 epoch | `ceph osd dump \| grep epoch` |
| `--force` | 是 | 强制执行 | - |

### 获取参数示例

```bash
# 1. 获取集群 FSID
ceph fsid
# 输出: 5127c8ec-df59-4a92-96ea-20b20f285cc7

# 2. 获取当前 epoch
ceph osd dump | grep ^epoch
# 输出: epoch 42

# 3. 获取 OSD FSID（如果 fsid 文件存在）
cat /var/lib/ceph/osd/ceph-0/fsid
# 输出: 2c2578c9-eb87-4585-ab94-38f66e7d4d27
```

### 执行示例

```bash
# 完整命令
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op repair-superblock \
  --osd-id 0 \
  --cluster-fsid 5127c8ec-df59-4a92-96ea-20b20f285cc7 \
  --current-epoch 42 \
  --force

# 输出:
# Attempting to rebuild OSD superblock...
#
# Detected OSD FSID: 2c2578c9-eb87-4585-ab94-38f66e7d4d27
#
# Superblock parameters:
#   cluster_fsid: 5127c8ec-df59-4a92-96ea-20b20f285cc7
#   osd_fsid: 2c2578c9-eb87-4585-ab94-38f66e7d4d27
#   whoami: 0
#   current_epoch: 42
#
# Successfully rebuilt superblock
# Superblock saved to ObjectStore
```

### 验证修复

```bash
# 启动 OSD
systemctl start ceph-osd@0

# 检查状态
ceph osd tree
```

### 设计实现

#### 核心逻辑

```cpp
int rebuild_superblock(ObjectStore *store, ...) {
  // 1. 尝试从 BlueStore/fsid 文件读取 OSD FSID
  uuid_d detected_fsid;
  if (!manual_osd_fsid.is_zero()) {
    osd_fsid = manual_osd_fsid;  // 手动指定
  } else if (read_osd_fsid_from_file(data_path, &detected_fsid) == 0) {
    osd_fsid = detected_fsid;  // 自动检测
  }

  // 2. 构建 Superblock
  OSDSuperblock superblock;
  superblock.cluster_fsid = cluster_fsid;
  superblock.osd_fsid = osd_fsid;
  superblock.whoami = osd_id;
  superblock.current_epoch = current_epoch;
  superblock.oldest_map = 1;
  superblock.newest_map = current_epoch;
  superblock.clean_thru = 0;  // 触发完整 peering

  // 3. 写入 ObjectStore
  ObjectStore::Transaction t;
  t.create_collection(coll_t::meta(), 0);
  bufferlist bl;
  encode(superblock, bl);
  t.write(coll_t::meta(), OSD_SUPERBLOCK_GOBJECT, 0, bl.length(), bl);
  store->queue_transaction(ch, std::move(t));

  // 4. 如果 fsid 文件丢失，重建它
  if (errno == ENOENT) {
    write_fsid_file(data_path, osd_fsid);
  }

  return 0;
}
```

#### 关键特性

1. **自动检测 OSD FSID**: 从 `<data-path>/fsid` 文件读取
2. **fsid 文件重建**: 文件丢失时自动创建
3. **BlueStore 标签修复**: 自动修复 block 设备标签
4. **完整 peering**: `clean_thru=0` 触发完整的 peering 过程
5. **错误处理**: 完善的错误检查和回滚

### 注意事项

1. ✅ **修复后 epoch 为指定值**，不是真实的历史 epoch
2. ✅ **past_intervals 信息丢失**，修复后建议执行 `fetch-osdmaps`
3. ✅ **必须先停止 OSD** 再执行修复
4. ⚠️ **需要准确的 cluster_fsid** 和 **current_epoch**

---

## RocksDB SST 修复

### 什么是 RocksDB SST？

RocksDB 是 BlueStore 用来存储元数据的键值数据库。SST (Sorted String Table) 是 RocksDB 的数据文件。

**SST 文件存储的内容**:
- Object 元数据（名称、大小、属性等）
- Collection 信息
- Onode 数据
- 分配器位图

### 损坏症状

OSD 启动失败，日志显示：
```
-1 rocksdb: Corruption: block checksum mismatch
-1 rocksdb: Corruption: Bad table magic number
-1 bluestore(/var/lib/ceph/osd/ceph-0) _open_db error opening db: Corruption
```

### 两步操作

#### 步骤 1: 扫描损坏

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op scan-rocksdb-corruption
```

**输出示例**:
```
Scanning RocksDB for corrupted SST files using RocksDB API...
Checking 000004.sst... OK
Checking 000007.sst... CORRUPTED (checksum error: Bad table magic number)
Checking 000010.sst... OK

Found 1 corrupted SST file(s):
  - 000007.sst
```

#### 步骤 2: 自动修复

**Dry-run 预览**（推荐先执行）:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op auto-repair-rocksdb \
  --dry-run
```

**实际修复**:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op auto-repair-rocksdb \
  --keep-corrupted \
  --temp-dir /data/tmp
```

### 参数说明

| 参数 | 必需 | 说明 |
|------|------|------|
| `--data-path` | 是 | OSD 数据目录 |
| `--op` | 是 | 操作类型 (`scan-rocksdb-corruption` 或 `auto-repair-rocksdb`) |
| `--dry-run` | 否 | 预览模式，不修改数据 |
| `--keep-corrupted` | 否 | 保留损坏文件的备份 |
| `--temp-dir` | 否 | 临时目录（默认 `/tmp`） |

### 完整修复流程

```bash
# 1. 停止 OSD
systemctl stop ceph-osd@0

# 2. 扫描损坏
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op scan-rocksdb-corruption

# 3. Dry-run 预览
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op auto-repair-rocksdb --dry-run

# 4. 执行修复
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op auto-repair-rocksdb --keep-corrupted

# 5. 启动 OSD
systemctl start ceph-osd@0

# 6. 检查日志
tail -f /var/log/ceph/ceph-osd.0.log
```

### 修复输出示例

```
=== RocksDB Auto Repair ===

Step 1: Scanning for corrupted SST files...
Found 1 corrupted SST file(s):
  - 000007.sst

Step 2: Preparing temporary directory...
Working directory: /tmp/rocksdb_repair_12345

Step 3.1: Processing 000007.sst...
  - Exporting corrupted file...
  - Repairing SST file using RocksDB API...
    Repaired keys: 8542 / 9834 (13.1% lost)
  - Backing up original file...
    Original saved as 000007.sst.backup.12345.1634567890.0
  - Importing repaired file...
  ✓ Successfully repaired

Step 4: Syncing BlueFS to disk...
  ✓ BlueFS synced successfully

Step 5: Cleaning up temporary files...

=== Repair Summary ===
Total corrupted files: 1
Successfully repaired: 1
Failed to repair: 0

IMPORTANT: Please restart the OSD to verify the repair.
Monitor OSD logs for any RocksDB errors.
```

### 设计实现

#### 修复流程

```cpp
int auto_repair_rocksdb(ObjectStore *store, ...) {
  // 1. 扫描损坏的 SST 文件
  vector<string> corrupted_files;
  scan_rocksdb_corruption(store, corrupted_files);

  // 2. 创建临时目录
  string work_dir = temp_dir + "/rocksdb_repair_" + stringify(getpid());
  mkdir(work_dir.c_str(), 0755);

  // 3. 修复每个损坏的文件
  for (const auto& sst_file : corrupted_files) {
    // 3.1 从 BlueFS 导出损坏文件
    export_bluefs_file(bluefs, sst_file, corrupted_path);

    // 3.2 使用 RocksDB API 修复
    repair_sst_file(corrupted_path, repaired_path);

    // 3.3 备份原文件（可选）
    if (keep_corrupted) {
      bluefs->rename("db", sst_file, "db", backup_name);
    }

    // 3.4 导入修复后的文件
    import_bluefs_file(bluefs, repaired_path, sst_file);
  }

  // 4. 同步 BlueFS 到磁盘
  if (repaired_count > 0 && !dry_run) {
    bluefs->sync_metadata(false);
  }

  // 5. 清理临时文件
  safe_exec({"rm", "-rf", work_dir});

  return 0;
}
```

#### 核心函数

**1. scan_rocksdb_corruption()** - 扫描损坏
```cpp
// 使用 RocksDB API 验证 checksum
rocksdb::SstFileReader sst_reader(opts);
rocksdb::Status status = sst_reader.Open(temp_file);

rocksdb::ReadOptions ropts;
ropts.verify_checksums = true;  // 强制验证
std::unique_ptr<rocksdb::Iterator> it(sst_reader.NewIterator(ropts));

// 遍历所有数据块
it->SeekToFirst();
while (it->Valid()) {
  it->Next();
  if (!it->status().ok()) {
    is_corrupted = true;  // 发现损坏
    break;
  }
}
```

**2. repair_sst_file()** - 修复文件
```cpp
// 打开损坏的 SST 读取
rocksdb::SstFileReader reader(opts);
reader.Open(input_path);

// 创建新的 SST 写入
rocksdb::SstFileWriter writer(env_opts, write_opts);
writer.Open(output_path);

// 关闭 checksum 验证，读取可访问的数据
rocksdb::ReadOptions ropts;
ropts.verify_checksums = false;  // 允许读取部分损坏的数据

// 复制有效的 key-value
uint64_t valid_keys = 0, skipped_keys = 0;
auto it = reader.NewIterator(ropts);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  status = writer.Put(it->key(), it->value());
  if (status.ok()) {
    valid_keys++;
  } else {
    skipped_keys++;
  }
}

// 完成写入
writer.Finish();

// 报告数据丢失
double loss_ratio = (double)skipped_keys / (valid_keys + skipped_keys);
if (loss_ratio > 0.5) {
  cerr << "WARNING: More than 50% data loss!" << endl;
}
```

**3. import_bluefs_file()** - 导入到 BlueFS
```cpp
// 读取本地文件
bufferlist bl;
bl.read_file(local_path.c_str(), &error);

// 写入 BlueFS
BlueFS::FileWriter* writer;
bluefs->open_for_write("db", target_name, &writer, false);
writer->append(bl);

// fsync 确保持久化
int r = bluefs->fsync(writer);
if (r < 0) {
  bluefs->close_writer(writer);
  bluefs->unlink("db", target_name);  // 失败时清理部分文件
  return r;
}

bluefs->close_writer(writer);
```

#### 关键优化

1. **临时文件清理** (行 1593):
   ```cpp
   // 使用 rm -rf 安全清理，避免 rmdir() 失败
   vector<string> rm_args = {"rm", "-rf", temp_dir};
   safe_exec(rm_args);
   ```

2. **BlueFS 持久化** (行 1952):
   ```cpp
   // 确保修复操作持久化到磁盘
   bluefs->sync_metadata(false);
   ```

3. **导入原子性** (行 1674-1686):
   ```cpp
   // fsync 失败时清理部分文件
   if (bluefs->fsync(writer) < 0) {
     bluefs->unlink("db", target_name);
   }
   ```

4. **数据丢失警告** (行 1806-1824):
   ```cpp
   // 显示数据丢失百分比
   double loss_ratio = (double)skipped_keys / total_keys;
   cout << "(" << (loss_ratio * 100) << "% lost)" << endl;

   if (loss_ratio > 0.5) {
     cerr << "WARNING: More than 50% data loss!" << endl;
   }
   ```

5. **OOM 保护** (行 1522-1528):
   ```cpp
   // 跳过大于 1GB 的文件，避免内存耗尽
   const uint64_t MAX_SST_SIZE = 1ULL * 1024 * 1024 * 1024;
   if (size > MAX_SST_SIZE) {
     cout << "SKIPPED (file too large)" << endl;
     continue;
   }
   ```

6. **文件命名冲突避免** (行 1503-1511):
   ```cpp
   // 使用递增索引而非时间戳，避免冲突
   size_t file_index = 0;
   string temp_file = temp_dir + "/" + file + "." + stringify(file_index++);
   ```

### 数据丢失风险

| 损坏程度 | 数据丢失 | 修复成功率 | 建议 |
|----------|----------|------------|------|
| **轻微** (< 10% 损坏) | < 5% | 95%+ | 直接修复 |
| **中等** (10-50% 损坏) | 10-30% | 70-90% | 修复后验证 |
| **严重** (> 50% 损坏) | 30-70% | 30-70% | 考虑使用备份 |

### 注意事项

1. ⚠️ **修复会丢失部分数据** - 损坏的 key-value 对会被跳过
2. ⚠️ **修复时间较长** - 大型 SST 文件可能需要几十分钟
3. ⚠️ **需要足够空间** - 临时目录需要能容纳 SST 文件
4. ⚠️ **最大文件限制** - 单个 SST 文件不能超过 1GB
5. ✅ **支持 dry-run** - 建议先预览再执行
6. ✅ **自动备份** - 使用 `--keep-corrupted` 保留原文件

---

## OSDMap 恢复

### 什么是 OSDMap？

OSDMap 是集群拓扑图，包含：
- 所有 OSD 的状态（up/down, in/out）
- CRUSH 规则
- Pool 配置
- Epoch 信息

OSD 启动时需要 OSDMap 来：
- 确定自己的位置
- 计算 PG 分布
- 与其他 OSD 通信

### 丢失症状

OSD 启动失败，日志显示：
```
-1 osd.0 0 load_pgs: failed to load OSDMap for epoch 1
-1 osd.0 0 OSD::init error: failed to load OSDMap
```

### 使用方法

#### 从 Monitor 获取 OSDMap

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op fetch-osdmaps \
  --epoch 1 \
  --epoch-end 100 \
  --force
```

### 参数说明

| 参数 | 必需 | 说明 |
|------|------|------|
| `--data-path` | 是 | OSD 数据目录 |
| `--epoch` | 是 | 起始 epoch |
| `--epoch-end` | 是 | 结束 epoch |
| `--full-osdmap` | 否 | 获取完整 OSDMap（而非增量） |
| `--force` | 是 | 强制覆盖已存在的 map |

### 获取当前 epoch

```bash
# 从 Monitor 获取
ceph osd dump | grep ^epoch
# 输出: epoch 42

# 或直接查看
ceph status | grep osdmap
# 输出: osdmap e42: 3 osds: 3 up, 3 in
```

### 完整恢复流程

```bash
# 1. 停止 OSD
systemctl stop ceph-osd@0

# 2. 获取当前 epoch
CURRENT_EPOCH=$(ceph osd dump | grep ^epoch | awk '{print $2}')
echo "Current epoch: $CURRENT_EPOCH"

# 3. 从 Monitor 获取 OSDMap
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op fetch-osdmaps \
  --epoch 1 \
  --epoch-end $CURRENT_EPOCH \
  --force

# 4. 启动 OSD
systemctl start ceph-osd@0

# 5. 验证
ceph osd tree
```

### 输出示例

```
Fetching OSDMaps from epoch 1 to 42 (incremental maps)
Writing incremental OSDMap epoch 1 (1234 bytes)
Writing incremental OSDMap epoch 2 (2345 bytes)
Writing incremental OSDMap epoch 3 (3456 bytes)
...
Writing incremental OSDMap epoch 42 (4567 bytes)
Fetch complete: 42 succeeded, 0 failed
```

### 设计实现

#### 核心逻辑

```cpp
int fetch_osdmaps_from_mon(ObjectStore *store, epoch_t first, epoch_t last,
                           bool get_full, bool force) {
  for (epoch_t e = first; e <= last; e++) {
    // 1. 创建临时文件（RAII 管理）
    char tmpfile[] = "/tmp/osdmap.XXXXXX";
    int fd = mkstemp(tmpfile);
    FdGuard fd_guard(fd);
    TempFileGuard file_guard(tmpfile);

    // 2. 使用 ceph 命令获取 OSDMap（安全方式）
    vector<string> cmd_args = {"ceph", "osd", "getmap", to_string(e)};
    if (!get_full) {
      cmd_args.push_back("--incremental");
    }
    cmd_args.push_back("-o");
    cmd_args.push_back(tmpfile);

    int ret = safe_exec(cmd_args);  // 避免命令注入
    if (ret != 0) {
      fail_count++;
      continue;
    }

    // 3. 读取临时文件
    bufferlist bl;
    string error;
    bl.read_file(tmpfile, &error);

    // 4. 写入到 ObjectStore
    if (get_full) {
      set_osdmap(store, e, bl, force);
    } else {
      set_inc_osdmap(store, e, bl, force);
    }

    success_count++;
  }

  return (success_count > 0) ? 0 : -EIO;
}
```

#### 关键特性

1. **RAII 资源管理**:
   ```cpp
   FdGuard fd_guard(fd);          // 自动关闭 fd
   TempFileGuard file_guard(tmpfile);  // 自动删除临时文件
   ```

2. **安全执行命令**:
   ```cpp
   // 使用 vector 构建参数，避免命令注入
   vector<string> cmd_args = {"ceph", "osd", "getmap", to_string(e)};
   safe_exec(cmd_args);
   ```

3. **增量和完整 OSDMap**:
   - **增量 (incremental)**: 只包含变化，节省空间
   - **完整 (full)**: 包含完整状态，更大但更安全

4. **失败容错**:
   ```cpp
   // 单个 epoch 失败不影响其他
   if (ret != 0) {
     fail_count++;
     continue;  // 继续下一个
   }
   ```

### 完整 vs 增量 OSDMap

| 类型 | 大小 | 速度 | 使用场景 |
|------|------|------|----------|
| **增量** | 小 (几 KB) | 快 | 正常恢复 |
| **完整** | 大 (几 MB) | 慢 | 增量损坏时 |

**选择建议**:
- 默认使用增量（`--incremental`）
- 增量获取失败时使用完整（`--full-osdmap`）

### 注意事项

1. ✅ **必须能连接 Monitor** - 需要网络通畅
2. ✅ **权限要求** - 需要 admin 权限或 OSD 权限
3. ⚠️ **网络问题** - Monitor 不可达时会失败
4. ⚠️ **epoch 范围** - 太大范围可能很慢

---

## 磁盘满恢复

### 什么是磁盘满？

**BlueFS 空间耗尽**，导致：
- RocksDB 无法启动（需要空间写 log）
- 无法写入新数据
- OSD 启动失败

**关键指标**:
```
BlueFS Usage: 95.0%  → 危险阈值
BlueFS Usage: 98.0%  → 严重
BlueFS Usage: 100.0% → 无法启动
```

### 症状

OSD 启动失败，日志显示：
```
-1 rocksdb: IO error: No space left on device
-1 bluestore db open failed: (28) No space left on device
-1 osd.0 0 OSD::init: unable to mount object store
```

### 核心策略

**两阶段恢复**:

1. **阶段 1: WAL 清理** ✅
   - 删除旧的 Write-Ahead Log 文件
   - **不需要启动 RocksDB**
   - 即使磁盘 100% 满也能工作
   - 快速释放空间（通常 10-50%）

2. **阶段 2: RocksDB Compaction** (可选)
   - 压缩 SST 文件
   - 需要启动 RocksDB
   - 磁盘 100% 满时会失败
   - 释放更多空间（通常 30-70%）

### 使用方法

#### 方法 1: 分析磁盘使用

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op analyze-disk-usage
```

**输出示例**:
```
=== Disk Usage Analysis ===

BlueFS Usage:
  Total: 100 GB
  Used: 97 GB
  Free: 3 GB
  Usage: 97.00%

⚠️  WARNING: Disk is critically full!
  OSD may fail to start due to insufficient space.
  Recommended action: run 'recover-full-disk' operation

RocksDB Files:
  SST files: 245 (85 GB)
  WAL files: 8 (12 GB)
  MANIFEST: 512 KB
  Other: 2 MB

Space Recovery Potential:
  RocksDB Compaction: ~25 GB - 60 GB
  WAL Cleanup: ~6 GB
```

#### 方法 2: 列出 WAL 文件

```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op list-wal-files
```

**输出示例**:
```
=== WAL Files Analysis ===

Found 8 WAL file(s) in db.wal:

  Log Number              Filename             Size
--------------------------------------------------
       10001     000010001.log            1.2 GB
       10002     000010002.log            1.5 GB
       10003     000010003.log            1.8 GB
       10004     000010004.log            2.1 GB
       10005     000010005.log            1.9 GB
       10006     000010006.log            1.7 GB
       10007     000010007.log            1.4 GB
       10008     000010008.log            0.4 GB (ACTIVE)
--------------------------------------------------
Total WAL size: 12 GB

ℹ️  Note: Multiple WAL files detected
  - Latest WAL (highest number): 000010008.log - ACTIVE, cannot delete
  - Older WALs: May be safe to delete if data is flushed to SST
  - Use 'clean-old-wal' operation to safely delete old WALs
```

#### 方法 3: 清理旧 WAL（手动）

**Dry-run 预览**:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op clean-old-wal \
  --dry-run
```

**实际清理**:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op clean-old-wal \
  --force
```

#### 方法 4: 自动恢复（推荐）

**基本模式**:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op recover-full-disk
```

**激进模式**（包含 Compaction）:
```bash
ceph-objectstore-tool \
  --data-path /var/lib/ceph/osd/ceph-0 \
  --op recover-full-disk \
  --aggressive
```

### 完整恢复流程

```bash
# 1. 停止 OSD（如果还在运行）
systemctl stop ceph-osd@0

# 2. 分析磁盘使用情况
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op analyze-disk-usage

# 3. 列出 WAL 文件
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op list-wal-files

# 4. 执行自动恢复
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op recover-full-disk

# 5. 再次检查空间
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 \
  --op analyze-disk-usage

# 6. 启动 OSD
systemctl start ceph-osd@0

# 7. 监控日志
tail -f /var/log/ceph/ceph-osd.0.log
```

### 恢复输出示例

```
=== OSD Full Disk Recovery ===

Initial disk usage: 97.50%
⚠️  Disk is critically full, proceeding with recovery...

ℹ️  Strategy: Clean old WAL files (does NOT require RocksDB to start)

Step 1: Clean Old WAL Files
----------------------------------------
Active WAL (will be kept): 000010008.log (log 10008)

Old WAL files to delete:
  000010001.log (log 10001, 1.2 GB) - deleted
  000010002.log (log 10002, 1.5 GB) - deleted
  000010003.log (log 10003, 1.8 GB) - deleted
  000010004.log (log 10004, 2.1 GB) - deleted
  000010005.log (log 10005, 1.9 GB) - deleted
  000010006.log (log 10006, 1.7 GB) - deleted
  000010007.log (log 10007, 1.4 GB) - deleted

=== Summary ===
Directory: db.wal
Total WAL files: 8
Deleted: 7
Kept (active): 1
Space freed: 11.6 GB

Step 2: Space Analysis
----------------------------------------
Usage after cleanup: 85.50%

=== Recovery Summary ===
Initial usage: 97.50%
Final usage: 85.50%
Space freed: 11.6 GB (12.00%)

✓ SUCCESS: Disk now has sufficient space
  OSD should be able to start normally
```

### 设计实现

#### 核心逻辑

**recover_full_disk()** - 自动恢复主函数
```cpp
int recover_full_disk(ObjectStore *store, bool aggressive, bool dry_run) {
  // 1. 检查初始空间
  uint64_t initial_total, initial_used;
  bluefs_get_usage_nautilus(bluefs, &initial_total, &initial_used);
  double initial_pct = (double)initial_used / initial_total * 100.0;

  // 1.1 判断是否需要恢复
  if (initial_pct < DISK_FULL_THRESHOLD) {  // 95%
    cout << "Disk not critically full, recovery may not be necessary" << endl;
    return 0;
  }

  // 2. 清理旧的 WAL 文件（核心步骤）
  int r = clean_old_wal(store, true, dry_run);

  // 3. 检查清理后的空间
  bluefs_get_usage_nautilus(bluefs, &after_total, &after_used);
  double after_pct = (double)after_used / after_total * 100.0;

  // 4. 激进模式：尝试 Compaction（可选）
  if (aggressive && after_pct >= DISK_FULL_THRESHOLD && !dry_run) {
    compact_rocksdb_offline(store, true);
  }

  // 5. 判断恢复是否成功
  if (after_pct < DISK_FULL_THRESHOLD) {
    cout << "✓ SUCCESS: Disk now has sufficient space" << endl;
    return 0;
  } else {
    cout << "⚠️  WARNING: Disk still critically full" << endl;
    return -ENOSPC;
  }
}
```

**clean_old_wal()** - WAL 清理核心函数
```cpp
int clean_old_wal(ObjectStore *store, bool force, bool dry_run) {
  // 1. 枚举 WAL 文件并排序
  vector<pair<uint64_t, string>> wal_files;
  collect_bluefs_wal_files(bluefs, &wal_files, &wal_dir);

  // 2. 只保留最后一个（active WAL）
  if (wal_files.size() == 1) {
    cout << "Only one WAL file found, cannot delete" << endl;
    return 0;
  }

  uint64_t active_log = wal_files.back().first;
  string active_file = wal_files.back().second;

  // 3. 删除旧的 WAL（除了最后一个）
  for (size_t i = 0; i < wal_files.size() - 1; i++) {
    const auto& [log_num, filename] = wal_files[i];

    if (!dry_run) {
      int r = bluefs->unlink(wal_dir, filename);  // 直接操作 BlueFS
      if (r == 0) {
        deleted_count++;
      }
    }
  }

  return 0;
}
```

**collect_bluefs_wal_files()** - 收集 WAL 文件
```cpp
int collect_bluefs_wal_files(BlueFS* bluefs,
                              vector<pair<uint64_t, string>>* wal_files,
                              string* wal_dir) {
  // 1. 尝试 db.wal 目录
  vector<string> files;
  int r = bluefs->readdir("db.wal", &files);

  if (r == 0) {
    *wal_dir = "db.wal";
  } else {
    // 2. 回退到 db 目录
    r = bluefs->readdir("db", &files);
    *wal_dir = "db";
  }

  // 3. 提取 WAL 文件并排序
  for (const auto& file : files) {
    // 匹配格式：000012345.log
    if (file.find(".log") != string::npos) {
      uint64_t log_num = extract_log_number(file);
      wal_files->push_back({log_num, file});
    }
  }

  // 4. 按 log number 排序
  sort(wal_files->begin(), wal_files->end());

  return 0;
}
```

#### 关键设计

**1. 不依赖 RocksDB 启动**
```cpp
// 直接操作 BlueFS，绕过 RocksDB
bluefs->unlink(wal_dir, filename);  // 不需要 RocksDB 运行
```

这是最关键的设计！传统方案需要 RocksDB 启动才能清理，但磁盘满时 RocksDB 启动不了，形成死锁。这个方案直接操作 BlueFS，打破死锁。

**2. 保护 Active WAL**
```cpp
// 只删除旧的 WAL，保留最后一个（active）
for (size_t i = 0; i < wal_files.size() - 1; i++) {
  // 删除 wal_files[0] 到 wal_files[n-2]
  // 保留 wal_files[n-1] (active)
}
```

**3. 95% 阈值判断**
```cpp
static const double DISK_FULL_THRESHOLD = 95.0;

// 判断是否需要恢复
if (usage_pct >= DISK_FULL_THRESHOLD) {
  // 需要恢复
}

// 判断恢复是否成功
if (usage_pct < DISK_FULL_THRESHOLD) {
  // 成功
}
```

**为什么是 95%？**
- RocksDB 启动需要写 log（约 1-2%）
- Compaction 需要临时空间（约 3-5%）
- 留 5% 作为安全余量

**4. 两阶段策略**
```cpp
// 阶段 1: WAL 清理（总是执行）
clean_old_wal(store, true, dry_run);

// 阶段 2: Compaction（aggressive 模式才执行）
if (aggressive && after_pct >= DISK_FULL_THRESHOLD && !dry_run) {
  compact_rocksdb_offline(store, true);
}
```

**5. 数据丢失警告**
```cpp
// WAL 清理可能丢失未 flush 的数据
if (deleted_count > 0 && !dry_run) {
  cout << "⚠️  WARNING: Old WAL files deleted" << endl;
  cout << "  - If RocksDB did not properly flush data to SST,
           some recent writes may be lost" << endl;
  cout << "  - This is usually safe if OSD was shut down cleanly" << endl;
  cout << "  - Use with caution in crash scenarios" << endl;
}
```

### WAL 清理的安全性

| 场景 | 安全性 | 数据风险 |
|------|--------|----------|
| **正常关闭** | ✅ 安全 | 无风险 |
| **异常崩溃** | ⚠️ 有风险 | 可能丢失最近写入 |
| **磁盘满导致无法启动** | ✅ 可接受 | 权衡：不恢复则完全无法启动 |

**建议**:
- 正常关闭后清理：完全安全
- 崩溃后清理：有风险但必要时可以接受
- 清理前先尝试启动 OSD，确认无法启动再清理

### 空间恢复预期

| 方法 | 释放空间 | 速度 | 成功率 | 依赖 RocksDB |
|------|----------|------|--------|--------------|
| **WAL 清理** | 10-50% | 快 (1 分钟) | 99%+ | ❌ 不需要 |
| **RocksDB Compaction** | 30-70% | 慢 (10 分钟) | 70% | ✅ 需要 |

### 注意事项

1. ✅ **不需要 RocksDB 启动** - 核心优势
2. ✅ **快速恢复** - 通常 1-5 分钟
3. ⚠️ **数据丢失风险** - 崩溃场景下可能丢失未 flush 的数据
4. ⚠️ **95% 是硬编码** - 无法配置
5. ✅ **支持 dry-run** - 可以先预览
6. ✅ **自动判断** - 自动决定是否需要恢复

---

## 故障诊断流程

### 快速诊断

```bash
# 1. 查看 OSD 状态
systemctl status ceph-osd@0

# 2. 查看最近的错误日志
tail -100 /var/log/ceph/ceph-osd.0.log | grep -i error

# 3. 尝试前台启动（获取详细错误）
sudo -u ceph /usr/bin/ceph-osd -f --cluster ceph --id 0
```

### 根据错误选择修复方法

| 错误关键词 | 问题类型 | 修复方法 |
|-----------|----------|----------|
| `unable to read osd superblock` | Superblock 损坏 | [Superblock 修复](#superblock-修复) |
| `Corruption: block checksum` | RocksDB 损坏 | [RocksDB SST 修复](#rocksdb-sst-修复) |
| `failed to load OSDMap` | OSDMap 丢失 | [OSDMap 恢复](#osdmap-恢复) |
| `No space left on device` | 磁盘满 | [磁盘满恢复](#磁盘满恢复) |

### 完整诊断流程图

```
OSD 无法启动
    ↓
检查日志错误
    ↓
┌─────────────┬──────────────┬──────────────┬──────────────┐
│ Superblock  │  RocksDB     │   OSDMap     │  Disk Full   │
│   损坏      │   损坏       │    丢失      │    空间满    │
└─────────────┴──────────────┴──────────────┴──────────────┘
    ↓              ↓              ↓              ↓
repair-         scan-          fetch-         recover-
superblock      rocksdb        osdmaps        full-disk
    ↓              ↓              ↓              ↓
启动 OSD ←────────┴──────────────┴──────────────┘
    ↓
验证功能
```

### 通用注意事项

1. ✅ **备份** - 修复前备份重要数据（如果可能）
2. ✅ **停止 OSD** - 修复时必须停止 OSD
3. ✅ **记录参数** - 记录 cluster_fsid、epoch 等信息
4. ✅ **验证修复** - 修复后验证 OSD 功能
5. ✅ **监控日志** - 启动后监控日志确认无错误
6. ⚠️ **数据风险** - 某些修复可能导致数据丢失

---

## 参考资料

### 相关文档

- [SUPERBLOCK_REPAIR_GUIDE.md](SUPERBLOCK_REPAIR_GUIDE.md) - Superblock 详细指南
- [ROCKSDB_REPAIR_GUIDE.md](ROCKSDB_REPAIR_GUIDE.md) - RocksDB 详细指南

### 测试套件

- [test_osd_recovery_suite.sh](test_osd_recovery_suite.sh) - 自动化测试脚本

运行测试：
```bash
cd /path/to/ceph/build
bash ../src/tools/test_osd_recovery_suite.sh

# 运行单个场景
bash ../src/tools/test_osd_recovery_suite.sh --scenario superblock
bash ../src/tools/test_osd_recovery_suite.sh --scenario rocksdb
bash ../src/tools/test_osd_recovery_suite.sh --scenario osdmap
bash ../src/tools/test_osd_recovery_suite.sh --scenario diskfull
```

### 代码位置

- **实现**: [src/tools/ceph_objectstore_tool.cc](ceph_objectstore_tool.cc)
- **测试**: [src/tools/test_osd_recovery_suite.sh](test_osd_recovery_suite.sh)

---

## 常见问题 (FAQ)

### Q1: 修复会导致数据丢失吗？

**A**: 取决于修复类型：
- **Superblock**: 元数据丢失，但数据完整（需要 re-peering）
- **RocksDB**: 可能丢失部分元数据（损坏的 key-value）
- **OSDMap**: 无数据丢失
- **Disk Full**: WAL 清理可能丢失最近写入（如果未 flush）

### Q2: 修复需要多长时间？

**A**:
- Superblock: 5-10 秒
- RocksDB: 5-60 分钟（取决于 SST 文件大小）
- OSDMap: 1-5 分钟
- Disk Full: 1-10 分钟

### Q3: 可以在线修复吗？

**A**: 不可以。所有修复操作都需要先停止 OSD。

### Q4: 修复失败怎么办？

**A**:
1. 检查错误日志
2. 确认参数正确
3. 尝试其他修复方法
4. 考虑重建 OSD（最后手段）

### Q5: 如何避免这些问题？

**A**:
1. 定期检查 OSD 健康状态
2. 监控磁盘空间（< 90%）
3. 定期备份关键元数据
4. 使用 RAID/冗余保护磁盘
5. 及时更新 Ceph 版本（修复已知 bug）

### Q6: 磁盘满了为什么 OSD 启动不了？

**A**: RocksDB 启动需要：
- 写 log 文件（约 1-2% 空间）
- Compaction 临时空间（约 3-5% 空间）
- 空间不足会导致 RocksDB 启动失败

### Q7: 为什么 WAL 清理不需要启动 RocksDB？

**A**: WAL 文件存储在 BlueFS 中，可以直接通过 BlueFS API 操作，不需要 RocksDB 运行。这是设计的核心优势。

### Q8: 清理 WAL 安全吗？

**A**:
- **正常关闭后**: 完全安全（数据已 flush 到 SST）
- **崩溃后**: 有风险（可能丢失未 flush 的数据）
- **权衡**: 不清理则 OSD 完全无法启动

---

## 技术支持

如有问题或需要帮助，请：
1. 查阅本文档
2. 查看详细日志（`/var/log/ceph/ceph-osd.*.log`）
3. 运行诊断命令
4. 联系 Ceph 社区或技术支持

---

**文档版本**: 1.0
**最后更新**: 2025-10-14
**作者**: Ceph OSD Recovery Tool Team
