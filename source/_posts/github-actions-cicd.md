---
title: GitHub Actions 自动化部署实战
date: 2026-02-20 14:30:00
tags:
  - GitHub Actions
  - CI/CD
  - DevOps
  - 自动化
categories:
  - 技术分享
keywords: GitHub Actions, CI/CD, 自动化部署, DevOps
description: 从零开始配置 GitHub Actions 工作流，实现前端项目的自动化测试、构建和部署
---

## 什么是 GitHub Actions

GitHub Actions 是 GitHub 提供的 CI/CD 服务，允许你在代码仓库中直接配置自动化工作流。无需额外服务器，完全免费（公开仓库无限制，私有仓库有免费额度）。

## 基础概念

### Workflow（工作流）
定义在 `.github/workflows/` 目录下的 YAML 文件，描述自动化任务的配置。

### Job（任务）
工作流中的执行单元，可以串行或并行运行。

### Step（步骤）
任务中的具体执行步骤，按顺序执行。

### Action（动作）
可复用的代码单元，GitHub Marketplace 提供了数千个现成 Action。

## 实战配置

### 场景一：前端项目自动部署

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to GitHub Pages

# 触发条件
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

# 权限设置
permissions:
  contents: read
  pages: write
  id-token: write

# 并发控制
concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    # 1. 检出代码
    - name: Checkout
      uses: actions/checkout@v4
      
    # 2. 设置 Node.js
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
        
    # 3. 安装依赖
    - name: Install dependencies
      run: npm ci
      
    # 4. 运行测试
    - name: Run tests
      run: npm test
      
    # 5. 构建项目
    - name: Build
      run: npm run build
      
    # 6. 上传构建产物
    - name: Upload artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    
    steps:
    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

### 场景二：多环境部署

```yaml
name: Multi-Environment Deploy

on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop' || github.event.inputs.environment == 'staging'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.goodnick123.io
    
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: |
          echo "Deploying to staging..."
          # 部署脚本

  deploy-production:
    if: github.ref == 'refs/heads/main' || github.event.inputs.environment == 'production'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://goodnick123.io
    needs: [deploy-staging]
    
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: |
          echo "Deploying to production..."
          # 部署脚本
```

### 场景三：自动发布 Release

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install and Build
      run: |
        npm ci
        npm run build
    
    - name: Create Release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false
    
    - name: Upload Release Asset
      uses: actions/upload-release-asset@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create_release.outputs.upload_url }}
        asset_path: ./dist/app.zip
        asset_name: app.zip
        asset_content_type: application/zip
```

## 进阶技巧

### 1. 缓存优化

```yaml
- name: Cache node_modules
  uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ${{ github.workspace }}/node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### 2. 矩阵构建

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x]
    os: [ubuntu-latest, windows-latest]
    
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

### 3. 使用 Secrets

```yaml
- name: Deploy with API Key
  env:
    API_KEY: ${{ secrets.API_KEY }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    echo "Deploying with secrets..."
    deploy-cli --api-key $API_KEY
```

### 4. 条件执行

```yaml
- name: Skip for dependabot
  if: github.actor == 'dependabot[bot]'
  run: echo "Skipping for dependabot"

- name: Run on main only
  if: github.ref == 'refs/heads/main'
  run: echo "Running on main branch"

- name: Check file changes
  if: contains(github.event.head_commit.modified, 'src/')
  run: echo "Source files changed"
```

## 调试技巧

### 1. 启用调试日志

在仓库设置中添加 `ACTIONS_STEP_DEBUG` secret，值为 `true`。

### 2. 本地测试

使用 `act` 工具本地运行 GitHub Actions：

```bash
# 安装 act
brew install act

# 运行默认工作流
act

# 运行特定 job
act -j build

# 使用特定事件
act push
act pull_request
```

### 3. 查看日志

```yaml
- name: Debug Info
  run: |
    echo "Event name: ${{ github.event_name }}"
    echo "Ref: ${{ github.ref }}"
    echo "Actor: ${{ github.actor }}"
    echo "Commit message: ${{ github.event.head_commit.message }}"
```

## 最佳实践

1. **最小权限原则**：只授予必要的权限
2. **失败快速**：将轻量级检查放在前面
3. **并发控制**：使用 concurrency 防止冲突
4. **环境隔离**：生产环境需要人工审批
5. **版本锁定**：使用特定版本的 Action

## 总结

GitHub Actions 让 CI/CD 变得简单高效。通过合理配置，你可以实现：

- ✅ 自动化测试
- ✅ 持续集成
- ✅ 自动部署
- ✅ 版本发布
- ✅ 代码质量检查

开始使用 GitHub Actions，让开发流程更加高效！

---

**相关资源**
- [GitHub Actions 官方文档](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
