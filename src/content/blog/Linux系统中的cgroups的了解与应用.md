---
title: Linux系统中的cgroups深度解析与实战应用
date: 2024-01-20
tags:
  - Linux
  - cgroups
  - Docker
  - 资源管理
  - 系统调优
  - 容器化
description: 深入讲解Linux cgroups控制组的原理、配置和实际应用，包含进程资源限制、容器管理、性能调优等实用技能。
icon: layer-group
category:
  - 运维笔记
  - Linux
  - 系统管理
  - 容器技术
index: true
cover: https://picsum.photos/1200/600?random=18
---
# Linux系统中的cgroups深度解析与实战应用

cgroups（Control Groups）是Linux内核提供的一种机制，用于限制、记录和隔离进程组的资源使用。它是现代容器技术（如Docker、Kubernetes）的核心基础，也是系统资源管理的重要工具。

## 📋 目录

- [cgroups基础概念](#cgroups基础概念)
- [cgroups架构体系](#cgroups架构体系)
- [基础操作实战](#基础操作实战)
- [高级应用场景](#高级应用场景)
- [Docker与cgroups](#docker与cgroups)
- [性能监控与调优](#性能监控与调优)
- [故障排除指南](#故障排除指南)
- [最佳实践](#最佳实践)

## 💡 cgroups基础概念

### 什么是cgroups

cgroups是Linux内核的一个特性，它提供了以下核心功能：

- **资源限制**（Resource Limiting）：限制进程组使用的资源上限
- **优先级分配**（Prioritization）：通过分配更多的CPU时间片、磁盘IO带宽给重要进程
- **资源统计**（Accounting）：统计系统实际用了多少资源
- **任务控制**（Control）：对进程组执行挂起、恢复等操作

### 核心术语

| 术语 | 说明 | 示例 |
|------|------|------|
| **cgroup** | 控制组，一组进程和一组资源控制参数 | `/sys/fs/cgroup/memory/myapp` |
| **subsystem** | 子系统，特定的资源控制器 | memory、cpu、blkio等 |
| **hierarchy** | 层级树，cgroup的树形组织结构 | 父子关系的目录结构 |
| **task** | 任务，系统中的进程或线程 | PID 1234 |

### 主要子系统

| 子系统 | 功能 | 主要参数 |
|--------|------|----------|
| **cpu** | CPU时间分配 | `cpu.shares`, `cpu.cfs_quota_us` |
| **memory** | 内存使用限制 | `memory.limit_in_bytes`, `memory.usage_in_bytes` |
| **blkio** | 块设备IO限制 | `blkio.throttle.read_bps_device` |
| **devices** | 设备访问控制 | `devices.allow`, `devices.deny` |
| **freezer** | 进程挂起/恢复 | `freezer.state` |
| **net_cls** | 网络分类标记 | `net_cls.classid` |
| **hugetlb** | 大页内存限制 | `hugetlb.limit_in_bytes` |

## 🏗️ cgroups架构体系

### cgroups v1 vs v2

#### cgroups v1 (Legacy)
```bash
# 典型的v1目录结构
/sys/fs/cgroup/
├── memory/
│   ├── myapp/
│   │   ├── memory.limit_in_bytes
│   │   ├── memory.usage_in_bytes
│   │   └── cgroup.procs
├── cpu/
│   ├── myapp/
│   │   ├── cpu.shares
│   │   └── cgroup.procs
└── blkio/
    ├── myapp/
    └── ...
```

#### cgroups v2 (Unified Hierarchy)
```bash
# 统一的v2目录结构
/sys/fs/cgroup/
├── myapp/
│   ├── memory.max
│   ├── memory.current
│   ├── cpu.weight
│   ├── io.weight
│   └── cgroup.procs
└── cgroup.controllers
```

### 检查系统cgroups版本
```bash
# 检查系统支持的cgroups版本
mount | grep cgroup

# 查看可用的控制器
cat /proc/cgroups

# 检查cgroups v2支持
ls /sys/fs/cgroup/cgroup.controllers 2>/dev/null && echo "支持 cgroups v2" || echo "仅支持 cgroups v1"
```

## 🚀 基础操作实战

### 1. 安装cgroups工具

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install cgroup-tools cgroup-lite
```

#### CentOS/RHEL/Rocky Linux
```bash
sudo yum install libcgroup libcgroup-tools
# 或者在较新版本中
sudo dnf install libcgroup libcgroup-tools
```

#### Arch Linux
```bash
sudo pacman -S libcgroup
```

### 2. 内存限制实战

#### 创建内存控制组
```bash
# 使用cgcreate创建内存控制组
sudo cgcreate -g memory:/webapp

# 或者手动创建目录（cgroups v1）
sudo mkdir -p /sys/fs/cgroup/memory/webapp
```

#### 设置内存限制
```bash
# 设置内存限制为512MB
sudo cgset -r memory.limit_in_bytes=536870912 webapp

# 或者手动设置
echo 536870912 | sudo tee /sys/fs/cgroup/memory/webapp/memory.limit_in_bytes

# 设置内存软限制（建议值）
sudo cgset -r memory.soft_limit_in_bytes=268435456 webapp

# 禁用swap使用
sudo cgset -r memory.swappiness=0 webapp
```

#### 启动进程并分配到控制组
```bash
# 方法1: 使用cgexec启动进程
sudo cgexec -g memory:webapp python3 my_memory_intensive_app.py

# 方法2: 启动后分配进程
python3 my_app.py &
sudo cgclassify -g memory:webapp $!

# 方法3: 手动添加PID
echo $PID | sudo tee /sys/fs/cgroup/memory/webapp/cgroup.procs
```

### 3. CPU资源控制

#### 创建CPU控制组
```bash
sudo cgcreate -g cpu:/compute_intensive
```

#### 设置CPU限制
```bash
# 设置CPU权重（相对值，默认1024）
sudo cgset -r cpu.shares=512 compute_intensive

# 设置CPU配额（绝对值）- 限制使用50%的CPU
sudo cgset -r cpu.cfs_quota_us=50000 compute_intensive
sudo cgset -r cpu.cfs_period_us=100000 compute_intensive

# 绑定到特定CPU核心
sudo cgset -r cpuset.cpus=0,1 compute_intensive
sudo cgset -r cpuset.mems=0 compute_intensive
```

### 4. 磁盘IO控制

#### 创建blkio控制组
```bash
sudo cgcreate -g blkio:/database
```

#### 设置IO限制
```bash
# 获取设备号
lsblk
# 假设 /dev/sda 是 8:0

# 限制读取速度为100MB/s
sudo cgset -r blkio.throttle.read_bps_device="8:0 104857600" database

# 限制写入速度为50MB/s
sudo cgset -r blkio.throttle.write_bps_device="8:0 52428800" database

# 限制IOPS
sudo cgset -r blkio.throttle.read_iops_device="8:0 1000" database
sudo cgset -r blkio.throttle.write_iops_device="8:0 500" database
```

## 🔧 高级应用场景

### 1. 多层级资源分配

#### 创建层级结构
```bash
# 创建服务层级
sudo cgcreate -g memory,cpu:/services
sudo cgcreate -g memory,cpu:/services/web
sudo cgcreate -g memory,cpu:/services/database
sudo cgcreate -g memory,cpu:/services/cache

# 设置总体限制
sudo cgset -r memory.limit_in_bytes=8589934592 services  # 8GB

# 按服务分配资源
sudo cgset -r memory.limit_in_bytes=4294967296 services/web      # 4GB
sudo cgset -r memory.limit_in_bytes=3221225472 services/database # 3GB
sudo cgset -r memory.limit_in_bytes=1073741824 services/cache    # 1GB

# 设置CPU权重
sudo cgset -r cpu.shares=2048 services/web
sudo cgset -r cpu.shares=1024 services/database
sudo cgset -r cpu.shares=512 services/cache
```

### 2. 自动化脚本管理

#### 资源分配脚本
```bash
#!/bin/bash
# setup_cgroups.sh

set -e

# 配置变量
SERVICES=("web" "api" "worker" "cache")
MEMORY_LIMITS=("2G" "1G" "512M" "256M")
CPU_SHARES=(2048 1024 512 256)

# 创建主组
sudo cgcreate -g memory,cpu,blkio:/production

# 为每个服务创建子组
for i in "${!SERVICES[@]}"; do
    service="${SERVICES[$i]}"
    memory="${MEMORY_LIMITS[$i]}"
    cpu_share="${CPU_SHARES[$i]}"
    
    echo "配置服务: $service"
    
    # 创建子组
    sudo cgcreate -g memory,cpu,blkio:/production/$service
    
    # 设置内存限制
    memory_bytes=$(echo "$memory" | sed 's/G/000000000/g; s/M/000000/g; s/K/000/g')
    sudo cgset -r memory.limit_in_bytes=$memory_bytes production/$service
    
    # 设置CPU权重
    sudo cgset -r cpu.shares=$cpu_share production/$service
    
    # 设置IO权重
    sudo cgset -r blkio.weight=500 production/$service
    
    echo "服务 $service 配置完成"
done

echo "所有cgroups配置完成"
```

### 3. 进程监控和自动分配

#### 智能进程分类脚本
```bash
#!/bin/bash
# auto_classify.sh

while true; do
    # 查找高内存使用的进程
    high_memory_pids=$(ps aux --no-headers | awk '$4 > 10 {print $2}')
    
    for pid in $high_memory_pids; do
        # 检查进程是否已在cgroup中
        current_cgroup=$(cat /proc/$pid/cgroup 2>/dev/null | grep memory | cut -d: -f3)
        
        if [[ "$current_cgroup" == "/" ]]; then
            # 获取进程信息
            cmd=$(ps -p $pid -o comm= 2>/dev/null)
            
            case "$cmd" in
                "nginx"|"apache2"|"httpd")
                    echo $pid | sudo tee /sys/fs/cgroup/memory/production/web/cgroup.procs
                    echo "进程 $pid ($cmd) 分配到 web 组"
                    ;;
                "mysql"|"postgres"|"mongodb")
                    echo $pid | sudo tee /sys/fs/cgroup/memory/production/database/cgroup.procs
                    echo "进程 $pid ($cmd) 分配到 database 组"
                    ;;
                "redis"|"memcached")
                    echo $pid | sudo tee /sys/fs/cgroup/memory/production/cache/cgroup.procs
                    echo "进程 $pid ($cmd) 分配到 cache 组"
                    ;;
            esac
        fi
    done
    
    sleep 10
done
```

## 🐳 Docker与cgroups

### Docker资源限制详解

#### 内存限制
```bash
# 基础内存限制
docker run -m 512m nginx

# 内存+swap限制
docker run -m 512m --memory-swap 1g nginx

# 禁用swap
docker run -m 512m --memory-swap 512m nginx

# 内存预留
docker run -m 1g --memory-reservation 512m nginx
```

#### CPU限制
```bash
# CPU shares（相对权重）
docker run --cpu-shares 512 nginx

# CPU限制（绝对值）
docker run --cpus="1.5" nginx

# 指定CPU核心
docker run --cpuset-cpus="0,1" nginx

# CPU配额
docker run --cpu-period=100000 --cpu-quota=50000 nginx  # 50% CPU
```

#### IO限制
```bash
# 磁盘读写速度限制
docker run --device-read-bps /dev/sda:1mb nginx
docker run --device-write-bps /dev/sda:1mb nginx

# IOPS限制
docker run --device-read-iops /dev/sda:1000 nginx
docker run --device-write-iops /dev/sda:1000 nginx
```

### Docker Compose资源配置

#### docker-compose.yml示例
```yaml
version: '3.8'

services:
  web:
    image: nginx
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
  database:
    image: postgres
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
    
  cache:
    image: redis
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 128M
```

### 自定义Docker daemon配置

#### /etc/docker/daemon.json
```json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "default-runtime": "runc",
  "cgroup-parent": "docker",
  "default-shm-size": "64M",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

## 📊 性能监控与调优

### 实时监控脚本

#### 综合监控脚本
```bash
#!/bin/bash
# cgroup_monitor.sh

CGROUP_PATH="/sys/fs/cgroup"

function monitor_memory() {
    local group=$1
    local limit=$(cat "$CGROUP_PATH/memory/$group/memory.limit_in_bytes")
    local usage=$(cat "$CGROUP_PATH/memory/$group/memory.usage_in_bytes")
    local usage_percent=$((usage * 100 / limit))
    
    echo "内存 - $group: $(($usage/1024/1024))MB / $(($limit/1024/1024))MB ($usage_percent%)"
}

function monitor_cpu() {
    local group=$1
    local shares=$(cat "$CGROUP_PATH/cpu/$group/cpu.shares" 2>/dev/null || echo "N/A")
    local usage=$(cat "$CGROUP_PATH/cpu/$group/cpuacct.usage" 2>/dev/null || echo "0")
    
    echo "CPU - $group: shares=$shares, usage=$usage ns"
}

function monitor_blkio() {
    local group=$1
    local read_bytes=$(awk '{sum+=$3} END {print sum}' "$CGROUP_PATH/blkio/$group/blkio.throttle.io_service_bytes" 2>/dev/null || echo "0")
    local write_bytes=$(awk '{sum+=$3} END {print sum}' "$CGROUP_PATH/blkio/$group/blkio.throttle.io_service_bytes" 2>/dev/null || echo "0")
    
    echo "IO - $group: 读取=$(($read_bytes/1024/1024))MB, 写入=$(($write_bytes/1024/1024))MB"
}

# 监控指定的cgroups
GROUPS=("production/web" "production/database" "production/cache")

while true; do
    clear
    echo "=== cgroups 资源监控 ==="
    echo "时间: $(date)"
    echo

    for group in "${GROUPS[@]}"; do
        echo "=== $group ==="
        monitor_memory "$group"
        monitor_cpu "$group"
        monitor_blkio "$group"
        echo
    done
    
    sleep 5
done
```

### 性能调优建议

#### 1. 内存调优
```bash
# 启用内存压缩
echo 1 | sudo tee /sys/fs/cgroup/memory/webapp/memory.use_hierarchy

# 设置内存回收策略
echo 1 | sudo tee /sys/fs/cgroup/memory/webapp/memory.oom_control

# 配置swap使用倾向
echo 10 | sudo tee /sys/fs/cgroup/memory/webapp/memory.swappiness
```

#### 2. CPU调优
```bash
# 启用CPU带宽控制
echo 100000 | sudo tee /sys/fs/cgroup/cpu/webapp/cpu.cfs_period_us
echo 50000 | sudo tee /sys/fs/cgroup/cpu/webapp/cpu.cfs_quota_us

# 设置实时优先级
echo 50 | sudo tee /sys/fs/cgroup/cpu/webapp/cpu.rt_runtime_us
echo 100000 | sudo tee /sys/fs/cgroup/cpu/webapp/cpu.rt_period_us
```

## 🛠️ 故障排除指南

### 常见问题诊断

#### 1. 权限问题
```bash
# 检查cgroup挂载状态
mount | grep cgroup

# 检查目录权限
ls -la /sys/fs/cgroup/

# 修复权限问题
sudo chmod 755 /sys/fs/cgroup/memory/webapp
sudo chown root:root /sys/fs/cgroup/memory/webapp/*
```

#### 2. 进程OOM问题
```bash
# 查看OOM killer日志
dmesg | grep -i "killed process"

# 查看cgroup OOM事件
cat /sys/fs/cgroup/memory/webapp/memory.oom_control

# 增加内存限制
sudo cgset -r memory.limit_in_bytes=1073741824 webapp
```

#### 3. 性能问题诊断
```bash
# 检查CPU节流
cat /sys/fs/cgroup/cpu/webapp/cpu.stat

# 检查内存压力
cat /sys/fs/cgroup/memory/webapp/memory.pressure_level

# 查看IO等待
cat /sys/fs/cgroup/blkio/webapp/blkio.time
```

### 调试工具集合

#### 资源使用分析脚本
```bash
#!/bin/bash
# cgroup_debug.sh

CGROUP=$1

if [[ -z "$CGROUP" ]]; then
    echo "用法: $0 <cgroup_name>"
    exit 1
fi

echo "=== cgroup调试信息: $CGROUP ==="

# 内存信息
echo "内存统计:"
cat /sys/fs/cgroup/memory/$CGROUP/memory.stat | head -10

echo -e "\n内存使用:"
echo "当前使用: $(cat /sys/fs/cgroup/memory/$CGROUP/memory.usage_in_bytes | numfmt --to=iec)"
echo "限制: $(cat /sys/fs/cgroup/memory/$CGROUP/memory.limit_in_bytes | numfmt --to=iec)"

# CPU信息
echo -e "\nCPU统计:"
cat /sys/fs/cgroup/cpu/$CGROUP/cpu.stat

# 进程列表
echo -e "\n进程列表:"
cat /sys/fs/cgroup/memory/$CGROUP/cgroup.procs | while read pid; do
    if [[ -n "$pid" ]]; then
        ps -p $pid -o pid,ppid,cmd --no-headers 2>/dev/null || echo "进程 $pid 不存在"
    fi
done
```

## 🎯 最佳实践

### 1. 资源配置策略

#### 生产环境配置模板
```bash
#!/bin/bash
# production_cgroups_setup.sh

# 系统总资源
TOTAL_MEMORY_GB=16
TOTAL_CPU_CORES=8

# 资源分配比例
WEB_MEMORY_PERCENT=40
DB_MEMORY_PERCENT=35
CACHE_MEMORY_PERCENT=15
SYSTEM_MEMORY_PERCENT=10

# 计算具体数值
WEB_MEMORY=$((TOTAL_MEMORY_GB * WEB_MEMORY_PERCENT / 100))
DB_MEMORY=$((TOTAL_MEMORY_GB * DB_MEMORY_PERCENT / 100))
CACHE_MEMORY=$((TOTAL_MEMORY_GB * CACHE_MEMORY_PERCENT / 100))

echo "配置生产环境cgroups..."

# 创建层级结构
sudo cgcreate -g memory,cpu,blkio:/production
sudo cgcreate -g memory,cpu,blkio:/production/web
sudo cgcreate -g memory,cpu,blkio:/production/database
sudo cgcreate -g memory,cpu,blkio:/production/cache

# 内存限制
sudo cgset -r memory.limit_in_bytes=${WEB_MEMORY}G production/web
sudo cgset -r memory.limit_in_bytes=${DB_MEMORY}G production/database
sudo cgset -r memory.limit_in_bytes=${CACHE_MEMORY}G production/cache

# CPU权重
sudo cgset -r cpu.shares=2048 production/web
sudo cgset -r cpu.shares=1536 production/database
sudo cgset -r cpu.shares=512 production/cache

# IO优先级
sudo cgset -r blkio.weight=800 production/database
sudo cgset -r blkio.weight=500 production/web
sudo cgset -r blkio.weight=200 production/cache

echo "生产环境cgroups配置完成"
```

### 2. 自动化管理

#### systemd集成
```ini
# /etc/systemd/system/webapp.service
[Unit]
Description=Web Application
After=network.target

[Service]
Type=simple
User=webapp
Group=webapp
ExecStart=/usr/local/bin/webapp
Slice=production-web.slice
MemoryMax=2G
CPUWeight=100

[Install]
WantedBy=multi-user.target
```

#### cgroup定期清理脚本
```bash
#!/bin/bash
# cleanup_cgroups.sh

# 清理空的cgroups
find /sys/fs/cgroup/memory -name "cgroup.procs" -exec sh -c '
    if [ ! -s "$1" ]; then
        dir=$(dirname "$1")
        echo "清理空的cgroup: $dir"
        rmdir "$dir" 2>/dev/null || true
    fi
' _ {} \;

# 清理僵尸进程的cgroups
for cgroup in /sys/fs/cgroup/memory/*/cgroup.procs; do
    while read pid; do
        if [[ -n "$pid" ]] && ! kill -0 "$pid" 2>/dev/null; then
            echo "清理僵尸进程 $pid 从 $cgroup"
            # 进程已死亡，但PID可能还在cgroup中
        fi
    done < "$cgroup"
done
```

### 3. 监控告警

#### 资源使用告警脚本
```bash
#!/bin/bash
# cgroup_alerts.sh

MEMORY_THRESHOLD=90
CPU_THRESHOLD=80

function check_memory_usage() {
    local cgroup=$1
    local limit=$(cat "/sys/fs/cgroup/memory/$cgroup/memory.limit_in_bytes")
    local usage=$(cat "/sys/fs/cgroup/memory/$cgroup/memory.usage_in_bytes")
    local usage_percent=$((usage * 100 / limit))
    
    if [[ $usage_percent -gt $MEMORY_THRESHOLD ]]; then
        echo "警告: $cgroup 内存使用率 ${usage_percent}% 超过阈值 ${MEMORY_THRESHOLD}%"
        # 发送告警通知
        # send_alert "Memory alert for $cgroup: ${usage_percent}%"
    fi
}

function check_cpu_usage() {
    local cgroup=$1
    # 简化的CPU使用率检查
    local cpu_usage=$(cat "/sys/fs/cgroup/cpu/$cgroup/cpuacct.usage")
    # 这里需要更复杂的计算来得到实际的CPU使用率
    
    echo "CPU使用统计 $cgroup: $cpu_usage ns"
}

# 检查所有生产环境cgroups
CGROUPS=("production/web" "production/database" "production/cache")

for cgroup in "${CGROUPS[@]}"; do
    if [[ -d "/sys/fs/cgroup/memory/$cgroup" ]]; then
        check_memory_usage "$cgroup"
        check_cpu_usage "$cgroup"
    fi
done
```

## 🔗 扩展资源

### 相关工具推荐

1. **systemd-cgtop** - 实时查看cgroup资源使用
2. **htop** - 支持cgroup信息显示的系统监控
3. **docker stats** - Docker容器资源监控
4. **cadvisor** - 容器资源监控Web界面
5. **prometheus + node_exporter** - 指标收集和监控

### 学习资源

- [Red Hat cgroups Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/)
- [Linux Kernel cgroups Documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/)
- [Docker Resource Management](https://docs.docker.com/config/containers/resource_constraints/)

---
*cgroups是Linux系统资源管理的核心技术，掌握它对于系统管理员、DevOps工程师和容器技术从业者都至关重要。通过合理的资源控制，可以显著提升系统的稳定性和性能。*