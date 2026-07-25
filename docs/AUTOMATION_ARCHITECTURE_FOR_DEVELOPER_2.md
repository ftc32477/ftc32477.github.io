# Quick-Start-Guide 开发者自动化构建与部署架构文档 (HonKit + GitHub Actions)

本文档面向 `quick-start-guide` 项目的开发者与维护者，详细说明了基于 **HonKit** 与 **GitHub Actions** 构建的自动化编译与部署（CI/CD）架构流程。

---

## 1. 架构概述 (Architecture Overview)

本项目采用 GitHub 托管源码，通过 GitHub Actions 实现静态文档站点的持续集成与持续部署。开发者只需将 Markdown 源文件及配置提交至 `main` 分支，系统将自动触发 CI/CD Pipeline 编译 HonKit 电子书并将其部署至 GitHub Pages。

### 核心技术栈
- **文档引擎**：[HonKit](https://github.com/honkit/honkit) (GitBook 的现代分支，全面支持 Node.js LTS)
- **自动化 CI/CD**：GitHub Actions
- **托管平台**：GitHub Pages (使用官方 Actions 部署模型)
- **包管理器**：npm / npx

---

## 2. 自动化架构拓扑图 (Architecture Diagram)

```
[ 开发者 / 贡献者 ]
       │
       │ 1. Git Push / PR (main 分支)
       ▼
┌───────────────────────────────────────────────────────────┐
│                    GitHub Repository                      │
└───────────────────────────┬───────────────────────────────┘
                            │
                            │ 2. Trigger Event
                            ▼
┌───────────────────────────────────────────────────────────┐
│                 GitHub Actions Runner                     │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Step 1: Checkout Repository Source Code            │  │
│  ├─────────────────────────────────────────────────────┤  │
│  │ Step 2: Setup Node.js Environment & Enable Cache    │  │
│  ├─────────────────────────────────────────────────────┤  │
│  │ Step 3: Install Dependencies (`npm ci`)             │  │
│  ├─────────────────────────────────────────────────────┤  │
│  │ Step 4: HonKit Build (`npx honkit build`)           │  │
│  │         └─ Output: `./_book`                        │  │
│  ├─────────────────────────────────────────────────────┤  │
│  │ Step 5: Upload Artifact to GitHub Pages             │  │
│  └────────────────────────┬────────────────────────────┘  │
└───────────────────────────┼───────────────────────────────┘
                            │
                            │ 3. Deploy Artifact
                            ▼
┌───────────────────────────────────────────────────────────┐
│                      GitHub Pages                         │
│             (托管并渲染 `_book` 静态站点)                  │
└───────────────────────────────────────────────────────────┘
```

---

## 3. 工作流阶段与技术细节 (Pipeline Phases)

### Phase 1: 事件触发机制 (Trigger)
- **自动触发**：当代码被 `push` 到 `main` 分支时触发；或者针对 `main` 分支发起 `pull_request` 时运行校验编译。
- **手动触发**：配置 `workflow_dispatch` 允许开发者在 GitHub Web UI 手动点击运行。

### Phase 2: 环境准备与缓存 (Setup & Caching)
- **Node.js LTS 环境**：采用 Node.js v20+ 运行环境，确保与 HonKit 的完美兼容。
- **依赖缓存机制**：使用 `actions/setup-node` 内置的 `cache: 'npm'`，将 `package-lock.json` 对应的缓存锁定，显著提升打包编译速度。

### Phase 3: HonKit 编译流程 (HonKit Compilation)
1. **依赖安装**：通过 `npm ci` 确保在 CI 环境下安装严格一致的依赖包。
2. **文档编译**：运行 `npx honkit build ./ ./_book`。
   - 解析 `SUMMARY.md` 目录结构与 `book.json` 插件设置。
   - 将所有 `.md` 文件编译转换为 HTML、CSS、JS 静态资源输出至 `_book/` 目录。

### Phase 4: 原生 GitHub Pages 部署 (Deployment)
- **免 token 部署**：基于 GitHub 官方推崇的 `id-token: write` (OIDC Token) 与 `pages: write` 最小化权限模型。
- **步骤组件**：
  1. `actions/upload-pages-artifact`：将 `_book/` 打包上传。
  2. `actions/deploy-pages`：安全无缝发布至 GitHub Pages。

---

## 4. 配置文件参考 (CI/CD Workflow Configuration)

在仓库根目录新建 `.github/workflows/deploy.yml` 配置文件：

```yaml
name: Deploy HonKit Documentation

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

# 设置 GITHUB_TOKEN 的最小权限以支持安全部署
permissions:
  contents: read
  pages: write
  id-token: write

# 确保同一时间只有一个部署工作流在运行，阻止并发冲突
concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Setup Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Project Dependencies
        run: |
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi

      - name: Build Documentation with HonKit
        run: npx honkit build ./ ./_book

      - name: Upload GitHub Pages Artifact
        if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
        uses: actions/upload-pages-artifact@v3
        with:
          path: './_book'

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## 5. 项目根目录推荐配置 (Project Root Setup)

为了确保本地开发与 GitHub Actions CI 环境的高度一致性，建议在项目根目录下准备以下配置文件：

### 1. `package.json`
确保项目中声明了 HonKit 依赖以及便捷的运行脚本：
```json
{
  "name": "quick-start-guide",
  "version": "1.0.0",
  "description": "Quick Start Guide Documentation",
  "main": "index.js",
  "scripts": {
    "build": "honkit build ./ ./_book",
    "serve": "honkit serve ./",
    "clean": "rimraf _book"
  },
  "devDependencies": {
    "honkit": "^6.0.0"
  }
}
```

### 2. `.gitignore`
避免将编译生成产物或依赖提交至源码库：
```gitignore
node_modules/
_book/
.honkit/
*.log
```

---

## 6. 本地开发与 CI 协同规范 (Local vs CI Parity)

1. **本地调试文档**：
   在本地根目录运行：
   ```bash
   npm install
   npm run serve
   ```
   然后访问 `http://localhost:4000` 实时预览改动。

2. **本地测试编译**：
   在提交代码前，建议运行：
   ```bash
   npm run build
   ```
   验证 `_book` 目录能否正常生成且无报错。

---

## 7. 最佳实践与注意事项 (Best Practices)

1. **GitHub Pages 仓库设置**：
   进入仓库 **Settings -> Pages -> Build and deployment**：
   - **Source** 选择：`GitHub Actions`（而非 Deploy from a branch）。
2. **插件兼容性**：
   如果在 `book.json` 中使用了额外的 GitBook/HonKit 插件，请尽量通过 `package.json` 管理依赖，避免在 CI 运行时临时从网络拉取非标准插件导致构建不稳定。
3. **文件名规范**：
   Linux Runner 对文件名大小写敏感，请确保 `SUMMARY.md` 中引用的文件路径与真实 Markdown 文件路径完全一致。
