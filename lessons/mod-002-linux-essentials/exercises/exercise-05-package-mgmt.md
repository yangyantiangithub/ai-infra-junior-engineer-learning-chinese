# 练习 05：ML 栈安装的包管理

## 概述

本练习教你使用 Linux 包管理器为 ML 基础设施安装和管理软件包。你将学习安装 Python、CUDA、Docker、ML 框架，并管理不同 Linux 发行版上的依赖关系。这些技能对于设置 ML 开发和生产环境至关重要。

## 学习目标

完成本练习后，你将能够：
- 理解不同的包管理系统（apt、yum、dnf）
- 安装系统包和开发工具
- 使用 pip 和 conda 管理 Python 包
- 安装 CUDA 和 GPU 驱动
- 设置 Docker 和容器运行时
- 安装 ML 框架（TensorFlow、PyTorch）
- 处理依赖冲突和版本管理
- 创建可重现的安装脚本

## 先决条件

- 已完成练习 01-04
- 已完成讲座 04：系统管理基础（包管理部分）
- 可访问 Linux 系统（Ubuntu/Debian 或 RHEL/CentOS）
- Sudo 权限（用于系统包安装）
- 用于下载包的互联网连接

## 所需时间

- 预计：75-90 分钟
- 难度：中级

## 第 1 部分：系统包管理

### 步骤 1：理解包管理器

```bash
# 创建工作空间
mkdir -p ~/package-management-lab
cd ~/package-management-lab

# 检测你的发行版
cat > detect_distro.sh << 'EOF'
#!/bin/bash
if [ -f /etc/os-release ]; then
    . /etc/os-release
    echo "Distribution: $NAME"
    echo "Version: $VERSION"
    case $ID in
        ubuntu|debian)
            echo "Package Manager: apt" ;;
        centos|rhel|fedora)
            echo "Package Manager: dnf/yum" ;;
    esac
fi
EOF

chmod +x detect_distro.sh
./detect_distro.sh
```

### 步骤 2：APT 包管理（Ubuntu/Debian）

```bash
cd ~/package-management-lab

# APT 命令参考
cat > apt_basics.sh << 'EOF'
#!/bin/bash

cat << 'COMMANDS'
APT BASIC COMMANDS
==================

Update Package Index:
---------------------
sudo apt update                 Update package lists
sudo apt upgrade                Upgrade all packages

Install Packages:
-----------------
sudo apt install <package>      Install package
sudo apt install -y <package>   Install without confirmation

Remove Packages:
----------------
sudo apt remove <package>       Remove package (keep config)
sudo apt purge <package>        Remove package and config

Search and Info:
----------------
apt search <keyword>            Search for packages
apt show <package>              Show package details
apt list --installed            List installed packages
COMMANDS
EOF

cat apt_basics.sh
```

### 步骤 3：YUM/DNF 包管理（RHEL/CentOS）

```bash
cd ~/package-management-lab

cat > yum_basics.sh << 'EOF'
#!/bin/bash

cat << 'COMMANDS'
YUM/DNF COMMANDS
================

sudo yum check-update           Check for updates
sudo yum install <package>      Install package
sudo yum groupinstall "Development Tools"
sudo yum remove <package>       Remove package
yum search <keyword>            Search packages
yum list installed              List installed packages
sudo yum config-manager --add-repo <url>
COMMANDS
EOF

cat yum_basics.sh
```

## 第 2 部分：Python 包管理

### 步骤 4：pip 包管理

```bash
cd ~/package-management-lab
mkdir python_packages
cd python_packages

# pip 命令参考
cat > pip_reference.md << 'EOF'
# pip Package Management

pip install package_name              Install latest version
pip install package_name==1.2.3       Install specific version
pip install -r requirements.txt       Install from file
pip list                              List installed packages
pip list --outdated                   List outdated packages
pip freeze > requirements.txt         Save to file

# Virtual Environments
python3 -m venv ml_env                Create virtual environment
source ml_env/bin/activate            Activate
deactivate                            Deactivate

# ML Frameworks
pip install tensorflow==2.13.0
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install scikit-learn xgboost transformers
EOF

# 创建虚拟环境设置脚本
cat > setup_ml_env.sh << 'EOF'
#!/bin/bash
ENV_NAME="ml_env"

python3 -m venv $ENV_NAME
source $ENV_NAME/bin/activate
pip install --upgrade pip

pip install numpy==1.24.3 pandas==2.0.3 scikit-learn==1.3.0
pip install matplotlib==3.7.2 jupyter==1.0.0
pip install tensorflow==2.13.0
pip install torch==2.0.1 torchvision==0.15.2
pip install tqdm pyyaml python-dotenv

pip freeze > requirements.txt
echo "Environment created: $ENV_NAME"
echo "Activate with: source $ENV_NAME/bin/activate"
deactivate
EOF

chmod +x setup_ml_env.sh
```

### 步骤 5：Conda 包管理

```bash
cd ~/package-management-lab/python_packages

cat > conda_reference.md << 'EOF'
# Conda Package Management

# Environment Management
conda create -n myenv python=3.10      Create environment
conda activate myenv                   Activate
conda deactivate                       Deactivate
conda env list                         List environments

# Package Management
conda install package_name             Install package
conda install -c conda-forge package   Install from channel
conda list                             List installed packages

# ML Environments
conda create -n tensorflow python=3.10
conda activate tensorflow
conda install -c conda-forge tensorflow-gpu cudatoolkit=11.8

conda create -n pytorch python=3.10
conda activate pytorch
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
EOF
```

## 第 3 部分：CUDA 和 GPU 驱动

### 步骤 6：安装 NVIDIA 驱动和 CUDA

```bash
cd ~/package-management-lab
mkdir gpu_setup
cd gpu_setup

cat > install_nvidia_cuda.md << 'EOF'
# NVIDIA Drivers and CUDA Installation

# Check GPU
lspci | grep -i nvidia
nvidia-smi

# Ubuntu Installation
sudo ubuntu-drivers autoinstall
# or install specific version
sudo apt install nvidia-driver-535
sudo reboot

# CUDA Toolkit (Ubuntu)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda-toolkit-12-2

# Add to PATH (~/.bashrc)
export PATH=/usr/local/cuda-12.2/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.2/lib64:$LD_LIBRARY_PATH
EOF

# 验证脚本
cat > verify_cuda.sh << 'EOF'
#!/bin/bash
echo "=== NVIDIA Check ==="
lspci | grep -i nvidia || echo "No NVIDIA GPU found"
echo ""
echo "=== Driver ==="
nvidia-smi || echo "nvidia-smi failed"
echo ""
echo "=== CUDA Version ==="
nvcc --version || echo "nvcc not found"
EOF

chmod +x verify_cuda.sh
```

## 第 4 部分：Docker 安装

### 步骤 7：安装 Docker

```bash
cd ~/package-management-lab
mkdir docker_setup
cd docker_setup

cat > install_docker.sh << 'EOF'
#!/bin/bash
set -e

echo "Installing Docker on Ubuntu/Debian"

sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

echo "Docker installed! Log out and back in for group changes."
EOF

chmod +x install_docker.sh
```

## 第 5 部分：完整 ML 栈安装

### 步骤 8：自动化 ML 栈设置

```bash
cd ~/package-management-lab
mkdir full_stack
cd full_stack

cat > install_ml_stack.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LOG_FILE="installation.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

detect_distro() {
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        DISTRO=$ID
        log "Detected: $NAME $VERSION"
    fi
}

install_system_packages() {
    log "Installing system packages..."
    sudo apt update
    sudo apt install -y build-essential git curl wget python3 python3-pip python3-dev python3-venv htop
    log "System packages installed"
}

setup_python_env() {
    log "Setting up Python environment..."
    python3 -m venv ~/ml_env
    source ~/ml_env/bin/activate
    pip install --upgrade pip
    pip install numpy pandas scikit-learn matplotlib jupyter tensorflow torch torchvision
    pip install tqdm pyyaml python-dotenv mlflow
    pip freeze > ~/ml_env_requirements.txt
    deactivate
    log "Python environment created"
}

detect_distro
install_system_packages
setup_python_env

log "Installation complete!"
EOF

chmod +x install_ml_stack.sh
```

## 验证

```bash
cd ~/package-management-lab

cat > validate_installation.sh << 'EOF'
#!/bin/bash

echo "=== ML Stack Validation ==="

check_command() {
    if command -v "$1" &> /dev/null; then
        echo "✓ $2: $($1 --version 2>&1 | head -1)"
    else
        echo "✗ $2: Not found"
    fi
}

check_command gcc "GCC"
check_command python3 "Python"
check_command git "Git"

if [ -f ~/ml_env/bin/python ]; then
    echo "✓ Virtual environment exists"
fi

check_command docker "Docker"
echo "=== Validation Complete ==="
EOF

chmod +x validate_installation.sh
./validate_installation.sh
```

---

**恭喜！** 你已经掌握了 ML 基础设施的包管理！
