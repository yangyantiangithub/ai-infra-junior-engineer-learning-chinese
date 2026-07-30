# 开发工具指南

初级 AI 基础设施工程师的必备工具和搭建建议。

---

## 目录

- [开发环境](#开发环境)
- [文本编辑器和 IDE](#文本编辑器和-ide)
- [终端和 Shell](#终端和-shell)
- [容器工具](#容器工具)
- [Kubernetes 工具](#kubernetes-工具)
- [版本控制](#版本控制)
- [Python 开发](#python-开发)
- [监控和调试](#监控和调试)
- [API 开发和测试](#api-开发和测试)
- [数据库工具](#数据库工具)
- [云平台](#云平台)
- [效率工具](#效率工具)

---

## 开发环境

### 操作系统

**推荐**：Linux（Ubuntu 22.04 LTS）或 macOS

**选项 1：Linux 原生**
- 最适合开发
- 原生 Docker 支持
- 大多数服务器运行 Linux
- 推荐发行版：
  - Ubuntu 22.04 LTS
  - Fedora
  - Pop!_OS

**选项 2：macOS**
- 适合开发
- 基于 Unix
- 安装 Homebrew 包管理器

**选项 3：Windows + WSL2**
- 适用于 Linux 的 Windows 子系统 2
- 接近原生的 Linux 性能
- 从 Microsoft Store 安装 Ubuntu 22.04
- 配置 VSCode 以支持 WSL

**选项 4：虚拟机**
- VirtualBox 或 VMware
- 安装 Ubuntu Desktop
- 最低 4GB 内存，25GB 存储

---

## 文本编辑器和 IDE

### Visual Studio Code（推荐）
**免费 | 跨平台**

**安装**：
```bash
# Ubuntu/Debian
sudo snap install code --classic

# macOS
brew install --cask visual-studio-code

# Windows
# 从 https://code.visualstudio.com/ 下载
```

**必备扩展**：
- Python（Microsoft）
- Docker（Microsoft）
- Kubernetes（Microsoft）
- GitLens（增强版 Git）
- Remote - SSH
- Remote - Containers
- YAML（Red Hat）
- Pylance（Python 语言服务器）
- autoDocstring
- Better Comments
- Error Lens
- indent-rainbow

**推荐设置**：
```json
{
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "editor.rulers": [88],
  "files.trimTrailingWhitespace": true
}
```

### PyCharm Community Edition
**免费 | 专注于 Python**
- 强大的 Python IDE
- 出色的调试功能
- 内置数据库工具
- 下载：https://www.jetbrains.com/pycharm/

### Vim/Neovim（高级）
**免费 | 基于终端**
- 在服务器上无处不在
- 学习曲线陡峭
- 高度可定制
- 值得学习基础知识

---

## 终端和 Shell

### 终端模拟器

**Linux**：
- 默认 GNOME Terminal（还不错）
- Terminator（分屏功能）
- Alacritty（GPU 加速）

**macOS**：
- iTerm2（强烈推荐）
- 默认 Terminal（够用）

**Windows**：
- Windows Terminal（推荐）
- ConEmu
- Hyper

### Shell 配置

**Zsh + Oh My Zsh（推荐）**：
```bash
# 安装 Zsh
sudo apt install zsh  # Ubuntu
brew install zsh      # macOS

# 安装 Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# ~/.zshrc 中的推荐插件：
plugins=(git docker docker-compose kubectl python pip)

# 推荐主题：
ZSH_THEME="robbyrussell"  # 或 "agnoster"
```

**有用的 Zsh 插件**：
- zsh-autosuggestions
- zsh-syntax-highlighting
- zsh-completions

### 终端复用器

**tmux**：
```bash
# 安装
sudo apt install tmux  # Ubuntu
brew install tmux      # macOS

# 基本命令
tmux                    # 启动会话
tmux ls                # 列出会话
tmux attach            # 重新连接
Ctrl+b d               # 分离
Ctrl+b %               # 垂直分割
Ctrl+b "               # 水平分割
```

---

## 容器工具

### Docker Desktop
**个人使用免费 | 跨平台**
- 下载：https://www.docker.com/products/docker-desktop
- 包含 Docker Engine、Docker Compose、Kubernetes
- 管理容器的 GUI

**Ubuntu 安装**：
```bash
# 卸载旧版本
sudo apt remove docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release

# 添加 Docker 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 设置仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 将用户添加到 docker 组
sudo usermod -aG docker $USER
# 注销并重新登录
```

### Docker Compose
**免费**
- 随 Docker Desktop 自带
- 独立安装：
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Dive - Docker 镜像分析
**免费**
```bash
# 安装
brew install dive  # macOS
# Ubuntu：从 GitHub releases 下载

# 分析镜像层
dive myimage:latest
```

### Portainer（Docker 管理 UI）
**免费**
```bash
docker volume create portainer_data
docker run -d -p 9000:9000 --name=portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce
```

---

## Kubernetes 工具

### kubectl
**免费 | 官方 CLI**
```bash
# 安装（Linux）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 安装（macOS）
brew install kubectl

# 验证
kubectl version --client
```

### 本地 Kubernetes 集群

**kind（Kubernetes in Docker）**：
```bash
# 安装
brew install kind  # macOS
# Linux：从 GitHub 下载

# 创建集群
kind create cluster --name dev
kind get clusters
kind delete cluster --name dev
```

**minikube**：
```bash
# 安装
brew install minikube  # macOS
# Ubuntu：从 GitHub 下载

# 启动
minikube start
minikube status
minikube dashboard  # Web UI
minikube stop
```

**Docker Desktop Kubernetes**：
- 在 Docker Desktop 设置中启用
- 单节点集群
- 易于使用

### kubectx 和 kubens
**免费 | 上下文和命名空间切换**
```bash
# 安装
brew install kubectx  # macOS

# 使用
kubectx                # 列出上下文
kubectx dev            # 切换到 dev 上下文
kubens                 # 列出命名空间
kubens production      # 切换命名空间
```

### k9s - Kubernetes CLI UI
**免费**
```bash
# 安装
brew install derailed/k9s/k9s  # macOS

# 运行
k9s
```

### Lens - Kubernetes IDE
**免费**
- 下载：https://k8slens.dev/
- 可视化 Kubernetes 管理
- 集群指标和日志
- 多集群支持

### Helm
**免费 | Kubernetes 包管理器**
```bash
# 安装
brew install helm  # macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash  # Linux

# 使用
helm repo add stable https://charts.helm.sh/stable
helm search repo <chart>
helm install myrelease stable/<chart>
```

---

## 版本控制

### Git
**免费 | 大多数操作系统内置**
```bash
# 安装
sudo apt install git  # Ubuntu
brew install git      # macOS

# 配置
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
```

### GitHub CLI (gh)
**免费**
```bash
# 安装
brew install gh       # macOS
sudo apt install gh   # Ubuntu

# 认证
gh auth login

# 使用
gh repo create
gh pr create
gh pr list
gh issue create
```

### GitKraken（Git GUI）
**公共仓库免费 | 私有仓库付费**
- 下载：https://www.gitkraken.com/
- 可视化 Git 界面
- 合并冲突解决

### SourceTree（Git GUI）
**免费**
- 下载：https://www.sourcetreeapp.com/
- GitKraken 的替代品

---

## Python 开发

### Python 版本管理

**pyenv**：
```bash
# 安装
curl https://pyenv.run | bash

# 添加到 shell 配置
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

# 安装 Python 版本
pyenv install 3.9.16
pyenv install 3.11.2
pyenv global 3.11.2
```

### 虚拟环境

**venv（内置）**：
```bash
# 创建
python -m venv venv

# 激活
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

# 停用
deactivate
```

**virtualenvwrapper（推荐）**：
```bash
# 安装
pip install virtualenvwrapper

# 添加到 ~/.bashrc 或 ~/.zshrc
export WORKON_HOME=$HOME/.virtualenvs
source /usr/local/bin/virtualenvwrapper.sh

# 使用
mkvirtualenv myproject
workon myproject
deactivate
rmvirtualenv myproject
```

### Python 工具

**Poetry（依赖管理）**：
```bash
# 安装
curl -sSL https://install.python-poetry.org | python3 -

# 使用
poetry new myproject
poetry add requests
poetry install
poetry run python app.py
```

**Black（代码格式化）**：
```bash
pip install black
black myfile.py
black .
```

**Flake8（代码检查）**：
```bash
pip install flake8
flake8 myfile.py
```

**Pylint（代码检查）**：
```bash
pip install pylint
pylint myfile.py
```

**pytest（测试）**：
```bash
pip install pytest pytest-cov
pytest
pytest --cov=src tests/
```

**ipython（增强版 Python Shell）**：
```bash
pip install ipython
ipython
```

---

## 监控和调试

### Prometheus
**免费 | 监控系统**
```bash
# Docker
docker run -d -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### Grafana
**免费 | 可视化平台**
```bash
# Docker
docker run -d -p 3000:3000 grafana/grafana
```

### htop
**免费 | 进程查看器**
```bash
sudo apt install htop
htop
```

### ctop
**免费 | 容器监控**
```bash
sudo wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 \
  -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop
ctop
```

### Sentry（错误跟踪）
**提供免费套餐**
- 注册：https://sentry.io/
- Python 集成：
```bash
pip install sentry-sdk
```

---

## API 开发和测试

### Postman
**免费套餐**
- 下载：https://www.postman.com/
- API 测试和开发
- 集合共享
- 环境变量

### Insomnia
**免费 | 开源**
- 下载：https://insomnia.rest/
- Postman 的替代品
- GraphQL 支持

### HTTPie
**免费 | CLI HTTP 客户端**
```bash
# 安装
pip install httpie

# 使用
http GET https://api.github.com
http POST https://httpbin.org/post name=John
```

### curl
**免费 | 内置**
```bash
curl https://api.github.com
curl -X POST -H "Content-Type: application/json" \
  -d '{"key":"value"}' https://httpbin.org/post
```

---

## 数据库工具

### pgAdmin（PostgreSQL）
**免费**
- 下载：https://www.pgadmin.org/
- PostgreSQL 的 GUI

### DBeaver
**免费 | 通用数据库工具**
- 下载：https://dbeaver.io/
- 支持多种数据库
- SQL 编辑器和可视化工具

### Redis Desktop Manager
**免费/付费**
- Redis GUI 客户端
- 替代品：RedisInsight（免费）

### MongoDB Compass
**免费**
- 官方 MongoDB GUI
- 下载：https://www.mongodb.com/products/compass

---

## 云平台

### AWS CLI
**免费**
```bash
# 安装
pip install awscli

# 配置
aws configure
```

### Google Cloud SDK
**免费**
```bash
# 安装
curl https://sdk.cloud.google.com | bash

# 初始化
gcloud init
```

### Azure CLI
**免费**
```bash
# 安装
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# 登录
az login
```

---

## 效率工具

### Notion
**个人使用免费**
- 笔记和文档管理
- 项目管理
- 知识库

### Obsidian
**免费**
- 基于 Markdown 的笔记
- 本地优先
- 图形化知识连接

### Draw.io (diagrams.net)
**免费**
- 架构图
- 流程图
- 在线或桌面版

### Excalidraw
**免费**
- 手绘风格图表
- 在线：https://excalidraw.com/

---

## 初学者推荐配置

### 最小配置
1. **操作系统**：Ubuntu 22.04 或 macOS
2. **IDE**：VSCode 带 Python 和 Docker 扩展
3. **终端**：默认终端 + Git
4. **Docker**：Docker Desktop
5. **K8s**：kind 或 minikube
6. **Python**：pyenv + venv

### 完整配置
以上所有内容加上：
- Zsh + Oh My Zsh
- kubectx/kubens
- k9s
- Postman
- DBeaver
- Prometheus + Grafana（Docker）

---

## 安装脚本

### 快速安装脚本（Ubuntu）
```bash
#!/bin/bash

# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y git curl wget build-essential

# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 安装 Python
sudo apt install -y python3 python3-pip python3-venv

# 安装 kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 安装 VSCode
sudo snap install code --classic

echo "安装完成！请注销并重新登录以使 Docker 组权限生效。"
```

---

## 浏览器扩展

### Chrome/Edge 扩展
- **JSON Formatter** - 美观打印 JSON
- **Wappalyzer** - 识别所用技术
- **React/Vue DevTools** - 前端调试
- **Lighthouse** - 性能审计

---

## 额外资源

- [VSCode Tips and Tricks](https://code.visualstudio.com/docs/getstarted/tips-and-tricks)
- [Oh My Zsh Plugins](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins)
- [awesome-docker](https://github.com/veggiemonk/awesome-docker)
- [awesome-kubernetes](https://github.com/ramitsurana/awesome-kubernetes)

---

**记住**：先从最小配置开始，根据需要逐步添加工具。不要试图一次性安装所有东西！
