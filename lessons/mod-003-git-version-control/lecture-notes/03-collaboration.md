# Lecture 03: 使用 Git 协作

## Table of Contents
1. [引言](#introduction)
2. [远程仓库](#remote-repositories)
3. [GitHub 和 GitLab 基础](#github-and-gitlab-fundamentals)
4. [克隆与 Fork](#cloning-and-forking)
5. [Push 和 Pull 操作](#push-and-pull-operations)
6. [Pull Request 和代码审查](#pull-requests-and-code-review)
7. [Issue 与项目管理](#issues-and-project-management)
8. [协作工作流](#collaborative-workflows)
9. [团队协作最佳实践](#team-collaboration-best-practices)
10. [常见问题与排错](#common-issues-and-troubleshooting)
11. [总结与要点](#summary-and-key-takeaways)

---

## 引言

### 为什么协作工具很重要

你已经在本地掌握了 Git——commit、branch 和 merge。但 AI 基础设施从来不是一个人的事情。你将与以下角色协作：
- **数据科学家**——需要你的基础设施来进行实验
- **机器学习工程师**——使用你的 pipeline 部署模型
- **DevOps 工程师**——管理生产部署
- **团队中的其他基础设施工程师**
- **开源贡献者**——改进共享工具

没有完善的协作工作流，混乱就会随之而来：
- 冲突的更改相互覆盖
- 代码审查无法进行
- 没有人知道谁改了什么、为什么改
- 部署未经批准就进行
- 知识只存在于个人头脑中

本讲将把你从一个单打独斗的 Git 用户转变为一个高效的团队协作者。

### 学习目标

学完本讲后，你将：

- 理解远程仓库及其如何实现协作
- 配置和管理远程连接
- Push 你的工作与他人分享，Pull 他人的工作
- 创建和审查 Pull Request (PR) / Merge Request (MR)
- 对基础设施代码实施有效的代码审查实践
- 使用 Issue 进行 bug 跟踪和功能规划
- 为开源项目做贡献
- 应用 GitHub Actions / GitLab CI 实现自动化
- 通过规则和策略保护关键分支
- 系统性地解决协作冲突

### AI 基础设施背景

在 AI 基础设施中，协作涉及：

**代码共享：**
- 基础设施即代码（Terraform、Kubernetes 清单）
- 训练脚本和 pipeline
- 模型服务 API
- 监控和日志配置

**知识共享：**
- 架构决策
- 部署流程
- 排错指南
- 实验结果

**质量控制：**
- 同行代码审查
- 自动化测试（CI/CD）
- 安全扫描
- 性能基准测试

**项目协调：**
- 功能规划
- Bug 跟踪
- 发布管理
- 轮值 On-call

---

## 远程仓库

### 什么是远程仓库

远程仓库是你项目托管在互联网或网络上的一个版本。它是每个人同步的共享副本。

```
Developer A's     Remote Repo        Developer B's
Local Repo        (GitHub)          Local Repo
    ↓                ↓                   ↓
  [Repo]  ←push→  [Repo]  ←pull→     [Repo]
           ←pull→          ←push→
```

每个人都把自己的更改 push 到远程仓库，并从远程仓库 pull 他人的更改。

### 常见的远程托管服务

1. **GitHub**（最流行）
   - 公有和私有仓库免费
   - 强大的社区和生态系统
   - GitHub Actions 用于 CI/CD
   - 最适合开源

2. **GitLab**（强大的 CI/CD）
   - 自托管或云服务
   - 内置 CI/CD pipeline
   - DevOps 重点功能
   - 在企业中流行

3. **Bitbucket**（Atlassian 生态系统）
   - 与 Jira 集成
   - 小团队免费
   - 适合使用 Atlassian 工具的企业

4. **自托管**（完全控制）
   - GitLab CE（社区版）
   - Gitea（轻量级）
   - 自定义方案

本课程中我们将重点使用 GitHub，但概念适用于所有平台。

### 查看远程仓库

```bash
# 列出已配置的远程仓库
git remote

# 输出：
# origin

# 显示远程 URL
git remote -v

# 输出：
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)
```

`origin` 是你克隆来源的远程仓库的默认名称。

### 添加远程仓库

```bash
# 添加新的远程仓库
git remote add <name> <url>

# 示例：添加 origin
git remote add origin https://github.com/user/ml-pipeline.git

# 添加额外的远程仓库（例如为 fork 做贡献）
git remote add upstream https://github.com/original-owner/ml-pipeline.git
```

### 远程命名惯例

常见的远程名称：
- **origin**：你的主要远程仓库（你的 fork 或主仓库）
- **upstream**：原始仓库（当你 fork 之后）
- **production**：生产环境部署的远程仓库
- **staging**：预发布环境的远程仓库

### 修改和删除远程仓库

```bash
# 修改远程 URL（例如从 HTTPS 切换到 SSH）
git remote set-url origin git@github.com:user/repo.git

# 重命名远程仓库
git remote rename old-name new-name

# 删除远程仓库
git remote remove origin

# 显示远程仓库的详细信息
git remote show origin
```

### HTTPS 与 SSH 认证

**HTTPS**（更简单但需要凭据）：
```bash
git remote add origin https://github.com/user/repo.git
# 提示输入用户名/密码或 token
```

**SSH**（更安全，无需重复认证）：
```bash
git remote add origin git@github.com:user/repo.git
# 使用 SSH 密钥，设置后无需再输密码
```

设置 SSH 密钥（一次性操作）：
```bash
# 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your_email@example.com"

# 复制公钥
cat ~/.ssh/id_ed25519.pub

# 添加到 GitHub：Settings → SSH and GPG keys → New SSH key
```

### 远程分支

远程分支是对远程仓库中分支状态的引用：

```bash
# 列出所有分支（本地和远程）
git branch -a

# 输出：
# * main
#   feature/new-model
#   remotes/origin/main
#   remotes/origin/develop
#   remotes/origin/feature/api-v2
```

`remotes/origin/main` 是一个远程跟踪分支——对远程分支的本地引用。

---

## GitHub 和 GitLab 基础

### 在 GitHub 上创建仓库

**通过网页界面：**
1. 点击 "+" → "New repository"
2. 输入仓库名称：`ml-inference-api`
3. 选择公开或私有
4. 可选项：添加 README、.gitignore、许可证
5. 点击 "Create repository"

**连接本地仓库：**
```bash
# 如果你已有本地仓库
git remote add origin https://github.com/user/ml-inference-api.git
git branch -M main
git push -u origin main
```

**从零开始：**
```bash
# 克隆新的（空）仓库
git clone https://github.com/user/ml-inference-api.git
cd ml-inference-api

# 添加文件
echo "# ML Inference API" > README.md
git add README.md
git commit -m "Initial commit"
git push origin main
```

### 仓库设置

需要配置的关键设置：

**常规：**
- 仓库名称和描述
- 默认分支（main vs master）
- 功能（Issues、Wiki、Projects）

**协作者：**
- 为团队成员分配不同权限
- 权限：Read、Write、Maintain、Admin

**分支：**
- 分支保护规则
- 必需的状态检查（CI/CD）
- merge 前必须的审查
- 限制谁可以 push

**Secrets：**
- 存储 CI/CD 的敏感数据
- API 密钥、凭据、token
- 加密存储，创建后不可见

### 理解 .github 目录

`.github` 目录存放 GitHub 特定的配置：

```
.github/
├── workflows/              # GitHub Actions (CI/CD)
│   ├── test.yml
│   └── deploy.yml
├── ISSUE_TEMPLATE/         # Issue 模板
│   ├── bug_report.md
│   └── feature_request.md
├── PULL_REQUEST_TEMPLATE.md  # PR 模板
├── CODEOWNERS              # 自动分配 PR 审查人
└── dependabot.yml          # 依赖更新
```

### GitHub vs GitLab：主要区别

| 功能 | GitHub | GitLab |
|---------|--------|--------|
| CI/CD | GitHub Actions | 内置 CI/CD（更成熟） |
| Pull Requests | Pull Requests | Merge Requests |
| 托管 | 云（GitHub.com） | 云或自托管 |
| 定价 | 公开免费，私有有限制 | 更多免费功能 |
| 集成 | 广泛的生态系统 | 专注于 DevOps |
| 容器仓库 | GitHub Packages | 内置 Container Registry |
| 规划工具 | Projects, Issues | Issues, Boards, Epics |

两者都很优秀——选择通常取决于组织偏好。

---

## 克隆与 Fork

### 克隆仓库

克隆会在本地创建远程仓库的副本：

```bash
# 通过 HTTPS 克隆
git clone https://github.com/user/ml-pipeline.git

# 通过 SSH 克隆
git clone git@github.com:user/ml-pipeline.git

# 克隆到指定目录
git clone https://github.com/user/ml-pipeline.git my-pipeline

# 克隆指定分支
git clone -b develop https://github.com/user/ml-pipeline.git

# 浅克隆（更快，历史更少）
git clone --depth 1 https://github.com/user/ml-pipeline.git
```

克隆之后：
- Git 自动添加 `origin` 远程仓库
- 设置远程跟踪分支
- 检出默认分支

### Fork 仓库

Fork 是在 GitHub 上创建他人仓库的自己的副本：

```
Original Repo          Your Fork
(upstream)            (origin)
    |                     |
    |------ fork ------→ |
    |                     |
    ↓                     ↓
[github.com/owner/repo] [github.com/you/repo]
```

**何时 fork：**
- 为开源项目做贡献
- 实验他人的代码
- 创建项目自己的版本

**工作流：**
1. 在 GitHub 上点击 "Fork"
2. 克隆你的 fork（而非原始仓库）
3. 将原始仓库添加为 upstream 远程仓库
4. 在你的 fork 上工作
5. 向原始仓库提交 Pull Request

```bash
# 克隆你的 fork
git clone https://github.com/YOU/ml-toolkit.git
cd ml-toolkit

# 添加上游远程仓库（原始仓库）
git remote add upstream https://github.com/ORIGINAL-OWNER/ml-toolkit.git

# 验证远程仓库
git remote -v
# origin    https://github.com/YOU/ml-toolkit.git (fetch)
# origin    https://github.com/YOU/ml-toolkit.git (push)
# upstream  https://github.com/ORIGINAL-OWNER/ml-toolkit.git (fetch)
# upstream  https://github.com/ORIGINAL-OWNER/ml-toolkit.git (push)
```

### 同步 Fork

让你的 fork 与原始仓库保持同步：

```bash
# 从 upstream 拉取更改
git fetch upstream

# 将上游更改 merge 到你的 main 分支
git checkout main
git merge upstream/main

# 或 rebase（更清晰的历史）
git rebase upstream/main

# push 更新到你的 fork
git push origin main
```

也可以通过 GitHub 网页界面上的 "Sync fork" 按钮自动完成。

---

## Push 和 Pull 操作

### Fetch 更改

Fetch 从远程仓库下载 commit 但不会 merge：

```bash
# 从默认远程仓库（origin）fetch
git fetch

# 从指定远程仓库 fetch
git fetch upstream

# fetch 指定分支
git fetch origin main

# fetch 所有远程仓库
git fetch --all

# 清理已删除的远程分支引用
git fetch --prune
```

fetch 后，远程跟踪分支会被更新：
```bash
git log origin/main  # 查看远程 commit
```

### Pull 更改

Pull 在一个命令中同时完成 fetch 和 merge：

```bash
# 从跟踪的分支 pull（默认）
git pull

# 等同于：
git fetch origin
git merge origin/main

# 从指定远程仓库和分支 pull
git pull origin develop

# 使用 rebase 代替 merge
git pull --rebase

# pull 所有分支
git pull --all
```

**Pull 使用 merge 与 rebase 的对比：**

Merge（默认）：
```
Before pull:
  origin/main: A ← B ← C
  local main:  A ← B ← D

After git pull:
  main: A ← B ← C ← D ← M
               ↘_____↗
          (merge commit M)
```

Rebase：
```
Before pull:
  origin/main: A ← B ← C
  local main:  A ← B ← D

After git pull --rebase:
  main: A ← B ← C ← D'
          (D replayed on top of C)
```

### Push 更改

Push 将你的 commit 发送到远程仓库：

```bash
# 将当前分支 push 到其远程跟踪分支
git push

# push 到指定远程仓库和分支
git push origin main

# push 并设置上游（首次使用）
git push -u origin feature/new-api

# 使用 -u 后，只需：
git push

# push 所有分支
git push --all

# push 所有 tag
git push --tags

# 强制 push（危险！）
git push --force

# 更安全的强制 push（仅当没有其他人 push 过）
git push --force-with-lease
```

### 理解跟踪分支

当你克隆仓库时，分支会自动跟踪远程分支：

```bash
# 查看跟踪信息
git branch -vv

# 输出：
# * main    abc123 [origin/main] Latest commit message
#   feature def456 [origin/feature: ahead 2] Feature commit
```

`[origin/main]` 表示本地 `main` 分支跟踪 `origin/main`。

手动设置跟踪：
```bash
# 创建跟踪远程分支的本地分支
git checkout -b feature origin/feature

# 或为已有分支设置跟踪
git branch --set-upstream-to=origin/feature feature
```

### Push 被拒：处理非快进更新

**问题**：在你之前已经有人 push 到了远程仓库：

```bash
git push

# 输出：
# ! [rejected]        main -> main (non-fast-forward)
# error: failed to push some refs to 'origin'
# hint: Updates were rejected because the tip of your current branch is behind
```

**解决方案**：
```bash
# 选项 1：先 pull 再 merge
git pull origin main
# （解决任何冲突）
git push origin main

# 选项 2：先 pull 再 rebase
git pull --rebase origin main
# （解决任何冲突）
git push origin main

# 选项 3：强制 push（只有在确认无误时才使用！）
git push --force-with-lease origin main
```

**切勿对共享分支使用强制 push**，除非与团队协调好。

---

## Pull Request 和代码审查

### 什么是 Pull Request

Pull Request（在 GitHub 上称为 PR，GitLab 上称为 Merge Request）是将你的分支 merge 到另一个分支的请求。它支持：
- merge 前的代码审查
- 关于更改的讨论
- 自动化测试（CI/CD）
- 审批工作流
- 更改记录

### 创建 Pull Request

**通过 GitHub 网页界面：**

1. Push 你的功能分支：
```bash
git checkout -b feature/add-caching
# 进行更改
git add .
git commit -m "Implement Redis caching"
git push -u origin feature/add-caching
```

2. 导航到 GitHub 上的仓库
3. 点击 "Compare & pull request" 按钮
4. 填写 PR 详细信息：
   - Title： "Add Redis caching layer"
   - Description： 解释做了什么、为什么做以及如何做的
   - Reviewers： 选择团队成员
   - Assignees： 谁来 merge 这个 PR
   - Labels： bug, feature, documentation 等
   - Milestone： 发布版本
   - Projects： 关联到项目面板

5. 点击 "Create pull request"

### Pull Request 模板

创建 `.github/PULL_REQUEST_TEMPLATE.md`：

```markdown
## Description
<!-- Describe your changes in detail -->

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## How Has This Been Tested?
<!-- Describe the tests you ran -->
- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual testing

## Checklist
- [ ] My code follows the style guidelines of this project
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes

## Related Issues
<!-- Link to related issues -->
Closes #123
```

### 代码审查最佳实践

**对于作者（创建 PR）：**

1. **保持 PR 小巧**：200-400 行为上限，聚焦于一件事
2. **写描述性的标题**："Add Redis caching" 而不是 "Update code"
3. **提供上下文**：解释为什么改，而不仅仅是改了什么
4. **先自我审查**：在请求审查之前先阅读自己的 diff
5. **及时回复**：快速响应反馈
6. **不要觉得是针对个人**：审查是为了代码质量

**对于审查者：**

1. **及时审查**：24 小时内完成
2. **建设性地提意见**：建议改进，而不仅仅是批评
3. **解释你的理由**："这可能引起 X 问题，因为 Y"
4. **区分重点**：必须修复 vs. 锦上添花 vs. 小细节
5. **慷慨地批准**：不要因小风格问题而卡住
6. **本地测试**：对于关键更改，pull 下来测试

**审查评论：**

```
❌ 不好："This is wrong."
✅ 好："This could lead to a race condition if two requests access
         the cache simultaneously. Consider using a lock here."

❌ 不好："Why did you do this?"
✅ 好："I'm curious about the reasoning for this approach. Have you
         considered using X instead? It might be more performant."

❌ 不好："Nit: space"
✅ 好："Nit (non-blocking): Extra space here. Our linter should
         catch this in CI."
```

### 审查 AI/ML 基础设施代码

需要特别注意的事项：

**配置更改：**
- 验证值是否正确对应环境
- 检查是否有硬编码的 secrets
- 确保向后兼容

**性能：**
- 能否处理预期的负载？
- 有无内存泄漏或资源问题？
- 对推理延迟的影响？

**可复现性：**
- 是否设置了随机种子？
- 依赖是否指定了版本？
- 结果能否复现？

**测试：**
- 逻辑要有单元测试
- pipeline 要有集成测试
- 性能基准测试

**文档：**
- API 变更是否有文档？
- README 是否更新？
- 复杂逻辑是否加了注释？

### 处理审查反馈

```bash
# 做出要求的修改
git checkout feature/add-caching
# 编辑文件
git add .
git commit -m "Address review feedback: improve error handling"
git push origin feature/add-caching
```

PR 会自动更新为新的 commit。

### 合并 Pull Request

**GitHub 上的三种 merge 策略：**

1. **Create a merge commit**（默认）
   - 保留所有 commit
   - 创建 merge commit
   - 完整的历史记录可见

2. **Squash and merge**
   - 将所有 commit 合并为一个
   - 干净的线性历史
   - 丢失单个 commit 的历史

3. **Rebase and merge**
   - 将 commit 重放到目标分支上
   - 线性历史
   - 没有 merge commit
   - 保留单个 commit

**Merge 按钮工作流：**
```
1. 确保 CI 通过（绿色勾号）
2. 确保必须的审查已批准
3. 解决所有 merge 冲突
4. 点击 "Merge pull request"
5. 选择 merge 策略
6. 确认 merge
7. 删除分支（可选但推荐）
```

### 分支保护规则

用规则保护关键分支（main、production）：

**Settings → Branches → Branch protection rules：**

- ✅ Require pull request reviews before merging
   - 需要的批准审查数：1-2
   - 新 commit 后废弃过时的审查

- ✅ Require status checks to pass before merging
   - 测试必须通过
   - linting 必须通过
   - 安全扫描必须通过

- ✅ Require branches to be up to date before merging
   - 强制在 merge PR 前先 merge main 分支

- ✅ Require conversation resolution before merging
   - 所有审查评论必须解决

- ✅ Restrict who can push to matching branches
   - 只有维护者可以直接 push

- ✅ Require signed commits
   - 所有 commit 必须经过 GPG 签名

这些规则保障了质量并防止意外破坏。

---

## Issue 与项目管理

### GitHub Issues

Issue 用于跟踪 bug、功能和任务：

**创建 Issue：**
1. 导航到 "Issues" 选项卡
2. 点击 "New issue"
3. 选择模板（如果有）
4. 填写标题和描述
5. 添加 labels、assignees、milestone
6. 提交 issue

### Issue 模板

创建 `.github/ISSUE_TEMPLATE/bug_report.md`：

```markdown
---
name: Bug report
about: Create a report to help us improve
title: '[BUG] '
labels: bug
assignees: ''
---

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Run training script with config '...'
2. Observe error '...'
3. See error

**Expected behavior**
What you expected to happen.

**Screenshots**
If applicable, add screenshots.

**Environment:**
 - OS: [e.g. Ubuntu 22.04]
 - Python version: [e.g. 3.11]
 - GPU: [e.g. NVIDIA A100]
 - CUDA version: [e.g. 12.1]

**Additional context**
Add any other context about the problem here.
```

功能请求模板（`.github/ISSUE_TEMPLATE/feature_request.md`）：

```markdown
---
name: Feature request
about: Suggest an idea for this project
title: '[FEATURE] '
labels: enhancement
assignees: ''
---

**Is your feature request related to a problem?**
A clear description of the problem.

**Describe the solution you'd like**
What you want to happen.

**Describe alternatives you've considered**
Alternative solutions or features.

**Additional context**
Any other context, screenshots, or examples.
```

### 将 Issue 链接到 Pull Request

在 PR 描述或 commit 中：
```
Closes #123
Fixes #456
Resolves #789
Related to #321
```

当 PR 被 merge 后，这些 issue 会自动关闭。

### 机器学习项目的标签

用标签组织 issue：

**类型：**
- `bug` - 有东西坏了
- `feature` - 新功能
- `enhancement` - 改进已有功能
- `documentation` - 文档改进

**优先级：**
- `critical` - 生产环境挂了
- `high` - 重要，阻碍工作
- `medium` - 应该尽快完成
- `low` - 锦上添花

**组件：**
- `training` - 训练 pipeline 问题
- `inference` - 模型服务问题
- `data-pipeline` - 数据处理
- `infrastructure` - IaC、Kubernetes
- `monitoring` - 日志、指标
- `security` - 安全漏洞

**状态：**
- `in-progress` - 有人正在处理
- `blocked` - 无法继续
- `needs-review` - 等待反馈
- `wontfix` - 不予处理

### Projects 和 Milestones

**Projects**：用于组织 issue 的看板
- 列：Backlog, To Do, In Progress, In Review, Done
- 在列之间拖拽 issue
- 跟踪 sprint/发布进度

**Milestones**：按发布分组 issue
- `v1.0.0` - 初始发布
- `v1.1.0` - 首个功能更新
- `Q4 2024` - 季度目标

---

## 协作工作流

### 工作流 1：集中式（小型团队）

大家都在同一个仓库上工作：

```
   Shared Repository (GitHub)
            |
    +-------+-------+
    |       |       |
  Dev A   Dev B   Dev C
```

流程：
1. 克隆仓库
2. 创建功能分支
3. push 分支，创建 PR
4. 审查、merge、删除分支

### 工作流 2：Fork（开源）

贡献者 fork 主仓库：

```
   Main Repository
   (original owner)
         |
    fork | fork | fork
    +----+----+----+
    |    |    |    |
 Fork  Fork  Fork  Fork
 (A)   (B)   (C)   (D)
```

流程：
1. Fork 仓库
2. 克隆你的 fork
3. 创建功能分支
4. push 到你的 fork
5. 从你的 fork 向主仓库创建 PR
6. 维护者审查并 merge

### 工作流 3：带保护主分支的共享仓库

团队中最常见的方式：

```
     Main Repository
           |
        main (protected)
           |
    +------+------+
    |      |      |
feature  fix  enhance
 branch branch branch
```

流程：
1. 克隆共享仓库
2. 从 main 创建功能分支
3. 在功能分支上 commit
4. push 功能分支
5. 向 main 创建 PR
6. CI/CD 自动运行
7. 需要审查
8. 批准后 merge

### AI/ML 团队协作模式

**实验跟踪：**
```
main
 ├── experiment/baseline-resnet
 ├── experiment/resnet-dropout
 ├── experiment/vit-transformer
 └── experiment/ensemble
```

每个团队成员在独立分支上运行实验，提交结果，最优者 merge 到 main。

**功能开发：**
```
develop
 ├── feature/add-model-v2
 ├── feature/optimize-preprocessing
 └── feature/add-monitoring
```

**环境分离：**
```
main (production)
 ├── staging (预生产测试)
 └── develop (集成)
```

---

## 团队协作最佳实践

### 沟通

1. **写清晰的 commit message**：他人需要理解你的更改
2. **在 PR 中记录**：解释上下文和理由
3. **在 issue 中评论**：让干系人了解进展
4. **使用 Discussions**：用于设计决策和问题讨论
5. **更新 README**：保持文档最新

### 代码标准

1. **风格指南**：统一代码风格（Python 用 PEP 8）
2. **Linting**：自动化风格检查（Black、Flake8）
3. **测试**：新功能要求编写测试
4. **文档**：为公开 API 编写文档
5. **类型提示**：在 Python 中使用类型注解

### 自动化

1. **CI/CD**：每个 PR 自动测试
2. **Pre-commit hooks**：在 commit 前发现问题
3. **依赖更新**：Dependabot 用于安全更新
4. **代码扫描**：安全漏洞检测
5. **性能基准测试**：捕捉性能退化

### 安全

1. **绝不 commit secrets**：使用环境变量
2. **使用 GitHub Secrets**：用于 CI/CD 凭据
3. **审查依赖**：检查漏洞
4. **签名 commit**：GPG 签名验证
5. **双因素认证**：所有团队成员必须启用

### 新团队成员入职

1. **CONTRIBUTING.md**：记录贡献流程
2. **CODE_OF_CONDUCT.md**：设定社区标准
3. **Good first issues**：为新成员标记简单 issue
4. **导师制**：将新成员与有经验的成员配对
5. **文档**：保持设置指南的更新

---

## 常见问题与排错

### 问题 1：Pull Request 中的 Merge 冲突

**症状**：PR 显示与目标分支存在冲突。

**解决方案**：
```bash
# 用 main 更新功能分支
git checkout feature/my-feature
git fetch origin
git merge origin/main
# 解决冲突
git add .
git commit
git push origin feature/my-feature
```

PR 会自动更新，冲突即被解决。

### 问题 2：意外推送了 Secrets

**症状**：API 密钥或密码被 commit 并 push。

**立即处理**：
1. 立即轮换受损的 secret
2. 从仓库历史中移除（使用 BFG Repo-Cleaner）
3. 添加到 .gitignore
4. 对团队进行正确的 secret 管理培训

### 问题 3：无法 Push 到远程

**症状**：push 时权限被拒绝。

**原因和解决方法**：
```bash
# 远程 URL 错误
git remote -v
git remote set-url origin <正确的-url>

# SSH 密钥问题
ssh -T git@github.com  # 测试 SSH 连接
# 如果未设置，将 SSH 密钥添加到 GitHub

# 没有写入权限
# 请仓库所有者将你添加为协作者
```

### 问题 4：分支发生分歧

**症状**：本地和远程分支产生了分歧。

**解决方案**：
```bash
# 查看差异
git fetch origin
git log --oneline --graph --all

# 选项 1：merge 远程更改
git pull origin main

# 选项 2：rebase 到远程
git pull --rebase origin main

# 选项 3：重置到远程（丢弃本地 commit）
git reset --hard origin/main
```

### 问题 5：Pull Request 过大

**症状**：PR 有 1000+ 行，难以审查。

**解决方案**：
```bash
# 拆分为多个 PR
git checkout main
git checkout -b feature/part1
# cherry-pick 相关 commit
git cherry-pick abc123 def456
git push origin feature/part1
# 为第一部分创建 PR

# 对其他部分重复上述操作
```

### 问题 6：Fork 已过时

**症状**：Fork 落后上游很多 commit。

**解决方案**：
```bash
# 同步 fork
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

或使用 GitHub 的 "Sync fork" 按钮。

---

## 总结与要点

### 核心概念

1. **远程仓库**：在 GitHub/GitLab 上共享的副本
2. **克隆 vs Fork**：克隆用于协作，Fork 用于贡献
3. **Fetch vs Pull**：Fetch 下载，Pull 下载 + merge
4. **Pull Request**：在 merge 前实现代码审查
5. **Issue**：跟踪 bug 和功能
6. **保护规则**：强制执行质量标准

### 必备命令速查表

```bash
# Remotes
git remote add <name> <url>     # 添加远程仓库
git remote -v                   # 列出远程仓库
git remote show origin          # 显示远程仓库信息

# Fetching and pulling
git fetch origin                # 从 origin fetch
git pull origin main            # fetch 并 merge
git pull --rebase origin main   # fetch 并 rebase

# Pushing
git push origin main            # push 到远程
git push -u origin feature      # push 并设置上游
git push --force-with-lease     # 安全的强制 push

# Cloning and forking
git clone <url>                 # 克隆仓库
git remote add upstream <url>   # 添加上游（用于 fork）
```

### Pull Request 工作流

```
1. 创建功能分支
2. 进行 commit
3. push 分支到远程
4. 创建 Pull Request
5. 处理审查反馈
6. CI/CD 通过
7. 获得批准
8. merge PR
9. 删除功能分支
```

### 最佳实践

1. **小巧、频繁的 PR**：更容易审查，更快 merge
2. **描述性的 PR 标题**：解释改了什么
3. **全面的 PR 描述**：解释为什么改
4. **先自我审查**：在审查者之前发现问题
5. **及时回应反馈**：保持 PR 推进
6. **定期同步**：经常从 main pull
7. **保护 main 分支**：要求审查和测试通过
8. **使用 issue 模板**：标准化 bug 报告和功能请求
9. **将 PR 链接到 issue**：自动跟踪和关闭
10. **清理分支**：merge 后删除

### 给 AI/ML 团队的建议

- **实验分支**：在独立分支上跟踪实验
- **模型版本控制**：用 tag 标记模型版本发布
- **数据隐私**：切勿将真实数据 commit 到公开仓库
- **可复现性**：记录精确的依赖和种子
- **性能测试**：在 CI/CD 中基准测试推理延迟
- **A/B 测试**：用分支管理竞争模型版本

### 下一步

下一讲我们将涵盖：
- 高级 Git 技术（rebase、cherry-pick、stash）
- 用于自动化的 Git hooks
- 用于管理依赖的 submodule
- 针对 ML 特有挑战的 Git 策略
- 高级排错

### 延伸阅读

- GitHub Docs: https://docs.github.com/
- GitLab Docs: https://docs.gitlab.com/
- Pro Git book（第 5-6 章）：https://git-scm.com/book/en/v2
- GitHub Flow guide: https://guides.github.com/introduction/flow/

---

**练习题**：完成 Exercise 05，通过真实团队场景练习协作工作流。
