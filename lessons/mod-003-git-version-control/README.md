# 模块 003：Git 与版本控制

## 模块概述

版本控制是现代软件开发和基础设施工程的基石。本模块提供关于 Git（业界标准的分布式版本控制系统）和 GitHub（领先的协作与代码托管平台）的全面培训。

你将学习的不仅仅是 Git 命令的使用，还包括理解使 Git 强大的底层概念。你将练习协作工作流，学习 commit 管理的最佳实践、分支策略，以及如何处理真实项目中出现的复杂场景。这些技能对于管理基础设施即代码、与团队协作以及维持专业开发标准至关重要。

完成本模块后，你将能够自信地使用 Git 进行日常工作、参与团队项目，并在整个生命周期中有效管理代码。

## 学习目标

完成本模块后，你将能够：

1. **理解版本控制概念**及其对基础设施工程的重要性
2. **使用基础 Git 命令**来跟踪变更和管理仓库
3. **创建有效的 commit**，遵循行业标准的清晰消息
4. **使用分支**来在隔离环境中开发功能和修复 bug
5. **使用 GitHub 进行协作**，包括 pull request、代码审查和 issue
6. **自信地解决合并冲突**并理解其产生原因
7. **使用高级 Git 功能**，如 rebasing、stashing 和 cherry-picking
8. **遵循适合团队协作的 Git 工作流**（Git Flow、GitHub Flow）
9. **管理仓库**，包括 remote、tag 和 release
10. **对基础设施即代码项目应用 Git 最佳实践**

## 预备知识

- 建议先完成模块 002（Linux 基础）
- 熟悉命令行界面
- 基本了解文件系统和文本文件
- GitHub 账号（免费版即可）

**推荐配置：**
- 系统已安装 Git 2.30+
- 支持 Git 集成的文本编辑器或 IDE（推荐 VS Code）
- 已配置 SSH 密钥的 GitHub 账号
- 终端访问（原生 Linux、macOS、WSL2 或 Windows 上的 Git Bash）

## 时间投入

- **总预计时间：** 28-35 小时
- **讲座与阅读：** 9-11 小时（5 场综合讲座）
- **动手练习：** 15-19 小时
- **项目与协作：** 4-5 小时

**推荐节奏：**
- 兼职（5-10 小时/周）：3-4 周
- 全职（20-30 小时/周）：1-2 周

Git 的精通来自实践。请预留时间进行实验、安全地犯错，并为常用工作流建立肌肉记忆。

## 模块结构

本模块包含 5 场综合讲座，为期 3-4 周：

### 第 1 周：Git 基础与分支
- **讲座 1：** Git 基础（120 分钟）
- **讲座 2：** 分支与合并（120 分钟）
- **主题：** 版本控制概念、基本命令、commit 工作流、分支、冲突
- **关键技能：** init、clone、add、commit、status、log、diff、branch、merge、冲突解决
- **练习：** 创建仓库、制作 commit、功能分支、解决冲突

### 第 2 周：使用 GitHub 进行协作
- **讲座 3：** 使用 Git 和 GitHub 协作（120 分钟）
- **主题：** 远程仓库、pull request、代码审查、协作工作流
- **关键技能：** push、pull、fetch、pull request、fork、代码审查
- **练习：** 贡献项目、PR 工作流、团队协作

### 第 3 周：高级技术与最佳实践
- **讲座 4：** 高级 Git 技术（120 分钟）
- **讲座 5：** 用于 ML 和基础设施项目的 Git（90 分钟）
- **主题：** 高级命令、工作流、故障排除、ML/IaC 模式
- **关键技能：** stash、rebase、cherry-pick、hook、Git LFS、CI/CD 集成
- **练习：** 复杂场景、恢复技术、基础设施仓库

## 详细主题分解

### 1. 版本控制简介（4-5 小时）

#### 1.1 为什么版本控制很重要
- 版本控制系统的历史
- 版本控制对基础设施的好处
- 分布式与集中式版本控制
- Git 与其他 VCS 对比（SVN、Mercurial）
- 在 AI/ML 基础设施中的使用场景

#### 1.2 Git 基础
- Git 架构与对象模型
- 三种状态：工作目录、暂存区、仓库
- 理解 commit 和 commit 图
- SHA-1 哈希与对象存储
- 理解 HEAD 和引用

#### 1.3 安装与配置 Git
- 跨平台安装
- 初始配置（user.name、user.email）
- Git 配置级别（system、global、local）
- 有用的配置选项
- 为 GitHub 设置 SSH 密钥
- 凭据管理

#### 1.4 获取帮助
- Git 文档结构
- 使用 `git help` 命令
- 理解 man 手册
- 在线资源与社区
- 有效阅读 Git 文档

### 2. 基本 Git 操作（6-8 小时）

#### 2.1 创建与克隆仓库
- 初始化新仓库（`git init`）
- 克隆已有仓库（`git clone`）
- 理解 .git 目录结构
- 仓库结构与约定
- README、.gitignore 和 LICENSE 文件

#### 2.2 跟踪变更
- 检查仓库状态（`git status`）
- 添加文件到暂存区（`git add`）
- 理解暂存策略
- 删除和移动文件
- 查看差异（`git diff`）
- 使用 .gitignore 忽略文件
- 全局和本地忽略模式

#### 2.3 提交变更
- 创建 commit（`git commit`）
- 编写有效的 commit 消息
- Commit 消息约定
- 修改 commit
- 理解 commit 最佳实践
- 原子性 commit
- 何时提交以及提交频率

#### 2.4 查看历史
- 查看 commit 历史（`git log`）
- 格式化 log 输出
- 过滤历史
- 查看特定 commit（`git show`）
- 搜索 commit 历史
- 理解祖先引用（HEAD^、HEAD~2）
- 图形化 log 展示

#### 2.5 撤销变更
- 丢弃工作目录变更
- 取消暂存文件
- 修改 commit
- 理解破坏性与非破坏性操作
- 还原 commit（`git revert`）
- 重置 commit（`git reset`）
- 安全注意事项

### 3. 分支与合并（8-10 小时）

#### 3.1 理解分支
- 什么是分支以及为什么重要
- 分支作为指向 commit 的指针
- 创建分支（`git branch`）
- 切换分支（`git checkout`、`git switch`）
- 列出和管理分支
- 分支命名约定
- 远程跟踪分支

#### 3.2 分支策略
- 功能分支
- 发布分支
- 热修复分支
- Git Flow 工作流
- GitHub Flow 工作流
- 主干开发
- 选择正确的策略

#### 3.3 合并分支
- 快进合并
- 三方合并
- 合并 commit
- 合并策略与选项
- 理解合并父提交
- 何时使用 merge 与 rebase
- 合并最佳实践

#### 3.4 解决冲突
- 理解冲突产生的原因
- 冲突标记与结构
- 手动解决冲突
- 使用合并工具
- 中止和重启合并
- 预防冲突
- 解决后的测试

#### 3.5 变基
- 什么是 rebasing？
- 交互式 rebase
- Rebase 与 merge 对比
- 何时使用 rebase
- 变基已发布历史的危险
- Rebase 最佳实践
- 解决 rebase 冲突

### 4. 远程仓库与 GitHub（8-10 小时）

#### 4.1 理解远程仓库
- 什么是远程仓库？
- 添加 remote（`git remote`）
- 查看远程信息
- 多个 remote
- 远程命名约定（origin、upstream）
- 远程 URL（HTTPS vs SSH）

#### 4.2 推送与拉取
- 推送变更（`git push`）
- 获取变更（`git fetch`）
- 拉取变更（`git pull`）
- 理解 fetch 与 pull 的区别
- 跟踪分支
- Push 和 pull 选项
- 强制推送（何时以及为何不应该）

#### 4.3 GitHub 基础
- 创建 GitHub 仓库
- 仓库设置与选项
- README 与仓库文档
- GitHub UI 导航
- 仓库可见性（公开 vs 私有）
- GitHub 定价与方案
- GitHub 功能概览

#### 4.4 协作工作流
- Fork 仓库
- Clone 与 fork 的区别
- Upstream 和 origin 约定
- 保持 fork 同步
- 直接协作
- 组织仓库
- 访问控制与权限

#### 4.5 Pull Request
- 创建 pull request
- Pull request 描述
- 关联 issue
- 请求审查
- Pull request 工作流
- Draft pull request
- Pull request 模板
- 将 issue 转换为 PR

#### 4.6 代码审查
- 代码审查最佳实践
- 对代码进行评论
- 建议修改
- 批准和请求更改
- 审查礼仪
- 回应反馈
- 迭代 pull request

#### 4.7 Issue 与项目管理
- 创建和管理 issue
- Issue 模板
- 标签和里程碑
- 分配 issue
- Issue 引用和链接
- 项目与看板
- GitHub Actions 简介

### 5. 高级 Git 技术（6-8 小时）

#### 5.1 Stashing 变更
- 什么是 stash？
- 暂存未提交的变更
- 应用暂存的变更
- 管理多个 stash
- Stash 选项与变体
- 何时使用 stash
- Stash 与 commit 对比

#### 5.2 标签与发布
- 注释标签与轻量标签
- 创建标签
- 推送标签
- 语义化版本
- GitHub releases
- 发布说明
- 版本策略

#### 5.3 Cherry-Picking
- 什么是 cherry-picking？
- 选择特定 commit
- Cherry-pick 使用场景
- 潜在问题
- Cherry-pick 与 merge 对比
- 何时使用 cherry-pick
- 维护 commit 历史

#### 5.4 高级历史操作
- 交互式 rebase 工作流
- 压缩 commit
- 拆分 commit
- 重新排序 commit
- 编辑 commit 消息
- 修复旧的 commit
- 历史重写注意事项

#### 5.5 Git Hooks
- 客户端 hook
- 服务端 hook
- Pre-commit hook
- Pre-push hook
- Commit 消息 hook
- Hook 使用场景
- 与团队共享 hook

#### 5.6 子模块与子树
- 使用子模块管理依赖
- 使用子模块
- 子模块陷阱
- Git 子树作为替代方案
- 单仓库 vs 多仓库
- 依赖管理最佳实践

#### 5.7 高级故障排除
- 使用 git bisect 查找 bug
- 使用 reflog 恢复丢失的 commit
- 使用 blame 查找谁修改了什么
- 搜索代码历史
- 恢复已删除的分支
- 处理 detached HEAD
- 常见问题与解决方案

### 6. Git 在基础设施中的最佳实践（4-5 小时）

#### 6.1 基础设施即代码与 Git
- 为什么 IaC 应该在版本控制中
- 基础设施的仓库结构
- 基础设施的分支策略
- 环境特定分支
- Git 中的 Terraform/CloudFormation
- 密钥管理注意事项
- 配置管理

#### 6.2 Commit 卫生
- 原子性 commit 原则
- Commit 消息标准（Conventional Commits）
- 应在 commit 中包含什么
- 不应提交的内容（密钥、二进制文件、依赖）
- Commit 频率
- 保持 commit 聚焦
- Pre-commit 验证

#### 6.3 分支管理
- 长寿分支与短寿分支
- 分支清理策略
- 保护分支
- 分支策略
- 命名约定
- 何时删除分支
- 分支生命周期

#### 6.4 协作最佳实践
- Pull request 指南
- 代码审查标准
- 专业地处理反馈
- PR 中的文档
- 测试要求
- 持续集成集成
- 团队约定

#### 6.5 安全注意事项
- 避免提交密钥
- 有效使用 .gitignore
- 从历史中移除敏感数据
- GitHub 安全功能
- 依赖扫描
- 代码扫描
- 密钥扫描

#### 6.6 性能与维护
- 仓库大小管理
- 大文件处理（Git LFS）
- 保持历史整洁
- 垃圾回收
- 浅克隆
- 部分克隆
- 归档策略

## 讲座大纲

> **状态更新（2025-10-28）：** 所有 5 场综合讲座现已完成！每场讲座将相关主题结合以实现高效学习，并包含 AI 基础设施示例的动手实验。

### 讲座 1：Git 基础（120 分钟）
**文件：** `lecture-notes/01-git-fundamentals.md`
- 版本控制系统简介
- 为什么版本控制对 AI 基础设施很重要
- Git 架构与对象模型（commit、tree、blob）
- 跨平台安装与配置
- 为 GitHub 设置 SSH 密钥
- 基本 Git 工作流：init、clone、add、commit、status
- 理解三种状态（工作区、暂存区、仓库）
- 使用 git log 查看历史
- 理解 diff
- 撤销变更（checkout、restore、reset）
- **实验：** 完成 Git 设置和基本工作流练习

### 讲座 2：分支与合并（120 分钟）
**文件：** `lecture-notes/02-branching-merging.md`
- 分支概念与理论
- 创建和切换分支（branch、checkout、switch）
- 合并策略（快进、三方合并）
- 理解合并 commit
- 分支可视化与历史
- **合并冲突：** 为什么会发生
- 冲突标记与结构
- 解决策略与工具
- 冲突解决后的测试
- 预防策略
- 分支最佳实践
- **实验：** 功能分支工作流与冲突解决

### 讲座 3：使用 Git 和 GitHub 协作（120 分钟）
**文件：** `lecture-notes/03-collaboration.md`
- 远程仓库概念
- 推送和拉取变更
- Fetch 与 pull 的区别
- GitHub 基础与账号设置
- Fork 与 clone 工作流
- 为开源项目做贡献
- **Pull request 工作流：** 创建、更新、合并
- 编写有效的 PR 描述
- 代码审查最佳实践
- 迭代与反馈循环
- 协作模式（fork-and-PR、共享仓库）
- GitHub issue 与项目管理
- **实验：** 使用 PR 完成协作工作流

### 讲座 4：高级 Git 技术（120 分钟）
**文件：** `lecture-notes/04-advanced-git.md`
- Rebasing 基础与使用场景
- 交互式 rebase 进行 commit 清理
- Rebase vs merge：何时使用哪种
- Stashing 变更（save、apply、pop、list）
- Cherry-picking commit
- 使用 reflog 进行恢复
- 使用 git bisect 调试
- 子模块简介
- Git hooks（pre-commit、pre-push）
- 高级故障排除
- 恢复技术（丢失的 commit、错误操作）
- **实验：** 高级工作流场景与恢复

### 讲座 5：用于 ML 和基础设施项目的 Git（90 分钟）
**文件：** `lecture-notes/05-git-for-ml-projects.md`
- 大规模 ML 项目的版本控制
- 处理大文件（Git LFS）
- Jupyter notebook 版本控制策略
- 基础设施即代码（IaC）模式
- ML/基础设施的仓库组织
- 生产系统的分支策略
- 密钥管理（git-secret、加密配置）
- CI/CD 集成基础
- 团队 Git 工作流（Git Flow、GitHub Flow、主干开发）
- AI 基础设施最佳实践
- 基础设施代码审查
- **实验：** 设置 ML/基础设施仓库

## 动手练习

> **状态更新（2025-10-28）：** 所有 8 个综合练习现已完成！每个练习包含详细的分步说明、ML 基础设施示例和验证检查点。

**总时间：** 10-14 小时动手实践

本模块包含 8 个渐进式练习，旨在为 AI 基础设施建立实用的 Git 技能：

---

### 练习 01：创建你的第一个 ML 项目仓库
**时间：** 60-90 分钟 | **难度：** 入门 | **前置条件：** 讲座 01

初始化并配置 Git 仓库，考虑 ML 特定因素。

**练习的技能：**
- `git init`、`git config`、`git status`、`git add`、`git commit`
- 为 ML 项目创建全面的 `.gitignore`
- ML 项目目录结构
- 编写专业的 commit 消息
- Git 三态工作流（工作区 → 暂存区 → 已提交）

---

### 练习 02：处理 Commit 与历史
**时间：** 75-90 分钟 | **难度：** 入门 | **前置条件：** 讲座 01、练习 01

掌握 commit 技术与历史导航。

**练习的技能：**
- Conventional commit 消息（feat:、fix:、docs:）
- `git log` 变体（--oneline、--stat、--grep、-S）
- `git commit --amend` 用于修复错误
- `git revert` vs `git reset` 用于撤销变更
- 搜索 commit 历史
- 使用 `git show` 检查 commit

---

### 练习 03：功能开发的分支
**时间：** 75-90 分钟 | **难度：** 入门到中级 | **前置条件：** 讲座 02、练习 02

使用分支实现并行开发。

**练习的技能：**
- 创建和切换分支（`git branch`、`git switch`）
- 分支命名约定（feature/、fix/、hotfix/）
- 同时在多个功能上工作
- 查看和比较分支
- 安全地删除分支
- `git stash` 用于上下文切换

---

### 练习 04：合并与冲突解决
**时间：** 90-120 分钟 | **难度：** 中级 | **前置条件：** 讲座 02、练习 03

掌握分支集成与冲突解决。

**练习的技能：**
- 快进合并 vs 三方合并
- `git merge --no-ff`、`git merge --squash`
- 识别和解决合并冲突
- Python 代码和 YAML 配置中的冲突标记
- 使用合并工具
- `git merge --abort` 取消合并
- 合并后验证

---

### 练习 05：协作与 Pull Request
**时间：** 90-120 分钟 | **难度：** 中级 | **前置条件：** 讲座 03、练习 04

使用 GitHub 实现团队协作工作流。

**练习的技能：**
- Fork 与 clone 工作流
- 管理 remote（origin vs upstream）
- 创建和管理 pull request
- 代码审查最佳实践
- 专业地回应反馈
- GitHub CLI（`gh pr create`、`gh pr checkout`）
- `git push --force-with-lease`（安全强制推送）

---

### 练习 06：用于 ML 工作流的 Git - DVC 与模型版本控制
**时间：** 90-120 分钟 | **难度：** 中级到高级 | **前置条件：** 讲座 03-04、练习 05

使用 DVC 进行数据、模型和实验的版本控制。

**练习的技能：**
- DVC 设置与配置（`dvc init`、`dvc add`）
- 使用 DVC 跟踪数据集
- 模型制品版本控制
- 用于可重现性的 DVC 流水线
- DVC 远程存储（S3、GCS、本地）
- 使用 YAML 配置进行实验跟踪
- 用于 ML 验证的 Pre-commit hook

---

### 练习 07：高级 Git 技术
**时间：** 120-150 分钟 | **难度：** 高级 | **前置条件：** 讲座 04、练习 01-06

掌握专家级 Git 操作。

**练习的技能：**
- 交互式 rebase（`git rebase -i`）用于历史清理
- 自定义 Git hooks（pre-commit、pre-push）
- 跨分支 cherry-picking commit
- 高级 stashing 工作流
- `git bisect` 用于 bug 查找
- 使用 `git reflog` 恢复
- Git 子模块用于依赖管理
- `git worktree` 用于并行开发

---

### 练习 08：Git LFS 与管理大型 ML 项目
**时间：** 90-120 分钟 | **难度：** 中级 | **前置条件：** 讲座 05、所有之前的练习

使用 Git LFS 处理 ML 项目的大文件。

**练习的技能：**
- 安装和配置 Git LFS
- 使用 LFS 跟踪模型和数据集
- `.gitattributes` 配置
- 模型的语义化版本控制（X.Y.Z）
- DVC + LFS 集成
- 高效克隆（`GIT_LFS_SKIP_SMUDGE=1`）
- 使用 `git lfs prune` 优化存储
- 将现有项目迁移到 LFS

---

### 技能进阶

**基础（练习 01-02）：** 仓库管理、commit、历史
**分支（练习 03-04）：** 分支、合并、冲突
**协作（练习 05）：** 远程仓库、PR、代码审查
**ML 工作流（练习 06-08）：** DVC、Git LFS、高级技术

## 评估与考核

### 知识检查
- 每个主要部分后的测验（共 6 个测验）
- 命令理解问题
- 工作流程场景分析
- 最佳实践评估
- 故障排除场景

### 实践评估
- **命令熟练度：** 在不参考文档的情况下演示 Git 命令
- **工作流完成：** 执行完整的功能开发工作流
- **冲突解决：** 解决 10 个不同的冲突场景
- **协作：** 以作者和审查者身份完成完整的 PR 周期
- **仓库管理：** 设置和维护项目仓库

### 能力标准
要成功完成本模块，你应该能够：
- 自信地创建和管理 Git 仓库
- 制作带有清晰消息的原子性 commit
- 有效使用分支进行功能开发
- 合并分支并独立解决冲突
- 使用 pull request 和代码审查进行协作
- 在需要时使用高级 Git 功能
- 遵循基础设施即代码的最佳实践
- 排查常见的 Git 问题
- 向他人解释 Git 概念

### 作品集项目
**基础设施仓库设置：**
创建一个完整的基础设施仓库，展示：
- 合理的仓库结构
- Terraform 或类似 IaC 代码
- 有效的分支策略
- Pull request 工作流
- 文档标准
- CI/CD 集成
- 安全最佳实践

该项目展示你结合 Git 和 IaC 的知识。

## 资源与参考

> **注意：** 请参阅 `resources/recommended-reading.md` 获取学习材料、书籍和在线资源的完整列表。

### 基础资源
- 官方 Git 文档（git-scm.com）
- GitHub 指南（guides.github.com）
- Pro Git 书籍（免费在线）
- Git 参考手册（`man git`）

### 推荐书籍
- "Pro Git" by Scott Chacon and Ben Straub（免费在线）
- "Git Pocket Guide" by Richard E. Silverman
- "Version Control with Git" by Jon Loeliger and Matthew McCullough

### 在线学习
- GitHub Learning Lab
- Learn Git Branching（交互式可视化）
- Git Immersion（动手教程）
- Atlassian Git 教程
- Oh Shit, Git!?!（常见问题与解决方案）

### 工具与扩展
- GitKraken（图形化 Git 客户端）
- Git Graph（VS Code 扩展）
- Hub CLI（GitHub 命令行界面）
- GitHub CLI（`gh` 命令）
- tig（文本模式 Git 界面）

### 练习平台
- GitHub（免费公开仓库）
- GitLab（替代平台）
- Bitbucket（替代平台）
- Learn Git Branching（浏览器练习）

## 开始入门

### 步骤 1：安装与配置 Git
1. 为你的平台安装 Git
2. 配置用户名和邮箱
3. 为 GitHub 设置 SSH 密钥
4. 测试 GitHub 连接
5. 配置有用的别名

### 步骤 2：创建 GitHub 账号
1. 注册免费 GitHub 账号
2. 完善个人资料
3. 添加 SSH 密钥
4. 启用双因素认证
5. 探索 GitHub 界面

### 步骤 3：从讲座 1 开始
- 阅读版本控制简介
- 理解 Git 架构
- 完成设置实验
- 创建你的第一个仓库

### 步骤 4：定期练习
- 按顺序完成练习
- 尝试各种命令
- 安全地犯错
- 建立肌肉记忆
- 在个人项目中使用 Git

### 步骤 5：协作
- 寻找开源项目
- 贡献小修复
- 练习 pull request 工作流
- 参与代码审查
- 培养协作技能

## 成功技巧

1. **每日练习：** 在所有代码中使用 Git，包括个人项目
2. **可视化：** 使用图形化工具理解 commit 图
3. **安全实验：** 创建练习仓库来尝试新命令
4. **阅读 commit 消息：** 在流行项目中研究好的 commit 消息
5. **使用别名：** 为常用命令创建快捷方式
6. **从错误中学习：** Git 让你很难永久丢失数据
7. **遵循约定：** 尽早采用标准的 commit 消息格式
8. **协作：** 与他人合作加速学习
9. **阅读文档：** Git 文档全面且编写良好
10. **建立工作流：** 开发一致的个人工作流

## 常见陷阱与解决方案

### 陷阱 1：提交太多或太少
**解决方案：** 练习原子性 commit；每个 commit 应代表一个逻辑变更

### 陷阱 2：含糊的 commit 消息
**解决方案：** 使用 Conventional Commits 格式；解释"为什么"而不仅仅是"是什么"

### 陷阱 3：在错误的分支上工作
**解决方案：** 开始工作前检查分支；经常使用 `git status`

### 陷阱 4：害怕合并冲突
**解决方案：** 冲突是正常的；在安全环境中练习解决冲突

### 陷阱 5：对共享分支强制推送
**解决方案：** 永远不要对 main/master 强制推送；理解 rebase 的影响

### 陷阱 6：提交密钥
**解决方案：** 正确使用 .gitignore；提交前扫描；使用 pre-commit hooks

### 陷阱 7：大型二进制文件
**解决方案：** 使用 Git LFS；尽可能不包含二进制文件；使用 .gitignore

### 陷阱 8：Detached HEAD 状态
**解决方案：** 理解 HEAD 的含义；如果要更改，创建分支

## 故障排除指南

### 问题：合并冲突
```bash
# 解决步骤
git status  # 识别冲突文件
# 编辑文件解决冲突
git add <resolved-files>
git commit  # 完成合并
```

### 问题：提交到了错误的分支
```bash
# 使用 cherry-pick 解决
git checkout correct-branch
git cherry-pick <commit-hash>
git checkout wrong-branch
git reset --hard HEAD~1  # 从错误分支移除
```

### 问题：需要撤销上次 commit
```bash
# 保留变更
git reset --soft HEAD~1

# 丢弃变更
git reset --hard HEAD~1
```

### 问题：丢失 commit
```bash
# 查找丢失的 commit
git reflog

# 恢复特定 commit
git checkout <commit-hash>
git checkout -b recovery-branch
```

## 下一步

完成本模块后，你将准备好：
- **模块 004：** Python 用于基础设施（在所有代码中使用 Git）
- **模块 005：** Docker 与容器（Dockerfile 的版本控制）
- **所有未来模块：** Git 在整个课程中都会使用

你还将准备好：
- 为开源项目做贡献
- 在团队项目中进行专业协作
- 有效管理基础设施即代码
- 建立公开仓库作品集

## 开发状态

**当前状态：** 模板阶段 - 全面的结构已就位

**现已可用：**
- 完整的模块结构
- 详细的主题分解
- 讲座大纲
- 练习框架
- 评估标准

**正在开发中：**
- 包含图表的完整讲座笔记
- 分步练习说明
- 交互式教程
- 视频演示
- 练习仓库
- 解决方案指南

**计划更新：**
- GitHub Actions 集成
- 高级工作流模式
- 团队协作场景
- 大规模 Git 技术
- 故障排除手册

## 反馈与贡献

本模块持续改进中。欢迎贡献：
- 报告问题或建议改进
- 分享你的学习经验
- 贡献示例仓库
- 建议更多资源
- 帮助改进文档

---

**模块维护者：** AI 基础设施课程团队
**联系方式：** ai-infra-curriculum@joshua-ferguson.com
**最后更新：** 2025-10-18
**版本：** 1.0.0-template
