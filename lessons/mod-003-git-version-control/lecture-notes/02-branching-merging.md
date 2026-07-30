# 讲座 02：分支与合并

## 目录
1. [引言](#引言)
2. [理解分支](#理解分支)
3. [分支操作](#分支操作)
4. [合并策略](#合并策略)
5. [解决合并冲突](#解决合并冲突)
6. [Git 工作流](#git-工作流)
7. [分支管理最佳实践](#分支管理最佳实践)
8. [AI/ML 项目中的分支](#aiml-项目中的分支)
9. [常见问题与故障排除](#常见问题与故障排除)
10. [总结与要点](#总结与要点)

---

## 引言

### 为什么分支对 AI 基础设施很重要

想象一下，你正在维护一个每秒处理数千请求的生产级 ML 推理 API。你需要：
- 添加新模型版本进行测试（但保持旧版本提供服务）
- 修复数据预处理流水线中的关键 bug
- 尝试新的缓存层
- 所有这些同时进行，而不影响生产环境

没有分支，这是不可能的。有了分支，这一切就变得简单了。

分支允许并行开发。它们是 Git 的杀手级特性——也是 Git 主导版本控制的原因。理解分支能让你从 Git 用户转变为 Git 大师。

### 学习目标

在本讲座结束时，你将：

- 理解 Git 如何实现分支（它们比你想象的要廉价）
- 自信地创建、切换和删除分支
- 使用不同策略合并分支（快进、三方合并、squash）
- 系统地解决合并冲突
- 实现常见的 Git 工作流（Feature Branch、Git Flow、GitHub Flow）
- 将分支策略应用于 ML 实验跟踪
- 使用分支进行环境特定配置（dev、staging、prod）
- 排查常见的分支问题

### AI 基础设施背景

在 AI 基础设施中，分支服务于特定目的：

**开发分支：**
- 功能开发（新模型架构、API 端点）
- 实验跟踪（不同超参数、架构）
- Bug 修复（紧急生产问题）
- 重构（不改变行为的代码改进）

**环境分支：**
- `main/master`：生产就绪代码
- `staging`：预生产测试
- `develop`：功能集成分支

**使用场景：**
- **A/B 测试**：不同模型版本在单独分支上
- **环境配置**：开发与生产配置
- **实验**：尝试新方法而不影响主代码库
- **热修复**：紧急生产修复

---

## 理解分支

### 什么是分支？

在 Git 中，分支只是一个轻量级、可移动的指向 commit 的指针。就是这样。当你创建一个分支时，Git 创建了一个包含它所指向 commit 的 SHA-1 哈希的 40 字节文件。

```
main 分支：       E
                    |
                    v
A ← B ← C ← D ← E
```

每个字母是一个 commit。`main` 分支指向 commit E。

### Git 如何知道你的当前分支

Git 维护一个名为 `HEAD` 的特殊指针，指向当前分支：

```
        HEAD
         |
         v
       main
         |
         v
A ← B ← C ← D ← E
```

当你制作一个 commit 时，当前分支向前移动：

```
        HEAD
         |
         v
       main
         |
         v
A ← B ← C ← D ← E ← F
```

### 创建分支

```bash
# 创建一个新分支
git branch feature/new-model

# 现在分支：
        HEAD
         |
         v
       main
         |
         v
A ← B ← C ← D ← E
         ^
         |
    feature/new-model
```

两个分支指向同一个 commit。没有文件被复制。Git 只是创建了一个指针。

### 切换分支

```bash
# 切换到新分支（旧方式）
git checkout feature/new-model

# 切换到新分支（新方式，Git 2.23+）
git switch feature/new-model

# 切换后：
         main
          |
          v
A ← B ← C ← D ← E
          ^
          |
        HEAD ← feature/new-model
```

现在 `HEAD` 指向 `feature/new-model`。

### 在分支上制作 Commit

```bash
# 进行更改并提交
echo "model = ResNet50()" >> model.py
git add model.py
git commit -m "Add ResNet50 model"

# 现在的树：
         main
          |
          v
A ← B ← C ← D ← E
                 \
                  F ← feature/new-model
                      ^
                      |
                     HEAD
```

`feature/new-model` 分支向前移动了。`main` 分支保持不动。

### 分支是廉价的

在 Git 中创建分支：
- **只需毫秒**
- **占用 41 字节**（40 字节 SHA-1 + 换行符）
- **不复制文件**
- **不复制历史**

这就是为什么 Git 鼓励使用分支。大胆地创建分支吧！

---

## 分支操作

### 创建分支

```bash
# 创建分支（不切换）
git branch feature/add-logging

# 一步创建并切换（旧方式）
git checkout -b feature/add-logging

# 一步创建并切换（新方式）
git switch -c feature/add-logging

# 从特定 commit 创建分支
git branch feature/old-code abc123

# 从远程分支创建分支
git checkout -b local-feature origin/remote-feature
```

### 列出分支

```bash
# 列出本地分支
git branch

# 输出：
#   feature/add-logging
# * main
#   feature/new-model

# * 表示当前分支

# 列出所有分支（包括远程）
git branch -a

# 只列出远程分支
git branch -r

# 显示每个分支的最新 commit
git branch -v

# 显示已合并的分支
git branch --merged

# 显示未合并的分支
git branch --no-merged
```

### 切换分支

```bash
# 切换到已有分支（新方式）
git switch main

# 切换到已有分支（旧方式）
git checkout main

# 创建并切换到新分支
git switch -c feature/experiment

# 切换到上一个分支
git switch -

# 这类似于 shell 中的 `cd -`
```

**重要**：在切换分支之前：
- 提交你的变更，或者
- 暂存你的变更（在讲座 04 中介绍）

否则，Git 可能会阻止切换或将未提交的变更带到新分支。

### 重命名分支

```bash
# 重命名当前分支
git branch -m new-name

# 重命名其他分支
git branch -m old-name new-name

# 示例：将 master 重命名为 main
git branch -m master main
```

### 删除分支

```bash
# 删除已合并的分支（安全）
git branch -d feature/completed

# 强制删除未合并的分支（危险）
git branch -D feature/abandoned

# 删除远程分支
git push origin --delete feature/old-branch
```

除非使用 `-D` 强制，否则 Git 会阻止删除包含未合并变更的分支。

### 查看分支差异

```bash
# 显示 feature 分支中不在 main 中的 commit
git log main..feature/new-model

# 显示分支之间不同的 commit
git log --left-right --oneline main...feature/new-model

# 输出：
# < abc123 Main branch commit
# > def456 Feature branch commit
```

---

## 合并策略

### 合并类型

Git 支持三种主要的合并策略：

1. **快进合并**（最简单）
2. **三方合并**（创建合并 commit）
3. **Squash 合并**（合并 commit）

### 快进合并

当目标分支在功能分支创建后没有变化时：

```
合并前：
       main
        |
        v
A ← B ← C
         \
          D ← E ← feature
                   ^
                   |
                  HEAD

在 main 上执行 git merge feature 后：
                  main
                   |
                   v
A ← B ← C ← D ← E
            ^
            |
          feature
```

Git 只是将 `main` 指针向前移动。不创建合并 commit。

```bash
# 执行快进合并
git checkout main
git merge feature
```

输出：
```
Updating abc123..def456
Fast-forward
 model.py | 10 ++++++++
 1 file changed, 10 insertions(+)
```

### 三方合并

当两个分支都发生了分叉时：

```
合并前：
         main
          |
          v
A ← B ← C ← F ← G
         \
          D ← E ← feature
                   ^
                   |
                  HEAD

在 main 上执行 git merge feature 后：
              main
               |
               v
A ← B ← C ← F ← G ← M
         \         /
          D ← E ←-/  feature
```

Git 创建一个带有两个父 commit 的新合并 commit（M）。

```bash
# 执行三方合并
git checkout main
git merge feature
```

Git 打开编辑器输入合并 commit 消息：
```
Merge branch 'feature' into main

# Please enter a commit message to explain why this merge is necessary.
```

### 阻止快进（总是创建合并 commit）

```bash
# 即使可能快进，也强制创建合并 commit
git merge --no-ff feature

# 这会创建显式的合并 commit，表明该功能已被合并
```

**何时使用 `--no-ff`：**
- 想要保留功能分支曾存在的记录
- 需要一次操作回滚整个功能
- 遵循 Git Flow 工作流

### Squash 合并

将功能分支的所有 commit 合并为单个 commit：

```
squash 合并前：
       main
        |
        v
A ← B ← C
         \
          D ← E ← F ← feature
             (3 个 commit)

执行 git merge --squash feature 后：
              main
               |
               v
A ← B ← C ← D'
        (所有变更在一个 commit 中)
```

```bash
# Squash 合并
git checkout main
git merge --squash feature
git commit -m "Add new model (squashed from 3 commits)"
```

**何时使用 squash：**
- 功能分支有凌乱的 commit 历史
- 希望在 main 分支上有干净、线性的历史
- 遵循 GitHub/GitLab PR 工作流

**缺点**：丢失功能分支的 commit 历史。

### Rebase（合并的替代方案）

除了合并，你也可以 rebase——将 commit 从一个分支重放到另一个分支上：

```
rebase 前：
         main
          |
          v
A ← B ← C ← F ← G
         \
          D ← E ← feature

在 feature 上执行 git rebase main 后：
                      feature
                       |
                       v
A ← B ← C ← F ← G ← D' ← E'

注意：D 和 E 被重放为 D' 和 E'（新 commit）
```

```bash
# 将 feature 变基到 main
git checkout feature
git rebase main

# 或者：从 main 变基
git rebase main feature
```

**Rebase 创建线性历史**（没有合并 commit），但会重写 commit。

**Rebase 的黄金法则**：永远不要对已推送到共享分支的 commit 执行 rebase。它会重写历史并给协作者带来问题。

### 对比：Merge vs. Rebase

| 方面 | Merge | Rebase |
|--------|-------|--------|
| 历史 | 保留完整历史 | 创建线性历史 |
| Commits | 创建合并 commit | 无合并 commit |
| 冲突 | 解决一次 | 可能需要多次解决 |
| 安全性 | 对共享分支安全 | 对共享分支危险 |
| 可追溯性 | 显示功能何时合并 | 更难追踪功能集成 |
| 使用场景 | 共享分支、发布 | 本地清理、功能分支 |

**对于 AI/ML 项目**：Merge 通常更安全，推荐用于共享分支。Rebase 适用于推送前清理本地 commit。

---

## 解决合并冲突

### 什么导致冲突？

当同一文件的相同部分在两个分支中被以不同方式修改时，就会发生合并冲突：

```
main 分支：
  batch_size = 32

feature 分支：
  batch_size = 64
```

Git 无法自动决定保留哪个值，所以它来问你。

### 冲突标记

当发生冲突时，Git 向文件中添加标记：

```python
def train_model():
<<<<<<< HEAD (current branch: main)
    batch_size = 32
=======
    batch_size = 64
>>>>>>> feature/new-config (incoming branch)

    model.train(batch_size=batch_size)
```

分解标记：
- `<<<<<<< HEAD`：你当前分支版本的开始
- `=======`：两个版本之间的分隔符
- `>>>>>>> branch-name`：传入分支版本的结束

### 解决冲突：分步指南

#### 步骤 1：尝试合并

```bash
git checkout main
git merge feature/new-config
```

输出：
```
Auto-merging config.py
CONFLICT (content): Merge conflict in config.py
Automatic merge failed; fix conflicts and then commit the result.
```

#### 步骤 2：识别冲突文件

```bash
git status
```

输出：
```
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   config.py
```

#### 步骤 3：打开文件并解决

```python
# 原始冲突文件：
def train_model():
<<<<<<< HEAD
    batch_size = 32
=======
    batch_size = 64
>>>>>>> feature/new-config

    model.train(batch_size=batch_size)

# 解决后（选择一个，或创建新方案）：
def train_model():
    batch_size = 64  # 选择了 feature 分支的值

    model.train(batch_size=batch_size)
```

完全删除冲突标记。

#### 步骤 4：标记为已解决

```bash
git add config.py
```

#### 步骤 5：完成合并

```bash
git commit
```

Git 提供一个默认的合并 commit 消息：
```
Merge branch 'feature/new-config' into main

# Conflicts:
#   config.py
```

### 高级冲突解决

#### 查看每个分支的变化

```bash
# 查看你的版本（当前分支）
git show :2:config.py

# 查看他们的版本（传入分支）
git show :3:config.py

# 查看共同祖先版本
git show :1:config.py
```

#### 使用合并工具

```bash
# 配置合并工具（一次性设置）
git config --global merge.tool vimdiff

# 启动合并工具
git mergetool
```

流行的合并工具：
- **vimdiff**（终端）
- **meld**（GUI）
- **kdiff3**（GUI）
- **VS Code**（内置）

#### 接受一方的所有变更

```bash
# 保留当前分支的所有变更（ours）
git checkout --ours config.py
git add config.py

# 保留传入分支的所有变更（theirs）
git checkout --theirs config.py
git add config.py
```

#### 中止有问题的合并

```bash
# 撤销合并并返回到合并前状态
git merge --abort
```

### 预防冲突

1. **频繁拉取**：与 main 分支保持同步
2. **沟通**：协调谁在处理哪些文件
3. **小 commit**：以小单位解决冲突更容易
4. **合并前 rebase**：在合并前用 main 更新功能分支
5. **使用 .gitattributes**：为特定文件定义合并策略

示例 `.gitattributes`：
```
# 永远不合并这些文件，始终保留 ours
config/local.yaml merge=ours

# 二进制文件不应合并
*.pkl binary
*.h5 binary
```

---

## Git 工作流

### Feature Branch 工作流

适合团队的最简单工作流：

```
main ← 始终可部署

feature/user-auth ← 新功能分支
feature/api-v2    ← 另一个功能分支
```

**流程**：
1. 从 `main` 为每个功能创建分支
2. 在功能分支上开发和提交
3. 完成后合并回 `main`
4. 删除功能分支

```bash
# 开始功能
git checkout main
git pull origin main
git checkout -b feature/add-caching

# 开发
git add .
git commit -m "Implement Redis caching layer"

# 完成功能
git checkout main
git pull origin main
git merge feature/add-caching
git push origin main
git branch -d feature/add-caching
```

**最适合**：小团队、简单项目、持续部署。

### Git Flow

一个更结构化的工作流，包含多种分支类型：

```
main ← 生产代码，打标签发布
 |
develop ← 集成分支
 |
 ├─ feature/* ← 功能分支
 ├─ release/* ← 发布准备
 └─ hotfix/*  ← 紧急修复
```

**分支类型**：

1. **main**：仅生产就绪代码
2. **develop**：功能集成分支
3. **feature/***：新功能（从 `develop` 分支）
4. **release/***：发布准备（从 `develop` 分支）
5. **hotfix/***：紧急生产修复（从 `main` 分支）

**功能工作流**：
```bash
# 开始功能
git checkout develop
git checkout -b feature/new-model

# 开发和提交
git add .
git commit -m "Add transformer model"

# 完成功能
git checkout develop
git merge --no-ff feature/new-model
git branch -d feature/new-model
git push origin develop
```

**发布工作流**：
```bash
# 开始发布
git checkout develop
git checkout -b release/1.2.0

# 最终测试、bug 修复、版本更新
git commit -m "Bump version to 1.2.0"

# 完成发布
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0
```

**热修复工作流**：
```bash
# 开始热修复
git checkout main
git checkout -b hotfix/critical-bug

# 修复 bug
git commit -m "Fix memory leak in inference"

# 完成热修复
git checkout main
git merge --no-ff hotfix/critical-bug
git tag -a v1.2.1 -m "Hotfix: memory leak"
git checkout develop
git merge --no-ff hotfix/critical-bug
git branch -d hotfix/critical-bug
```

**最适合**：计划发布、较大团队、版本化软件。

### GitHub Flow

比 Git Flow 更简单，针对持续部署优化：

```
main ← 始终可部署
 |
 ├─ feature/branch-1
 ├─ feature/branch-2
 └─ fix/branch-3
```

**流程**：
1. `main` 分支始终可部署
2. 从 `main` 创建描述性分支
3. 定期提交到你的分支
4. 打开 Pull Request 进行讨论
5. 从分支部署用于测试（可选）
6. 审查后合并到 `main`
7. 立即从 `main` 部署

```bash
# 开始功能
git checkout main
git pull origin main
git checkout -b improve-model-accuracy

# 开发
git add .
git commit -m "Implement ensemble model"
git push origin improve-model-accuracy

# 在 GitHub 上创建 Pull Request
# 批准和 CI 通过后：
git checkout main
git pull origin main
```

**最适合**：Web 应用、持续部署、GitHub/GitLab 项目。

### 主干开发

单分支，功能分支寿命极短：

```
main ← trunk（每个人都频繁提交）
 |
 └─ tiny-feature（存在不到 24 小时）
```

**关键原则**：
- 频繁提交到 `main`（每天多次）
- 功能分支寿命极短（小时，不是天）
- 对不完整功能使用功能开关
- 高度依赖自动化测试

**最适合**：经验丰富的团队、强大的 CI/CD、功能开关基础设施。

---

## 分支管理最佳实践

### 命名约定

使用描述性、分层的分支名称：

```bash
# 功能分支
feature/add-model-serving
feature/user-authentication
feature/ml-monitoring

# Bug 修复分支
fix/memory-leak
fix/api-timeout
bugfix/training-crash

# 实验分支
experiment/new-architecture
exp/hyperparameter-tuning

# 热修复分支
hotfix/critical-security-issue
hotfix/production-crash

# 发布分支
release/v1.2.0
release/2023-10-15
```

**模式**：
- 使用 `/` 创建层次结构
- 使用 `-` 分隔单词
- 描述性但简洁
- 包含工单/问题编号：`feature/MOD-123-add-caching`

### 分支生命周期

**短寿分支**（小时到天）：
- 主干开发中的功能分支
- Bug 修复
- 小功能

**中寿分支**（天到周）：
- Feature branch 工作流中的功能分支
- 实验分支

**长寿分支**（永久）：
- `main` / `master`
- `develop`（在 Git Flow 中）
- 环境分支（staging、production）

**经验法则**：分支存在时间越长，合并越困难。尽早并经常合并。

### 保持分支最新

```bash
# 用 main 的变更更新功能分支
git checkout feature/my-feature
git fetch origin
git merge origin/main

# 或者使用 rebase 以保持更干净的历史（如果分支未共享）
git checkout feature/my-feature
git fetch origin
git rebase origin/main
```

定期更新可以避免以后出现大规模、困难的合并。

### 清理分支

```bash
# 列出已合并到 main 的分支
git branch --merged main

# 删除所有已合并的分支（小心！）
git branch --merged main | grep -v "\*" | xargs -n 1 git branch -d

# 修剪不再存在的远程跟踪分支
git fetch --prune

# 或者配置自动修剪
git config --global fetch.prune true
```

---

## AI/ML 项目中的分支

### 实验分支

跟踪不同的 ML 实验：

```bash
# 创建实验分支
git checkout -b experiment/resnet50-baseline
git checkout -b experiment/resnet50-dropout-05
git checkout -b experiment/vit-transformer
```

每个分支包含特定实验的代码。在分支中提交实验结果：

```bash
# experiment/resnet50-baseline
git add config.yaml results.json
git commit -m "Baseline: accuracy=0.87, loss=0.34"

# experiment/resnet50-dropout-05
git add config.yaml results.json
git commit -m "With dropout=0.5: accuracy=0.89, loss=0.31"
```

比较实验：
```bash
git diff experiment/resnet50-baseline experiment/resnet50-dropout-05 -- config.yaml
```

将获胜实验合并到 main：
```bash
git checkout main
git merge experiment/resnet50-dropout-05
```

### 模型版本分支

跟踪生产中的不同模型版本：

```
main ← 当前开发
 |
 ├─ model/v1.0 ← 生产模型版本 1.0
 ├─ model/v1.1 ← 生产模型版本 1.1
 └─ model/v2.0-beta ← 下一个主要版本
```

### 环境特定分支

**不推荐**但有时会使用：

```
production ← 生产环境配置
staging    ← 预发布环境配置
develop    ← 开发环境配置
```

**更好的方法**：使用带环境特定配置文件的单分支：
```
configs/
  ├── dev.yaml
  ├── staging.yaml
  └── prod.yaml
```

### 数据流水线分支

用于数据工程工作：

```bash
git checkout -b pipeline/add-feature-engineering
git checkout -b pipeline/optimize-preprocessing
git checkout -b pipeline/add-data-validation
```

---

## 常见问题与故障排除

### 问题 1：意外提交到错误分支

**症状**：在 `main` 上而不是功能分支上做了 commit。

**解决方案**：
```bash
# 从当前位置创建功能分支
git branch feature/my-feature

# 将 main 重置为 origin（从 main 移除 commit）
git reset --hard origin/main

# 切换到功能分支（包含你的 commit）
git checkout feature/my-feature
```

### 问题 2：需要将未提交的变更移到另一个分支

**症状**：在错误的分支上开始工作。

**解决方案**：
```bash
# 暂存你的变更
git stash

# 切换到正确的分支
git checkout correct-branch

# 应用暂存的变更
git stash pop
```

### 问题 3：合并冲突太复杂

**症状**：合并冲突压倒性的。

**解决方案**：
```bash
# 中止合并
git merge --abort

# 尝试一次变基一个 commit
git rebase -i origin/main

# 或向团队成员寻求帮助
```

### 问题 4：删除了包含未合并工作的分支

**症状**：不小心删除了一个分支。

**解决方案**：
```bash
# 找到已删除分支的最后一个 commit
git reflog

# 输出：
# abc123 HEAD@{0}: checkout: moving from deleted-branch to main
# def456 HEAD@{1}: commit: My last commit on deleted-branch

# 重新创建分支
git branch recovered-branch def456
```

Git 在 reflog 中保留已删除分支约 30 天。

### 问题 5：分支与远程分叉

**症状**：无法推送，因为分支已分叉。

**解决方案**：
```bash
# 查看不同之处
git fetch origin
git log HEAD..origin/main --oneline

# 选项 1：合并远程变更
git pull origin main

# 选项 2：变基到远程
git pull --rebase origin main

# 选项 3：强制推送（危险！）
git push --force-with-lease origin main
```

**除非团队协调一致，否则永远不要对共享分支强制推送。**

---

## 总结与要点

### 核心概念

1. **分支是轻量级的**：只是指向 commit 的 41 字节指针
2. **HEAD 跟踪当前分支**：显示你所在的位置
3. **三种合并策略**：快进、三方合并、squash
4. **Rebase 重写历史**：本地有用，远程危险
5. **冲突是正常的**：学会系统地解决它们

### 必备命令速查表

```bash
# 分支操作
git branch                    # 列出分支
git branch <name>            # 创建分支
git switch <name>            # 切换到分支（新）
git switch -c <name>         # 创建并切换（新）
git checkout <name>          # 切换到分支（旧）
git checkout -b <name>       # 创建并切换（旧）
git branch -d <name>         # 删除已合并的分支
git branch -D <name>         # 强制删除分支

# 合并
git merge <branch>           # 将分支合并到当前分支
git merge --no-ff <branch>   # 强制合并 commit
git merge --squash <branch>  # Squash 合并
git merge --abort            # 中止合并

# 变基
git rebase <branch>          # 变基到分支
git rebase -i <commit>       # 交互式变基

# 冲突解决
git status                   # 查看冲突文件
git diff                     # 查看冲突详情
git add <file>              # 标记为已解决
git merge --abort           # 中止合并
```

### 工作流选择指南

| 工作流 | 团队规模 | 发布节奏 | 复杂度 | 最适合 |
|----------|-----------|-----------------|------------|----------|
| Feature Branch | 小 | 持续 | 低 | 初创公司、Web 应用 |
| Git Flow | 中-大 | 计划 | 高 | 版本化软件、移动应用 |
| GitHub Flow | 任意 | 持续 | 中 | Web 应用、SaaS、API |
| 主干开发 | 大（经验丰富） | 持续 | 低 | 成熟团队、强大 CI/CD |

### 最佳实践

1. **尽早分支、经常分支**：分支是廉价的
2. **保持分支短寿**：在几天内合并，而不是几周
3. **描述性地命名分支**：`feature/add-user-auth`，而不是 `feature1`
4. **定期更新分支**：经常从 main 合并或变基
5. **删除已合并的分支**：保持仓库整洁
6. **永远不要强制推送共享分支**：需要协调
7. **合并前审查**：使用 Pull Request
8. **编写有意义的合并消息**：解释为什么合并功能
9. **使用一致的工作流**：团队应同意一个工作流
10. **自动化测试**：CI/CD 应测试所有分支

### 对于 AI/ML 项目

- **实验跟踪**：每个实验一个分支，提交结果
- **模型版本控制**：用模型版本打标签发布
- **数据流水线隔离**：数据工程单独分支
- **可重现性**：将精确配置与代码一起提交
- **A/B 测试**：使用分支管理不同模型版本

### 下一步

在下一讲中，我们将介绍：
- 使用 Git 协作（远程仓库、GitHub/GitLab）
- Pull Request 和代码审查
- Fork 和为开源做贡献
- Issue 和项目管理
- CI/CD 集成

### 扩展阅读

- Git 分支模型：https://nvie.com/posts/a-successful-git-branching-model/
- GitHub Flow：https://guides.github.com/introduction/flow/
- 主干开发：https://trunkbaseddevelopment.com/

---

**练习**：完成练习 03 和 04，通过动手场景掌握分支和冲突解决。
