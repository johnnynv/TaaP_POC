# Tekton 云原生测试平台设计方案

## 📋 方案概述

设计一个基于Tekton的云原生测试平台，利用Kubernetes原生能力，提供统一UI体验和轻量级报告生成，支持多Git平台的pytest/go test/shell test执行。

## 🏗️ 核心架构

### 整体设计
```
┌─────────────────────────────────────────────────────────────────┐
│                    Git平台 (GitHub/GitLab/Others)                │
└─────────────────────────────┬───────────────────────────────────┘
                              │ Webhook
┌─────────────────────────────▼───────────────────────────────────┐
│                    Tekton Triggers                               │
│   EventListener → TriggerBinding → TriggerTemplate              │
└─────────────────────────────┬───────────────────────────────────┘
                              │ 创建PipelineRun
┌─────────────────────────────▼───────────────────────────────────┐
│                    Tekton Pipelines                              │
│   代码拉取 → 测试类型检测 → 并行测试执行 → 报告生成 → 通知        │
└─────────────────────────────┬───────────────────────────────────┘
                              │ 存储结果
┌─────────────────────────────▼───────────────────────────────────┐
│                 轻量级报告系统 + Tekton Dashboard                  │
│   JUnit XML解析 → 简洁HTML报告 → 趋势图表 → 统一UI              │
└─────────────────────────────────────────────────────────────────┘
```

### 核心组件

1. **Tekton Triggers** - webhook事件处理
2. **Tekton Pipelines** - 测试流程编排
3. **Tekton Dashboard** - 统一UI界面
4. **轻量级报告引擎** - 替代Allure的简洁方案
5. **云原生存储** - PVC + ConfigMap

## 🔧 技术架构详细设计

### 1. Webhook事件处理

#### EventListener配置
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: git-webhook-listener
  namespace: testing-platform
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    # GitHub触发器
    - name: github-trigger
      interceptors:
        - github:
            secretRef:
              secretName: github-webhook-secret
              secretKey: secretToken
        - cel:
            filter: |
              body.action in ['opened', 'synchronize', 'reopened'] || 
              body.ref.startsWith('refs/heads/')
      bindings:
        - ref: git-binding
      template:
        ref: test-pipeline-template
    
    # GitLab触发器
    - name: gitlab-trigger
      interceptors:
        - gitlab:
            secretRef:
              secretName: gitlab-webhook-secret
              secretKey: secretToken
        - cel:
            filter: "body.object_kind in ['push', 'merge_request']"
      bindings:
        - ref: git-binding
      template:
        ref: test-pipeline-template
    
    # 通用Git触发器（其他平台）
    - name: generic-git-trigger
      interceptors:
        - cel:
            filter: "has(body.repository) && has(body.repository.clone_url)"
      bindings:
        - ref: git-binding
      template:
        ref: test-pipeline-template

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: git-binding
spec:
  params:
    - name: git-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: |
        $(body.head_commit.id || body.after || body.object_attributes.last_commit.id || 'main')
    - name: git-branch
      value: |
        $(body.ref.replaceAll('refs/heads/', '') || body.object_attributes.source_branch || 'main')
    - name: repo-name
      value: $(body.repository.name)
    - name: commit-message
      value: |
        $(body.head_commit.message || body.object_attributes.title || 'Test run')

---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: test-pipeline-template
spec:
  params:
    - name: git-url
    - name: git-revision  
    - name: git-branch
    - name: repo-name
    - name: commit-message
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: test-run-$(tt.params.repo-name)-
        labels:
          app: testing-platform
          repo: $(tt.params.repo-name)
          branch: $(tt.params.git-branch)
        annotations:
          git-url: $(tt.params.git-url)
          commit-message: $(tt.params.commit-message)
      spec:
        pipelineRef:
          name: universal-test-pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
          - name: git-revision
            value: $(tt.params.git-revision)
          - name: git-branch
            value: $(tt.params.git-branch)
          - name: repo-name
            value: $(tt.params.repo-name)
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 2Gi
                storageClassName: fast-ssd
```

### 2. 通用测试Pipeline

#### 主Pipeline定义
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: universal-test-pipeline
  namespace: testing-platform
spec:
  params:
    - name: git-url
      type: string
      description: "Git repository URL"
    - name: git-revision
      type: string
      default: "main"
    - name: git-branch
      type: string
      default: "main"
    - name: repo-name
      type: string
  
  workspaces:
    - name: shared-workspace
      description: "Shared workspace for pipeline"
  
  results:
    - name: test-status
      description: "Overall test status"
    - name: test-summary
      description: "Test execution summary"
  
  tasks:
    # 1. 代码拉取
    - name: git-clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
        - name: deleteExisting
          value: "true"
      workspaces:
        - name: output
          workspace: shared-workspace
    
    # 2. 环境信息收集
    - name: collect-env-info
      taskRef:
        name: collect-env-info
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - git-clone
    
    # 3. 测试类型自动检测
    - name: detect-test-types
      taskRef:
        name: detect-test-types
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - collect-env-info
    
    # 4. 并行测试执行
    - name: execute-pytest
      when:
        - input: $(tasks.detect-test-types.results.has-pytest)
          operator: in
          values: ["true"]
      taskRef:
        name: pytest-execution
      params:
        - name: test-markers
          value: ""  # 可通过配置文件指定
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - detect-test-types
    
    - name: execute-go-tests
      when:
        - input: $(tasks.detect-test-types.results.has-go-tests)
          operator: in
          values: ["true"]
      taskRef:
        name: go-test-execution
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - detect-test-types
    
    - name: execute-shell-tests
      when:
        - input: $(tasks.detect-test-types.results.has-shell-tests)
          operator: in
          values: ["true"]
      taskRef:
        name: shell-test-execution
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - detect-test-types
    
    # 5. 测试结果聚合
    - name: aggregate-results
      taskRef:
        name: aggregate-test-results
      params:
        - name: repo-name
          value: $(params.repo-name)
        - name: git-branch
          value: $(params.git-branch)
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - execute-pytest
        - execute-go-tests
        - execute-shell-tests
    
    # 6. 生成轻量级报告
    - name: generate-reports
      taskRef:
        name: lightweight-report-generator
      params:
        - name: repo-name
          value: $(params.repo-name)
        - name: git-branch
          value: $(params.git-branch)
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - aggregate-results
    
    # 7. 发送通知
    - name: send-notifications
      taskRef:
        name: notification-sender
      params:
        - name: repo-name
          value: $(params.repo-name)
        - name: git-branch
          value: $(params.git-branch)
        - name: test-status
          value: $(tasks.aggregate-results.results.overall-status)
      workspaces:
        - name: source
          workspace: shared-workspace
      runAfter:
        - generate-reports
  
  finally:
    # 清理工作空间
    - name: cleanup-workspace
      taskRef:
        name: cleanup-task
      workspaces:
        - name: source
          workspace: shared-workspace
```

### 3. 核心Task定义

#### 测试类型检测Task
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: detect-test-types
  namespace: testing-platform
spec:
  description: "自动检测项目中的测试类型"
  workspaces:
    - name: source
      description: "Source code workspace"
  
  results:
    - name: has-pytest
      description: "是否包含pytest测试"
    - name: has-go-tests
      description: "是否包含Go测试"
    - name: has-shell-tests
      description: "是否包含Shell测试"
    - name: test-config
      description: "测试配置信息"
  
  steps:
    - name: detect-pytest
      image: python:3.9-alpine
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        HAS_PYTEST="false"
        
        # 检查pytest配置文件
        if [ -f "pytest.ini" ] || [ -f "pyproject.toml" ] || [ -f "setup.cfg" ]; then
          HAS_PYTEST="true"
        fi
        
        # 检查测试文件
        if find . -name "test_*.py" -o -name "*_test.py" | head -1 | grep -q .; then
          HAS_PYTEST="true"
        fi
        
        # 检查tests目录
        if [ -d "tests" ] && find tests -name "*.py" | head -1 | grep -q .; then
          HAS_PYTEST="true"
        fi
        
        echo -n "$HAS_PYTEST" | tee $(results.has-pytest.path)
        echo "检测到pytest: $HAS_PYTEST"
    
    - name: detect-go-tests
      image: golang:1.19-alpine
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        HAS_GO_TESTS="false"
        
        # 检查go.mod文件
        if [ -f "go.mod" ]; then
          # 检查测试文件
          if find . -name "*_test.go" | head -1 | grep -q .; then
            HAS_GO_TESTS="true"
          fi
        fi
        
        echo -n "$HAS_GO_TESTS" | tee $(results.has-go-tests.path)
        echo "检测到Go测试: $HAS_GO_TESTS"
    
    - name: detect-shell-tests
      image: alpine:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        HAS_SHELL_TESTS="false"
        
        # 检查shell测试脚本
        if [ -f "test.sh" ] || [ -f "run-tests.sh" ] || [ -f "tests.sh" ]; then
          HAS_SHELL_TESTS="true"
        fi
        
        # 检查tests目录中的shell脚本
        if [ -d "tests" ] && find tests -name "*.sh" | head -1 | grep -q .; then
          HAS_SHELL_TESTS="true"
        fi
        
        echo -n "$HAS_SHELL_TESTS" | tee $(results.has-shell-tests.path)
        echo "检测到Shell测试: $HAS_SHELL_TESTS"
    
    - name: generate-config
      image: alpine:latest
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -e
        
        # 生成测试配置信息
        cat > test-config.json <<EOF
        {
          "timestamp": "$(date -Iseconds)",
          "pytest": "$(cat $(results.has-pytest.path))",
          "go_tests": "$(cat $(results.has-go-tests.path))",
          "shell_tests": "$(cat $(results.has-shell-tests.path))",
          "workspace": "$(workspaces.source.path)"
        }
        EOF
        
        echo -n "$(cat test-config.json)" | tee $(results.test-config.path)
```

#### Pytest执行Task
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: pytest-execution
  namespace: testing-platform
spec:
  description: "执行pytest测试并生成报告"
  params:
    - name: test-markers
      description: "pytest标记筛选"
      default: ""
    - name: python-version
      description: "Python版本"
      default: "3.9"
  
  workspaces:
    - name: source
      description: "Source code workspace"
  
  results:
    - name: test-count
      description: "测试总数"
    - name: passed-count
      description: "通过测试数"
    - name: failed-count
      description: "失败测试数"
    - name: status
      description: "执行状态"
  
  steps:
    - name: setup-environment
      image: python:$(params.python-version)-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/bash
        set -e
        
        echo "🐍 设置Python测试环境..."
        
        # 更新pip
        pip install --upgrade pip
        
        # 安装基础测试依赖
        pip install pytest pytest-cov pytest-html pytest-json-report
        
        # 安装项目依赖
        if [ -f "requirements.txt" ]; then
          echo "📦 安装requirements.txt依赖..."
          pip install -r requirements.txt
        elif [ -f "pyproject.toml" ]; then
          echo "📦 安装pyproject.toml依赖..."
          pip install .
        elif [ -f "setup.py" ]; then
          echo "📦 安装setup.py依赖..."
          pip install -e .
        fi
        
        echo "✅ 环境设置完成"
    
    - name: execute-tests
      image: python:$(params.python-version)-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/bash
        set -e
        
        echo "🧪 开始执行pytest测试..."
        
        # 构建pytest命令
        PYTEST_CMD="python -m pytest"
        
        # 添加标记筛选
        if [ -n "$(params.test-markers)" ]; then
          PYTEST_CMD="$PYTEST_CMD -m '$(params.test-markers)'"
        fi
        
        # 添加输出格式
        PYTEST_CMD="$PYTEST_CMD \
          --verbose \
          --tb=short \
          --junitxml=junit-report.xml \
          --html=html-report.html --self-contained-html \
          --json-report --json-report-file=json-report.json \
          --cov=. --cov-report=xml --cov-report=html \
          --cov-report=term-missing"
        
        echo "执行命令: $PYTEST_CMD"
        
        # 执行测试（允许失败，后续处理结果）
        eval $PYTEST_CMD || true
        
        echo "✅ 测试执行完成"
    
    - name: analyze-results
      image: python:$(params.python-version)-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/usr/bin/env python3
        import json
        import xml.etree.ElementTree as ET
        import os
        
        print("📊 分析测试结果...")
        
        # 默认值
        total_tests = 0
        passed_tests = 0
        failed_tests = 0
        status = "unknown"
        
        # 尝试从JSON报告解析
        if os.path.exists('json-report.json'):
            try:
                with open('json-report.json', 'r') as f:
                    data = json.load(f)
                    summary = data.get('summary', {})
                    total_tests = summary.get('total', 0)
                    passed_tests = summary.get('passed', 0) 
                    failed_tests = summary.get('failed', 0)
                    status = "success" if failed_tests == 0 else "failed"
                print(f"从JSON报告解析: 总计{total_tests}, 通过{passed_tests}, 失败{failed_tests}")
            except Exception as e:
                print(f"JSON报告解析失败: {e}")
        
        # 备用：从JUnit XML解析
        if total_tests == 0 and os.path.exists('junit-report.xml'):
            try:
                tree = ET.parse('junit-report.xml')
                root = tree.getroot()
                
                for testsuite in root.findall('.//testsuite'):
                    total_tests += int(testsuite.get('tests', 0))
                    failed_tests += int(testsuite.get('failures', 0))
                    failed_tests += int(testsuite.get('errors', 0))
                
                passed_tests = total_tests - failed_tests
                status = "success" if failed_tests == 0 else "failed"
                print(f"从XML报告解析: 总计{total_tests}, 通过{passed_tests}, 失败{failed_tests}")
            except Exception as e:
                print(f"XML报告解析失败: {e}")
        
        # 写入结果
        with open('/tekton/results/test-count', 'w') as f:
            f.write(str(total_tests))
        with open('/tekton/results/passed-count', 'w') as f:
            f.write(str(passed_tests))
        with open('/tekton/results/failed-count', 'w') as f:
            f.write(str(failed_tests))
        with open('/tekton/results/status', 'w') as f:
            f.write(status)
        
        print(f"✅ 结果分析完成: {status}")
```

### 4. 轻量级报告系统

#### 报告生成Task
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: lightweight-report-generator
  namespace: testing-platform
spec:
  description: "生成轻量级HTML测试报告"
  params:
    - name: repo-name
      description: "仓库名称"
    - name: git-branch
      description: "Git分支"
  
  workspaces:
    - name: source
      description: "Source code workspace"
  
  steps:
    - name: generate-unified-report
      image: python:3.9-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/usr/bin/env python3
        import json
        import xml.etree.ElementTree as ET
        import os
        import datetime
        from pathlib import Path
        
        print("📊 生成统一测试报告...")
        
        # 报告数据结构
        report_data = {
            "metadata": {
                "repo_name": "$(params.repo-name)",
                "branch": "$(params.git-branch)",
                "timestamp": datetime.datetime.now().isoformat(),
                "generator": "Tekton Testing Platform v1.0"
            },
            "summary": {
                "total_tests": 0,
                "passed_tests": 0,
                "failed_tests": 0,
                "skipped_tests": 0,
                "duration": 0,
                "success_rate": 0
            },
            "test_suites": [],
            "coverage": {},
            "artifacts": []
        }
        
        # 收集pytest结果
        if os.path.exists('json-report.json'):
            print("📖 解析pytest结果...")
            try:
                with open('json-report.json', 'r') as f:
                    pytest_data = json.load(f)
                    
                summary = pytest_data.get('summary', {})
                report_data["summary"]["total_tests"] += summary.get('total', 0)
                report_data["summary"]["passed_tests"] += summary.get('passed', 0)
                report_data["summary"]["failed_tests"] += summary.get('failed', 0)
                report_data["summary"]["skipped_tests"] += summary.get('skipped', 0)
                report_data["summary"]["duration"] += summary.get('duration', 0)
                
                # 添加测试套件信息
                for test in pytest_data.get('tests', []):
                    report_data["test_suites"].append({
                        "name": test.get('nodeid', ''),
                        "status": test.get('outcome', 'unknown'),
                        "duration": test.get('duration', 0),
                        "file": test.get('file', ''),
                        "error": test.get('call', {}).get('longrepr', '') if test.get('outcome') == 'failed' else ''
                    })
                    
            except Exception as e:
                print(f"解析pytest JSON失败: {e}")
        
        # 收集覆盖率信息
        if os.path.exists('coverage.xml'):
            print("📈 解析覆盖率信息...")
            try:
                tree = ET.parse('coverage.xml')
                root = tree.getroot()
                
                coverage_elem = root.find('.//coverage')
                if coverage_elem is not None:
                    report_data["coverage"] = {
                        "line_rate": float(coverage_elem.get('line-rate', 0)) * 100,
                        "branch_rate": float(coverage_elem.get('branch-rate', 0)) * 100,
                        "lines_covered": coverage_elem.get('lines-covered', 0),
                        "lines_valid": coverage_elem.get('lines-valid', 0)
                    }
            except Exception as e:
                print(f"解析覆盖率XML失败: {e}")
        
        # 计算成功率
        if report_data["summary"]["total_tests"] > 0:
            report_data["summary"]["success_rate"] = (
                report_data["summary"]["passed_tests"] / 
                report_data["summary"]["total_tests"] * 100
            )
        
        # 收集产物
        artifacts = []
        for pattern in ['*.html', '*.xml', '*.json', 'htmlcov/*', 'coverage.html']:
            for file_path in Path('.').glob(pattern):
                if file_path.is_file():
                    artifacts.append(str(file_path))
        report_data["artifacts"] = artifacts
        
        # 保存JSON数据
        with open('unified-report.json', 'w', encoding='utf-8') as f:
            json.dump(report_data, f, indent=2, ensure_ascii=False)
        
        print("✅ 统一报告数据生成完成")
    
    - name: generate-html-report
      image: python:3.9-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/usr/bin/env python3
        import json
        import os
        from datetime import datetime
        
        print("🎨 生成HTML报告...")
        
        # 读取报告数据
        with open('unified-report.json', 'r', encoding='utf-8') as f:
            data = json.load(f)
        
        # 生成HTML报告模板
        html_template = f"""
        <!DOCTYPE html>
        <html lang="zh-CN">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>测试报告 - {data['metadata']['repo_name']}</title>
            <style>
                * {{ margin: 0; padding: 0; box-sizing: border-box; }}
                body {{ font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; 
                       background: #f5f7fa; color: #2c3e50; line-height: 1.6; }}
                .container {{ max-width: 1200px; margin: 0 auto; padding: 20px; }}
                
                .header {{ background: white; border-radius: 8px; padding: 30px; margin-bottom: 20px; 
                          box-shadow: 0 2px 10px rgba(0,0,0,0.1); }}
                .header h1 {{ color: #2c3e50; margin-bottom: 10px; }}
                .header .meta {{ color: #7f8c8d; }}
                
                .summary {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); 
                           gap: 20px; margin-bottom: 30px; }}
                .summary-card {{ background: white; border-radius: 8px; padding: 25px; 
                               box-shadow: 0 2px 10px rgba(0,0,0,0.1); text-align: center; }}
                .summary-card h3 {{ color: #34495e; margin-bottom: 15px; }}
                .summary-card .number {{ font-size: 2.5em; font-weight: bold; margin-bottom: 10px; }}
                .success {{ color: #27ae60; }}
                .failure {{ color: #e74c3c; }}
                .warning {{ color: #f39c12; }}
                .info {{ color: #3498db; }}
                
                .progress-bar {{ width: 100%; height: 20px; background: #ecf0f1; 
                               border-radius: 10px; overflow: hidden; margin: 15px 0; }}
                .progress-fill {{ height: 100%; background: linear-gradient(90deg, #27ae60, #2ecc71); 
                                transition: width 0.3s ease; }}
                
                .test-details {{ background: white; border-radius: 8px; padding: 25px; 
                               box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; }}
                .test-item {{ border-bottom: 1px solid #ecf0f1; padding: 15px 0; }}
                .test-item:last-child {{ border-bottom: none; }}
                .test-name {{ font-weight: 600; margin-bottom: 5px; }}
                .test-status {{ display: inline-block; padding: 4px 12px; border-radius: 4px; 
                              font-size: 12px; font-weight: bold; }}
                .status-passed {{ background: #d5f4e6; color: #27ae60; }}
                .status-failed {{ background: #ffeaea; color: #e74c3c; }}
                .status-skipped {{ background: #fff3cd; color: #f39c12; }}
                
                .coverage-section {{ background: white; border-radius: 8px; padding: 25px; 
                                   box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; }}
                .coverage-bar {{ width: 100%; height: 25px; background: #ecf0f1; 
                               border-radius: 12px; overflow: hidden; position: relative; }}
                .coverage-fill {{ height: 100%; border-radius: 12px; transition: width 0.3s ease; }}
                .coverage-text {{ position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); 
                                font-weight: bold; color: white; text-shadow: 1px 1px 2px rgba(0,0,0,0.5); }}
                
                .artifacts {{ background: white; border-radius: 8px; padding: 25px; 
                            box-shadow: 0 2px 10px rgba(0,0,0,0.1); }}
                .artifact-item {{ display: inline-block; margin: 5px; padding: 8px 16px; 
                                background: #3498db; color: white; text-decoration: none; 
                                border-radius: 4px; font-size: 14px; }}
                .artifact-item:hover {{ background: #2980b9; }}
                
                @media (max-width: 768px) {{
                    .container {{ padding: 10px; }}
                    .summary {{ grid-template-columns: 1fr; }}
                }}
            </style>
        </head>
        <body>
            <div class="container">
                <div class="header">
                    <h1>🧪 测试报告</h1>
                    <div class="meta">
                        <strong>项目:</strong> {data['metadata']['repo_name']} &nbsp;&nbsp;
                        <strong>分支:</strong> {data['metadata']['branch']} &nbsp;&nbsp;
                        <strong>时间:</strong> {data['metadata']['timestamp'][:19].replace('T', ' ')}
                    </div>
                </div>
                
                <div class="summary">
                    <div class="summary-card">
                        <h3>总测试数</h3>
                        <div class="number info">{data['summary']['total_tests']}</div>
                    </div>
                    <div class="summary-card">
                        <h3>通过测试</h3>
                        <div class="number success">{data['summary']['passed_tests']}</div>
                    </div>
                    <div class="summary-card">
                        <h3>失败测试</h3>
                        <div class="number failure">{data['summary']['failed_tests']}</div>
                    </div>
                    <div class="summary-card">
                        <h3>成功率</h3>
                        <div class="number {'success' if data['summary']['success_rate'] >= 80 else 'warning' if data['summary']['success_rate'] >= 60 else 'failure'}">
                            {data['summary']['success_rate']:.1f}%
                        </div>
                        <div class="progress-bar">
                            <div class="progress-fill" style="width: {data['summary']['success_rate']}%"></div>
                        </div>
                    </div>
                </div>
        """
        
        # 添加覆盖率部分
        if data.get('coverage'):
            coverage = data['coverage']
            line_rate = coverage.get('line_rate', 0)
            coverage_color = '#27ae60' if line_rate >= 80 else '#f39c12' if line_rate >= 60 else '#e74c3c'
            
            html_template += f"""
                <div class="coverage-section">
                    <h3>📊 代码覆盖率</h3>
                    <div class="coverage-bar">
                        <div class="coverage-fill" style="width: {line_rate}%; background: {coverage_color};"></div>
                        <div class="coverage-text">{line_rate:.1f}%</div>
                    </div>
                    <p style="margin-top: 15px; color: #7f8c8d;">
                        覆盖行数: {coverage.get('lines_covered', 0)} / {coverage.get('lines_valid', 0)}
                    </p>
                </div>
            """
        
        # 添加测试详情
        if data.get('test_suites'):
            html_template += """
                <div class="test-details">
                    <h3>📝 测试详情</h3>
            """
            
            for test in data['test_suites'][:50]:  # 限制显示前50个
                status_class = f"status-{test['status']}" if test['status'] in ['passed', 'failed', 'skipped'] else 'status-skipped'
                html_template += f"""
                    <div class="test-item">
                        <div class="test-name">{test['name']}</div>
                        <span class="test-status {status_class}">{test['status'].upper()}</span>
                        <span style="color: #7f8c8d; margin-left: 15px;">⏱ {test['duration']:.3f}s</span>
                """
                
                if test.get('error'):
                    html_template += f"""
                        <div style="margin-top: 10px; padding: 10px; background: #ffeaea; border-radius: 4px; font-family: monospace; font-size: 12px; color: #e74c3c;">
                            {test['error'][:200]}{'...' if len(test['error']) > 200 else ''}
                        </div>
                    """
                
                html_template += "</div>"
            
            if len(data['test_suites']) > 50:
                html_template += f"""
                    <div style="text-align: center; padding: 20px; color: #7f8c8d;">
                        ... 还有 {len(data['test_suites']) - 50} 个测试用例
                    </div>
                """
            
            html_template += "</div>"
        
        # 添加产物链接
        if data.get('artifacts'):
            html_template += """
                <div class="artifacts">
                    <h3>📎 测试产物</h3>
            """
            for artifact in data['artifacts']:
                html_template += f'<a href="{artifact}" class="artifact-item">{os.path.basename(artifact)}</a>'
            html_template += "</div>"
        
        html_template += """
            </div>
        </body>
        </html>
        """
        
        # 保存HTML报告
        with open('test-report.html', 'w', encoding='utf-8') as f:
            f.write(html_template)
        
        print("✅ HTML报告生成完成: test-report.html")
```

### 5. 部署配置

#### 一键部署脚本
```bash
#!/bin/bash
# deploy-tekton-testing-platform.sh

set -e

NAMESPACE="testing-platform"
DOMAIN="${DOMAIN:-tekton-test.local}"

echo "🚀 部署Tekton云原生测试平台..."

# 1. 创建命名空间
echo "📦 创建命名空间..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# 2. 安装Tekton核心组件
echo "🔧 安装Tekton Pipelines..."
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

echo "🔧 安装Tekton Triggers..."
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

echo "🔧 安装Tekton Dashboard..."
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# 3. 等待核心组件就绪
echo "⏳ 等待Tekton组件启动..."
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines --timeout=300s
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines-resolvers --timeout=300s

# 4. 部署平台组件
echo "🏗️ 部署平台组件..."

# 创建服务账户
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers-sa
  namespace: $NAMESPACE
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tekton-triggers-role
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["eventlisteners", "triggerbindings", "triggertemplates"]
    verbs: ["get", "list", "create", "update", "delete"]
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns", "taskruns"]
    verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-triggers-binding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers-sa
    namespace: $NAMESPACE
roleRef:
  kind: ClusterRole
  name: tekton-triggers-role
  apiGroup: rbac.authorization.k8s.io
EOF

# 5. 部署Pipeline资源
echo "📝 部署Pipeline和Task资源..."
kubectl apply -f tekton-resources/ -n $NAMESPACE

# 6. 配置Dashboard访问
echo "🌐 配置Dashboard访问..."
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dashboard
  namespace: tekton-pipelines
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
---
apiVersion: networking.k8s.io/v1
kind: Ingress  
metadata:
  name: tekton-webhook
  namespace: $NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: webhook.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: el-git-webhook-listener
            port:
              number: 8080
EOF

# 7. 创建存储类（如果不存在）
echo "💾 检查存储类..."
if ! kubectl get storageclass fast-ssd &>/dev/null; then
  echo "创建存储类..."
  kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zones: us-central1-a,us-central1-b
allowVolumeExpansion: true
EOF
fi

echo "✅ 部署完成！"
echo ""
echo "🌐 访问地址:"
echo "  Tekton Dashboard: http://$DOMAIN"
echo "  Webhook URL: http://webhook.$DOMAIN"
echo ""
echo "📋 下一步:"
echo "  1. 在Git平台配置webhook指向: http://webhook.$DOMAIN"
echo "  2. 访问Dashboard查看Pipeline执行状态"
echo "  3. 推送代码触发测试"
echo ""
echo "🔍 本地访问(如果没有外部域名):"
echo "  kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097"
echo "  kubectl port-forward -n $NAMESPACE svc/el-git-webhook-listener 8080:8080"
```

## 📊 轻量级报告特性

### 相比Allure的优势

| 特性 | Allure | 轻量级方案 |
|------|--------|------------|
| **安装复杂度** | 中等 | 极简 |
| **资源占用** | 较高 | 极低 |
| **依赖关系** | Java + 多个库 | 仅Python标准库 |
| **报告大小** | 较大 | 小巧 |
| **加载速度** | 中等 | 快速 |
| **定制能力** | 固定模板 | 完全可定制 |
| **云原生** | 需要额外配置 | 原生支持 |

### 报告功能特性

1. **统一数据格式** - JSON结构化数据
2. **美观HTML** - 响应式设计，支持移动端
3. **实时图表** - CSS动画进度条和状态展示
4. **详细信息** - 失败用例错误信息
5. **覆盖率展示** - 可视化覆盖率数据
6. **产物链接** - 直接访问测试产物
7. **趋势分析** - 可扩展历史数据对比

## 🎯 实施路线图

### 第1周: 基础环境
- [ ] Kubernetes集群准备
- [ ] Tekton组件安装
- [ ] 基础Pipeline配置

### 第2周: 核心功能
- [ ] webhook触发配置
- [ ] 测试类型检测
- [ ] pytest/go test/shell test支持

### 第3周: 报告系统
- [ ] 轻量级报告生成
- [ ] HTML模板优化
- [ ] Dashboard集成

### 第4周: 生产就绪
- [ ] 安全配置
- [ ] 性能优化
- [ ] 用户培训

## 💰 成本估算

```yaml
基础设施成本:
  kubernetes_cluster: "$300-600/月"
  storage: "$50-100/月" 
  networking: "$30-50/月"

开发成本:
  setup_time: "2-4周"
  maintenance: "低"
  training: "中等"

年度总成本: "$15,000-25,000"
```

## 📈 预期收益

```yaml
效率提升:
  自动化测试: "节省80%手工测试时间"
  快速反馈: "5-10分钟获得测试结果"
  统一界面: "减少50%工具切换时间"

质量改善:
  一致性测试: "100%标准化测试流程"
  及时发现: "提前发现90%潜在问题"
  覆盖率提升: "提高30%代码覆盖率"
```

这个方案专注于云原生和轻量级，避免了过度工程化，同时提供了统一的UI体验和完整的测试能力。 