# 练习 04：LLM 基础 - 运行你的第一个语言模型

**难度**: 初级
**时长**: 2-3 小时
**前置要求**: Python 基础，基本 ML 概念

## 学习目标

到本练习结束时，你将：

1. 理解什么是大语言模型（LLMs）
2. 使用 Hugging Face Transformers 运行预训练 LLM 的推理
3. 理解 LLM 输入（prompts）和输出（生成文本）
4. 识别 LLM 资源需求和约束
5. 部署一个简单的 LLM API 端点

## 你将构建什么

一个简单的文本生成 API：
- 加载一个小型预训练语言模型
- 通过 HTTP 端点接受文本 prompt
- 使用 LLM 生成响应
- 将生成的文本返回给用户

## 背景：什么是 LLMs？

### 什么是大语言模型？

大语言模型（LLM）是一种在大规模文本数据上训练的神经网络，用于理解和生成类人文本。示例包括：
- GPT（OpenAI）
- LLaMA（Meta）
- BERT（Google）
- T5, Flan-T5（Google）

### 为什么基础设施工程师关心 LLMs

作为基础设施工程师，你不会训练 LLMs（这很昂贵且专业化），但你将：
- **部署** LLMs 用于应用程序
- **服务** 大规模的 LLM 推理
- **优化** LLM 性能和成本
- **管理** 模型版本和更新
- **监控** LLM 资源使用

### 关键概念

**1. Prompt**：给 LLM 的输入文本
```
Prompt: "用一句话解释 Kubernetes："
```

**2. 生成**：LLM 产生的输出文本
```
生成: "Kubernetes 是一个开源容器编排平台，可自动部署、扩展和管理容器化应用程序。"
```

**3. Tokens**：模型处理的文本单元（大致为单词/子词）
```
"Hello world" → ["Hello", " world"] → 2 tokens
```

**4. 上下文窗口**：模型一次可以处理的最大 tokens 数量
```
小模型: 512-1024 tokens
大模型: 4096-32768+ tokens
```

## 第 1 部分：环境设置

### 步骤 1：安装依赖

创建一个新项目目录：

```bash
mkdir llm-basics-exercise
cd llm-basics-exercise
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
```

安装所需包：

```bash
pip install transformers torch flask requests
```

**包说明**：
- `transformers`: Hugging Face 的 LLM 库
- `torch`: PyTorch（transformers 的后端）
- `flask`: Web 框架用于 API
- `requests`: HTTP 客户端用于测试

### 步骤 2：验证安装

创建 `test_install.py`：

```python
import torch
from transformers import pipeline

print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")

# 测试一个简单的 pipeline（首次运行将下载模型）
print("Testing Transformers...")
generator = pipeline('text-generation', model='gpt2', max_length=30)
result = generator("Hello, I am", max_length=20, num_return_sequences=1)
print(result[0]['generated_text'])
print("✓ Installation successful!")
```

运行它：

```bash
python test_install.py
```

**预期输出**：
```
PyTorch version: 2.0.0
CUDA available: False
Testing Transformers...
Downloading model...
Hello, I am a software engineer based in San Francisco...
✓ Installation successful!
```

## 第 2 部分：理解 LLM 推理

### 步骤 3：基本文本生成

创建 `basic_generation.py`：

```python
from transformers import pipeline

# 初始化模型（只发生一次，然后缓存）
print("Loading model...")
generator = pipeline(
    'text-generation',
    model='gpt2',           # 小型 124M 参数模型
    device=-1                # 使用 CPU (-1), 或 0 为 GPU
)
print("Model loaded!")

# 生成文本
prompt = "Machine learning is"
print(f"\nPrompt: {prompt}")

results = generator(
    prompt,
    max_length=50,          # 最大总 tokens
    num_return_sequences=3, # 生成 3 个变体
    temperature=0.7,        # 随机性（0=确定性，1=创造）
    top_k=50                # 每一步考虑 top 50 tokens
)

# 显示结果
for i, result in enumerate(results, 1):
    print(f"\nGeneration {i}:")
    print(result['generated_text'])
```

运行它：

```bash
python basic_generation.py
```

**发生了什么？**
1. 模型从 Hugging Face Hub 加载（首次下载）
2. Prompt 被转换为 tokens
3. 模型逐个 token 生成
4. Tokens 被转换回文本
5. 由于随机性，创建了多个变体

### 步骤 4：理解参数

创建 `parameter_exploration.py`：

```python
from transformers import pipeline

generator = pipeline('text-generation', model='gpt2', device=-1)

prompt = "The future of AI is"

print("=" * 60)
print("Experiment 1: Temperature Effect")
print("=" * 60)

# 低温度（确定性）
result = generator(prompt, max_length=30, temperature=0.1, num_return_sequences=1)
print(f"\nTemperature=0.1 (deterministic):\n{result[0]['generated_text']}")

# 高温度（创造性）
result = generator(prompt, max_length=30, temperature=1.5, num_return_sequences=1)
print(f"\nTemperature=1.5 (creative):\n{result[0]['generated_text']}")

print("\n" + "=" * 60)
print("Experiment 2: Max Length Effect")
print("=" * 60)

# 短文本生成
result = generator(prompt, max_length=20, temperature=0.7, num_return_sequences=1)
print(f"\nMax length=20:\n{result[0]['generated_text']}")

# 长文本生成
result = generator(prompt, max_length=80, temperature=0.7, num_return_sequences=1)
print(f"\nMax length=80:\n{result[0]['generated_text']}")
```

**关键参数说明**：
- **temperature**: 控制随机性（0.0-2.0）
  - 低（0.1-0.5）：可预测、集中
  - 中（0.7-1.0）：平衡
  - 高（1.0-2.0）：创造性、多样化
- **max_length**: 总 tokens（prompt + 生成）
- **top_k**: 只考虑概率最高的 K 个下一个 tokens
- **top_p**: 核采样（概率阈值）

## 第 3 部分：构建 LLM API

### 步骤 5：创建 Flask API

创建 `llm_api.py`：

```python
from flask import Flask, request, jsonify
from transformers import pipeline
import time

app = Flask(__name__)

# 启动时加载模型一次
print("Loading LLM model...")
generator = pipeline(
    'text-generation',
    model='gpt2',
    device=-1  # CPU
)
print("Model ready!")

@app.route('/health', methods=['GET'])
def health():
    """健康检查端点"""
    return jsonify({
        "status": "healthy",
        "model": "gpt2"
    })

@app.route('/generate', methods=['POST'])
def generate():
    """文本生成端点"""
    try:
        # 获取请求数据
        data = request.get_json()
        prompt = data.get('prompt', '')
        max_length = data.get('max_length', 50)
        temperature = data.get('temperature', 0.7)

        # 验证
        if not prompt:
            return jsonify({"error": "Prompt is required"}), 400

        if max_length > 200:
            return jsonify({"error": "max_length cannot exceed 200"}), 400

        # 生成
        start_time = time.time()
        result = generator(
            prompt,
            max_length=max_length,
            temperature=temperature,
            num_return_sequences=1
        )
        inference_time = time.time() - start_time

        # 返回响应
        return jsonify({
            "prompt": prompt,
            "generated_text": result[0]['generated_text'],
            "inference_time_seconds": round(inference_time, 2),
            "parameters": {
                "max_length": max_length,
                "temperature": temperature
            }
        })

    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

### 步骤 6：测试 API

**终端 1** - 启动服务器：

```bash
python llm_api.py
```

**终端 2** - 使用 curl 测试：

```bash
# 健康检查
curl http://localhost:5000/health

# 简单生成
curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Kubernetes is a",
    "max_length": 50,
    "temperature": 0.7
  }'

# 更具创造性
curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "The best way to deploy machine learning models is",
    "max_length": 100,
    "temperature": 1.0
  }'
```

创建 `test_api.py` 用于 Python 测试：

```python
import requests
import json

API_URL = "http://localhost:5000"

def test_generation(prompt, max_length=50, temperature=0.7):
    response = requests.post(
        f"{API_URL}/generate",
        json={
            "prompt": prompt,
            "max_length": max_length,
            "temperature": temperature
        }
    )

    if response.status_code == 200:
        data = response.json()
        print(f"\nPrompt: {data['prompt']}")
        print(f"Generated: {data['generated_text']}")
        print(f"Inference time: {data['inference_time_seconds']}s")
    else:
        print(f"Error: {response.json()}")

if __name__ == '__main__':
    # 测试用例
    test_generation("Docker containers are")
    test_generation("Machine learning infrastructure requires", max_length=80)
    test_generation("The future of AI is", temperature=1.2)
```

## 第 4 部分：资源监控

### 步骤 7：监控资源使用

创建 `monitor_resources.py`：

```python
import psutil
import time
from transformers import pipeline

def get_memory_usage():
    """获取当前内存使用（MB）"""
    process = psutil.Process()
    return process.memory_info().rss / 1024 / 1024

print("=" * 60)
print("LLM Resource Monitoring")
print("=" * 60)

# 加载模型前
mem_before = get_memory_usage()
print(f"\nMemory before loading model: {mem_before:.2f} MB")

# 加载模型
print("Loading model...")
start_time = time.time()
generator = pipeline('text-generation', model='gpt2', device=-1)
load_time = time.time() - start_time

mem_after = get_memory_usage()
print(f"Memory after loading model: {mem_after:.2f} MB")
print(f"Model memory usage: {mem_after - mem_before:.2f} MB")
print(f"Load time: {load_time:.2f} seconds")

# 运行推理
print("\nRunning inference...")
prompt = "Machine learning is"
start_time = time.time()

result = generator(prompt, max_length=50, num_return_sequences=1)

inference_time = time.time() - start_time
mem_during = get_memory_usage()

print(f"Inference time: {inference_time:.2f} seconds")
print(f"Memory during inference: {mem_during:.2f} MB")
print(f"\nGenerated text:\n{result[0]['generated_text']}")

# 资源摘要
print("\n" + "=" * 60)
print("Resource Summary")
print("=" * 60)
print(f"Total memory used: {mem_during:.2f} MB")
print(f"Model load time: {load_time:.2f}s")
print(f"Inference time: {inference_time:.2f}s")
print(f"Tokens per second: ~{50/inference_time:.1f}")
```

**预期输出**：
```
LLM Resource Monitoring
============================================================

Memory before loading model: 45.23 MB
Loading model...
Memory after loading model: 550.67 MB
Model memory usage: 505.44 MB
Load time: 2.34 seconds

Running inference...
Inference time: 1.23 seconds
Memory during inference: 562.11 MB

Resource Summary
============================================================
Total memory used: 562.11 MB
Model load time: 2.34s
Inference time: 1.23s
Tokens per second: ~40.7
```

## 第 5 部分：不同模型的比较

### 步骤 8：比较模型大小

创建 `compare_models.py`：

```python
import time
from transformers import pipeline
import psutil

def test_model(model_name):
    """测试模型并返回指标"""
    print(f"\n{'=' * 60}")
    print(f"Testing: {model_name}")
    print(f"{'=' * 60}")

    # 内存前
    process = psutil.Process()
    mem_before = process.memory_info().rss / 1024 / 1024

    # 加载模型
    start = time.time()
    try:
        generator = pipeline('text-generation', model=model_name, device=-1)
        load_time = time.time() - start

        # 内存后
        mem_after = process.memory_info().rss / 1024 / 1024
        model_memory = mem_after - mem_before

        # 运行推理
        start = time.time()
        result = generator("Hello, I am", max_length=50, num_return_sequences=1)
        inference_time = time.time() - start

        print(f"✓ Model loaded successfully")
        print(f"  Load time: {load_time:.2f}s")
        print(f"  Memory usage: {model_memory:.2f} MB")
        print(f"  Inference time: {inference_time:.2f}s")
        print(f"  Output: {result[0]['generated_text'][:80]}...")

        return {
            "model": model_name,
            "load_time": load_time,
            "memory_mb": model_memory,
            "inference_time": inference_time,
            "success": True
        }
    except Exception as e:
        print(f"✗ Error: {str(e)}")
        return {"model": model_name, "success": False, "error": str(e)}

# 测试不同大小的模型
models = [
    "gpt2",              # 124M parameters (~500MB)
    "distilgpt2",        # 82M parameters (~350MB)
    "gpt2-medium",       # 355M parameters (~1.5GB) - may be slow on CPU
]

results = []
for model in models:
    results.append(test_model(model))
    time.sleep(2)  # 测试之间冷却

# 摘要
print(f"\n{'=' * 60}")
print("Model Comparison Summary")
print(f"{'=' * 60}")

for result in results:
    if result['success']:
        print(f"\n{result['model']}:")
        print(f"  Memory: {result['memory_mb']:.2f} MB")
        print(f"  Load: {result['load_time']:.2f}s")
        print(f"  Inference: {result['inference_time']:.2f}s")
```

## 挑战

### 挑战 1：为 API 添加温度控制

修改 `llm_api.py`：
- 验证 temperature 在 0.0 到 2.0 之间
- 如果无效则返回错误
- 在 API 响应中记录该参数

### 挑战 2：实现流式响应

研究并实现逐 token 流式传输，而不是等待完整生成完成。

提示：查看 transformers 中的 `TextIteratorStreamer`。

### 挑战 3：多模型支持

扩展 API：
- 支持多个模型（gpt2, distilgpt2）
- 添加 `/models` 端点列出可用模型
- 向 `/generate` 端点添加 `model` 参数
- 按需加载模型（懒加载）

### 挑战 4：添加缓存

实现一个简单的缓存：
- 如果相同的 prompt + 参数被请求，返回缓存结果
- 设置缓存过期时间（例如 5 分钟）
- 添加 `/cache/clear` 端点

## 基础设施注意事项

### 资源需求

| 模型 | 参数 | 内存 | CPU 推理 | GPU 推理 |
|-------|-----------|--------|---------------|---------------|
| distilgpt2 | 82M | ~350 MB | ~2-3 tokens/s | ~50-100 tokens/s |
| gpt2 | 124M | ~500 MB | ~1-2 tokens/s | ~30-50 tokens/s |
| gpt2-medium | 355M | ~1.5 GB | ~0.5-1 tokens/s | ~20-30 tokens/s |
| gpt2-large | 774M | ~3 GB | ~0.2-0.5 tokens/s | ~10-20 tokens/s |

### 生产部署清单

- [ ] 模型在服务请求之前加载（而不是每个请求加载）
- [ ] 健康检查端点已实现
- [ ] 请求超时已配置
- [ ] 最大请求长度已验证
- [ ] OOM（内存不足）的错误处理
- [ ] 已收集指标（延迟、吞吐量）
- [ ] 用于调试的日志
- [ ] 限流已实现
- [ ] 资源监控（CPU, 内存, GPU）

## 关键要点

1. **LLMs 是资源密集型的**：即使是小模型也使用数百 MB
2. **CPU 上推理慢**：生产环境推荐 GPU
3. **模型加载成本高**：启动时加载一次，而不是每个请求加载
4. **Prompts 很重要**：不同的 prompts 产生不同的结果
5. **参数影响输出**：Temperature, max_length, top_k/top_p
6. **上下文窗口有限**：无法处理无限文本
7. **基于 token 的定价**：云 LLM API 按 token 收费

## 下一步

完成此练习后：

1. **探索更大的模型**（需要更多资源）：
   - Flan-T5（更好的指令遵循）
   - LLaMA-2（需要 Meta 批准）
   - Mistral-7B（优秀的开源选择）

2. **学习优化技术**：
   - 量化（减少模型大小）
   - 批处理（同时处理多个请求）
   - GPU 部署
   - 模型量化（4-bit, 8-bit）

3. **学习生产 LLM 服务**：
   - vLLM（高吞吐量服务）
   - TGI（Hugging Face Text Generation Inference）
   - TensorRT-LLM（NVIDIA 优化）

4. **探索 prompt 工程**：
   - 系统 prompts
   - Few-shot 学习
   - Chain-of-thought prompting

## 资源

- [Hugging Face Transformers 文档](https://huggingface.co/docs/transformers)
- [GPT-2 模型卡](https://huggingface.co/gpt2)
- [LLM 推理优化](https://huggingface.co/docs/transformers/llm_tutorial)
- [vLLM 文档](https://vllm.readthedocs.io/)

---

**恭喜！** 你已经部署了你的第一个 LLM！你现在了解了运行语言模型的基础知识以及服务它们所需的基础设施需求。
