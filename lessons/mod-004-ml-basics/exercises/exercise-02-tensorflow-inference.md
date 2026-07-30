# 练习 02：TensorFlow 模型推理

## 练习概述

**目标**：加载预训练的 TensorFlow/Keras 模型，对图像运行推理，测量性能，并探索 TensorFlow Serving 基础。

**难度**：初级
**预计时间**：2-3 小时
**前置要求**：讲座 03（TensorFlow 基础）

**你将学到**：
- 从 keras.applications 加载预训练模型
- 为 TensorFlow 模型预处理图像
- 使用 TensorFlow/Keras API 运行推理
- 测量推理延迟和吞吐量
- 了解 TensorFlow Serving
- SavedModel 格式和模型导出

---

## 学习目标

到本练习结束时，你将能够：

✅ 从 keras.applications 加载预训练 Keras 模型
✅ 使用 TensorFlow 预处理函数预处理图像
✅ 使用 Keras 和 TensorFlow API 运行推理
✅ 测量和优化 TensorFlow 推理性能
✅ 以 SavedModel 格式导出模型
✅ 理解 TensorFlow Serving 基础
✅ 比较 TensorFlow 和 PyTorch 推理模式

---

## 第 1 部分：环境设置

### 步骤 1.1：安装依赖

```bash
# 创建虚拟环境
python -m venv tensorflow_inference_env
source tensorflow_inference_env/bin/activate  # Windows: tensorflow_inference_env\Scripts\activate

# 安装 TensorFlow 和依赖
pip install tensorflow pillow requests numpy matplotlib
```

**注意**：TensorFlow 2.x 默认包含 Keras，因此无需单独安装 Keras。

### 步骤 1.2：验证安装

```python
# verify_installation.py
import tensorflow as tf
import numpy as np

print(f"TensorFlow version: {tf.__version__}")
print(f"Keras version: {tf.keras.__version__}")
print(f"Numpy version: {np.__version__}")

# 检查 GPU
gpus = tf.config.list_physical_devices('GPU')
if gpus:
    print(f"GPU available: {len(gpus)} device(s)")
    for gpu in gpus:
        print(f"  - {gpu.name}")
else:
    print("Running on CPU")

# 检查 GPU 内存配置
if gpus:
    try:
        # 防止 TensorFlow 分配所有 GPU 内存
        for gpu in gpus:
            tf.config.experimental.set_memory_growth(gpu, True)
        print("GPU memory growth enabled")
    except RuntimeError as e:
        print(f"Memory growth setting error: {e}")
```

**预期输出**：
```
TensorFlow version: 2.15.0
Keras version: 2.15.0
Numpy version: 1.24.3
GPU available: 1 device(s)
  - /physical_device:GPU:0
GPU memory growth enabled
```

### 步骤 1.3：下载测试图像

```python
# download_images.py
import urllib.request
import os

# 创建图像目录
os.makedirs("test_images", exist_ok=True)

# 下载样本图像
images = {
    "elephant.jpg": "https://upload.wikimedia.org/wikipedia/commons/3/37/African_Bush_Elephant.jpg",
    "tiger.jpg": "https://upload.wikimedia.org/wikipedia/commons/5/56/Tiger.50.jpg",
    "airplane.jpg": "https://upload.wikimedia.org/wikipedia/commons/2/2d/Boeing_777-31H_Air_China_B-2086.jpg"
}

for filename, url in images.items():
    filepath = os.path.join("test_images", filename)
    print(f"Downloading {filename}...")
    try:
        urllib.request.urlretrieve(url, filepath)
        print(f"  ✓ Saved to {filepath}")
    except Exception as e:
        print(f"  ✗ Error: {e}")

print("\nAll images downloaded!")
print(f"Images location: {os.path.abspath('test_images')}")
```

✅ **检查点**：你应该已经设置好 TensorFlow 环境并下载了 3 张测试图像。

---

## 第 2 部分：加载预训练模型

### 步骤 2.1：从 keras.applications 加载 MobileNetV2

```python
# load_model.py
import tensorflow as tf
from tensorflow import keras

# 加载预训练 MobileNetV2
print("Loading MobileNetV2...")
model = keras.applications.MobileNetV2(
    weights='imagenet',  # 使用预训练的 ImageNet 权重
    include_top=True,    # 包含分类头
    input_shape=(224, 224, 3)
)

print(f"Model loaded: {model.name}")
print(f"Input shape: {model.input_shape}")
print(f"Output shape: {model.output_shape}")

# 统计参数
trainable_params = sum([tf.size(w).numpy() for w in model.trainable_weights])
total_params = sum([tf.size(w).numpy() for w in model.weights])

print(f"Trainable parameters: {trainable_params:,}")
print(f"Total parameters: {total_params:,}")
```

**预期输出**：
```
Downloading data from https://storage.googleapis.com/tensorflow/keras-applications/mobilenet_v2/mobilenet_v2_weights_tf_dim_ordering_tf_kernels_1.0_224.h5
14536120/14536120 [==============================] - 2s 0us/step
Model loaded: mobilenetv2_1.00_224
Input shape: (None, 224, 224, 3)
Output shape: (None, 1000)
Trainable parameters: 3,538,984
Total parameters: 3,538,984
```

### 步骤 2.2：理解模型架构

**任务**：检查模型结构以理解其层。

```python
# examine_model.py
import tensorflow as tf
from tensorflow import keras

model = keras.applications.MobileNetV2(weights='imagenet')

# 打印模型摘要
print("Model Architecture Summary:")
model.summary()

print("\n" + "="*60)

# 按类型统计层数
layer_counts = {}
for layer in model.layers:
    layer_type = layer.__class__.__name__
    layer_counts[layer_type] = layer_counts.get(layer_type, 0) + 1

print("\nLayer Counts:")
for layer_type, count in sorted(layer_counts.items()):
    print(f"  {layer_type}: {count}")

# 模型输入/输出信息
print("\nModel Expectations:")
print(f"  Input shape: {model.input_shape}")
print(f"  Input dtype: {model.input.dtype}")
print(f"  Output shape: {model.output_shape}")
print(f"  Output classes: 1000 (ImageNet)")
print(f"  Expected input range: [0, 255] or [0, 1] depending on preprocessing")

# 获取前几层和后几层
print("\nFirst 5 layers:")
for i, layer in enumerate(model.layers[:5]):
    print(f"  {i+1}. {layer.name} ({layer.__class__.__name__})")

print("\nLast 5 layers:")
for i, layer in enumerate(model.layers[-5:]):
    print(f"  {len(model.layers)-5+i+1}. {layer.name} ({layer.__class__.__name__})")
```

**要回答的问题**（写下你的答案）：
1. MobileNetV2 总共有多少层？
2. 预期的输入形状是什么？
3. MobileNetV2 使用了哪些类型的层？
4. 这与 PyTorch 中的 ResNet50 相比如何？

✅ **检查点**：模型加载成功，你理解了其架构。

---

## 第 3 部分：图像预处理

### 步骤 3.1：实现预处理管道

```python
# preprocess.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
from PIL import Image

def load_and_preprocess_image(image_path, target_size=(224, 224)):
    """
    加载并预处理图像用于 MobileNetV2。

    Args:
        image_path: 图像文件路径
        target_size: 目标图像大小（高度, 宽度）

    Returns:
        preprocessed_image: 预处理后的图像 tensor，可供模型使用
    """
    # 方法 1：使用 keras.preprocessing（更简单）
    print(f"Loading image: {image_path}")

    # 加载图像
    img = keras.preprocessing.image.load_img(
        image_path,
        target_size=target_size
    )
    print(f"Original image size: {img.size}")

    # 转换为数组
    img_array = keras.preprocessing.image.img_to_array(img)
    print(f"Image array shape: {img_array.shape}")
    print(f"Array dtype: {img_array.dtype}")
    print(f"Value range: [{img_array.min():.1f}, {img_array.max():.1f}]")

    # 添加 batch 维度：(224, 224, 3) -> (1, 224, 224, 3)
    img_batch = np.expand_dims(img_array, axis=0)
    print(f"Batch shape: {img_batch.shape}")

    # 应用模型特定的预处理
    preprocessed = keras.applications.mobilenet_v2.preprocess_input(img_batch)
    print(f"After preprocessing range: [{preprocessed.min():.3f}, {preprocessed.max():.3f}]")

    return preprocessed

# 测试预处理
preprocessed_image = load_and_preprocess_image("test_images/elephant.jpg")
```

**预期输出**：
```
Loading image: test_images/elephant.jpg
Original image size: (224, 224)
Image array shape: (224, 224, 3)
Array dtype: float32
Value range: [0.0, 255.0]
Batch shape: (1, 224, 224, 3)
After preprocessing range: [-1.000, 1.000]
```

### 步骤 3.2：使用 TensorFlow API 的替代预处理

```python
# preprocess_tf.py
import tensorflow as tf

def preprocess_with_tf(image_path, target_size=(224, 224)):
    """
    使用纯 TensorFlow API 预处理图像（更灵活）。

    这种方法更适合生产管道和 TensorFlow Serving。
    """
    # 读取图像文件
    image_raw = tf.io.read_file(image_path)

    # 解码图像（自动检测格式）
    image = tf.image.decode_image(image_raw, channels=3)

    # 确保设置了形状（某些操作需要）
    image.set_shape([None, None, 3])

    print(f"Original shape: {image.shape}")
    print(f"Original dtype: {image.dtype}")

    # 调整图像大小
    image = tf.image.resize(image, target_size)
    print(f"Resized shape: {image.shape}")

    # 如有必要，转换为 float32
    image = tf.cast(image, tf.float32)

    # 添加 batch 维度
    image = tf.expand_dims(image, 0)
    print(f"Batch shape: {image.shape}")

    # MobileNetV2 预处理：缩放到 [-1, 1]
    image = tf.keras.applications.mobilenet_v2.preprocess_input(image)
    print(f"Preprocessed range: [{tf.reduce_min(image):.3f}, {tf.reduce_max(image):.3f}]")

    return image

# 测试 TensorFlow 预处理
tf_preprocessed = preprocess_with_tf("test_images/elephant.jpg")
```

### 步骤 3.3：理解预处理差异

**任务**：回答这些关于 TensorFlow 预处理的问题：

1. **`preprocess_input` 对 MobileNetV2 做了什么？**
   - 提示：它将像素值从 [0, 255] 缩放到 [-1, 1]

2. **为什么 TensorFlow 使用 channels-last 格式（H, W, C）？**
   - 提示：与 PyTorch 的 channels-first（C, H, W）比较

3. **什么时候你会使用纯 TensorFlow API 而不是 Keras 工具？**
   - 提示：考虑 TensorFlow Serving 等部署场景

✅ **检查点**：你能使用多种方法预处理图像了。

---

## 第 4 部分：运行推理

### 步骤 4.1：使用 Keras 的基本推理

```python
# inference.py
import tensorflow as tf
from tensorflow import keras
import numpy as np

# 加载模型
print("Loading model...")
model = keras.applications.MobileNetV2(weights='imagenet')

# 加载并预处理图像
image_path = "test_images/elephant.jpg"
img = keras.preprocessing.image.load_img(image_path, target_size=(224, 224))
img_array = keras.preprocessing.image.img_to_array(img)
img_batch = np.expand_dims(img_array, axis=0)
preprocessed = keras.applications.mobilenet_v2.preprocess_input(img_batch)

print(f"Input shape: {preprocessed.shape}")
print(f"Input dtype: {preprocessed.dtype}")

# 运行推理（Keras API - 高级）
print("\nRunning inference...")
predictions = model.predict(preprocessed, verbose=0)

print(f"Output shape: {predictions.shape}")
print(f"Output dtype: {predictions.dtype}")
print(f"Output sample (first 5 values): {predictions[0, :5]}")
print(f"Predictions sum to: {predictions.sum():.6f}")  # softmax 后应约为 1
```

**预期输出**：
```
Loading model...
Input shape: (1, 224, 224, 3)
Input dtype: float32

Running inference...
Output shape: (1, 1000)
Output dtype: float32
Output sample (first 5 values): [1.234e-05 3.456e-06 2.345e-04 1.123e-05 4.567e-07]
Predictions sum to: 1.000000
```

### 步骤 4.2：获取带标签的 Top 预测

```python
# get_predictions.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import json
import urllib.request

# 下载 ImageNet 类别标签
print("Downloading ImageNet labels...")
labels_url = "https://storage.googleapis.com/download.tensorflow.org/data/imagenet_class_index.json"
urllib.request.urlretrieve(labels_url, "imagenet_class_index.json")

# 加载标签
with open("imagenet_class_index.json") as f:
    class_idx = json.load(f)
    # 转换为类别名称列表
    labels = [class_idx[str(i)][1] for i in range(len(class_idx))]

print(f"Loaded {len(labels)} class labels")

# 加载模型
model = keras.applications.MobileNetV2(weights='imagenet')

# 加载并预处理图像
image_path = "test_images/elephant.jpg"
img = keras.preprocessing.image.load_img(image_path, target_size=(224, 224))
img_array = keras.preprocessing.image.img_to_array(img)
img_batch = np.expand_dims(img_array, axis=0)
preprocessed = keras.applications.mobilenet_v2.preprocess_input(img_batch)

# 运行推理
predictions = model.predict(preprocessed, verbose=0)

# 解码预测结果（TensorFlow/Keras 提供了辅助函数）
decoded_predictions = keras.applications.mobilenet_v2.decode_predictions(
    predictions,
    top=5
)

# 显示结果
print(f"\nClassifying: {image_path}")
print("Top 5 Predictions:")
for i, (imagenet_id, class_name, probability) in enumerate(decoded_predictions[0]):
    print(f"  {i+1}. {class_name:<30} {probability*100:>6.2f}% (ID: {imagenet_id})")
```

**预期输出**：
```
Downloading ImageNet labels...
Loaded 1000 class labels

Classifying: test_images/elephant.jpg
Top 5 Predictions:
  1. African_elephant            92.45% (ID: n02504458)
  2. tusker                      6.32% (ID: n02504013)
  3. Indian_elephant             1.12% (ID: n02504013)
  4. triceratops                 0.08% (ID: n01704323)
  5. ram                         0.02% (ID: n02412080)
```

### 步骤 4.3：使用 TensorFlow 的函数式 API 进行推理

```python
# inference_functional.py
import tensorflow as tf
from tensorflow import keras
import numpy as np

# 加载模型
model = keras.applications.MobileNetV2(weights='imagenet')

# 创建预处理 + 推理函数
@tf.function  # 该装饰器将函数编译为图以获得更好的性能
def predict_image(image_path):
    """
    使用 tf.function 的完整预处理 + 推理管道。
    这更接近 TensorFlow Serving 的工作方式。
    """
    # 读取并预处理
    image_raw = tf.io.read_file(image_path)
    image = tf.image.decode_image(image_raw, channels=3)
    image.set_shape([None, None, 3])
    image = tf.image.resize(image, [224, 224])
    image = tf.cast(image, tf.float32)
    image = tf.expand_dims(image, 0)
    image = keras.applications.mobilenet_v2.preprocess_input(image)

    # 运行推理
    predictions = model(image, training=False)

    return predictions

# 测试函数
print("Running tf.function inference...")
predictions = predict_image("test_images/elephant.jpg")
print(f"Predictions shape: {predictions.shape}")
print(f"Top prediction probability: {tf.reduce_max(predictions).numpy():.4f}")

# 获取 top 5 类别
top5_idx = tf.argsort(predictions[0], direction='DESCENDING')[:5]
top5_probs = tf.gather(predictions[0], top5_idx)

print("\nTop 5 class indices and probabilities:")
for idx, prob in zip(top5_idx.numpy(), top5_probs.numpy()):
    print(f"  Class {idx}: {prob:.4f}")
```

**为什么使用 `@tf.function`？**
- 将 Python 代码编译为 TensorFlow 图
- 重复调用时速度更快
- 生产部署必不可少
- 类似于 PyTorch 中的 TorchScript

✅ **检查点**：你能使用多种 TensorFlow API 运行推理了！

---

## 第 5 部分：性能测量

### 步骤 5.1：测量推理延迟

```python
# measure_latency.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import time

# 加载模型
print("Loading model...")
model = keras.applications.MobileNetV2(weights='imagenet')

# 准备测试输入
dummy_input = np.random.rand(1, 224, 224, 3).astype(np.float32)
dummy_input = keras.applications.mobilenet_v2.preprocess_input(dummy_input)

# 预热（前几次运行由于 TensorFlow 图编译而较慢）
print("Warming up (this takes a moment)...")
for _ in range(20):
    _ = model.predict(dummy_input, verbose=0)

print("Warmup complete!")

# 测量延迟
print("\nMeasuring inference latency...")
iterations = 100

start = time.time()
for _ in range(iterations):
    _ = model.predict(dummy_input, verbose=0)
elapsed = time.time() - start

avg_latency_ms = (elapsed / iterations) * 1000
throughput = iterations / elapsed

print(f"Average latency: {avg_latency_ms:.2f} ms")
print(f"Throughput: {throughput:.2f} images/second")
```

### 步骤 5.2：使用 tf.function 的优化推理

```python
# measure_latency_optimized.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import time

# 加载模型
model = keras.applications.MobileNetV2(weights='imagenet')

# 创建优化的推理函数
@tf.function
def predict_optimized(inputs):
    return model(inputs, training=False)

# 准备测试输入
dummy_input = tf.random.normal([1, 224, 224, 3])
dummy_input = keras.applications.mobilenet_v2.preprocess_input(dummy_input)

# 预热
print("Warming up with @tf.function...")
for _ in range(20):
    _ = predict_optimized(dummy_input)

# 测量延迟
iterations = 100
start = time.time()
for _ in range(iterations):
    _ = predict_optimized(dummy_input)
elapsed = time.time() - start

avg_latency_ms = (elapsed / iterations) * 1000
throughput = iterations / elapsed

print(f"Optimized average latency: {avg_latency_ms:.2f} ms")
print(f"Optimized throughput: {throughput:.2f} images/second")
print("\nNote: @tf.function should be significantly faster!")
```

### 步骤 5.3：批量推理性能

```python
# batch_inference.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import time

# 加载模型
model = keras.applications.MobileNetV2(weights='imagenet')

# 创建优化的推理函数
@tf.function
def predict_batch(inputs):
    return model(inputs, training=False)

# 测试不同的批大小
batch_sizes = [1, 4, 8, 16, 32]
print(f"{'Batch Size':<12} {'Latency/Batch (ms)':<20} {'Latency/Image (ms)':<20} {'Throughput (img/s)':<20}")
print("=" * 72)

for batch_size in batch_sizes:
    # 创建虚拟批次
    dummy_input = tf.random.normal([batch_size, 224, 224, 3])
    dummy_input = keras.applications.mobilenet_v2.preprocess_input(dummy_input)

    # 预热
    for _ in range(10):
        _ = predict_batch(dummy_input)

    # 测量
    iterations = 50
    start = time.time()

    for _ in range(iterations):
        _ = predict_batch(dummy_input)

    elapsed = time.time() - start
    latency_per_batch = (elapsed / iterations) * 1000
    latency_per_image = latency_per_batch / batch_size
    throughput = (batch_size * iterations) / elapsed

    print(f"{batch_size:<12} {latency_per_batch:<20.2f} {latency_per_image:<20.2f} {throughput:<20.2f}")
```

**预期输出（CPU）**：
```
Batch Size   Latency/Batch (ms)   Latency/Image (ms)   Throughput (img/s)
========================================================================
1            35.23                35.23                28.39
4            98.45                24.61                40.63
8            182.34               22.79                43.88
16           348.12               21.76                45.96
32           672.89               21.03                47.55
```

**关键观察**：更大的批次提高了吞吐量，但增加了每批次的延迟。

✅ **检查点**：你能测量和优化 TensorFlow 推理性能了。

---

## 第 6 部分：SavedModel 格式和 TensorFlow Serving 基础

### 步骤 6.1：以 SavedModel 格式导出模型

```python
# export_savedmodel.py
import tensorflow as tf
from tensorflow import keras
import os

# 加载模型
print("Loading model...")
model = keras.applications.MobileNetV2(weights='imagenet')

# 定义导出路径
export_path = "saved_models/mobilenetv2/1"
os.makedirs(export_path, exist_ok=True)

# 以 SavedModel 格式保存
print(f"Saving model to {export_path}...")
model.save(export_path, save_format='tf')

print("Model saved successfully!")

# 验证保存的模型结构
print("\nSavedModel directory structure:")
for root, dirs, files in os.walk("saved_models"):
    level = root.replace("saved_models", '').count(os.sep)
    indent = ' ' * 2 * level
    print(f"{indent}{os.path.basename(root)}/")
    subindent = ' ' * 2 * (level + 1)
    for file in files:
        print(f"{subindent}{file}")
```

**预期输出**：
```
Loading model...
Saving model to saved_models/mobilenetv2/1...
INFO:tensorflow:Assets written to: saved_models/mobilenetv2/1/assets
Model saved successfully!

SavedModel directory structure:
saved_models/
  mobilenetv2/
    1/
      assets/
      variables/
        variables.data-00000-of-00001
        variables.index
      saved_model.pb
      keras_metadata.pb
```

### 步骤 6.2：加载并使用 SavedModel

```python
# load_savedmodel.py
import tensorflow as tf
import numpy as np
from tensorflow import keras

# 加载 SavedModel
print("Loading SavedModel...")
loaded_model = tf.keras.models.load_model("saved_models/mobilenetv2/1")

print(f"Model loaded: {loaded_model.name}")
print(f"Input shape: {loaded_model.input_shape}")
print(f"Output shape: {loaded_model.output_shape}")

# 使用加载的模型测试推理
dummy_input = np.random.rand(1, 224, 224, 3).astype(np.float32)
dummy_input = keras.applications.mobilenet_v2.preprocess_input(dummy_input)

print("\nRunning inference with loaded model...")
predictions = loaded_model.predict(dummy_input, verbose=0)
print(f"Predictions shape: {predictions.shape}")
print(f"Predictions sum: {predictions.sum():.6f}")

print("\n✓ SavedModel works correctly!")
```

### 步骤 6.3：使用 CLI 检查 SavedModel

```python
# inspect_savedmodel.py
import subprocess
import os

# 使用 TensorFlow 的 saved_model_cli 检查模型
export_path = "saved_models/mobilenetv2/1"

print("Inspecting SavedModel with saved_model_cli...")
print("=" * 60)

# 显示所有签名
result = subprocess.run(
    ['saved_model_cli', 'show', '--dir', export_path, '--all'],
    capture_output=True,
    text=True
)

print(result.stdout)

print("\n" + "=" * 60)
print("Key Information:")
print("- signature_def: The model's input/output interface")
print("- serving_default: Default signature for TensorFlow Serving")
print("- inputs: Expected input tensors (name, shape, dtype)")
print("- outputs: Output tensors")
```

**替代方案：Python API 检查**

```python
# inspect_savedmodel_python.py
import tensorflow as tf

# 加载 SavedModel
loaded = tf.saved_model.load("saved_models/mobilenetv2/1")

print("Available signatures:")
print(list(loaded.signatures.keys()))

# 获取默认服务签名
serving_fn = loaded.signatures['serving_default']

print("\nServing signature details:")
print(f"Inputs: {serving_fn.structured_input_signature}")
print(f"Outputs: {serving_fn.structured_outputs}")
```

### 步骤 6.4：TensorFlow Serving 简介

**概念概述**（本练习不需要 Docker）

TensorFlow Serving 是一个生产级的模型服务系统。关键概念：

**用于服务的 SavedModel 结构**：
```
saved_models/
└── mobilenetv2/          # 模型名称
    ├── 1/                # 版本号
    │   ├── saved_model.pb
    │   ├── variables/
    │   └── assets/
    └── 2/                # 新版本（用于 A/B 测试、回滚）
        ├── saved_model.pb
        ├── variables/
        └── assets/
```

**TensorFlow Serving 的工作方式**：
1. 监控目录以发现模型版本
2. 自动加载最新版本
3. 通过 REST API 或 gRPC 提供预测服务
4. 支持批处理、模型版本管理和热交换

**REST API 请求示例**（概念性）：
```bash
# 如果 TensorFlow Serving 在 localhost:8501 上运行
curl -X POST http://localhost:8501/v1/models/mobilenetv2:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      {
        "input_1": [[[0.5, 0.5, 0.5], ...]]  # 224x224x3 image
      }
    ]
  }'
```

**基础设施工程师视角**：
- SavedModel 是部署格式（类似 ONNX 或 TorchScript）
- 版本管理内置在目录结构中
- REST API 使其与语言无关（任何客户端都可使用）
- 可以水平扩展（多个 TensorFlow Serving 实例）
- 与 Kubernetes, Docker, 负载均衡器集成

✅ **检查点**：你理解了 SavedModel 格式和 TensorFlow Serving 概念。

---

## 第 7 部分：完整应用

### 步骤 7.1：构建图像分类器 CLI

```python
# image_classifier.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import argparse
import time
import os
import json

class TensorFlowImageClassifier:
    """生产就绪的 TensorFlow 图像分类器"""

    def __init__(self, model_name='MobileNetV2', use_saved_model=False, model_path=None):
        """
        初始化分类器。

        Args:
            model_name: keras.applications 中的模型名称
            use_saved_model: 如果为 True，则从 SavedModel 路径加载
            model_path: SavedModel 路径（use_saved_model=True 时需要）
        """
        print(f"Initializing classifier...")

        # 加载模型
        if use_saved_model:
            if not model_path:
                raise ValueError("model_path required when use_saved_model=True")
            print(f"Loading SavedModel from {model_path}...")
            self.model = keras.models.load_model(model_path)
        else:
            print(f"Loading {model_name} from keras.applications...")
            model_class = getattr(keras.applications, model_name)
            self.model = model_class(weights='imagenet')

        # 获取预处理函数
        if use_saved_model:
            # 对 saved models 假设使用 MobileNetV2 预处理
            self.preprocess_fn = keras.applications.mobilenet_v2.preprocess_input
        else:
            preprocess_module = getattr(
                keras.applications,
                model_name.lower()
            )
            self.preprocess_fn = preprocess_module.preprocess_input

        # 加载 ImageNet 标签
        self._load_labels()

        # 创建优化的推理函数
        @tf.function
        def predict_fn(inputs):
            return self.model(inputs, training=False)

        self.predict_fn = predict_fn

        # 预热
        print("Warming up model...")
        dummy_input = tf.random.normal([1, 224, 224, 3])
        for _ in range(5):
            _ = self.predict_fn(dummy_input)

        print("Classifier ready!\n")

    def _load_labels(self):
        """加载 ImageNet 类别标签"""
        labels_path = "imagenet_class_index.json"

        if not os.path.exists(labels_path):
            print("Downloading ImageNet labels...")
            import urllib.request
            url = "https://storage.googleapis.com/download.tensorflow.org/data/imagenet_class_index.json"
            urllib.request.urlretrieve(url, labels_path)

        with open(labels_path) as f:
            class_idx = json.load(f)
            self.labels = [class_idx[str(i)][1] for i in range(len(class_idx))]

    def predict(self, image_path, top_k=5):
        """
        预测图像类别。

        Args:
            image_path: 图像文件路径
            top_k: 返回的 top 预测数量

        Returns:
            results: 包含 'class', 'probability', 'class_id' 的字典列表
            inference_time_ms: 推理时间（毫秒）
        """
        # 加载并预处理图像
        img = keras.preprocessing.image.load_img(image_path, target_size=(224, 224))
        img_array = keras.preprocessing.image.img_to_array(img)
        img_batch = np.expand_dims(img_array, axis=0)
        preprocessed = self.preprocess_fn(img_batch)

        # 转换为 tensor
        input_tensor = tf.convert_to_tensor(preprocessed, dtype=tf.float32)

        # 运行推理并计时
        start = time.time()
        predictions = self.predict_fn(input_tensor)
        inference_time_ms = (time.time() - start) * 1000

        # 获取 top k 预测
        top_k_idx = tf.argsort(predictions[0], direction='DESCENDING')[:top_k]
        top_k_probs = tf.gather(predictions[0], top_k_idx)

        # 格式化结果
        results = []
        for idx, prob in zip(top_k_idx.numpy(), top_k_probs.numpy()):
            results.append({
                'class': self.labels[idx],
                'probability': float(prob),
                'class_id': int(idx)
            })

        return results, inference_time_ms

def main():
    parser = argparse.ArgumentParser(description='TensorFlow Image Classifier')
    parser.add_argument('image', type=str, help='Path to image file')
    parser.add_argument('--model', type=str, default='MobileNetV2',
                       help='Model name from keras.applications')
    parser.add_argument('--top-k', type=int, default=5,
                       help='Number of top predictions')
    parser.add_argument('--saved-model', type=str, default=None,
                       help='Path to SavedModel (optional)')

    args = parser.parse_args()

    # 创建分类器
    use_saved = args.saved_model is not None
    classifier = TensorFlowImageClassifier(
        model_name=args.model,
        use_saved_model=use_saved,
        model_path=args.saved_model
    )

    # 运行预测
    print(f"Classifying: {args.image}")
    print("=" * 60)
    results, inference_time = classifier.predict(args.image, top_k=args.top_k)

    # 打印结果
    print(f"\nTop {args.top_k} Predictions (inference: {inference_time:.2f}ms):")
    for i, result in enumerate(results):
        print(f"  {i+1}. {result['class']:<30} {result['probability']*100:>6.2f}%")
    print()

if __name__ == "__main__":
    main()
```

**使用示例**：
```bash
# 基本使用
python image_classifier.py test_images/elephant.jpg

# 不同模型
python image_classifier.py test_images/tiger.jpg --model ResNet50

# 使用 SavedModel
python image_classifier.py test_images/airplane.jpg --saved-model saved_models/mobilenetv2/1

# 获取 top 10 预测
python image_classifier.py test_images/elephant.jpg --top-k 10
```

✅ **检查点**：你有了一个完整的、生产就绪的图像分类器！

---

## 第 8 部分：PyTorch vs TensorFlow 对比

### 关键差异总结

| 方面 | PyTorch | TensorFlow/Keras |
|--------|---------|------------------|
| **图像格式** | Channels-first（C, H, W） | Channels-last（H, W, C） |
| **模型加载** | `torch.hub.load()` | `keras.applications.ModelName()` |
| **评估模式** | 需要 `model.eval()` | 不需要（但使用 `training=False`） |
| **无梯度** | `with torch.no_grad():` | 推理时不需要 |
| **预处理** | 手动 transforms 管道 | 模型特定的 `preprocess_input()` |
| **Batch 维度** | `unsqueeze(0)` | `np.expand_dims(axis=0)` |
| **优化** | TorchScript | `@tf.function` |
| **服务** | TorchServe | TensorFlow Serving |
| **导出格式** | .pt, .pth, TorchScript | SavedModel (.pb) |

### 步骤 8.1：并排对比

```python
# framework_comparison.py
import time
import numpy as np

# TensorFlow
import tensorflow as tf
from tensorflow import keras

# PyTorch（如果可用）
try:
    import torch
    from torchvision import transforms, models
    PYTORCH_AVAILABLE = True
except ImportError:
    PYTORCH_AVAILABLE = False
    print("PyTorch not available, skipping comparison")

def benchmark_tensorflow():
    """对 TensorFlow 推理进行基准测试"""
    model = keras.applications.MobileNetV2(weights='imagenet')

    @tf.function
    def predict(x):
        return model(x, training=False)

    dummy_input = tf.random.normal([1, 224, 224, 3])

    # 预热
    for _ in range(10):
        _ = predict(dummy_input)

    # 测量
    iterations = 100
    start = time.time()
    for _ in range(iterations):
        _ = predict(dummy_input)
    elapsed = time.time() - start

    return (elapsed / iterations) * 1000

def benchmark_pytorch():
    """对 PyTorch 推理进行基准测试"""
    if not PYTORCH_AVAILABLE:
        return None

    device = torch.device('cpu')
    model = models.mobilenet_v2(pretrained=True)
    model.to(device)
    model.eval()

    dummy_input = torch.rand(1, 3, 224, 224).to(device)

    # 预热
    with torch.no_grad():
        for _ in range(10):
            _ = model(dummy_input)

    # 测量
    iterations = 100
    start = time.time()
    with torch.no_grad():
        for _ in range(iterations):
            _ = model(dummy_input)
    elapsed = time.time() - start

    return (elapsed / iterations) * 1000

# 运行基准测试
print("Benchmarking MobileNetV2 on CPU...")
print("=" * 60)

tf_latency = benchmark_tensorflow()
print(f"TensorFlow latency: {tf_latency:.2f} ms")

if PYTORCH_AVAILABLE:
    pt_latency = benchmark_pytorch()
    print(f"PyTorch latency:    {pt_latency:.2f} ms")

    if pt_latency < tf_latency:
        speedup = tf_latency / pt_latency
        print(f"\nPyTorch is {speedup:.2f}x faster")
    else:
        speedup = pt_latency / tf_latency
        print(f"\nTensorFlow is {speedup:.2f}x faster")
else:
    print("PyTorch not available for comparison")
```

---

## 拓展目标（可选挑战）

### 挑战 1：多模型比较

**任务**：创建一个脚本，在同一图像上比较多个 Keras 模型。

```python
def compare_models(image_path, models_to_test):
    """
    在同一图像上比较多个模型。

    Args:
        image_path: 图像路径
        models_to_test: keras.applications 中的模型名称列表

    Returns:
        比较结果
    """
    # TODO: 实现多模型比较
    # 提示：测试 MobileNetV2, ResNet50, EfficientNetB0
    # 比较：延迟、top-1 预测、置信度
    pass
```

### 挑战 2：使用 Docker 的 TensorFlow Serving

**任务**：使用 Docker 将你的 SavedModel 部署到 TensorFlow Serving。

```bash
# 拉取 TensorFlow Serving 镜像
docker pull tensorflow/serving

# 运行 TensorFlow Serving
docker run -p 8501:8501 \
  --mount type=bind,source=/path/to/saved_models/mobilenetv2,target=/models/mobilenetv2 \
  -e MODEL_NAME=mobilenetv2 \
  -t tensorflow/serving

# 使用 curl 测试
curl -X POST http://localhost:8501/v1/models/mobilenetv2:predict \
  -H "Content-Type: application/json" \
  -d @request.json
```

### 挑战 3：混合精度推理

**任务**：实现 FP16 混合精度推理以获得更快的 GPU 性能。

```python
# 启用混合精度
from tensorflow.keras import mixed_precision
policy = mixed_precision.Policy('mixed_float16')
mixed_precision.set_global_policy(policy)

# TODO: 在 GPU 上测试性能提升
```

---

## 验证和测试

### 测试你的实现

```python
# test_classifier.py
import os
import tensorflow as tf

def test_tensorflow_classifier():
    """测试 TensorFlow 分类器实现"""
    from image_classifier import TensorFlowImageClassifier

    # 测试初始化
    classifier = TensorFlowImageClassifier(model_name='MobileNetV2')
    assert classifier.model is not None, "Model should be loaded"
    assert len(classifier.labels) == 1000, "Should have 1000 ImageNet labels"

    # 测试预测
    test_image = "test_images/elephant.jpg"
    if os.path.exists(test_image):
        results, latency = classifier.predict(test_image, top_k=5)

        assert len(results) == 5, "Should return 5 predictions"
        assert results[0]['probability'] >= results[1]['probability'], "Should be sorted"
        assert latency > 0, "Latency should be positive"

        # 检查概率范围
        for result in results:
            assert 0 <= result['probability'] <= 1, "Probabilities should be in [0, 1]"

        print("✅ All tests passed!")
    else:
        print(f"⚠ Test image not found: {test_image}")

if __name__ == "__main__":
    test_tensorflow_classifier()
```

---

## 反思问题

回答这些问题以巩固你的学习：

1. **TensorFlow 的 channels-last 格式与 PyTorch 的 channels-first 有何不同？**
   - 你的答案：

2. **`@tf.function` 的目的是什么，它如何提高性能？**
   - 你的答案：

3. **为什么 SavedModel 格式对生产部署很重要？**
   - 你的答案：

4. **TensorFlow Serving 和 TorchServe 之间有哪些关键区别？**
   - 你的答案：

5. **什么时候你会选择 TensorFlow 而不是 PyTorch 进行推理？**
   - 你的答案：

---

## 总结

**你完成了什么**：
✅ 加载并使用预训练 Keras 模型
✅ 使用 TensorFlow API 预处理图像
✅ 使用 Keras 和 TensorFlow 函数式 API 运行推理
✅ 使用 `@tf.function` 测量和优化推理性能
✅ 以 SavedModel 格式导出模型
✅ 理解 TensorFlow Serving 基础
✅ 构建了完整的、生产就绪的图像分类器
✅ 比较了 TensorFlow 和 PyTorch 的方法

**获得的关键技能**：
- 从 keras.applications 加载模型
- 使用 TensorFlow/Keras 工具进行图像预处理
- SavedModel 导出和检查
- 使用 `@tf.function` 进行性能优化
- TensorFlow Serving 概念
- 框架比较（TensorFlow vs PyTorch）

**基础设施工程师要点**：
- SavedModel 是 TensorFlow 的部署格式（与语言无关）
- TensorFlow Serving 提供生产就绪的模型服务
- 版本管理内置在目录结构中
- `@tf.function` 编译为图以获得更好的性能
- TensorFlow 与云平台（GCP, AWS, Azure）集成良好

---

## 下一步

1. **完成两个练习**：比较你的 PyTorch 和 TensorFlow 实现
2. **尝试 TensorFlow Serving**：使用 Docker 部署模型（可选）
3. **实验**：测试不同的 keras.applications 模型
4. **进入项目**：在模块项目中应用这些概念

---

**练习版本**: 1.0
**最后更新**: 2025 年 10 月
**预计时间**: 2-3 小时
**难度**: 初级
