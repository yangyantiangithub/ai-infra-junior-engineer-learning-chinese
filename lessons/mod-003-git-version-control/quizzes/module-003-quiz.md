# 模块 003：Git & 版本控制 - 综合测验

## 说明

本测验涵盖模块 003：Git & 版本控制的所有主题。包含 30 道题目，涵盖：
- Git 基础与基础知识
- 分支与合并
- 协作工作流
- 高级 Git 技术
- 使用 DVC 和 Git LFS 的 ML 特定工作流

**评分：**
- 每题 1 分
- 总分：30 分
- 及格分数：24/30（80%）

**预计时间：** 45-60 分钟

请回答所有问题，然后在最后核对答案。

---

## 第一部分：Git 基础（第 1-8 题）

### 问题 1：Git 基础
Git 中暂存区的作用是什么？

A) 存储已提交的变更
B) 在提交前临时保存变更
C) 存储远程仓库信息
D) 缓存文件内容以加快访问速度

**答案：** B

**解析：** 暂存区（也称为"index"）是一个中间区域，你可以在其中准备提交的变更。它允许你使用 `git add` 选择性选择哪些变更包含在下一次 commit 中。

---

### 问题 2：Commit 标识
什么唯一标识每个 Git commit？

A) Commit 消息
B) 时间戳
C) SHA-1 哈希
D) 作者姓名

**答案：** C

**解析：** 每个 Git commit 由 SHA-1 哈希（40 字符十六进制字符串）唯一标识。该哈希根据 commit 内容、父 commit、作者、时间戳和消息计算，确保唯一性。

---

### 问题 3：Git 工作流
在 Git 的三态工作流中，三个区域是什么？

A) 远程、本地、缓存
B) 工作目录、暂存区、仓库
C) 编辑、添加、提交
D) 创建、修改、删除

**答案：** B

**解析：** Git 有三个主要区域：工作目录（你编辑文件的地方）、暂存区/Index（你准备 commit 的地方）和仓库（commit 存储的地方）。当你使用 Git 时，文件会经过这些区域。

---

### 问题 4：.gitignore
你不小心将一个 5GB 的模型文件提交到了 Git。你应该怎么做来防止这种情况？

A) 使用更小的模型
B) 先压缩文件
C) 在提交前将文件模式添加到 .gitignore
D) 将其提交到单独的分支

**答案：** C

**解析：** 匹配 `.gitignore` 中模式的文件永远不会被 Git 跟踪。对于 ML 项目，你应该将 `*.h5`、`*.pth`、`models/` 和 `data/` 等模式添加到 `.gitignore`，然后对大文件使用 Git LFS 或 DVC。

---

### 问题 5：查看变更
`git diff` 和 `git diff --staged` 有什么区别？

A) 没有区别，它们是一样的
B) `git diff` 显示未暂存的变更，`git diff --staged` 显示已暂存的变更
C) `git diff` 显示本地变更，`git diff --staged` 显示远程变更
D) `git diff --staged` 速度更快

**答案：** B

**解析：** `git diff` 显示工作目录中尚未暂存的变更。`git diff --staged`（或 `--cached`）显示已经暂存并将包含在下一次 commit 中的变更。

---

### 问题 6：Commit 消息
哪条 commit 消息遵循 conventional commit 最佳实践？

A) "fixed bug"
B) "Updated files"
C) "fix(preprocessing): handle grayscale images correctly"
D) "Changes to preprocessing module"

**答案：** C

**解析：** Conventional commits 使用 `type(scope): subject` 格式。type（feat、fix、docs 等）对变更进行分类，scope 指定变更区域，subject 描述变更。这样可以实现自动化和清晰的历史记录。

---

### 问题 7：Git Log
哪个命令以紧凑的单行格式显示最近 5 个 commit？

A) `git log -5`
B) `git log --oneline -5`
C) `git show -5`
D) `git log --short -5`

**答案：** B

**解析：** `git log --oneline` 以紧凑格式显示 commit（哈希的前 7 个字符 + 消息）。添加 `-5` 将输出限制为最近 5 个 commit。这对于快速浏览近期历史很有用。

---

### 问题 8：撤销变更
你错误地暂存了一个文件。如何在不丢失变更的情况下取消暂存？

A) `git reset --hard <file>`
B) `git checkout <file>`
C) `git restore --staged <file>`
D) `git rm --cached <file>`

**答案：** C

**解析：** `git restore --staged <file>`（或更旧的 `git reset HEAD <file>`）取消暂存文件，同时保留工作目录中的变更。`--hard` 会丢弃变更，`checkout` 可能会丢弃变更，而 `rm --cached` 用于从跟踪中移除。

---

## 第二部分：分支与合并（第 9-16 题）

### 问题 9：创建分支
创建并立即切换到新分支的快捷命令是什么？

A) `git branch -b feature/new`
B) `git switch -c feature/new`
C) `git checkout --new feature/new`
D) `git create feature/new`

**答案：** B

**解析：** `git switch -c <branch-name>` 创建一个新分支并立即切换到它。更旧的语法 `git checkout -b <branch-name>` 同样有效。`-c` 标志表示"create"。

---

### 问题 10：分支命名
哪个分支名称遵循常见的 bug 修复约定？

A) `mybugfix`
B) `fix-the-bug-in-api`
C) `fix/memory-leak-preprocessing`
D) `bugfix_20240115`

**答案：** C

**解析：** 分支命名约定使用 `feature/`、`fix/`、`hotfix/`、`experiment/` 等前缀，后跟带连字符的描述性名称。这使得分支目的一目了然，并有助于自动化。

---

### 问题 11：快进合并
快进合并何时发生？

A) 当分支有冲突的变更时
B) 当目标分支与功能分支没有分叉时
C) 当你使用 `--ff` 标志时
D) 当合并超过两个分支时

**答案：** B

**解析：** 快进发生在目标分支在功能分支创建后没有新 commit 时。Git 只是向前移动分支指针，而不是创建合并 commit。使用 `--no-ff` 强制创建合并 commit。

---

### 问题 12：合并冲突
你正在合并一个分支，并在文件中看到冲突标记。它们是什么意思？

```
<<<<<<< HEAD
code A
=======
code B
>>>>>>> feature-branch
```

A) HEAD 是正确的，feature-branch 是错误的
B) Code A 来自当前分支，code B 来自 feature-branch
C) Git 自动选择了更好的选项
D) 两个版本都会自动保留

**答案：** B

**解析：** 冲突标记显示冲突的变更：`<<<<<<< HEAD` 标记当前分支版本的开始，`=======` 分隔两个版本，`>>>>>>> branch-name` 标记传入分支版本的结束。你必须通过编辑手动解决。

---

### 问题 13：Squash 合并
何时使用 `git merge --squash`？

A) 加快合并过程
B) 将功能分支的所有 commit 合并为一个 commit
C) 压缩文件大小
D) 一次合并多个分支

**答案：** B

**解析：** `--squash` 将功能分支的所有 commit 合并为目标分支上的单个 commit。当功能分支有很多你不希望出现在主分支历史中的小型 WIP commit 时，这很有用。

---

### 问题 14：合并策略
哪个命令通过始终优先选择当前分支的版本来解决冲突？

A) `git merge -X ours feature-branch`
B) `git merge --prefer-current feature-branch`
C) `git merge --keep-mine feature-branch`
D) `git merge -X theirs feature-branch`

**答案：** A

**解析：** `git merge -X ours` 使用"ours"策略，自动解决冲突以支持当前分支。`-X theirs` 则相反。适用于生成的文件，其中某个版本肯定是正确的。

---

### 问题 15：查看分支
哪个命令显示包括远程跟踪分支在内的所有分支？

A) `git branch`
B) `git branch -r`
C) `git branch -a`
D) `git branch --all-remote`

**答案：** C

**解析：** `git branch` 只显示本地分支。`git branch -r` 显示远程跟踪分支。`git branch -a`（--all）同时显示本地和远程跟踪分支，给出所有分支的完整视图。

---

### 问题 16：删除分支
如果你尝试使用 `git branch -d` 删除一个未合并的分支会发生什么？

A) Git 立即删除它
B) Git 显示警告但删除
C) Git 拒绝并建议使用 `-D` 代替
D) Git 首先自动合并它

**答案：** C

**解析：** `git branch -d`（小写）只删除已合并的分支，以保护你免受数据丢失。对于未合并的分支，Git 拒绝并建议使用 `-D`（大写）强制删除。在使用 `-D` 之前务必仔细检查。

---

## 第三部分：协作（第 17-22 题）

### 问题 17：远程仓库
`git remote add upstream <url>` 的作用是什么？

A) 创建一个新的远程仓库
B) 添加一个名为"upstream"的另一个远程仓库的引用
C) 将本地变更上传到远程
D) 下载远程变更

**答案：** B

**解析：** `git remote add <name> <url>` 创建一个指向远程仓库的命名引用。"upstream"通常在你 fork 了项目时指原始仓库，而"origin"指你的 fork。

---

### 问题 18：Fetching vs Pulling
`git fetch` 和 `git pull` 有什么区别？

A) 没有区别，它们是别名
B) `fetch` 下载变更但不合并；`pull` 下载并合并
C) `fetch` 比 `pull` 快
D) `pull` 只适用于主分支

**答案：** B

**解析：** `git fetch` 从远程下载 commit、文件和引用，但不会合并到你的工作中。`git pull` 执行 `fetch` 后跟 `merge`（或 `rebase`），从而更新当前分支。当你想要先查看变更再集成时，使用 `fetch`。

---

### 问题 19：Pull Request
Pull request（PR）的目的是什么？

A) 将变更从远程拉到本地
B) 请求对仓库的写入权限
C) 提出变更并在合并前进行代码审查
D) 请求对 Git 问题的帮助

**答案：** C

**解析：** Pull request 是 GitHub/GitLab 提出变更的机制。它们在合并到主分支之前支持代码审查、讨论、自动化测试和审批工作流。对于团队协作至关重要。

---

### 问题 20：代码审查评论
哪种类型的代码审查评论表示必须修复的阻塞性问题？

A) 💅 Nitpick
B) ❓ Question
C) 👍 Nice
D) 🚨 BLOCKER

**答案：** D

**解析：** 代码审查评论应分类：BLOCKER（合并前必须修复）、重要建议（应该修复）、Nitpicks（可选）、Questions（需要澄清）和 Praise（好的工作）。这有助于作者优先处理反馈。

---

### 问题 21：Fork 工作流
在基于 fork 的工作流中，如何保持你的 fork 与原始仓库同步？

A) 删除你的 fork 并重新 fork
B) `git fetch upstream` 并合并 upstream/main
C) Pull request 自动同步 fork
D) Fork 会自动保持同步

**答案：** B

**解析：** 将原始仓库添加为"upstream"远程，使用 `git fetch upstream` 获取变更，然后使用 `git merge upstream/main` 合并。这使你的 fork 的 main 分支与原始项目保持同步。

---

### 问题 22：为 PR 进行 Rebase
你有一个开放的 PR，但 main 分支已经前进了。你应该怎么做？

A) 关闭 PR 并创建一个新的
B) 将你的分支变基到最新的 main 上并强制推送
C) 什么都不做，GitHub 会自动处理
D) 删除你的分支并重新开始

**答案：** B

**解析：** `git fetch upstream && git rebase upstream/main` 用最新的 main 更新你的分支，将你的 commit 重放在上面。然后 `git push --force-with-lease` 更新 PR。这保持 PR 历史整洁并解决冲突。

---

## 第四部分：高级 Git（第 23-26 题）

### 问题 23：交互式 Rebase
在交互式 rebase 中，"squash"命令的作用是什么？

A) 删除 commit
B) 将 commit 与前一个合并，保留两个消息
C) 编辑 commit 消息
D) 重新排序 commit

**答案：** B

**解析：** 在 `git rebase -i` 中，"squash"将 commit 与前一个合并，并组合两个 commit 消息。"fixup"类似但丢弃消息。这通过合并相关 commit 来清理历史。

---

### 问题 24：Cherry-Picking
何时使用 `git cherry-pick`？

A) 删除特定 commit
B) 从另一个分支应用特定的 commit
C) 选择哪些文件要提交
D) 选择最佳的合并策略

**答案：** B

**解析：** `git cherry-pick <commit-hash>` 将特定 commit 的变更应用到当前分支。适用于从一个分支应用 bug 修复到另一个分支，而无需合并所有变更。

---

### 问题 25：Git Bisect
`git bisect` 解决什么问题？

A) 合并有大量冲突的分支
B) 使用二分查找找到引入 bug 的 commit
C) 将大型 commit 拆分成更小的
D) 自动解决合并冲突

**答案：** B

**解析：** `git bisect` 使用二分查找来找到引入 bug 的 commit。标记一个好（工作正常）和一个坏（损坏）的 commit，Git 会检出中间的 commit 进行测试，直到找到有问题的 commit。可以使用测试脚本自动化。

---

### 问题 26：Reflog
你不小心重置到了一个旧的 commit，丢失了你的工作。如何恢复？

A) 工作永久丢失
B) 使用 `git reflog` 找到丢失的 commit 并恢复它
C) 只能从备份恢复
D) 使用 `git recover`

**答案：** B

**解析：** `git reflog` 显示所有 HEAD 移动的日志，包括来自 reset、rebase 等的"丢失"commit。在 reflog 中找到 commit 哈希，然后 `git checkout <hash>` 或在该点创建一个分支来恢复你的工作。

---

## 第五部分：ML 特定 Git（第 27-30 题）

### 问题 27：DVC 目的
为什么对 ML 数据集使用 DVC（Data Version Control）而不是 Git？

A) DVC 比 Git 更快
B) Git 不是为大型二进制文件设计的；DVC 高效地处理它们
C) DVC 有更好的合并冲突解决
D) ML 框架要求使用 DVC

**答案：** B

**解析：** Git 存储完整的文件历史，这使得它对大型数据集和模型效率低下。DVC 在 Git 中存储元数据，将实际数据存储在单独的存储（S3、GCS 等）中，为数据提供版本控制而不使 Git 仓库膨胀。

---

### 问题 28：Git LFS
在 ML 项目中，哪些类型的文件应该使用 Git LFS 跟踪？

A) Python 代码文件
B) 配置文件
C) 模型权重（.pth、.h5、.pkl）
D) 文档

**答案：** C

**解析：** Git LFS（Large File Storage）专为大型二进制文件设计，如模型权重（.pth、.h5）、pickled 模型（.pkl）、ONNX 文件等。LFS 在 Git 中存储指针，实际文件存储在 LFS 存储中，防止仓库膨胀。

---

### 问题 29：实验跟踪
你正在运行多个 ML 实验。最佳的 Git 实践是什么？

A) 将所有实验提交到 main 分支
B) 创建实验分支并将配置作为 YAML 文件跟踪
C) 不为实验使用 Git
D) 为每个实验创建一个新仓库

**答案：** B

**解析：** 创建实验分支（experiment/name）并将实验配置作为 YAML 文件进行版本控制。这保持了可重现性，允许比较实验，并保持历史整洁。标记成功的实验以便轻松引用。

---

### 问题 30：模型版本控制
如何对用于生产部署的训练模型进行版本控制？

A) 直接将文件复制到生产环境
B) 在训练模型的 commit 上使用 Git tag
C) 创建带有元数据的 git tag，用 LFS 跟踪模型，并在 JSON 中记录
D) 通过邮件发送模型文件

**答案：** C

**解析：** 最佳实践：标记训练 commit（git tag model-v1.0.0），使用 Git LFS 跟踪权重，包含带有训练配置/指标/数据版本的元数据 JSON，并在模型注册表中记录。这确保了从数据到部署模型的完全可追溯性。

---

## 答案汇总

1. B - 暂存区在提交前保存变更
2. C - SHA-1 哈希唯一标识 commit
3. B - 工作目录、暂存区、仓库
4. C - 在提交前将模式添加到 .gitignore
5. B - diff 显示未暂存，diff --staged 显示已暂存
6. C - 带有 type、scope、subject 的 conventional commit 格式
7. B - git log --oneline -5
8. C - git restore --staged 取消暂存而不丢失变更
9. B - git switch -c 创建并切换
10. C - fix/ 前缀加描述性名称
11. B - 当目标分支没有分叉时
12. B - HEAD 是当前分支，feature-branch 是传入分支
13. B - 将所有 commit 合并为一个
14. A - git merge -X ours 优先选择当前分支
15. C - git branch -a 显示所有分支
16. C - Git 拒绝并建议 -D
17. B - 添加指向远程仓库的命名引用
18. B - fetch 下载，pull 下载并合并
19. C - 提出变更以供代码审查
20. D - BLOCKER 表示必须修复的问题
21. B - Fetch upstream 并合并
22. B - 变基到 main 并强制推送
23. B - 与前一个合并，保留两个消息
24. B - 从另一个分支应用特定的 commit
25. B - 使用二分查找找到引入 bug 的 commit
26. B - 使用 reflog 查找和恢复
27. B - Git 对大型二进制文件效率不高
28. C - 模型权重和大型二进制文件
29. B - 带有配置版本控制的实验分支
30. C - Tag + LFS + 元数据 JSON 用于可追溯性

---

## 评分指南

**28-30 正确（93-100%）**: 优秀！你已掌握用于 ML 基础设施的 Git。

**24-27 正确（80-90%）**: 良好！你很好地理解了 Git。复习做错的题目。

**20-23 正确（67-79%）**: 及格，但复习关键概念，特别是：
- 分支工作流
- 协作实践
- ML 特定工具

**低于 20（< 67%）**: 复习课程材料并多加练习：
- 重复练习 01-07
- 重读讲座笔记
- 使用真实仓库练习

---

## 关键概念复习

如果你在某些领域做错了题目，请复习以下主题：

### Git 基础（Q1-8）
- 三态工作流（工作区/暂存区/仓库）
- ML 项目的 .gitignore
- Commit 消息最佳实践
- 查看和搜索历史

### 分支与合并（Q9-16）
- 分支命名约定
- 快进合并 vs 三方合并
- 冲突解决
- 合并策略

### 协作（Q17-22）
- Fork 和 upstream 工作流
- Pull request 最佳实践
- 代码审查指南
- 保持 fork 同步

### 高级技术（Q23-26）
- 交互式 rebase 用于整洁历史
- Cherry-picking 特定 commit
- 使用 bisect 调试
- 使用 reflog 恢复

### ML 工作流（Q27-30）
- DVC 用于数据版本控制
- Git LFS 用于大文件
- 实验跟踪
- 模型版本控制与部署

---

## 下一步

1. **复习** 你做错的题目的相关主题
2. **练习** 这些主题的练习
3. **应用** 在真实的 ML 项目中使用 Git
4. **继续** 到模块 004，如果你得分 80% 或以上

记住：Git 熟练度来自实践。你在真实项目中使用这些工作流越多，它们就会变得越自然！
