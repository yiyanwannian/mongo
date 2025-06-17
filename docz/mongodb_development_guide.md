# MongoDB 开发指南

## 快速开始

### 环境准备

#### 系统要求
- **操作系统**: Linux、macOS、Windows
- **磁盘空间**: 核心构建需要13GB，完整构建需要600GB
- **内存**: 建议16GB以上
- **CPU**: 支持x86-64、ARM64、PPC64LE、S390X架构

#### 编译器要求
- **GCC**: 14.2或更新版本
- **Clang**: 19.1或更新版本  
- **Visual Studio**: 2022 17.0或更新版本
- **Xcode**: 16.4或更新版本

#### 依赖库
```bash
# Ubuntu/Debian
sudo apt-get install build-essential libcurl4-openssl-dev liblzma-dev

# Fedora/RHEL
sudo dnf install libcurl-devel

# macOS (使用Homebrew)
brew install llvm@19 lld@19
```

### 构建步骤

#### 1. 安装Bazel构建工具
```bash
python buildscripts/install_bazel.py
export PATH=~/.local/bin:$PATH
```

#### 2. 构建核心组件
```bash
# 构建mongod数据库服务器
bazel build install-mongod

# 构建mongos路由器
bazel build install-mongos

# 构建核心组件(mongod + mongos)
bazel build install-core

# 构建开发版本(包含shell)
bazel build install-devcore

# 构建完整发行版
bazel build install-dist
```

#### 3. 运行测试
```bash
# 运行单元测试
bazel test //src/mongo/...

# 运行集成测试
python buildscripts/resmoke.py run --suite=core

# 运行特定测试套件
python buildscripts/resmoke.py run --suite=replica_sets
```

## 项目结构详解

### 核心目录结构
```
src/mongo/
├── base/              # 基础工具类和数据结构
│   ├── error_codes/   # 错误码定义
│   ├── string_data/   # 字符串处理
│   └── status/        # 状态和错误处理
├── bson/              # BSON数据格式
│   ├── bsonobj/       # BSON对象
│   ├── bsonelement/   # BSON元素
│   └── util/          # BSON工具
├── client/            # 客户端连接
│   ├── connection/    # 连接管理
│   └── replica_set/   # 复制集客户端
├── db/                # 数据库核心
│   ├── auth/          # 认证授权
│   ├── catalog/       # 数据库目录
│   ├── commands/      # 数据库命令
│   ├── concurrency/   # 并发控制
│   ├── exec/          # 查询执行
│   ├── index/         # 索引管理
│   ├── ops/           # 操作处理
│   ├── pipeline/      # 聚合管道
│   ├── query/         # 查询处理
│   ├── repl/          # 复制系统
│   ├── s/             # 分片功能
│   ├── storage/       # 存储引擎
│   └── views/         # 视图支持
├── s/                 # mongos路由器
│   ├── catalog/       # 分片目录
│   ├── client/        # 分片客户端
│   ├── commands/      # 分片命令
│   └── query/         # 分片查询
└── util/              # 通用工具
    ├── concurrency/   # 并发工具
    ├── net/           # 网络工具
    └── time/          # 时间工具
```

### 关键组件说明

#### 1. 存储引擎层 (`src/mongo/db/storage/`)
- **接口定义**: 存储引擎抽象接口
- **WiredTiger集成**: WiredTiger存储引擎适配
- **记录管理**: 记录ID和记录存储
- **事务支持**: 事务和恢复单元

#### 2. 查询处理 (`src/mongo/db/query/`)
- **查询解析**: SQL到内部表示的转换
- **查询规划**: 查询计划生成和优化
- **查询执行**: 多种执行引擎支持
- **索引选择**: 索引使用策略

#### 3. 复制系统 (`src/mongo/db/repl/`)
- **Oplog管理**: 操作日志处理
- **复制协调**: 复制状态管理
- **选举机制**: 主节点选举
- **读写关注**: 一致性控制

#### 4. 分片系统 (`src/mongo/s/`)
- **路由逻辑**: 查询路由和负载均衡
- **元数据管理**: 分片配置和状态
- **块管理**: 数据块分布和迁移
- **分片命令**: 分片相关操作

## 开发工作流

### 1. 代码贡献流程
```bash
# 1. Fork项目并克隆
git clone git@github.com:your-username/mongo.git
cd mongo

# 2. 创建功能分支
git checkout -b feature/your-feature-name

# 3. 进行开发
# ... 编写代码 ...

# 4. 运行测试
bazel test //src/mongo/...
python buildscripts/resmoke.py run --suite=core

# 5. 提交更改
git add .
git commit -m "Add your feature description"

# 6. 推送并创建PR
git push origin feature/your-feature-name
```

### 2. 调试技巧

#### 使用GDB调试
```bash
# 构建调试版本
bazel build --config=dbg install-mongod

# 启动GDB
gdb bazel-bin/install/bin/mongod
(gdb) run --dbpath /data/db
```

#### 日志调试
```bash
# 启用详细日志
mongod --dbpath /data/db --logpath /var/log/mongod.log --logappend --verbose

# 设置特定组件日志级别
db.setLogLevel(2, "query")
db.setLogLevel(3, "replication")
```

### 3. 性能分析

#### 使用内置性能工具
```javascript
// 启用性能分析
db.setProfilingLevel(2)

// 查看慢查询
db.system.profile.find().sort({ts: -1}).limit(5)

// 分析查询计划
db.collection.find({field: "value"}).explain("executionStats")
```

#### 使用外部工具
```bash
# 使用perf分析
perf record -g bazel-bin/install/bin/mongod --dbpath /data/db
perf report

# 使用valgrind检查内存
valgrind --tool=memcheck bazel-bin/install/bin/mongod --dbpath /data/db
```

## 测试策略

### 1. 单元测试
```cpp
// 示例单元测试
#include "mongo/unittest/unittest.h"

namespace mongo {
namespace {

TEST(MyComponentTest, BasicFunctionality) {
    MyComponent component;
    ASSERT_EQ(component.getValue(), expectedValue);
}

} // namespace
} // namespace mongo
```

### 2. 集成测试
```javascript
// JavaScript集成测试示例
(function() {
    "use strict";
    
    const rst = new ReplSetTest({nodes: 3});
    rst.startSet();
    rst.initiate();
    
    const primary = rst.getPrimary();
    const testDB = primary.getDB("test");
    
    // 执行测试操作
    assert.commandWorked(testDB.collection.insert({_id: 1, data: "test"}));
    
    rst.stopSet();
})();
```

### 3. 性能测试
```bash
# 运行基准测试
python buildscripts/resmoke.py run --suite=benchmarks

# 自定义性能测试
python buildscripts/resmoke.py run --suite=core --perfReportFile=perf_results.json
```

## 代码规范

### 1. C++代码规范
- 使用4空格缩进
- 类名使用PascalCase
- 函数名使用camelCase
- 常量使用kConstantName格式
- 包含适当的注释和文档

### 2. JavaScript测试规范
- 使用严格模式 `"use strict"`
- 函数名使用camelCase
- 适当的错误处理
- 清理测试资源

### 3. 提交信息规范
```
类型(范围): 简短描述

详细描述说明更改的内容和原因。

Fixes: #issue_number
```

## 常见问题解决

### 1. 构建问题
```bash
# 清理构建缓存
bazel clean

# 强制重新构建
bazel build --config=force_rebuild install-mongod

# 禁用警告作为错误
bazel build --disable_warnings_as_errors=True install-mongod
```

### 2. 测试问题
```bash
# 运行特定测试
python buildscripts/resmoke.py run jstests/core/find.js

# 调试测试失败
python buildscripts/resmoke.py run --suite=core --shuffle=off --repeat=1
```

### 3. 性能问题
- 检查存储引擎配置
- 优化查询和索引
- 调整缓存大小
- 监控系统资源使用

## 有用的资源

### 官方文档
- [MongoDB Manual](https://docs.mongodb.com/manual/)
- [MongoDB Developer Center](https://www.mongodb.com/developer/)
- [MongoDB University](https://learn.mongodb.com)

### 社区资源
- [MongoDB Community Forums](https://mongodb.com/community/forums/)
- [Server Development Forum](https://mongodb.com/community/forums/c/server-dev)
- [GitHub Issues](https://github.com/mongodb/mongo/issues)

### 开发工具
- **IDE推荐**: CLion, Visual Studio Code, Visual Studio
- **调试工具**: GDB, LLDB, Valgrind
- **性能分析**: perf, Intel VTune, MongoDB Profiler
- **代码分析**: Clang Static Analyzer, Coverity
