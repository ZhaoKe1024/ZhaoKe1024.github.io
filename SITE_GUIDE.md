# 个人主页维护手册（ZhaoKe1024.github.io）

这个仓库已经按 `ZhaoKe1024/README.md` 的内容改造成个人主页。本文件是后续自己改内容时的"操作地图"。

- 站点地址（部署后）：<https://ZhaoKe1024.github.io>
- 技术栈：Jekyll 4.4 + [academicpages](https://github.com/academicpages/academicpages.github.io) 主题（fork 自 Minimal Mistakes）
- 构建方式：GitHub Actions（`.github/workflows/jekyll.yml`），**不需要本地装任何东西**，push 即自动构建部署

---

## 1. 已经改好了什么

| 文件 | 改动 |
|:--|:--|
| `_config.yml` | 站点标题/描述/URL/仓库名、侧边栏作者信息（头像、邮箱、GitHub、单位、方向）、论文分类（Journal Articles / Conference Papers / Preprints）、时区 |
| `_pages/about.md` | **首页**：个人简介（中英）、研究方向、开源项目表、联系方式 —— 来源是你的 `README.md` |
| `_data/navigation.yml` | 顶部导航：Publications / Projects / Blog / CV（Talks、Teaching 暂时隐藏） |
| `_pages/portfolio.html` | 改为 Projects 页，地址 `/projects/` |
| `_portfolio/portfolio-01..04-*.md` | OPT-MCM、OptSkills、dsh-style-asoul、IntelligentAlgorithmScheduler 四个项目卡片 |
| `_publications/*.md` | 3 个示例条目（期刊 / 会议 / 预印本），**占位，等你换成真论文** |
| `_posts/2026-11-13-hello-world.md` | 第一篇博客（同时是写作模板） |
| `_pages/cv.md` | CV 页，已改成 ECNU 读博的骨架，年份/导师是占位 |
| `_talks`、`_teaching` | 各留 1 个占位条目，删掉了模板假数据 |

---

## 2. 目录速查：想改什么，去哪个文件

```
_config.yml                  ← 全站设置（标题、URL、侧边栏作者、导航之外的全局项）
_data/navigation.yml         ← 顶部菜单
_data/ui-text.yml            ← 界面文字（可切换中文，见 5.1）
_pages/about.md              ← 首页（permalink: /）
_pages/publications.html     ← /publications/ 论文列表页（一般不用动）
_pages/portfolio.html        ← /projects/ 项目列表页（一般不用动）
_pages/cv.md                 ← /cv/
_pages/year-archive.html     ← /year-archive/ 博客归档
_publications/*.md           ← 一篇论文一个文件
_posts/YYYY-MM-DD-*.md       ← 一篇博客一个文件
_drafts/                     ← 草稿（不发布）
_portfolio/*.md              ← 一个项目一个文件
_talks/  _teaching/          ← 报告 / 教学
images/                      ← 头像、配图、og-image
files/                       ← PDF、bib、slides 等可下载文件
```

---

## 3. 让网站真正运行起来

### 3.1 首次部署（推荐，零本地环境）

1. 把改动提交并推送到 GitHub：

   ```bash
   cd C:\Users\culle\Documents\PythonProjects\ZhaoKe1024.github.io
   git add -A
   git commit -m "Rewrite homepage from README; add projects, publications, blog"
   git push origin master
   ```

   推之前可以只看改了哪些文件：`git status`。

2. 打开仓库 <https://github.com/ZhaoKe1024/ZhaoKe1024.github.io> →
   **Settings → Pages → Build and deployment → Source** 选 **GitHub Actions**。
   （如果已经是 *Deploy from a branch*，一定要改成 **GitHub Actions**，因为本模板的
   `jekyll.yml` 用的是 Actions 部署，选分支会走另一条老路径。）
3. 打开 **Actions** 标签，看到 `Deploy Academic Pages template site to Pages` 在跑；
   第一次大约 1–3 分钟。绿勾后访问 <https://ZhaoKe1024.github.io>。
4. 之后再改任何 Markdown，只要 `git push`，Actions 会自动重新构建（约 1 分钟）。

> 重要前提：仓库名必须是 `ZhaoKe1024.github.io`（用户名 + `.github.io`），且默认分支是
> `master`（工作流里写死了 `branches: ["master"]`）。如果你以后改成 `main`，
> 记得同步改 `.github/workflows/jekyll.yml` 里的分支名。

### 3.2 本地预览（可选，需要装 Ruby）

Windows 上用 RubyInstaller 最省事：

1. 安装 Ruby 3.1–3.3（带 Devkit）：<https://rubyinstaller.org/downloads/>，
   安装器最后一步选 `MSYS2 and MINGW development toolchain`。
2. 在这个仓库目录执行：

   ```bash
   gem install bundler
   bundle install          # 首次会编译原生扩展，慢一点，属正常
   bundle exec jekyll serve -l
   ```

3. 浏览器打开 <http://127.0.0.1:4000>。改 Markdown 会自动刷新；
   注意改了 `_config.yml` 必须 Ctrl+C 重启。
4. 草稿预览：`bundle exec jekyll serve -l --drafts`。
5. 只想构建不预览：`bundle exec jekyll build`，产物在 `_site/`（已被 git 忽略）。

### 3.3 不想装 Ruby 的另外两条路

- **GitHub Codespaces**：仓库页 → Code → Codespaces → 新建。仓库自带 `.devcontainer/`，
  容器起来后自动 `jekyll serve`，端口 4000 会自动转发，浏览器直接打开。
- **Docker**：装 Docker Desktop 后在本目录 `docker compose up`，同样暴露 4000 端口。

### 3.4 构建失败怎么排查

1. 去仓库 **Actions** → 点开失败的那次 → 展开红色的 step 看报错。
2. 90% 的失败是 **YAML 写错**：`_config.yml` 或文章开头 `---` 之间的缩进/冒号/引号问题。
   报错里常见 `YAML Exception`、`could not find expected ':'`。
3. 本地能跑就用本地跑：`bundle exec jekyll build --verbose`。
4. 纯内容错误（比如日期写错）也会让整个站点构建失败 —— 这就是为什么改完先看 Actions。

---

## 4. 日常改写：四件最常做的事

### 4.1 改首页（个人介绍）

编辑 `_pages/about.md`，文件开头 `---` 之间是元数据（`permalink: /` 表示它就是首页），
下面是普通 Markdown。写完 push 即可。

### 4.2 改侧边栏（头像、姓名、邮箱、链接）

编辑 `_config.yml` 里的 `author:` 段：

```yaml
author:
  avatar   : "profile.png"          # 换成 images/profile.png（建议 400x400 方图）
  name     : "Ke Zhao"
  bio      : "一句话简介"
  location : "Shanghai, China"
  employer : "East China Normal University"
  email    : "kezhao@stu.ecnu.edu.cn"
  github   : "ZhaoKe1024"
  googlescholar : # 填了才显示图标
```

- 头像文件就放在 `images/` 目录，文件名写在这里即可（不要写 `/images/` 前缀）。
- 想加 Google Scholar / ORCID / arXiv / 知乎 / X 等图标，把上面 `author:` 里对应键填上
  （`_config.yml` 里每项都有注释说明）。
- 想换主题配色：`site_theme:` 改为 `air`、`sunrise`、`mint`、`dirt`、`contrast` 之一。

### 4.3 加/改论文（发表情况）

一篇论文 = `_publications/` 下一个 Markdown 文件。**推荐流程：复制一个占位文件改名**。

文件名建议 `YYYY-MM-DD-短标题.md`（日期用于排序，`_publications` 里不是 URL 的一部分）。

```markdown
---
title: "论文标题（会显示在列表里）"
collection: publications
category: conferences          # manuscripts / conferences / preprints，决定出现在哪个小标题下
permalink: /publication/2026-iclr-optbench   # 建议写上，保持 URL 稳定
excerpt: '一句话摘要（列表页显示）'
date: 2026-05-01
venue: 'ICLR 2026'             # 期刊名或会议名
paperurl: 'https://arxiv.org/abs/2605.xxxxx'   # 有 PDF 就放到 files/ 里并写 '/files/xxx.pdf'
slidesurl: 'https://...'       # 可选，会生成 slides 按钮
bibtexurl: '/files/xxx.bib'    # 可选，会生成 cite/bibtex 按钮
citation: 'Ke Zhao, Co-author. (2026). &quot;Paper Title.&quot; <i>Venue</i>.'
---

这里写详情页正文（motivation、方法、实验、BibTeX 都行）。
```

要点：

- `category` 必须在 `_config.yml` 的 `publication_category` 里有定义，
  否则这篇论文不会出现在列表页。
- 加新分类：在 `_config.yml` 的 `publication_category` 下加一个键，例如

  ```yaml
  workshop:
    title: 'Workshop Papers'
  ```

  然后在论文里写 `category: workshop`。
- 想让某篇不显示：把 `collection: publications` 改掉或删文件。
- `citation` 里 HTML 实体 `&quot;` 是模板写法，可以直接用中文引号 `“”` 替代。

### 4.4 写博客

1. 在 `_posts/` 新建 `2026-12-01-my-post.md`（**文件名日期必须是 `YYYY-MM-DD-` 开头**）。
2. 开头元数据：

   ```markdown
   ---
   title: "文章标题"
   date: 2026-12-01 10:00:00 +0800
   permalink: /posts/2026/12/my-post/      # 可选，不写就是 /2026/12/01/my-post/
   tags:
     - llm4opt
     - tool
   categories:
     - notes
   ---
   ```

3. 正文支持：Markdown、表格、代码块、`$$` 数学公式（MathJax）、Mermaid 图、Plotly。
4. 文章会自动出现在：`/year-archive/`（Blog 菜单）、`/tags/xxx/`、`/categories/xxx/`、RSS `/feed.xml`。
5. 草稿：写到 `_drafts/`（文件名不用日期），`jekyll serve --drafts` 本地看，push 不会发布；
   发布时用日期重命名并移进 `_posts/`。
6. 访问提示：博客顶部菜单就叫 **Blog**，指向 `/year-archive/`。

### 4.5 加项目

1. 在 `_portfolio/` 新建 `portfolio-05-xxx.md`：

   ```markdown
   ---
   title: "项目名"
   excerpt: "一句话介绍<br/><a href='https://github.com/...'>GitHub →</a>"
   collection: portfolio
   permalink: /projects/xxx/
   ---

   详情正文……
   ```

2. 新项目用 `portfolio-05-...` 这样递增前缀控制顺序（列表按文件名排序）。
3. 首页 `_pages/about.md` 里的项目表是手写的 Markdown 表格，加了新项目记得同步一行。

### 4.6 改菜单 / 隐藏页面

`_data/navigation.yml`：改顺序就挪行，删掉一条就从菜单消失（页面还在，只是没入口）。
`_pages/talks.html`、`teaching.html` 还在，想恢复只把对应两行取消注释即可。

---

## 5. 常见坑

1. **YAML 里的冒号**：`title: Hello: World` 会报错，要写成 `title: "Hello: World"`。
   值里有 `:`、`#`、`&`、`*`、`{`、`[` 时一律加引号。
2. **缩进必须空格**，不能用 Tab。
3. **文章文件名日期必须与 `date` 一致**，否则排序/URL 会乱。
4. **改了 `_config.yml`** 本地 serve 要重启；线上 push 后自动生效。
5. **站内链接用绝对路径**（`/publications/`、`/images/x.png`），不要用相对路径或带域名。
6. **部署分支**：工作流只监听 `master`；改分支名要同步改 `jekyll.yml`。
7. **仓库名和 `_config.yml` 里的 `url` 必须一致**（`https://ZhaoKe1024.github.io`），
   否则 RSS、sitemap、分享卡片里的链接会指错。
8. **别把大文件放进仓库**：PDF 放 `files/` 没问题，但视频/数据集建议走外部链接。
9. **Windows 换行**：本仓库文件混有 CRLF，`git status` 会显示很多"M"，不影响构建；
   想干净可以用 `.gitattributes` 统一为 LF（`:*.md text eol=lf`），但这会带来一次性大 diff。

---

## 6. 上线前自检清单

- [ ] `_config.yml`：`title` / `name` / `description` / `url` / `repository` / `author.email` / `author.github` 都是自己的
- [ ] `images/profile.png` 换成自己的照片
- [ ] `_publications/` 里的占位论文替换/删除
- [ ] `_pages/cv.md` 的年份、导师、学校补全
- [ ] 首页项目表链接可用
- [ ] Actions 构建绿勾，打开 <https://ZhaoKe1024.github.io> 无明显 404
- [ ] 手机上看一眼（模板自带响应式布局）

---

## 7. 要不要把 README 和主页做成同一个源？

现在是"两份内容"：`ZhaoKe1024/ZhaoKe1024` 仓库的 `README.md`（GitHub 个人主页）和本站首页。
如果不想两边维护，可以：把 README 精简成一句简介 + 链接到本站，本站作为主内容；
或者用脚本把首页正文同步写回 README（去掉 Jekyll front matter 即可）。
