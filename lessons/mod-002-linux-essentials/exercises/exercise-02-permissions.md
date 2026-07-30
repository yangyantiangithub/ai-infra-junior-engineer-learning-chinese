# 练习 02：ML 团队的文件权限和访问控制

## 概述

本练习教你如何在多用户 ML 基础设施环境中管理文件权限和访问控制。你将学习为共享数据集、模型目录和需要多名团队成员对资源进行受控访问的协作开发环境配置权限。

## 学习目标

完成本练习后，你将能够：
- 理解 Linux 权限模型（user、group、other）和权限位
- 使用 chmod（数字和符号）配置文件和目录权限
- 使用 chown 和 chgrp 管理文件所有权
- 设置访问控制列表（ACL）以实现细粒度权限
- 使用 umask 应用默认权限
- 实现 ML 基础设施的安全最佳实践
- 为 ML 团队协作创建安全的共享目录

## 先决条件

- 已完成练习 01
- 已完成讲座 03：权限和安全
- 可访问具有 sudo 权限的 Linux 系统（用于某些任务）
- 理解 Linux 用户和组概念
- 至少 2GB 可用磁盘空间

## 所需时间

- 预计：75-90 分钟
- 难度：中级

## 第 1 部分：理解权限模型

### 步骤 1：分析当前权限

```bash
# 为本次练习创建工作空间
mkdir -p ~/ml-permissions-lab
cd ~/ml-permissions-lab

# 创建各种文件供检查
touch dataset.csv model.h5 training_script.py config.yaml
mkdir shared_models team_data

# 检查权限
ls -l
```

### 步骤 2：权限计算（数字模式）

```bash
cat > permission_reference.txt << 'EOF'
PERMISSION BITS REFERENCE
=========================

Symbolic    Numeric    Binary    Meaning
---------   -------    ------    -------
---         0          000       No permissions
--x         1          001       Execute only
-w-         2          010       Write only
-wx         3          011       Write and execute
r--         4          100       Read only
r-x         5          101       Read and execute
rw-         6          110       Read and write
rwx         7          111       Read, write, and execute

COMPLETE PERMISSIONS (User Group Other)
---------------------------------------
644 = rw-r--r--  (Standard file)
755 = rwxr-xr-x  (Executable/Directory)
600 = rw-------  (Private file)
700 = rwx------  (Private directory)
775 = rwxrwxr-x  (Group shared)
664 = rw-rw-r--  (Group editable)
EOF
```

## 第 2 部分：使用 chmod 修改权限

### 步骤 3：数字模式权限

```bash
cd ~/ml-permissions-lab

# 创建测试文件
touch private_model.h5 shared_dataset.csv team_script.py

# 设置私有文件（仅所有者，读/写）
chmod 600 private_model.h5

# 设置共享数据集（所有者 rw，组 rw，其他人读）
chmod 664 shared_dataset.csv

# 设置可执行脚本（所有者 rwx，组 rx，其他人 rx）
chmod 755 team_script.py
```

### 步骤 4：符号模式权限

```bash
cd ~/ml-permissions-lab
mkdir symbolic_practice
cd symbolic_practice

# 为所有者添加执行权限
chmod u+x data_prep.py

# 为组添加写权限
chmod g+w results.csv

# 删除其他人的读权限
chmod o-r model_v1.h5

# 设置精确权限
chmod u=rwx,g=rx,o=r data_prep.py
```

### 步骤 5：真实 ML 场景

```bash
cd ~/ml-permissions-lab
mkdir ml_project_team
cd ml_project_team

# 场景 1：共享数据集目录
mkdir -p datasets/{raw,processed}
chmod 755 datasets/raw
chmod 775 datasets/processed

# 场景 2：模型仓库
mkdir -p models/{checkpoints,production}
chmod 775 models/checkpoints
chmod 755 models/production

# 场景 3：训练脚本
mkdir -p scripts
chmod 755 scripts/*.py 2>/dev/null || true

# 场景 4：敏感配置
mkdir -p configs/secrets
chmod 700 configs/secrets
```

## 第 3 部分：所有权管理

### 步骤 6：理解 chown 和 chgrp

```bash
cd ~/ml-permissions-lab

# 创建团队协作结构
mkdir team_collaboration
cd team_collaboration

mkdir -p {datasets,models,experiments,notebooks,scripts}
chmod 775 datasets models experiments notebooks scripts
```

## 第 4 部分：高级 ACL

### 步骤 7：使用 ACL

```bash
cd ~/ml-permissions-lab
mkdir acl_practice
cd acl_practice

# 创建共享模型目录
mkdir shared_model_registry
touch shared_model_registry/model_v1.h5

# 查看默认 ACL
getfacl shared_model_registry/model_v1.h5

# ACL 命令参考
cat > acl_reference.txt << 'EOF'
ACL COMMANDS REFERENCE
======================

View ACLs:
----------
getfacl file              View ACLs for file

Set ACLs:
---------
setfacl -m u:user:rwx file       Set user permissions
setfacl -m g:group:rw file       Set group permissions

Default ACLs (for directories):
--------------------------------
setfacl -d -m g:group:rw dir     New files inherit group permissions

Remove ACLs:
------------
setfacl -x u:user file           Remove user ACL
setfacl -b file                  Remove all ACLs
EOF
```

## 第 5 部分：umask 的默认权限

### 步骤 8：理解和设置 umask

```bash
cd ~/ml-permissions-lab
mkdir umask_practice
cd umask_practice

# 检查当前 umask
umask

# 测试不同 umask 值
umask 0022
touch test_022_file
mkdir test_022_dir
ls -l test_022_file  # 应为 644
ls -ld test_022_dir  # 应为 755

umask 0002
touch test_002_file
mkdir test_002_dir
ls -l test_002_file  # 应为 664
ls -ld test_002_dir  # 应为 775

umask 0077
touch test_077_file
mkdir test_077_dir
ls -l test_077_file  # 应为 600
ls -ld test_077_dir  # 应为 700
```

## 第 6 部分：安全最佳实践

### 步骤 9：实施安全模式

```bash
cd ~/ml-permissions-lab
mkdir security_patterns
cd security_patterns

# 模式：最小权限
mkdir -p {public,shared,restricted,private}
chmod 755 public
chmod 775 shared
chmod 750 restricted
chmod 700 private
```

## 最终验证

```bash
cd ~/ml-permissions-lab

cat > validate_exercise.sh << 'EOF'
#!/bin/bash

PASS=0
FAIL=0

for dir in ml_project_team security_patterns; do
    if [ -d "$dir" ]; then
        echo "✓ Directory: $dir"
        ((PASS++))
    fi
done

echo "Passed: $PASS"
echo "Failed: $FAIL"
EOF

chmod +x validate_exercise.sh
./validate_exercise.sh
```

---

**恭喜！** 你已经掌握了 ML 基础设施的 Linux 文件权限和访问控制。
