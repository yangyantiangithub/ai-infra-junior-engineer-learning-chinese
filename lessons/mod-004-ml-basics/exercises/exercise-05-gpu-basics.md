# 练习 05：GPU 基础 - 加速 ML 推理

**难度**: 初级-中级
**时长**: 2-3 小时
**前置要求**: Python 基础，已完成练习 01-04

**注意**：本练习可以在没有物理 GPU 的情况下使用云资源或仅 CPU 模式完成，以学习概念。

## 学习目标

到本练习结束时，你将：

1. 理解为什么 GPU 能加速 ML 工作负载
2. 检查 GPU 可用性和能力
3. 在 CPU 和 GPU 之间移动模型和数据
4. 比较 CPU vs GPU 推理性能
5. 理解 GPU 内存管理基础
6. 排查常见的 GPU 相关问题

## 你将构建什么

一个性能比较工具：
- 检测可用硬件（CPU/GPU）
- 在 CPU 和 GPU 上运行同一模型
- 测量和比较性能
- 生成性能报告

## 背景：为什么 ML 使用 GPU？

### CPU vs GPU 架构

**CPU（中央处理器）**：
- 核心少（通常 4-64 个）
- 高时钟速度
- 复杂指令集
- 适合顺序任务

**GPU（图形处理器）**：
- 数千个核心（2000-10000+ 个）
- 每个核心较低的时钟速度
- 针对并行操作优化
- 非常适合矩阵运算（ML 的基础）

### 为什么 ML 受益于 GPU

机器学习本质上是矩阵乘法：

```
输入（1000 个值）× 权重（1000×1000 矩阵）= 输出（1000 个值）
```

**在 CPU 上**：逐行乘以列，顺序执行
**在 GPU 上**：同时执行数千个元素的乘法

**加速示例**：
- 训练 ResNet-50：CPU ~2 周，GPU ~1 天
- LLM 推理：CPU ~1 token/s，GPU ~50 tokens/s

## 第 1 部分：GPU 检测和设置

### 步骤 1：安装支持 CUDA 的 PyTorch

首先，检查是否有 NVIDIA GPU：

```bash
# Linux/Mac
nvidia-smi

# 如果命令未找到，则没有 NVIDIA GPU 或驱动
```

**如果有 NVIDIA GPU**：

```bash
# 安装支持 CUDA 的 PyTorch
pip3 install torch torchvision --index-url https://download.pytorch.org/whl/cu118
```

**如果没有 GPU**（我们仍然可以学习概念）：

```bash
# 安装仅 CPU 的 PyTorch
pip3 install torch torchvision
```

安装其他依赖：

```bash
pip install transformers psutil
```

### 步骤 2：检查 GPU 可用性

创建 `check_gpu.py`：

```python
import torch
import sys

print("=" * 60)
print("GPU Detection and Information")
print("=" * 60)

# 检查 CUDA 可用性
cuda_available = torch.cuda.is_available()
print(f"\nCUDA Available: {cuda_available}")

if cuda_available:
    # GPU 信息
    gpu_count = torch.cuda.device_count()
    print(f"Number of GPUs: {gpu_count}")

    for i in range(gpu_count):
        print(f"\n--- GPU {i} ---")
        print(f"Name: {torch.cuda.get_device_name(i)}")
        print(f"Compute Capability: {torch.cuda.get_device_capability(i)}")

        # 内存信息
        total_memory = torch.cuda.get_device_properties(i).total_memory
        print(f"Total Memory: {total_memory / 1024**3:.2f} GB")

        # 当前内存使用
        allocated = torch.cuda.memory_allocated(i) / 1024**3
        reserved = torch.cuda.memory_reserved(i) / 1024**3
        print(f"Currently Allocated: {allocated:.2f} GB")
        print(f"Currently Reserved: {reserved:.2f} GB")

    # CUDA 版本
    print(f"\nCUDA Version: {torch.version.cuda}")
    print(f"cuDNN Version: {torch.backends.cudnn.version()}")
    print(f"cuDNN Enabled: {torch.backends.cudnn.enabled}")

else:
    print("\nNo CUDA-capable GPU detected.")
    print("This is fine for learning! We can still run everything on CPU.")
    print("\nTo use GPU in the future:")
    print("1. Get a machine with NVIDIA GPU")
    print("2. Install NVIDIA drivers")
    print("3. Install CUDA toolkit")
    print("4. Install PyTorch with CUDA support")

# PyTorch 配置
print("\n" + "=" * 60)
print("PyTorch Configuration")
print("=" * 60)
print(f"PyTorch Version: {torch.__version__}")
print(f"Python Version: {sys.version.split()[0]}")

# 在不同设备上测试 tensor 创建
print("\n" + "=" * 60)
print("Device Testing")
print("=" * 60)

# CPU tensor
cpu_tensor = torch.randn(1000, 1000)
print(f"✓ CPU tensor created: {cpu_tensor.shape}")
print(f"  Device: {cpu_tensor.device}")

# GPU tensor（如果可用）
if cuda_available:
    gpu_tensor = torch.randn(1000, 1000, device='cuda')
    print(f"✓ GPU tensor created: {gpu_tensor.shape}")
    print(f"  Device: {gpu_tensor.device}")
else:
    print("⚠ GPU tensors not available (CPU only mode)")
```

运行它：

```bash
python check_gpu.py
```

**有 GPU 的示例输出**：
```
============================================================
GPU Detection and Information
============================================================

CUDA Available: True
Number of GPUs: 1

--- GPU 0 ---
Name: NVIDIA GeForce RTX 3060
Compute Capability: (8, 6)
Total Memory: 12.00 GB
Currently Allocated: 0.00 GB
Currently Reserved: 0.00 GB

CUDA Version: 11.8
cuDNN Version: 8700
cuDNN Enabled: True

============================================================
PyTorch Configuration
============================================================
PyTorch Version: 2.0.0
Python Version: 3.10.12

============================================================
Device Testing
============================================================
✓ CPU tensor created: torch.Size([1000, 1000])
  Device: cpu
✓ GPU tensor created: torch.Size([1000, 1000])
  Device: cuda:0
```

## 第 2 部分：在 CPU 和 GPU 之间移动数据

### 步骤 3：设备管理基础

创建 `device_management.py`：

```python
import torch
import time

print("Device Management Basics")
print("=" * 60)

# 确定设备
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Using device: {device}\n")

# 在 CPU 上创建 tensor
print("1. Creating tensor on CPU")
cpu_tensor = torch.randn(1000, 1000)
print(f"   Shape: {cpu_tensor.shape}")
print(f"   Device: {cpu_tensor.device}")
print(f"   Memory: {cpu_tensor.element_size() * cpu_tensor.nelement() / 1024**2:.2f} MB")

if torch.cuda.is_available():
    print("\n2. Moving tensor to GPU")
    start = time.time()
    gpu_tensor = cpu_tensor.to('cuda')  # or cpu_tensor.cuda()
    transfer_time = time.time() - start

    print(f"   Device: {gpu_tensor.device}")
    print(f"   Transfer time: {transfer_time*1000:.2f} ms")

    print("\n3. Moving tensor back to CPU")
    start = time.time()
    cpu_tensor_back = gpu_tensor.to('cpu')  # or gpu_tensor.cpu()
    transfer_time = time.time() - start

    print(f"   Device: {cpu_tensor_back.device}")
    print(f"   Transfer time: {transfer_time*1000:.2f} ms")

    # 检查内存使用
    print("\n4. GPU Memory Usage")
    allocated = torch.cuda.memory_allocated() / 1024**2
    reserved = torch.cuda.memory_reserved() / 1024**2
    print(f"   Allocated: {allocated:.2f} MB")
    print(f"   Reserved: {reserved:.2f} MB")

    # 清理 GPU 内存
    print("\n5. Cleaning up GPU memory")
    del gpu_tensor
    torch.cuda.empty_cache()

    allocated = torch.cuda.memory_allocated() / 1024**2
    print(f"   Allocated after cleanup: {allocated:.2f} MB")

else:
    print("\n⚠ GPU not available - skipping GPU operations")

print("\n" + "=" * 60)
print("Best Practices:")
print("=" * 60)
print("1. Move model to device ONCE at startup")
print("2. Move input data to same device as model")
print("3. Move output back to CPU if needed for further processing")
print("4. Clean up GPU memory when done with large tensors")
```

## 第 3 部分：CPU vs GPU 性能对比

### 步骤 4：矩阵运算基准测试

创建 `benchmark_operations.py`：

```python
import torch
import time

def benchmark_matmul(size, device, iterations=100):
    """在指定设备上对矩阵乘法进行基准测试"""
    # 创建随机矩阵
    a = torch.randn(size, size, device=device)
    b = torch.randn(size, size, device=device)

    # 预热（GPU 很重要！）
    for _ in range(10):
        _ = torch.matmul(a, b)

    # 在计时前同步 GPU
    if device.type == 'cuda':
        torch.cuda.synchronize()

    # 基准测试
    start = time.time()
    for _ in range(iterations):
        c = torch.matmul(a, b)

    # 再次同步（GPU 操作是异步的）
    if device.type == 'cuda':
        torch.cuda.synchronize()

    elapsed = time.time() - start
    avg_time = elapsed / iterations

    return avg_time * 1000  # 转换为毫秒

# 运行基准测试
print("=" * 60)
print("Matrix Multiplication Benchmark")
print("=" * 60)

sizes = [100, 500, 1000, 2000]

for size in sizes:
    print(f"\nMatrix size: {size}x{size}")

    # CPU 基准测试
    cpu_time = benchmark_matmul(size, torch.device('cpu'))
    print(f"  CPU: {cpu_time:.2f} ms")

    # GPU 基准测试（如果可用）
    if torch.cuda.is_available():
        gpu_time = benchmark_matmul(size, torch.device('cuda'))
        speedup = cpu_time / gpu_time
        print(f"  GPU: {gpu_time:.2f} ms")
        print(f"  Speedup: {speedup:.1f}x faster")
    else:
        print("  GPU: Not available")

# 总结
print("\n" + "=" * 60)
print("Key Observations:")
print("=" * 60)
print("1. GPU is slower for small matrices (overhead costs)")
print("2. GPU advantage grows with matrix size")
print("3. For ML, most operations are large enough to benefit from GPU")
```

**预期输出**：
```
Matrix Multiplication Benchmark
============================================================

Matrix size: 100x100
  CPU: 0.12 ms
  GPU: 0.25 ms
  Speedup: 0.5x faster  ← GPU 对小矩阵更慢！

Matrix size: 500x500
  CPU: 8.45 ms
  GPU: 0.85 ms
  Speedup: 9.9x faster

Matrix size: 1000x1000
  CPU: 42.33 ms
  GPU: 1.87 ms
  Speedup: 22.6x faster

Matrix size: 2000x2000
  CPU: 421.56 ms
  GPU: 5.34 ms
  Speedup: 78.9x faster  ← 巨大加速！
```

### 步骤 5：模型推理对比

创建 `inference_benchmark.py`：

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
import time
import psutil

def get_memory_usage():
    """获取当前进程内存（MB）"""
    process = psutil.Process()
    return process.memory_info().rss / 1024 / 1024

def benchmark_model(model_name, device, num_runs=10):
    """在指定设备上对模型推理进行基准测试"""
    print(f"\n{'=' * 60}")
    print(f"Testing on {device}")
    print(f"{'=' * 60}")

    # 加载前内存
    mem_before = get_memory_usage()

    # 加载模型和 tokenizer
    print("Loading model...")
    start = time.time()

    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForCausalLM.from_pretrained(model_name)
    model = model.to(device)

    load_time = time.time() - start

    # 加载后内存
    mem_after = get_memory_usage()
    model_memory = mem_after - mem_before

    print(f"✓ Model loaded in {load_time:.2f}s")
    print(f"  Memory usage: {model_memory:.2f} MB")

    # GPU 内存（如适用）
    if device.type == 'cuda':
        gpu_memory = torch.cuda.memory_allocated() / 1024 / 1024
        print(f"  GPU memory: {gpu_memory:.2f} MB")

    # 准备输入
    prompt = "Machine learning infrastructure requires"
    inputs = tokenizer(prompt, return_tensors="pt")
    inputs = {k: v.to(device) for k, v in inputs.items()}

    # 预热
    print("Warming up...")
    with torch.no_grad():
        for _ in range(3):
            _ = model.generate(**inputs, max_length=50)

    # 基准测试
    print(f"Running {num_runs} inference iterations...")
    inference_times = []

    for i in range(num_runs):
        # 如果是 GPU，同步
        if device.type == 'cuda':
            torch.cuda.synchronize()

        start = time.time()

        with torch.no_grad():
            outputs = model.generate(**inputs, max_length=50)

        # 如果是 GPU，同步
        if device.type == 'cuda':
            torch.cuda.synchronize()

        elapsed = time.time() - start
        inference_times.append(elapsed)

        if i == 0:  # 显示第一个输出
            generated_text = tokenizer.decode(outputs[0], skip_special_tokens=True)
            print(f"\nSample output:\n{generated_text}\n")

    # 统计
    avg_time = sum(inference_times) / len(inference_times)
    min_time = min(inference_times)
    max_time = max(inference_times)

    print(f"Inference Statistics:")
    print(f"  Average: {avg_time:.3f}s")
    print(f"  Min: {min_time:.3f}s")
    print(f"  Max: {max_time:.3f}s")
    print(f"  Tokens/second: ~{50/avg_time:.1f}")

    # 清理
    del model
    if device.type == 'cuda':
        torch.cuda.empty_cache()

    return {
        'device': str(device),
        'load_time': load_time,
        'memory_mb': model_memory,
        'avg_inference_time': avg_time,
        'tokens_per_sec': 50/avg_time
    }

# 主要基准测试
model_name = "gpt2"  # 用于测试的小模型

print("=" * 60)
print(f"Model Inference Benchmark: {model_name}")
print("=" * 60)

results = []

# CPU 基准测试
cpu_result = benchmark_model(model_name, torch.device('cpu'))
results.append(cpu_result)

# GPU 基准测试（如果可用）
if torch.cuda.is_available():
    gpu_result = benchmark_model(model_name, torch.device('cuda'))
    results.append(gpu_result)

    # 比较
    print(f"\n{'=' * 60}")
    print("Comparison Summary")
    print(f"{'=' * 60}")

    speedup = cpu_result['avg_inference_time'] / gpu_result['avg_inference_time']
    print(f"\nInference Speedup: {speedup:.1f}x faster on GPU")
    print(f"CPU: {cpu_result['avg_inference_time']:.3f}s per inference")
    print(f"GPU: {gpu_result['avg_inference_time']:.3f}s per inference")

    print(f"\nThroughput:")
    print(f"CPU: {cpu_result['tokens_per_sec']:.1f} tokens/s")
    print(f"GPU: {gpu_result['tokens_per_sec']:.1f} tokens/s")

else:
    print("\n⚠ No GPU available for comparison")
```

## 第 4 部分：GPU 内存管理

### 步骤 6：理解 GPU 内存

创建 `gpu_memory_management.py`：

```python
import torch

if not torch.cuda.is_available():
    print("GPU not available - skipping this exercise")
    exit(0)

def print_gpu_memory(label=""):
    """打印当前 GPU 内存使用"""
    allocated = torch.cuda.memory_allocated() / 1024**2
    reserved = torch.cuda.memory_reserved() / 1024**2
    total = torch.cuda.get_device_properties(0).total_memory / 1024**2

    if label:
        print(f"\n{label}")
    print(f"  Allocated: {allocated:.2f} MB")
    print(f"  Reserved: {reserved:.2f} MB")
    print(f"  Total Available: {total:.2f} MB")
    print(f"  Free: {total - allocated:.2f} MB")

print("GPU Memory Management")
print("=" * 60)

# 初始状态
print_gpu_memory("Initial state:")

# 分配 tensor
print("\nAllocating 1000x1000 tensor...")
tensor1 = torch.randn(1000, 1000, device='cuda')
print_gpu_memory("After allocation:")

# 分配更多
print("\nAllocating another 2000x2000 tensor...")
tensor2 = torch.randn(2000, 2000, device='cuda')
print_gpu_memory("After second allocation:")

# 删除 tensor
print("\nDeleting first tensor...")
del tensor1
print_gpu_memory("After deletion (memory still reserved):")

# 清空缓存
print("\nEmptying cache...")
torch.cuda.empty_cache()
print_gpu_memory("After emptying cache:")

# 内存不足示例（小心！）
print("\n" + "=" * 60)
print("Handling Out of Memory")
print("=" * 60)

try:
    # 尝试分配巨大的 tensor
    print("\nTrying to allocate massive tensor...")
    huge_tensor = torch.randn(50000, 50000, device='cuda')
except RuntimeError as e:
    print(f"✗ Out of Memory Error:")
    print(f"  {str(e)}")
    print("\n  This is expected! GPU memory is limited.")
    print("  In production, you would:")
    print("  - Use smaller batch sizes")
    print("  - Use gradient checkpointing")
    print("  - Use model parallelism")
    print("  - Use mixed precision (FP16)")

# 清理
del tensor2
torch.cuda.empty_cache()
print_gpu_memory("\nAfter cleanup:")
```

## 第 5 部分：构建设备无关的 API

### 步骤 7：智能设备选择

创建 `smart_device_api.py`：

```python
from flask import Flask, request, jsonify
from transformers import pipeline
import torch
import time

app = Flask(__name__)

# 智能设备选择
device = 0 if torch.cuda.is_available() else -1  # 0 = GPU, -1 = CPU
device_name = "GPU" if device == 0 else "CPU"

print(f"Loading model on {device_name}...")
generator = pipeline('text-generation', model='gpt2', device=device)
print(f"✓ Model ready on {device_name}!")

@app.route('/info', methods=['GET'])
def info():
    """获取系统信息"""
    info_data = {
        "device": device_name,
        "cuda_available": torch.cuda.is_available(),
        "model": "gpt2"
    }

    if torch.cuda.is_available():
        info_data.update({
            "gpu_name": torch.cuda.get_device_name(0),
            "gpu_memory_total_gb": torch.cuda.get_device_properties(0).total_memory / 1024**3,
            "gpu_memory_allocated_gb": torch.cuda.memory_allocated() / 1024**3
        })

    return jsonify(info_data)

@app.route('/generate', methods=['POST'])
def generate():
    """生成文本并附带性能指标"""
    data = request.get_json()
    prompt = data.get('prompt', '')

    if not prompt:
        return jsonify({"error": "Prompt required"}), 400

    # 测量推理时间
    start = time.time()

    result = generator(
        prompt,
        max_length=data.get('max_length', 50),
        temperature=data.get('temperature', 0.7),
        num_return_sequences=1
    )

    inference_time = time.time() - start

    # 计算 tokens/秒（近似值）
    tokens_generated = len(result[0]['generated_text'].split())
    tokens_per_sec = tokens_generated / inference_time

    response = {
        "prompt": prompt,
        "generated_text": result[0]['generated_text'],
        "device": device_name,
        "inference_time_seconds": round(inference_time, 3),
        "tokens_per_second": round(tokens_per_sec, 1)
    }

    if torch.cuda.is_available():
        response["gpu_memory_used_gb"] = torch.cuda.memory_allocated() / 1024**3

    return jsonify(response)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

测试它：

```bash
python smart_device_api.py

# 在另一个终端：
curl http://localhost:5000/info

curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "GPUs accelerate machine learning by", "max_length": 60}'
```

## 挑战

### 挑战 1：混合精度训练

研究并实现 FP16（半精度）推理。将性能和内存使用与 FP32 进行比较。

### 挑战 2：批处理

修改 API：
- 在单个请求中接受多个 prompts
- 在 GPU 上作为批次处理
- 比较批处理与单独推理的性能

### 挑战 3：GPU 利用率监控

添加实时 GPU 利用率跟踪：
- 使用 `nvidia-smi` 或 `py3nvml` 库
- 记录 GPU 温度、利用率百分比
- 如果利用率低于阈值则告警

### 挑战 4：多 GPU 支持

如果你有多 GPU 系统：
- 实现模型并行
- 或数据并行
- 比较性能

## 关键要点

1. **GPU 大幅加速 ML**：通常实现 10-100 倍加速
2. **并非所有操作都同等受益**：小操作有开销
3. **内存有限**：GPU 内存很宝贵，需谨慎管理
4. **设备放置很重要**：保持模型和数据在同一设备上
5. **同步至关重要**：GPU 操作是异步的
6. **预热很重要**：首次运行总是较慢
7. **批大小影响利用率**：更大的批次更好地利用 GPU

## 基础设施注意事项

### 何时使用 GPU vs CPU

**对以下情况使用 GPU**：
- 训练（几乎总是）
- 有延迟要求的实时推理
- 高吞吐量推理
- 大批量处理
- 大模型（> 1 亿参数）

**CPU 适用于**：
- 小模型
- 低请求量
- 成本敏感型部署
- 开发/测试
- 延迟无关的批量推理

### GPU 成本优化

云 GPU 定价（近似值）：
- **T4**（16GB）：$0.35/小时
- **A10G**（24GB）：$1.00/小时
- **A100**（40GB）：$3.00-4.00/小时

**优化策略**：
1. 使用 spot 实例进行训练（60-90% 折扣）
2. 批量处理推理请求
3. 使用更小/量化模型
4. 根据需求自动扩展
5. 开发/测试用 CPU，生产用 GPU

## 下一步

1. **学习 GPU 优化**：
   - 模型量化（INT8, INT4）
   - Flash Attention
   - Kernel fusion

2. **探索分布式训练**：
   - DataParallel
   - DistributedDataParallel
   - DeepSpeed, FSDP

3. **研究推理优化**：
   - TensorRT
   - ONNX Runtime
   - vLLM, TGI

4. **监控生产 GPU**：
   - NVML/nvidia-smi
   - DCGM（Data Center GPU Manager）
   - Grafana 仪表板

## 资源

- [PyTorch CUDA 语义](https://pytorch.org/docs/stable/notes/cuda.html)
- [NVIDIA 开发者资源](https://developer.nvidia.com/)
- [GPU 性能指南](https://pytorch.org/tutorials/recipes/recipes/tuning_guide.html)

---

**恭喜！** 你现在理解了 ML 基础设施的 GPU 基础知识。你能就何时以及如何在生产系统中使用 GPU 做出明智的决策。
