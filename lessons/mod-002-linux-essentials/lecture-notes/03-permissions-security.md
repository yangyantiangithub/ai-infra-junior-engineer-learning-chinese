# 讲座 03：权限和安全

## 目录
1. [简介](#简介)
2. [理解 Linux 权限](#理解-linux-权限)
3. [用户和组](#用户和组)
4. [使用 chmod 管理权限](#使用-chmod-管理权限)
5. [使用 chown 更改所有权](#使用-chown-更改所有权)
6. [特殊权限](#特殊权限)
7. [默认权限和 umask](#默认权限和-umask)
8. [访问控制列表（ACL）](#访问控制列表-acl)
9. [AI 基础设施的安全最佳实践](#ai-基础设施的安全最佳实践)
10. [总结和要点](#总结和要点)

## 简介

安全在 AI 基础设施中至关重要。你将管理敏感数据集、专有模型、API 密钥和生产系统。理解 Linux 权限可以确保你在保护这些资产的同时，促进团队间的协作。

本讲座涵盖 Linux 权限模型、用户管理以及 AI 基础设施工程师所需的安全实践。

### 学习目标

完成本讲座后，你将能够：
- 理解 Linux 权限模型（读、写、执行）
- 使用符号和数字表示法解释和修改文件权限
- 有效管理用户和组
- 自信地使用 `chmod`、`chown` 和 `chgrp`
- 实现特殊权限（setuid、setgid、sticky bit）
- 使用 umask 配置默认权限
- 在复杂场景中应用访问控制列表（ACL）
- 遵循最佳实践保护 AI 基础设施

### 先决条件
- 讲座 01：Linux 基础
- 理解多用户系统
- 基本安全意识

### 为什么权限在 AI 基础设施中很重要

**数据隐私**：保护敏感的训练数据和 PII（个人身份信息）

**模型安全**：防止未经授权访问价值数百万 R&D 投入的专有模型

**访问控制**：管理谁能运行实验、访问 GPU 和修改生产系统

**合规性**：满足监管要求（GDPR、HIPAA、SOC 2）

**协作**：在保持安全边界的同时实现团队访问

## 理解 Linux 权限

Linux 中的每个文件和目录对三类用户有三个权限级别。

### 权限分类（谁）

1. **用户（u）**：文件所有者
2. **组（g）**：文件所属组的成员
3. **其他人（o）**：系统上的所有其他用户

### 权限类型（什么）

1. **读（r）**：查看文件内容或列出目录内容
2. **写（w）**：修改文件或在目录中创建/删除文件
3. **执行（x）**：将文件作为程序运行或访问目录

### 查看权限

```bash
ls -l train.py
# -rwxr-xr-- 1 alice mlteam 2048 Oct 15 14:30 train.py
# │└┬┘└┬┘└┬┘   │     │      │    │           │
# │ │  │  │    │     │      │    │           └─ 文件名
# │ │  │  │    │     │      │    └─ 修改日期
# │ │  │  │    │     │      └─ 文件大小
# │ │  │  │    │     └─ 组所有者
# │ │  │  │    └─ 文件所有者
# │ │  │  └─ 其他人权限（r--）
# │ │  └─ 组权限（r-x）
# │ └─ 用户/所有者权限（rwx）
# └─ 文件类型（- = 文件，d = 目录，l = 链接）
```

### 文件类型指示符

```bash
-    普通文件
d    目录
l    符号链接
b    块设备（磁盘驱动器）
c    字符设备（终端）
p    命名管道（FIFO）
s    套接字
```

### 权限按上下文的意义

**对于普通文件**：
- `r`（读）：可以查看文件内容（`cat`、`less`、`vim`）
- `w`（写）：可以修改文件内容
- `x`（执行）：可以将文件作为程序/脚本运行

**对于目录**：
- `r`（读）：可以列出目录内容（`ls`）
- `w`（写）：可以在目录中创建、删除、重命名文件
- `x`（执行）：可以访问目录（`cd`）、访问其中的文件

### 权限示例

```bash
# 文件示例
-rw-r--r--  config.yaml       # 所有者：读/写，组：读，其他人：读
-rwxr-xr-x  train.py          # 所有者：读/写/执行，组：读/执行，其他人：读/执行
-rw-------  api_key.txt       # 所有者：读/写，组：无，其他人：无（私有）
-rwxrwxrwx  dangerous.sh      # 每个人：完全访问（安全风险！）

# 目录示例
drwxr-xr-x  models/           # 所有者：完全，组：读/列出，其他人：读/列出
drwx------  .ssh/             # 所有者：完全，组：无，其他人：无（安全）
drwxrwxr-x  shared/           # 所有者+组：完全，其他人：读/列出
drwxrwxrwt  /tmp/             # 特殊：设置了 sticky bit
```

## 用户和组

### 理解 /etc/passwd

用户账户信息存储在 `/etc/passwd` 中：

```bash
cat /etc/passwd | grep alice
# alice:x:1001:1001:Alice Johnson:/home/alice:/bin/bash
#   │   │  │    │         │            │           └─ 默认 shell
#   │   │  │    │         │            └─ 主目录
#   │   │  │    │         └─ 全名/注释
#   │   │  │    └─ 主组 ID（GID）
#   │   │  └─ 用户 ID（UID）
#   │   └─ 密码（x = 存储在 /etc/shadow 中）
#   └─ 用户名
```

### 理解 /etc/group

组信息存储在 `/etc/group` 中：

```bash
cat /etc/group | grep mlteam
# mlteam:x:2001:alice,bob,charlie
#   │    │  │      └─ 组成员
#   │    │  └─ 组 ID（GID）
#   │    └─ 密码（很少使用）
#   └─ 组名
```

### 查看用户和组信息

```bash
# 当前用户
whoami                          # alice
id                              # uid=1001(alice) gid=1001(alice) groups=1001(alice),2001(mlteam)
groups                          # alice mlteam docker

# 特定用户
id bob                          # 显示 bob 的 UID、GID、组
groups bob                      # 显示 bob 的组

# 所有登录用户
who                             # 显示当前登录的用户
w                               # 显示谁登录了以及他们在做什么
```

### 创建用户（需要 sudo/root）

```bash
# 创建用户并创建主目录
sudo useradd -m -s /bin/bash charlie
# -m：创建主目录
# -s：设置默认 shell

# 设置密码
sudo passwd charlie

# 创建用户并指定 UID 和组
sudo useradd -m -u 1005 -g mlteam -G docker,admin dave
# -u：特定 UID
# -g：主组
# -G：附加组

# 现代替代方案（更用户友好）
sudo adduser charlie            # 交互式用户创建
```

### 创建组

```bash
# 创建组
sudo groupadd mlteam
sudo groupadd -g 2001 mlteam    # 指定 GID

# 将用户添加到组
sudo usermod -aG mlteam alice   # -aG：追加到组（不从其他组移除）
sudo usermod -aG docker alice   # 添加到 docker 组

# 从组中移除用户
sudo gpasswd -d alice mlteam

# 验证
groups alice
```

### 修改用户

```bash
# 更改用户的主组
sudo usermod -g newgroup alice

# 更改用户的主目录
sudo usermod -d /new/home/path alice

# 更改用户的 shell
sudo usermod -s /bin/zsh alice

# 更改用户名
sudo usermod -l newname oldname

# 锁定/解锁用户账户
sudo usermod -L alice           # 锁定
sudo usermod -U alice           # 解锁
```

### 删除用户和组

```bash
# 删除用户（保留主目录）
sudo userdel alice

# 删除用户和主目录
sudo userdel -r alice

# 删除组
sudo groupdel mlteam
```

### Sudo 和权限提升

`sudo` 命令允许授权用户以 root 身份运行命令。

**配置**：`/etc/sudoers`（只能使用 `visudo` 编辑！）

```bash
# 授予用户 sudo 访问权限
sudo usermod -aG sudo alice     # 在 Ubuntu/Debian 上
sudo usermod -aG wheel alice    # 在 RHEL/CentOS 上

# 以 root 身份运行命令
sudo apt update
sudo systemctl restart nginx

# 以不同用户身份运行命令
sudo -u bob python train.py

# 以 root 身份运行 shell（小心！）
sudo -i                         # 登录 shell
sudo -s                         # 保留环境

# 检查 sudo 访问权限
sudo -l                         # 列出允许的命令

# 使用 sudo 运行上一条命令
sudo !!
```

**最佳实践**：
- 对单个命令使用 `sudo`，而不是使用 `sudo su` 获取 root shell
- 定期审查 sudo 访问权限
- 记录 sudo 使用情况以便审计
- 尽可能使用特定的命令权限

## 使用 chmod 管理权限

`chmod`（change mode）修改文件和目录权限。

### 符号表示法

**语法**：`chmod [who][operation][permissions] file`

**谁**：`u`（用户）、`g`（组）、`o`（其他人）、`a`（所有）
**操作**：`+`（添加）、`-`（删除）、`=`（精确设置）
**权限**：`r`（读）、`w`（写）、`x`（执行）

**示例**：
```bash
# 为所有者添加执行权限
chmod u+x train.py

# 删除组和其他人的写权限
chmod go-w config.yaml

# 精确设置权限：所有者读/写，组读，其他人无
chmod u=rw,g=r,o= sensitive_data.csv

# 为所有人添加执行权限
chmod a+x script.sh
chmod +x script.sh              # a+x 的简写

# 删除其他人的所有权限
chmod o-rwx private_model.h5

# 使文件对所有人只读
chmod a-w readonly_config.yaml

# 多个操作
chmod u+x,go-w train.py         # 所有者：添加执行，组/其他人：删除写
```

### 数字（八进制）表示法

每个权限有一个数值：
- `r`（读）= 4
- `w`（写）= 2
- `x`（执行）= 1

为每个类别组合值：
- `7` = rwx（4+2+1）
- `6` = rw-（4+2）
- `5` = r-x（4+1）
- `4` = r--（4）
- `0` = ---（无）

**语法**：`chmod [user][group][others] file`

**常见模式**：
```bash
chmod 644 file.txt              # rw-r--r--（标准文件）
chmod 755 script.sh             # rwxr-xr-x（可执行脚本）
chmod 700 private.key           # rwx------（私有文件）
chmod 600 api_key.txt           # rw-------（机密）
chmod 777 dangerous.sh          # rwxrwxrwx（避免 - 安全风险！）
chmod 444 readonly.txt          # r--r--r--（只读）

# 目录
chmod 755 public_dir/           # rwxr-xr-x（标准目录）
chmod 750 team_dir/             # rwxr-x---（团队可访问）
chmod 700 private_dir/          # rwx------（私有目录）
chmod 775 shared_dir/           # rwxrwxr-x（团队可写）
```

### 递归权限更改

```bash
# 递归更改权限
chmod -R 755 project/           # 所有文件和目录

# 仅目录
find project/ -type d -exec chmod 755 {} \;

# 仅文件
find project/ -type f -exec chmod 644 {} \;

# 设置目录为 755，文件为 644
chmod -R u+rwX,g+rX,o+rX project/
# 大写 X：仅向目录和已可执行的文件添加执行权限
```

### AI 基础设施权限模式

```bash
# 模型文件（组只读）
chmod 640 models/*.h5
# rw-r-----：所有者可以更新，团队可以读取，其他人被阻止

# 训练脚本（可执行）
chmod 750 scripts/train.py
# rwxr-x---：所有者可以修改/运行，团队可以运行，其他人被阻止

# 配置文件
chmod 640 config/*.yaml
# rw-r-----：所有者可以编辑，团队可以读取

# API 密钥和机密
chmod 600 .env secrets.yaml
# rw-------：仅所有者，完全私密

# 共享数据集目录
chmod 775 /data/shared/
# rwxrwxr-x：团队可以添加/修改，其他人可以读取

# 日志文件
chmod 664 logs/*.log
# rw-rw-r--：所有者和组可以写，其他人可以读

# 临时处理目录
chmod 1777 /data/temp/
# rwxrwxrwt：每个人都可以使用，sticky bit 防止被其他人删除
```

### 权限故障排查

```bash
# 读取文件时"权限被拒绝"
ls -l file.txt                  # 检查你是否拥有读权限
# 修复：chmod u+r file.txt（如果你所有者）或联系所有者

# 运行脚本时"权限被拒绝"
ls -l script.sh                 # 检查执行权限
# 修复：chmod u+x script.sh

# 无法 cd 进入目录
ls -ld directory/               # 检查目录的执行权限
# 修复：chmod u+x directory/

# 无法在目录中创建文件
ls -ld directory/               # 检查写权限
# 修复：chmod u+w directory/

# 检查有效权限
namei -l /path/to/file          # 显示整个路径的权限
```

## 使用 chown 更改所有权

`chown`（change owner）修改文件的所有权。

### 基本语法

```bash
# 更改所有者
sudo chown alice file.txt

# 更改所有者和组
sudo chown alice:mlteam file.txt

# 仅更改组（使用 chown）
sudo chown :mlteam file.txt

# 递归
sudo chown -R alice:mlteam project/

# 保留符号链接
sudo chown -h alice link_name
```

### 使用 chgrp

`chgrp` 仅更改组所有者。

```bash
# 更改组
sudo chgrp mlteam file.txt

# 递归
sudo chgrp -R mlteam project/

# 详细输出
sudo chgrp -v mlteam file.txt
```

### 实际示例

```bash
# 员工离职时转移项目所有权
sudo chown -R newowner:mlteam /projects/alice_project/

# 设置共享数据集目录
sudo mkdir /data/shared/imagenet
sudo chown root:mlteam /data/shared/imagenet
sudo chmod 775 /data/shared/imagenet
# 现在 mlteam 成员可以添加/修改文件

# 提取归档后修复所有权
tar -xzf archive.tar.gz
sudo chown -R alice:mlteam extracted_directory/

# 为 web 服务器设置所有权
sudo chown -R www-data:www-data /var/www/ml-api/

# Docker 卷所有权
sudo chown -R 1000:1000 /docker/volumes/ml-data/
```

### 结合 chown 和 chmod

```bash
# 设置新项目目录
sudo mkdir /projects/new_model
sudo chown alice:mlteam /projects/new_model
sudo chmod 770 /projects/new_model
# 结果：Alice 拥有，mlteam 可以完全访问，其他人被阻止

# 保护机密文件
sudo chown alice:alice .api_key
sudo chmod 600 .api_key
# 结果：只有 alice 可以读/写

# 共享模型仓库
sudo mkdir /models/production
sudo chown root:mlteam /models/production
sudo chmod 2775 /models/production  # 注意：2 设置 setgid
# 结果：团队可以添加模型，所有新文件继承 mlteam 组
```

## 特殊权限

除了标准 rwx，Linux 还有三种特殊权限。

### Setuid（设置用户 ID）- 4

当在可执行文件上设置时，它以所有者的权限运行，而不是执行者的权限。

```bash
# 语法：chmod u+s file 或 chmod 4755 file
chmod u+s program
chmod 4755 program
# -rwsr-xr-x：注意所有者的 's' 代替 'x'

# 实际示例：passwd 命令
ls -l /usr/bin/passwd
# -rwsr-xr-x root root /usr/bin/passwd
# 任何用户都可以运行 passwd，但它以 root 身份执行（需要 root 权限修改 /etc/shadow）
```

**AI 基础设施用例**：
```bash
# 需要提升权限的 GPU 监控工具
sudo chown root:mlteam /usr/local/bin/gpu-monitor
sudo chmod 4750 /usr/local/bin/gpu-monitor
# mlteam 成员可以运行它，以 root 权限执行以访问 GPU 统计信息
```

**安全警告**：Setuid 如果被滥用则非常危险。仔细审查任何 setuid 程序。

### Setgid（设置组 ID）- 2

**在可执行文件上**：以文件的组权限运行，而不是执行者的组。

**在目录上**：新文件继承目录的组，而不是创建者的主组。

```bash
# 语法：chmod g+s file/dir 或 chmod 2755 file/dir
chmod g+s directory/
chmod 2775 directory/
# drwxrwsr-x：注意组的 's' 代替 'x'

# 实际用途：共享项目目录
sudo mkdir /projects/shared
sudo chown root:mlteam /projects/shared
sudo chmod 2775 /projects/shared
# 现在当任何人在这里创建文件时，它会自动获得组 'mlteam'
```

**示例**：
```bash
# 没有 setgid
cd /normal/directory
touch file.txt
ls -l file.txt
# -rw-r--r-- alice alice file.txt（创建者的主组）

# 使用 setgid
cd /projects/shared  # 设置了 setgid，组为 mlteam
touch file.txt
ls -l file.txt
# -rw-r--r-- alice mlteam file.txt（继承了目录的组！）
```

### Sticky Bit - 1

防止用户在目录中删除不属于他们的文件，即使他们有写权限。

```bash
# 语法：chmod +t directory 或 chmod 1777 directory
chmod +t /shared/temp/
chmod 1777 /shared/temp/
# drwxrwxrwt：注意其他人的 't' 代替 'x'

# 经典示例：/tmp
ls -ld /tmp
# drwxrwxrwt root root /tmp
# 任何人都可以创建文件，但只能删除自己的文件
```

**AI 基础设施用例**：
```bash
# 共享临时处理目录
sudo mkdir /data/temp
sudo chmod 1777 /data/temp
# 每个人都可以创建临时文件，但不能删除其他人的文件
```

### 查看特殊权限

```bash
# 查找 setuid 文件（潜在安全风险）
find / -perm -4000 -type f 2>/dev/null

# 查找 setgid 文件
find / -perm -2000 -type f 2>/dev/null

# 查找 sticky bit 目录
find / -perm -1000 -type d 2>/dev/null

# 查找具有任何特殊权限的文件
find /path -perm /7000 -ls
```

### 带特殊权限的数字表示法

```
[special][user][group][others]
   4       7     5      5

4000 = setuid
2000 = setgid
1000 = sticky bit
```

**示例**：
```bash
chmod 4755 file     # setuid，rwxr-xr-x
chmod 2755 dir      # setgid，rwxr-sr-x
chmod 1777 dir      # sticky，rwxrwxrwt
chmod 6755 file     # setuid+setgid，rwsr-sr-x
chmod 7755 dir      # 所有特殊权限
```

## 默认权限和 umask

`umask` 设置新创建文件和目录的默认权限。

### 理解 umask

**umask** 是一个从默认权限中*减去*的掩码：
- 文件默认：`666`（rw-rw-rw-）
- 目录默认：`777`（rwxrwxrwx）
- 从这些默认值中减去 umask

**常见 umask 值**：
```
umask 022：
  文件：666 - 022 = 644（rw-r--r--）
  目录：777 - 022 = 755（rwxr-xr-x）

umask 002：
  文件：666 - 002 = 664（rw-rw-r--）
  目录：777 - 002 = 775（rwxrwxr-x）

umask 077：
  文件：666 - 077 = 600（rw-------）
  目录：777 - 077 = 700（rwx------）

umask 027：
  文件：666 - 027 = 640（rw-r-----）
  目录：777 - 027 = 750（rwxr-x---）
```

### 查看和设置 umask

```bash
# 查看当前 umask
umask
# 0022

# 以符号表示法查看
umask -S
# u=rwx,g=rx,o=rx

# 为当前会话设置 umask
umask 027                       # 更严格
umask 002                       # 更宽松

# 测试新的 umask
touch testfile
mkdir testdir
ls -ld testfile testdir
```

### 使 umask 持久化

```bash
# 对于单个用户：~/.bashrc 或 ~/.profile
echo "umask 027" >> ~/.bashrc

# 对于所有用户：/etc/profile 或 /etc/bash.bashrc
sudo sh -c 'echo "umask 027" >> /etc/profile'

# 对于特定组（在其 shell 配置中）
if groups | grep -q mlteam; then
    umask 002  # 团队协作宽松
else
    umask 077  # 其他人严格
fi
```

### AI 基础设施建议

```bash
# 普通用户：umask 022（标准）
# 文件：644，目录：755

# 开发团队：umask 002（协作）
# 文件：664，目录：775
# 结合共享目录的 setgid

# 安全敏感：umask 077（严格）
# 文件：600，目录：700
# 用于处理敏感数据的系统

# 推荐给团队环境中的 ML 工程师
umask 002
```

## 访问控制列表（ACL）

ACL 提供了超越用户/组/其他人的细粒度权限。

### 为什么需要 ACL？

标准权限限于一个用户和一个组。ACL 允许：
- 多个用户具有不同权限
- 多个组具有不同权限
- 目录中新文件的默认权限

### 查看 ACL

```bash
# 检查文件是否有 ACL
ls -l file.txt
# -rw-r--r--+：'+' 表示设置了 ACL

# 查看 ACL 详情
getfacl file.txt
# # file: file.txt
# # owner: alice
# # group: mlteam
# user::rw-
# user:bob:r--
# user:charlie:rw-
# group::r--
# group:dataeng:rw-
# mask::rw-
# other::r--
```

### 设置 ACL

```bash
# 授予用户 bob 读访问权限
setfacl -m u:bob:r file.txt

# 授予用户 charlie 读+写访问权限
setfacl -m u:charlie:rw file.txt

# 授予组 dataeng 读+写访问权限
setfacl -m g:dataeng:rw file.txt

# 删除特定 ACL 条目
setfacl -x u:bob file.txt

# 删除所有 ACL
setfacl -b file.txt

# 递归 ACL
setfacl -R -m u:bob:r directory/
```

### 目录的默认 ACL

```bash
# 为目录中新文件设置默认 ACL
setfacl -d -m u:bob:rw /projects/shared/
# 现在 /projects/shared/ 中创建的所有新文件将授予 bob rw 访问权限

# 与正常 ACL 结合
setfacl -m u:bob:rwx /projects/shared/          # 用于现有内容
setfacl -d -m u:bob:rw /projects/shared/        # 用于新内容

# 查看默认 ACL
getfacl /projects/shared/
```

### 实际 ACL 示例

```bash
# 场景：多个团队需要不同级别的数据集访问权限
sudo setfacl -m g:mlteam:rwx /data/shared-dataset/
sudo setfacl -m g:dataeng:r-x /data/shared-dataset/
sudo setfacl -m g:analysts:r-- /data/shared-dataset/
# mlteam：完全访问，dataeng：读+列出，analysts：仅读

# 授予特定用户模型仓库的访问权限
setfacl -m u:alice:rwx /models/production/
setfacl -m u:bob:r-x /models/production/
# alice 可以修改，bob 可以读取

# 设置带默认值的共享项目
sudo setfacl -m g:mlteam:rwx /projects/experiment/
sudo setfacl -d -m g:mlteam:rwx /projects/experiment/
sudo setfacl -d -m u::rwx /projects/experiment/
sudo setfacl -d -m o::--- /projects/experiment/
# 团队有完全访问权限，新文件继承团队访问权限，其他人被阻止
```

### 复制和备份 ACL

```bash
# 备份 ACL
getfacl -R /projects/shared > acl_backup.txt

# 恢复 ACL
setfacl --restore=acl_backup.txt

# 从一个目录复制 ACL 到另一个
getfacl /source/dir | setfacl --set-file=- /dest/dir
```

## AI 基础设施的安全最佳实践

### 1. 最小权限原则

授予最低必要权限。

```bash
# 不好：每个人有完全访问权限
chmod 777 sensitive_data/

# 好：仅特定访问
chmod 750 sensitive_data/
chown alice:mlteam sensitive_data/
# Alice：完全控制，mlteam：读+执行，其他人：无
```

### 2. 保护敏感数据

```bash
# API 密钥和机密
chmod 600 .env api_keys.yaml
chown alice:alice .env

# SSH 密钥
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 644 ~/.ssh/authorized_keys

# 模型权重（专有）
chmod 640 models/proprietary_*.h5
chown alice:mlteam models/proprietary_*.h5

# 数据库凭据
chmod 600 database.conf
```

### 3. 保护共享资源

```bash
# 共享数据集 - 大多数只读
sudo mkdir /data/shared/imagenet
sudo chown root:mlteam /data/shared/imagenet
sudo chmod 755 /data/shared/imagenet
# 只有 root 可以修改，每个人都可以读取

# 团队工作空间 - 协作
sudo mkdir /projects/team-workspace
sudo chown root:mlteam /projects/team-workspace
sudo chmod 2775 /projects/team-workspace
# 团队可以协作，新文件继承 mlteam 组

# 临时处理 - 带 sticky bit
sudo mkdir /data/temp-processing
sudo chmod 1777 /data/temp-processing
# 每个人都可以创建文件，不能删除其他人的文件
```

### 4. 定期审计权限

```bash
# 查找全局可写文件（安全风险）
find /home -type f -perm -002 2>/dev/null

# 查找 setuid/setgid 文件（潜在权限提升）
find / -perm -4000 -o -perm -2000 -type f 2>/dev/null | tee setuid_files.txt

# 查找无所有者的文件（孤立文件）
find / -nouser -o -nogroup 2>/dev/null

# 查找最近修改的文件
find /etc -mtime -7 -type f

# 检查谁可以访问敏感目录
ls -la /data/sensitive/
getfacl /data/sensitive/
```

### 5. 实现基于角色的访问

```bash
# 为角色创建组
sudo groupadd ml-developers
sudo groupadd ml-researchers
sudo groupadd ml-operations
sudo groupadd data-engineers

# 将用户分配到适当的组
sudo usermod -aG ml-developers alice
sudo usermod -aG ml-researchers bob
sudo usermod -aG ml-operations charlie

# 设置基于角色的目录权限
sudo mkdir /projects/{development,research,production}
sudo chown root:ml-developers /projects/development
sudo chown root:ml-researchers /projects/research
sudo chown root:ml-operations /projects/production
sudo chmod 2770 /projects/*
```

### 6. 保护 GPU 访问

```bash
# 限制 GPU 访问到特定组
sudo groupadd gpu-users
sudo usermod -aG gpu-users alice

# 设置 GPU 设备权限（在 /etc/udev/rules.d/ 中）
sudo sh -c 'echo "KERNEL==\"nvidia*\", GROUP=\"gpu-users\", MODE=\"0660\"" > /etc/udev/rules.d/70-gpu-access.rules'
sudo udevadm control --reload-rules
```

### 7. 日志文件安全

```bash
# 日志应该可读但不可由应用程序写入
chmod 644 /var/log/ml-training/*.log
chown root:adm /var/log/ml-training/

# 敏感日志
chmod 640 /var/log/auth.log
chmod 640 /var/log/ml-api/access.log
```

### 8. 脚本安全

```bash
# 测试后使脚本只读
chmod 555 production_train.py

# 包含机密的脚本应受限制
chmod 700 deploy_with_credentials.sh

# 为团队脚本使用 setgid
sudo chown root:mlteam /scripts/shared_train.sh
sudo chmod 2755 /scripts/shared_train.sh
```

### 9. 自动化权限检查

```bash
# 创建审计脚本
cat > /usr/local/bin/permission-audit.sh << 'EOF'
#!/bin/bash
echo "=== Permission Audit $(date) ===" | tee -a /var/log/permission-audit.log
echo "World-writable files:"
find /home -type f -perm -002 2>/dev/null | tee -a /var/log/permission-audit.log
echo "Setuid/Setgid files:"
find / -perm -4000 -o -perm -2000 -type f 2>/dev/null | tee -a /var/log/permission-audit.log
echo "Orphaned files:"
find /home -nouser -o -nogroup 2>/dev/null | tee -a /var/log/permission-audit.log
EOF

sudo chmod 755 /usr/local/bin/permission-audit.sh

# 安排每周审计（添加到 crontab）
sudo crontab -e
# 0 2 * * 0 /usr/local/bin/permission-audit.sh
```

### 10. Docker 和容器安全

```bash
# Docker 卷应有适当的所有权
sudo chown -R 1000:1000 /docker/volumes/ml-data
sudo chmod 755 /docker/volumes/ml-data

# Docker 守护进程组（允许无 sudo 访问 Docker）
sudo groupadd docker
sudo usermod -aG docker alice
# 注意：这很方便但会降低安全性 - 谨慎使用
```

## 总结和要点

### 掌握的命令

**权限命令**：
- `chmod` - 更改文件权限（符号或数字）
- `chown` - 更改文件所有者和组
- `chgrp` - 更改文件组
- `umask` - 设置新文件的默认权限

**用户/组管理**：
- `useradd`/`adduser` - 创建用户
- `usermod` - 修改用户
- `userdel` - 删除用户
- `groupadd` - 创建组
- `gpasswd` - 管理组成员身份

**ACL 命令**：
- `getfacl` - 查看 ACL
- `setfacl` - 设置 ACL

**信息命令**：
- `whoami` - 当前用户
- `id` - 用户/组 ID
- `groups` - 用户组
- `who`/`w` - 登录用户

### 权限模式快速参考

```bash
# 文件
600（rw-------）  机密文件（API 密钥、私钥）
640（rw-r-----）  共享配置，组可读
644（rw-r--r--）  标准文件，所有人可读
664（rw-rw-r--）  团队可编辑文件
755（rwxr-xr-x）  可执行脚本，公共

# 目录
700（rwx------）  私有目录
750（rwxr-x---）  团队可读目录
755（rwxr-xr-x）  公共目录
770（rwxrwx---）  团队协作目录
775（rwxrwxr-x）  团队可写，公共可读
1777（rwxrwxrwt） 共享临时（sticky bit）
2775（rwxrwsr-x） 带 setgid 的团队协作
```

### 关键概念

1. **三个权限级别**：用户、组、其他人（ugo）
2. **三种权限类型**：读、写、执行（rwx）
3. **两种表示法**：符号（u+x）和数字（755）
4. **特殊权限**：setuid（4）、setgid（2）、sticky bit（1）
5. **默认权限**：由 umask 控制
6. **ACL**：超越标准 ugo 的细粒度权限

### 安全原则

✅ **最小权限原则**：最低必要访问
✅ **纵深防御**：多层安全
✅ **定期审计**：定期检查权限
✅ **记录访问**：知道谁有什么访问权限
✅ **安全默认**：严格的 umask
✅ **基于组的访问**：使用组进行团队访问
✅ **保护机密**：敏感文件使用 600 权限

### AI 基础设施最佳实践

1. **数据集安全**：大多数用户只读（目录 755，文件 644）
2. **模型保护**：限制写权限（专有模型使用 640）
3. **团队协作**：使用带组访问的 setgid 目录（2775）
4. **API 密钥**：严格权限（600），仅所有者访问
5. **共享资源**：临时目录使用 sticky bit（1777）
6. **GPU 访问**：基于组的访问控制
7. **日志文件**：应用程序可读但不可写（644）
8. **脚本**：生产环境中只读（555 或 755）

### 常见错误避免

❌ `chmod 777` - 除非绝对必要且临时，否则永远不要使用
❌ 以 root 身份运行服务 - 使用专用服务账户
❌ 将机密存储在全局可读文件中
❌ 提取归档时忽略文件所有权
❌ 忘记为团队环境设置 umask
❌ 不在共享团队目录上设置 setgid
❌ 对复杂权限场景忽视 ACL
❌ 不定期审计权限

### 故障排查指南

**问题**：读取文件时"权限被拒绝"
- **检查**：`ls -l file` - 你有读权限吗？
- **修复**：`chmod u+r file`（如果是所有者）或联系所有者

**问题**：无法写入目录
- **检查**：`ls -ld directory` - 你有写+执行权限吗？
- **修复**：`chmod u+wx directory`

**问题**：脚本无法执行
- **检查**：`ls -l script.sh` - 是否设置了执行权限？
- **修复**：`chmod u+x script.sh`

**问题**：创建的文件有错误的组
- **检查**：`ls -ld directory` - 设置了 setgid 吗？
- **修复**：`chmod g+s directory`

**问题**：其他用户在共享目录中删除我的文件
- **检查**：目录权限
- **修复**：`chmod +t directory`（sticky bit）

### 练习

1. 创建具有适当权限的多用户 ML 项目结构
2. 使用 ACL 设置安全的共享数据集目录
3. 为团队协作配置 umask
4. 审计系统的全局可写文件
5. 为 ML 团队实现基于角色的访问
6. 保护 API 密钥和凭据
7. 设置 setgid 目录以实现自动组继承

### 下一步

在**讲座 03：进程管理**中，你将学习：
- 理解进程和系统资源
- 使用 ps、top、htop 进行监控
- 使用 systemd 管理服务
- 作业控制和后台进程
- 资源限制和优先级

### 额外资源

- **Man 手册页**：`man chmod`、`man chown`、`man setfacl`、`man umask`
- **Linux 安全**："Understanding Linux File Permissions"（LinuxAcademy）
- **ACL 指南**：Red Hat ACL 文档
- **安全加固**：CIS Linux 基准

### 快速参考卡

```bash
# 查看权限
ls -l file                    # 标准查看
ls -ld directory             # 目录本身
getfacl file                 # ACL 查看
id                           # 当前用户信息
groups                       # 当前用户组

# 更改权限
chmod 644 file               # 数字
chmod u+x file               # 符号
chmod -R 755 dir/            # 递归

# 更改所有权
chown user:group file        # 所有者和组
chown -R user:group dir/     # 递归
chgrp group file             # 仅组

# 特殊权限
chmod u+s file               # Setuid
chmod g+s dir                # Setgid
chmod +t dir                 # Sticky bit
chmod 4755 file              # Setuid 数字

# ACL
setfacl -m u:bob:rw file     # 授予用户访问
setfacl -m g:team:r file     # 授予组访问
setfacl -x u:bob file        # 删除 ACL 条目
setfacl -b file              # 删除所有 ACL

# 默认权限
umask                        # 查看 umask
umask 027                    # 设置 umask
```

---

**讲座 02 结束**

继续学习**讲座 03：进程管理**，了解如何监控和管理运行中的进程和系统服务。
