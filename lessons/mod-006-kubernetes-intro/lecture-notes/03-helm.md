# Lecture 03: Helm - Kubernetes 包管理器

## 目录
1. [引言](#引言)
2. [为什么用 Helm？](#为什么用-helm)
3. [Helm 架构](#helm-架构)
4. [安装 Helm](#安装-helm)
5. [使用 Helm Chart](#使用-helm-chart)
6. [创建 Helm Chart](#创建-helm-chart)
7. [Helm 模板](#helm-模板)
8. [Values 和自定义](#values-和自定义)
9. [Chart 依赖](#chart-依赖)
10. [Helm 最佳实践](#helm-最佳实践)
11. [ML 应用的 Helm](#ml-应用的-helm)
12. [总结和关键要点](#总结和关键要点)

## 引言

管理 Kubernetes manifest 会很快变得复杂。随着应用增长，你将发现自己需要管理数十或数百个包含重复配置的 YAML 文件。**Helm** 通过为 Kubernetes 提供包管理来解决这个问题。

将 Helm 视为 Kubernetes 的 "apt"、"yum" 或 "npm"——它让你能够用单个命令定义、安装和升级哪怕是最复杂的 Kubernetes 应用。

### 学习目标

完成本讲座后，你将：
- 理解 Helm 是什么以及为什么它至关重要
- 安装和配置 Helm
- 使用仓库中的公共 Helm chart
- 为你的应用创建自定义 Helm chart
- 使用模板和 values 进行配置
- 管理 chart 依赖
- 应用 Helm 最佳实践
- 用 Helm 部署 ML 应用

### 前置要求
- Lecture 02：在 Kubernetes 上部署应用
- kubectl 已配置并正常工作
- 可访问 Kubernetes 集群
- 基本理解 Go 模板（有帮助但不是必需的）

## 为什么用 Helm？

### 没有 Helm 的问题

想象一下部署一个包含以下组件的 Web 应用：
- 前端 Deployment
- 后端 Deployment
- 数据库 StatefulSet
- 3 个 Service
- 2 个 ConfigMap
- 3 个 Secret
- Ingress
- ServiceAccount
- RBAC 规则

**没有 Helm**：
```bash
# 部署到开发环境
kubectl apply -f frontend-deployment-dev.yaml
kubectl apply -f backend-deployment-dev.yaml
kubectl apply -f database-statefulset-dev.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f database-service.yaml
kubectl apply -f configmap-dev.yaml
kubectl apply -f secrets-dev.yaml
kubectl apply -f ingress-dev.yaml
# ... 等等

# 部署到生产环境（不同的文件！）
kubectl apply -f frontend-deployment-prod.yaml
kubectl apply -f backend-deployment-prod.yaml
# ... 对所有资源重复，使用生产配置
```

**挑战**：
1. 为每个环境重复 YAML 文件
2. 手动管理配置差异
3. 没有部署版本控制
4. 难以回滚
5. 没有依赖管理
6. 难以共享和复用配置

### 使用 Helm 的解决方案

**使用 Helm**：
```bash
# 部署到开发环境
helm install myapp ./mychart -f values-dev.yaml

# 部署到生产环境
helm install myapp ./mychart -f values-prod.yaml

# 升级
helm upgrade myapp ./mychart

# 回滚
helm rollback myapp

# 卸载
helm uninstall myapp
```

**好处**：
1. **模板化**：单个模板，多个环境
2. **打包**：将相关资源捆绑在一起
3. **版本控制**：跟踪部署历史
4. **回滚**：轻松回滚到之前版本
5. **依赖**：管理 chart 依赖
6. **共享**：公共和私有 chart 仓库
7. **Hooks**：在安装/升级之前/之后运行 job

## Helm 架构

### Helm 组件

```
┌─────────────────────────────────────────┐
│          Helm 客户端 (CLI)              │
│  命令: install, upgrade, rollback       │
└────────────────┬────────────────────────┘
                 │
                 │ 使用 kubectl 配置
                 │
                 ▼
┌─────────────────────────────────────────┐
│      Kubernetes API Server              │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│         Kubernetes 资源                 │
│  (Deployments, Services, ConfigMaps)    │
└─────────────────────────────────────────┘
```

**关键点**：Helm 3 仅客户端（没有像 Helm 2 那样的 Tiller 服务）。它使用你的 kubectl 配置直接连接到 Kubernetes。

### 关键概念

**1. Chart**：包含所有资源定义的 Helm 包
```
mychart/
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认值
├── templates/          # Kubernetes manifest（模板化）
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── charts/             # 依赖的 chart
```

**2. Release**：部署到 Kubernetes 的 chart 实例
```bash
# 创建名为 "my-nginx" 的 release
helm install my-nginx bitnami/nginx
```

**3. Repository**：chart 的集合
```bash
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 搜索仓库
helm search repo nginx
```

**4. Values**：chart 的配置参数
```yaml
# values.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.21"
```

## 安装 Helm

### 安装方法

**macOS**：
```bash
brew install helm
```

**Linux**：
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Windows (Chocolatey)**：
```bash
choco install kubernetes-helm
```

**从二进制文件**：
```bash
# 从 https://github.com/helm/helm/releases 下载
wget https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz
tar -zxvf helm-v3.12.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### 验证安装

```bash
# 检查版本
helm version

# 应输出：
# version.BuildInfo{Version:"v3.12.0", ...}

# 获取帮助
helm help

# 列出可用命令
helm --help
```

### 配置 Helm

```bash
# 添加 stable 仓库
helm repo add stable https://charts.helm.sh/stable

# 添加 Bitnami 仓库（广泛使用）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新仓库
helm repo update

# 列出仓库
helm repo list

# 搜索 chart
helm search repo nginx
helm search repo postgres
```

## 使用 Helm Chart

### 查找 Chart

**Search Hub** (https://artifacthub.io)：
```bash
# 在 Artifact Hub 中搜索
helm search hub prometheus

# 在已添加的仓库中搜索
helm search repo prometheus
```

**流行的 Chart 仓库**：
- **Bitnami**：https://charts.bitnami.com/bitnami
- **Prometheus Community**：https://prometheus-community.github.io/helm-charts
- **Grafana**：https://grafana.github.io/helm-charts
- **Elastic**：https://helm.elastic.co
- **NGINX**：https://kubernetes.github.io/ingress-nginx

### 安装 Chart

```bash
# 使用默认值安装
helm install my-nginx bitnami/nginx

# 使用自定义名称安装
helm install my-web-server bitnami/nginx

# 在特定 namespace 中安装
helm install my-nginx bitnami/nginx --namespace production --create-namespace

# 使用自定义值安装
helm install my-nginx bitnami/nginx --set replicaCount=3

# 使用 values 文件安装
helm install my-nginx bitnami/nginx -f my-values.yaml

# Dry run（查看将要安装的内容）
helm install my-nginx bitnami/nginx --dry-run --debug

# 安装特定版本
helm install my-nginx bitnami/nginx --version 13.2.0
```

### 查看 Release 信息

```bash
# 列出 release
helm list
helm list --all-namespaces

# 获取 release 详情
helm status my-nginx

# 获取 release values
helm get values my-nginx

# 获取所有 values（包括默认值）
helm get values my-nginx --all

# 获取 manifest
helm get manifest my-nginx

# 获取 release 历史
helm history my-nginx
```

### 升级 Release

```bash
# 使用新值升级
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# 使用 values 文件升级
helm upgrade my-nginx bitnami/nginx -f new-values.yaml

# 升级到特定版本
helm upgrade my-nginx bitnami/nginx --version 13.2.1

# 强制重新创建资源
helm upgrade my-nginx bitnami/nginx --force

# 等待升级完成
helm upgrade my-nginx bitnami/nginx --wait --timeout 5m
```

### 回滚

```bash
# 回滚到上一个版本
helm rollback my-nginx

# 回滚到特定修订版本
helm rollback my-nginx 2

# 列出历史以查看修订版本
helm history my-nginx
```

### 卸载 Release

```bash
# 卸载 release
helm uninstall my-nginx

# 卸载但保留历史
helm uninstall my-nginx --keep-history

# 如果保留了历史可以回滚
helm rollback my-nginx
```

### 示例：安装 PostgreSQL

```bash
# 添加 Bitnami 仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 查看 chart values
helm show values bitnami/postgresql > postgres-values.yaml

# 创建自定义 values
cat > my-postgres-values.yaml <<EOF
auth:
  postgresPassword: "supersecret"
  database: "myapp"
primary:
  persistence:
    size: 20Gi
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
metrics:
  enabled: true
EOF

# 安装 PostgreSQL
helm install postgres bitnami/postgresql -f my-postgres-values.yaml

# 检查状态
helm status postgres
kubectl get pods

# 获取连接信息
export POSTGRES_PASSWORD=$(kubectl get secret postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
kubectl run postgres-client --rm -it --image=postgres:14 -- psql -h postgres-postgresql -U postgres
```

## 创建 Helm Chart

### Chart 结构

```bash
# 创建新 chart
helm create mychart

# 目录结构
mychart/
├── .helmignore         # 要忽略的文件（类似 .gitignore）
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认配置值
├── charts/             # Chart 依赖
└── templates/          # Kubernetes manifest
    ├── NOTES.txt       # 安装后显示的帮助文本
    ├── _helpers.tpl    # 模板辅助函数
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    └── tests/
        └── test-connection.yaml
```

### Chart.yaml

Chart.yaml 文件定义 chart 元数据：

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 0.1.0        # Chart 版本 (SemVer)
appVersion: "1.0.0"   # 应用版本

# 可选字段
keywords:
  - web
  - api
  - python
home: https://example.com
sources:
  - https://github.com/example/mychart
maintainers:
  - name: John Doe
    email: john@example.com
    url: https://example.com
icon: https://example.com/icon.png
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

**字段说明**：
- **apiVersion**：Chart API 版本（Helm 3 为 v2）
- **name**：Chart 名称
- **version**：Chart 版本（chart 变更时递增）
- **appVersion**：被部署应用的版本
- **type**：`application` 或 `library`

### values.yaml

默认配置值：

```yaml
# values.yaml
replicaCount: 2

image:
  repository: myapp
  pullPolicy: IfNotPresent
  tag: "1.0.0"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext:
  fsGroup: 2000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}
```

## Helm 模板

模板是带有 Go 模板语法的 Kubernetes manifest。

### 基本模板

**模板语法**：
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 8080
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

**内置对象**：
- **`.Release.Name`**：Release 名称
- **`.Release.Namespace`**：Namespace
- **`.Release.IsUpgrade`**：如果是升级则为 true
- **`.Release.IsInstall`**：如果是安装则为 true
- **`.Chart.Name`**：来自 Chart.yaml 的 chart 名称
- **`.Chart.Version`**：Chart 版本
- **`.Chart.AppVersion`**：应用版本
- **`.Values`**：来自 values.yaml 和覆盖配置的值
- **`.Files`**：访问 chart 中的文件
- **`.Capabilities`**：关于 Kubernetes 集群的信息

### 模板函数

**字符串函数**：
```yaml
# 引号
value: {{ .Values.name | quote }}

# 大写/小写
name: {{ .Values.name | upper }}
name: {{ .Values.name | lower }}

# 默认值
replicas: {{ .Values.replicaCount | default 3 }}

# 去除空白
name: {{ .Values.name | trim }}
```

**逻辑函数**：
```yaml
# If/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

{{- if .Values.postgresql.enabled }}
# PostgreSQL 配置
{{- else }}
# 外部数据库配置
{{- end }}

# With（设置作用域）
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}

# Range（循环）
{{- range .Values.environments }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

**YAML 函数**：
```yaml
# toYaml：转换为 YAML
resources:
  {{- toYaml .Values.resources | nindent 2 }}

# toJson：转换为 JSON
config: {{ .Values.config | toJson }}

# indent/nindent：缩进行
labels:
  {{- toYaml .Values.labels | nindent 4 }}
```

### 模板辅助函数 (_helpers.tpl)

可复用的模板片段：

```yaml
# templates/_helpers.tpl

{{/*
展开 chart 的名称。
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
创建默认的完全限定应用名称。
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
通用标签
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{ include "mychart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
选择器标签
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

**使用辅助函数**：
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
```

### 条件资源

仅在启用时创建资源：

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
        {{- end }}
  {{- end }}
{{- end }}
```

### NOTES.txt

安装后提供说明：

```
# templates/NOTES.txt
感谢安装 {{ .Chart.Name }}。

您的 release 名为 {{ .Release.Name }}。

要了解更多关于 release 的信息，请尝试：

  $ helm status {{ .Release.Name }}
  $ helm get all {{ .Release.Name }}

{{- if .Values.ingress.enabled }}
应用可通过以下地址访问：
{{- range .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
{{- end }}
{{- else }}
获取应用 URL，请运行：
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "mychart.name" . }}" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:80
  echo "访问 http://127.0.0.1:8080"
{{- end }}
```

## Values 和自定义

### 多个 Values 文件

```bash
# 基础 values
# values.yaml
replicaCount: 2
image:
  tag: "1.0.0"

# 开发环境覆盖
# values-dev.yaml
replicaCount: 1
resources:
  limits:
    memory: "256Mi"

# 生产环境覆盖
# values-prod.yaml
replicaCount: 5
resources:
  limits:
    memory: "1Gi"
ingress:
  enabled: true

# 安装
helm install myapp ./mychart -f values-dev.yaml
helm install myapp ./mychart -f values-prod.yaml
```

### Values 合并

Values 按以下顺序合并（后者的优先级更高）：
1. values.yaml 中的默认值
2. 来自 `-f file1.yaml` 的值
3. 来自 `-f file2.yaml` 的值（如果有多个 -f）
4. 来自 `--set key=value` 的值

```bash
# 所有这些值都会被合并
helm install myapp ./mychart \
  -f values.yaml \
  -f values-prod.yaml \
  --set replicaCount=10 \
  --set image.tag=1.2.3
```

### 设置 Values

**命令行**：
```bash
# 简单值
--set replicaCount=3

# 嵌套值
--set image.tag=1.2.0

# 列表
--set environments[0].name=DEV,environments[0].value=development

# 多个值
--set key1=val1,key2=val2

# 字符串值
--set name="My App"

# null 值
--set serviceAccount.name=null
```

### Values Schema 验证

定义 schema 以验证 values：

```yaml
# values.schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string"
        },
        "tag": {
          "type": "string",
          "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+$"
        }
      }
    }
  }
}
```

Helm 在 install/upgrade 期间根据此 schema 验证 values。

## Chart 依赖

### 定义依赖

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    tags:
      - database

  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    tags:
      - cache
```

### 管理依赖

```bash
# 下载依赖
helm dependency update ./mychart

# 这会创建：
# mychart/charts/postgresql-11.x.x.tgz
# mychart/charts/redis-17.x.x.tgz
# mychart/Chart.lock

# 列出依赖
helm dependency list ./mychart

# 构建依赖（从 charts/）
helm dependency build ./mychart
```

### 配置依赖

在父 chart 的 values.yaml 中配置依赖值：

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: secret
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 8Gi
```

### 条件依赖

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    condition: postgresql.enabled    # 仅在 true 时安装

  - name: redis
    tags:
      - cache
    condition: redis.enabled
```

```bash
# 禁用依赖
helm install myapp ./mychart --set postgresql.enabled=false

# 按标签启用/禁用
helm install myapp ./mychart --set tags.cache=false
```

## Helm 最佳实践

### Chart 结构

1. **保持模板简单**：复杂逻辑属于应用代码
2. **使用辅助函数**：用 _helpers.tpl 避免重复
3. **文档化 values**：在 values.yaml 中注释所有值
4. **提供默认值**：在 values.yaml 中提供合理的默认值
5. **支持自定义**：允许覆盖重要设置

### 命名约定

```yaml
# 好的命名
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

# 在资源中使用
name: {{ include "mychart.fullname" . }}-deployment
name: {{ include "mychart.fullname" . }}-service
name: {{ include "mychart.fullname" . }}-configmap
```

### 标签

使用标准标签：

```yaml
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end }}
```

### 资源管理

始终允许资源自定义：

```yaml
# values.yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

# 模板
resources:
  {{- toYaml .Values.resources | nindent 10 }}
```

### 安全

```yaml
# values.yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

# 模板
securityContext:
  {{- toYaml .Values.securityContext | nindent 10 }}
```

### 文档

```yaml
# 带注释的 values.yaml
# 要部署的副本数量
# @default -- 2
replicaCount: 2

# 镜像配置
image:
  # 镜像仓库
  # @default -- myapp
  repository: myapp

  # 镜像拉取策略
  # @default -- IfNotPresent
  pullPolicy: IfNotPresent

  # 镜像标签（默认为 Chart.AppVersion）
  # @default -- Chart.AppVersion
  tag: ""
```

### 测试 Chart

```bash
# 检查 chart
helm lint ./mychart

# Dry run 安装
helm install myapp ./mychart --dry-run --debug

# 模板渲染（不需要 API server）
helm template myapp ./mychart

# 带 values 的模板
helm template myapp ./mychart -f values-prod.yaml

# 测试 release
helm test myapp
```

### Chart 测试

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

```bash
# 运行测试
helm test myapp
```

## ML 应用的 Helm

### ML 模型服务 Chart

```yaml
# Chart.yaml
apiVersion: v2
name: ml-model-server
description: Helm chart for ML model serving
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

```yaml
# values.yaml
replicaCount: 3

image:
  repository: tensorflow/serving
  tag: "2.8.0-gpu"
  pullPolicy: IfNotPresent

model:
  name: "my_model"
  version: "1"
  basePath: "/models"
  configFile: ""

gpu:
  enabled: true
  count: 1

resources:
  requests:
    memory: "2Gi"
    cpu: "1"
  limits:
    memory: "4Gi"
    cpu: "2"

service:
  type: ClusterIP
  restPort: 8501
  grpcPort: 8500

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true

redis:
  enabled: true
  auth:
    enabled: false

persistence:
  enabled: true
  size: 100Gi
  storageClass: ""
  accessMode: ReadOnlyMany

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - ml-model-server
        topologyKey: kubernetes.io/hostname
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ml-model-server.fullname" . }}
  labels:
    {{- include "ml-model-server.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "ml-model-server.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "ml-model-server.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: model-server
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - name: rest
          containerPort: {{ .Values.service.restPort }}
        - name: grpc
          containerPort: {{ .Values.service.grpcPort }}
        env:
        - name: MODEL_NAME
          value: {{ .Values.model.name | quote }}
        - name: MODEL_BASE_PATH
          value: {{ .Values.model.basePath | quote }}
        {{- if .Values.redis.enabled }}
        - name: REDIS_HOST
          value: {{ .Release.Name }}-redis-master
        - name: REDIS_PORT
          value: "6379"
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- if .Values.gpu.enabled }}
          limits:
            nvidia.com/gpu: {{ .Values.gpu.count }}
        {{- end }}
        volumeMounts:
        - name: model-storage
          mountPath: {{ .Values.model.basePath }}
          readOnly: true
        livenessProbe:
          httpGet:
            path: /v1/models/{{ .Values.model.name }}
            port: rest
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/models/{{ .Values.model.name }}
            port: rest
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: model-storage
        {{- if .Values.persistence.enabled }}
        persistentVolumeClaim:
          claimName: {{ include "ml-model-server.fullname" . }}-models
        {{- else }}
        emptyDir: {}
        {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 部署 ML 应用

```bash
# 创建生产环境 values
cat > ml-prod-values.yaml <<EOF
replicaCount: 5

model:
  name: "resnet50"
  version: "1"

gpu:
  enabled: true
  count: 1

resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60

redis:
  enabled: true
  master:
    persistence:
      size: 10Gi

persistence:
  enabled: true
  size: 500Gi
  storageClass: "fast-ssd"

ingress:
  enabled: true
  hosts:
    - host: ml-api.example.com
      paths:
        - path: /
          pathType: Prefix
EOF

# 安装
helm install ml-prod ./ml-model-server -f ml-prod-values.yaml

# 升级
helm upgrade ml-prod ./ml-model-server -f ml-prod-values.yaml

# 监控 rollout
kubectl rollout status deployment/ml-prod-ml-model-server
```

## 总结和关键要点

### 关键概念

- **Helm** 是 Kubernetes 的包管理器
- **Chart** 打包 Kubernetes 应用
- **Release** 是部署到集群的 chart 实例
- **Repository** 分发 chart
- **Values** 自定义 chart 行为
- **Templates** 使用 Go 模板实现动态 manifest

### Helm 命令

```bash
# 仓库管理
helm repo add/update/list/remove

# Chart 管理
helm create/package/lint

# Release 管理
helm install/upgrade/rollback/uninstall

# 信息查询
helm list/status/history/get

# 测试
helm test/lint/template
```

### Chart 结构

```
mychart/
├── Chart.yaml         # 元数据
├── values.yaml        # 默认值
├── templates/         # Kubernetes manifest
│   ├── _helpers.tpl   # 辅助模板
│   ├── NOTES.txt      # 使用说明
│   └── *.yaml         # 资源模板
└── charts/            # 依赖
```

### 最佳实践

1. 对 chart 使用语义化版本控制
2. 用注释文档化所有 values
3. 提供合理的默认值
4. 使用辅助函数实现可复用模板
5. 应用标准标签
6. 允许资源自定义
7. 用 `helm lint` 和 `helm test` 测试 chart
8. 对复杂应用使用依赖
9. 正确保护 secret
10. 将 chart 纳入版本控制

### 后续步骤

- **Lecture 04**：Kubernetes 运维
- **Exercise 02**：为你的应用创建 Helm chart
- 练习使用 Artifact Hub 上的公共 chart
- 为你的 ML 模型创建 chart

### 延伸阅读

- Helm 文档：https://helm.sh/docs/
- Artifact Hub：https://artifacthub.io/
- Chart 最佳实践：https://helm.sh/docs/chart_best_practices/
- Helm GitHub：https://github.com/helm/helm

---

**恭喜！** 你现在理解了 Helm，可以有效地打包 Kubernetes 应用。练习创建 chart 以巩固你的知识。
