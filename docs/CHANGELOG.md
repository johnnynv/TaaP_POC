# 文档更新日志

## [2025-01-20] - 新增 Tekton Webhook 集成指南

### 新增内容
- **文件**: `docs/tekton-webhook-integration-guide.md`
- **描述**: GitHub 和 GitLab Webhook 集成 Tekton 完整指南

### 主要内容包括
1. **完整的 YAML 配置文件**
   - 基础权限和命名空间配置
   - 通用 Pipeline 定义
   - GitHub 专用配置 (TriggerBinding, TriggerTemplate, EventListener)
   - GitLab 专用配置 (TriggerBinding, TriggerTemplate, EventListener)

2. **详细配置差异对比**
   - 组件级别差异分析
   - Webhook 载荷字段映射对比
   - Interceptor 配置差异
   - CEL 过滤器差异

3. **实际部署步骤**
   - 分步骤部署指南
   - GitHub/GitLab 平台配置步骤
   - 测试和验证方法

4. **核心发现**
   - 配置复杂度基本相同，都需要 4-5 个 YAML 文件
   - 代码差异度约 20-30%，主要在字段映射
   - Pipeline 定义完全相同，可重用
   - 从一个平台迁移到另一个平台的工作量相对较小

### 更新背景
用户询问如何配置 GitHub 和 GitLab 的 webhook 到 Tekton，以 push 事件为例，以及需要准备的 YAML 文件和平台间的差异。

---

## [2025-01-20] - 新增 Webhook 触发条件文档

### 新增内容
- **文件**: `docs/webhook-trigger-conditions.md`
- **描述**: 详细说明了 GitHub 和 GitLab 中 webhook 的触发条件和限制

### 主要内容包括
1. **GitHub Webhook 触发条件**
   - 必要条件：事件发生、载荷限制(25MB)、端点可访问、响应时间(<10秒)
   - 支持的事件类型
   - 配置示例(GUI 和 REST API)
   - 载荷示例

2. **GitLab Webhook 触发条件**
   - 必要条件：配置的事件发生、秘钥验证、SSL验证、URL格式
   - 支持的事件类型
   - 配置示例(GUI 和 REST API)
   - 载荷示例

3. **重要发现**
   - 🔑 **关键结论**: Webhook 触发不依赖于代码差距大小，而是基于事件驱动
   - 无论是1行代码还是10000行代码的变更，只要触发订阅的事件，都会调用 webhook
   - 影响触发的因素：载荷大小限制、事件类型配置、特殊限制条件

4. **实际示例对比**
   - 提供了详细的触发场景对比表
   - 包含常见问题解答
   - 最佳实践建议

### 更新背景
响应用户关于 GitHub 和 GitLab webhook 触发条件的查询，特别是关于代码差距大小对 webhook 触发的影响。

---

## 文档维护说明

- 本日志记录文档的重要更新和变更
- 每次更新后请在此处添加相应的记录
- 格式：`[日期] - 更新描述` 