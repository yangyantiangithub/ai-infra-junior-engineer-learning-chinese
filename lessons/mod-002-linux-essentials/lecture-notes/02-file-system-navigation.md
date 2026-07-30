# 讲座 02：文件系统和导航

## 目录
1. [简介](#简介)
2. [理解 Linux 文件系统](#理解-linux-文件系统)
3. [基本导航命令](#基本导航命令)
4. [文件操作](#文件操作)
5. [查找文件和目录](#查找文件和目录)
6. [使用链接](#使用链接)
7. [归档和压缩](#归档和压缩)
8. [AI 基础设施的最佳实践](#ai-基础设施的最佳实践)
9. [总结和要点](#总结和要点)

## 简介

作为 AI 基础设施工程师，你将花费大量时间与 Linux 文件系统打交道。无论是部署机器学习模型、管理 GPU 服务器还是编排容器，理解文件系统结构和导航都是所有未来工作的基础。

本讲座介绍 Linux 文件系统层级以及 AI 基础设施场景中导航和管理文件与目录的基本命令。

### 学习目标

完成本讲座后，你将能够：
- 理解 Linux 文件系统层级和组织
- 使用命令行工具高效导航
- 自信地执行文件和目录操作
- 使用各种搜索工具快速查找文件
- 使用符号链接和硬链接
- 为数据和模型管理创建和提取归档
- 将 Linux 基础应用到 AI 基础设施场景

### 先决条件
- 讲座 01：Linux 和命令行简介
- 可访问 Linux 系统（Ubuntu 22.04 LTS、CentOS 8 或类似系统）
- 基本理解操作系统的功能
- 已安装并可访问终端模拟器

### 为什么 AI 基础设施需要文件系统知识？

**数据管理**：训练数据集、模型文件和结果必须在可能数百 GB 或 TB 的数据中正确组织。

**模型存储**：理解在何处以及如何存储模型、检查点和配置可确保可重现的部署。

**性能**：知道对不同工作负载使用哪些文件系统（训练数据使用 NVMe SSD、归档使用网络存储）会影响性能。

**协作**：正确的文件组织使团队能够有效地在 ML 项目上协同工作。

## 理解 Linux 文件系统

### 文件系统层级标准（FHS）

与使用盘符（C:\、D:\）的 Windows 不同，Linux 使用从根目录 `/` 开始的单一统一树结构。

```
/                          # 根 - 文件系统层级的顶层
├── bin/                   # 基本用户命令二进制文件
├── boot/                  # 引导加载程序文件（内核、initrd）
├── dev/                   # 设备文件（硬件接口）
├── etc/                   # 系统配置文件
├── home/                  # 用户主目录
│   ├── alice/
│   └── bob/
├── lib/                   # 共享库
├── media/                 # 可移动介质挂载点
├── mnt/                   # 临时挂载点
├── opt/                   # 可选/附加软件包
├── proc/                  # 进程和内核信息（虚拟）
├── root/                  # root 用户的主目录
├── run/                   # 运行时数据
├── sbin/                  # 系统管理二进制文件
├── srv/                   # 服务数据（web 服务器、FTP）
├── sys/                   # 系统/内核信息（虚拟）
├── tmp/                   # 临时文件（重启时清除）
├── usr/                   # 用户程序和数据
│   ├── bin/              # 用户命令二进制文件
│   ├── lib/              # /usr/bin 的库
│   ├── local/            # 本地安装的软件
│   └── share/            # 架构无关的数据
└── var/                   # 可变数据（日志、数据库）
    ├── log/              # 日志文件
    ├── cache/            # 应用程序缓存
    └── tmp/              # 临时文件（重启后持久）
```

### AI 基础设施的关键目录

**`/home/username/`**：你的个人工作空间。开发期间在此存储 ML 笔记本、训练脚本和数据集。

**示例**：
```bash
/home/alice/
├── ml-projects/
│   ├── image-classifier/
│   │   ├── data/
│   │   ├── models/
│   │   ├── notebooks/
│   │   └── train.py
│   └── nlp-sentiment/
└── datasets/
```

**`/opt/`**：在此安装自定义 ML 框架、CUDA 工具包或专有软件。

**示例**：
```bash
/opt/
├── cuda-11.8/
├── ml-framework/
└── custom-models/
```

**`/var/log/`**：对故障排查至关重要。训练日志、服务日志和错误消息位于此处。

**示例**：
```bash
/var/log/
├── training/
│   ├── model-2024-01-15.log
│   └── errors.log
├── inference/
└── system/
```

**`/etc/`**：Docker、Kubernetes 和 NVIDIA 驱动等服务的配置文件。

**示例**：
```bash
/etc/
├── docker/
│   └── daemon.json
├── kubernetes/
└── nvidia/
    └── cuda.conf
```

**`/tmp/`**：用于中间训练数据或模型检查点的临时存储。**警告：重启时内容可能被删除！**

**`/usr/local/`**：自定义构建的软件，如特定 Python 版本或从源码编译的 ML 库。

### 路径概念

**绝对路径**：从根目录 `/` 开始

```bash
/home/alice/ml-projects/image-classifier/train.py
```

**相对路径**：相对于当前位置

```bash
# 如果当前在 /home/alice/
ml-projects/image-classifier/train.py

# 如果当前在 /home/alice/ml-projects/
image-classifier/train.py
```

**特殊路径符号**：
- `.` - 当前目录
- `..` - 上级目录
- `~` - 当前用户的主目录
- `-` - 上一个目录

```bash
cd ~                    # 前往主目录
cd ..                   # 上一级
cd -                    # 返回上一个目录
cd ~/ml-projects        # 前往主目录下的 ml-projects
./script.sh             # 在当前目录运行脚本
```

## 基本导航命令

### pwd - 打印工作目录

始终知道你在文件系统中的位置。

```bash
pwd
# 输出：/home/alice/ml-projects

# 在脚本中验证位置
CURRENT_DIR=$(pwd)
echo "Running training from: $CURRENT_DIR"
```

**AI 基础设施用例**：运行训练作业时，始终验证工作目录以确保数据路径正确。

```bash
#!/bin/bash
# 在训练前验证我们在正确的位置
if [[ $(pwd) != "/opt/ml-training" ]]; then
    echo "Error: Must run from /opt/ml-training"
    exit 1
fi

python train.py
```

### ls - 列出目录内容

最常用的 Linux 命令。使用各种选项列出文件和目录。

**基本用法**：
```bash
ls                      # 列出当前目录中的文件
ls /var/log            # 列出 /var/log 中的文件
ls ~                   # 列出主目录中的文件
```

**常见选项**：
```bash
ls -l                   # 长格式（权限、所有者、大小、日期）
ls -a                   # 显示隐藏文件（以 . 开头）
ls -h                   # 人类可读大小（KB、MB、GB）
ls -t                   # 按修改时间排序（最新的在前）
ls -r                   # 反向顺序
ls -R                   # 递归（显示子目录）

# 组合选项
ls -lah                 # 长格式、所有文件、人类可读
ls -lt                  # 长格式、按时间排序
ls -lhS                 # 长格式、人类可读、按大小排序
```

**实际示例**：
```bash
# 查找当前目录中最大的模型文件
ls -lhS *.h5 | head -5

# 列出所有 Python 文件，显示详细信息
ls -lh *.py

# 查找最近修改的训练检查点
ls -lt checkpoints/ | head -10

# 检查总磁盘使用量
ls -lh --block-size=G
```

**输出解释**：
```bash
$ ls -lh model.h5
-rw-r--r-- 1 alice mlteam 500M Oct 15 14:30 model.h5
│   │  │  │ │     │      │    │           │
│   │  │  │ │     │      │    │           └─ 文件名
│   │  │  │ │     │      │    └─ 修改日期/时间
│   │  │  │ │     │      └─ 文件大小（人类可读）
│   │  │  │ │     └─ 组所有者
│   │  │  │ └─ 文件所有者
│   │  │  └─ 硬链接数量
│   │  └─ 组权限（读）
│   └─ 所有者权限（读、写）
│   └─ 文件类型（- = 普通文件，d = 目录，l = 链接）
```

### cd - 更改目录

高效地在文件系统中导航。

```bash
cd /var/log                      # 绝对路径
cd logs/training                 # 相对路径
cd                               # 前往主目录
cd ~                             # 也前往主目录
cd ..                            # 上一级
cd ../..                         # 上两级
cd -                             # 切换到上一个目录

# 高级导航
cd ~/ml-projects/model-v2        # 使用绝对路径直接跳转
cd ../../data/processed          # 相对导航
```

**专业技巧**：
```bash
# 快速目录切换
cd /var/log/nginx
cd -                    # 返回上一个目录
cd -                    # 回到 /var/log/nginx（切换）

# 导航到同级目录
cd ../another-project   # 从 /home/alice/project-a 到 project-b

# 使用 Tab 补全节省时间
cd /var/lo<TAB>         # 自动补全为 /var/log/
```

**AI 基础设施示例**：
```bash
# 导航到训练目录
cd ~/ml-projects/image-classifier

# 检查我们在哪里
pwd

# 前往数据目录
cd data/raw

# 回到项目根目录
cd ../..

# 检查训练日志
cd /var/log/training

# 返回项目
cd -
```

### tree - 可视化目录结构

以树形显示目录层级（可能需要安装：`sudo apt install tree`）。

```bash
tree                           # 从当前目录显示树
tree -L 2                      # 限制深度为 2 级
tree -d                        # 仅目录
tree -h                        # 人类可读大小
tree -a                        # 包含隐藏文件

# 实用 ML 示例
tree -L 3 ml-project/
# ml-project/
# ├── data/
# │   ├── raw/
# │   └── processed/
# ├── models/
# │   ├── checkpoints/
# │   └── final/
# ├── notebooks/
# └── src/
#     ├── train.py
#     └── evaluate.py
```

**排除模式**：
```bash
# 排除 Python 缓存和 git 目录
tree -I '__pycache__|.git'

# 排除大型数据目录
tree -I 'data|*.h5|*.pt'
```

## 文件操作

### 创建文件和目录

**touch - 创建空文件**：
```bash
touch train.py                    # 创建单个文件
touch model.py utils.py          # 创建多个文件
touch experiment_{1..5}.log      # 创建编号文件

# 更新现有文件的修改时间
touch -t 202310151430 model.h5   # 设置特定时间戳
```

**mkdir - 创建目录**：
```bash
mkdir models                     # 创建单个目录
mkdir data logs checkpoints      # 创建多个目录
mkdir -p data/raw/images         # 根据需要创建父目录
mkdir -p experiments/{exp1,exp2,exp3}/checkpoints

# 验证结构
tree experiments/
# experiments/
# ├── exp1/
# │   └── checkpoints/
# ├── exp2/
# │   └── checkpoints/
# └── exp3/
#     └── checkpoints/
```

**AI 基础设施示例**：
```bash
# 设置 ML 项目结构
mkdir -p ml-project/{data/{raw,processed},models/{checkpoints,final},notebooks,src,logs}

tree ml-project/
# ml-project/
# ├── data/
# │   ├── raw/
# │   └── processed/
# ├── logs/
# ├── models/
# │   ├── checkpoints/
# │   └── final/
# ├── notebooks/
# └── src/
```

### 复制文件和目录

**cp - 复制文件**：
```bash
cp source.py destination.py              # 复制文件
cp model.h5 models/backup/               # 复制到目录
cp -r experiments/ backup/               # 递归复制目录
cp -p config.yaml config.yaml.bak        # 保留时间戳
cp -i model.h5 models/                   # 交互式（覆盖前提示）
cp -u *.py backup/                       # 更新（仅复制较新文件）

# 高级模式
cp train_{v1,v2}.py archive/             # 复制特定文件
cp --backup=numbered model.h5 models/    # 创建编号备份
```

**真实 ML 场景**：
```bash
# 重新训练前备份模型
cp -p model_best.h5 model_best.h5.$(date +%Y%m%d)

# 将数据集复制到本地 SSD 以加快训练速度
cp -r /mnt/nas/datasets/imagenet /local/ssd/datasets/

# 复制实验配置
cp -r experiments/baseline experiments/experiment_2

# 带有时间戳的备份
cp model.h5 backups/model_$(date +%Y%m%d_%H%M%S).h5
```

### 移动和重命名

**mv - 移动/重命名文件**：
```bash
mv old_name.py new_name.py              # 重命名文件
mv model.h5 models/                     # 移动文件
mv experiments/old_exp experiments/new  # 重命名目录
mv -i file.py dest/                     # 交互式模式
mv *.log logs/                          # 移动多个文件
mv -n file.py dest/                     # 不覆盖

# 使用模式批量重命名
for file in model_v*.h5; do
    mv "$file" "${file/v/version_}"
done
# model_v1.h5 → model_version_1.h5
```

**实际用途**：
```bash
# 组织训练好的模型
mv model_epoch_*.h5 models/checkpoints/

# 完成后重命名实验
mv experiment_temp experiment_resnet50_baseline

# 将日志移动到归档
mv logs/training_$(date -d '30 days ago' +%Y%m).log logs/archive/

# 暂存模型用于部署
mv models/checkpoints/best_model.h5 models/production/
```

### 删除文件和目录

**rm - 删除文件**（⚠️ **谨慎使用 - 无法撤销！**）：
```bash
rm file.txt                         # 删除单个文件
rm file1.txt file2.txt              # 删除多个文件
rm -i *.log                         # 交互式（确认每次删除）
rm -f locked_file                   # 强制删除（忽略警告）
rm -r directory/                    # 递归删除目录
rm -rf temp_data/                   # 强制递归删除

# 安全实践
rm -i *.h5                          # 对于重要文件始终确认
rm -v deleted_file                  # 详细模式（显示删除内容）
```

**⚠️ 危险命令 - 永远不要运行这些**：
```bash
# 绝不要这样做 - 它们会删除一切！
rm -rf /                            # 删除整个系统（现代系统上有保护）
rm -rf /*                           # 同样的灾难
rm -rf $VARIABLE/                   # 如果 $VARIABLE 为空，从根开始删除！

# 在运行 rm -rf 之前始终验证
echo rm -rf $DIR/*                  # 检查将删除什么
```

**安全删除实践**：
```bash
# 创建回收站目录而不是立即删除
mkdir -p ~/.trash
alias trash='mv --target-directory=$HOME/.trash'

# 使用回收站代替 rm
trash old_model.h5                  # 移动到回收站，可以恢复

# 定期清空回收站
find ~/.trash -mtime +30 -delete    # 删除超过 30 天的文件

# 对于关键操作，使用交互式模式
rm -ri experiments/failed_*         # 确认每次删除
```

**AI 基础设施安全**：
```bash
# 删除训练数据前验证
ls -lh data/to_delete/
du -sh data/to_delete/
# 然后删除
rm -rf data/to_delete/

# 创建带有确认的清理脚本
cat > cleanup.sh << 'EOF'
#!/bin/bash
echo "This will delete:"
find checkpoints/ -mtime +7 -name "*.ckpt"
read -p "Proceed? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    find checkpoints/ -mtime +7 -name "*.ckpt" -delete
    echo "Cleanup complete"
fi
EOF
```

### 查看文件内容

**cat - 连接并显示**：
```bash
cat file.txt                        # 显示整个文件
cat file1.txt file2.txt             # 显示多个文件
cat file1.txt file2.txt > combined.txt  # 合并文件
cat -n file.py                      # 显示行号
```

**less - 分页浏览文件**：
```bash
less large_log.txt                  # 查看大文件，支持分页

# less 中的导航：
# Space/PgDn - 下一页
# b/PgUp - 上一页
# /pattern - 向前搜索
# ?pattern - 向后搜索
# n - 下一个搜索结果
# q - 退出
```

**head/tail - 查看文件开头/结尾**：
```bash
head file.txt                       # 前 10 行
head -n 20 file.txt                # 前 20 行
tail file.txt                       # 最后 10 行
tail -n 50 training.log            # 最后 50 行
tail -f training.log               # 实时跟踪日志（Ctrl+C 停止）

# 监控训练进度
tail -f logs/training.log | grep "epoch"
```

**实际 AI 示例**：
```bash
# 检查 CSV 数据集的前几行
head -20 data.csv

# 实时监控训练
tail -f training.log

# 查看最后 100 个错误
grep ERROR training.log | tail -100

# 检查以文本格式保存的模型架构
cat model_architecture.txt

# 比较配置
cat config_v1.yaml config_v2.yaml

# 提取特定行
sed -n '100,200p' large_dataset.csv  # 第 100-200 行
```

## 查找文件和目录

### find - 强大的文件搜索

`find` 命令是查找文件和目录最强大的工具之一。

**基本语法**：`find [path] [expression]`

**按名称搜索**：
```bash
find . -name "*.py"                    # 查找所有 Python 文件
find /var/log -name "*.log"            # 查找日志文件
find ~ -name "model.h5"                # 查找特定文件
find . -iname "*.PY"                   # 不区分大小写搜索
```

**按类型搜索**：
```bash
find . -type f                         # 仅文件
find . -type d                         # 仅目录
find . -type l                         # 仅符号链接
```

**按大小搜索**：
```bash
find . -size +1G                       # 大于 1GB 的文件
find . -size -100M                     # 小于 100MB 的文件
find . -size +500M -size -2G           # 500MB 到 2GB 之间的文件

# 查找大型模型文件
find models/ -type f -size +100M -exec ls -lh {} \;
```

**按时间搜索**：
```bash
find . -mtime -7                       # 最近 7 天内修改
find . -mtime +30                      # 超过 30 天前修改
find . -mmin -60                       # 最近 60 分钟内修改
find . -atime -1                       # 最近一天内访问
```

**执行操作**：
```bash
# 删除旧的检查点文件
find checkpoints/ -name "*.ckpt" -mtime +7 -delete

# 将最近的模型复制到备份
find models/ -name "*.h5" -mtime -1 -exec cp {} backup/ \;

# 列出大文件及其详情
find . -size +1G -exec ls -lh {} \;

# 统计 Python 文件数量
find . -name "*.py" -type f | wc -l

# 查找并压缩旧日志
find logs/ -name "*.log" -mtime +30 -exec gzip {} \;
```

**AI 基础设施示例**：
```bash
# 查找所有 Jupyter 笔记本
find . -name "*.ipynb"

# 查找大于 500MB 的模型文件
find . \( -name "*.h5" -o -name "*.pt" \) -size +500M

# 查找最近修改的训练脚本
find . -name "train*.py" -mtime -7

# 定位项目中的所有 requirements.txt 文件
find ~ -name "requirements.txt"

# 查找空目录（失败的实验？）
find experiments/ -type d -empty

# 查找全局可写文件（安全问题）
find . -type f -perm 0777

# 清理 Python 缓存
find . -type d -name "__pycache__" -exec rm -rf {} +

# 查找 90 天内未访问的数据集
find /data/datasets -type f -atime +90 -size +1G

# 查找损坏的符号链接
find . -type l ! -exec test -e {} \; -print
```

### locate - 快速数据库搜索

比 `find` 更快，但使用数据库（每天更新）。

```bash
# 先更新数据库（需要 sudo）
sudo updatedb

# 搜索文件
locate model.h5                        # 查找所有 model.h5 文件
locate -i python                       # 不区分大小写
locate -c "*.log"                      # 计数匹配项
locate -e file.txt                     # 仅存在的文件

# 限制结果
locate -l 10 "*.py"                    # 仅显示前 10 个结果

# 搜索特定路径
locate -b '\model.h5'                  # 仅基本名称（精确名称）
```

**何时使用**：
- 使用 `locate` 进行按名称的快速搜索
- 使用 `find` 进行复杂查询（大小、时间、权限）

### which - 查找命令位置

定位 PATH 中的可执行文件。

```bash
which python                           # /usr/bin/python
which python3                          # /usr/bin/python3
which docker                           # /usr/bin/docker
which nvidia-smi                       # /usr/bin/nvidia-smi

# 查找所有实例
which -a python                        # 显示 PATH 中所有 python
```

**AI 基础设施用途**：
```bash
# 验证正确的 Python 版本
which python
python --version

# 检查 CUDA 安装
which nvcc
nvcc --version

# 定位 Docker
which docker
docker --version

# 查找 pip 位置（虚拟环境检查）
which pip

# 验证虚拟环境已激活
which python
# 应显示：/path/to/venv/bin/python
```

### whereis - 定位二进制、源码、手册

```bash
whereis python                         # 二进制、源码、man 手册页
whereis -b python                      # 仅二进制
whereis -m python                      # 仅 man 手册页
```

## 使用链接

Linux 支持两种链接：**硬链接**和**符号（软）链接**。

### 符号链接（Symlinks）

符号链接就像一个快捷方式 - 它指向另一个文件或目录。

**创建符号链接**：
```bash
ln -s /path/to/original /path/to/link

# 示例
ln -s /data/datasets ~/datasets                    # 快速访问数据集
ln -s /opt/cuda-11.8 /opt/cuda                     # 版本管理
ln -s models/model_best.h5 current_model.h5       # 指向最新模型
ln -s /mnt/nas/training_data ~/training_data      # 网络存储快捷方式

# 验证
ls -l current_model.h5
# lrwxrwxrwx 1 alice mlteam 20 Oct 15 14:30 current_model.h5 -> models/model_best.h5
```

**符号链接特性**：
- 可以链接到文件或目录
- 可以跨文件系统（不同分区/驱动器）
- 如果目标被移动或删除，会变为损坏
- 有自己的权限（通常为 777，但目标权限生效）
- 文件大小很小（仅存储路径）

**AI 基础设施用例**：
```bash
# 链接到当前生产模型
ln -s models/model_v3.h5 production_model.h5

# 数据集的版本管理
ln -s /data/imagenet-2023 /data/imagenet-current

# Python 虚拟环境
ln -s /opt/venvs/ml-env ~/.venv

# 配置管理
ln -s /etc/ml-service/config.prod.yaml /etc/ml-service/config.yaml

# 快速访问日志目录
ln -s /var/log/ml-training ~/logs

# 共享模型仓库
ln -s /mnt/shared/models ~/shared-models
```

### 硬链接

硬链接为同一文件数据创建多个目录条目。

```bash
ln original.txt hardlink.txt          # 创建硬链接（没有 -s 标志）

# 验证 - 相同的 inode 编号
ls -li original.txt hardlink.txt
# 12345678 -rw-r--r-- 2 alice mlteam 1024 Oct 15 14:30 original.txt
# 12345678 -rw-r--r-- 2 alice mlteam 1024 Oct 15 14:30 hardlink.txt
#    ↑ 相同 inode = 相同的文件数据
```

**硬链接特性**：
- 共享相同的 inode（相同的文件数据）
- 不能链接到目录（大多数情况下）
- 不能跨文件系统
- 文件持续存在，直到所有硬链接被删除
- 与原始文件无法区分

**何时使用**：
- 大多数情况下使用**符号链接**（目录、跨文件系统、清晰性）
- 在希望文件持久存在的备份场景中使用**硬链接**

### 管理链接

```bash
# 检查文件是否为符号链接
ls -l file.txt                         # 如果是符号链接则显示 ->
file file.txt                          # 显示符号链接信息
readlink file.txt                      # 显示符号链接的目标
readlink -f file.txt                   # 跟踪所有符号链接到最终目标

# 查找损坏的符号链接
find . -type l ! -exec test -e {} \; -print

# 删除符号链接（不影响目标）
rm symlink_name                        # 安全 - 仅删除链接
unlink symlink_name                    # 替代命令

# 更新符号链接
ln -sf new_target existing_link        # 强制更新

# 列出所有符号链接
find . -type l -ls

# 列出当前目录中的符号链接
ls -l | grep ^l
```

## 归档和压缩

对于管理数据集、模型文件和备份至关重要。

### tar - 磁带归档

将多个文件合并为一个归档文件。

**创建归档**：
```bash
tar -cvf archive.tar files/            # 创建 tar 归档
tar -czvf archive.tar.gz files/        # 创建压缩 tar（gzip）
tar -cjvf archive.tar.bz2 files/       # 创建压缩 tar（bzip2）
tar -cJvf archive.tar.xz files/        # 创建压缩 tar（xz - 最佳压缩）

# 选项：
# c - 创建
# x - 提取
# v - 详细模式
# f - 文件
# z - gzip 压缩
# j - bzip2 压缩
# J - xz 压缩
```

**提取归档**：
```bash
tar -xvf archive.tar                   # 提取 tar
tar -xzvf archive.tar.gz               # 提取 gzip 压缩的 tar
tar -xjvf archive.tar.bz2              # 提取 bzip2 压缩的 tar
tar -xvf archive.tar -C /target/dir    # 提取到特定目录

# 提取单个文件
tar -xzvf archive.tar.gz path/to/file.txt
```

**列出内容**：
```bash
tar -tvf archive.tar                   # 列出内容而不提取
tar -tzvf archive.tar.gz               # 列出压缩归档
tar -tzvf archive.tar.gz | grep model  # 查找特定文件
```

**AI 基础设施示例**：
```bash
# 备份模型检查点
tar -czvf model_checkpoints_$(date +%Y%m%d).tar.gz checkpoints/

# 归档完成的实验
tar -czvf experiment_resnet50_baseline.tar.gz experiments/resnet50/

# 压缩大型数据集
tar -cJvf imagenet_subset.tar.xz data/imagenet_subset/

# 提取预训练权重
tar -xzvf pretrained_weights.tar.gz -C models/pretrained/

# 备份配置和代码（排除大文件）
tar -czvf project_backup_$(date +%Y%m%d).tar.gz \
    --exclude='*.h5' \
    --exclude='*.pt' \
    --exclude='__pycache__' \
    --exclude='.git' \
    ml-project/

# 验证归档完整性
tar -tzvf archive.tar.gz > /dev/null && echo "Archive OK" || echo "Archive corrupted"

# 为大型数据集创建拆分归档
tar -czvf - large_dataset/ | split -b 1G - dataset.tar.gz.
```

### gzip/gunzip - 压缩

压缩单个文件。

```bash
gzip file.txt                          # 压缩为 file.txt.gz（替换原文件）
gzip -k file.txt                       # 保留原文件
gzip -9 file.txt                       # 最大压缩（1-9 级）
gunzip file.txt.gz                     # 解压缩
gzip -d file.txt.gz                    # 也用于解压缩

# 查看压缩文件而不提取
zcat file.txt.gz                       # 查看内容
zless file.txt.gz                      # 分页浏览内容
zgrep "pattern" file.txt.gz            # 搜索压缩文件

# 压缩多个文件
gzip -r directory/                     # 递归压缩

# 检查压缩比
gzip -l file.txt.gz
```

### zip/unzip - 跨平台归档

```bash
zip archive.zip file1.txt file2.txt    # 创建 zip 归档
zip -r archive.zip directory/          # 递归
zip -9 archive.zip files/              # 最大压缩
unzip archive.zip                      # 提取
unzip -l archive.zip                   # 列出内容
unzip archive.zip -d /target/dir       # 提取到目录

# AI 基础设施用途
zip -r model_package.zip models/ config.yaml requirements.txt
unzip -q pretrained_model.zip          # 安静模式提取

# 密码保护
zip -e -r secure.zip sensitive_data/
unzip secure.zip  # 将提示输入密码
```

### 压缩对比

```bash
# 原始文件：1GB 模型
# gzip (.gz)：  ~300MB，快速压缩/解压缩
# bzip2 (.bz2)： ~250MB，较慢但压缩更好
# xz (.xz)：    ~200MB，最慢但最佳压缩
# zip：         ~300MB，跨平台兼容

# 根据优先级选择：
# 速度：gzip（.tar.gz）
# 压缩比：xz（.tar.xz）
# 兼容性：zip
# 平衡：bzip2（.tar.bz2）
```

**实际工作流**：
```bash
# 每日备份（速度优先）
tar -czvf backup_$(date +%Y%m%d).tar.gz important_data/

# 长期归档（压缩优先）
tar -cJvf archive_$(date +%Y%m).tar.xz completed_projects/

# 与 Windows 用户共享
zip -r project_share.zip project/

# 大型数据集分发（拆分成块）
tar -czvf - dataset/ | split -b 1G - dataset.tar.gz.
# 创建 dataset.tar.gz.aa、dataset.tar.gz.ab 等
# 重建：cat dataset.tar.gz.* | tar -xzvf -
```

## AI 基础设施的最佳实践

### 1. 一致地组织你的项目

```bash
# 标准 ML 项目结构
ml-project/
├── data/
│   ├── raw/              # 原始不可变数据
│   ├── processed/        # 已清洗/转换的数据
│   └── external/         # 外部数据集
├── models/
│   ├── checkpoints/      # 训练检查点
│   ├── final/            # 生产模型
│   └── experiments/      # 实验模型
├── notebooks/            # Jupyter 笔记本
├── src/                  # 源代码
│   ├── data/            # 数据处理
│   ├── models/          # 模型定义
│   ├── training/        # 训练脚本
│   └── evaluation/      # 评估脚本
├── tests/               # 单元测试
├── logs/                # 训练日志
├── configs/             # 配置文件
├── scripts/             # 工具脚本
├── docs/                # 文档
├── requirements.txt     # 依赖
├── README.md            # 项目概述
└── .gitignore          # Git 忽略规则
```

### 2. 使用描述性命名

```bash
# 好名称
model_resnet50_imagenet_20231015.h5
experiment_baseline_lr0.001_batch32/
training_log_2023_10_15_14_30.log

# 差名称
model1.h5
exp_new/
log.txt
```

### 3. 对数据位置进行版本控制

```bash
# 使用符号链接处理数据集版本
ln -s /data/imagenet/2023 ~/projects/classifier/data/imagenet

# 新数据到达时易于更新
rm ~/projects/classifier/data/imagenet
ln -s /data/imagenet/2024 ~/projects/classifier/data/imagenet
```

### 4. 自动化清理

```bash
# 删除旧检查点（保留最后 10 个）
ls -t checkpoints/*.ckpt | tail -n +11 | xargs rm

# 每月归档旧日志
find logs/ -name "*.log" -mtime +30 -exec gzip {} \;

# 清理 Python 缓存
find . -type d -name "__pycache__" -exec rm -rf {} +
find . -type f -name "*.pyc" -delete

# 自动化清理脚本
cat > cleanup.sh << 'EOF'
#!/bin/bash
echo "Cleaning old checkpoints..."
find checkpoints/ -name "*.ckpt" -mtime +7 -delete
echo "Compressing old logs..."
find logs/ -name "*.log" -mtime +30 ! -name "*.gz" -exec gzip {} \;
echo "Removing Python cache..."
find . -type d -name "__pycache__" -exec rm -rf {} +
echo "Cleanup complete"
EOF
chmod +x cleanup.sh
```

### 5. 在脚本中使用绝对路径

```bash
# 不好 - 依赖于当前目录
python train.py --data data/train

# 好 - 明确且无歧义
python /home/alice/ml-project/src/train.py \
    --data /home/alice/ml-project/data/train

# 更好 - 使用变量
PROJECT_ROOT="/home/alice/ml-project"
python $PROJECT_ROOT/src/train.py --data $PROJECT_ROOT/data/train

# 最好 - 检测脚本位置
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
python $PROJECT_ROOT/src/train.py --data $PROJECT_ROOT/data/train
```

### 6. 记录文件操作

```bash
# 在数据目录中创建 README 文件
cat > data/imagenet/README.txt << EOF
ImageNet 2023 dataset
Downloaded: 2023-10-15
Source: http://www.image-net.org/
Preprocessing: Resized to 224x224, normalized to [-1,1]
Classes: 1000
Total images: 1,281,167 training, 50,000 validation
EOF

# 记录模型元数据
cat > models/model_v1_metadata.txt << EOF
Model: ResNet50
Training date: $(date)
Accuracy: 0.92
Dataset: ImageNet 2023
Hyperparameters:
  - Learning rate: 0.001
  - Batch size: 32
  - Epochs: 100
EOF
```

### 7. 破坏性操作前进行检查

```bash
# 删除前始终验证
ls checkpoints/*.ckpt | wc -l        # 统计文件数
ls checkpoints/*.ckpt | head -5      # 预览文件
# 然后删除
rm checkpoints/*.ckpt

# 使用 echo 预览
echo rm -rf experiments/failed_*     # 显示将删除什么

# 创建确认包装器
safe_rm() {
    echo "About to delete:"
    ls -lh "$@"
    read -p "Continue? (y/n) " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        rm "$@"
    fi
}
```

### 8. 主动处理磁盘空间

```bash
# 在大型操作前检查可用空间
df -h /data                          # 检查磁盘空间
du -sh data/                         # 检查目录大小
du -h --max-depth=1 /data | sort -h  # 查找大目录

# 训练时监控
watch -n 60 'df -h /data && du -sh checkpoints/'

# 设置告警
DISK_USAGE=$(df -h /data | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 90 ]; then
    echo "WARNING: Disk space above 90%!"
    # 发送告警（邮件、Slack 等）
fi

# 查找最大的文件
find /data -type f -size +1G -exec ls -lh {} \; | sort -k5 -h

# 磁盘使用报告
du -h --max-depth=2 /data | sort -hr | head -20
```

## 总结和要点

### 掌握的基本命令

**导航**：
- `pwd` - 知道你在哪里
- `cd` - 在目录间移动
- `ls` - 列出内容
- `tree` - 可视化结构

**文件操作**：
- `touch` - 创建文件
- `mkdir -p` - 创建目录层级
- `cp -r` - 复制文件/目录
- `mv` - 移动/重命名
- `rm -rf` - 删除（小心！）

**查看**：
- `cat` - 显示文件
- `less` - 分页浏览文件
- `head/tail` - 查看文件开头/结尾
- `tail -f` - 跟踪日志

**查找**：
- `find` - 强大的搜索
- `locate` - 快速数据库搜索
- `which` - 查找命令
- `whereis` - 定位二进制文件

**链接**：
- `ln -s` - 创建符号链接
- `ln` - 创建硬链接

**归档**：
- `tar -czf` - 创建压缩归档
- `tar -xzf` - 提取归档
- `gzip/gunzip` - 压缩
- `zip/unzip` - 跨平台

### 关键概念

1. **文件系统层级**：从 `/` 开始的单一树，逻辑组织
2. **路径**：绝对与相对，特殊符号（`.`、`..`、`~`、`-`）
3. **一切都是文件**：设备、进程、套接字 - 都以文件形式呈现
4. **无法撤销**：大多数操作是永久性的 - 执行前验证
5. **区分大小写**：`File.txt` ≠ `file.txt` ≠ `FILE.TXT`

### AI 基础设施应用

- 使用一致的结构组织 ML 项目
- 在代码、数据和模型之间快速导航
- 高效地查找和管理大型模型文件
- 归档实验和数据集
- 通过日志文件监控训练
- 自动化清理和维护任务

### 最佳实践

✅ 使用描述性的文件和目录名称
✅ 使用一致的结构组织项目
✅ 破坏性操作前进行验证
✅ 在脚本中使用绝对路径
✅ 记录你的文件组织
✅ 自动化重复性任务
✅ 主动监控磁盘空间
✅ 保留重要数据的备份

❌ 不要在未经验证的情况下使用 `rm -rf`
❌ 不要将重要数据存储在 `/tmp` 中
❌ 不要在文件名中使用空格（使用下划线/连字符）
❌ 除非必要，不要以 root 身份工作
❌ 不要忽略磁盘空间警告

### 练习

1. 创建完整的 ML 项目结构
2. 使用 `find` 定位所有大于 1MB 的 Python 文件
3. 归档并压缩一个目录，然后在别处提取它
4. 创建符号链接来组织多个数据集版本
5. 编写脚本清理旧的检查点文件
6. 使用 `tail -f` 实时监控训练日志
7. 查找并删除所有 Python 缓存目录

### 下一步

在**讲座 03：权限和安全**中，你将学习：
- 理解 Linux 权限模型
- 管理用户和组
- 保护 ML 基础设施
- 设置多用户环境
- 访问控制最佳实践

### 快速参考卡

```bash
# 导航
pwd                    # 打印工作目录
cd /path               # 更改目录
ls -lah               # 列出所有文件及详情
tree -L 2             # 显示目录树

# 文件操作
touch file.txt        # 创建文件
mkdir -p dir/subdir   # 创建目录
cp -r src/ dst/       # 递归复制
mv old new            # 移动/重命名
rm -i file            # 删除（交互式）

# 查看
cat file.txt          # 显示文件
less file.txt         # 分页浏览文件
head -20 file.txt     # 前 20 行
tail -f log.txt       # 跟踪日志文件

# 查找
find . -name "*.py"   # 按名称查找
find . -size +1G      # 按大小查找
locate filename       # 快速数据库搜索
which command         # 查找命令位置

# 归档
tar -czvf arch.tar.gz dir/  # 创建归档
tar -xzvf arch.tar.gz       # 提取归档
gzip file.txt               # 压缩文件
unzip archive.zip           # 提取 zip
```

---

**讲座 02 结束**

继续学习**讲座 03：权限和安全**，了解如何保护你的 Linux 系统和为 AI 基础设施管理多用户环境。
