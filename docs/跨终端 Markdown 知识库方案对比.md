# 跨终端 Markdown 知识库 — 可行方案对比

> 目标：搭建一个 **电脑、平板、手机** 都能用的 Markdown 笔记知识库，零成本或低成本，可自托管。

---

## 一、方案总览

| 方案 | 类型 | 部署难度 | 实时协作 | 离线可用 | 推荐场景 |
|------|:----:|:--------:|:--------:|:--------:|----------|
| **MkDocs** | 静态生成 | ⭐⭐ | ❌ | ✅（静态文件） | 个人/团队文档站 |
| **Docsify** | 动态加载 | ⭐ | ❌ | ❌ | 轻量个人知识库 |
| **VitePress** | 静态生成 | ⭐⭐ | ❌ | ✅ | 技术文档站 |
| **Outline** | 全功能知识库 | ⭐⭐⭐⭐ | ✅ | ❌ | 团队协作知识库 |
| **Wiki.js** | Wiki引擎 | ⭐⭐⭐ | ✅ | ❌ | 企业级Wiki |
| **BookStack** | Wiki引擎 | ⭐⭐⭐ | ✅ | ❌ | 结构化知识管理 |
| **Obsidian + 同步** | 本地+云同步 | ⭐ | ❌ | ✅ | 个人重度笔记用户 |

---

## 二、推荐方案详解

### 方案 A：MkDocs + Material（⭐ 首选推荐）

**适合场景**：个人技术笔记、项目文档，想要**干净美观的静态网站**。

```
$ pip install mkdocs mkdocs-material
$ mkdocs new my-docs
$ cd my-docs && mkdocs serve     # 本地预览 http://localhost:8000
$ mkdocs build                   # 生成静态文件
```

**目录结构**：
```
my-docs/
├── mkdocs.yml           # 配置文件
├── docs/
│   ├── index.md         # 首页
│   ├── 入门指南.md
│   ├── C#笔记/
│   │   ├── index.md
│   │   └── 异步回调.md
│   └── AGV仿真/
│       ├── index.md
│       └── 需求设计.md
└── site/                # build 输出，部署到服务器即可
```

**部署方式**：
| 方式 | 成本 | 说明 |
|------|:----:|------|
| GitHub Pages | 免费 | `mkdocs gh-deploy` 一键部署 |
| Gitee Pages | 免费 | 国内访问快 |
| Vercel / Netlify | 免费 | 自动从 Git 仓库构建 |
| 自建 Nginx/OSS | 低 | 放到任意静态文件服务器 |

**优势**：
- 纯静态，无需后端，加载极快
- Material 主题非常漂亮，支持搜索、导航、标签
- 本地 Markdown 文件即源，不锁定数据
- 可以把你已有的 NoteForge `.md` 直接搬过来

**配置文件示例**（`mkdocs.yml`）：

```yaml
site_name: 我的知识库
theme:
  name: material
  language: zh
  features:
    - navigation.tabs
    - navigation.sections
markdown_extensions:
  - pymdownx.highlight
  - pymdownx.superfences
  - tables
  - admonition
```

---

### 方案 B：Docsify（⭐ 最简单，零构建）

**适合场景**：想要**最简单**的方案，一个 HTML 搞定，无需构建步骤。

```bash
$ npm i -g docsify-cli
$ docsify init ./docs
$ docsify serve ./docs       # 本地预览
```

**目录**：
```
docs/
├── index.html        # 唯一的入口页面
├── README.md         # 首页内容
├── 指南.md
└── C#笔记/
    └── 异步回调.md
```

**工作原理**：
```
浏览器请求 index.html
  → docsify.js 加载
    → 解析 URL 中的路径
      → 用 Ajax 加载对应 .md 文件
        → 渲染成 HTML
```

**优势**：
- **零构建**，改 Markdown 文件即生效
- 部署 = 把文件夹扔到任意 HTTP 服务器
- 支持全文搜索、复制代码、emoji
- 可以嵌入 Vue 组件做扩展

**劣势**：
- 首次加载需要等服务端获取 Markdown 文件
- SEO 不友好（但个人知识库一般不需要）

---

### 方案 C：Outline（⭐ 最强功能，需 Docker）

**适合场景**：团队协作、需要多人编辑、权限管理。

```bash
docker-compose up -d   # 一键启动
```

**特性**：
- 实时协作编辑（类似 Notion）
- Markdown 原生支持 + 快捷键
- 支持 SSO/OAuth 登录
- 嵌套文档树 + 拖拽排序
- 全文搜索 + 收藏
- 支持导出 Markdown（数据不锁定）

**部署要求**：
- 需要 Docker + 2GB 内存
- 需要 PostgreSQL + S3/OSS 存储
- 需要域名（用于登录认证）

**适合**：有服务器运维能力、需要多人共享的场景。

---

### 方案 D：VitePress（⭐ 前端开发者优先）

**适合场景**：前端技术栈、需要极快的加载速度和 Vue 组件支持。

```bash
$ npm create vitepress my-docs
$ cd my-docs && npm run docs:dev
```

**特性**：
- Vite 驱动，开发体验极好
- 默认支持全文搜索
- 支持 Vue 组件嵌入（可以写交互示例）
- 支持国际化
- 比 MkDocs 更现代，生态不如 MkDocs 丰富

---

## 三、数据迁移兼容性

### 你已有的 NoteForge 笔记如何迁移

NoteForge 中的笔记是纯 Markdown + JSON 元数据，迁移非常方便：

| 原始位置 | 迁移方式 |
|----------|----------|
| `.nfbook` JSON | 提取 notes 中的 content 字段，另存为 `.md` |
| 单个 `.md` 文件 | **直接复制**，零修改 |
| 图片/附件 | 放到 `docs/assets/` 目录，引用路径改为相对路径 |

**NoteForge → MkDocs 迁移脚本思路**：

```python
import json, os

with open("ProfControl.nfbook", encoding="utf-8") as f:
    book = json.load(f)

for note_id, note in book["notes"].items():
    title = note["title"].replace("/", "／")
    with open(f"docs/{title}.md", "w", encoding="utf-8") as out:
        out.write(note["content"])
```

---

## 四、推荐实施路线

### 个人使用（推荐方案）→ MkDocs + Material

```
第1步：pip install mkdocs mkdocs-material
第2步：mkdocs new knowledge-base
第3步：把 NoteForge 的 .md 文件复制到 docs/ 目录
第4步：配置 mkdocs.yml（导航、主题、插件）
第5步：mkdocs serve 本地预览
第6步：推送到 GitHub → GitHub Pages 自动发布
```

**跨终端访问**：
- **电脑**：浏览器访问 GitHub Pages 地址 / 本地 `mkdocs serve`
- **手机/平板**：浏览器打开即可，Material 主题自适应移动端
- **离线**：`mkdocs build` 生成的静态文件可以放到本地任何地方打开

### 团队协作 → Outline

```
第1步：准备 Linux 服务器 + Docker
第2步：docker-compose up -d 启动 Outline
第3步：配置 OAuth 登录（GitHub/GitLab）
第4步：导入现有 Markdown 笔记
第5步：团队成员通过浏览器访问
```

---

## 五、最终建议

| 你的情况 | 推荐方案 |
|----------|----------|
| 个人技术笔记，想要快速上线 | ⭐ **MkDocs + Material**（GitHub Pages 部署） |
| 不想学新工具，最小成本 | ⭐ **Docsify**（改 `.md` 文件即可用） |
| 团队协作，实时编辑 | **Outline**（Docker 部署） |
| 前端技术栈，想要极致体验 | **VitePress** |
| 重度本地用户，偶尔同步 | **Obsidian** + 自己的 WebDAV/S3 同步 |

**我个人最推荐 MkDocs + Material**，理由：
1. 你已有大量 `.md` 文件（NoteForge 格式），可以直接搬
2. 静态站加载最快，手机电脑体验都好
3. GitHub Pages 免费托管，不需要服务器
4. Material 主题中文支持好，有搜索和导航
5. 不锁定数据——文件就是纯 Markdown

---

## 附录：快速起步命令

### MkDocs 一句话起步

```bash
pip install mkdocs mkdocs-material
mkdocs new my-notes
cd my-notes
# 编辑 mkdocs.yml 添加 material 主题
echo 'theme: 
  name: material
  language: zh' > mkdocs.yml
mkdir -p docs/C#笔记 docs/AGV仿真
# 复制你的 .md 文件到 docs/ 下
mkdocs serve
```

浏览器打开 `http://localhost:8000` 即可查看。

### GitHub Pages 发布

```bash
cd my-notes
mkdocs gh-deploy
# 访问 https://你的用户名.github.io/my-notes/
```

---

*笔记生成日期：2026-06-22*
