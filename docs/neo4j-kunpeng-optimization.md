# Neo4j 鲲鹏服务器优化指南

本文档详细介绍 Neo4j 在华为鲲鹏(ARM)服务器上的部署、配置和性能优化方法。

## 目录

- [1. 环境准备](#1-环境准备)
- [2. 部署方式](#2-部署方式)
- [3. JVM 优化（毕昇JDK）](#3-jvm-优化毕昇jdk)
- [4. 内存配置优化](#4-内存配置优化)
- [5. 存储优化](#5-存储优化)
- [6. 网络与 Bolt 协议优化](#6-网络与-bolt-协议优化)
- [7. 查询优化](#7-查询优化)
- [8. 监控与诊断](#8-监控与诊断)
- [9. 最佳实践总结](#9-最佳实践总结)

---

## 1. 环境准备

### 1.1 硬件要求

| 配置项 | 最低要求 | 推荐配置 |
|--------|----------|----------|
| CPU | 鲲鹏920 4核 | 鲲鹏920 16核+ |
| 内存 | 8GB | 64GB+ |
| 存储 | SATA HDD | NVMe SSD |
| 网络 | 1Gbps | 10Gbps |

### 1.2 操作系统

推荐使用以下操作系统：
- **openEuler 20.03/22.03 LTS** - 华为官方支持，与毕昇JDK深度优化
- **银河麒麟 V10 (ARM版)** - 国产化信创环境首选
- **CentOS 7.9+ / Ubuntu 20.04+** - 通用 Linux 发行版

### 1.3 JDK 选择

**强烈推荐使用毕昇JDK**（华为基于OpenJDK定制的ARM优化版本）：

```bash
# 下载毕昇JDK
wget https://mirrors.huaweicloud.com/kunpeng/archive/bisheng_jdk/bisheng-jdk-11.0.13-linux-aarch64.tar.gz

# 解压并配置环境变量
tar -xzf bisheng-jdk-11.0.13-linux-aarch64.tar.gz -C /opt/
export JAVA_HOME=/opt/bisheng-jdk-11.0.13
export PATH=$JAVA_HOME/bin:$PATH
```

**毕昇JDK 优势：**
- SPECjbb benchmark 性能提升 **55%**（critical）、**16%**（max）
- SPECjvm 平均性能提升 **4.6%**
- 针对 ARM 架构优化的 GC 算法
- 支持鲲鹏硬件加速（KAE 加解密）
- G1 NUMA-Aware 优化，充分发挥多核优势

---

## 2. 部署方式

### 2.1 Docker 部署（推荐）

Docker 方式可避免系统环境差异，特别适合鲲鹏 ARM 环境：

```bash
# 安装 Docker
yum install -y docker
systemctl start docker
systemctl enable docker

# 拉取并运行 Neo4j
docker run -d \
  --name neo4j \
  -p 7474:7474 \
  -p 7687:7687 \
  -v neo4j_data:/data \
  -v neo4j_logs:/logs \
  -v neo4j_import:/var/lib/neo4j/import \
  -v neo4j_plugins:/plugins \
  -e NEO4J_AUTH=neo4j/your_password \
  --ulimit nofile=65536:65536 \
  neo4j:5-community
```

### 2.2 二进制部署

```bash
# 下载 Neo4j
wget https://dist.neo4j.org/neo4j-community-5.17.0-unix.tar.gz
tar -xzf neo4j-community-5.17.0-unix.tar.gz -C /opt/

# 配置环境变量
export NEO4J_HOME=/opt/neo4j-community-5.17.0
export PATH=$NEO4J_HOME/bin:$PATH

# 启动服务
neo4j start
```

---

## 3. JVM 优化（毕昇JDK）

### 3.1 内存模型差异

ARM 架构与 x86 存在重要差异：

| 特性 | x86 | ARM (鲲鹏) |
|------|-----|------------|
| 内存序 | 强内存序 | 弱内存序 |
| CAS指令 | cmpxchgl | Ldaxr/Stlxr |
| SIMD | SSE/AVX | NEON/SVE |

### 3.2 推荐 JVM 参数

在 `conf/neo4j.conf` 中配置：

```properties
# 堆内存设置（建议为物理内存的 50%-60%）
server.java.additional=-Xms16g
server.java.additional=-Xmx16g

# G1 垃圾收集器（推荐）
server.java.additional=-XX:+UseG1GC
server.java.additional=-XX:MaxGCPauseMillis=200
server.java.additional=-XX:G1HeapRegionSize=32m

# G1 NUMA-Aware（毕昇JDK特有优化）
server.java.additional=-XX:+UseNUMA

# G1 Uncommit 特性（低负载时归还内存给OS）
server.java.additional=-XX:G1UncommitEnabled=true

# 并行 GC 线程数（根据CPU核心数调整）
server.java.additional=-XX:ParallelGCThreads=16
server.java.additional=-XX:ConcGCThreads=4

# 大页内存（可选，需要系统配置）
server.java.additional=-XX:+UseLargePages

# JIT 优化
server.java.additional=-XX:+AggressiveOpts
server.java.additional=-XX:+UseFastAccessorMethods
```

### 3.3 ZGC 配置（大内存场景）

对于 64GB+ 内存的服务器，可考虑 ZGC：

```properties
server.java.additional=-XX:+UnlockExperimentalVMOptions
server.java.additional=-XX:+UseZGC
server.java.additional=-Xmx64g
server.java.additional=-XX:ConcGCThreads=8
```

### 3.4 毕昇JDK 特有优化

```properties
# KAE 硬件加速（需要鲲鹏硬件支持）
server.java.additional=-Dsecurity.provider.1=org.bisheng.jce.provider.KAEProvider

# SVE 向量指令优化（鲲鹏920支持）
server.java.additional=-XX:UseSVE=2
```

---

## 4. 内存配置优化

### 4.1 内存分配原则

Neo4j 内存 = 堆内存 + 页面缓存 + 操作系统预留

```
总内存 = heap + pagecache + (2-4GB OS预留)
```

### 4.2 页面缓存配置

```properties
# 页面缓存大小（建议为堆内存的 50%-100%）
dbms.memory.pagecache.size=8g

# 事务日志缓冲区
dbms.tx_log.rotation.retention_policy=100M size

# 内核页面缓存预热
dbms.memory.pagecache.warmup.enabled=true
```

### 4.3 内存配置计算示例

| 服务器内存 | 堆内存 | 页面缓存 | 系统预留 |
|-----------|--------|----------|----------|
| 16GB | 8GB | 4GB | 4GB |
| 32GB | 16GB | 12GB | 4GB |
| 64GB | 32GB | 28GB | 4GB |
| 128GB | 64GB | 56GB | 8GB |

---

## 5. 存储优化

### 5.1 文件系统选择

| 文件系统 | 推荐度 | 说明 |
|----------|--------|------|
| XFS | ⭐⭐⭐⭐⭐ | 推荐首选，高性能 |
| EXT4 | ⭐⭐⭐⭐ | 稳定可靠 |
| NFS/NAS | ❌ | 不推荐，性能差 |

### 5.2 磁盘挂载优化

```bash
# XFS 挂载选项
mount -t xfs -o noatime,nodiratime,logbufs=8,logbsize=256k /dev/sdb1 /data

# /etc/fstab 配置
/dev/sdb1 /data xfs noatime,nodiratime,logbufs=8,logbsize=256k 0 0
```

### 5.3 NVMe SSD 优化

```properties
# 关闭 Neo4j 的 fsync（仅限 NVMe SSD，有数据风险）
dbms.tx_log.rotation.tx_prune_strategy=NO_PRUNE

# 批量写入优化
dbms.tx_log.rotation.batch_size=1024
```

### 5.4 数据目录分离

```properties
# 数据目录分离配置
dbms.directories.data=/data/neo4j/data
dbms.directories.logs=/var/log/neo4j
dbms.tx_log.location=/data/neo4j/tx-logs
```

---

## 6. 网络与 Bolt 协议优化

### 6.1 Bolt 连接池配置

```properties
# Bolt 线程配置
dbms.connector.bolt.thread_pool_max_size=200
dbms.connector.bolt.thread_pool_min_size=50
dbms.connector.bolt.thread_pool_keep_alive=5m

# 连接超时
dbms.connector.bolt.connection_timeout=5m

# 启用 Bolt 协议
dbms.connector.bolt.enabled=true
dbms.connector.bolt.listen_address=0.0.0.0:7687

# 启用 TLS（生产环境推荐）
dbms.connector.bolt.tls_level=REQUIRED
```

### 6.2 HTTP 连接器配置

```properties
dbms.connector.http.enabled=true
dbms.connector.http.listen_address=0.0.0.0:7474

# 启用 HTTPS
dbms.connector.https.enabled=true
dbms.connector.https.listen_address=0.0.0.0:7473
```

### 6.3 系统网络优化

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# 应用配置
sysctl -p
```

---

## 7. 查询优化

### 7.1 索引优化

```cypher
-- 创建索引
CREATE INDEX FOR (n:Person) ON (n.name);
CREATE INDEX FOR (n:Person) ON (n.email, n.phone);

-- 创建全文索引
CREATE FULLTEXT INDEX personFulltext FOR (n:Person) ON EACH [n.name, n.bio];

-- 查看索引使用情况
CALL db.indexes();
```

### 7.2 查询分析

```cypher
-- 使用 PROFILE 分析查询
PROFILE MATCH (p:Person)-[:KNOWS]->(f)
WHERE p.name = 'Alice'
RETURN f.name;

-- 使用 EXPLAIN 查看执行计划
EXPLAIN MATCH (p:Person)-[:KNOWS]->(f)
WHERE p.name = 'Alice'
RETURN f.name;
```

### 7.3 查询优化原则

1. **使用参数化查询**
```cypher
-- 推荐
:param name => 'Alice';
MATCH (p:Person {name: $name}) RETURN p;
```

2. **限制返回结果**
```cypher
MATCH (p:Person)-[:KNOWS]->(f)
RETURN f LIMIT 100;
```

3. **使用 WITH 分割复杂查询**
```cypher
MATCH (p:Person)
WHERE p.age > 30
WITH p
MATCH (p)-[:KNOWS]->(f)
RETURN p, collect(f) as friends;
```

---

## 8. 监控与诊断

### 8.1 Neo4j 监控指标

```properties
# 启用 Prometheus 指标
metrics.enabled=true
metrics.prometheus.enabled=true
metrics.prometheus.endpoint=0.0.0.0:2004

# 启用 JMX
dbms.jvm.additional=-Dcom.sun.management.jmxremote.port=3637
dbms.jvm.additional=-Dcom.sun.management.jmxremote.authenticate=false
dbms.jvm.additional=-Dcom.sun.management.jmxremote.ssl=false
```

### 8.2 日志配置

```properties
# 调试日志
dbms.logs.debug.level=INFO

# 查询日志
dbms.logs.query.enabled=true
dbms.logs.query.threshold=1s
dbms.logs.query.page_logging_enabled=true
```

### 8.3 性能诊断命令

```cypher
-- 查看当前事务
CALL dbms.listTransactions();

-- 查看系统信息
CALL dbms.info();

-- 查看内存使用
CALL dbms.memory.usage();

-- 查看查询统计
CALL dbms.queryJmx("org.neo4j:*");
```

---

## 9. 最佳实践总结

### 9.1 鲲鹏平台专项优化清单

| 优化项 | 推荐配置 | 优先级 |
|--------|----------|--------|
| JDK | 毕昇JDK 11/17 | ⭐⭐⭐⭐⭐ |
| GC | G1GC + NUMA-Aware | ⭐⭐⭐⭐⭐ |
| 堆内存 | 物理内存 50%-60% | ⭐⭐⭐⭐⭐ |
| 页面缓存 | 堆内存 50%-100% | ⭐⭐⭐⭐ |
| 文件系统 | XFS + noatime | ⭐⭐⭐⭐ |
| 存储 | NVMe SSD | ⭐⭐⭐⭐ |
| 网络 | 10Gbps + 优化 TCP | ⭐⭐⭐ |

### 9.2 配置文件示例

完整的 `neo4j.conf` 优化配置：

```properties
#===============================
# Neo4j 鲲鹏服务器优化配置
#===============================

# 内存配置
dbms.memory.heap.initial_size=16g
dbms.memory.heap.max_size=16g
dbms.memory.pagecache.size=12g

# JVM 配置（毕昇JDK）
server.java.additional=-XX:+UseG1GC
server.java.additional=-XX:MaxGCPauseMillis=200
server.java.additional=-XX:G1HeapRegionSize=32m
server.java.additional=-XX:+UseNUMA
server.java.additional=-XX:ParallelGCThreads=16
server.java.additional=-XX:ConcGCThreads=4

# 网络配置
dbms.connector.bolt.enabled=true
dbms.connector.bolt.listen_address=0.0.0.0:7687
dbms.connector.http.enabled=true
dbms.connector.http.listen_address=0.0.0.0:7474

# 线程池
dbms.connector.bolt.thread_pool_max_size=200
dbms.connector.bolt.thread_pool_min_size=50

# 日志
dbms.logs.query.enabled=true
dbms.logs.query.threshold=1s

# 监控
metrics.enabled=true
metrics.prometheus.enabled=true
```

---

## 参考资源

- [毕昇JDK 官方仓库](https://gitee.com/openeuler/bishengjdk-11)
- [Neo4j 官方性能文档](https://neo4j.com/docs/operations-manual/current/performance/)
- [Neo4j 内存配置指南](https://neo4j.com/docs/operations-manual/current/performance/memory-configuration/)
- [鲲鹏开发者社区](https://www.hikunpeng.com/developer)
- [openEuler 官方文档](https://docs.openeuler.org/)

---

> **注意**：本文档基于 Neo4j 5.x 版本和鲲鹏920处理器编写，不同版本可能略有差异。生产环境部署前请进行充分的性能测试。
