# 先决条件测验

## 概述

**目的**：评估初级 AI 基础设施工程师课程所需的基础知识

**时长**：30 分钟

**总题数**：20 道

**及格分数**：70%（答对 14/20 题）

**涵盖主题**：
- Python 基础（5 题）
- Linux/命令行（5 题）
- Git 版本控制（4 题）
- 基础网络（3 题）
- 数据库基础（3 题）

**说明**：
1. 尽你所能回答所有问题
2. 有些题目有多个正确答案（标为"选择所有适用项"）
3. 测验期间不要使用外部资源
4. 记录你的答案并与末尾的答案对照
5. 如果分数 < 70%，请在开始课程前复习推荐的资源

---

## Python 基础（第 1-5 题）

### 第 1 题：数据类型
**难度**：简单

以下哪种 Python 数据类型是**不可变**的？

A) List
B) Dictionary
C) Tuple
D) Set

**你的答案**：____

---

### 第 2 题：列表推导式
**难度**：中等

以下代码的输出是什么？

```python
numbers = [1, 2, 3, 4, 5]
result = [x * 2 for x in numbers if x % 2 == 0]
print(result)
```

A) `[2, 4, 6, 8, 10]`
B) `[4, 8]`
C) `[2, 4]`
D) `[1, 3, 5]`

**你的答案**：____

---

### 第 3 题：函数参数
**难度**：中等

调用 `calculate(10, 5)` 时，该函数将返回什么？

```python
def calculate(a, b=2, *args, **kwargs):
    return a + b
```

A) `10`
B) `15`
C) `12`
D) 错误

**你的答案**：____

---

### 第 4 题：异常处理
**难度**：简单

关于 Python 异常处理，哪个说法是正确的？

A) `finally` 块总是执行，即使异常被抛出
B) `else` 块只在异常被抛出时执行
C) 如果有 `try` 块，必须也有 `finally` 块
D) `except` 一次只能捕获一种异常类型

**你的答案**：____

---

### 第 5 题：虚拟环境
**难度**：简单

Python 虚拟环境的主要目的是什么？

A) 让 Python 代码运行更快
B) 隔离项目依赖并避免冲突
C) 允许 Python 脚本并发执行
D) 自动格式化 Python 代码

**你的答案**：____

---

## Linux/命令行（第 6-10 题）

### 第 6 题：文件权限
**难度**：中等

命令 `chmod 755 script.sh` 的作用是什么？

A) 所有者：读/写/执行，组：读/执行，其他：读/执行
B) 所有者：读/写，组：读/写，其他：读/写
C) 所有人获得全部权限
D) 所有者：仅执行，组：仅读，其他：仅写

**你的答案**：____

---

### 第 7 题：进程管理
**难度**：中等

以下哪个命令用于查找所有正在运行的 Python 进程？

A) `find python`
B) `ps aux | grep python`
C) `ls -la python`
D) `top python`

**你的答案**：____

---

### 第 8 题：重定向和管道
**难度**：简单

以下命令的作用是什么？

```bash
cat logfile.txt | grep ERROR > errors.txt
```

A) 在 logfile.txt 中搜索"ERROR"并显示结果
B) 在 logfile.txt 中搜索"ERROR"并将结果追加到 errors.txt
C) 在 logfile.txt 中搜索"ERROR"并将结果覆盖写入 errors.txt
D) 创建名为 errors.txt 的新文件，包含 logfile.txt 的所有内容

**你的答案**：____

---

### 第 9 题：环境变量
**难度**：简单

如何在 Linux (bash) 中设置环境变量？

A) `SET VAR=value`
B) `export VAR=value`
C) `VAR := value`
D) `env VAR=value`

**你的答案**：____

---

### 第 10 题：磁盘使用
**难度**：简单

哪个命令显示当前目录的磁盘空间使用情况？

A) `df -h`
B) `du -sh`
C) `ls -lh`
D) `free -h`

**你的答案**：____

---

## 使用 Git 进行版本控制（第 11-14 题）

### 第 11 题：Git 基础
**难度**：简单

向 Git 仓库提交更改的正确命令顺序是什么？

A) `git commit -m "message"` → `git add .` → `git push`
B) `git add .` → `git commit -m "message"` → `git push`
C) `git push` → `git add .` → `git commit -m "message"`
D) `git commit -m "message"` → `git push` → `git add .`

**你的答案**：____

---

### 第 12 题：分支
**难度**：中等

`git checkout -b feature-branch` 的作用是什么？

A) 切换到名为 feature-branch 的已有分支
B) 创建名为 feature-branch 的新分支并切换到该分支
C) 删除名为 feature-branch 的分支
D) 将 feature-branch 合并到当前分支

**你的答案**：____

---

### 第 13 题：撤销更改
**难度**：中等

你对一个文件做了更改但尚未暂存。哪个命令可以丢弃这些更改？

A) `git reset --hard`
B) `git checkout -- filename`
C) `git revert filename`
D) `git rm filename`

**你的答案**：____

---

### 第 14 题：远程仓库
**难度**：简单

`git clone` 的作用是什么？

A) 在远程服务器上创建本地仓库的副本
B) 将远程仓库的副本下载到本地
C) 同步本地和远程仓库之间的更改
D) 在远程仓库中创建新分支

**你的答案**：____

---

## 基础网络（第 15-17 题）

### 第 15 题：HTTP 方法
**难度**：简单

哪个 HTTP 方法通常用于从服务器检索数据？

A) POST
B) PUT
C) GET
D) DELETE

**你的答案**：____

---

### 第 16 题：IP 地址
**难度**：中等

以下哪一个是有效的私有 IP 地址？

A) `8.8.8.8`
B) `192.168.1.100`
C) `256.100.50.25`
D) `173.194.45.67`

**你的答案**：____

---

### 第 17 题：端口
**难度**：简单

HTTP 流量的默认端口是什么？

A) 22
B) 443
C) 80
D) 3306

**你的答案**：____

---

## 数据库基础（第 18-20 题）

### 第 18 题：SQL 基础
**难度**：简单

哪条 SQL 命令用于从数据库检索数据？

A) GET
B) SELECT
C) FETCH
D) RETRIEVE

**你的答案**：____

---

### 第 19 题：数据库关系
**难度**：中等

什么是外键？

A) 表的主要标识符
B) 加密的唯一键
C) 引用另一个表主键的字段
D) 授予数据库访问权限的密钥

**你的答案**：____

---

### 第 20 题：SQL 与 NoSQL
**难度**：中等

**选择所有适用项**：关于 NoSQL 数据库，哪些说法是正确的？

A) 它们总是像关系型数据库一样使用固定 schema
B) 它们比 SQL 数据库更容易进行水平扩展
C) MongoDB 和 Redis 是 NoSQL 数据库的例子
D) 它们无法处理结构化数据

**你的答案**（列出所有适用项）：____

---

## 答案

**在完成测验之前不要查看答案！**

<details>
<summary>点击查看答案</summary>

### 答案与解释

**1. C - Tuple**
- Tuple 是不可变的；一旦创建就无法修改
- List、Dictionary 和 Set 是可变的

**2. B - [4, 8]**
- 列表推导式筛选出偶数 (2, 4) 然后乘以 2
- 结果：[2*2, 4*2] = [4, 8]

**3. B - 15**
- 函数调用时 a=10, b=5
- 返回 10 + 5 = 15
- 默认值 b=2 被传入的参数覆盖

**4. A - `finally` 块总是执行，即使异常被抛出**
- finally 块无论是否有异常都会执行
- else 仅在没有异常时执行
- finally 是可选的
- except 可以捕获多种异常类型

**5. B - 隔离项目依赖并避免冲突**
- 虚拟环境创建隔离的 Python 环境
- 防止项目之间的依赖冲突
- 不影响执行速度

**6. A - 所有者：读/写/执行，组：读/执行，其他：读/执行**
- 7 = 读(4) + 写(2) + 执行(1)
- 5 = 读(4) + 执行(1)
- 755 是常见于可执行脚本的权限设置

**7. B - `ps aux | grep python`**
- ps aux 列出所有进程
- grep 过滤包含"python"的行
- 这是查找特定进程的标准方法

**8. C - 在 logfile.txt 中搜索"ERROR"并将结果覆盖写入 errors.txt**
- cat 读取文件
- grep 过滤出"ERROR"
- > 重定向输出并覆盖（>> 会追加）

**9. B - `export VAR=value`**
- export 使变量可用于子进程
- 这是 bash/sh 语法
- SET 是 Windows 语法

**10. B - `du -sh`**
- du = disk usage
- -s = 汇总（总计）
- -h = 人类可读格式
- df 显示文件系统空间，而非目录使用情况

**11. B - `git add .` → `git commit -m "message"` → `git push`**
- 首先暂存更改 (add)
- 然后提交暂存的更改
- 最后推送到远程仓库

**12. B - 创建名为 feature-branch 的新分支并切换到该分支**
- -b 标志创建新分支
- 等同于：git branch feature-branch; git checkout feature-branch
- git switch -c 是新语法

**13. B - `git checkout -- filename`**
- 丢弃对文件的未暂存更改
- 将文件恢复到上次提交后的状态
- git restore filename 是新语法

**14. B - 将远程仓库的副本下载到本地**
- 克隆整个仓库，包括历史记录
- 创建以仓库名命名的新目录
- 自动设置远程追踪

**15. C - GET**
- GET 用于检索数据
- POST 用于发送数据以创建/更新
- PUT 用于更新已有数据
- DELETE 用于删除数据

**16. B - `192.168.1.100`**
- 192.168.x.x 是私有 IP 范围
- 此外还有：10.x.x.x 和 172.16.x.x - 172.31.x.x
- 8.8.8.8 是 Google 的公共 DNS
- 256 超出了有效范围 (0-255)

**17. C - 80**
- HTTP 默认使用端口 80
- HTTPS 使用端口 443
- SSH 使用端口 22
- MySQL 使用端口 3306

**18. B - SELECT**
- SELECT 是用于查询数据的 SQL 命令
- 示例：SELECT * FROM users;
- GET 是 HTTP 方法，不是 SQL

**19. C - 引用另一个表主键的字段**
- 外键建立表之间的关系
- 它们强制参照完整性
- 指向相关联表的主键

**20. B 和 C**
- B：NoSQL 数据库通常更容易水平扩展
- C：MongoDB（文档存储）和 Redis（键值存储）是 NoSQL
- A 错误：NoSQL 数据库通常具有灵活的 schema
- D 错误：NoSQL 可以处理结构化、半结构化和非结构化数据

</details>

---

## 评分指南

### 计算你的分数

统计正确答案的数量：
- 第 1-19 题：每题 1 分
- 第 20 题：1 分（必须同时选中 B 和 C，且不选其他）

**正确总数**：___ / 20

**百分比**：___ %

### 分数解读

| 分数 | 百分比 | 解读 | 建议 |
|------|--------|------|------|
| 18-20 | 90-100% | 优秀 | 自信地继续 |
| 14-17 | 70-89% | 良好 | 继续，复习薄弱环节 |
| 10-13 | 50-69% | 一般 | 开始前先复习资源 |
| 0-9 | 0-49% | 需要改进 | 先完成先决条件 |

### 主题分析

记录每个主题的得分：

| 主题 | 你的得分 | 满分 | 百分比 |
|------|----------|------|--------|
| Python 基础 | ___ | 5 | ___% |
| Linux/命令行 | ___ | 5 | ___% |
| Git 版本控制 | ___ | 4 | ___% |
| 基础网络 | ___ | 3 | ___% |
| 数据库基础 | ___ | 3 | ___% |

**薄弱环节**（某主题 < 60%）：
- [ ] Python 基础
- [ ] Linux/命令行
- [ ] Git 版本控制
- [ ] 基础网络
- [ ] 数据库基础

---

## 补救资源

如果你的分数低于 70% 或存在薄弱环节，请复习以下资源：

### Python 基础
- [Python 官方教程](https://docs.python.org/3/tutorial/)
- [Real Python - Python 基础](https://realpython.com/learning-paths/python3-introduction/)
- [Codecademy Python 课程](https://www.codecademy.com/learn/learn-python-3)
- 练习：在 HackerRank 上完成 50 道 Python 练习

### Linux/命令行
- [Linux Journey](https://linuxjourney.com/)
- [The Linux Command Line 书籍](http://linuxcommand.org/tlcl.php)（免费）
- [Bash 脚本教程](https://www.shellscript.sh/)
- 练习：搭建 Linux VM 并仅使用终端操作一周

### Git 版本控制
- [Pro Git 书籍](https://git-scm.com/book/en/v2)（免费，第 1-3 章）
- [Learn Git Branching](https://learngitbranching.js.org/)（互动式）
- [GitHub Learning Lab](https://lab.github.com/)
- 练习：创建仓库，进行 20 次提交并练习分支

### 基础网络
- [计算机网络：自顶向下方法](https://www.youtube.com/playlist?list=PLhb9gLxUWF1yCIaGxdvP2xBYdLIJLgxW7)（YouTube 讲座）
- [HTTP 基础](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP)
- 练习：使用 curl 和 Postman 发起 API 请求

### 数据库基础
- [SQL 教程 - W3Schools](https://www.w3schools.com/sql/)
- [PostgreSQL 教程](https://www.postgresqltutorial.com/)
- [MongoDB 大学](https://university.mongodb.com/)（免费课程）
- 练习：创建数据库，编写 20 个不同的查询

---

## 重考政策

- 本测验可 **无限次** 重考
- 两次尝试之间等待 **24 小时**
- 利用失败的尝试发现知识盲区
- 3 次尝试后分数仍 < 70% 的，先完成补救资源

---

## 后续步骤

### 如果你通过了（70%+）
1. 复习你做错的题目
2. 继续学习**模块 1：AI 基础设施简介**
3. 保留此测验供日后参考
4. 加入社区学习小组

### 如果你需要复习（<70%）
1. 重点关注得分 < 60% 的主题
2. 完成薄弱环节的推荐资源
3. 动手练习
4. 24 小时后重考
5. 持续复习直到达到 70%+

---

## 学习建议

在重考或继续学习之前：

1. **动手实践**：不要只读，要实际编码和执行命令
2. **间隔重复**：在多天内复习概念
3. **教给别人**：讲解概念以巩固理解
4. **真实项目**：使用这些基础技能构建小项目
5. **社区**：与同伴讨论问题

---

## 题目反馈

发现错误或有建议？请在仓库中提交 Issue，包含：
- 题号
- 你的问题
- 改进建议

---

**祝你好运！** 记住，这个测验是学习工具，而不是障碍。目标是确保你具备在本课程中取得成功所需的基础。
