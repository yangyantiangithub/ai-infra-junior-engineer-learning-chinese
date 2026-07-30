# Lecture 03：用于 AI 基础设施的 AWS 网络与安全

## 讲座概述

安全、架构良好的网络基础设施是生产级 ML 系统的基础。本讲座涵盖 AWS 网络服务——VPC、子网、路由、安全组、NACL——以及如何为 AI/ML 工作负载设计隔离、安全的网络架构。你将学习构建多层网络，包括面向公网的负载均衡器、私有应用服务器和隔离的数据层，实施纵深防御安全原则。

到本讲座结束时，你将了解如何创建生产级 VPC 架构、遵循最小权限原则配置安全组、为合规实施网络分段以及排查连接问题。你还将学习跨账户访问和微服务架构中服务间认证的高级 IAM 模式。

**预计阅读时间：** 75-90 分钟
**动手实验：** 练习 03 – 网络与安全实验
**前置要求：** Lecture 02（AWS 核心服务）、IP 地址和子网划分的基础知识、熟悉 TCP/IP 协议

---

## 1. VPC（Virtual Private Cloud）基础

### 1.1 什么是 VPC？

**VPC = AWS 云中的隔离虚拟网络**

将 VPC 想象成你在 AWS 中的私有数据中心，你可以完全控制：
- **IP 地址范围**（CIDR 块）
- **子网**（网络段）
- **路由表**（流量如何流动）
- **网络网关**（互联网访问、VPN 连接）
- **安全策略**（防火墙、访问控制）

**默认 VPC：**
- 创建账户时 AWS 在每个区域创建一个默认 VPC
- 默认即可使用
- CIDR：172.31.0.0/16（65536 个 IP 地址）
- 每个 AZ 中的公有子网
- 已附加 Internet Gateway
- **适合入门，不推荐用于生产**

### 1.2 VPC CIDR 块和 IP 寻址

**CIDR 表示法：** `10.0.0.0/16`
- **网络部分**：前 16 位（10.0）
- **主机部分**：后 16 位（65536 个地址：10.0.0.0 - 10.0.255.255）

**常见 VPC CIDR 块：**

| CIDR | IP 范围 | 可用 IP | 用例 |
|------|----------|----------|------|
| 10.0.0.0/16 | 10.0.0.0 - 10.0.255.255 | 65531 | 大型生产 VPC |
| 10.0.0.0/24 | 10.0.0.0 - 10.0.0.255 | 251 | 小型开发/测试 VPC |
| 172.16.0.0/12 | 172.16.0.0 - 172.31.255.255 | 1048571 | 非常大的 VPC |
| 192.168.0.0/16 | 192.168.0.0 - 192.168.255.255 | 65531 | 常用于本地集成 |

**AWS 为每个子网保留 5 个 IP：**
- **x.x.x.0**：网络地址
- **x.x.x.1**：VPC 路由器
- **x.x.x.2**：DNS 服务器（AWS 提供）
- **x.x.x.3**：保留供将来使用
- **x.x.x.255**：广播地址（VPC 中不使用，但保留）

**示例：** 10.0.0.0/24 子网
- 总 IP：256
- 可用 IP：256 - 5 = **251**
- 第一个可用：10.0.0.4
- 最后一个可用：10.0.0.254

### 1.3 创建 VPC（CLI）

**创建 VPC：**

```bash
# 使用 10.0.0.0/16 CIDR 块创建 VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ml-production-vpc}]'

# 输出包含 VpcId（例如 vpc-0abcdef1234567890）

# 启用 DNS 主机名（RDS、ELB 需要）
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0abcdef1234567890 \
  --enable-dns-hostnames

# 启用 DNS 支持（默认已启用）
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0abcdef1234567890 \
  --enable-dns-support
```

**查看 VPC：**

```bash
# 列出所有 VPC
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table

# 输出：
-----------------------------------------------------
|                  DescribeVpcs                     |
+----------------------+----------------+-----------+
|  vpc-0abc123       |  10.0.0.0/16   |  ml-production-vpc |
|  vpc-default456    |  172.31.0.0/16 |  default-vpc       |
+----------------------+----------------+-----------+
```

---

## 2. 子网：网络分段

### 2.1 公有 vs 私有子网

**公有子网：**
- **路由到 Internet Gateway**（0.0.0.0/0 → igw-xxxxx）
- 实例具有**公有 IP 地址**
- **用例**：负载均衡器、堡垒机、NAT Gateway

**私有子网：**
- **没有直接到 Internet Gateway 的路由**
- 实例只有**私有 IP 地址**
- 通过公有子网中的 **NAT Gateway** 进行出站互联网访问
- **用例**：应用服务器、数据库（安全最佳实践）

### 2.2 多层网络架构

**典型生产 ML 架构：**

```
VPC：10.0.0.0/16

├─ 公有子网 1（10.0.1.0/24）[us-east-1a]
│  └─ Application Load Balancer（ALB）
│  └─ NAT Gateway
├─ 公有子网 2（10.0.2.0/24）[us-east-1b]
│  └─ ALB（用于高可用）
│  └─ NAT Gateway（用于高可用）
│
├─ 私有子网 1（10.0.11.0/24）[us-east-1a]
│  └─ EC2 推理服务器
│  └─ ECS/EKS 工作节点
├─ 私有子网 2（10.0.12.0/24）[us-east-1b]
│  └─ EC2 推理服务器（高可用）
│
├─ 数据子网 1（10.0.21.0/24）[us-east-1a]
│  └─ RDS 主库
│  └─ ElastiCache Redis
├─ 数据子网 2（10.0.22.0/24）[us-east-1b]
│  └─ RDS 备用库（Multi-AZ）
```

**为什么多层？**
- **安全**：互联网不能直接访问应用服务器或数据库
- **合规**：满足网络隔离要求（PCI-DSS、HIPAA）
- **纵深防御**：多层安全

### 2.3 创建子网（CLI）

**创建公有子网：**

```bash
# 在 us-east-1a 中创建公有子网
aws ec2 create-subnet \
  --vpc-id vpc-0abcdef1234567890 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a},{Key=Tier,Value=public}]'

# 启用自动分配公有 IP（使其成为"公有"子网）
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0abc111 \
  --map-public-ip-on-launch
```

**创建私有子网：**

```bash
# 在 us-east-1a 中创建私有子网
aws ec2 create-subnet \
  --vpc-id vpc-0abcdef1234567890 \
  --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1a},{Key=Tier,Value=private}]'

# 不启用自动分配公有 IP（使其成为"私有"子网）
```

**在多个 AZ 中创建子网（高可用性）：**

```bash
# 在 us-east-1b 中创建公有子网
aws ec2 create-subnet \
  --vpc-id vpc-0abcdef1234567890 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1b}]'

aws ec2 modify-subnet-attribute --subnet-id subnet-0abc222 --map-public-ip-on-launch

# 在 us-east-1b 中创建私有子网
aws ec2 create-subnet \
  --vpc-id vpc-0abcdef1234567890 \
  --cidr-block 10.0.12.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1b}]'
```

### 2.4 子网 IP 规划最佳实践

**为增长预留 CIDR 空间：**

```
VPC：10.0.0.0/16（65536 个 IP）

├─ 公有子网：10.0.0.0/20（4096 个 IP）
│  ├─ us-east-1a：10.0.1.0/24（256 个 IP）
│  ├─ us-east-1b：10.0.2.0/24（256 个 IP）
│  ├─ us-east-1c：10.0.3.0/24（256 个 IP）
│  └─ 预留：10.0.4.0/22（1024 个 IP 供将来 AZ 使用）
│
├─ 私有子网：10.0.16.0/20（4096 个 IP）
│  ├─ us-east-1a：10.0.16.0/24（256 个 IP）
│  ├─ us-east-1b：10.0.17.0/24（256 个 IP）
│  ├─ us-east-1c：10.0.18.0/24（256 个 IP）
│  └─ 预留：10.0.20.0/22（1024 个 IP）
│
├─ 数据子网：10.0.32.0/20（4096 个 IP）
│  ├─ us-east-1a：10.0.32.0/24（256 个 IP）
│  ├─ us-east-1b：10.0.33.0/24（256 个 IP）
│  └─ 预留：10.0.34.0/23（512 个 IP）
│
└─ 为将来层级预留：10.0.48.0/20 - 10.0.255.0/20
```

**提示：**
- 对大多数用例使用 /24 子网（256 个 IP）
- 对 EKS 节点池使用 /20 或 /22（许多 pod = 许多 IP）
- 为未来增长留出间隙
- 记录你的 IP 分配方案

---

## 3. Internet Gateway 和 NAT Gateway

### 3.1 Internet Gateway（IGW）

**Internet Gateway = VPC 通往互联网的门**

**特征：**
- 水平扩展、冗余、高可用（AWS 管理）
- 无带宽限制
- 免费（无小时费用，仅数据传出收费）

**创建并挂载 IGW：**

```bash
# 创建 Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=ml-vpc-igw}]'

# 输出：igw-0xyz789

# 附加到 VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0xyz789 \
  --vpc-id vpc-0abcdef1234567890
```

**为公有子网创建路由表：**

```bash
# 创建路由表
aws ec2 create-route-table \
  --vpc-id vpc-0abcdef1234567890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-route-table}]'

# 输出：rtb-0public123

# 添加指向 Internet Gateway 的路由（0.0.0.0/0 → IGW）
aws ec2 create-route \
  --route-table-id rtb-0public123 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0xyz789

# 将路由表与公有子网关联
aws ec2 associate-route-table \
  --route-table-id rtb-0public123 \
  --subnet-id subnet-0abc111  # public-subnet-1a

aws ec2 associate-route-table \
  --route-table-id rtb-0public123 \
  --subnet-id subnet-0abc222  # public-subnet-1b
```

### 3.2 NAT Gateway

**NAT Gateway = 允许私有子网实例访问互联网（仅出站）**

**为什么需要 NAT Gateway？**
- 私有子网实例需要下载包（apt update、pip install）
- 需要调用外部 API（S3、DynamoDB、外部服务）
- 但**不应**从互联网可达（安全）

**NAT Gateway vs NAT Instance：**

| 功能 | NAT Gateway（托管） | NAT Instance（EC2） |
|---------|----------------------|--------------------|
| **可用性** | AWS 托管高可用 | 手动（单实例） |
| **带宽** | 最高 45 Gbps | 取决于实例类型 |
| **维护** | AWS 处理 | 你打补丁/维护 |
| **成本** | $0.045/小时 + $0.045/GB | EC2 实例成本 |
| **推荐** | ✅ 是 | ❌ 否（传统） |

**创建 NAT Gateway：**

```bash
# 第 1 步：分配弹性 IP（NAT Gateway 的静态公有 IP）
aws ec2 allocate-address --domain vpc
# 输出：AllocationId：eipalloc-0natip123

# 第 2 步：在公有子网中创建 NAT Gateway
aws ec2 create-nat-gateway \
  --subnet-id subnet-0abc111 \
  --allocation-id eipalloc-0natip123 \
  --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=nat-gateway-1a}]'

# 输出：NatGatewayId：nat-0natgw123

# 等待 NAT Gateway 变为可用（大约 2-3 分钟）
aws ec2 wait nat-gateway-available --nat-gateway-ids nat-0natgw123
```

**为私有子网创建路由表：**

```bash
# 创建私有路由表
aws ec2 create-route-table \
  --vpc-id vpc-0abcdef1234567890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-route-table-1a}]'

# 输出：rtb-0private123

# 添加指向 NAT Gateway 的路由（0.0.0.0/0 → NAT Gateway）
aws ec2 create-route \
  --route-table-id rtb-0private123 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0natgw123

# 与私有子网关联
aws ec2 associate-route-table \
  --route-table-id rtb-0private123 \
  --subnet-id subnet-0private111  # private-subnet-1a
```

**高可用：每个 AZ 中的 NAT Gateway：**

```bash
# 在 us-east-1b 中创建 NAT Gateway（用于高可用）
aws ec2 allocate-address --domain vpc
# 输出：eipalloc-0natip456

aws ec2 create-nat-gateway \
  --subnet-id subnet-0abc222 \
  --allocation-id eipalloc-0natip456 \
  --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=nat-gateway-1b}]'

# 为 us-east-1b 中的私有子网创建单独的路由表
aws ec2 create-route-table \
  --vpc-id vpc-0abcdef1234567890 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-route-table-1b}]'

aws ec2 create-route \
  --route-table-id rtb-0private456 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0natgw456

aws ec2 associate-route-table \
  --route-table-id rtb-0private456 \
  --subnet-id subnet-0private222
```

**成本考量：**
- **NAT Gateway**：$0.045/小时 × 2 个 AZ × 730 小时 = 每月 $65.70
- **数据处理**：$0.045/GB 传输
- **开发替代方案**：单个 AZ 中的单个 NAT Gateway（非高可用，每月节省 $32.85）

---

## 4. 安全组：实例级防火墙

### 4.1 安全组基础

**安全组 = EC2 实例的有状态防火墙**

**关键特征：**
- **有状态**：如果允许入站请求，出站响应自动允许
- **默认拒绝**：除非明确允许，一切被阻止
- **仅允许规则**：不能创建拒绝规则（使用 NACL 实现）
- **实例级**：应用于网络接口（ENI）

**示例：Web 服务器安全组**

```bash
# 创建安全组
aws ec2 create-security-group \
  --group-name web-server-sg \
  --description "Allow HTTP/HTTPS from internet" \
  --vpc-id vpc-0abcdef1234567890

# 输出：GroupId：sg-0websg123

# 允许来自任何地方的 HTTP（端口 80）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0websg123 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# 允许来自任何地方的 HTTPS（端口 443）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0websg123 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# 仅允许来自你 IP 的 SSH（端口 22）（安全最佳实践）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0websg123 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.25/32  # 你的 IP 地址
```

**查看安全组规则：**

```bash
aws ec2 describe-security-groups --group-ids sg-0websg123

# 输出显示入站/出站规则
```

### 4.2 多层安全组架构

**最佳实践：引用其他安全组（而非 CIDR 块）**

```bash
# 创建 ALB 安全组（面向公网）
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Load balancer security group" \
  --vpc-id vpc-0abcdef1234567890

# 允许来自互联网的 HTTP/HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id sg-0albsg123 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-0albsg123 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# 创建应用服务器安全组
aws ec2 create-security-group \
  --group-name app-server-sg \
  --description "Application servers" \
  --vpc-id vpc-0abcdef1234567890

# 仅允许来自 ALB 安全组的流量（不来自互联网！）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0appsg123 \
  --protocol tcp --port 5000 \
  --source-group sg-0albsg123  # 引用 ALB SG，而非 CIDR

# 创建数据库安全组
aws ec2 create-security-group \
  --group-name database-sg \
  --description "RDS databases" \
  --vpc-id vpc-0abcdef1234567890

# 仅允许来自应用服务器的 PostgreSQL（端口 5432）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0dbsg123 \
  --protocol tcp --port 5432 \
  --source-group sg-0appsg123  # 仅来自应用服务器

# 创建堡垒机安全组
aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Bastion host for SSH access" \
  --vpc-id vpc-0abcdef1234567890

# 仅允许来自公司 VPN 的 SSH
aws ec2 authorize-security-group-ingress \
  --group-id sg-0bastionsg123 \
  --protocol tcp --port 22 \
  --cidr 198.51.100.0/24  # 公司 VPN CIDR

# 允许从堡垒机到应用服务器的 SSH
aws ec2 authorize-security-group-ingress \
  --group-id sg-0appsg123 \
  --protocol tcp --port 22 \
  --source-group sg-0bastionsg123
```

**安全组链：**

```
互联网
  ↓（允许 HTTP/HTTPS）
[ALB - sg-0albsg123]
  ↓（仅允许来自 ALB SG 的端口 5000）
[应用服务器 - sg-0appsg123]
  ↓（仅允许来自应用 SG 的端口 5432）
[数据库 - sg-0dbsg123]

[堡垒机 - sg-0bastionsg123]
  ↓（允许 SSH 到应用服务器）
[应用服务器 - sg-0appsg123]
```

**优点：**
- 如果扩展应用服务器（添加实例），它们自动获得数据库访问
- 无需用新的 IP 地址更新数据库安全组
- 清晰的安全边界

### 4.3 ML 基础设施的常见安全组模式

**1. ML 训练实例安全组：**

```bash
aws ec2 create-security-group \
  --group-name ml-training-sg \
  --description "GPU instances for model training" \
  --vpc-id vpc-0abcdef1234567890

# 仅允许来自堡垒机的 SSH
aws ec2 authorize-security-group-ingress \
  --group-id sg-0trainsg123 \
  --protocol tcp --port 22 \
  --source-group sg-0bastionsg123

# 仅允许来自 VPN 的 Jupyter notebook 访问（端口 8888）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0trainsg123 \
  --protocol tcp --port 8888 \
  --cidr 198.51.100.0/24  # 公司 VPN

# 允许 S3 访问（出站）- 默认允许所有出站，无需规则
```

**2. ML 推理服务安全组：**

```bash
aws ec2 create-security-group \
  --group-name ml-inference-sg \
  --description "Inference API servers" \
  --vpc-id vpc-0abcdef1234567890

# 仅允许来自 ALB 的 API 流量（端口 5000）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0infersg123 \
  --protocol tcp --port 5000 \
  --source-group sg-0albsg123

# 允许来自 ALB 的健康检查
#（如果健康检查使用相同端口，以上规则已涵盖）
```

**3. EKS 集群安全组：**

```bash
# EKS 控制平面安全组（AWS 管理，但你可以添加规则）
# EKS 工作节点安全组

aws ec2 create-security-group \
  --group-name eks-worker-sg \
  --description "EKS worker nodes" \
  --vpc-id vpc-0abcdef1234567890

# 允许来自控制平面的 kubelet API（端口 10250）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0eksworkersg123 \
  --protocol tcp --port 10250 \
  --source-group sg-0ekscontrolsg123

# 允许工作节点之间的所有流量（pod-to-pod 通信）
aws ec2 authorize-security-group-ingress \
  --group-id sg-0eksworkersg123 \
  --protocol -1 \
  --source-group sg-0eksworkersg123  # 自引用
```

### 4.4 安全组最佳实践

**1. 最小权限：**
- 只打开绝对必要的端口
- 将源限制到特定 IP/安全组，而非 0.0.0.0/0

**2. 使用描述性名称和标签：**
```bash
--tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=ml-inference-prod},{Key=Environment,Value=production},{Key=Team,Value=ml-platform}]'
```

**3. 定期审计：**
```bash
# 列出具有过于宽松规则（开放到 0.0.0.0/0）的安全组
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].{Name:GroupName,ID:GroupId}' \
  --output table
```

**4. 记录你的安全组：**
```bash
# 为规则添加描述（需要 JSON 输入）
cat > sg-rule.json <<'EOF'
{
  "IpPermissions": [{
    "IpProtocol": "tcp",
    "FromPort": 5000,
    "ToPort": 5000,
    "UserIdGroupPairs": [{
      "GroupId": "sg-0albsg123",
      "Description": "Allow inference API traffic from ALB"
    }]
  }]
}
EOF

aws ec2 authorize-security-group-ingress \
  --group-id sg-0infersg123 \
  --cli-input-json file://sg-rule.json
```

---

## 5. Network ACL（NACL）

### 5.1 NACL vs 安全组

| 功能 | 安全组 | NACL |
|---------|--------|-------|
| **层级** | 实例（ENI） | 子网 |
| **状态** | 有状态（返回流量自动允许） | 无状态（必须明确允许返回） |
| **规则** | 仅允许 | 允许和拒绝 |
| **规则评估** | 所有规则被评估 | 规则按编号顺序评估 |
| **默认** | 拒绝所有入站、允许所有出站 | 允许所有入站/出站 |
| **用例** | 主要防御 | 额外防御层 |

**何时使用 NACL：**
- 阻止特定 IP 地址（DDoS 缓解）
- 网络级控制的合规要求
- 纵深防御（安全组之外的额外层）

### 5.2 创建自定义 NACL

**创建 NACL：**

```bash
# 创建网络 ACL
aws ec2 create-network-acl \
  --vpc-id vpc-0abcdef1234567890 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=private-subnet-nacl}]'

# 输出：NetworkAclId：acl-0nacl123
```

**添加规则：**

```bash
# 允许来自任何地方的入站 HTTP（端口 80）
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# 允许入站 HTTPS（端口 443）
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --rule-number 110 \
  --protocol tcp \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# 允许入站临时端口（出站连接的返回流量）
# Linux 客户端使用端口 32768-60999，Windows 使用 49152-65535
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --rule-number 120 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# 拒绝来自特定恶意 IP 的流量
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --rule-number 50 \
  --protocol -1 \
  --cidr-block 203.0.113.50/32 \
  --rule-action deny

# 允许出站 HTTP/HTTPS
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --egress \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --egress \
  --rule-number 110 \
  --protocol tcp \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# 允许出站临时端口（返回流量）
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --egress \
  --rule-number 120 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# 将 NACL 与子网关联
aws ec2 replace-network-acl-association \
  --association-id aclassoc-0abc123 \
  --network-acl-id acl-0nacl123
```

**规则编号最佳实践：**
- 使用 10 的增量（100、110、120...）以允许以后插入规则
- 较低编号的规则先被评估
- 对拒绝规则使用小于 100 的编号（在允许规则之前评估）

### 5.3 示例：阻止特定 IP 范围

```bash
# 场景：阻止来自已知恶意 IP 范围 198.51.100.0/24 的流量

# 入站拒绝规则
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --rule-number 10 \
  --protocol -1 \
  --cidr-block 198.51.100.0/24 \
  --rule-action deny

# 出站拒绝规则（无状态，需要双向）
aws ec2 create-network-acl-entry \
  --network-acl-id acl-0nacl123 \
  --egress \
  --rule-number 10 \
  --protocol -1 \
  --cidr-block 198.51.100.0/24 \
  --rule-action deny
```

---

## 6. 网络安全的高级 IAM

### 6.1 VPC 端点：安全的 S3/DynamoDB 访问

**问题：** 从私有子网到 S3/DynamoDB 的流量经过 NAT Gateway → 互联网 → AWS（昂贵、安全性较差）

**解决方案：** VPC 端点（到 AWS 服务的私有连接）

**类型：**
- **Gateway Endpoint**：S3、DynamoDB（免费）
- **Interface Endpoint**：大多数其他服务（按小时 + 数据收费）

**创建 S3 VPC 端点：**

```bash
# 创建 S3 gateway 端点
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abcdef1234567890 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-0private123 rtb-0private456

# 现在 S3 流量留在 AWS 网络内，不使用 NAT Gateway
# 来自私有子网的流量：EC2 → S3 端点 → S3（永不经互联网）
```

**用于 VPC 端点的 S3 存储桶策略：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessFromVPCEndpointOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-ml-training-data-123456789",
        "arn:aws:s3:::my-ml-training-data-123456789/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0s3endpoint123"
        }
      }
    }
  ]
}
```

**优点：**
- **成本节省**：S3/DynamoDB 流量无需 NAT Gateway 数据费用
- **安全**：流量永不离开 AWS 网络
- **性能**：更低的延迟

### 6.2 带有网络条件的 IAM 策略

**限制 EC2 启动到特定 VPC：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:Vpc": "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0abcdef1234567890"
        }
      }
    }
  ]
}
```

**要求 MFA 用于安全组修改：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

### 6.3 跨账户 VPC 对等连接和 IAM

**VPC 对等连接：** 连接两个 VPC（同一账户或不同账户）进行私有通信。

**创建 VPC 对等连接：**

```bash
# 在账户 A（请求方）
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-0accountA123 \
  --peer-vpc-id vpc-0accountB456 \
  --peer-owner-id 987654321098 \
  --peer-region us-east-1

# 在账户 B（接受方）
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-0peering123

# 更新两个 VPC 中的路由表
# 账户 A：添加路由 10.1.0.0/16 → pcx-0peering123
aws ec2 create-route \
  --route-table-id rtb-0accountA123 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-0peering123

# 账户 B：添加路由 10.0.0.0/16 → pcx-0peering123
aws ec2 create-route \
  --route-table-id rtb-0accountB456 \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-0peering123

# 更新安全组以允许来自对等 VPC CIDR 的流量
```

**用例：账户 A 中的 ML 训练，账户 B 中的推理**
- 训练 VPC（账户 A）：10.0.0.0/16
- 推理 VPC（账户 B）：10.1.0.0/16
- 对等连接允许在账户 A 中训练的模型在账户 B 的推理环境中进行测试

---

## 7. 网络故障排查

### 7.1 VPC Flow Logs

**将 Flow Logs 启用并发送到 CloudWatch：**

```bash
# 创建 CloudWatch 日志组
aws logs create-log-group --log-group-name /aws/vpc/flowlogs

# 为 Flow Logs 创建 IAM 角色
cat > flow-logs-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "vpc-flow-logs.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name VPCFlowLogsRole \
  --assume-role-policy-document file://flow-logs-trust-policy.json

# 附加写入 CloudWatch 的策略
aws iam attach-role-policy \
  --role-name VPCFlowLogsRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# 创建 flow log
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0abcdef1234567890 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/VPCFlowLogsRole
```

**Flow Log 格式：**

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

示例：
2 123456789012 eni-0abc123 10.0.1.5 198.51.100.5 45678 443 6 10 5200 1634567890 1634567920 ACCEPT OK
```

**查询 Flow Logs（CloudWatch Logs Insights）：**

```sql
# 查找被拒绝的连接
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 100

# 最高流量源 IP
stats sum(bytes) as totalBytes by srcAddr
| sort totalBytes desc
| limit 10
```

### 7.2 Reachability Analyzer

**测试资源之间的连接：**

```bash
# 分析从 EC2 实例到 RDS 的路径
aws ec2 create-network-insights-path \
  --source eni-0source123 \
  --destination eni-0destination456 \
  --protocol tcp \
  --destination-port 5432

# 运行分析
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-0path123

# 检查结果
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids nia-0analysis123

# 输出显示：
# - Reachable：是/否
# - Path：源 → SG → 路由表 → NAT/IGW → 目标 SG
# - Blocking Rule：如果不可达，显示哪个 SG/NACL/路由在阻塞
```

### 7.3 常见连接问题

**1. 无法 SSH 到公有子网中的 EC2：**

检查清单：
- [ ] 实例有公有 IP？
- [ ] Internet Gateway 已附加到 VPC？
- [ ] 路由表有 0.0.0.0/0 → IGW？
- [ ] 安全组允许来自你 IP 的 SSH（端口 22）？
- [ ] NACL 允许入站 SSH 和出站临时端口？
- [ ] 使用正确的 SSH 密钥？

```bash
# 验证公有 IP
aws ec2 describe-instances --instance-ids i-0abc123 \
  --query 'Reservations[0].Instances[0].PublicIpAddress'

# 验证路由表
aws ec2 describe-route-tables --route-table-ids rtb-0public123

# 检查安全组
aws ec2 describe-security-groups --group-ids sg-0websg123
```

**2. 私有子网中的 EC2 无法访问互联网：**

检查清单：
- [ ] NAT Gateway 存在于公有子网中？
- [ ] 路由表有 0.0.0.0/0 → NAT Gateway？
- [ ] NAT Gateway 有弹性 IP？
- [ ] 安全组允许出站流量（默认为是）？
- [ ] NACL 允许出站 HTTP/HTTPS 和入站临时端口？

**3. RDS 不可从 EC2 访问：**

检查清单：
- [ ] RDS 安全组允许来自 EC2 安全组的流量？
- [ ] RDS 与 EC2 在同一个 VPC 中？
- [ ] 使用正确的端点主机名（而非 IP）？
- [ ] 允许端口 5432（PostgreSQL）或 3306（MySQL）？

```bash
# 从 EC2 测试连接
telnet ml-metadata-db.abc123.us-east-1.rds.amazonaws.com 5432

# 如果连接超时，检查安全组
```

---

## 8. 生产 VPC 架构示例

**完整 Terraform 模板（参考）：**

```hcl
# VPC
resource "aws_vpc" "ml_production" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "ml-production-vpc"
    Environment = "production"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.ml_production.id
}

# 公有子网
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.ml_production.id
  cidr_block        = cidrsubnet(aws_vpc.ml_production.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "public"
  }
}

# 私有子网
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.ml_production.id
  cidr_block        = cidrsubnet(aws_vpc.ml_production.cidr_block, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
    Tier = "private"
  }
}

# NAT Gateways（每个 AZ 一个用于高可用）
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

# 路由表
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.ml_production.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-route-table"
  }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.ml_production.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "private-route-table-${count.index + 1}"
  }
}

# 路由表关联
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# 安全组
resource "aws_security_group" "alb" {
  name        = "alb-sg"
  description = "Allow HTTP/HTTPS from internet"
  vpc_id      = aws_vpc.ml_production.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Allow traffic from ALB only"
  vpc_id      = aws_vpc.ml_production.id

  ingress {
    from_port       = 5000
    to_port         = 5000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "db" {
  name        = "db-sg"
  description = "Allow PostgreSQL from app servers only"
  vpc_id      = aws_vpc.ml_production.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# VPC 端点
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.ml_production.id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = concat(
    [aws_route_table.public.id],
    aws_route_table.private[*].id
  )
}
```

---

## 9. 关键要点

1. **VPC** 在 AWS 中提供隔离的网络；仔细规划 CIDR 块并为增长留出空间。

2. **多层架构**（公有/私有/数据子网）是生产的安全最佳实践。

3. **NAT Gateway** 允许私有实例访问互联网进行更新/API 调用，同时不被互联网访问。

4. **安全组**是有状态的实例级防火墙；对多层架构使用组引用（而非 CIDR）。

5. **NACL** 提供无状态的子网级防御；用于拒绝规则和合规要求。

6. **VPC 端点**通过将 S3/DynamoDB 流量保持在 AWS 网络内来节省成本并提高安全性。

7. **Flow Logs** 对故障排查和安全审计至关重要；在生产 VPC 上启用。

8. **Multi-AZ 部署**（每个 AZ 中的 NAT Gateway、跨 AZ 的子网）提供高可用性。

---

## 下一步是什么？

**Lecture 04** 介绍**部署和 ML 工作负载**——使用 Terraform 的基础设施即代码、使用 ECS/EKS 的容器编排、使用 Lambda 的无服务器 ML、托管 ML 平台（SageMaker）和成本优化策略。

**练习 03** 提供构建生产 VPC（带公有/私有子网）、为多层架构配置安全组、部署堡垒机以及实施 VPC 端点的动手实践。

---

## 延伸阅读

- **AWS VPC User Guide**：https://docs.aws.amazon.com/vpc/
- **AWS Well-Architected Framework - Security Pillar**：https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/
- **VPC Peering Guide**：https://docs.aws.amazon.com/vpc/latest/peering/
- **VPC Flow Logs**：https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html
- **AWS Security Best Practices**：https://aws.amazon.com/architecture/security-identity-compliance/

---

**End of Lecture 03**
