# fbs_knowlegebase · 无痕未来知识库

一个聚焦非自杀性自伤行为（NSSI）议题的内容知识库，包含亲历者故事、研究综述、行动组织等四个板块。

**线上访问**

- 站点：<https://ly.futurebeyondscars.cn/>（GitHub Pages 镜像：<https://karen0758.github.io/knowledgebase/>）
- 后台：<https://karen0758.github.io/knowledgebase/admin/>

本文档的重点不在"内容"本身，而在**这个站点的前端展示和后台管理是怎么搭起来的** —— 一套零服务器、零运维成本、可被任何想做内容站的人复刻的方案。

---

## 一、整体技术栈

| 层 | 选型 | 作用 |
|---|---|---|
| 前端展示 | **Hugo** | 静态站点生成器，把 markdown 编译成 HTML |
| 后台管理 | **Sveltia CMS** | 纯浏览器内容编辑器，直接读写 GitHub 仓库 |
| 数据存储 | **GitHub 仓库** | 仓库本身就是数据库，所有内容都是 markdown |
| 构建部署 | **GitHub Actions + Pages** | push 自动触发构建并发布 |
| 域名 | **CNAME** | GitHub Pages 绑定自定义域名 |

整条链路**没有任何长期运行的服务器进程**：CMS 是浏览器端的应用，每一次保存都是对 GitHub 仓库的一次 commit，commit 又触发 Actions 重建静态站点。从前端到后台没有一台需要自己维护的机器。

```
┌───────────────┐   编辑/提交        ┌───────────────┐
│  Sveltia CMS  │ ─────────────────▶ │  GitHub Repo  │
│  (浏览器后台)  │  通过 GitHub API   │   (main 分支)  │
└───────────────┘                    └───────┬───────┘
        ▲                                    │ push 触发
        │ GitHub OAuth                       ▼
┌───────────────┐                    ┌───────────────┐
│  OAuth App    │                    │GitHub Actions │
│   + 代理       │                    │  hugo build   │
└───────────────┘                    └───────┬───────┘
                                             │ 部署 ./public
                                             ▼
                                     ┌───────────────┐
                                     │ GitHub Pages  │
                                     │   + CNAME     │
                                     └───────────────┘
```

---

## 二、知识库前端是怎么搭的

### 2.1 为什么用 Hugo（而不是 Next.js / WordPress）

- **纯静态**：编译产物是 HTML/CSS/JS，可以白嫖 GitHub Pages，没有任何主机费用
- **内容即文件**：所有文章都是 `content/` 目录下的 markdown，没有数据库
- **构建快**：Hugo 是 Go 写的，本项目几十篇内容编译只需要几百毫秒
- **不依赖前端框架**：模板用 Go template，写一个 `index.html` 就能跑

### 2.2 单页架构（这个项目的特别之处）

大部分 Hugo 站点会写 `single.html` + `list.html`，每篇文章一个独立 URL。**本项目反其道而行**：

```
layouts/
├── index.html              ← 整个站点只有这一个模板
└── _default/_markup/
    └── render-image.html   ← 图片渲染器（自动 lazy load）
```

整个站点就是一个 856 行的 `layouts/index.html`，所有内容（四个板块、几十篇文章）在**构建期被全部预渲染到这一个 HTML 页面**，前端通过 CSS + 少量 JS 实现 tab 切换和侧栏列表点击。

为什么这样设计：

- 内容量适中（几十篇），全部塞进一页 HTML 也只有几百 KB
- 一次加载，零跳转，切换极快，体验更像桌面应用而不是网页
- 视觉上是"书桌+折叠纸张"的拟物风格，单页布局比多页路由更适合表达
- 不需要做任何客户端路由 / SPA 框架

代价是：

- 内容若进一步增长（数百篇以上）就该拆成多页
- 没有独立 URL 不利于 SEO 和分享单篇 —— 但本项目以"整体浏览"为主，可以接受

### 2.3 内容是怎么组织的

每个板块对应 `content/` 下的一个目录，每个目录里的 markdown 就是一篇内容：

```
content/
├── stories/        亲历者故事    caroline.md, emma.md, julian.md, juliet.md
├── research/       研究综述      china-nssi-review.md, how-to-stop.md
├── activities/     行动组织      flow-art.md, mental-health-day.md, ...
└── pages/          独立页面      resources.md, help.md
```

每篇 markdown 用 front matter（YAML 头）声明字段，例如 `content/research/china-nssi-review.md`：

```markdown
---
title: 中国非自杀性自伤行为研究综述
weight: 1                      # 数字越小越靠前
preview: 侧边栏摘要文字
meta_label: 研究综述
subtitle: 副标题（英文原标题等）
---

正文...
```

模板用 `{{ range (where site.RegularPages "Section" "research").ByWeight }}` 一次性把同一 section 的所有文章按 weight 排序渲染出来。

**关键约定**：这些 front matter 字段名必须和后台 CMS 配置（`static/admin/config.yml`）声明的字段一一对应 —— 后台填什么字段，模板里就能读什么字段。这是整个系统能跑通的核心约定。

---

## 三、知识库后端管理是怎么搭的

### 3.1 为什么不用 WordPress / Strapi / 自建后端

传统 CMS 都需要一台服务器跑后端 + 一个数据库 —— 这意味着持续的运维成本、安全更新、备份。对于一个志愿者维护的小型内容站来说不可承受。

**Sveltia CMS 的核心思路是**：把后台 UI 也变成静态文件，所有"写"操作直接通过 GitHub API 提交 commit。这样：

- 后台部署 = 把两个文件放到 `static/admin/` 下
- 鉴权 = 用户的 GitHub 账号
- 数据持久化 = git commit 到仓库
- 没有任何后端代码、没有任何数据库、没有任何运维

代价是：

- 编辑者必须有 GitHub 账号并被授权访问仓库
- 每次保存有 1–2 分钟的"等 CI 构建"延迟（不像传统 CMS 即时生效）

### 3.2 后台只有两个文件

**`static/admin/index.html`** —— 入口 HTML，从 unpkg 加载 CMS：

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8" /><title>无痕未来 · 内容管理</title></head>
<body>
  <script src="https://unpkg.com/@sveltia/cms/dist/sveltia-cms.js"></script>
</body>
</html>
```

**`static/admin/config.yml`** —— 整个后台的"全部配置"，声明每个集合的字段：

```yaml
backend:
  name: github
  repo: Karen0758/knowledgebase
  branch: main

media_folder: "assets/images"
public_folder: "/images"
locale: "zh_Hans"

collections:
  - name: "research"
    label: "研究动态"
    folder: "content/research"
    create: true
    slug: "{{slug}}"
    fields:
      - { label: "标题",     name: "title",      widget: "string" }
      - { label: "排序权重", name: "weight",     widget: "number", value_type: "int" }
      - { label: "侧边栏预览文字", name: "preview", widget: "string" }
      - { label: "分类标签", name: "meta_label", widget: "string" }
      - { label: "副标题",   name: "subtitle",   widget: "string" }
      - { label: "正文",     name: "body",       widget: "markdown" }

  - name: "pages"
    label: "独立页面"
    files:                    # 固定文件集合（不允许新增，只编辑）
      - name: "resources"
        label: "网站资源"
        file: "content/pages/resources.md"
        fields:
          - { label: "标题", name: "title", widget: "string" }
          - { label: "正文", name: "body",  widget: "markdown" }
```

CMS 启动时读 `config.yml`，自动生成对应的后台表单 —— **后台 UI 完全由这份 YAML 决定**。

集合分两种类型：

| 类型 | 用法 | 例子 |
|---|---|---|
| `folder` | 在目录下不断新建文章 | 研究综述、亲历者故事、行动组织 |
| `files` | 固定编辑某几个文件 | "网站资源"、"求助入口"这种单页 |

### 3.3 字段约定（前后端的关键契合点）

`config.yml` 里 `fields.name` 必须 = markdown front matter 的键 = 模板 `{{ .Params.xxx }}` 里读的字段。三处必须完全一致，否则后台填了前台读不到：

```
config.yml: name = "subtitle"
   ↓
markdown front matter: subtitle: "副标题文字"
   ↓
layouts/index.html: {{ .Params.subtitle }}
```

新加一个字段 = 改 `config.yml` 加一行 fields + 在模板里读取。

### 3.4 编辑者用 GitHub OAuth 登录

Sveltia CMS 在浏览器里跑，**不能持有 OAuth client secret**（持有就泄露了），所以登录流程需要一个**OAuth 代理**来在服务端完成 token 交换：

```
1. 编辑者打开 /admin/
2. 点击登录 → 跳转到 GitHub 授权页
3. GitHub 回调到 OAuth 代理（一个 Cloudflare Worker）
4. 代理用 client_secret 换取 access token
5. token 回到浏览器，CMS 用它调 GitHub API
```

涉及的资源：

- **GitHub OAuth App**：在 GitHub Developer settings 里创建，得到 client_id / client_secret
- **OAuth 代理**：本项目用社区维护的 [`sveltia-cms-auth`](https://github.com/sveltia/sveltia-cms-auth) 部署在 Cloudflare Workers，免费
- **代理地址**：通过 `backend.base_url` 在 `config.yml` 里指给 CMS

**简化方案**：如果只是个人单机使用，可以跳过 OAuth 代理，登录时直接贴 GitHub **Personal Access Token** —— Sveltia CMS 也支持。

---

## 四、部署：GitHub Actions + Pages

`.github/workflows/hugo.yml` 把整条部署链路自动化。关键步骤：

```yaml
on:
  push:
    branches: [main]      # main 任何 push 都触发

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.161.1
    steps:
      - 安装 hugo extended
      - actions/checkout
      - actions/configure-pages         # 自动拿到 Pages 的 base_url
      - run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - actions/upload-pages-artifact (path: ./public)

  deploy:
    needs: build
    - actions/deploy-pages              # 部署到 Pages
```

注意 `--baseURL` 在 CI 里被动态覆盖：`hugo.toml` 里写的是自定义域名 `https://ly.futurebeyondscars.cn/`，CI 构建时 `actions/configure-pages` 输出 GitHub Pages 实际的 base URL 并通过命令行参数覆盖。这样**同一份代码既能部署到 Pages 默认域名，也能部署到自定义域名**。

自定义域名通过 `static/CNAME` 文件配置（内容就一行：`ly.futurebeyondscars.cn`），Hugo 会把 `static/` 下的所有文件原样拷到站点根目录，GitHub Pages 看到 CNAME 自动启用自定义域。

---

## 五、项目结构

```
fbs_knowlegebase/
├── hugo.toml                      站点配置（baseURL、title 等）
├── content/                       所有内容（markdown）
│   ├── stories/                   亲历者故事
│   ├── research/                  研究综述
│   ├── activities/                行动组织
│   └── pages/                     独立单页（resources、help）
├── layouts/
│   ├── index.html                 整个站点的模板（单页架构）
│   └── _default/_markup/
│       └── render-image.html      图片渲染钩子（lazy load）
├── static/
│   ├── admin/
│   │   ├── index.html             Sveltia CMS 入口
│   │   └── config.yml             后台字段配置
│   ├── images/                    静态图片
│   └── CNAME                      自定义域名
├── assets/                        CMS 上传的图片落地目录
├── .github/workflows/hugo.yml     GitHub Actions 部署流程
└── public/                        构建产物（gitignored）
```

---

## 六、本地开发

需要 [Hugo Extended](https://gohugo.io/installation/) ≥ 0.161.1。

```bash
git clone https://github.com/Karen0758/knowledgebase.git
cd knowledgebase
hugo server -D
```

- 前台：<http://localhost:1313/>
- 后台：<http://localhost:1313/admin/>（界面可预览，编辑提交需登录 GitHub）

改完 push 到 `main`，GitHub Actions 会自动重新部署。

---

## 协议

代码部分按 MIT 协议开源，欢迎参考这套搭建方案做自己的内容站；`content/` 下的文章版权归原作者所有。
