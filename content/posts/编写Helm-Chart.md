---
title: "编写 helm chart"
date: "2022-09-30"
categories:
    - "技术"
tags:
    - "helm"
toc: false
indent: false
original: true
draft: false
---

## 更新记录

| 时间       | 内容 |
| ---------- | ---- |
| 2022-09-30 | 初稿 |
| 2022-10-01 | 调整目录层级 & 修改调试部分 & 增加优化部分 |

## 软件版本

| soft       | Version |
| ---------- | ------- |
| Helm       | v3.10.0 |
| Kubernetes | 1.24.4  |

## 一、Chart 介绍

### 1.1、创建 父Chart & 子Chart

``` zsh
# 创建 helm workspace
➜  mkdir helm; cd helm

# 创建 父Chart
➜  helm create nrmm

# 创建 子Chart
➜  helm create nrmm-enforce
➜  helm create nrmm-gateway
➜  helm create nrmm-machine
➜  helm create nrmm-site
➜  helm create nrmm-third
```

### 1.2、Chart 目录层级

执行 `helm create` 命令会自动生成 chart文件

``` zsh
➜  tree nrmm
nrmm
├── Chart.yaml                       # 对这个 chart 的说明
├── charts                           # charts 目录存放其他依赖的 chart（我们称之为子chart）
├── templates                        # 目录存放模板文件
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml                      # values 包含 chart 的默认值。这些值可以在用户执行 `helm install` 或 `helm upgrade` 时覆盖
```

## 二、调试

### 2.1、调试 父Chart

①、首先删除 templates 目录下无用的文件

``` zsh
# 因为创建父chart 目的是连接、管理其他子chart, 所以 templates 目录下的 yaml 基本上没有用
➜  rm -rf nrmm/templates/{deployment,hpa,ingress,service,serviceaccount,tests}*
```

②、创建 templates/namespace.yaml 模板文件

``` zsh
➜  vim templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.nameSpace }}
```

③、修改 values.yaml 文件

``` zsh
# 增加 nameSpace 变量
➜  vim values.yaml
nameSpace: craftsman
```

④、调试 chart

可以使用 `helm install --dry-run` 命令用来调试chart, `--dry-run` 只会将 渲染模板的内容输出，并不会在 kubernetes 中真的创建这些资源

``` zsh
# 已经可以看到 生成出来的 yaml 资源了
➜  helm install --dry-run demo nrmm
NAME: demo
LAST DEPLOYED: Sat Oct  1 16:27:29 2022
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: nrmm/templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: craftsman

```

### 2.2、调试 子Chart

这里以子项目 gateway 作为调试示例

①、首先删除无用模板

``` zsh
# 删除无用文件
➜  rm -rf nrmm-gateway/templates/{hpa,serviceaccount，tests}*
```

②、更新模板文件

``` yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.nameSpace }}
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        app: {{ .Chart.Name }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}


# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := .Chart.Name -}}
{{- $svcPort := .Values.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
  {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className}}
  {{- end }}
{{- end }}
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1
{{- else if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  namespace: {{ .Values.nameSpace }}
  name: {{ $fullName }}
  labels:
    app: {{ .Chart.Name }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if and .pathType (semverCompare ">=1.18-0" $.Capabilities.KubeVersion.GitVersion) }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.GitVersion }}
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
              {{- else }}
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}


# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: {{ .Values.nameSpace }}
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - name: gateway
      protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      nodePort: {{ .Values.service.nodePort }}
  selector:
    app: {{ .Chart.Name }}


```

③、更新 values 文件

``` zsh
➜  vim values.yaml
nameSpace: nrmm    # 增加 namespace 字段

image:
  repository: harbor.gjr.net/test_all/nrmm-gateway # 修改 image
  tag: ""                                          # tag 置为空, 可在调试时 传入参数修改

# gateway 服务修改类型为 NodePort, 并设置 nodePort 参数
service:
  type: NodePort
  port: 8090
  nodePort: 30090
```

④、调试 chart

``` zsh
# 使用 --set 参数传入镜像tag
➜  helm install --dry-run demo nrmm-gateway --set image.tag=21
NAME: demo
LAST DEPLOYED: Sat Oct  1 16:43:17 2022
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: nrmm-gateway/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: nrmm
  name: nrmm-gateway
  labels:
    app: nrmm-gateway
spec:
  type: NodePort
  ports:
    - name: gateway
      protocol: TCP
      port: 8090
      targetPort: 8090
      nodePort: 30090
  selector:
    app: nrmm-gateway
---
# Source: nrmm-gateway/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nrmm
  name: nrmm-gateway
  labels:
    app: nrmm-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nrmm-gateway
  template:
    metadata:
      labels:
        app: nrmm-gateway
    spec:
      securityContext:
        {}
      containers:
        - name: nrmm-gateway
          securityContext:
            {}
          image: "harbor.gjr.net/test_all/nrmm-gateway:21"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8090
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
---
# Source: nrmm-gateway/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: nrmm
  name: nrmm-gateway
  labels:
    app: nrmm-gateway
spec:
  ingressClassName: nginx
  rules:
    - host: "nrmm.apitest.gjr.net"
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: nrmm-gateway
                port:
                  number: 8090

```

输出 kubernetes yaml资源清单后 检查渲染的内容是否正确

其它 子chart 根据 gateway 项目的调试方法 逐一调试, 必须 dry-run 渲染出的内容正确以后, 才能进行子父联调

### 2.3、子父联调

①、修改 chart 文件

``` zsh
# 将依赖关系写入 父chart的 Chart文件中
➜  vim nrmm/Chart.yaml
dependencies:
  - name: nrmm-enforce                   # name 必须跟 Chart文件中的 name保持一致, 这里小踩过一个坑
    repository: file://../nrmm-enforce        # file://是针对Chart.yaml的相对路径, 也可以写远程 repo 的地址
    version: 0.1.0
  - name: nrmm-gateway
    repository: file://../nrmm-gateway
    version: 0.1.0
  - name: nrmm-machine
    repository: file://../nrmm-machine
    version: 0.1.0
  - name: nrmm-site
    repository: file://../nrmm-site
    version: 0.1.0
  - name: nrmm-third
    repository: file://../nrmm-third
    version: 0.1.0
```

②、更新 依赖关系

``` zsh
# 执行命令 导入依赖项目
➜  helm dependency update nrmm
Saving 5 charts
Deleting outdated charts

# 查看 nrmm 项目的目录层级
➜  tree nrmm
nrmm
├── Chart.lock
├── Chart.yaml
├── charts                          # 可以看到 charts 文件夹下已经导入了 其它依赖 chart 项目
│   ├── nrmm-enforce-0.1.0.tgz
│   ├── nrmm-gateway-0.1.0.tgz
│   ├── nrmm-machine-0.1.0.tgz
│   ├── nrmm-site-0.1.0.tgz
│   └── nrmm-third-0.1.0.tgz
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── namespace.yaml
└── values.yaml

2 directories, 11 files
```

③、调试

``` zsh
➜  helm install --dry-run demo nrmm
```

## 三、优化部分

上述步骤完成的多依赖构建, 已经满足一键部署多个服务的需求了, 但是通过实际操作下来, 会发现一些问题: 

每个子 chart 中都有相同的 values 值。例如：replicaCount, nameSpace...
有很多模版都是相同的。例如：deployment.yaml, service.yaml...

如果有二三十个微服务后端的话, 会有一大部分时间在做重复工作, 所以我们就在想能不能复用这些常量和模版

### 3.1、全局常量

在父Chart的 values.yaml 文件中追加一个 global 变量组，将所有 子Chart 会用到的变量设置为全局常量
此值可供所有 Chart使用, 通过 {{ .Values.global.nameSpace }} 这样我们就无需在每个子 Chart的 `values.yaml` 中重复定义这些值了  
只需修改一下模版中的使用方式即可

``` zsh
➜  vim nrmm/values.yaml
global:
  nameSpace: craftsman
```

### 3.2、共享常量

在 父Chart 的 values 文件中设置一个节点为 子Chart名 的变量组并设置参数。就可以在 子Chart 中使用 {{ .Values.replicaCount }} 来引用这个变量了
当 Helm 发现节点名是 子chart名 时，它会自动引用这个常量到 子chart 的 values.yaml 中
因此，在 nrmm-gateway 中，你也可以通过 {{ .Values.image.repository }} 来访问 父Chart values 文件中设置的这个常量
如果出现 父Chart 与 子Chart 中都定义了相同的 values 值，那么 helm 会以 父Chart 为准，覆盖 子Chart 中相同的值
这样，我们仅需在 父Chart 设置一次变量即可, 省去了打开每一个 子Chart, 修改每一个 子Chart的 values 文件

``` yaml
nrmm-enforce:
  replicaCount: 1
  image:
    repository: harbor.gjr.net/test_all/nrmm-enforce
    tag: ""
  service:
    type: ClusterIP
    port: 8092

nrmm-gateway:
  replicaCount: 1
  image:
    repository: harbor.gjr.net/test_all/nrmm-gateway
    tag: ""
  service:
    type: NodePort
    port: 8090
    nodePort: 30090
  
nrmm-machine:
  replicaCount: 1
  image:
    repository: harbor.gjr.net/test_all/nrmm-machine
    tag: ""
  service:
    type: ClusterIP
    port: 8081
```

### 3.3、路径优化

现在所有的 子Chart 都跟 父Chart 目录同级, 不方便管理, 将所有 子Chart 都移动至 父Chart 的 charts 目录下

``` zsh
# 移动 子Chart
➜  mv nrmm-{enforce,gateway,machine,site,third} nrmm/charts

# 删除 lock文件
➜  cd nrmm; rm -f Chart.lock

# 修改 Chart文件
➜  vim Chart.yaml
dependencies:
  - name: nrmm-enforce
    repository: ""                   # repository 置为空即可
    version: 0.1.0
  - name: nrmm-gateway
    repository: ""
    version: 0.1.0
  - name: nrmm-machine
    repository: ""
    version: 0.1.0
  - name: nrmm-site
    repository: ""
    version: 0.1.0
  - name: nrmm-third
    repository: ""
    version: 0.1.0
```


### 3.4、仅安装 子Chart

在使用多依赖 Chart 的过程中, 我们不一定一次部署所有的服务, 有的时候我们只需要安装一个或多个服务, 那么仅需对 Chart 文件中依赖项部分进行改造即可。

``` yaml
dependencies:
  - name: nrmm-enforce
    repository: ""
    version: 0.1.0
    condition: nrmm-enforce.enable       # 增加 condition 选项, 值为 values文件中顶级节点 nrmm-enforce 下的 enable, 需要设为布尔值
  - name: nrmm-gateway
    repository: ""
    version: 0.1.0
    condition: nrmm-gateway.enable
  - name: nrmm-machine
    repository: ""
    version: 0.1.0
    condition: nrmm-machine.enable
  - name: nrmm-site
    repository: ""
    version: 0.1.0
    condition: nrmm-site.enable
  - name: nrmm-third
    repository: ""
    version: 0.1.0
    condition: nrmm-third.enable
```

values 设置 enable

``` yaml
nrmm-enforce:
  enable: false        # 默认设为 false
  replicaCount: 1
  image:
    repository: harbor.gjr.net/test_all/nrmm-enforce
    tag: ""
  service:
    type: ClusterIP
    port: 8092
```

执行安装 nrmm-enforce 子Chart

``` zsh
# 只有服务 enable 设为 true 才启用
➜  helm install --dry-run --set nrmm-enforce.enable=true --set nrmm-enforce.image.tag=21 demo nrmm/
```

> 参考列表：
>
> 1、[Helm3 Chart 多依赖微服务构建案例](https://blog.csdn.net/qq_39680564/article/details/107516510)  
> 2、[helm 官方手册 - Helm 依赖](https://v3.helm.sh/zh/docs/helm/helm_dependency/)  
> 3、[helm 官方手册 - 依赖项中的条件与标签](https://helm.sh/docs/topics/charts/#tags-and-condition-fields-in-dependencies)  
> 