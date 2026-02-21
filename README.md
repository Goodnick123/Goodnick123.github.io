# Goodnick's Tech Blog

基于 Hexo 构建的技术博客，记录学习心得与项目实践。

## 🚀 快速开始

### 1. 环境准备

确保已安装 Node.js (v18+) 和 Git：

```bash
# 检查 Node.js 版本
node --version  # v18.0.0 或更高

# 检查 Git
git --version
```

### 2. 安装 Hexo

```bash
# 全局安装 Hexo CLI
npm install -g hexo-cli

# 验证安装
hexo --version
```

### 3. 本地运行

```bash
# 进入项目目录
cd goodnick-hexo-blog

# 安装依赖
npm install

# 启动本地服务器
hexo server

# 访问 http://localhost:4000
```

### 4. 新建文章

```bash
# 使用命令创建新文章
hexo new post "文章标题"

# 或手动在 source/_posts/ 目录下创建 .md 文件
```

文章格式示例：

```markdown
---
title: 文章标题
date: 2026-02-21 10:00:00
tags:
  - Tag1
  - Tag2
categories:
  - 技术分享
---

正文内容...
```

## 📁 项目结构

```
.
├── _config.yml          # 站点配置文件
├── package.json         # 依赖配置
├── source/              # 源码目录
│   ├── _posts/          # 文章目录
│   ├── about/           # 关于页面
│   └── categories/      # 分类页面
├── themes/              # 主题目录
├── scaffolds/           # 脚手架
└── public/              # 生成的静态文件
```

## 🎨 主题配置

当前使用默认的 Landscape 主题，可在 `_config.yml` 中修改：

```yaml
theme: landscape
```

### 推荐主题

1. **NexT** - 流行的中文主题
   ```bash
   git clone https://github.com/next-theme/hexo-theme-next themes/next
   ```

2. **Butterfly** - 简洁美观
   ```bash
   git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
   ```

3. **Fluid** - Material Design
   ```bash
   git clone https://github.com/fluid-dev/hexo-theme-fluid.git themes/fluid
   ```

## 📝 写作指南

### Markdown 语法

Hexo 支持标准 Markdown 语法：

```markdown
# 标题
## 二级标题
### 三级标题

**粗体** *斜体* ~~删除线~~

- 列表项 1
- 列表项 2
  - 子项

1. 有序列表
2. 第二项

[链接文本](https://example.com)

![图片描述](图片地址)

> 引用内容

`代码` ```代码块```
```

### 代码高亮

支持多种编程语言的语法高亮：

```javascript
const hello = () => {
  console.log('Hello, Hexo!');
};
```

### 文章元数据

```yaml
---
title: 文章标题（必填）
date: 2026-02-21 10:00:00（自动生成）
updated: 2026-02-21 12:00:00（可选）
tags:
  - 标签1
  - 标签2
categories:
  - 分类1
  - 子分类
keywords: SEO 关键词
description: 文章描述
---
```

## 🌐 部署到 GitHub Pages

### 1. 配置部署

编辑 `_config.yml`：

```yaml
deploy:
  type: git
  repo: https://github.com/Goodnick123/Goodnick123.github.io.git
  branch: main
```

### 2. 安装部署插件

```bash
npm install hexo-deployer-git --save
```

### 3. 部署

```bash
# 清理缓存
hexo clean

# 生成静态文件
hexo generate

# 部署到 GitHub Pages
hexo deploy
```

### 4. 使用 GitHub Actions 自动部署

创建 `.github/workflows/pages.yml`：

```yaml
name: Pages

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          submodules: recursive
          
      - name: Use Node.js 20
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          
      - name: Cache NPM dependencies
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
            
      - name: Install Dependencies
        run: npm install
        
      - name: Build
        run: npm run build
        
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

## 🔧 常用命令

```bash
# 启动本地服务器
hexo server
hexo s

# 生成静态文件
hexo generate
hexo g

# 部署网站
hexo deploy
hexo d

# 清理缓存文件
hexo clean

# 组合命令（清理 + 生成 + 部署）
hexo clean && hexo g && hexo d

# 新建文章
hexo new post "文章标题"

# 新建页面
hexo new page "页面名称"

# 草稿
hexo new draft "草稿标题"
hexo publish "草稿标题"  # 发布草稿
```

## 🎨 自定义样式

在 `source/css/` 目录下创建自定义样式文件：

```css
/* source/css/custom.css */
.post-title {
  color: #333;
  font-size: 2em;
}

.post-content p {
  line-height: 1.8;
}
```

在主题配置中引入自定义样式。

## 📊 SEO 优化

### 已配置

- ✅ Sitemap 生成
- ✅ Feed 订阅
- ✅ 文章关键词和描述
- ✅ 友好的 URL 结构

### 额外建议

1. 注册 [Google Search Console](https://search.google.com/search-console)
2. 添加 `google-site-verification` 到头部
3. 创建 `robots.txt`
4. 优化文章 meta 信息

## 🐛 常见问题

### 1. 部署失败

```bash
# 确保配置了正确的仓库地址
git remote -v

# 检查分支名称
hexo deploy 默认推送到 main 分支
```

### 2. 样式不生效

```bash
# 清理缓存后重新生成
hexo clean && hexo generate
```

### 3. 本地图片不显示

使用相对路径：
```markdown
![描述](./image.png)
```

或使用 Hexo 资源文件夹功能。

## 📚 学习资源

- [Hexo 官方文档](https://hexo.io/docs/)
- [Hexo 主题列表](https://hexo.io/themes/)
- [Markdown 指南](https://www.markdownguide.org/)

## 📝 写作计划

- [x] TypeScript 高级类型
- [x] GitHub Actions CI/CD
- [x] Node.js 性能优化
- [ ] React 最佳实践
- [ ] 微服务架构设计
- [ ] Docker 入门到实践
- [ ] Kubernetes 基础
- [ ] 前端工程化

## 📄 License

MIT License © Goodnick

---

**Happy Writing!** 🎉
