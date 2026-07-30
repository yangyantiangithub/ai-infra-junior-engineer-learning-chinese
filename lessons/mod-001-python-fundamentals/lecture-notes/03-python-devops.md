# Lecture 03：Python 用于 DevOps 操作

## 目录
1. [引言](#引言)
2. [子进程管理](#子进程管理)
3. [文件和目录操作](#文件和目录操作)
4. [构建 CLI 工具](#构建-cli-工具)
5. [环境变量与系统交互](#环境变量与系统交互)
6. [AI 基础设施的实践示例](#ai-基础设施的实践示例)
7. [总结与最佳实践](#总结与最佳实践)

---

## 引言

### Python 作为基础设施语言

Python 在编排基础设施操作方面表现出色：
- **自动化**：部署模型、管理容器、配置资源
- **集成**：连接不同的工具和系统
- **脚本编写**：用可维护的 Python 代码替代复杂的 bash 脚本
- **工具开发**：为团队构建内部 CLI 工具

在 AI 基础设施中，你将使用 Python 来：
- 在 Kubernetes 集群上启动训练作业
- 管理云存储中的模型构件
- 自动化部署流水线
- 监控 GPU 利用率和日志
- 为数据科学家构建实用工具

### 学习目标

完成本讲座后，你将能够：
- 使用 subprocess 安全执行外部命令
- 使用 pathlib 执行文件和目录操作
- 使用 argparse 构建专业的 CLI 工具
- 有效使用环境变量
- 实现常见的基础设施自动化模式
- 在操作脚本中优雅处理错误

---

## 子进程管理

### 为什么使用子进程？

AI 基础设施涉及与外部工具的交互：
```bash
# 启动用于训练的 Docker 容器
docker run --gpus all nvidia/pytorch:latest python train.py

# 上传模型到 S3
aws s3 cp model.pt s3://my-bucket/models/

# 检查 GPU 状态
nvidia-smi

# 使用 kubectl 部署
kubectl apply -f deployment.yaml
```

Python 的 `subprocess` 模块让你可以在代码中调用这些命令。

### 基本子进程用法

#### 简单命令执行

```python
import subprocess

# 运行命令，等待完成
result = subprocess.run(["ls", "-la"])
print(f"Exit code: {result.returncode}")
```

#### 捕获输出

```python
import subprocess

# 捕获 stdout 和 stderr
result = subprocess.run(
    ["nvidia-smi", "--query-gpu=name,memory.total", "--format=csv"],
    capture_output=True,
    text=True
)

if result.returncode == 0:
    print("GPU Info:")
    print(result.stdout)
else:
    print(f"Error: {result.stderr}")
```

#### 检查是否成功

```python
import subprocess
import sys

try:
    # 如果命令失败则引发 CalledProcessError
    result = subprocess.run(
        ["docker", "ps"],
        check=True,
        capture_output=True,
        text=True
    )
    print(result.stdout)
except subprocess.CalledProcessError as e:
    print(f"Command failed with exit code {e.returncode}")
    print(f"Error output: {e.stderr}")
    sys.exit(1)
```

### 高级子进程模式

#### 带超时运行

```python
import subprocess

try:
    # 30 秒超时
    result = subprocess.run(
        ["python", "long_running_training.py"],
        timeout=30,
        capture_output=True,
        text=True
    )
except subprocess.TimeoutExpired:
    print("Training script timed out!")
```

#### 实时流式输出

```python
import subprocess

# 使用 Popen 进行实时输出流
process = subprocess.Popen(
    ["python", "train.py"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    bufsize=1  # 行缓冲
)

# 逐行读取输出
for line in process.stdout:
    print(f"[TRAIN] {line.rstrip()}")

# 等待完成
exit_code = process.wait()
print(f"Training finished with exit code: {exit_code}")
```

#### 并发输出流

```python
import subprocess
import threading

def stream_output(pipe, prefix):
    """从管道流式输出并添加前缀"""
    for line in pipe:
        print(f"[{prefix}] {line.rstrip()}")

process = subprocess.Popen(
    ["python", "train.py"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

# 并发流式输出 stdout 和 stderr
stdout_thread = threading.Thread(
    target=stream_output,
    args=(process.stdout, "OUT")
)
stderr_thread = threading.Thread(
    target=stream_output,
    args=(process.stderr, "ERR")
)

stdout_thread.start()
stderr_thread.start()

# 等待完成
exit_code = process.wait()
stdout_thread.join()
stderr_thread.join()

print(f"Exit code: {exit_code}")
```

### Shell 与直接执行

```python
import subprocess

# 直接执行（首选 - 更安全）
subprocess.run(["ls", "-la", "/tmp"])

# Shell 执行（危险 - 注入风险）
subprocess.run("ls -la /tmp", shell=True)  # 避免！
```

**为什么 shell=True 是危险的：**
```python
# 用户输入可能包含恶意命令
user_input = "/tmp; rm -rf /"
subprocess.run(f"ls {user_input}", shell=True)  # 灾难！

# 安全的替代方案
subprocess.run(["ls", user_input])  # 参数被视为数据而非代码
```

**何时使用 shell=True（很少需要）：**
- 需要 shell 功能（管道、通配符、变量扩展）
- 仅限受信任的硬编码命令

```python
# 使用管道（需要 shell）
subprocess.run(
    "ps aux | grep python | wc -l",
    shell=True,
    capture_output=True,
    text=True
)

# 更安全的替代：改用 Python
import subprocess

ps_result = subprocess.run(
    ["ps", "aux"],
    capture_output=True,
    text=True
)
python_processes = [
    line for line in ps_result.stdout.split('\n')
    if 'python' in line.lower()
]
print(f"Python processes: {len(python_processes)}")
```

### 实际基础设施示例

#### 启动 Docker 容器

```python
import subprocess
from typing import Optional

def run_training_container(
    image: str,
    script_path: str,
    gpu_count: int = 1,
    volumes: Optional[dict] = None
) -> int:
    """在 Docker 容器中启动训练"""

    cmd = [
        "docker", "run",
        "--gpus", f"all" if gpu_count == -1 else str(gpu_count),
        "--rm",  # 完成后移除容器
        "-it"    # 交互式终端
    ]

    # 添加卷挂载
    if volumes:
        for host_path, container_path in volumes.items():
            cmd.extend(["-v", f"{host_path}:{container_path}"])

    # 添加镜像和命令
    cmd.extend([image, "python", script_path])

    print(f"Launching container: {' '.join(cmd)}")

    result = subprocess.run(cmd)
    return result.returncode

# 使用
exit_code = run_training_container(
    image="nvidia/pytorch:23.10-py3",
    script_path="/workspace/train.py",
    gpu_count=2,
    volumes={
        "/local/data": "/workspace/data",
        "/local/models": "/workspace/models"
    }
)
```

#### 检查 GPU 可用性

```python
import subprocess
import json
from typing import List, Dict, Any

def get_gpu_info() -> List[Dict[str, Any]]:
    """使用 nvidia-smi 获取 GPU 信息"""
    try:
        result = subprocess.run(
            [
                "nvidia-smi",
                "--query-gpu=index,name,memory.total,memory.used,memory.free,utilization.gpu",
                "--format=csv,noheader,nounits"
            ],
            capture_output=True,
            text=True,
            check=True
        )

        gpus = []
        for line in result.stdout.strip().split('\n'):
            idx, name, mem_total, mem_used, mem_free, util = line.split(', ')
            gpus.append({
                "index": int(idx),
                "name": name,
                "memory_total_mb": int(mem_total),
                "memory_used_mb": int(mem_used),
                "memory_free_mb": int(mem_free),
                "utilization_percent": int(util)
            })

        return gpus

    except subprocess.CalledProcessError:
        print("nvidia-smi not available or failed")
        return []
    except FileNotFoundError:
        print("nvidia-smi not found (no NVIDIA drivers installed)")
        return []

# 使用
gpus = get_gpu_info()
for gpu in gpus:
    print(f"GPU {gpu['index']}: {gpu['name']}")
    print(f"  Memory: {gpu['memory_used_mb']}/{gpu['memory_total_mb']} MB")
    print(f"  Utilization: {gpu['utilization_percent']}%")
```

#### 上传模型到 S3

```python
import subprocess
from pathlib import Path

def upload_to_s3(
    local_path: Path,
    s3_uri: str,
    storage_class: str = "STANDARD"
) -> bool:
    """使用 AWS CLI 上传文件到 S3"""

    if not local_path.exists():
        print(f"Error: {local_path} does not exist")
        return False

    cmd = [
        "aws", "s3", "cp",
        str(local_path),
        s3_uri,
        "--storage-class", storage_class
    ]

    print(f"Uploading {local_path} to {s3_uri}...")

    try:
        result = subprocess.run(
            cmd,
            check=True,
            capture_output=True,
            text=True
        )
        print("Upload successful!")
        return True

    except subprocess.CalledProcessError as e:
        print(f"Upload failed: {e.stderr}")
        return False

# 使用
model_path = Path("/models/bert-finetuned.pt")
s3_uri = "s3://my-models/production/bert-finetuned-v2.pt"
upload_to_s3(model_path, s3_uri)
```

---

## 文件和目录操作

### 使用 pathlib（现代，推荐）

```python
from pathlib import Path

# 创建 Path 对象
project_root = Path("/home/user/ml-project")
models_dir = project_root / "models"  # 连接路径
config_file = project_root / "config" / "training.yaml"

# 检查存在性
if config_file.exists():
    print("Config found")

# 检查类型
if models_dir.is_dir():
    print("Models directory exists")

if config_file.is_file():
    print("Config file exists")
```

### 读写文件

#### 文本文件

```python
from pathlib import Path

config_file = Path("config.txt")

# 写入
config_file.write_text("batch_size=32\nlearning_rate=0.001")

# 读取
content = config_file.read_text()
print(content)

# 逐行
for line in config_file.read_text().splitlines():
    print(line)
```

#### 上下文管理器（推荐）

```python
from pathlib import Path

log_file = Path("training.log")

# 使用上下文管理器写入
with open(log_file, 'w') as f:
    f.write("Training started\n")
    f.write("Epoch 1 complete\n")

# 追加
with open(log_file, 'a') as f:
    f.write("Epoch 2 complete\n")

# 读取
with open(log_file, 'r') as f:
    for line in f:
        print(line.rstrip())
```

#### 二进制文件

```python
import pickle
from pathlib import Path

model_data = {"weights": [1.0, 2.0, 3.0], "bias": 0.5}

# 写入二进制
model_file = Path("model.pkl")
with open(model_file, 'wb') as f:
    pickle.dump(model_data, f)

# 读取二进制
with open(model_file, 'rb') as f:
    loaded_model = pickle.load(f)
```

### 目录操作

#### 创建目录

```python
from pathlib import Path

# 创建单个目录
models_dir = Path("/tmp/ml-project/models")
models_dir.mkdir(exist_ok=True)  # 如果存在则不会失败

# 创建嵌套目录
checkpoints_dir = Path("/tmp/ml-project/checkpoints/experiment-1")
checkpoints_dir.mkdir(parents=True, exist_ok=True)  # 创建所有父目录
```

#### 列出目录内容

```python
from pathlib import Path

project_dir = Path("/tmp/ml-project")

# 列出所有项目
for item in project_dir.iterdir():
    print(item)

# 仅列出文件
for file in project_dir.iterdir():
    if file.is_file():
        print(file)

# 仅列出目录
for directory in project_dir.iterdir():
    if directory.is_dir():
        print(directory)
```

#### Glob 模式

```python
from pathlib import Path

models_dir = Path("/tmp/models")

# 查找所有 .pt 文件
for model_file in models_dir.glob("*.pt"):
    print(model_file)

# 递归搜索（** 匹配任意数量的目录）
for checkpoint in models_dir.glob("**/checkpoint-*.pt"):
    print(checkpoint)

# 特定模式
for config in models_dir.glob("*/config.yaml"):
    print(config)
```

#### 文件信息

```python
from pathlib import Path
import os

file_path = Path("model.pt")

# 大小
size_bytes = file_path.stat().st_size
size_mb = size_bytes / (1024 * 1024)
print(f"Size: {size_mb:.2f} MB")

# 时间戳
import datetime

modified_time = file_path.stat().st_mtime
modified_dt = datetime.datetime.fromtimestamp(modified_time)
print(f"Last modified: {modified_dt}")

# 权限
mode = file_path.stat().st_mode
print(f"Permissions: {oct(mode)}")
```

### ML 的实用文件操作

#### 清理旧检查点

```python
from pathlib import Path
import time

def cleanup_old_checkpoints(
    checkpoint_dir: Path,
    keep_recent: int = 5,
    max_age_days: int = 30
) -> None:
    """移除旧的检查点文件"""

    checkpoints = sorted(
        checkpoint_dir.glob("checkpoint-*.pt"),
        key=lambda p: p.stat().st_mtime,
        reverse=True  # 最新的在前
    )

    # 保留最近 N 个检查点
    for checkpoint in checkpoints[keep_recent:]:
        age_seconds = time.time() - checkpoint.stat().st_mtime
        age_days = age_seconds / (24 * 3600)

        if age_days > max_age_days:
            print(f"Removing old checkpoint: {checkpoint}")
            checkpoint.unlink()  # 删除文件

# 使用
cleanup_old_checkpoints(
    checkpoint_dir=Path("/models/checkpoints"),
    keep_recent=5,
    max_age_days=30
)
```

#### 组织模型文件

```python
from pathlib import Path
import shutil
from typing import Dict

def organize_models_by_experiment(
    models_dir: Path,
    output_dir: Path
) -> Dict[str, int]:
    """将模型文件按照实验组织到目录中"""

    output_dir.mkdir(parents=True, exist_ok=True)
    experiments: Dict[str, int] = {}

    # 模式：experiment-name-epoch-10.pt
    for model_file in models_dir.glob("*.pt"):
        parts = model_file.stem.split('-')

        if len(parts) >= 2:
            experiment_name = '-'.join(parts[:-2])
            experiment_dir = output_dir / experiment_name
            experiment_dir.mkdir(exist_ok=True)

            # 复制到组织好的位置
            dest = experiment_dir / model_file.name
            shutil.copy2(model_file, dest)

            experiments[experiment_name] = experiments.get(experiment_name, 0) + 1

    return experiments

# 使用
results = organize_models_by_experiment(
    models_dir=Path("/tmp/models"),
    output_dir=Path("/tmp/organized_models")
)

for exp, count in results.items():
    print(f"{exp}: {count} files")
```

#### 验证数据完整性

```python
import hashlib
from pathlib import Path

def calculate_file_hash(file_path: Path) -> str:
    """计算文件的 SHA256 哈希值"""
    sha256 = hashlib.sha256()

    with open(file_path, 'rb') as f:
        # 对大文件分块读取
        for chunk in iter(lambda: f.read(4096), b""):
            sha256.update(chunk)

    return sha256.hexdigest()

def verify_dataset(
    data_dir: Path,
    manifest_file: Path
) -> bool:
    """验证数据集文件是否与清单哈希匹配"""

    # 加载清单
    manifest = {}
    with open(manifest_file) as f:
        for line in f:
            file_path, expected_hash = line.strip().split()
            manifest[file_path] = expected_hash

    all_valid = True

    for file_path, expected_hash in manifest.items():
        full_path = data_dir / file_path

        if not full_path.exists():
            print(f"Missing: {file_path}")
            all_valid = False
            continue

        actual_hash = calculate_file_hash(full_path)
        if actual_hash != expected_hash:
            print(f"Corrupted: {file_path}")
            all_valid = False
        else:
            print(f"Valid: {file_path}")

    return all_valid
```

---

## 构建 CLI 工具

### 使用 argparse

```python
import argparse
from pathlib import Path

def main():
    parser = argparse.ArgumentParser(
        description="Train ML model with specified configuration"
    )

    # 位置参数
    parser.add_argument(
        "config",
        type=Path,
        help="Path to training configuration file"
    )

    # 可选参数
    parser.add_argument(
        "--epochs",
        type=int,
        default=10,
        help="Number of training epochs (default: 10)"
    )

    parser.add_argument(
        "--batch-size",
        type=int,
        default=32,
        help="Batch size for training (default: 32)"
    )

    parser.add_argument(
        "--gpu",
        type=int,
        default=0,
        help="GPU device ID (default: 0)"
    )

    # 布尔标志
    parser.add_argument(
        "--verbose",
        action="store_true",
        help="Enable verbose logging"
    )

    args = parser.parse_args()

    # 使用参数
    print(f"Config: {args.config}")
    print(f"Epochs: {args.epochs}")
    print(f"Batch size: {args.batch_size}")
    print(f"GPU: {args.gpu}")
    print(f"Verbose: {args.verbose}")

if __name__ == "__main__":
    main()
```

**使用：**
```bash
python train.py config.yaml --epochs 20 --batch-size 64 --verbose
```

### 高级 argparse 功能

#### 子命令

```python
import argparse

def train_command(args):
    print(f"Training model: {args.model}")

def evaluate_command(args):
    print(f"Evaluating model: {args.model}")

def deploy_command(args):
    print(f"Deploying model: {args.model} to {args.environment}")

def main():
    parser = argparse.ArgumentParser(description="ML model management tool")
    subparsers = parser.add_subparsers(dest="command", required=True)

    # Train 子命令
    train_parser = subparsers.add_parser("train", help="Train a model")
    train_parser.add_argument("--model", required=True, help="Model name")
    train_parser.add_argument("--epochs", type=int, default=10)
    train_parser.set_defaults(func=train_command)

    # Evaluate 子命令
    eval_parser = subparsers.add_parser("evaluate", help="Evaluate a model")
    eval_parser.add_argument("--model", required=True, help="Model name")
    eval_parser.set_defaults(func=evaluate_command)

    # Deploy 子命令
    deploy_parser = subparsers.add_parser("deploy", help="Deploy a model")
    deploy_parser.add_argument("--model", required=True, help="Model name")
    deploy_parser.add_argument("--environment", choices=["dev", "staging", "prod"])
    deploy_parser.set_defaults(func=deploy_command)

    args = parser.parse_args()
    args.func(args)

if __name__ == "__main__":
    main()
```

**使用：**
```bash
python ml_tool.py train --model bert --epochs 20
python ml_tool.py evaluate --model bert
python ml_tool.py deploy --model bert --environment prod
```

#### 参数分组

```python
import argparse

parser = argparse.ArgumentParser()

# 模型配置组
model_group = parser.add_argument_group("model", "Model configuration")
model_group.add_argument("--model-name", required=True)
model_group.add_argument("--model-type", choices=["bert", "gpt"])

# 训练配置组
train_group = parser.add_argument_group("training", "Training parameters")
train_group.add_argument("--epochs", type=int, default=10)
train_group.add_argument("--batch-size", type=int, default=32)

# 硬件配置组
hw_group = parser.add_argument_group("hardware", "Hardware settings")
hw_group.add_argument("--gpu", type=int)
hw_group.add_argument("--num-workers", type=int, default=4)

args = parser.parse_args()
```

### 完整 CLI 示例：模型部署工具

```python
#!/usr/bin/env python3
"""
Model deployment CLI tool
"""
import argparse
import logging
import sys
from pathlib import Path
from typing import Optional

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def deploy_model(
    model_path: Path,
    environment: str,
    replicas: int,
    gpu_enabled: bool
) -> bool:
    """将模型部署到指定环境"""
    logger.info(f"Deploying {model_path} to {environment}")
    logger.info(f"Replicas: {replicas}, GPU: {gpu_enabled}")

    # 实现代码会在这里
    # - 构建 Docker 镜像
    # - 推送到仓库
    # - 更新 Kubernetes 部署
    # - 等待 rollout

    return True

def list_deployments(environment: Optional[str] = None) -> None:
    """列出当前部署"""
    logger.info(f"Listing deployments for environment: {environment or 'all'}")
    # 实现代码会查询 Kubernetes

def rollback_deployment(deployment_name: str, environment: str) -> bool:
    """将部署回滚到先前版本"""
    logger.info(f"Rolling back {deployment_name} in {environment}")
    # 实现代码会使用 kubectl rollback
    return True

def main():
    parser = argparse.ArgumentParser(
        description="ML model deployment tool",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  # Deploy model to production
  %(prog)s deploy model.pt --environment prod --replicas 3 --gpu

  # List deployments
  %(prog)s list --environment prod

  # Rollback deployment
  %(prog)s rollback bert-service --environment prod
        """
    )

    parser.add_argument(
        "--verbose",
        action="store_true",
        help="Enable verbose logging"
    )

    subparsers = parser.add_subparsers(dest="command", required=True)

    # Deploy 命令
    deploy_parser = subparsers.add_parser("deploy", help="Deploy a model")
    deploy_parser.add_argument(
        "model_path",
        type=Path,
        help="Path to model file"
    )
    deploy_parser.add_argument(
        "--environment",
        choices=["dev", "staging", "prod"],
        required=True,
        help="Deployment environment"
    )
    deploy_parser.add_argument(
        "--replicas",
        type=int,
        default=1,
        help="Number of replicas"
    )
    deploy_parser.add_argument(
        "--gpu",
        action="store_true",
        help="Enable GPU support"
    )

    # List 命令
    list_parser = subparsers.add_parser("list", help="List deployments")
    list_parser.add_argument(
        "--environment",
        choices=["dev", "staging", "prod"],
        help="Filter by environment"
    )

    # Rollback 命令
    rollback_parser = subparsers.add_parser("rollback", help="Rollback deployment")
    rollback_parser.add_argument("deployment_name", help="Deployment name")
    rollback_parser.add_argument(
        "--environment",
        choices=["dev", "staging", "prod"],
        required=True
    )

    args = parser.parse_args()

    if args.verbose:
        logger.setLevel(logging.DEBUG)

    try:
        if args.command == "deploy":
            if not args.model_path.exists():
                logger.error(f"Model file not found: {args.model_path}")
                sys.exit(1)

            success = deploy_model(
                model_path=args.model_path,
                environment=args.environment,
                replicas=args.replicas,
                gpu_enabled=args.gpu
            )
            sys.exit(0 if success else 1)

        elif args.command == "list":
            list_deployments(args.environment)
            sys.exit(0)

        elif args.command == "rollback":
            success = rollback_deployment(
                args.deployment_name,
                args.environment
            )
            sys.exit(0 if success else 1)

    except KeyboardInterrupt:
        logger.info("Interrupted by user")
        sys.exit(130)
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## 环境变量与系统交互

### 读取环境变量

```python
import os

# 带默认值读取
model_path = os.getenv("MODEL_PATH", "/default/model.pt")

# 必需变量（缺失时引发 KeyError）
api_key = os.environ["API_KEY"]

# 检查变量是否存在
if "CUDA_VISIBLE_DEVICES" in os.environ:
    print(f"CUDA devices: {os.environ['CUDA_VISIBLE_DEVICES']}")
```

### 设置环境变量

```python
import os

# 为当前进程和子进程设置
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"

# 使用自定义环境启动子进程
import subprocess

custom_env = os.environ.copy()
custom_env["CUDA_VISIBLE_DEVICES"] = "2,3"
custom_env["OMP_NUM_THREADS"] = "8"

subprocess.run(
    ["python", "train.py"],
    env=custom_env
)
```

### 系统信息

```python
import os
import platform
import sys

print(f"Operating System: {platform.system()}")
print(f"Platform: {platform.platform()}")
print(f"Python Version: {sys.version}")
print(f"CPU Count: {os.cpu_count()}")
print(f"Current User: {os.getlogin()}")
print(f"Current Directory: {os.getcwd()}")
```

---

## AI 基础设施的实践示例

### Kubernetes 作业启动器

```python
#!/usr/bin/env python3
import subprocess
import yaml
from pathlib import Path
from typing import Dict, Any

def create_training_job(
    job_name: str,
    image: str,
    command: list,
    gpu_count: int = 1,
    namespace: str = "ml-training"
) -> Dict[str, Any]:
    """生成 Kubernetes 作业清单"""

    job_manifest = {
        "apiVersion": "batch/v1",
        "kind": "Job",
        "metadata": {
            "name": job_name,
            "namespace": namespace
        },
        "spec": {
            "template": {
                "spec": {
                    "restartPolicy": "Never",
                    "containers": [{
                        "name": "trainer",
                        "image": image,
                        "command": command,
                        "resources": {
                            "limits": {
                                "nvidia.com/gpu": gpu_count
                            }
                        }
                    }]
                }
            }
        }
    }

    return job_manifest

def submit_job(manifest: Dict[str, Any]) -> bool:
    """向 Kubernetes 提交作业"""
    manifest_file = Path("/tmp/job-manifest.yaml")

    # 写入清单
    with open(manifest_file, 'w') as f:
        yaml.dump(manifest, f)

    # 使用 kubectl 应用
    try:
        subprocess.run(
            ["kubectl", "apply", "-f", str(manifest_file)],
            check=True
        )
        print(f"Job submitted: {manifest['metadata']['name']}")
        return True
    except subprocess.CalledProcessError:
        print("Failed to submit job")
        return False
    finally:
        manifest_file.unlink()

# 使用
manifest = create_training_job(
    job_name="bert-finetuning-001",
    image="my-registry/pytorch-training:latest",
    command=["python", "train.py", "--config", "config.yaml"],
    gpu_count=4
)
submit_job(manifest)
```

---

## 总结与最佳实践

### 子进程最佳实践

✅ 对简单命令使用 `subprocess.run()`
✅ 始终使用 `check=True` 捕获错误
✅ 对长时间运行命令使用 `timeout`
✅ 除非绝对必要，避免使用 `shell=True`
✅ 使用列表格式的命令：`["cmd", "arg1", "arg2"]`
✅ 捕获并记录输出以便调试
✅ 处理 `CalledProcessError` 和 `TimeoutExpired`

### 文件操作最佳实践

✅ 使用 `pathlib.Path` 而不是字符串路径
✅ 对文件操作使用上下文管理器（`with`）
✅ 创建目录时使用 `exist_ok=True` 和 `parents=True`
✅ 为并发访问实现文件锁定
✅ 在操作前验证文件存在性
✅ 优雅处理权限错误

### CLI 开发最佳实践

✅ 使用描述性的帮助消息
✅ 提供合理的默认值
✅ 尽早验证输入
✅ 对复杂工具使用子命令
✅ 实现 `--verbose` 和 `--quiet` 标志
✅ 返回适当的退出码
✅ 在帮助文本中添加示例

---

## 下一步

你已经完成了 Python 编程的基础讲座！现在可以继续学习更高级的主题：

### 继续学习

**Lecture 04：异步编程**（`04-async-programming.md`）
- 学习使用 async/await 编写并发代码
- 同时处理多个 API 请求
- 构建高效的 I/O 密集型操作
- 为练习 06：异步模型监控做准备

**Lecture 05：测试与代码质量**（`05-testing-code-quality.md`）
- 掌握 pytest 进行全面测试
- 实施代码质量工具（Black、Ruff、mypy）
- 设置自动化质量检查
- 编写可维护、专业的 Python 代码

### 应用你的知识

练习讲座 1-3 的技能：
- 为基础架构自动化构建 CLI 工具
- 为外部工具编写子进程包装器
- 实现文件处理实用程序
- 创建配置管理脚本

---

**讲座版本**：1.0
**最后更新**：2025 年 10 月
**字数**：约 3,800 字

**下一步**：进入 `04-async-programming.md`
