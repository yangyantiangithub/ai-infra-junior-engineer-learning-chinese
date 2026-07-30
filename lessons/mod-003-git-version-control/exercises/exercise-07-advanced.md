# 练习 07：高级 Git 技术

## 概述

**难度**：高级
**预计时间**：120-150 分钟
**前置条件**：练习 01-06，讲座 04 - 高级 Git

在本练习中，你将掌握高级 Git 技术，包括交互式 rebasing、Git hooks、cherry-picking、高级 stashing 工作流、bisecting 查找 bug、子模块和恢复技术。这些技能对于复杂的 ML 基础设施项目至关重要。

## 学习目标

完成本练习后，你将：

- 使用交互式 rebase 清理 commit 历史
- 实现自定义 Git hooks 以实现自动化
- 跨分支 cherry-pick 特定 commit
- 掌握 stashing 工作流
- 使用 git bisect 查找引入 bug 的 commit
- 使用 Git 子模块
- 从 Git 错误中恢复
- 使用 Reflog 进行时间旅行
- 高级合并策略

## 场景

你正在维护一个拥有复杂历史的大型 ML 基础设施代码库。你需要：
- 在合并到 main 之前清理混乱的 commit 历史
- 使用 hooks 自动化代码质量检查
- 从一个分支选择性应用修复到另一个分支
- 查找性能退化是何时引入的
- 将第三方依赖管理为子模块
- 从意外的数据丢失中恢复

## 前置条件

```bash
# 为高级技术创建项目
mkdir ml-platform-advanced
cd ml-platform-advanced
git init
git config init.defaultBranch main

# 创建初始结构
mkdir -p src/{api,models,pipeline} tests scripts
touch README.md
git add .
git commit -m "Initial commit"
```

---

## 第一部分：交互式 Rebase

### 任务 1.1：清理 Commit 历史

使用交互式 rebase 重新组织 commit。

**说明：**

```bash
# 创建一个混乱的功能分支
git switch -c feature/model-serving

# 制作几个小 commit（模拟混乱的开发过程）
echo "class ModelServer:" > src/api/server.py
git add src/api/server.py
git commit -m "wip"

echo "    def __init__(self):" >> src/api/server.py
git add src/api/server.py
git commit -m "add init"

echo "        self.model = None" >> src/api/server.py
git add src/api/server.py
git commit -m "add model attr"

echo "    def load_model(self, path):" >> src/api/server.py
git add src/api/server.py
git commit -m "load method"

echo "        self.model = load(path)" >> src/api/server.py
git add src/api/server.py
git commit -m "fix: typo in load"

echo "    def predict(self, data):" >> src/api/server.py
git add src/api/server.py
git commit -m "predict method"

# 查看混乱的历史
git log --oneline -6

# 使用交互式 rebase 清理
git rebase -i HEAD~5

# 编辑器打开，显示：
# pick abc123 wip
# pick def456 add init
# pick ghi789 add model attr
# pick jkl012 load method
# pick mno345 fix: typo in load
# pick pqr678 predict method

# 改为：
# pick abc123 wip
# squash def456 add init
# squash ghi789 add model attr
# squash jkl012 load method
# fixup mno345 fix: typo in load
# pick pqr678 predict method

# 保存并关闭。编辑器再次打开用于 commit 消息：
# 改为：
# "feat(api): add model serving infrastructure
#
# Implement ModelServer class with:
# - Model loading from path
# - Prediction interface
#
# Foundation for ML model deployment."

# 查看清理后的历史
git log --oneline -3
```

**Rebase 命令：**
- `pick` - 保持 commit 原样
- `reword` - 保留变更，编辑消息
- `edit` - 暂停以修改 commit
- `squash` - 与前一个合并，保留两个消息
- `fixup` - 与前一个合并，丢弃此消息
- `drop` - 完全移除 commit

---

### 任务 1.2：重新排序 Commit

按逻辑顺序重新组织 commit。

**说明：**

```bash
# 创建顺序错误的 commit
echo "# Tests" > tests/test_server.py
git add tests/test_server.py
git commit -m "test: add server tests"

echo "# Documentation" > docs/server.md
git add docs/server.md
git commit -m "docs: document server API"

# 哦，应该在测试之前添加文档
git rebase -i HEAD~2

# 重新排序：
# pick <hash> docs: document server API
# pick <hash> test: add server tests

# 保存 - commit 现在按逻辑顺序排列
git log --oneline -4
```

---

### 任务 1.3：拆分大 Commit

将一个 commit 拆分为多个逻辑 commit。

**说明：**

```bash
# 制作包含多个变更的大 commit
cat >> src/api/server.py << 'EOF'

    def health_check(self):
        """Check server health."""
        return {"status": "healthy"}

    def metrics(self):
        """Get server metrics."""
        return {"requests": 0}
EOF

cat > src/api/config.py << 'EOF'
"""Server configuration."""

CONFIG = {
    "port": 8000,
    "host": "0.0.0.0"
}
EOF

git add src/api/
git commit -m "Add features"

# 哦，一个 commit 里太多了。拆分它：
git rebase -i HEAD~1

# 将"pick"改为"edit"
# Git 会在该 commit 处暂停

# 重置 commit 但保留变更
git reset HEAD^

# 现在分别提交各部分
git add src/api/server.py
git commit -m "feat(api): add health check and metrics endpoints"

git add src/api/config.py
git commit -m "feat(config): add server configuration module"

# 继续 rebase
git rebase --continue

# 查看拆分后的 commit
git log --oneline -3
```

---

## 第二部分：Git Hooks

### 任务 2.1：Pre-Commit Hook

创建 hook 在提交前运行检查。

**说明：**

```bash
# 创建 pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
# Pre-commit hook: Run code quality checks

echo "Running pre-commit checks..."

# Check for Python syntax errors
echo "1. Checking Python syntax..."
python_files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -n "$python_files" ]; then
    for file in $python_files; do
        python -m py_compile "$file"
        if [ $? -ne 0 ]; then
            echo "❌ Syntax error in $file"
            exit 1
        fi
    done
    echo "✅ Python syntax OK"
fi

# Check for debug statements
echo "2. Checking for debug statements..."
if git diff --cached | grep -i "print(\|pdb\|breakpoint("; then
    echo "❌ Found debug statements. Remove before committing:"
    git diff --cached | grep -n -i "print(\|pdb\|breakpoint("
    echo ""
    echo "To commit anyway: git commit --no-verify"
    exit 1
fi
echo "✅ No debug statements found"

# Check for large files
echo "3. Checking file sizes..."
max_size=1048576  # 1 MB
for file in $(git diff --cached --name-only --diff-filter=ACM); do
    if [ -f "$file" ]; then
        size=$(wc -c < "$file")
        if [ $size -gt $max_size ]; then
            echo "❌ File too large: $file ($size bytes > $max_size bytes)"
            echo "Consider using Git LFS for large files"
            exit 1
        fi
    fi
done
echo "✅ File sizes OK"

# Check for AWS keys (security)
echo "4. Checking for secrets..."
if git diff --cached | grep -E "AKIA[0-9A-Z]{16}"; then
    echo "❌ Possible AWS key detected!"
    exit 1
fi
echo "✅ No secrets detected"

echo ""
echo "✅ All pre-commit checks passed!"
exit 0
EOF

chmod +x .git/hooks/pre-commit

# 测试 hook
echo "print('debug')" > src/test.py
git add src/test.py
git commit -m "test commit"

# Hook 应该阻止它！

# 移除调试语句
echo "# No debug" > src/test.py
git add src/test.py
git commit -m "feat: add test module"

# 应该成功
```

---

### 任务 2.2：Post-Commit Hook

在成功提交后运行操作。

**说明：**

```bash
cat > .git/hooks/post-commit << 'EOF'
#!/bin/bash
# Post-commit hook: Log commits

commit_hash=$(git rev-parse HEAD)
commit_msg=$(git log -1 --pretty=%B)
timestamp=$(date +"%Y-%m-%d %H:%M:%S")

echo "[$timestamp] $commit_hash: $commit_msg" >> .git/commit-log.txt

# Notify about commit
echo "✅ Commit logged: $commit_hash"
EOF

chmod +x .git/hooks/post-commit

# 测试它
echo "# Update" >> README.md
git add README.md
git commit -m "docs: update README"

# 检查日志
cat .git/commit-log.txt
```

---

### 任务 2.3：Pre-Push Hook

在推送到远程之前进行验证。

**说明：**

```bash
cat > .git/hooks/pre-push << 'EOF'
#!/bin/bash
# Pre-push hook: Run tests before push

echo "Running pre-push checks..."

# Run tests
echo "1. Running tests..."
# pytest tests/ -v
echo "✅ Tests passed (simulated)"

# Check branch name
current_branch=$(git branch --show-current)
if [ "$current_branch" = "main" ]; then
    echo "⚠️  Pushing directly to main!"
    read -p "Are you sure? (y/N) " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        echo "❌ Push aborted"
        exit 1
    fi
fi

# Check commit messages
echo "2. Validating commit messages..."
commits=$(git log @{u}.. --pretty=format:"%s")
while read -r msg; do
    if ! echo "$msg" | grep -E "^(feat|fix|docs|test|refactor|chore)(\(.+\))?: .+"; then
        echo "❌ Invalid commit message format: $msg"
        echo "Use: type(scope): message"
        echo "Example: feat(api): add new endpoint"
        exit 1
    fi
done <<< "$commits"
echo "✅ Commit messages valid"

echo ""
echo "✅ All pre-push checks passed!"
exit 0
EOF

chmod +x .git/hooks/pre-push

# 测试（模拟 - 没有实际远程）
# git push origin feature-branch
```

---

## 第三部分：Cherry-Picking

### 任务 3.1：应用特定 Commit

选择性将 commit 应用到另一个分支。

**说明：**

```bash
# 在 main 上创建热修复
git switch main
git switch -c hotfix/critical-bug

echo "def fix_critical_bug():" > src/api/fixes.py
echo "    return 'fixed'" >> src/api/fixes.py
git add src/api/fixes.py
git commit -m "fix: critical security vulnerability in auth"

echo "def another_fix():" >> src/api/fixes.py
git add src/api/fixes.py
git commit -m "fix: memory leak in model loading"

# 返回到功能分支
git switch feature/model-serving

# 只 cherry-pick 安全修复
git log hotfix/critical-bug --oneline
# 记下安全修复的 commit 哈希

git cherry-pick <security-fix-hash>

# 现在功能分支有了安全修复，没有其他变更

# 查看历史
git log --oneline --graph --all -10
```

---

### 任务 3.2：Cherry-Pick 多个 Commit

挑选一系列 commit。

**说明：**

```bash
# Cherry-pick 多个 commit
git cherry-pick <hash1> <hash2> <hash3>

# 或 cherry-pick 一个范围
git cherry-pick <start-hash>..<end-hash>

# 如果出现冲突：
# 1. 解决冲突
# 2. git add <resolved-files>
# 3. git cherry-pick --continue

# 要中止：
# git cherry-pick --abort
```

---

## 第四部分：高级 Stashing 工作流

### 任务 4.1：带消息的 Stash

使用描述保存工作进度。

**说明：**

```bash
# 进行更改
echo "# WIP feature" >> src/api/server.py

# 带描述暂存
git stash push -m "WIP: new caching feature for predictions"

# 列出 stash
git stash list

# 输出：
# stash@{0}: On feature/model-serving: WIP: new caching feature

# 稍后应用
git stash pop

# 或者应用而不从 stash 中删除
# git stash apply stash@{0}
```

---

### 任务 4.2：部分 Stashing

只暂存特定文件。

**说明：**

```bash
# 对多个文件进行更改
echo "# Change 1" >> src/api/server.py
echo "# Change 2" >> src/models/model.py
echo "# Change 3" >> tests/test_api.py

# 只暂存 server.py
git stash push -m "server changes" src/api/server.py

# 其他文件仍有变更
git status

# 应用 stash
git stash pop
```

---

### 任务 4.3：从 Stash 创建分支

将暂存的工作转换为分支。

**说明：**

```bash
# 暂存一些工作
echo "# Experimental feature" >> src/api/server.py
git stash

# 从 stash 创建分支
git stash branch experiment/new-feature

# 现在在新分支上，stash 的变更已应用
git status
```

---

## 第五部分：Git Bisect

### 任务 5.1：使用 Bisect 查找 Bug

使用二分查找找到 bug 的引入。

**说明：**

```bash
# 创建带有 bug 的历史
git switch main
git switch -c debug/find-bug

# 制作好的 commit
for i in {1..10}; do
    echo "version $i" >> src/models/model.py
    git add src/models/model.py
    git commit -m "Update model v$i"
done

# 在 commit 6 中引入 bug
git checkout HEAD~4
echo "BUGGY_CODE = True" >> src/models/model.py
git add src/models/model.py
git commit -m "Update model v6"

# 继续添加更多 commit
for i in {7..10}; do
    echo "version $i" >> src/models/model.py
    git add src/models/model.py
    git commit -m "Update model v$i"
done

# 现在使用 bisect 查找 bug 是何时引入的
git bisect start

# 标记当前为 bad
git bisect bad

# 标记较早的 commit 为 good（在 bug 之前）
git bisect good HEAD~10

# Git 检出中间的 commit
# 测试它（手动或使用脚本）
cat src/models/model.py | grep "BUGGY_CODE"

# 如果 bug 存在：
git bisect bad

# 如果没有 bug：
# git bisect good

# Git 继续二分查找
# 重复直到 Git 找到精确的 commit

# 完成后重置
git bisect reset
```

---

### 任务 5.2：使用脚本自动化 Bisect

让 Git 自动执行 bisect 过程。

**说明：**

```bash
# 创建测试脚本
cat > scripts/test_for_bug.sh << 'EOF'
#!/bin/bash
# Return 0 if good, 1 if bad

if grep -q "BUGGY_CODE" src/models/model.py; then
    echo "Bug found!"
    exit 1
else
    echo "No bug"
    exit 0
fi
EOF

chmod +x scripts/test_for_bug.sh

# 运行自动 bisect
git bisect start HEAD HEAD~10
git bisect run scripts/test_for_bug.sh

# Git 自动找到有问题的 commit！
```

---

## 第六部分：Git Reflog 与恢复

### 任务 6.1：恢复丢失的 Commit

使用 reflog 查找和恢复 commit。

**说明：**

```bash
# 制作几个 commit
echo "important work" > important.txt
git add important.txt
git commit -m "Important work"

# 记下 commit 哈希
git log --oneline -1

# 意外重置（丢失 commit）
git reset --hard HEAD~1

# Commit 消失了！
git log --oneline -1

# 使用 reflog 找到它
git reflog

# 输出显示：
# abc123 HEAD@{0}: reset: moving to HEAD~1
# def456 HEAD@{1}: commit: Important work

# 恢复 commit
git checkout HEAD@{1}

# 或者在该点创建分支
git branch recovery HEAD@{1}
git switch recovery

# 你的工作恢复了！
```

---

### 任务 6.2：撤销 Rebase

从糟糕的 rebase 中恢复。

**说明：**

```bash
# Rebase 之前
git log --oneline -5

# 执行 rebase
git rebase -i HEAD~3
# 进行更改并完成

# 哦，搞砸了 rebase！

# 在 reflog 中找到 rebase 前的状态
git reflog

# 查找："rebase -i (start)"
# 之前的条目是 rebase 前的状态

# 重置到 rebase 前
git reset --hard HEAD@{5}  # 根据 reflog 调整编号

# Rebase 已撤销！
```

---

## 第七部分：Git 子模块

### 任务 7.1：添加子模块

将外部依赖管理为子模块。

**说明：**

```bash
# 添加子模块（外部库）
# git submodule add https://github.com/user/ml-utils.git libs/ml-utils

# 本练习模拟：
mkdir -p libs
cd libs
mkdir ml-utils
cd ml-utils
git init
echo "# ML Utilities" > README.md
git add README.md
git commit -m "Initial commit"
cd ../..

# 注册为子模块
git submodule add ./libs/ml-utils libs/ml-utils

# 创建了 .gitmodules 文件
cat .gitmodules

git add .gitmodules libs/ml-utils
git commit -m "chore: add ml-utils submodule"
```

---

### 任务 7.2：更新子模块

更新到子模块的最新版本。

**说明：**

```bash
# 更新子模块
cd libs/ml-utils
# git pull origin main  # 如果有真实的远程

# 更新到特定 commit
git checkout <commit-hash>
cd ../..

# 提交子模块更新
git add libs/ml-utils
git commit -m "chore: update ml-utils submodule to v2.0"
```

---

### 任务 7.3：克隆包含子模块的仓库

**说明：**

```bash
# 克隆包含子模块的仓库时：
# git clone --recursive <repo-url>

# 如果已经克隆：
# git submodule init
# git submodule update

# 或者一步完成：
# git submodule update --init --recursive
```

---

## 第八部分：高级合并策略

### 任务 8.1：八爪鱼合并

一次合并多个分支。

**说明：**

```bash
# 创建多个功能分支
git switch -c feature/a
echo "Feature A" > feature-a.txt
git add feature-a.txt
git commit -m "feat: add feature A"

git switch main
git switch -c feature/b
echo "Feature B" > feature-b.txt
git add feature-b.txt
git commit -m "feat: add feature B"

git switch main
git switch -c feature/c
echo "Feature C" > feature-c.txt
git add feature-c.txt
git commit -m "feat: add feature C"

# 一次性合并所有（如果没有冲突）
git switch main
git merge feature/a feature/b feature/c -m "Merge features A, B, and C"

# 三个都在一个 commit 中合并了！
git log --oneline --graph -5
```

---

### 任务 8.2：Ours/Theirs 策略

自动解决冲突。

**说明：**

```bash
# 合并时，使用策略优先选择一方

# 始终优先选择当前分支（ours）
# git merge -X ours feature-branch

# 始终优先选择传入分支（theirs）
# git merge -X theirs feature-branch

# 适用于某个版本肯定正确的生成文件
```

---

## 第九部分：Git Worktrees

### 任务 9.1：多个工作目录

同时在多个分支上工作。

**说明：**

```bash
# 为另一个分支添加 worktree
git worktree add ../ml-platform-feature feature/model-serving

# 现在你有两个工作目录：
# - ml-platform-advanced/（main 分支）
# - ml-platform-feature/（feature/model-serving 分支）

# 同时在两个中工作
cd ../ml-platform-feature
# 进行更改

cd ../ml-platform-advanced
# 不同的分支，不同的变更

# 列出 worktrees
git worktree list

# 完成后移除 worktree
git worktree remove ../ml-platform-feature
```

---

## 第十部分：故障排除与恢复

### 任务 10.1：修复 Detached HEAD

从 detached HEAD 状态恢复。

**说明：**

```bash
# 意外检出了 commit 哈希
git checkout HEAD~5

# 你处于 detached HEAD 状态！
# 制作几个 commit
echo "work" > work.txt
git add work.txt
git commit -m "Work in detached HEAD"

# 创建分支以保存工作
git branch temp-work

# 切换到它
git switch temp-work

# 如果需要，合并到 main
git switch main
git merge temp-work
```

---

### 任务 10.2：从历史中移除敏感数据

清理已提交的密钥。

**说明：**

```bash
# 如果你提交了密钥：
# 使用：git filter-branch 或 BFG Repo-Cleaner

# 使用 filter-branch 的示例：
# git filter-branch --force --index-filter \
#   "git rm --cached --ignore-unmatch path/to/secret.key" \
#   --prune-empty --tag-name-filter cat -- --all

# 然后强制推送：
# git push origin --force --all
# git push origin --force --tags

# 警告：会重写历史！与团队协调。
```

---

## 验证

### 验证你的技能

```bash
# 检查你的 Git 专业知识
git log --oneline --graph --all -20
git reflog -10
git stash list
git worktree list

# 验证 hooks
ls -la .git/hooks/

# 测试恢复技能
# git reflog
# git fsck --lost-found
```

---

## 挑战任务

### 挑战 1：为复杂命令创建别名

```bash
git config alias.lol "log --graph --decorate --oneline --all"
git lol
```

### 挑战 2：Bisect 性能退化

使用 bisect 查找推理延迟增加的时间点。

### 挑战 3：交互式 Rebase 大师

将 20 个 commit 变基为逻辑组。

---

## 故障排除

**问题**："无法解决的 rebase 冲突"
```bash
git rebase --abort
# 尝试不同的方法或改为合并
```

**问题**："丢失了重要的工作"
```bash
git reflog
git fsck --lost-found
```

**问题**："提交到了错误的分支"
```bash
git log --oneline -1  # 记下 commit 哈希
git reset --hard HEAD~1  # 从当前分支移除
git checkout correct-branch
git cherry-pick <hash>  # 添加到正确的分支
```

---

## 反思问题

1. **何时应该使用交互式 rebase vs. merge？**
2. **重写历史有什么危险？**
3. **bisect 如何帮助调试复杂问题？**
4. **何时使用子模块 vs. 包管理器？**
5. **reset、revert 和 checkout 之间的区别是什么？**

---

## 总结

恭喜！你已经掌握了高级 Git 技术：

- ✅ 交互式 rebasing 用于整洁历史
- ✅ 自定义 Git hooks 用于自动化
- ✅ Cherry-picking 特定变更
- ✅ 高级 stashing 工作流
- ✅ Bisecting 查找 bug
- ✅ Reflog 用于恢复
- ✅ Git 子模块
- ✅ 高级合并策略
- ✅ Worktrees 用于并行工作
- ✅ 故障排除与恢复

### 学会的关键命令

```bash
git rebase -i HEAD~N         # 交互式 rebase
git cherry-pick <hash>       # 应用特定 commit
git stash push -m "msg"      # 带消息的 stash
git bisect start             # 二分查找查找 bug
git reflog                   # 查看引用日志
git submodule add <url>      # 添加子模块
git worktree add <path>      # 创建 worktree
git filter-branch            # 重写历史
```

### 下一步

- 在真实项目中应用这些技术
- 完成模块 003 测验
- 练习高级工作流

**出色的工作！** 你现在是 Git 专家，随时可以应对复杂的 ML 基础设施项目。
