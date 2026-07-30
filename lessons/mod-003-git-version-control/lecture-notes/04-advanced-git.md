# Lecture 04: 高级 Git 技术

## Table of Contents
1. [引言](#introduction)
2. [交互式 Rebase](#interactive-rebasing)
3. [Cherry-Pick commit](#cherry-picking-commits)
4. [Stash 更改](#stashing-changes)
5. [Git Hooks](#git-hooks)
6. [Submodule 和 Subtree](#submodules-and-subtrees)
7. [Tag 和 Release](#tags-and-releases)
8. [高级历史操作](#advanced-history-manipulation)
9. [Git 在机器学习中的应用](#git-for-machine-learning)
10. [性能与优化](#performance-and-optimization)
11. [恢复与灾难管理](#recovery-and-disaster-management)
12. [总结与要点](#summary-and-key-takeaways)

---

## 引言

### 为什么要掌握高级 Git

你已经学会了 Git 基础、branch 和协作。你可以处理日常工作。那么为什么还要更进一步？

高级 Git 技术能让你从称职变为杰出：
- **在他人看到之前清理杂乱的历史**
- **用 hooks 自动化重复性任务**
- **从让他人恐慌的灾难中恢复**
- **针对 ML 特有挑战优化工作流**
- **用 submodule 管理复杂项目**
- **以手术般精准的方式处理紧急情况**

在 AI 基础设施领域，这些技能区分了初级工程师和高级工程师。

### 学习目标

学完本讲后，你将：

- 使用交互式 rebase 来清理 commit 历史
- 在分支之间 cherry-pick 特定 commit
- 用 stash 暂存进行中的工作以便快速切换上下文
- 创建 Git hooks 来自动化测试、linting 和部署
- 用 submodule 管理大型 ML 项目
- 正确使用 tag 进行模型版本管理
- 使用高级命令：reflog、bisect、filter-branch
- 从看似灾难性的错误中恢复
- 应用针对 ML 工作流的 Git 最佳实践
- 优化大规模仓库的 Git 性能

### AI 基础设施背景

高级 Git 能够实现：

**实验管理：**
- 在 merge 前清理实验性 commit
- Cherry-pick 成功的实验到生产环境
- 用 tag 标记模型版本以实现可复现性

**自动化：**
- Pre-commit hooks：检查代码格式、检测 secrets
- Pre-push hooks：在 push 前运行测试
- Post-merge hooks：触发模型重新训练

**项目组织：**
- Submodule 用于共享库（数据处理工具、模型架构）
- Subtree 用于供应商依赖
- Tag 用于模型发布和部署

**灾难恢复：**
- 恢复已删除的分支
- 修复意外推送的 secrets
- 撤销有问题的 merge

---

## 交互式 Rebase

### 什么是交互式 Rebase

交互式 rebase（`git rebase -i`）让你编辑 commit 历史：
- 重新排序 commit
- 将多个 commit 压缩为一个
- 编辑 commit message
- 拆分 commit
- 删除 commit

**黄金法则**：只 rebase 那些尚未 push 到共享分支的 commit。

### 何时使用交互式 Rebase

**使用场景：**
- 在创建 PR 前清理杂乱的本地 commit
- 修正 commit message 中的拼写错误
- 按逻辑顺序重新排列 commit
- 将零散的 WIP commit 合并为有意义的 commit

**示例场景**：
```
Your local commits:
* 789 Fix typo
* 456 WIP: still working
* 123 WIP: add feature
```

你想要干净的历史：
```
* abc Implement user authentication feature
```

### 交互式 Rebase 分步讲解

#### 第 1 步：启动交互式 Rebase

```bash
# 对最近 3 个 commit 进行 rebase
git rebase -i HEAD~3

# 从指定 commit 开始 rebase（不包括该 commit）
git rebase -i abc123

# 从分支起点开始 rebase
git rebase -i main
```

#### 第 2 步：选择操作

Git 会打开一个编辑器显示 commit：

```
pick 123 WIP: add feature
pick 456 WIP: still working
pick 789 Fix typo

# Rebase abc..789 onto abc (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

#### 第 3 步：修改计划

**将所有 commit 压缩为一个：**
```
pick 123 WIP: add feature
squash 456 WIP: still working
squash 789 Fix typo
```

保存并关闭编辑器。

#### 第 4 步：编辑 Commit Message

Git 会打开另一个编辑器供你编辑合并后的 commit message：

```
# This is a combination of 3 commits.
# The first commit's message is:
WIP: add feature

# This is the 2nd commit message:
WIP: still working

# This is the 3rd commit message:
Fix typo
```

替换为：
```
Implement user authentication feature

Added login/logout functionality with JWT tokens.
Includes input validation and error handling.
```

保存并关闭。

#### 第 5 步：结果

```bash
git log --oneline
# abc Implement user authentication feature
```

三个杂乱的 commit 现在变成了一个干净的 commit。

### 交互式 Rebase 操作

**Pick**（默认）：照原样使用 commit
```
pick 123 Add feature
```

**Reword**：修改 commit message
```
reword 123 Add feature
# Git 将打开编辑器来修改 message
```

**Edit**：停留在该 commit 以便修改
```
edit 123 Add feature
# Git 暂停；你可以进行修改
git add modified_file.py
git commit --amend
git rebase --continue
```

**Squash**：与上一个 commit 合并，保留两者的 message
```
pick 123 Add feature
squash 456 Add tests
# Git 合并 commit，让你编辑合并后的 message
```

**Fixup**：与上一个 commit 合并，丢弃此 commit 的 message
```
pick 123 Add feature
fixup 456 Fix typo
# Git 合并 commit，只保留第一个 message
```

**Drop**：删除 commit
```
drop 123 Experimental change
```

**Exec**：运行 shell 命令
```
pick 123 Add feature
exec pytest tests/
pick 456 Add another feature
exec pytest tests/
# 在每个 commit 后运行测试；如果测试失败则停止
```

### 重排序 Commit

只需重新排列各行：

```
pick 789 Fix documentation
pick 123 Add feature
pick 456 Add tests
```

Commit 将按此顺序应用。

### 拆分 Commit

**场景**：一个 commit 做了太多事，你想拆分开。

```bash
git rebase -i HEAD~1
# 将该 commit 标记为 'edit'

# Git 在该 commit 处暂停
git reset HEAD^   # 取消暂存所有内容

# 现在分别提交更改
git add file1.py
git commit -m "Add feature A"

git add file2.py
git commit -m "Add feature B"

# 继续 rebase
git rebase --continue
```

### 中止 Rebase

如果出现问题：
```bash
git rebase --abort
```

返回到 rebase 前的状态。

---

## Cherry-Pick commit

### 什么是 Cherry-Pick

Cherry-pick 将某个特定 commit 从一个分支应用到另一个分支，而不需要 merge 整个分支。

```
main:     A ← B ← C
                    ↑ cherry-pick D
feature:  A ← B ← D ← E ← F
```

Cherry-pick 之后：
```
main:     A ← B ← C ← D'
feature:  A ← B ← D ← E ← F
```

注意：`D'` 是一个新的 commit，与 `D` 有相同的更改但 hash 不同。

### 何时使用 Cherry-Pick

**使用场景：**
- Hotfix：将紧急修复从 develop 应用到生产分支
- 选择性 merge：只想取特定 commit，而非整个分支
- Bug 修复：在功能分支完成前将修复应用到 main
- 实验：尝试另一个分支的某个 commit

### Cherry-Pick 基本用法

```bash
# Cherry-pick 单个 commit
git checkout main
git cherry-pick abc123

# Cherry-pick 多个 commit
git cherry-pick abc123 def456 ghi789

# Cherry-pick 一段范围的 commit
git cherry-pick abc123..def456
```

### Cherry-Pick 的选项

```bash
# Cherry-pick 但不提交（先审查）
git cherry-pick -n abc123
# 根据需要修改
git commit

# Cherry-pick 并编辑 commit message
git cherry-pick -e abc123

# Cherry-pick 时记录原始 commit
git cherry-pick -x abc123
# 会在 message 中添加 "(cherry picked from commit abc123)"
```

### 处理 Cherry-Pick 冲突

如果发生冲突：

```bash
git cherry-pick abc123

# CONFLICT (content): Merge conflict in model.py
# Automatic cherry-pick failed; fix conflicts and commit

# 在文件中解决冲突
git add model.py
git cherry-pick --continue

# 或中止
git cherry-pick --abort
```

### Cherry-Pick 用例：Hotfix

**场景**：生产环境出现严重 bug，修复已在 develop 分支上。

```bash
# 当前状态：
# main:    A ← B ← C (production)
# develop: A ← B ← D ← E ← F (fix in E)

# 应用修复到生产分支
git checkout main
git cherry-pick <commit-hash-of-E>
git push origin main

# 部署到生产环境
# main:    A ← B ← C ← E'
```

---

## Stash 更改

### 什么是 Stash

Stash 可以临时搁置（stash）更改，让你无需 commit 就能切换分支。

**场景**：
- 正在处理功能 A
- 有紧急 bug 需要切换到另一个分支
- 更改还没准备好 commit
- 解决方案：将其 stash

### 基本 Stashing

```bash
# Stash 当前更改
git stash

# 输出：
# Saved working directory and index state WIP on main: abc123 Last commit message

# 现在工作目录是干净的
git status
# nothing to commit, working tree clean

# 切换分支，处理紧急工作
git checkout hotfix
# 修复 bug，commit，push

# 回到原来的工作
git checkout main
git stash pop   # 应用并从 stash 中移除
```

### Stash 命令

```bash
# 带 message 的 stash
git stash push -m "WIP: implementing caching"

# Stash 包含未跟踪的文件
git stash -u

# Stash 包含被忽略的文件
git stash -a

# 列出所有 stash
git stash list
# stash@{0}: WIP on main: abc123 Last commit
# stash@{1}: WIP on feature: def456 Previous commit

# 显示 stash 内容
git stash show stash@{0}

# 显示详细 diff
git stash show -p stash@{0}
```

### 应用 Stash

```bash
# 应用最近的 stash（保留在 stash 列表中）
git stash apply

# 应用指定 stash
git stash apply stash@{1}

# 应用并从 stash 列表中移除
git stash pop

# 应用到不同分支
git checkout other-branch
git stash apply stash@{0}
```

### 管理 Stash

```bash
# 删除指定 stash
git stash drop stash@{0}

# 删除所有 stash
git stash clear

# 从 stash 创建分支
git stash branch feature/new-work stash@{0}
# 创建分支，应用 stash，删除 stash
```

### 部分 Stash

```bash
# 只 stash 特定文件
git stash push -m "Partial stash" file1.py file2.py

# 交互式 stashing
git stash -p
# Git 会逐个询问每个更改：
# Stash this hunk [y,n,q,a,d,/,s,e,?]?
```

### Stash 用例：上下文切换

**场景**：正在实现新的模型架构时，出现了关键的生产问题。

```bash
# 正在开发新模型
# 多个文件已修改，但尚未准备好 commit

# 紧急：生产问题
git stash push -u -m "WIP: transformer architecture"

# 修复生产问题
git checkout main
git checkout -b hotfix/memory-leak
# 修复问题
git add .
git commit -m "Fix memory leak in batch processing"
git push origin hotfix/memory-leak

# 回到模型开发工作
git checkout feature/transformer
git stash pop

# 继续工作
```

---

## Git Hooks

### 什么是 Git Hooks

Git hooks 是 Git 在特定时间点自动执行的脚本：
- commit 之前/之后
- push 之前/之后
- merge 之前/之后
- 等等

Hooks 能够实现自动化：
- 在 commit 前检查代码格式
- 在 push 前运行测试
- 检测 secrets
- 格式化 commit message
- merge 后部署

### Hooks 存储位置

Hooks 存储在 `.git/hooks/` 目录下：

```bash
ls .git/hooks/
# applypatch-msg.sample
# pre-commit.sample
# pre-push.sample
# ...
```

移除 `.sample` 扩展名即可激活。

### 客户端 Hooks

**pre-commit**：在 commit 创建前运行
- 检查代码格式
- 检测 secrets
- 运行快速测试

**prepare-commit-msg**：修改 commit message 模板
- 添加工单号
- 强制执行 message 格式

**commit-msg**：验证 commit message
- 强制执行 message 约定
- 检查 message 长度

**post-commit**：在 commit 后运行
- 发送通知
- 记录 commit

**pre-push**：在 push 前运行
- 运行完整测试套件
- 检查分支命名
- 防止强制 push

### 创建 Pre-Commit Hook

**示例**：防止提交含有 "TODO" 注释的文件。

创建 `.git/hooks/pre-commit`：

```bash
#!/bin/bash

# Check for TODO comments
if git grep -n "TODO" -- '*.py'; then
    echo "Error: Commit contains TODO comments"
    echo "Remove TODOs or skip this check with --no-verify"
    exit 1
fi

echo "Pre-commit checks passed"
exit 0
```

赋予可执行权限：
```bash
chmod +x .git/hooks/pre-commit
```

测试它：
```bash
echo "# TODO: fix this" >> model.py
git add model.py
git commit -m "Add model"

# 输出：
# model.py:1:# TODO: fix this
# Error: Commit contains TODO comments
# Remove TODOs or skip this check with --no-verify
```

### Pre-Commit Hook：代码格式化

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running Black (formatter)..."
black --check .
if [ $? -ne 0 ]; then
    echo "Formatting issues found. Run 'black .' to fix."
    exit 1
fi

echo "Running Flake8 (linter)..."
flake8 .
if [ $? -ne 0 ]; then
    echo "Linting issues found. Fix them before committing."
    exit 1
fi

echo "Running MyPy (type checker)..."
mypy src/
if [ $? -ne 0 ]; then
    echo "Type checking failed."
    exit 1
fi

echo "All pre-commit checks passed!"
exit 0
```

### Pre-Push Hook：测试

```bash
#!/bin/bash
# .git/hooks/pre-push

echo "Running tests before push..."
pytest tests/ -v

if [ $? -ne 0 ]; then
    echo "Tests failed. Fix them before pushing."
    exit 1
fi

echo "All tests passed!"
exit 0
```

### Pre-Commit Hook：Secrets 检测

对于处理凭据的 ML 项目至关重要：

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check for potential secrets
if git diff --cached | grep -iE "(api[_-]?key|secret|password|token|credentials).*=.*['\"][^'\"]{10,}['\"]"; then
    echo "Error: Potential secret detected in staged changes"
    echo "Never commit API keys, passwords, or tokens"
    exit 1
fi

# Check for AWS keys
if git diff --cached | grep -E "AKIA[0-9A-Z]{16}"; then
    echo "Error: AWS access key detected"
    exit 1
fi

echo "No secrets detected"
exit 0
```

### 使用 Pre-Commit 框架

除了手动编写 hook 脚本，还可以使用 `pre-commit` 框架：

安装：
```bash
pip install pre-commit
```

创建 `.pre-commit-config.yaml`：

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=10000']
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

安装 hooks：
```bash
pre-commit install
```

现在 hooks 会在每次 commit 时自动运行。

手动运行：
```bash
pre-commit run --all-files
```

### 服务端 Hooks

**pre-receive**：在服务端强制执行策略
- 拒绝没有工单号的 commit
- 强制执行分支命名
- 防止对 main 分支强制 push

**post-receive**：在 push 后触发操作
- 部署到生产环境
- 发送通知
- 触发 CI/CD

**注意**：服务端 hooks 需要服务器访问权限（在 GitHub/GitLab 免费版上不可用）。请使用 GitHub Actions 或 GitLab CI 代替。

---

## Submodule 和 Subtree

### 什么是 Submodule

Submodule 允许你在一个 Git 仓库中包含另一个 Git 仓库。适用于：
- 共享库
- 供应商依赖
- 公共工具

### 添加 Submodule

```bash
# 添加 submodule
git submodule add https://github.com/org/ml-utils.git lib/ml-utils

# 这会创建：
# - lib/ml-utils/ 目录（submodule）
# - .gitmodules 文件（submodule 配置）

# 提交 submodule
git add .gitmodules lib/ml-utils
git commit -m "Add ml-utils submodule"
```

### 克隆带有 Submodule 的仓库

```bash
# 克隆仓库
git clone https://github.com/org/ml-pipeline.git

# 初始化和更新 submodule
cd ml-pipeline
git submodule init
git submodule update

# 或一步完成：
git clone --recurse-submodules https://github.com/org/ml-pipeline.git
```

### 更新 Submodule

```bash
# 将 submodule 更新到最新 commit
cd lib/ml-utils
git pull origin main

# 返回主仓库
cd ../..
git add lib/ml-utils
git commit -m "Update ml-utils submodule"

# 一次更新所有 submodule
git submodule update --remote --merge
```

### 删除 Submodule

```bash
# 删除 submodule
git submodule deinit lib/ml-utils
git rm lib/ml-utils
rm -rf .git/modules/lib/ml-utils
git commit -m "Remove ml-utils submodule"
```

### Submodule 的挑战

Submodule 功能强大但使用起来有技巧：
- 容易忘记更新
- Detached HEAD 状态令人困惑
- 克隆后需要额外命令
- 协作可能比较困难

**替代方案**：Git subtree（更简单但灵活性较低）。

### Git Subtree

Subtree 将另一个仓库整合到一个子目录中：

```bash
# 添加 subtree
git subtree add --prefix lib/ml-utils https://github.com/org/ml-utils.git main --squash

# Pull 更新
git subtree pull --prefix lib/ml-utils https://github.com/org/ml-utils.git main --squash

# 将更改推回上游
git subtree push --prefix lib/ml-utils https://github.com/org/ml-utils.git main
```

**Subtree 优点：**
- 不需要 `.gitmodules` 文件
- 克隆正常工作
- 对协作者更简单

**Subtree 缺点：**
- 历史可能变得杂乱
- 向上游贡献更改更困难

### ML 项目用例

**场景**：多个 ML 项目共享数据预处理工具。

**选项 1：Submodule**
```
ml-training-pipeline/
├── .gitmodules
├── src/
└── lib/
    └── data-utils/  (submodule: github.com/org/data-utils)

ml-inference-api/
├── .gitmodules
├── src/
└── lib/
    └── data-utils/  (submodule: github.com/org/data-utils)
```

两个项目引用同一个共享库。`data-utils` 中的更新会反映在两个项目中。

**选项 2：Package**
对于 ML 来说更好：发布为 pip 包。

```bash
# 在 data-utils 仓库中
python -m build
twine upload dist/*

# 在其他项目中
pip install data-utils
```

对于 Python 项目来说更易于维护。

---

## Tag 和 Release

### 什么是 Tag

Tag 是对特定 commit 的命名引用。与 branch 不同，tag 不会移动。

**使用场景：**
- 标记发布（v1.0.0、v2.1.3）
- 标记模型版本（model-v1、model-v2）
- 标记重要节点（experiment-baseline）

### 轻量 Tag 与注解 Tag

**轻量 tag**（简单指针）：
```bash
git tag v1.0.0
```

**注解 tag**（带有元数据的完整对象）：
```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

注解 tag 包含：
- 标注者姓名和邮箱
- 日期
- 消息
- GPG 签名（可选）

**发布时始终使用注解 tag。**

### 创建 Tag

```bash
# 创建注解 tag
git tag -a v1.0.0 -m "First production release"

# 为特定 commit 打 tag
git tag -a v0.9.0 abc123 -m "Beta release"

# 签名 tag（使用 GPG 密钥）
git tag -s v1.0.0 -m "Signed release"
```

### 列出 Tag

```bash
# 列出所有 tag
git tag

# 列出匹配模式的 tag
git tag -l "v1.*"
# v1.0.0
# v1.0.1
# v1.1.0

# 显示 tag 详情
git show v1.0.0
```

### 推送 Tag

Tag 不会自动推送：

```bash
# 推送特定 tag
git push origin v1.0.0

# 推送所有 tag
git push origin --tags

# 仅推送注解 tag
git push origin --follow-tags
```

### 删除 Tag

```bash
# 删除本地 tag
git tag -d v1.0.0

# 删除远程 tag
git push origin --delete v1.0.0
```

### 检出 Tag

```bash
# 检出 tag（detached HEAD 状态）
git checkout v1.0.0

# 从 tag 创建分支
git checkout -b hotfix/v1.0.1 v1.0.0
```

### 机器学习项目的语义化版本

使用语义化版本（semver）：`MAJOR.MINOR.PATCH`

**对于模型版本：**
- `MAJOR`：破坏性 API 变更，不同的模型架构
- `MINOR`：新功能，向后兼容的改进
- `PATCH`：Bug 修复，小的调整

示例：
- `model-v1.0.0`：初始生产模型
- `model-v1.1.0`：添加了对新输入特征的支持
- `model-v1.1.1`：修复了预处理 bug
- `model-v2.0.0`：从 ResNet 切换到 Transformer（破坏性变更）

### 创建 GitHub Release

```bash
# 创建 tag
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0
```

在 GitHub 上：
1. 转到 "Releases"
2. 点击 "Draft a new release"
3. 选择 tag v1.0.0
4. 添加 release notes
5. 附上二进制文件（模型文件，如适用）
6. 发布 release

或使用 GitHub CLI：
```bash
gh release create v1.0.0 --notes "Release notes here" model.onnx
```

---

## 高级历史操作

### Git Reflog：你的安全网

Reflog 记录了对 HEAD 的每一次变更：

```bash
git reflog

# 输出：
# abc123 HEAD@{0}: commit: Add feature
# def456 HEAD@{1}: checkout: moving from main to feature
# ghi789 HEAD@{2}: commit: Fix bug
```

Reflog 条目大约 90 天后过期，但可以在灾难中拯救你。

### 恢复已删除的分支

```bash
# 哎呀，删除了包含重要工作的分支
git branch -D important-feature

# 在 reflog 中找到它
git reflog
# abc123 HEAD@{5}: commit: Important feature work

# 重新创建分支
git checkout -b important-feature abc123
```

### 恢复丢失的 Commit

```bash
# 重置到了错误的 commit，丢失了工作
git reset --hard HEAD~10

# 在 reflog 中找到丢失的 commit
git reflog
# abc123 HEAD@{3}: commit: The commit I need

# 从丢失的 commit 创建分支
git branch recovered abc123
```

### Git Bisect：二分查找 Bug

**场景**：100 个 commit 前测试通过，现在失败了。是哪个 commit 导致的？

手动测试需要几个小时。Bisect 使用二分查找自动完成。

```bash
# 启动 bisect
git bisect start

# 将当前 commit 标记为 bad
git bisect bad

# 标记已知正常的 commit
git bisect good abc123

# Git 检出中间的 commit
# 手动或自动测试它
pytest tests/

# 如果测试通过：
git bisect good

# 如果测试失败：
git bisect bad

# Git 继续二分查找
# 大约 log2(n) 步后，找到有问题的 commit

# 找到后重置
git bisect reset
```

自动化 bisect：
```bash
git bisect start HEAD abc123
git bisect run pytest tests/
# Git 自动测试每个 commit，直到找到导致问题的那个
```

### Git Filter-Branch：重写历史

**警告**：功能强大且危险。只有在明确知道自己在做什么时才使用。

**用例**：从整个历史中删除敏感文件。

```bash
# 从所有 commit 中删除文件
git filter-branch --tree-filter 'rm -f secrets.yaml' HEAD

# 从 index 中删除文件
git filter-branch --index-filter 'git rm --cached --ignore-unmatch secrets.yaml' HEAD
```

**更好的替代方案**：使用 BFG Repo-Cleaner（更快、更安全）。

```bash
# 安装 BFG
brew install bfg  # 或下载 JAR

# 删除文件
bfg --delete-files secrets.yaml

# 清理
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

### 重写作者信息

```bash
# 修改最近一次 commit 的作者
git commit --amend --author="New Name <new@email.com>"

# 修改多个 commit 的作者
git rebase -i HEAD~5
# 将 commit 标记为 'edit'
# 对每个 commit：
git commit --amend --author="New Name <new@email.com>"
git rebase --continue
```

---

## Git 在机器学习中的应用

### DVC：数据版本控制

Git 不适合处理大文件。DVC 将 Git 扩展到 ML 领域：

```bash
# 安装 DVC
pip install dvc

# 初始化 DVC
dvc init

# 跟踪大文件
dvc add data/train.csv
git add data/train.csv.dvc .gitignore
git commit -m "Add training data"

# 将数据推送到远程存储（S3、GCS 等）
dvc push

# 克隆并获取数据
git clone repo.git
dvc pull
```

DVC 创建 `.dvc` 文件由 Git 跟踪，而实际数据存储在云存储中。

### MLflow 集成

通过 Git 集成跟踪实验：

```python
import mlflow

# 自动记录 Git 元数据
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.001)
    mlflow.log_metric("accuracy", 0.95)

    # MLflow 记录：
    # - Git commit hash
    # - Git branch
    # - Git repo URL
```

复现实验：
```bash
# 查找实验的 Git commit
mlflow experiments list

# 检出该 commit
git checkout abc123

# 重新运行实验
python train.py
```

### 实验的 Git 工作流

```bash
# 创建实验分支
git checkout -b experiment/resnet-dropout-05

# 修改配置
cat > config.yaml << EOF
model: resnet50
dropout: 0.5
learning_rate: 0.001
EOF

# 运行实验
python train.py --config config.yaml

# 提交结果
git add config.yaml results.json
git commit -m "Experiment: ResNet50 with dropout=0.5, acc=0.89"

# 打 tag 以备将来参考
git tag -a exp-dropout-05 -m "Dropout experiment, accuracy=0.89"

# 如果成功，merge 到 main
git checkout main
git merge experiment/resnet-dropout-05
```

### ML 项目的 .gitattributes

```
# .gitattributes

# 将大文件标记为二进制（不尝试 diff）
*.pkl binary
*.h5 binary
*.pth binary
*.onnx binary

# 无输出地跟踪 notebook（更干净的 diff）
*.ipynb filter=nbstrip_full

# 对大文件使用 LFS
*.pth filter=lfs diff=lfs merge=lfs -text
*.h5 filter=lfs diff=lfs merge=lfs -text
```

### Jupyter Notebook 版本控制

Notebook 是 JSON 格式，难以 diff。解决方案：

**选项 1：在 commit 前剥离输出**

安装 nbstripout：
```bash
pip install nbstripout
nbstripout --install
```

现在 notebook 在 commit 时不包含输出（更干净的 diff）。

**选项 2：ReviewNB（GitHub App）**

在 Pull Request 中提供可视化的 notebook diff。

**选项 3：Jupytext**

将 notebook 转换为 .py 文件：
```bash
pip install jupytext
jupytext --to py notebook.ipynb
```

在 Git 中跟踪 .py 版本。

---

## 性能与优化

### 浅克隆

对于 CI/CD，你不需要完整历史：

```bash
# 仅克隆最新 commit
git clone --depth 1 https://github.com/org/repo.git

# 克隆最近 10 个 commit
git clone --depth 10 https://github.com/org/repo.git
```

**优点**：更快，下载更小
**缺点**：历史有限，无法 push

### 部分克隆

克隆时不包含大文件：

```bash
git clone --filter=blob:limit=1m https://github.com/org/repo.git
# 仅下载 < 1MB 的文件
# 更大的文件按需下载
```

### Git LFS 用于大文件

```bash
# 安装 Git LFS
git lfs install

# 跟踪文件类型
git lfs track "*.pth"
git lfs track "*.h5"

# Git LFS 在仓库中存储指针，实际文件存储在 LFS 存储中
```

### 优化仓库

```bash
# 垃圾回收
git gc --aggressive --prune=now

# 验证仓库完整性
git fsck

# 统计对象数量
git count-objects -vH
```

### Git 性能配置

```bash
# 增加大文件的 HTTP 缓冲区
git config --global http.postBuffer 524288000

# 启用并行 fetch
git config --global fetch.parallel 4

# 启用文件系统缓存（Windows）
git config --global core.fscache true

# 启用 commit graph 以加快操作速度
git config --global core.commitGraph true
git config --global gc.writeCommitGraph true
```

---

## 恢复与灾难管理

### 从 Reset --hard 恢复

```bash
# 意外执行了：
git reset --hard HEAD~5

# 通过 reflog 恢复
git reflog
# 找到 reset 之前的 commit
git reset --hard HEAD@{1}
```

### 从 Rebase 灾难恢复

```bash
# Rebase 出了问题
git rebase --abort  # 如果仍在进行中

# 如果已经完成
git reflog
# 找到 rebase 前的状态
git reset --hard HEAD@{5}
```

### 撤销 Git Push

```bash
# 推送了错误的 commit 到远程

# 选项 1：Revert（安全，创建新的 commit）
git revert abc123
git push origin main

# 选项 2：强制 push（危险！）
git reset --hard HEAD~1
git push --force-with-lease origin main
```

### 从已删除的仓库恢复

如果你有本地克隆：
```bash
# 你的本地克隆就是一个完整的备份
# 重新创建远程仓库
# Push 所有内容
git push --all origin
git push --tags origin
```

如果没有本地克隆：联系 GitHub 支持。

---

## 总结与要点

### 核心概念

1. **交互式 rebase**：在共享之前清理历史
2. **Cherry-pick**：在分支之间应用特定 commit
3. **Stash**：临时保存工作以便切换上下文
4. **Hooks**：在 Git 生命周期节点实现自动化
5. **Tag**：标记重要节点（发布、实验）
6. **Reflog**：恢复丢失工作的安全网

### 必备命令速查表

```bash
# 交互式 rebase
git rebase -i HEAD~3          # 对最近 3 个 commit 进行 rebase
git rebase -i main            # 从分支起点开始 rebase

# Cherry-pick
git cherry-pick abc123        # 应用特定 commit
git cherry-pick abc..def      # 应用一段范围的 commit

# Stash
git stash                     # Stash 更改
git stash pop                 # 应用并移除 stash
git stash list                # 列出 stash

# Tag
git tag -a v1.0.0 -m "msg"   # 创建注解 tag
git push origin v1.0.0        # 推送 tag
git push origin --tags        # 推送所有 tag

# 恢复
git reflog                    # 查看 reflog
git reset --hard HEAD@{1}     # 重置到 reflog 条目

# Bisect
git bisect start              # 启动 bisect
git bisect good/bad           # 标记 commit
git bisect reset              # 结束 bisect
```

### 最佳实践

1. **绝不对共享分支使用 rebase**：只 rebase 本地、未推送的 commit
2. **使用 pre-commit hooks**：在问题进入历史之前捕获它们
3. **使用 tag 标记发布**：始终为生产部署打 tag
4. **保留 reflog**：除非必要，不要运行 `git reflog expire`
5. **谨慎使用 cherry-pick**：尽可能优先使用 merge
6. **经常 stash**：比杂乱的 commit 更好
7. **测试交互式 rebase**：先在测试仓库上练习
8. **为 hooks 编写文档**：添加 README 说明 hooks 的功能
9. **数据使用 DVC**：而不是 Git
10. **签名 tag**：实现已验证的发布

### 给 AI/ML 项目的建议

- **实验分支**：每个实验一个分支，成功的打 tag
- **Pre-commit hooks**：检查大文件、secrets 和代码格式
- **DVC 集成**：正确地对数据和模型进行版本管理
- **MLflow 跟踪**：将 Git 元数据与实验一起记录
- **语义化版本**：正确地为模型发布打 tag
- **Jupyter notebook**：剥离输出或使用 Jupytext
- **Git LFS**：用于必须放在 Git 中的模型权重

### 下一步

至此，Git 模块完成。你现在掌握了：
- Git 基础（第 1 讲）
- Branch 和 merge（第 2 讲）
- 协作（第 3 讲）
- 高级技术（第 4 讲）

**后续步骤：**
1. 完成所有练习来实践这些概念
2. 在你的项目中设置 pre-commit hooks
3. 为开源项目做贡献
4. 在你的团队中实施 Git 工作流

### 延伸阅读

- Pro Git book（全书）：https://git-scm.com/book/en/v2
- Git 文档：https://git-scm.com/doc
- DVC 文档：https://dvc.org/doc
- Pre-commit 框架：https://pre-commit.com/
- 语义化版本：https://semver.org/

---

**练习题**：完成 Exercise 06 和 07，通过动手 ML 项目场景掌握高级 Git 技术。
