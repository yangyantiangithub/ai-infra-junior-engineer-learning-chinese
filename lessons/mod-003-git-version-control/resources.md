# 模块 003 — Git & 版本控制资源

## 官方文档

- **Pro Git 书籍** — [git-scm.com/book](https://git-scm.com/book/en/v2)。免费、全面、定期更新。权威参考手册。
- **Git 参考** — [git-scm.com/docs](https://git-scm.com/docs)。按命令划分的文档。
- **GitHub 文档** — [docs.github.com](https://docs.github.com/)。特别关注 "Get started" 和 "Using Git" 部分。

## 书籍

- **Pro Git** by Scott Chacon and Ben Straub。上文已提到；值得再次提及，因为它既免费*又是*现存最好的 Git 书籍。
- **Git for Teams** by Emma Jane Hogbin Westby。侧重于协作工作流。
- **Version Control with Git（第 3 版）** by Prem Kumar Ponuthorai。涵盖包括 GitHub Actions 集成的现代主题。

## 交互式学习

- **Learn Git Branching** — [learngitbranching.js.org](https://learngitbranching.js.org/)。可视化、动手操作。特别适合学习分支、rebase、合并。
- **GitHub Skills** — [skills.github.com](https://skills.github.com/)。GitHub 内的交互式课程。
- **Oh My Git!** — [ohmygit.org](https://ohmygit.org/)。教授 Git 的开源游戏。

## 速查表

- **GitHub 的 Git 速查表** — [training.github.com/downloads/github-git-cheat-sheet.pdf](https://training.github.com/downloads/github-git-cheat-sheet.pdf)。
- **Atlassian Git 教程** — [atlassian.com/git](https://www.atlassian.com/git/tutorials)。在 merge vs. rebase 方面尤为出色。

## ML 特定版本控制

- **DVC 文档** — [dvc.org/doc](https://dvc.org/doc)。数据 + 模型版本控制，与 Git 互补。
- **MLflow 文档** — [mlflow.org/docs/latest](https://mlflow.org/docs/latest/index.html)。实验跟踪 + 模型注册表。
- **nbstripout** — [github.com/kynan/nbstripout](https://github.com/kynan/nbstripout)。在 commit 前清除 Jupyter notebook 输出。
- **jupytext** — [jupytext.readthedocs.io](https://jupytext.readthedocs.io/)。将 `.py` 与 `.ipynb` 一起保留，以获得友好的 diff。

## 工具

- **lazygit** — [github.com/jesseduffield/lazygit](https://github.com/jesseduffield/lazygit)。Git 的终端 UI。快速。
- **gh CLI** — [cli.github.com](https://cli.github.com/)。官方 GitHub CLI，用于 PR、issue、release。
- **GitHub Desktop / Sublime Merge / GitKraken** — GUI 客户端。如果你更喜欢可视化工作流，选择一个；否则 CLI 就很好。
- **pre-commit** — [pre-commit.com](https://pre-commit.com/)。管理 pre-commit hook 的框架。

## 工作流参考

- **Conventional Commits** — [conventionalcommits.org](https://www.conventionalcommits.org/)。commit 消息标准。
- **GitFlow** — [nvie.com/posts/a-successful-git-branching-model](https://nvie.com/posts/a-successful-git-branching-model/)。经典的分支模型。重量级；对于现代团队来说很少是正确的选择，但值得理解。
- **GitHub Flow** — [docs.github.com/en/get-started/quickstart/github-flow](https://docs.github.com/en/get-started/using-github/github-flow)。轻量级 branch + PR 模型。今天更常见。
- **Trunk-based development** — [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)。单一主分支，功能开关。用于高节奏团队。

## 本课程中的交叉引用

- 工程师轨道的 `mod-109-infrastructure-as-code`，用于 GitOps 与 ArgoCD / FluxCD。
- 模块 005（Docker）— 对 Dockerfile + .dockerignore 进行版本控制。
- 模块 006（Kubernetes）— 对清单文件进行版本控制。
