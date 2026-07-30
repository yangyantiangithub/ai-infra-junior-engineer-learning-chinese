# 讲座 03：面向基础设施工程师的 TensorFlow 基础

## 学习目标

到本讲座结束时，你将能够：

- 理解 TensorFlow/Keras 架构和生态系统
- 使用 TensorFlow 模型格式（SavedModel, H5, TFLite）
- 从多种来源加载预训练 TensorFlow 模型
- 在生产环境中使用 TensorFlow 模型运行推理
- 使用 TensorFlow Serving 部署模型
- 从基础设施角度比较 TensorFlow 和 PyTorch
- 排查常见的 TensorFlow 部署问题

**时长**: 8-10 小时
**难度**: 初级到中级
**前置要求**: Module 001（Python），讲座 01（ML 概述），讲座 02（PyTorch 基础）

---

## 1. TensorFlow 简介

### 什么是 TensorFlow？

TensorFlow 是由 Google 开发的开源机器学习框架。它是目前最成熟和最具生产就绪性的 ML 框架之一。

**行业采用情况**：
- 被以下公司使用：Google, Airbnb, Uber, Twitter, Dropbox
- 为以下产品提供动力：Google Search, Gmail, Google Photos, YouTube 推荐
- 在以下领域实力强劲：生产部署、移动/边缘 ML、企业环境

**基础设施视角**：
- **静态计算图**：更多优化机会
- **TensorFlow Serving**：经过实战考验的生产服务
- **TensorFlow Lite**：移动和边缘部署
- **TensorFlow.js**：基于浏览器的 ML
- **TPU 支持**：Google 的自定义 ML 加速器
- **企业特性**：更好的多语言支持（Python, C++, Java, Go）

### TensorFlow 和 Keras

**Keras** 是 TensorFlow 的高级 API（自 TF 2.0 起内置）：
```python
# 旧方式（TensorFlow 1.x）
import tensorflow as tf
import keras

# 现代方式（TensorFlow 2.x）
import tensorflow as tf
# Keras 已内置
from tensorflow import keras
```

**对于基础设施**：始终使用 `tensorflow.keras`，而非独立的 Keras。

---

## 2. TensorFlow 安装与环境

### 安装选项

```bash
# 仅 CPU 版本（轻量级）
pip install tensorflow

# 启用 GPU 版本（相同包，自动检测 GPU）
pip install tensorflow[and-cuda]

# 或指定版本
pip install tensorflow==2.14.0

# 验证安装
python -c "import tensorflow as tf; print(tf.__version__); print(tf.config.list_physical_devices('GPU'))"
```

### 验证安装

```python
import tensorflow as tf

# 检查 TensorFlow 版本
print(f"TensorFlow version: {tf.__version__}")

# 检查 GPU 可用性
gpus = tf.config.list_physical_devices('GPU')
print(f"GPUs available: {len(gpus)}")

if gpus:
    for gpu in gpus:
        print(f"  {gpu.name}: {gpu.device_type}")

    # 获取 GPU 详情
    gpu_details = tf.config.experimental.get_device_details(gpus[0])
    print(f"GPU compute capability: {gpu_details.get('compute_capability')}")

# 检查 CPU 能力
print(f"Built with CUDA: {tf.test.is_built_with_cuda()}")
print(f"GPU support: {tf.test.is_gpu_available()}")
```

**预期输出（有 GPU）**：
```
TensorFlow version: 2.14.0
GPUs available: 1
  /physical_device:GPU:0: GPU
GPU compute capability: (8, 0)
Built with CUDA: True
GPU support: True
```

### 基础设施注意事项

**Docker 镜像**：
```dockerfile
# 官方 TensorFlow 镜像
FROM tensorflow/tensorflow:2.14.0-gpu

# 或仅 CPU
FROM tensorflow/tensorflow:2.14.0

# 自定义构建
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04
RUN pip install tensorflow==2.14.0
```

**依赖项**：
- **tensorflow**：核心库（约 500MB）
- **tensorboard**：可视化工具（已包含）
- **tensorflow-datasets**：常用数据集（可选）
- **tensorflow-hub**：预训练模型（可选）

---

## 3. TensorFlow 模型格式

TensorFlow 支持多种模型格式，各有不同的用例：

### 格式 1：SavedModel（推荐用于生产）

**是什么**：包含权重和计算图的完整 TensorFlow 程序。

```python
import tensorflow as tf

# 创建一个简单模型
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(10, activation='softmax')
])

# 保存为 SavedModel（基于目录的格式）
model.save('my_model/')

# 目录结构：
# my_model/
#   ├── saved_model.pb          # 计算图
#   ├── variables/
#   │   ├── variables.data-*    # 权重值
#   │   └── variables.index     # 权重索引
#   └── assets/                 # 额外文件（可选）

# 加载 SavedModel
loaded_model = tf.keras.models.load_model('my_model/')
```

**特性**：
- ✅ **生产就绪**：与 TensorFlow Serving 兼容
- ✅ **与语言无关**：可在 C++, Java, Go 中加载
- ✅ **完整**：包含图和权重
- ✅ **版本化**：支持模型版本管理
- ❌ **体积较大**：比 H5 更多文件

### 格式 2：HDF5（.h5 或 .keras）

**是什么**：包含架构和权重的单个文件。

```python
# 保存为 H5
model.save('my_model.h5')

# 或使用 .keras 扩展名（TF 2.x 推荐）
model.save('my_model.keras')

# 加载 H5
loaded_model = tf.keras.models.load_model('my_model.h5')
```

**特性**：
- ✅ **单个文件**：易于管理
- ✅ **紧凑**：比 SavedModel 小
- ✅ **Keras 专用**：仅适用于 Keras 模型
- ❌ **不适用于 TF Serving**：不能与 TensorFlow Serving 一起使用
- ❌ **仅 Python**：在其他语言中更难以加载

### 格式 3：TensorFlow Lite（.tflite）

**是什么**：针对移动和边缘设备优化的格式。

```python
# 转换为 TFLite
converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite_model = converter.convert()

# 保存
with open('model.tflite', 'wb') as f:
    f.write(tflite_model)

# 加载并运行推理
interpreter = tf.lite.Interpreter(model_path='model.tflite')
interpreter.allocate_tensors()

input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# 运行推理
interpreter.set_tensor(input_details[0]['index'], input_data)
interpreter.invoke()
output = interpreter.get_tensor(output_details[0]['index'])
```

**特性**：
- ✅ **优化**：更小的体积，更快的推理
- ✅ **移动就绪**：支持 Android, iOS
- ✅ **边缘设备**：Raspberry Pi, 微控制器
- ❌ **操作有限**：并非所有 TensorFlow 操作都支持
- ❌ **需要转换**：部署时需要额外步骤

### 格式对比

| 格式 | 大小 | 速度 | TF Serving | 移动 | 用例 |
|--------|------|-------|------------|--------|----------|
| **SavedModel** | 大 | 快 | ✅ 是 | ❌ 否 | 生产服务器 |
| **H5/Keras** | 中等 | 快 | ❌ 否 | ❌ 否 | 开发、实验 |
| **TFLite** | 小 | 非常快 | ❌ 否 | ✅ 是 | 移动、边缘设备 |

**基础设施决策树**：
```
你是否部署到移动/边缘设备？
├─ 是 → 使用 TFLite
└─ 否 → 你是否使用 TensorFlow Serving？
    ├─ 是 → 使用 SavedModel
    └─ 否 → 使用 SavedModel（面向未来）或 H5（方便）
```

---

## 4. 加载预训练模型

### 从 TensorFlow Hub 加载

```python
import tensorflow as tf
import tensorflow_hub as hub

# 加载预训练模型
model = tf.keras.Sequential([
    hub.KerasLayer("https://tfhub.dev/google/imagenet/mobilenet_v2_100_224/classification/5")
])

# 构建模型（初始化权重）
model.build([None, 224, 224, 3])

print(f"Model inputs: {model.inputs}")
print(f"Model outputs: {model.outputs}")
```

### 从 Keras Applications 加载

```python
from tensorflow.keras.applications import ResNet50, VGG16, MobileNetV2

# 加载预训练 ResNet50
model = ResNet50(weights='imagenet')

print(f"Model name: {model.name}")
print(f"Number of layers: {len(model.layers)}")
print(f"Number of parameters: {model.count_params():,}")
```

**输出**：
```
Model name: resnet50
Number of layers: 175
Number of parameters: 25,636,712
```

### 从 HuggingFace（通过 transformers）加载

```python
from transformers import TFAutoModel, AutoTokenizer

# 加载 BERT 模型（TensorFlow 版本）
model_name = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = TFAutoModel.from_pretrained(model_name)

print(f"Model type: {type(model)}")
print(f"Model parameters: {model.num_parameters():,}")
```

### 从本地文件加载

```python
import tensorflow as tf

# 加载 SavedModel
model = tf.keras.models.load_model('my_model/')

# 加载 H5
model = tf.keras.models.load_model('my_model.h5')

# 仅加载权重（需要模型架构）
model = ResNet50(weights=None)  # 无权重的架构
model.load_weights('resnet50_weights.h5')

print("Model loaded successfully!")
```

### 基础设施最佳实践

```python
import os
import tensorflow as tf

# 设置自定义模型缓存目录
os.environ['TFHUB_CACHE_DIR'] = '/data/models/tfhub'

# 在容器构建期间预下载模型
model = tf.keras.applications.ResNet50(weights='imagenet')
model.save('/data/models/resnet50/')

# 在生产环境中，从本地路径加载
model = tf.keras.models.load_model('/data/models/resnet50/')
```

---

## 5. 使用 TensorFlow 运行推理

### 图像分类示例

```python
import tensorflow as tf
from tensorflow.keras.applications import ResNet50
from tensorflow.keras.preprocessing import image
from tensorflow.keras.applications.resnet50 import preprocess_input, decode_predictions
import numpy as np
import urllib.request

# 加载预训练模型
model = ResNet50(weights='imagenet')

# 下载并加载图像
url = "https://storage.googleapis.com/download.tensorflow.org/example_images/grace_hopper.jpg"
urllib.request.urlretrieve(url, "grace_hopper.jpg")
img = image.load_img("grace_hopper.jpg", target_size=(224, 224))

# 预处理图像
img_array = image.img_to_array(img)
img_array = np.expand_dims(img_array, axis=0)  # 添加 batch 维度
img_array = preprocess_input(img_array)

print(f"Input shape: {img_array.shape}")  # (1, 224, 224, 3)

# 运行推理
predictions = model.predict(img_array)

# 解码预测结果
decoded = decode_predictions(predictions, top=5)[0]

print("\nTop 5 predictions:")
for i, (imagenet_id, label, score) in enumerate(decoded):
    print(f"{i+1}. {label}: {score:.4f}")
```

**输出**：
```
Input shape: (1, 224, 224, 3)

Top 5 predictions:
1. military_uniform: 0.8324
2. suit: 0.0514
3. mortarboard: 0.0349
4. Windsor_tie: 0.0142
5. bow_tie: 0.0123
```

### 批量推理

```python
import tensorflow as tf
import numpy as np
import time

# 加载模型
model = tf.keras.applications.ResNet50(weights='imagenet')

# 对不同批大小进行基准测试
batch_sizes = [1, 8, 16, 32, 64]
input_shape = (224, 224, 3)

print(f"{'Batch Size':<12} {'Time (s)':<12} {'Throughput (samples/s)':<25} {'Latency/Sample (ms)':<20}")

for batch_size in batch_sizes:
    # 创建虚拟输入
    dummy_input = np.random.rand(batch_size, *input_shape).astype(np.float32)

    # 预热
    _ = model.predict(dummy_input, verbose=0)

    # 基准测试
    start = time.time()
    _ = model.predict(dummy_input, verbose=0)
    elapsed = time.time() - start

    throughput = batch_size / elapsed
    latency_per_sample = (elapsed / batch_size) * 1000

    print(f"{batch_size:<12} {elapsed:<12.4f} {throughput:<25.2f} {latency_per_sample:<20.2f}")
```

**典型输出（GPU）**：
```
Batch Size   Time (s)     Throughput (samples/s)    Latency/Sample (ms)
1            0.0234       42.74                     23.40
8            0.0456       175.44                    5.70
16           0.0789       202.79                    4.93
32           0.1456       219.78                    4.55
64           0.2845       224.96                    4.45
```

### 生产推理模式

```python
import tensorflow as tf
import numpy as np

class ImageClassifier:
    def __init__(self, model_path: str):
        """使用模型初始化分类器"""
        self.model = tf.keras.models.load_model(model_path)
        self.model.compile()  # 为推理进行优化

    def preprocess(self, image_data: np.ndarray) -> np.ndarray:
        """预处理输入图像"""
        # 调整大小为模型输入尺寸
        image_data = tf.image.resize(image_data, [224, 224])
        # 归一化
        image_data = tf.keras.applications.resnet50.preprocess_input(image_data)
        return image_data

    def predict(self, images: np.ndarray) -> np.ndarray:
        """对批量图像运行推理"""
        # 预处理
        processed = self.preprocess(images)

        # 预测
        predictions = self.model.predict(processed, verbose=0)

        return predictions

    def predict_single(self, image: np.ndarray) -> np.ndarray:
        """对单张图像运行推理"""
        # 添加 batch 维度
        image_batch = np.expand_dims(image, axis=0)
        predictions = self.predict(image_batch)
        return predictions[0]

# 使用
classifier = ImageClassifier('resnet50_model/')
dummy_image = np.random.rand(224, 224, 3).astype(np.float32)
result = classifier.predict_single(dummy_image)
print(f"Prediction shape: {result.shape}")
```

---

## 6. TensorFlow Serving：生产部署

### 什么是 TensorFlow Serving？

TensorFlow Serving 是一个灵活、高性能的机器学习模型服务系统，专为生产环境设计。

**特性**：
- **REST 和 gRPC API**：多协议支持
- **模型版本管理**：同时服务多个版本
- **热交换**：无需停机即可更新模型
- **批处理**：自动请求批处理
- **监控**：内置指标
- **多模型服务**：一个实例服务多个模型

### 安装 TensorFlow Serving

```bash
# 选项 1：Docker（推荐）
docker pull tensorflow/serving

# 选项 2：APT（Ubuntu/Debian）
echo "deb [arch=amd64] http://storage.googleapis.com/tensorflow-serving-apt stable tensorflow-model-server tensorflow-model-server-universal" | sudo tee /etc/apt/sources.list.d/tensorflow-serving.list
curl https://storage.googleapis.com/tensorflow-serving-apt/tensorflow-serving.release.pub.gpg | sudo apt-key add -
sudo apt update && sudo apt install tensorflow-model-server
```

### 准备用于服务的模型

```python
import tensorflow as tf

# 加载或创建模型
model = tf.keras.applications.MobileNetV2(weights='imagenet')

# 以 TensorFlow Serving 格式保存
model_version = 1
export_path = f'./serving_models/mobilenet/{model_version}'
model.save(export_path)

print(f"Model saved to {export_path}")

# 目录结构：
# serving_models/
#   └── mobilenet/
#       └── 1/              # 版本号
#           ├── saved_model.pb
#           └── variables/
```

### 运行 TensorFlow Serving

```bash
# 使用 Docker
docker run -p 8501:8501 \
  --mount type=bind,source=$(pwd)/serving_models/mobilenet,target=/models/mobilenet \
  -e MODEL_NAME=mobilenet \
  tensorflow/serving

# 服务器将启动并暴露：
# - REST API 在端口 8501
# - gRPC API 在端口 8500（默认）
```

### 发起推理请求

**REST API**：
```python
import requests
import json
import numpy as np

# 准备输入数据
data = np.random.rand(1, 224, 224, 3).tolist()

# 创建请求 payload
payload = {
    "instances": data
}

# 发送请求
url = "http://localhost:8501/v1/models/mobilenet:predict"
response = requests.post(url, json=payload)

# 解析响应
predictions = response.json()["predictions"]
print(f"Predictions shape: {np.array(predictions).shape}")
```

**gRPC API**（更快）：
```python
import grpc
import numpy as np
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import prediction_service_pb2_grpc
import tensorflow as tf

# 创建 gRPC 通道
channel = grpc.insecure_channel('localhost:8500')
stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)

# 准备请求
request = predict_pb2.PredictRequest()
request.model_spec.name = 'mobilenet'
request.model_spec.signature_name = 'serving_default'

# 添加输入数据
input_data = np.random.rand(1, 224, 224, 3).astype(np.float32)
request.inputs['input_1'].CopyFrom(
    tf.make_tensor_proto(input_data, shape=input_data.shape)
)

# 获取响应
result = stub.Predict(request, 10.0)  # 10 秒超时
print(f"Result: {result}")
```

### TensorFlow Serving 配置

```bash
# 模型配置文件（models.config）
cat > models.config <<EOF
model_config_list {
  config {
    name: 'mobilenet'
    base_path: '/models/mobilenet'
    model_platform: 'tensorflow'
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
  config {
    name: 'resnet'
    base_path: '/models/resnet'
    model_platform: 'tensorflow'
  }
}
EOF

# 使用配置文件运行
docker run -p 8501:8501 \
  --mount type=bind,source=$(pwd)/serving_models,target=/models \
  --mount type=bind,source=$(pwd)/models.config,target=/config/models.config \
  tensorflow/serving \
  --model_config_file=/config/models.config
```

### 生产 Docker 配置

```dockerfile
# Dockerfile for TensorFlow Serving with custom models
FROM tensorflow/serving:latest

# 复制模型
COPY serving_models/ /models/

# 复制配置
COPY models.config /config/models.config

# 暴露端口
EXPOSE 8500 8501

# 设置环境变量
ENV MODEL_NAME=mobilenet

# 启动 TensorFlow Serving
CMD ["tensorflow_model_server", \
     "--model_config_file=/config/models.config", \
     "--rest_api_port=8501", \
     "--allow_version_labels_for_unavailable_models=true"]
```

---

## 7. TensorFlow vs PyTorch 对比

### 从基础设施角度看

| 方面 | TensorFlow | PyTorch |
|--------|------------|---------|
| **生产服务** | TensorFlow Serving（成熟） | TorchServe（较新） |
| **模型格式** | SavedModel（目录） | State Dict / 完整模型（文件） |
| **移动部署** | TFLite（非常成熟） | PyTorch Mobile（发展中） |
| **多语言** | C++, Java, Go, JavaScript | 主要是 Python, C++ |
| **云集成** | 强（Google Cloud） | 发展中 |
| **企业采用** | 更高（更老、更稳定） | 快速增长 |
| **调试** | 较困难（静态图） | 更容易（动态图） |
| **学习曲线** | 较陡峭 | 较平缓 |
| **社区** | 庞大，企业导向 | 庞大，研究导向 |

### 代码对比

**PyTorch**：
```python
import torch

# 加载模型
model = torch.hub.load('pytorch/vision', 'resnet50', pretrained=True)
model.eval()

# 推理
with torch.no_grad():
    output = model(input_tensor)
```

**TensorFlow**：
```python
import tensorflow as tf

# 加载模型
model = tf.keras.applications.ResNet50(weights='imagenet')

# 推理（无需特殊上下文）
output = model.predict(input_array)
```

### 何时选择哪种？

**选择 TensorFlow 如果**：
- ✅ 需要成熟的生产服务（TensorFlow Serving）
- ✅ 广泛部署到移动/边缘设备
- ✅ 需要多语言支持的企业环境
- ✅ 大量使用 Google Cloud Platform
- ✅ 需要经过实战考验的生产工具

**选择 PyTorch 如果**：
- ✅ 研究导向的组织
- ✅ 开发时更容易调试
- ✅ 团队更喜欢 Pythonic、直观的 API
- ✅ 主要部署到服务器（而非移动设备）
- ✅ 大量使用 HuggingFace 生态系统

**两者都使用**：
- 将模型转换为 ONNX 以实现与框架无关的部署
- 使用数据科学团队偏好的任何框架
- 构建支持两者的基础设施

---

## 8. 常见问题和故障排查

### 问题 1：GPU 未检测到

**问题**：TensorFlow 明明有 GPU 却不使用。

**诊断**：
```python
import tensorflow as tf

# 检查 GPU
print(tf.config.list_physical_devices('GPU'))

# 启用日志
tf.debugging.set_log_device_placement(True)

# 测试 GPU 计算
with tf.device('/GPU:0'):
    a = tf.constant([[1.0, 2.0], [3.0, 4.0]])
    b = tf.constant([[5.0, 6.0], [7.0, 8.0]])
    c = tf.matmul(a, b)
    print(c)
```

**解决方案**：
```bash
# 1. 检查 CUDA 安装
nvidia-smi

# 2. 重新安装带有 GPU 支持的 TensorFlow
pip uninstall tensorflow
pip install tensorflow[and-cuda]

# 3. 检查 CUDA 和 cuDNN 版本是否匹配 TensorFlow 要求
# TensorFlow 2.14 需要 CUDA 11.8 和 cuDNN 8.7
```

### 问题 2：模型加载错误

**错误**：
```
ValueError: Unable to load model. Unable to restore object of class _tf_keras_metric
```

**解决方案**：
```python
# 加载时使用 custom_objects
import tensorflow as tf

# 如果需要，定义自定义对象
custom_objects = {'CustomMetric': CustomMetric}

# 使用自定义对象加载
model = tf.keras.models.load_model('model.h5', custom_objects=custom_objects)

# 或使用 compile=False 跳过指标加载
model = tf.keras.models.load_model('model.h5', compile=False)
```

### 问题 3：输入形状不匹配

**错误**：
```
ValueError: Input 0 of layer "resnet50" is incompatible with the layer: expected shape=(None, 224, 224, 3), found shape=(None, 3, 224, 224)
```

**问题**：TensorFlow 使用 channels-last（NHWC），PyTorch 使用 channels-first（NCHW）。

**解决方案**：
```python
import numpy as np

# PyTorch 格式：[N, C, H, W]
pytorch_format = np.random.rand(1, 3, 224, 224)

# 转换为 TensorFlow 格式：[N, H, W, C]
tensorflow_format = np.transpose(pytorch_format, (0, 2, 3, 1))

print(f"PyTorch shape: {pytorch_format.shape}")      # (1, 3, 224, 224)
print(f"TensorFlow shape: {tensorflow_format.shape}") # (1, 224, 224, 3)

# 使用 TensorFlow 格式
output = model.predict(tensorflow_format)
```

### 问题 4：内存不足

**错误**：
```
ResourceExhaustedError: OOM when allocating tensor
```

**解决方案**：
```python
import tensorflow as tf

# 1. 启用内存增长（不要一次性分配所有 GPU 内存）
gpus = tf.config.list_physical_devices('GPU')
if gpus:
    for gpu in gpus:
        tf.config.experimental.set_memory_growth(gpu, True)

# 2. 设置内存限制
if gpus:
    tf.config.set_logical_device_configuration(
        gpus[0],
        [tf.config.LogicalDeviceConfiguration(memory_limit=4096)]  # 4GB
    )

# 3. 减少批大小
batch_size = 8  # 替代 32

# 4. 使用混合精度
tf.keras.mixed_precision.set_global_policy('mixed_float16')
```

### 问题 5：TensorFlow Serving 连接问题

**错误**：
```
requests.exceptions.ConnectionError: HTTPConnectionPool
```

**诊断**：
```bash
# 检查 TensorFlow Serving 是否在运行
curl http://localhost:8501/v1/models/mobilenet

# 应返回模型元数据
```

**解决方案**：
```python
import requests
import time

def wait_for_server(url, timeout=60):
    """等待 TensorFlow Serving 准备就绪"""
    start = time.time()
    while time.time() - start < timeout:
        try:
            response = requests.get(url)
            if response.status_code == 200:
                print("Server is ready!")
                return True
        except requests.exceptions.ConnectionError:
            time.sleep(1)
    return False

# 等待服务器
server_url = "http://localhost:8501/v1/models/mobilenet"
if wait_for_server(server_url):
    # 发起推理请求
    pass
```

---

## 9. 性能优化

### 混合精度训练和推理

```python
import tensorflow as tf

# 启用混合精度
policy = tf.keras.mixed_precision.Policy('mixed_float16')
tf.keras.mixed_precision.set_global_policy(policy)

# 加载模型
model = tf.keras.applications.ResNet50(weights='imagenet')

# 模型将自动使用 float16 进行计算
# 在需要时使用 float32 以确保数值稳定性

print(f"Compute dtype: {policy.compute_dtype}")  # float16
print(f"Variable dtype: {policy.variable_dtype}")  # float32
```

**好处**：
- 在现代 GPU 上推理速度提升约 2 倍
- 内存使用减少约 50%
- 精度损失极小

### 带优化的 TensorFlow Lite 转换

```python
import tensorflow as tf

# 加载模型
model = tf.keras.applications.MobileNetV2(weights='imagenet')

# 创建转换器
converter = tf.lite.TFLiteConverter.from_keras_model(model)

# 启用优化
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# 可选：量化为 int8 以获得更小的模型
def representative_dataset():
    for _ in range(100):
        data = np.random.rand(1, 224, 224, 3).astype(np.float32)
        yield [data]

converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.uint8
converter.inference_output_type = tf.uint8

# 转换
tflite_model = converter.convert()

# 保存
with open('mobilenet_quantized.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"Original model size: {model.count_params() * 4 / 1024 / 1024:.2f} MB")
print(f"TFLite model size: {len(tflite_model) / 1024 / 1024:.2f} MB")
```

### 用于 XLA 的模型编译

```python
import tensorflow as tf

# 加载模型
model = tf.keras.applications.ResNet50(weights='imagenet')

# 使用 XLA 编译
model.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    jit_compile=True  # 启用 XLA 编译
)

# 初始编译后推理将更快
```

---

## 10. 资源估算

### 模型大小计算

```python
import tensorflow as tf

def get_model_size(model):
    """计算模型大小（MB）"""
    total_params = model.count_params()
    # 假设 float32（每个参数 4 字节）
    size_mb = (total_params * 4) / (1024 ** 2)
    return size_mb

model = tf.keras.applications.ResNet50(weights='imagenet')
print(f"Model size: {get_model_size(model):.2f} MB")
print(f"Total parameters: {model.count_params():,}")
```

### 内存需求

```python
import tensorflow as tf
import numpy as np

def estimate_memory(model, batch_size, input_shape):
    """估算推理所需内存"""
    # 模型大小
    model_size = get_model_size(model)

    # 输入大小
    input_size = np.prod(input_shape) * batch_size * 4 / (1024 ** 2)  # MB

    # 粗略估算：激活值约为模型大小的 2-3 倍
    activation_size = model_size * 2.5

    total_mb = model_size + input_size + activation_size
    return {
        'model_mb': model_size,
        'input_mb': input_size,
        'activation_mb': activation_size,
        'total_mb': total_mb,
        'recommended_gpu_gb': total_mb / 1024 * 1.5  # 50% 缓冲
    }

model = tf.keras.applications.ResNet50(weights='imagenet')
memory = estimate_memory(model, batch_size=32, input_shape=(224, 224, 3))

print("Memory Estimates:")
for key, value in memory.items():
    print(f"  {key}: {value:.2f}")
```

---

## 关键要点

### 对于基础设施工程师

1. **SavedModel 是生产格式**：用于 TensorFlow Serving 部署
2. **TensorFlow Serving 很成熟**：经过生产环境实战考验
3. **Channels-last 格式**：TensorFlow 使用 [N, H, W, C]，而非 [N, C, H, W]
4. **多种模型格式**：根据部署目标选择
5. **混合精度很强大**：只需少量更改即可实现 2 倍加速
6. **内存增长很重要**：启用以避免 OOM 错误
7. **REST API 很方便**：gRPC 在高吞吐量下更快
8. **模型版本管理内置**：TF Serving 原生支持 A/B 测试

### 生产清单

部署 TensorFlow 模型前：

- [ ] 模型以适当格式保存（服务器端使用 SavedModel）
- [ ] 模型已用预期的输入形状测试
- [ ] TensorFlow Serving 配置已测试
- [ ] 模型版本管理策略已定义
- [ ] 内存需求已估算
- [ ] 已启用 GPU 内存增长（如适用）
- [ ] 健康检查端点已实现
- [ ] 监控和日志配置完成
- [ ] 输入预处理已验证
- [ ] 输出后处理已验证

---

## 快速参考

### 基本命令

```python
# 安装检查
import tensorflow as tf
print(tf.__version__)
print(tf.config.list_physical_devices('GPU'))

# 加载模型
model = tf.keras.models.load_model('model/')
model = tf.keras.applications.ResNet50(weights='imagenet')

# 推理
predictions = model.predict(input_array)

# 启用 GPU 内存增长
gpus = tf.config.list_physical_devices('GPU')
for gpu in gpus:
    tf.config.experimental.set_memory_growth(gpu, True)
```

### TensorFlow Serving

```bash
# 运行 TensorFlow Serving
docker run -p 8501:8501 \
  --mount type=bind,source=/path/to/model,target=/models/mymodel \
  -e MODEL_NAME=mymodel \
  tensorflow/serving

# 测试 REST API
curl -X POST http://localhost:8501/v1/models/mymodel:predict \
  -d '{"instances": [[1.0, 2.0, 3.0]]}'
```

---

## 下一步

在下一讲中，我们将涵盖：
- **ONNX 格式**：与框架无关的部署
- **模型转换**：PyTorch ↔ TensorFlow ↔ ONNX
- **优化技术**：量化、剪枝、蒸馏

继续学习 `lecture-notes/04-model-formats.md`

---

**讲座版本**: 1.0
**最后更新**: 2025 年 10 月
**预计完成时间**: 8-10 小时
**难度**: 初级到中级
