# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 交流规则

与用户交流使用中文。

## 项目概述

Ceph是一个分布式存储系统，提供对象存储、块存储和文件系统存储。版本：14.2.22（Nautilus分支）

## 构建系统

### 初始化构建

```bash
# 安装依赖
./install-deps.sh

# 初始化子模块
git submodule update --init --recursive

# 配置构建（默认创建Debug构建）
./do_cmake.sh

# 性能构建使用：
./do_cmake.sh -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

### 编译

```bash
cd build
make                    # 编译所有目标
make -j$(nproc)        # 并行编译
make [target_name]     # 编译指定目标
make vstart            # 编译足够运行测试集群的组件
```

构建输出位置：
- 二进制文件: `build/bin/`
- 库文件: `build/lib/`

### CMake重要选项

- `WITH_CCACHE=ON` - 使用ccache加速编译
- `WITH_RADOSGW=ON/OFF` - 是否构建RADOS Gateway
- `WITH_BLUESTORE=ON/OFF` - BlueStore后端支持
- `WITH_TESTS=ON/OFF` - 是否构建测试
- `DIAGNOSTICS_COLOR=always` - 保持错误诊断颜色

## 测试

### 运行测试集群

```bash
cd build
make vstart
../src/vstart.sh --debug --new -x --localhost --bluestore

# 测试集群命令
./bin/ceph -s                                # 查看集群状态
./bin/rados -p rbd bench 30 write           # 运行基准测试
./bin/rbd create foo --size 1000            # 创建RBD镜像

# 停止测试集群
../src/stop.sh
```

vstart.sh环境变量：
- `CEPH_NUM_MON` - Monitor数量（默认3）
- `CEPH_NUM_OSD` - OSD数量（默认3）
- `CEPH_NUM_MDS` - MDS数量（默认3）
- `CEPH_NUM_MGR` - Manager数量（默认1）

### 运行单元测试

```bash
cd build
make                        # 先构建
ctest -j$(nproc)           # 运行所有测试（并行）
make check -j$(nproc)      # 构建并运行测试

# 运行单个测试
ctest -R [test_name_regex]           # 正则匹配测试名
ctest -V -R [test_name_regex]        # 详细输出

# 注意：
# - unittest* 开头的测试可用ctest运行
# - ceph_test* 开头的测试需要手动运行
```

测试日志位置：`build/Testing/Temporary/`

### 完整测试套件

```bash
# 运行make check（包含单元和集成测试）
./run-make-check.sh
```

## 代码架构

### 核心组件目录

- **src/mon/** - Monitor守护进程，维护集群映射主副本
- **src/osd/** - OSD守护进程，处理数据存储和复制
- **src/mds/** - MDS守护进程，CephFS元数据服务器
- **src/mgr/** - Manager守护进程，集群管理和监控
- **src/os/** - ObjectStore后端实现
  - `filestore/` - FileStore后端（传统）
  - `bluestore/` - BlueStore后端（新式，直接管理裸设备）
- **src/osd/PG.h/cc** - Placement Group实现，数据分组和复制单元
- **src/crush/** - CRUSH算法，数据放置算法
- **src/common/** - 通用工具和库
- **src/msg/** - 消息传递层
- **src/auth/** - 认证框架（Cephx）
- **src/global/** - 全局初始化和配置

### 客户端接口

- **src/librados/** - RADOS C/C++库，对象存储接口
- **src/librbd/** - RBD（RADOS Block Device）库
- **src/client/** - CephFS客户端
- **src/rgw/** - RADOS Gateway（S3/Swift对象存储）

### 关键工具

- **src/tools/** - 命令行工具
  - `ceph_objectstore_tool.cc` - ObjectStore离线操作工具
  - `rados/` - rados命令行工具
  - `rbd/` - rbd命令行工具
- **src/ceph-volume/** - OSD部署和管理工具
- **src/pybind/** - Python绑定

### 测试代码

- **src/test/** - 单元测试和集成测试
- **qa/** - 质量保证测试套件
  - `qa/suites/rados/` - RADOS测试套件
  - `qa/suites/rbd/` - RBD测试套件
  - `qa/suites/fs/` - CephFS测试套件
  - `qa/suites/rgw/` - RGW测试套件
  - `qa/suites/upgrade/` - 升级测试

## 架构要点

### RADOS架构

RADOS（Reliable Autonomic Distributed Object Store）是Ceph的基础：
- **Monitor集群** 维护cluster map（OSD map, Monitor map, PG map等）
- **OSD** 存储数据对象，执行复制、恢复、再平衡
- **CRUSH算法** 伪随机数据分布，避免中心化查找
- **Placement Group (PG)** 对象分组单元，简化数据管理

数据流：Client → librados → Monitor（获取cluster map） → OSD（通过CRUSH计算位置）

### ObjectStore抽象

ObjectStore是OSD存储后端的抽象接口：
- **FileStore** - 基于文件系统（ext4/xfs + journal）
- **BlueStore** - 直接管理裸设备，消除双写问题，原生校验和

### PG状态机

PG经历多个状态：creating, active, clean, degraded, peering, recovering等
理解PG状态对调试OSD问题至关重要。

## 开发工作流

### 提交补丁

1. 在GitHub上创建Pull Request
2. 代码审查（reviewer分配）
3. 集成测试（tester运行teuthology测试）
4. 合并到master分支

### 提交信息要求

每个提交必须包含 `Signed-off-by:` 行，表示接受Developer's Certificate of Origin。

```
commit message summary

Longer description if needed.

Signed-off-by: Your Name <your.email@example.com>
```

### Issue跟踪

- 使用tracker.ceph.com跟踪Bug和功能请求
- 优先级：Normal → 经过bug scrub后分配优先级
- 状态：New → Verified → In Progress → Pending Backport

## 文档

- **doc/** - Sphinx文档源码
- 构建文档：`admin/build-doc`
- 在线文档：http://docs.ceph.com/

## 重要文件

- **src/ceph.in** - ceph命令行工具主脚本
- **src/vstart.sh** - 启动开发测试集群
- **src/stop.sh** - 停止测试集群
- **do_cmake.sh** - CMake配置辅助脚本
- **run-make-check.sh** - 完整测试运行脚本

## 常见任务

### 修改OSD代码后测试

```bash
cd build
make ceph-osd           # 只编译OSD
../src/stop.sh          # 停止旧集群
rm -rf dev out          # 清理旧数据
../src/vstart.sh --debug --new -x --localhost --bluestore
./bin/ceph -s           # 验证集群
```

### 添加新的unittest

1. 在 `src/test/` 创建测试文件
2. 在相应 `CMakeLists.txt` 添加测试目标（使用 `add_ceph_unittest`）
3. 运行 `ctest -R your_test_name`

### 调试崩溃

- 测试集群日志：`build/out/*.log`
- 核心转储：检查ulimit设置
- 使用 `--debug` 参数启动vstart获取详细日志
- OSD日志级别：`ceph tell osd.* injectargs '--debug-osd 20'`

## 代码风格

- C++17标准（要求GCC 7+）
- 遵循现有代码风格
- 缩进：2空格（见文件头注释 `// vim: ts=8 sw=2 smarttab`）
