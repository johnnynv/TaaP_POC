# GitHub 和 GitLab Webhook 集成 Tekton 完整指南

## 概述

本文档详细说明如何配置 GitHub 和 GitLab 的 webhook 到 Tekton，实现代码推送时自动触发 CI/CD 流水线。

## 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                Git平台 (GitHub/GitLab)                          │
│                     Push Event                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │ HTTP POST (Webhook)
┌─────────────────────────────▼───────────────────────────────────┐
│                    Tekton Triggers                               │
│   EventListener → TriggerBinding → TriggerTemplate              │
└─────────────────────────────┬───────────────────────────────────┘
                              │ 创建 PipelineRun
┌─────────────────────────────▼───────────────────────────────────┐
│                    Tekton Pipelines                              │
│        代码拉取 → 构建 → 测试 → 部署                              │
└─────────────────────────────────────────────────────────────────┘
```

## 核心组件说明

### Tekton Triggers 组件
1. **EventListener** - 接收 webhook 请求的长期运行服务
2. **TriggerBinding** - 从 webhook 载荷中提取参数
3. **TriggerTemplate** - 定义要创建的 Pipeline/Task 资源
4. **Interceptor** - 验证和处理 webhook 载荷

## 通用基础配置

### 1. 命名空间和权限配置

```yaml
# namespace-and-rbac.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tekton-webhook
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers-sa
  namespace: tekton-webhook
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tekton-triggers-role
  namespace: tekton-webhook
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["eventlisteners", "triggerbindings", "triggertemplates", "triggers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns", "taskruns"]
    verbs: ["create", "get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tekton-triggers-binding
  namespace: tekton-webhook
subjects:
  - kind: ServiceAccount
    name: tekton-triggers-sa
    namespace: tekton-webhook
roleRef:
  kind: Role
  name: tekton-triggers-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tekton-triggers-cluster-role
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["clustertriggerbindings", "clusterinterceptors"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-triggers-cluster-binding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers-sa
    namespace: tekton-webhook
roleRef:
  kind: ClusterRole
  name: tekton-triggers-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

### 2. 基础 Pipeline 定义

```yaml
# base-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: git-push-pipeline
  namespace: tekton-webhook
spec:
  params:
    - name: git-url
      type: string
      description: Git repository URL
    - name: git-revision
      type: string
      description: Git revision to build
      default: main
    - name: repo-name
      type: string
      description: Repository name
  
  workspaces:
    - name: source-code
      description: Workspace for source code
  
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
        kind: ClusterTask
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: source-code
    
    - name: build-and-test
      taskSpec:
        workspaces:
          - name: source
        steps:
          - name: build
            image: docker.io/library/node:18-alpine
            workingDir: $(workspaces.source.path)
            script: |
              #!/bin/sh
              echo "=== 构建应用 ==="
              echo "仓库: $(params.repo-name)"
              echo "分支: $(params.git-revision)"
              echo "URL: $(params.git-url)"
              
              # 检查文件
              ls -la
              
              # 这里可以添加实际的构建逻辑
              # npm install
              # npm run build
              # npm test
              
              echo "=== 构建完成 ==="
      workspaces:
        - name: source
          workspace: source-code
      runAfter:
        - git-clone
```

## GitHub Webhook 配置

### 1. GitHub TriggerBinding

```yaml
# github-trigger-binding.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
  namespace: tekton-webhook
spec:
  params:
    - name: git-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: $(body.after)
    - name: repo-name
      value: $(body.repository.name)
    - name: repo-full-name
      value: $(body.repository.full_name)
    - name: head-commit-id
      value: $(body.head_commit.id)
    - name: head-commit-message
      value: $(body.head_commit.message)
    - name: pusher-name
      value: $(body.pusher.name)
    - name: pusher-email
      value: $(body.pusher.email)
    - name: ref
      value: $(body.ref)
    - name: branch-name
      value: $(body.ref.split('/')[2])  # refs/heads/main -> main
```

### 2. GitHub TriggerTemplate

```yaml
# github-trigger-template.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: github-push-template
  namespace: tekton-webhook
spec:
  params:
    - name: git-url
      description: Git repository URL
    - name: git-revision
      description: Git revision to build
    - name: repo-name
      description: Repository name
    - name: repo-full-name
      description: Repository full name
    - name: head-commit-id
      description: Head commit ID
    - name: head-commit-message
      description: Head commit message
    - name: pusher-name
      description: Pusher name
    - name: pusher-email
      description: Pusher email
    - name: ref
      description: Git reference
    - name: branch-name
      description: Branch name
  
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: github-push-run-
        namespace: tekton-webhook
        labels:
          app: $(tt.params.repo-name)
          git-provider: github
          branch: $(tt.params.branch-name)
        annotations:
          git-url: $(tt.params.git-url)
          commit-id: $(tt.params.head-commit-id)
          commit-message: $(tt.params.head-commit-message)
          pusher: $(tt.params.pusher-name)
      spec:
        serviceAccountName: tekton-triggers-sa
        pipelineRef:
          name: git-push-pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
          - name: git-revision
            value: $(tt.params.git-revision)
          - name: repo-name
            value: $(tt.params.repo-name)
        workspaces:
          - name: source-code
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi
```

### 3. GitHub EventListener

```yaml
# github-event-listener.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-webhook-listener
  namespace: tekton-webhook
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push-trigger
      interceptors:
        - github:
            secretRef:
              secretName: github-webhook-secret
              secretKey: secretToken
            eventTypes:
              - push
        - cel:
            filter: |
              body.ref.startsWith('refs/heads/') && 
              !body.deleted &&
              body.head_commit != null
      bindings:
        - ref: github-push-binding
      template:
        ref: github-push-template
  resources:
    kubernetesResource:
      serviceType: ClusterIP
      spec:
        template:
          spec:
            containers:
              - name: event-listener
                resources:
                  requests:
                    memory: "64Mi"
                    cpu: "50m"
                  limits:
                    memory: "128Mi"
                    cpu: "100m"
---
# 创建 webhook secret
apiVersion: v1
kind: Secret
metadata:
  name: github-webhook-secret
  namespace: tekton-webhook
type: Opaque
data:
  secretToken: bXlfc2VjcmV0X3Rva2Vu  # base64 encoded "my_secret_token"
---
# 暴露 EventListener 服务
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: github-webhook-ingress
  namespace: tekton-webhook
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # 可选
spec:
  tls:
    - hosts:
        - webhook.yourdomain.com
      secretName: webhook-tls
  rules:
    - host: webhook.yourdomain.com
      http:
        paths:
          - path: /github
            pathType: Prefix
            backend:
              service:
                name: el-github-webhook-listener
                port:
                  number: 8080
```

## GitLab Webhook 配置

### 1. GitLab TriggerBinding

```yaml
# gitlab-trigger-binding.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: gitlab-push-binding
  namespace: tekton-webhook
spec:
  params:
    - name: git-url
      value: $(body.project.git_http_url)
    - name: git-revision
      value: $(body.after)
    - name: repo-name
      value: $(body.project.name)
    - name: repo-full-name
      value: $(body.project.path_with_namespace)
    - name: head-commit-id
      value: $(body.checkout_sha)
    - name: head-commit-message
      value: $(body.commits[0].message)
    - name: pusher-name
      value: $(body.user_name)
    - name: pusher-email
      value: $(body.user_email)
    - name: ref
      value: $(body.ref)
    - name: branch-name
      value: $(body.ref.split('/')[2])  # refs/heads/main -> main
    - name: project-id
      value: $(body.project_id)
    - name: project-url
      value: $(body.project.web_url)
```

### 2. GitLab TriggerTemplate

```yaml
# gitlab-trigger-template.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: gitlab-push-template
  namespace: tekton-webhook
spec:
  params:
    - name: git-url
      description: Git repository URL
    - name: git-revision
      description: Git revision to build
    - name: repo-name
      description: Repository name
    - name: repo-full-name
      description: Repository full name
    - name: head-commit-id
      description: Head commit ID
    - name: head-commit-message
      description: Head commit message
    - name: pusher-name
      description: Pusher name
    - name: pusher-email
      description: Pusher email
    - name: ref
      description: Git reference
    - name: branch-name
      description: Branch name
    - name: project-id
      description: GitLab project ID
    - name: project-url
      description: GitLab project URL
  
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: gitlab-push-run-
        namespace: tekton-webhook
        labels:
          app: $(tt.params.repo-name)
          git-provider: gitlab
          branch: $(tt.params.branch-name)
          project-id: $(tt.params.project-id)
        annotations:
          git-url: $(tt.params.git-url)
          commit-id: $(tt.params.head-commit-id)
          commit-message: $(tt.params.head-commit-message)
          pusher: $(tt.params.pusher-name)
          project-url: $(tt.params.project-url)
      spec:
        serviceAccountName: tekton-triggers-sa
        pipelineRef:
          name: git-push-pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
          - name: git-revision
            value: $(tt.params.git-revision)
          - name: repo-name
            value: $(tt.params.repo-name)
        workspaces:
          - name: source-code
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 1Gi
```

### 3. GitLab EventListener

```yaml
# gitlab-event-listener.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: gitlab-webhook-listener
  namespace: tekton-webhook
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: gitlab-push-trigger
      interceptors:
        - gitlab:
            secretRef:
              secretName: gitlab-webhook-secret
              secretKey: secretToken
            eventTypes:
              - Push Hook
        - cel:
            filter: |
              body.object_kind == 'push' && 
              body.ref.startsWith('refs/heads/') &&
              body.before != body.after &&
              body.total_commits_count > 0
      bindings:
        - ref: gitlab-push-binding
      template:
        ref: gitlab-push-template
  resources:
    kubernetesResource:
      serviceType: ClusterIP
      spec:
        template:
          spec:
            containers:
              - name: event-listener
                resources:
                  requests:
                    memory: "64Mi"
                    cpu: "50m"
                  limits:
                    memory: "128Mi"
                    cpu: "100m"
---
# 创建 webhook secret
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-webhook-secret
  namespace: tekton-webhook
type: Opaque
data:
  secretToken: bXlfc2VjcmV0X3Rva2Vu  # base64 encoded "my_secret_token"
---
# 暴露 EventListener 服务
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-webhook-ingress
  namespace: tekton-webhook
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # 可选
spec:
  tls:
    - hosts:
        - webhook.yourdomain.com
      secretName: webhook-tls
  rules:
    - host: webhook.yourdomain.com
      http:
        paths:
          - path: /gitlab
            pathType: Prefix
            backend:
              service:
                name: el-gitlab-webhook-listener
                port:
                  number: 8080
```

## 配置差异对比

### GitHub vs GitLab 主要差异

| 组件 | GitHub | GitLab | 差异程度 |
|------|--------|---------|----------|
| **EventListener** | ✅ 相同结构 | ✅ 相同结构 | **最小** |
| **TriggerBinding** | `body.repository.clone_url` | `body.project.git_http_url` | **中等** |
| **TriggerTemplate** | ✅ 几乎相同 | ✅ 几乎相同 | **最小** |
| **Interceptor** | `github:` | `gitlab:` | **小** |
| **CEL Filter** | `body.ref.startsWith('refs/heads/')` | `body.object_kind == 'push'` | **小** |
| **Webhook 载荷** | 不同的 JSON 结构 | 不同的 JSON 结构 | **大** |

### 详细差异分析

#### 1. Webhook 载荷字段映射

| 信息 | GitHub 字段 | GitLab 字段 | 说明 |
|------|-------------|-------------|------|
| 仓库 URL | `body.repository.clone_url` | `body.project.git_http_url` | 字段名不同 |
| 提交 SHA | `body.after` | `body.after` | 相同 |
| 仓库名 | `body.repository.name` | `body.project.name` | 结构不同 |
| 推送者 | `body.pusher.name` | `body.user_name` | 结构不同 |
| 分支引用 | `body.ref` | `body.ref` | 相同 |

#### 2. Interceptor 配置

**GitHub:**
```yaml
interceptors:
  - github:
      secretRef:
        secretName: github-webhook-secret
        secretKey: secretToken
      eventTypes:
        - push
```

**GitLab:**
```yaml
interceptors:
  - gitlab:
      secretRef:
        secretName: gitlab-webhook-secret  
        secretKey: secretToken
      eventTypes:
        - Push Hook
```

#### 3. CEL 过滤器

**GitHub:**
```yaml
- cel:
    filter: |
      body.ref.startsWith('refs/heads/') && 
      !body.deleted &&
      body.head_commit != null
```

**GitLab:**
```yaml
- cel:
    filter: |
      body.object_kind == 'push' && 
      body.ref.startsWith('refs/heads/') &&
      body.before != body.after &&
      body.total_commits_count > 0
```

## 部署步骤

### 1. 部署基础资源

```bash
# 创建命名空间和权限
kubectl apply -f namespace-and-rbac.yaml

# 部署基础 Pipeline
kubectl apply -f base-pipeline.yaml
```

### 2. 部署 GitHub Webhook

```bash
# 部署 GitHub webhook 组件
kubectl apply -f github-trigger-binding.yaml
kubectl apply -f github-trigger-template.yaml  
kubectl apply -f github-event-listener.yaml

# 获取 EventListener 服务名
kubectl get svc -n tekton-webhook | grep github

# 检查 EventListener 状态
kubectl get eventlistener github-webhook-listener -n tekton-webhook
```

### 3. 部署 GitLab Webhook

```bash
# 部署 GitLab webhook 组件
kubectl apply -f gitlab-trigger-binding.yaml
kubectl apply -f gitlab-trigger-template.yaml
kubectl apply -f gitlab-event-listener.yaml

# 获取 EventListener 服务名
kubectl get svc -n tekton-webhook | grep gitlab

# 检查 EventListener 状态  
kubectl get eventlistener gitlab-webhook-listener -n tekton-webhook
```

### 4. 配置 Git 平台 Webhook

#### GitHub 配置步骤
1. 进入 GitHub 仓库设置
2. 选择 "Webhooks" → "Add webhook"
3. Payload URL: `https://webhook.yourdomain.com/github`
4. Content type: `application/json`
5. Secret: `my_secret_token`
6. Events: 选择 "Just the push event"

#### GitLab 配置步骤
1. 进入 GitLab 项目设置
2. 选择 "Webhooks" → "Add new webhook"
3. URL: `https://webhook.yourdomain.com/gitlab`
4. Secret Token: `my_secret_token`
5. Trigger: 选择 "Push events"
6. SSL verification: 启用

## 测试和验证

### 1. 测试 GitHub Webhook

```bash
# 模拟 GitHub push 事件
curl -X POST https://webhook.yourdomain.com/github \
  -H "Content-Type: application/json" \
  -H "X-GitHub-Event: push" \
  -H "X-Hub-Signature-256: sha256=..." \
  -d '{
    "ref": "refs/heads/main",
    "after": "abc123...",
    "repository": {
      "name": "test-repo",
      "clone_url": "https://github.com/user/test-repo.git"
    },
    "head_commit": {
      "id": "abc123...",
      "message": "Test commit"
    },
    "pusher": {
      "name": "testuser",
      "email": "test@example.com"
    }
  }'
```

### 2. 测试 GitLab Webhook

```bash
# 模拟 GitLab push 事件
curl -X POST https://webhook.yourdomain.com/gitlab \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Event: Push Hook" \
  -H "X-Gitlab-Token: my_secret_token" \
  -d '{
    "object_kind": "push",
    "ref": "refs/heads/main", 
    "after": "abc123...",
    "before": "def456...",
    "total_commits_count": 1,
    "project": {
      "name": "test-repo",
      "git_http_url": "https://gitlab.com/user/test-repo.git",
      "web_url": "https://gitlab.com/user/test-repo"
    },
    "commits": [{
      "id": "abc123...",
      "message": "Test commit"
    }],
    "user_name": "testuser",
    "user_email": "test@example.com"
  }'
```

### 3. 查看运行结果

```bash
# 查看 PipelineRun
kubectl get pipelinerun -n tekton-webhook

# 查看 PipelineRun 详情
kubectl describe pipelinerun <pipelinerun-name> -n tekton-webhook

# 查看日志
kubectl logs -f <pod-name> -n tekton-webhook
```

## 总结

GitHub 和 GitLab 的 webhook 集成到 Tekton 的主要差异：

1. **配置复杂度**: 基本相同，都需要 4-5 个 YAML 文件
2. **代码差异度**: 约 20-30% 的配置需要调整
3. **主要差异**: 
   - Webhook 载荷字段映射
   - Interceptor 类型 (`github:` vs `gitlab:`)  
   - CEL 过滤器条件
   - 事件类型名称

4. **相同部分**:
   - TriggerTemplate 结构几乎相同
   - Pipeline 定义完全相同
   - 基础 RBAC 配置相同
   - EventListener 结构相同

总体而言，从一个平台迁移到另一个平台的工作量相对较小，主要是调整字段映射和过滤条件。 