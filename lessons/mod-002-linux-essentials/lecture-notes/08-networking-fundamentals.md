# 讲座 08：网络基础

## 目录
1. [简介](#简介)
2. [TCP/IP 网络基础](#tcpip-网络基础)
3. [网络配置](#网络配置)
4. [SSH 和远程访问](#ssh-和远程访问)
5. [网络诊断和排查](#网络诊断和排查)
6. [防火墙基础](#防火墙基础)
7. [AI 基础设施网络](#ai-基础设施网络)
8. [总结和要点](#总结和要点)

## 简介

网络是 AI 基础设施的基础。你需要为分布式训练配置网络、设置远程访问 GPU 服务器、排查连接问题以及保护 ML 服务安全。理解 Linux 网络使你能够构建、维护和排查生产 AI 系统。

本讲座涵盖 AI 基础设施工程的基本网络概念和实用技能。

### 学习目标

完成本讲座后，你将能够：
- 理解 TCP/IP 基础和网络寻址
- 配置网络接口和路由
- 使用 SSH 进行安全远程访问
- 使用命令行工具诊断网络问题
- 为 ML 服务配置防火墙
- 将网络概念应用于分布式 AI 基础设施
- 排查常见网络问题

### 先决条件
- 讲座 01-07（从 Linux 基础到文本处理）
- 基本理解 IP 地址和网络
- 可访问具有网络连接能力的 Linux 系统

### 为什么 AI 基础设施需要网络？

**分布式训练**：多节点训练需要可靠、高带宽的网络

**远程访问**：从任何地方管理 GPU 服务器和训练作业

**模型服务**：部署可通过网络访问的推理 API

**数据传输**：在存储和计算之间移动大型数据集

**安全**：保护 ML 服务免受未经授权的访问

**故障排查**：调试复杂基础设施中的连接问题

**时长**：120 分钟
**难度**：中级

---

## TCP/IP 网络基础

### OSI 和 TCP/IP 模型

**OSI 模型**（7 层）：
```
7. 应用层    - HTTP、SSH、FTP
6. 表示层    - 加密、压缩
5. 会话层    - 连接管理
4. 传输层    - TCP、UDP
3. 网络层    - IP、路由
2. 数据链路层 - 以太网、MAC 地址
1. 物理层    - 线缆、信号
```

**TCP/IP 模型**（4 层 - 实际重点）：
```
4. 应用层    - HTTP、SSH、DNS
3. 传输层    - TCP、UDP
2. 互联网层  - IP、ICMP
1. 链路层    - 以太网、WiFi
```

### IP 地址

**IPv4 地址**：32 位地址（例如 192.168.1.100）
- 4 个八位组，每个 0-255
- 约 43 亿个唯一地址

**IPv6 地址**：128 位地址（例如 2001:0db8:85a3::8a2e:0370:7334）
- 更大的地址空间
- 越来越重要

**私有 IP 范围**（不通过互联网路由）：
- 10.0.0.0/8（10.0.0.0 - 10.255.255.255）
- 172.16.0.0/12（172.16.0.0 - 172.31.255.255）
- 192.168.0.0/16（192.168.0.0 - 192.168.255.255）

**特殊地址**：
- 127.0.0.1 - 回环地址（localhost）
- 0.0.0.0 - 所有接口 / 默认路由
- 255.255.255.255 - 广播地址

### 子网划分基础

**CIDR 表示法**：IP/前缀（例如 192.168.1.0/24）

**常见子网掩码**：
- /8 = 255.0.0.0（16,777,214 个主机）
- /16 = 255.255.0.0（65,534 个主机）
- /24 = 255.255.255.0（254 个主机）
- /28 = 255.255.255.240（14 个主机）

### 端口和协议

**知名端口**（0-1023）：
- 22：SSH
- 80：HTTP
- 443：HTTPS
- 53：DNS

**AI/ML 服务端口**（示例）：
- 6006：TensorBoard
- 8888：Jupyter Notebook
- 8501：TensorFlow Serving REST
- 5000：Flask 默认
- 8000：FastAPI 默认

**TCP vs UDP**：
- **TCP**：面向连接、可靠、有序（web、SSH、数据库）
- **UDP**：无连接、快速、无保证（DNS、流媒体、游戏）

### DNS（域名系统）

**DNS 解析**：
```
用户输入：api.example.com
    ↓
DNS 查询 → DNS 服务器
    ↓
DNS 服务器返回：203.0.113.42
    ↓
连接到：203.0.113.42
```

**DNS 配置**：`/etc/resolv.conf`
```bash
nameserver 8.8.8.8          # Google DNS
nameserver 8.8.4.4          # Google DNS（备用）
nameserver 1.1.1.1          # Cloudflare DNS
```

**测试 DNS**：
```bash
# 测试 DNS 解析
nslookup google.com

# 更多详细信息
dig google.com
```

---

## 网络配置

### 查看网络接口

**ip 命令**（现代）：
```bash
# 显示所有接口
ip addr show
ip a                            # 短格式

# 显示特定接口
ip addr show eth0

# 显示简要摘要
ip -br addr
```

**ifconfig**（旧但仍常见）：
```bash
# 显示所有接口
ifconfig
```

**常见接口名称**：
- `eth0`、`eth1`：以太网（旧命名）
- `ens33`、`enp0s3`：以太网（新可预测命名）
- `wlan0`、`wlp2s0`：无线
- `lo`：回环（127.0.0.1）
- `docker0`：Docker 桥接

### 配置 IP 地址

**临时配置**（重启后丢失）：
```bash
# 分配 IP 地址
sudo ip addr add 192.168.1.100/24 dev eth0

# 删除 IP 地址
sudo ip addr del 192.168.1.100/24 dev eth0

# 启用接口
sudo ip link set eth0 up

# 禁用接口
sudo ip link set eth0 down
```

### 持久配置 - Netplan（Ubuntu 18.04+）

配置文件：`/etc/netplan/*.yaml`

**静态 IP 配置**：
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**应用配置**：
```bash
# 测试配置（提示确认）
sudo netplan try

# 应用配置
sudo netplan apply
```

### 路由

**查看路由表**：
```bash
# 显示路由
ip route show
ip r                            # 短格式

# 显示特定路由
ip route get 8.8.8.8
```

**添加/删除路由**：
```bash
# 添加静态路由
sudo ip route add 10.0.0.0/24 via 192.168.1.254

# 删除路由
sudo ip route del 10.0.0.0/24

# 添加默认网关
sudo ip route add default via 192.168.1.1
```

### 主机名配置

```bash
# 查看当前主机名
hostname
hostnamectl

# 设置主机名（持久）
sudo hostnamectl set-hostname ml-gpu-node-01
```

---

## SSH 和远程访问

SSH（安全 Shell）对于管理远程系统至关重要。

### 基本 SSH 用法

**连接到远程服务器**：
```bash
# 基本连接
ssh username@hostname
ssh user@192.168.1.100

# 指定端口
ssh -p 2222 user@hostname

# 执行单个命令
ssh user@hostname "ls -la"
ssh user@hostname "nvidia-smi"
```

### SSH 密钥认证

**生成 SSH 密钥对**：
```bash
# 生成 RSA 密钥（4096 位）
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# 生成 ED25519 密钥（现代、更安全）
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**将公钥复制到服务器**：
```bash
# 方法 1：ssh-copy-id（最简单）
ssh-copy-id user@hostname

# 方法 2：手动复制
cat ~/.ssh/id_rsa.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**设置正确权限**：
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### SSH 配置文件

**客户端配置**：`~/.ssh/config`

```bash
# 所有主机的默认设置
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3

# ML GPU 服务器
Host ml-gpu-1
    HostName 192.168.1.100
    User mluser
    Port 22
    IdentityFile ~/.ssh/ml_gpu_key
```

**使用配置**：
```bash
# 简单连接
ssh ml-gpu-1

# 而不是：
ssh -i ~/.ssh/ml_gpu_key -p 22 mluser@192.168.1.100
```

### 安全文件传输

**scp - 安全复制**：
```bash
# 复制文件到远程
scp local_file.txt user@hostname:/remote/path/

# 从远程复制文件
scp user@hostname:/remote/file.txt /local/path/

# 递归复制目录
scp -r local_directory user@hostname:/remote/path/
```

**rsync - 高效同步**：
```bash
# 基本同步
rsync -avz source/ user@hostname:/destination/

# -a：归档模式（保留权限、时间等）
# -v：详细模式
# -z：传输时压缩

# 带进度同步
rsync -avzP source/ user@hostname:/destination/

# 删除源端没有的目标端文件
rsync -avz --delete source/ user@hostname:/destination/
```

**ML 数据传输示例**：
```bash
# 将训练数据集同步到 GPU 服务器
rsync -avzP /local/datasets/imagenet/ ml-gpu-1:/data/imagenet/

# 将模型同步到生产环境
rsync -avz --delete /local/models/ production:/models/
```

### SSH 隧道

**本地端口转发**（在本地访问远程服务）：
```bash
# 将本地端口 8888 转发到远程端口 8888
ssh -L 8888:localhost:8888 user@remote-server

# 现在在 http://localhost:8888 访问远程 Jupyter
```

**远程端口转发**（远程访问本地服务）：
```bash
# 使本地端口 8000 在远程端口 9000 上可访问
ssh -R 9000:localhost:8000 user@remote-server
```

---

## 网络诊断和排查

### 连接测试

**ping - 测试可达性**：
```bash
# 基本 ping
ping google.com

# 限制为 4 个数据包
ping -c 4 google.com
```

**traceroute - 追踪网络路径**：
```bash
# 追踪到目的地的路由
traceroute google.com

# 仅显示 IP 地址
traceroute -n google.com
```

**mtr - 持续 traceroute**：
```bash
# 交互式网络诊断
mtr google.com
```

### 端口和服务测试

**nc（netcat）- 网络瑞士军刀**：
```bash
# 测试 TCP 端口
nc -zv hostname 80

# 测试多个端口
nc -zv hostname 80 443 8080

# 测试 UDP 端口
nc -zvu hostname 53
```

**nmap - 网络扫描器**：
```bash
# 扫描常见端口
nmap hostname

# 扫描特定端口
nmap -p 22,80,443 hostname
```

### 查看活动连接

**ss - 套接字统计**（现代替代 netstat）：
```bash
# 显示所有 TCP 连接
ss -ta

# 显示所有监听套接字
ss -tln

# 显示进程
sudo ss -tlnp
```

**lsof - 列出打开的文件**（包括网络）：
```bash
# 显示所有网络连接
sudo lsof -i

# 显示特定端口
sudo lsof -i :8080

# 哪个进程在使用端口？
sudo lsof -i :22
```

### DNS 故障排查

```bash
# 测试 DNS 解析
nslookup google.com

# 详细 DNS 查询
dig google.com

# 查询特定记录类型
dig google.com A           # IPv4 地址

# 反向 DNS 查找
dig -x 8.8.8.8
```

### AI 基础设施故障排查示例

**分布式训练无法连接**：
```bash
# 1. 检查节点间的网络连接
ping ml-worker-1

# 2. 检查训练端口是否开放
nc -zv ml-worker-1 6000

# 3. 检查防火墙
sudo ufw status

# 4. 检查什么在监听
sudo ss -tlnp | grep 6000
```

**无法访问 Jupyter 笔记本**：
```bash
# 1. Jupyter 是否在运行？
ps aux | grep jupyter

# 2. 它在哪个端口？
sudo lsof -i | grep jupyter

# 3. 本地能否访问？
curl http://localhost:8888

# 4. 使用 SSH 隧道
ssh -L 8888:localhost:8888 user@gpu-server
```

---

## 防火墙基础

防火墙根据规则控制网络流量。

### ufw - 不复杂的防火墙（Ubuntu/Debian）

**基本操作**：
```bash
# 启用防火墙
sudo ufw enable

# 禁用防火墙
sudo ufw disable

# 检查状态
sudo ufw status
```

**允许规则**：
```bash
# 允许 SSH（重要 - 不要把自己锁在外面！）
sudo ufw allow 22/tcp

# 允许 HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 允许来自特定 IP
sudo ufw allow from 192.168.1.50

# 允许来自特定 IP 到特定端口
sudo ufw allow from 192.168.1.50 to any port 22
```

**拒绝规则**：
```bash
# 拒绝特定端口
sudo ufw deny 3306/tcp
```

**ML 服务的防火墙**：
```bash
# 允许 SSH
sudo ufw allow 22/tcp

# 允许 Jupyter Notebook（限制到本地网络）
sudo ufw allow from 192.168.1.0/24 to any port 8888

# 允许 ML API
sudo ufw allow 8000/tcp

# 启用防火墙
sudo ufw enable
```

---

## AI 基础设施网络

### 分布式训练网络架构

**示例：4 节点 GPU 集群**

```
                     Internet
                         |
                    [Gateway]
                         |
           ┌────────────┴────────────┐
           |   管理网络               |
           |   (10.0.0.0/24)        |
           └────────────┬────────────┘
                        |
        ┌───────────────┼───────────────┐
        |               |               |
    [Master]        [Worker1]      [Worker2]
    10.0.0.10       10.0.0.11      10.0.0.12
        |               |               |
        └───────────────┼───────────────┘
                        |
           ┌────────────┴────────────┐
           |   训练网络               |
           |   (192.168.10.0/24)    |
           |   (高速/InfiniBand)    |
           └────────────┬────────────┘
                        |
        ┌───────────────┼───────────────┐
        |               |               |
    192.168.10.1  192.168.10.11   192.168.10.12
```

### 模型服务网络

**负载均衡 API 架构**：

```
            Internet
                |
           [Load Balancer]
                |
        ┌───────┴───────┐
        |               |
    [API-1]         [API-2]
    (Inference)     (Inference)
        |               |
        └───────┬───────┘
                |
          [Model Store]
          [Monitoring]
```

### 网络监控

```bash
# 监控带宽
iftop -i eth0

# 监控连接数
watch -n 1 'ss -tun | wc -l'
```

---

## 总结和要点

### 掌握的概念

**TCP/IP 基础**：
- IP 寻址和子网划分
- 端口和协议
- DNS 解析
- TCP vs UDP

**配置**：
- 查看接口（ip、ifconfig）
- 配置 IP（Netplan、NetworkManager）
- 路由表
- 主机名管理

**SSH**：
- 远程访问
- 密钥认证
- 文件传输（scp、rsync）
- SSH 隧道

**诊断**：
- 连接测试（ping、traceroute）
- 端口测试（nc、telnet、nmap）
- 连接查看（ss、netstat、lsof）
- DNS 故障排查（dig、nslookup）

**防火墙**：
- ufw（Ubuntu/Debian）
- firewalld（RHEL/CentOS）
- 规则管理
- 安全最佳实践

### 关键命令

```bash
# 查看网络配置
ip addr show
ip route show
cat /etc/resolv.conf

# 测试连接
ping host
traceroute host
nc -zv host port

# 查看连接
ss -tuln
sudo lsof -i :port

# SSH
ssh user@host
scp file user@host:/path/
rsync -avz source/ user@host:/dest/

# 防火墙
sudo ufw allow port/tcp
sudo ufw status
```

### AI 基础设施最佳实践

✅ **为服务器使用静态 IP**：可预测的寻址
✅ **分离网络**：管理、训练、存储
✅ **训练使用高带宽**：分布式训练需要 10+ Gbps
✅ **推理使用低延迟**：优化实时响应
✅ **安全的 SSH 访问**：仅密钥，无密码
✅ **默认防火墙**：显式允许，默认拒绝
✅ **监控网络健康**：带宽、延迟、错误
✅ **记录拓扑**：网络图、IP 分配
✅ **使用 SSH 隧道**：安全访问内部服务
✅ **彻底测试**：在生产前验证连接

### 排查检查清单

1. **物理/链路层**：接口是否启用？线缆是否连接？
2. **网络层**：IP 是否配置？能否 ping 通网关？
3. **路由**：能否到达外部 IP？
4. **DNS**：能否解析名称？
5. **防火墙**：端口是否允许？
6. **服务**：是否在监听？
7. **应用**：检查日志

### 快速参考卡

```bash
# 网络信息
ip addr show                    # 显示 IP
ip route show                   # 显示路由
hostname                        # 显示主机名

# 测试
ping host                       # 测试连接
nc -zv host port               # 测试端口
dig domain                      # DNS 查询

# 连接
ss -tuln                       # 监听端口
sudo lsof -i :port             # 端口使用情况

# SSH
ssh user@host                  # 连接
ssh-keygen -t ed25519          # 生成密钥
ssh-copy-id user@host          # 复制密钥
scp file user@host:/path/      # 复制文件
rsync -avz src/ user@host:dst/ # 同步

# 防火墙
sudo ufw allow port/tcp        # 允许端口
sudo ufw status                # 检查状态
sudo ufw enable                # 启用防火墙
```

---

**讲座 08：网络基础结束**

**恭喜完成模块 002：Linux 基础！**

你现在已经打下了 Linux 系统、shell 脚本、文本处理和网络的坚实基础——这些都是 AI 基础设施工程的基本技能。

**下一模块**：模块 003 - 使用 Git 的版本控制
