# Git 速查表

基本 Git 命令和工作流程的快速参考指南。

---

## 目录

- [安装与配置](#安装与配置)
- [基本命令](#基本命令)
- [分支](#分支)
- [远程仓库](#远程仓库)
- [撤销更改](#撤销更改)
- [查看历史](#查看历史)
- [高级操作](#高级操作)
- [Git 工作流](#git-工作流)
- [故障排查](#故障排查)

---

## 安装与配置

### 初始设置
```bash
# 设置用户名
git config --global user.name "Your Name"

# 设置邮箱
git config --global user.email "your.email@example.com"

# 设置默认编辑器
git config --global core.editor "vim"
git config --global core.editor "code --wait"  # VS Code

# 设置默认分支名
git config --global init.defaultBranch main

# 启用颜色输出
git config --global color.ui auto

# 查看配置
git config --list
git config --global --list
git config user.name
```

### SSH 设置
```bash
# 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your.email@example.com"

# 启动 SSH agent
eval "$(ssh-agent -s)"

# 添加 SSH 密钥
ssh-add ~/.ssh/id_ed25519

# 复制公钥
cat ~/.ssh/id_ed25519.pub
# 添加到 GitHub/GitLab 设置中

# 测试连接
ssh -T git@github.com
```

### 别名
```bash
# 创建快捷方式
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg 'log --oneline --graph --decorate --all'
```

---

## 基本命令

### 仓库初始化
```bash
# 初始化新仓库
git init

# 克隆现有仓库
git clone <repository-url>
git clone <repository-url> <directory-name>

# 克隆特定分支
git clone -b <branch-name> <repository-url>
```

### 状态和信息
```bash
# 检查状态
git status
git status -s  # 简短格式

# 显示更改
git diff  # 未暂存的更改
git diff --staged  # 已暂存的更改
git diff HEAD  # 所有更改

# 显示提交日志
git log
git log --oneline
git log --graph --oneline --all
git log --author="John"
git log --since="2 weeks ago"
git log --until="yesterday"
git log -p  # 附带补丁
git log -3  # 最近 3 个提交
```

### 暂存和提交
```bash
# 暂存文件
git add <file>
git add .  # 所有文件
git add *.py  # 所有 Python 文件
git add -A  # 所有更改包括删除

# 取消暂存文件
git reset <file>
git restore --staged <file>  # 新语法

# 提交更改
git commit -m "提交信息"
git commit -am "信息"  # 添加并提交已跟踪文件
git commit --amend  # 修改上次提交
git commit --amend --no-edit  # 保留提交信息

# 删除文件
git rm <file>  # 从仓库和文件系统删除
git rm --cached <file>  # 仅从仓库删除

# 移动/重命名文件
git mv <旧名> <新名>
```

---

## 分支

### 创建和切换分支
```bash
# 列出分支
git branch  # 本地分支
git branch -r  # 远程分支
git branch -a  # 所有分支

# 创建分支
git branch <branch-name>

# 切换到分支
git checkout <branch-name>
git switch <branch-name>  # 新语法

# 创建并切换
git checkout -b <branch-name>
git switch -c <branch-name>  # 新语法

# 从特定提交创建分支
git branch <branch-name> <commit-hash>

# 删除分支
git branch -d <branch-name>  # 安全删除
git branch -D <branch-name>  # 强制删除

# 重命名分支
git branch -m <旧名> <新名>
git branch -m <新名>  # 重命名当前分支
```

### 合并
```bash
# 将分支合并到当前分支
git merge <branch-name>

# 带提交信息的合并
git merge <branch-name> -m "合并信息"

# 中止合并
git merge --abort

# 合并策略
git merge --no-ff <branch-name>  # 始终创建合并提交
git merge --squash <branch-name>  # 压缩提交
```

### 变基
```bash
# 将当前分支变基到另一个分支
git rebase <branch-name>

# 交互式变基
git rebase -i HEAD~3  # 最近 3 个提交
git rebase -i <commit-hash>

# 解决冲突后继续
git rebase --continue

# 跳过当前提交
git rebase --skip

# 中止变基
git rebase --abort
```

---

## 远程仓库

### 管理远程仓库
```bash
# 列出远程仓库
git remote
git remote -v  # 带 URL

# 添加远程仓库
git remote add origin <repository-url>
git remote add <name> <url>

# 删除远程仓库
git remote remove <name>

# 重命名远程仓库
git remote rename <旧名> <新名>

# 更改远程仓库 URL
git remote set-url origin <new-url>

# 显示远程信息
git remote show origin
```

### 拉取和抓取
```bash
# 从远程抓取（不合并）
git fetch
git fetch origin
git fetch --all  # 所有远程

# 拉取（抓取 + 合并）
git pull
git pull origin main

# 变基方式拉取
git pull --rebase
```

### 推送
```bash
# 推送到远程
git push
git push origin main

# 推送并设置上游
git push -u origin main
git push --set-upstream origin feature-branch

# 推送所有分支
git push --all

# 推送标签
git push --tags

# 强制推送（危险！）
git push --force
git push --force-with-lease  # 更安全的强制推送

# 删除远程分支
git push origin --delete <branch-name>
```

### 跟踪分支
```bash
# 创建跟踪分支
git checkout -b <branch> origin/<branch>
git checkout --track origin/<branch>

# 为现有分支设置上游
git branch --set-upstream-to=origin/main main

# 查看跟踪分支
git branch -vv
```

---

## 撤销更改

### 丢弃更改
```bash
# 丢弃文件的未暂存更改
git checkout -- <file>
git restore <file>  # 新语法

# 丢弃所有未暂存更改
git checkout -- .
git restore .  # 新语法

# 取消暂存文件
git reset HEAD <file>
git restore --staged <file>  # 新语法
```

### 重置
```bash
# 软重置（保留更改在暂存区）
git reset --soft HEAD~1

# 混合重置（保留更改但取消暂存）- 默认
git reset HEAD~1
git reset --mixed HEAD~1

# 硬重置（丢弃所有更改）
git reset --hard HEAD~1
git reset --hard <commit-hash>

# 重置到远程状态
git reset --hard origin/main
```

### 还原
```bash
# 还原提交（创建新提交）
git revert <commit-hash>

# 还原但不提交
git revert -n <commit-hash>

# 还原合并提交
git revert -m 1 <merge-commit-hash>
```

### 清理
```bash
# 删除未跟踪的文件
git clean -n  # 预演
git clean -f  # 强制删除文件
git clean -fd  # 删除文件和目录
git clean -fX  # 删除被忽略的文件
git clean -fx  # 删除所有未跟踪的文件
```

---

## 查看历史

### 日志命令
```bash
# 基本日志
git log
git log --oneline

# 图形化日志
git log --graph --oneline --all
git log --graph --pretty=format:'%h %d %s (%cr) <%an>'

# 按作者筛选
git log --author="John"

# 按日期筛选
git log --since="2 weeks ago"
git log --after="2023-01-01" --before="2023-12-31"

# 按文件筛选
git log <file>
git log --follow <file>  # 跟踪重命名

# 搜索提交
git log --grep="bug fix"
git log -S"function_name"  # 搜索代码

# 显示统计
git log --stat
git log --shortstat
```

### Show 命令
```bash
# 显示提交详情
git show <commit-hash>
git show HEAD
git show HEAD~2

# 显示提交中的特定文件
git show <commit-hash>:<file>

# 显示特定提交时的文件
git show HEAD~3:path/to/file
```

### Diff 命令
```bash
# 显示未暂存的更改
git diff

# 显示已暂存的更改
git diff --staged
git diff --cached

# 比较分支
git diff main..feature
git diff main...feature  # 从共同祖先起

# 比较提交
git diff <commit1> <commit2>
git diff HEAD~2 HEAD

# 比较特定文件
git diff <file>
git diff main..feature <file>
```

### Blame
```bash
# 显示每行是谁修改的
git blame <file>

# 带行号显示
git blame -L 10,20 <file>

# 带提交信息显示
git blame -l <file>
```

---

## 高级操作

### 贮藏
```bash
# 贮藏更改
git stash
git stash save "工作进行中"

# 列出贮藏
git stash list

# 应用贮藏
git stash apply  # 保留贮藏
git stash pop  # 应用并删除贮藏

# 应用特定贮藏
git stash apply stash@{2}

# 删除贮藏
git stash drop stash@{0}

# 清除所有贮藏
git stash clear

# 从贮藏创建分支
git stash branch <branch-name>
```

### 捡选
```bash
# 将特定提交应用到当前分支
git cherry-pick <commit-hash>

# 捡选但不提交
git cherry-pick -n <commit-hash>

# 捡选范围
git cherry-pick <commit1>..<commit2>
```

### 标签
```bash
# 列出标签
git tag
git tag -l "v1.*"

# 创建轻量标签
git tag v1.0.0

# 创建注释标签
git tag -a v1.0.0 -m "版本 1.0.0"

# 标记特定提交
git tag v1.0.0 <commit-hash>

# 推送标签
git push origin v1.0.0
git push origin --tags  # 所有标签

# 删除标签
git tag -d v1.0.0  # 本地
git push origin --delete v1.0.0  # 远程

# 检出标签
git checkout v1.0.0
```

### 子模块
```bash
# 添加子模块
git submodule add <repository-url> <path>

# 初始化子模块
git submodule init

# 更新子模块
git submodule update
git submodule update --remote

# 带子模块克隆
git clone --recursive <repository-url>

# 更新所有子模块
git submodule foreach git pull origin main
```

---

## Git 工作流

### 功能分支工作流
```bash
# 1. 创建功能分支
git checkout -b feature/new-feature

# 2. 进行更改并提交
git add .
git commit -m "添加新功能"

# 3. 推送到远程
git push -u origin feature/new-feature

# 4. 创建合并请求（在 GitHub/GitLab 上）

# 5. PR 批准后，合并并删除分支
git checkout main
git pull origin main
git branch -d feature/new-feature
```

### Gitflow 工作流
```bash
# 主要分支：main, develop

# 开始功能
git checkout develop
git checkout -b feature/new-feature

# 完成功能
git checkout develop
git merge --no-ff feature/new-feature
git branch -d feature/new-feature
git push origin develop

# 开始发布
git checkout -b release/1.0.0 develop
# 进行发布提交
git checkout main
git merge --no-ff release/1.0.0
git tag -a v1.0.0
git checkout develop
git merge --no-ff release/1.0.0
git branch -d release/1.0.0

# 热修复
git checkout -b hotfix/1.0.1 main
# 修复 bug
git checkout main
git merge --no-ff hotfix/1.0.1
git tag -a v1.0.1
git checkout develop
git merge --no-ff hotfix/1.0.1
git branch -d hotfix/1.0.1
```

### 变基工作流
```bash
# 保持功能分支与 main 同步
git checkout feature/my-feature
git fetch origin
git rebase origin/main

# 如有冲突则解决
# ... 编辑文件 ...
git add <已解决的文件>
git rebase --continue

# 强制推送（重写历史）
git push --force-with-lease origin feature/my-feature
```

---

## 故障排查

### 常见问题

#### 合并冲突
```bash
# 1. 查看冲突文件
git status

# 2. 编辑文件解决冲突
# 查找冲突标记：<<<<<<<, =======, >>>>>>>

# 3. 标记为已解决
git add <已解决的文件>

# 4. 完成合并
git commit  # 用于合并
git rebase --continue  # 用于变基
```

#### 撤销上次提交
```bash
# 保留更改在暂存区
git reset --soft HEAD~1

# 保留更改但取消暂存
git reset HEAD~1

# 丢弃更改
git reset --hard HEAD~1
```

#### 恢复已删除的分支
```bash
# 查找提交 hash
git reflog

# 重建分支
git branch <branch-name> <commit-hash>
```

#### 删除已提交的文件
```bash
# 从历史中删除（危险！）
git filter-branch --tree-filter 'rm -f <file>' HEAD

# 更好的方案：使用 git filter-repo（单独安装）
git filter-repo --path <file> --invert-paths
```

#### 修改提交信息
```bash
# 上次提交
git commit --amend -m "新信息"

# 更早的提交（交互式变基）
git rebase -i HEAD~3
# 将要编辑的提交的 'pick' 改为 'reword'
```

#### 同步 fork 与上游
```bash
# 添加上游远程仓库
git remote add upstream <原始仓库-url>

# 抓取上游
git fetch upstream

# 合并上游到本地
git checkout main
git merge upstream/main

# 推送到你的 fork
git push origin main
```

---

## 最佳实践

### 提交信息
```bash
# 好的提交信息格式：
# <type>(<scope>): <subject>
#
# <body>
#
# <footer>

# 示例：
git commit -m "feat(api): 添加用户认证接口"
git commit -m "fix(database): 解决连接超时问题"
git commit -m "docs(readme): 更新安装说明"

# 类型：feat, fix, docs, style, refactor, test, chore
```

### .gitignore
```bash
# Python
__pycache__/
*.py[cod]
*.so
.env
venv/
.venv/

# IDE
.vscode/
.idea/
*.swp

# 操作系统
.DS_Store
Thumbs.db

# 项目特定
/data/
/logs/
*.log
```

### 技巧
1. **频繁提交**：小而有针对性的提交
2. **推送前先拉取**：避免冲突
3. **使用分支**：保持 main 稳定
4. **写好的提交信息**：解释为什么，而不是什么
5. **提交前审查**：`git diff --staged`
6. **不要提交密钥**：使用 .gitignore
7. **尽早建立 .gitignore**：在第一次提交之前

---

## 快速参考

### 基本命令
```bash
# 克隆
git clone <url>

# 状态
git status

# 添加
git add <file>
git add .

# 提交
git commit -m "信息"

# 推送
git push origin main

# 拉取
git pull origin main

# 分支
git branch <name>
git checkout <name>
git checkout -b <name>

# 合并
git merge <branch>

# 日志
git log --oneline
```

### 有用的别名
```bash
# 添加到 ~/.gitconfig 或使用 git config --global alias.* 命令

[alias]
    st = status -s
    co = checkout
    br = branch
    ci = commit
    unstage = reset HEAD --
    last = log -1 HEAD
    lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
    undo = reset --soft HEAD~1
    amend = commit --amend --no-edit
```

---

## 额外资源

- [Official Git Documentation](https://git-scm.com/doc)
- [Pro Git Book](https://git-scm.com/book/en/v2)（免费在线阅读）
- [GitHub Git Cheat Sheet](https://training.github.com/downloads/github-git-cheat-sheet/)
- [Learn Git Branching](https://learngitbranching.js.org/)（交互式教程）
- [Oh Shit, Git!?!](https://ohshitgit.com/)（修复错误）

---

**掌握这些命令以进行有效的版本控制！**
