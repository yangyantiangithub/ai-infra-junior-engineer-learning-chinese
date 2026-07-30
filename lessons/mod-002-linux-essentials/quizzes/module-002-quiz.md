# 模块 002：Linux 基础 - 综合测验

## 说明

本测验涵盖模块 002：Linux 基础的所有主题。包含 30 道题，涵盖：
- Linux 基础和导航
- 文件系统和权限
- 进程管理
- Shell 脚本
- 包管理
- 日志分析
- 故障排查

**评分：**
- 每题 1 分
- 总分：30 分
- 及格线：24/30（80%）

**预计时间：** 45-60 分钟

请回答所有问题，然后在末尾核对答案。

---

## 第一部分：Linux 基础和导航（第 1-6 题）

### 第 1 题：文件系统层级
Linux 中哪个目录通常包含用户主目录？

A) /usr
B) /home
C) /var
D) /etc

**答案：** B

**解释：** /home 包含用户主目录（例如 /home/alice、/home/bob）。/usr 包含用户程序和文件，/var 包含日志等可变数据，/etc 包含系统配置文件。

---

### 第 2 题：绝对路径与相对路径
以下哪个是绝对路径？

A) ../data/train.csv
B) ./models/model.h5
C) /var/ml/datasets/train
D) models/checkpoint

**答案：** C

**解释：** 绝对路径从根目录（/）开始。相对路径从当前目录开始，可能使用 .（当前）或 ..（上级）。选项 C 以 / 开头，是绝对路径。

---

### 第 3 题：文件类型
在 `ls -l` 的输出中，第一个字符 'd' 表示什么？

A) 已删除文件
B) 目录
C) 磁盘文件
D) 设备文件

**答案：** B

**解释：** 在 `ls -l` 输出中，第一个字符表示文件类型：
- = 普通文件，d = 目录，l = 符号链接，b = 块设备，c = 字符设备，p = 管道，s = 套接字。

---

### 第 4 题：隐藏文件
哪个命令会显示隐藏文件（以 . 开头的文件）？

A) ls
B) ls -h
C) ls -a
D) ls --hidden

**答案：** C

**解释：** `ls -a` 显示所有文件，包括隐藏文件（以 . 开头）。`ls -h` 显示人类可读的大小，而不是隐藏文件。

---

### 第 5 题：创建嵌套目录
哪个命令可以在单个命令中创建嵌套目录？

A) mkdir -r /path/to/nested/dir
B) mkdir -p /path/to/nested/dir
C) mkdir --recursive /path/to/nested/dir
D) mkdir /path /path/to /path/to/nested /path/to/nested/dir

**答案：** B

**解释：** `mkdir -p`（或 --parents）根据需要创建父目录。选项 D 可以工作但效率低下。mkdir 没有 -r 标志。

---

### 第 6 题：查找文件
你需要在项目目录及其子目录中查找所有 Python 文件（.py）。哪个命令最好？

A) ls *.py
B) locate *.py
C) find . -name "*.py"
D) grep -r "*.py"

**答案：** C

**解释：** `find . -name "*.py"` 从当前目录（.）开始递归搜索匹配模式的文件。`ls *.py` 只检查当前目录，`locate` 搜索数据库（可能已过时），`grep` 搜索文件内容而不是文件名。

---

## 第二部分：文件权限（第 7-12 题）

### 第 7 题：权限数字表示法
数字权限 755 代表什么？

A) rwxr-xr-x
B) rwxrwxrwx
C) rw-r--r--
D) r-xr-xr-x

**答案：** A

**解释：** 755 = rwxr-xr-x。第一位数字（7=rwx）代表所有者，第二位（5=r-x）代表组，第三位（5=r-x）代表其他人。7=4+2+1（读+写+执行），5=4+1（读+执行）。

---

### 第 8 题：chmod 命令
哪个命令使脚本对所有用户可执行？

A) chmod 644 script.sh
B) chmod +x script.sh
C) chmod 755 script.sh
D) B 和 C 都对

**答案：** D

**解释：** 两种方式 `chmod +x script.sh`（符号）和 `chmod 755 script.sh`（数字）都使文件可执行。`chmod +x` 为所有用户添加执行权限，755 = rwxr-xr-x（所有者可以写，其他所有人都可以读和执行）。

---

### 第 9 题：文件所有权
你需要将文件的所有者更改为 'mluser'，组更改为 'mlteam'。哪个命令正确？

A) chmod mluser:mlteam file.txt
B) chown mluser:mlteam file.txt
C) chgrp mluser:mlteam file.txt
D) chown mlteam:mluser file.txt

**答案：** B

**解释：** `chown user:group file` 同时更改所有者和组。语法是 user:group，所以 mluser:mlteam 是正确的。`chmod` 更改权限，而不是所有权。`chgrp` 只更改组。

---

### 第 10 题：目录权限
为什么目录需要执行（x）权限？

A) 为了运行其中的脚本
B) 为了访问其内容
C) 为了删除目录
D) 为了在其中创建文件

**答案：** B

**解释：** 目录上的执行权限允许你访问其内容（cd 进入目录、列出文件、访问文件）。没有 x 权限，即使你有读权限，也无法进入目录。

---

### 第 11 题：umask
如果 umask 设置为 0022，新创建文件的默认权限是什么？

A) 777
B) 755
C) 644
D) 600

**答案：** C

**解释：** 文件默认 666，目录默认 777。umask 0022 进行减法：666 - 022 = 644（rw-r--r--）用于文件。对于目录：777 - 022 = 755（rwxr-xr-x）。

---

### 第 12 题：安全文件创建
你需要创建一个包含 API 密钥的文件，只有你可以读取。哪个命令确保从创建时就有正确的权限？

A) touch api_keys.txt && chmod 600 api_keys.txt
B) umask 0077 && touch api_keys.txt
C) touch api_keys.txt && chown $(whoami) api_keys.txt
D) A 和 B 都对

**答案：** D

**解释：** 两种方法都有效。选项 A 创建文件然后设置权限为 600（rw-------）。选项 B 先设置严格的 umask，因此文件以 600 权限创建。选项 C 只更改所有权，不更改权限。

---

## 第三部分：进程管理（第 13-18 题）

### 第 13 题：进程状态
在 `ps aux` 的输出中，STAT 值 'D' 表示什么？

A) 死亡/僵尸进程
B) 不可中断睡眠（通常是 I/O）
C) 在 CPU 上运行
D) 停止/挂起

**答案：** B

**解释：** D = 不可中断睡眠，通常等待 I/O。R = 运行中，S = 睡眠（可中断），T = 停止，Z = 僵尸。处于 D 状态的进程在 I/O 完成前不能被终止。

---

### 第 14 题：终止进程
你的训练进程（PID 5432）对 Ctrl+C 没有响应。你应该尝试什么顺序？

A) 立即 kill -9 5432
B) kill -15 5432，等待，然后根据需要 kill -9 5432
C) killall python
D) 重启

**答案：** B

**解释：** 总是先尝试优雅终止（kill -15 或 SIGTERM），允许进程进行清理。如果等待后它没有响应（例如 10 秒），然后使用 kill -9（SIGKILL）强制终止。SIGKILL 应该是最后的手段。

---

### 第 15 题：后台进程
哪个命令在后台运行训练脚本并保护它不受挂断信号影响？

A) python train.py &
B) nohup python train.py &
C) python train.py > /dev/null
D) bg python train.py

**答案：** B

**解释：** `nohup command &` 在后台运行（&）并且不受挂断信号影响（nohup），即使用户登出也会继续运行。仅使用 & 会在后台运行但会在登出时终止。`bg` 恢复已挂起的作业，不启动新作业。

---

### 第 16 题：进程监控
哪个命令显示按 CPU 排序的实时 CPU 和内存使用情况？

A) ps aux
B) top
C) htop
D) B 和 C 都对

**答案：** D

**解释：** `top` 和 `htop` 都显示实时资源使用情况。两者默认按 CPU 排序。`htop` 具有更友好的界面和颜色编码。`ps aux` 是快照，不是实时的。

---

### 第 17 题：GPU 进程管理
如何找出哪个进程正在使用 GPU 0？

A) ps aux | grep gpu
B) nvidia-smi
C) lspci | grep nvidia
D) top -gpu

**答案：** B

**解释：** `nvidia-smi` 显示 GPU 利用率和每个 GPU 上的进程。`lspci | grep nvidia` 显示 GPU 硬件但不显示使用情况。top 没有 `-gpu` 标志。`ps aux | grep gpu` 不会显示 GPU 使用情况。

---

### 第 18 题：持久会话
你需要启动一个训练作业，该作业在你断开 SSH 连接后继续运行。哪种工具最好？

A) screen
B) tmux
C) nohup
D) 以上都是

**答案：** D

**解释：** 三种方式都可以。`screen` 和 `tmux` 提供可以重新连接的持久终端会话。`nohup` 使进程忽略挂断信号。tmux 比 screen 更现代。选择取决于是否需要监控（screen/tmux）还是只需运行（nohup）。

---

## 第四部分：Shell 脚本（第 19-22 题）

### 第 19 题：Bash Shebang
脚本开头 `#!/bin/bash` 的目的是什么？

A) 这是一条注释
B) 指定使用哪个解释器
C) 使脚本可执行
D) 设置工作目录

**答案：** B

**解释：** shebang（#!）告诉系统使用哪个解释器来执行脚本。#!/bin/bash 使用 bash，#!/usr/bin/env python3 使用 Python。它由内核处理，不是注释。你仍然需要 chmod +x 使其可执行。

---

### 第 20 题：出错时退出
哪个命令应该放在脚本顶部附近，以便在任何命令失败时立即退出？

A) set -e
B) exit 1
C) trap ERR
D) set -x

**答案：** A

**解释：** `set -e`（或 `set -o errexit`）导致脚本在任何命令返回非零值时立即退出。`set -u` 在未定义变量时退出，`set -o pipefail` 在管道错误时退出。`set -x` 启用调试模式（打印命令）。`exit 1` 以错误码 1 退出。

---

### 第 21 题：命令替换
哪个语法正确地将命令输出捕获到变量中？

A) VAR=`ls`
B) VAR=$(ls)
C) VAR=${ls}
D) A 和 B 都对

**答案：** D

**解释：** 反引号 `command` 和 $(...) 都执行命令替换。$(command) 是首选的（POSIX 标准，更容易嵌套）。${var} 是变量扩展，不是命令替换。

---

### 第 22 题：检查命令是否成功
在 bash 脚本中如何检查上一条命令是否成功？

A) if [ $? -eq 0 ]; then
B) if [ $SUCCESS ]; then
C) if [ $RESULT == "ok" ]; then
D) if [ last_command ]; then

**答案：** A

**解释：** `$?` 包含上一条命令的退出码。0 = 成功，非零 = 失败。因此 `if [ $? -eq 0 ]` 检查成功。也可以直接使用 `if command; then` 而无需检查 $?。

---

## 第五部分：包管理（第 23-26 题）

### 第 23 题：系统包
哪个命令更新 Ubuntu/Debian 上的包索引？

A) sudo apt upgrade
B) sudo apt update
C) sudo apt install updates
D) sudo apt refresh

**答案：** B

**解释：** `sudo apt update` 更新包索引（可用包列表）。`sudo apt upgrade` 安装已安装包的更新。始终在 `apt upgrade` 之前运行 `apt update`。

---

### 第 24 题：Python 虚拟环境
为什么应该在 ML 项目中使用 Python 虚拟环境？

A) 它们提供更好的性能
B) 它们隔离项目依赖
C) 它们需要更少的磁盘空间
D) 它们启用 GPU 支持

**答案：** B

**解释：** 虚拟环境隔离依赖，防止项目之间的冲突。不同项目可以使用不同版本的包。它们不会提高性能或启用 GPU 支持。它们会使用更多磁盘空间（重复包）。

---

### 第 25 题：pip 与 conda
关于 pip 和 conda，哪个说法是正确的？

A) pip 只能安装 Python 包
B) conda 只能安装 Python 包
C) pip 比 conda 快
D) conda 不能使用虚拟环境

**答案：** A

**解释：** pip 从 PyPI 安装 Python 包。Conda 从 conda 渠道安装 Python 包和系统库（CUDA 等）。Conda 还管理环境。速度因情况而异。两者都支持隔离。

---

### 第 26 题：CUDA 安装
安装 CUDA 后，应用程序必须设置哪个环境变量才能找到 CUDA 库？

A) CUDA_PATH
B) LD_LIBRARY_PATH
C) LIBRARY_PATH
D) CUDA_HOME

**答案：** B

**解释：** `LD_LIBRARY_PATH` 告诉动态链接器在哪里找到共享库。必须包含 CUDA lib 目录（例如 /usr/local/cuda/lib64）。`PATH` 应包括 bin 目录用于可执行文件。CUDA_HOME 和 CUDA_PATH 是可选的辅助变量。

---

## 第六部分：日志分析（第 27-29 题）

### 第 27 题：查看日志文件
哪个命令实时跟踪日志文件，显示新写入的条目？

A) cat -f /var/log/syslog
B) tail /var/log/syslog
C) tail -f /var/log/syslog
D) head -f /var/log/syslog

**答案：** C

**解释：** `tail -f` 跟踪文件，持续显示新行。类似于 `tail --follow`。`cat` 和 `head` 没有 -f 选项。普通 `tail` 显示最后 10 行但不跟踪。

---

### 第 28 题：过滤日志
你需要从训练日志中找出过去一小时内所有的 ERROR 消息。哪个命令效果最好？

A) cat training.log | grep ERROR
B) grep ERROR training.log
C) journalctl --since "1 hour ago" | grep ERROR
D) B 和 C 都对

**答案：** D

**解释：** 对于应用日志，`grep ERROR training.log` 是直接的。对于由 systemd 管理的系统日志，`journalctl --since "1 hour ago" | grep ERROR` 有效。选项 A 可以工作但效率低下（无用的 cat 使用）。选择取决于日志位置。

---

### 第 29 题：提取指标
哪个工具最适合从日志中提取结构化数据（例如从训练日志中提取 loss 值）？

A) grep
B) awk
C) sed
D) cat

**答案：** B

**解释：** `awk` 擅长处理结构化文本和提取字段。可以轻松提取匹配模式的值、进行计算、格式化输出。`grep` 找到模式但不擅长提取值。`sed` 用于文本转换。`cat` 只显示内容。

---

## 第七部分：故障排查（第 30 题）

### 第 30 题：系统化排查
你的 ML 训练作业失败，显示"设备上没有剩余空间"。你的第一步应该是什么？

A) 立即删除所有检查点文件
B) 使用 df -h 检查磁盘使用情况
C) 重启系统
D) 增加 swap 空间

**答案：** B

**解释：** 第一步始终是诊断：使用 `df -h` 验证问题并了解情况。只有在确认磁盘已满后，才应决定要清理什么。盲目删除文件可能会丢失重要数据。重启不会释放磁盘空间。Swap 帮助内存，而不是磁盘。

---

## 答案汇总

1. B
2. C
3. B
4. C
5. B
6. C
7. A
8. D
9. B
10. B
11. C
12. D
13. B
14. B
15. B
16. D
17. B
18. D
19. B
20. A
21. D
22. A
23. B
24. B
25. A
26. B
27. C
28. D
29. B
30. B

---

## 评分指南

**27-30 正确（90-100%）：** 优秀！你已经掌握了 ML 基础设施的 Linux 基础。

**24-26 正确（80-89%）：** 良好！你及格了，但请复习你出错的部分。

**20-23 正确（67-79%）：** 一般。复习模块材料后重新参加测验。

**低于 20（< 67%）：** 需要改进。仔细复习所有讲座和练习后再重新参加。

---

## 主题复习建议

基于常见错误，请复习这些部分：

**如果你错过了第 1-6 题：** 复习讲座 01（Linux 基础）和练习 01（导航）

**如果你错过了第 7-12 题：** 复习讲座 02（文件权限）和练习 02（权限）

**如果你错过了第 13-18 题：** 复习讲座 03（进程管理）和练习 03（进程）

**如果你错过了第 19-22 题：** 复习讲座 04（Shell 脚本）和练习 04（脚本）

**如果你错过了第 23-26 题：** 复习讲座 05（包管理）和练习 05（包管理）

**如果你错过了第 27-29 题：** 复习练习 06（日志分析）

**如果你错过了第 30 题：** 复习练习 07（故障排查）

---

## 额外练习

为了巩固你的学习：

1. **动手实践：** 如果还没有完成，请完成所有练习
2. **实际应用：** 搭建测试 ML 环境并练习这些命令
3. **创建速查表：** 总结关键命令以便快速参考
4. **教他人：** 解释概念有助于巩固理解
5. **构建项目：** 将技能应用到实际的 ML 基础设施项目中

---

## 下一步

一旦你在本测验中达到至少 80%：

1. 进入**模块 003：使用 Docker 进行容器化**
2. 保留本模块的材料作为参考
3. 在你的项目中定期练习这些技能
4. 使用所学知识构建自动化脚本

---

**恭喜完成模块 002 测验！**

如果你的得分达到 80% 或更高，你可以继续前进。如果没有，请复习建议的材料并重新参加测验。请记住，掌握 Linux 基础对于成为成功的 AI 基础设施工程师至关重要。
