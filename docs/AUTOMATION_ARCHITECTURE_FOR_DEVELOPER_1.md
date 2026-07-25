# FTC 32477 团队文档自动化发布与构建架构指南

本文档详细说明了从 **GitBook Cloud** 编辑，到 **GitHub 仓库** 托管源码，再到通过 **GitHub Actions** 自动编译并同步至 **GitHub Pages** 的全套自动化工作流。

---

## 📐 1. 系统全景架构图

```text
[ GitBook Cloud (app.gitbook.com) ]
       │
       │ 1. 团队可视化编辑并保存 (GitHub Sync 双向同步)
       ▼
[ 源仓库: ftc32477/quick-start-guide (main 分支) ]
       │
       │ 2. 代码提交触发 GitHub Actions 工作流
       ▼
[ GitHub Actions CI/CD Runner (Ubuntu) ]
       │
       ├── ⚙️ 环境依赖配置:
       │     ├── Node.js 22 LTS
       │     ├── HonKit (GitBook 现代化开源替代)
       │     ├── Calibre 引擎 (PDF 渲染)
       │     └── 中文字体库 (fonts-wqy-zenhei, fonts-noto-cjk)
       │
       ├── 🛠️ 步骤 1: 编译 HTML 静态网站 -> _book/
       └── 🛠️ 步骤 2: 导出 PDF 指南 -> _book/team-guide.pdf
       │
       │ 3. 使用跨仓库凭证 (GH_PAT) 提交更新
       ▼
[ 目标仓库: ftc32477/ftc32477.github.io (main 分支) ]
       └── /docs/ 文件夹 (包含编译好的网页与 PDF)
       │
       │ 4. GitHub Pages 自动部署托管
       ▼
[ 🌐 最终访问端点 ]
       ├── 在线说明书网站: https://ftc32477.github.io/docs/
       └── PDF 离线下载:   https://ftc32477.github.io/docs/team-guide.pdf
```

---

## 📁 2. 涉及的相关文件路径与仓库结构

完整系统由两个 GitHub 仓库共同协同工作：

### 2.1 源仓库: `ftc32477/quick-start-guide`
用于存放文档 Markdown 源码及自动化编译配置文件。

```text
quick-start-guide/
├── README.md                          # 📍 文档首页 / 介绍页（包含 PDF 下载链接）
├── SUMMARY.md                         # 📍 GitBook / HonKit 核心目录树配置文件
├── book.json                          # 📍 HonKit & PDF 构建属性配置（字号、页边距等）
├── styles/
│   └── pdf.css                        # 📍 PDF 专属自定义 CSS 样式表（标题颜色、字体、行高）
├── .github/
│   └── workflows/
│       └── sync-docs.yml              # 📍 GitHub Actions 自动化编译与同步工作流脚本
└── AUTOMATION_ARCHITECTURE.md        # 📍 [本文档] 团队内部运维架构指南（不加入 SUMMARY.md）
```

### 2.2 目标仓库: `ftc32477/ftc32477.github.io`
主站仓库，负责通过 GitHub Pages 开放对外访问。

```text
ftc32477.github.io/
└── docs/                              # 📍 自动由 Actions 覆盖更新的静态资源发布目录
    ├── index.html                     # 编译后的网页入口
    ├── team-guide.pdf                 # 编译生成的最新 PDF 指南文件
    └── gitbook/                       # 网页运行所需的静态样式与脚本资源
```

---

## ⚙️ 3. 自动化核心原理与工作流程

### 步骤一：GitBook Cloud 编辑与同步 (GitHub Sync)
* **原理**：GitBook 官方提供了与 GitHub 仓库的双向同步插件。
* **流程**：队员在 `app.gitbook.com` 上像写 Word/Notion 一样直接修改文档，点击发布后，GitBook 会以 Bot 名义向 `ftc32477/quick-start-guide` 仓库的 `main` 分支提交一个 `git commit`。

### 步骤二：GitHub Actions 触发与编译
* **触发条件**：`.github/workflows/sync-docs.yml` 监听 `quick-start-guide` 仓库 `main` 分支的 `push` 事件。
* **工作流执行序列**：
  1. **环境初始化**：拉取 Ubuntu Runner，安装 Node.js 22。
  2. **安装中文字体与 Calibre**：运行 `sudo apt-get install -y calibre fonts-wqy-zenhei fonts-noto-cjk`。Calibre 是 HonKit 导出 PDF 所需的底层引擎，中文字体库解决渲染 PDF 时中文变空白或乱码的问题。
  3. **编译 HTML 网站**：执行 `npx honkit build ./ _book`，依据 `SUMMARY.md` 将 Markdown 转化为静态 HTML 文件夹 `_book`。
  4. **导出 PDF**：执行 `npx honkit pdf ./ _book/team-guide.pdf`，结合 `book.json` 与 `styles/pdf.css` 将全文打包渲染为一个 `.pdf` 文件并放入 `_book/` 目录中。

### 步骤三：跨仓库凭证与自动推送
* **认证凭证**：仓库中配置了名位 `GH_PAT` 的 Repository Secret，持有对组织内仓库的写入权限（Personal Access Token）。
* **同步逻辑**：
  1. 使用 `GH_PAT` 克隆 `ftc32477/ftc32477.github.io` 仓库。
  2. 清空目标仓库原有的 `docs/` 文件夹，将刚刚编译好的 `_book/` 内容复制到 `target-repo/docs/`。
  3. 检查是否有文件变更（`git diff`），若有变更则自动提交并 `git push` 到目标仓库的 `main` 分支。

### 步骤四：GitHub Pages 自动部署
* **原理**：`ftc32477.github.io` 启用了 GitHub Pages 服务，自动监测仓库中的文件变更，将 `docs/` 目录暴露给外部 Web 访问。

---

## 🛠️ 4. 维护与配置调整说明

| 维护需求 | 涉及文件 / 路径 | 修改方法与说明 |
| :--- | :--- | :--- |
| **修改文档目录节点** | `SUMMARY.md` | 在 Markdown 中增加/删除链接列表，定义左侧导航树 |
| **修改 PDF 页边距/默认字号** | `book.json` | 调整 `"fontSize"`、`"paperSize"`、`"margin"` 属性 |
| **修改 PDF 字体样式/颜色/排版** | `styles/pdf.css` | 修改 CSS 属性（如 `h1` 颜色、`line-height` 行高等） |
| **修改导出 PDF 的文件名** | `.github/workflows/sync-docs.yml` | 修改 `npx honkit pdf ./ _book/新文件名.pdf` |
| **修复跨仓库同步失败/权限错** | GitHub 个人设置 -> Token | 重新生成 PAT，并更新源仓库 Secrets 中的 `GH_PAT` |

---

## 📌 5. 文档存放与“不影响主要内容发布”策略

为了让团队内部维护文档（如本文档）保留在 GitHub 仓库中，同时**不出现在对公众开放的说明书网页侧边栏**中，遵守以下规则：

1. **核心原则（SUMMARY.md 驱动）**：
   HonKit / GitBook 只会将显式写入 `SUMMARY.md` 中的 Markdown 文件渲染到网页侧边栏目录中。
2. **推荐方案**：
   * **方案 A（根目录放置，推荐）**：直接保存在根目录（如 `AUTOMATION_ARCHITECTURE.md` 或 `DEVELOPER_GUIDE.md`），只要**不将它写入 `SUMMARY.md`**，HonKit 编译时就不会将其放入网站菜单中。
   * **方案 B（隐藏目录放置）**：保存在 `.github/` 文件夹下（如 `.github/ARCHITECTURE.md`）。以 `.` 开头的文件夹默认会被大多数静态编译器忽略。
   * **方案 C（内部文档专有文件夹）**：建立 `internal/` 或 `docs-dev/` 文件夹存放运维文档，同样不写入 `SUMMARY.md` 即可。
