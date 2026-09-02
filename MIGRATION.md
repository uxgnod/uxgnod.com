# uxgnod.com 迁移指南：静态 HTML → Astro

> 旧项目（纯静态 HTML）已归档：本地 `~/Developer/uxgnod.com_bak`，GitHub `uxgnod/uxgnod.com_bak`。
> 新项目（本仓库）= Astro 7 minimal 模板。目标：视觉不变，URL 不变，博客跑在 `/blog/`。

---

## 0. 全局认知：迁移后各文件去哪了

Astro 的核心目录约定（先记住这个，后面每一步都是在往里放东西）：

```
uxgnod.com/
├── astro.config.mjs        ← Astro 配置（域名、集成插件）
├── wrangler.jsonc          ← Cloudflare 部署配置（新建）
├── package.json            ← 脚本：dev / build / deploy
├── public/                 ← 「原样拷贝」区：文件原封不动进 dist 根目录
│   ├── favicon.svg
│   ├── plan.txt
│   └── _headers
└── src/
    ├── pages/              ← 一个 .astro 文件 = 一个 URL
    │   ├── index.astro     ← →  /
    │   ├── about.astro     ← →  /about/
    │   ├── 404.astro       ← →  /404.html（错误页）
    │   └── blog/           ← 之后博客的页面
    ├── content/blog/       ← 博客的 Markdown 文章
    └── content.config.ts   ← 文章的字段定义（之后博客部分）
```

文件对照表：

| 旧文件（`_bak`） | 新位置 | 说明 |
|---|---|---|
| `index.html` | `src/pages/index.astro` | 首页，几乎原样搬运（见步骤 3） |
| `about/index.html` | `src/pages/about.astro` | URL 仍是 `/about/` |
| `writing/index.html` | `src/pages/writing.astro`（过渡） | 博客上线后删除，链接改指 `/blog/` |
| `404.html` | `src/pages/404.astro` | 构建产物恰好是 `dist/404.html` |
| `plan.txt` | `public/plan.txt` | 原样拷贝 |
| `_headers` | `public/_headers` | 原样拷贝 |
| `favicon.svg` | `public/favicon.svg` | 覆盖模板自带的 |
| `design/` `serve.py` | 不迁 | 设计稿留 `_bak` 归档；`astro dev` 取代 serve.py |
| `wrangler.jsonc` | 新建，内容基本一致 | 见步骤 2 |

**`.astro` 文件是什么**：就是 HTML，顶部可以有一段 `---` 包围的 JS（frontmatter，构建时运行，用来取数据）。没有 frontmatter 时，它和普通 HTML 几乎没区别 —— 所以我们的页面能「原样搬」。

---

## 1. 基础配置

### 1.1 astro.config.mjs — 告诉 Astro 自己的域名

`site` 是给 RSS / sitemap 生成绝对链接用的，现在就配上，一劳永逸：

```js
// @ts-check
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://uxgnod.com',
});
```

> 说明：Astro 默认 `build.format: 'directory'`，即 `/about` 构建成 `dist/about/index.html`。
> 这和 Cloudflare 的 `auto-trailing-slash` 规则正好匹配，不用改。

### 1.2 安装 wrangler，加一条 deploy 脚本

```bash
npm install -D wrangler
```

然后编辑 `package.json`，在 `"scripts"` 里加一行：

```json
"scripts": {
  "dev": "astro dev",
  "build": "astro build",
  "preview": "astro preview",
  "astro": "astro",
  "deploy": "astro build && wrangler deploy"
},
```

> `deploy` = 先构建出 `dist/`，再让 wrangler 把 `dist/` 推上 Cloudflare。以后发布就一条命令。

### 1.3 新建 wrangler.jsonc

在项目根目录新建 `wrangler.jsonc`（和旧项目几乎一样）：

```jsonc
{
  "name": "uxgnod",
  "compatibility_date": "2026-09-01",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "404-page",
    "html_handling": "auto-trailing-slash"
  }
}
```

逐行解释：

- `"name": "uxgnod"` —— **Worker 名字必须和旧的一致**。同名部署 = 直接覆盖旧 Worker，之前绑定的自定义域名 `uxgnod.com` 自动继承，不用碰 DNS。
- `"directory": "./dist"` —— 指向 Astro 的构建输出（这正是旧项目里指向了不存在目录的那个配置，迁完它就正确了）。
- `"not_found_handling": "404-page"` —— 404 时返回 `dist/404.html`（由步骤 5 的 `404.astro` 生成）。
- `"html_handling": "auto-trailing-slash"` —— `/about` 自动补成 `/about/` 并命中 `about/index.html`。

### 1.4 tsconfig / .gitignore

模板已配好（`strict` 模式，`dist/` 已忽略），**不用动**。

---

## 2. 迁移静态资源 → `public/`

`public/` 里的文件会被**原封不动**拷进 `dist/` 根目录。把旧项目的这几个文件放进来：

```bash
# 在新项目根目录执行（旧项目路径按你的实际位置）
cp ../uxgnod.com_bak/favicon.svg  public/favicon.svg   # 覆盖模板默认图标
cp ../uxgnod.com_bak/plan.txt     public/plan.txt
cp ../uxgnod.com_bak/_headers     public/_headers
rm public/favicon.ico                          # 模板自带的 ico，用不上了
```

> `_headers` 是 Cloudflare 的响应头规则文件，必须出现在 **部署目录（dist）的根**才能生效。
> 放在 `public/` 里，构建时就会被拷到 `dist/_headers`，规则照旧生效。

---

## 3. 首页：`index.html` → `src/pages/index.astro`

把模板的占位 `src/pages/index.astro` 整个替换为下面的内容 —— 就是你现在的首页，只有两处小改动（都有注释标出）：

```astro
---
// frontmatter：目前是空的。以后需要取数据（比如最新文章）时写在这里，构建时执行。
---

<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="dark">
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
<title>董旭 · Dong Xu</title>
<meta name="description" content="董旭的个人主页。">
<style is:global>
  /* ⚠️ 唯一的语法改动 1：Astro 默认会给 <style> 加作用域（只对本组件生效）。
     我们的样式是全局的（body、::selection 等），所以加 is:global 保持原样。 */
  :root{
    --bg:#0f0e0c; --ink:#d8cdb6; --dim:#8f8570;
    --amber:#ffb454; --line:#3a352b;
  }
  *{box-sizing:border-box}
  body{
    margin:0; min-height:100svh; display:flex; align-items:center;
    background:var(--bg); color:var(--ink);
    font:16px/1.9 "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Noto Sans SC", sans-serif;
  }
  /* 一点点好看：左上角一抹极淡的暖色辉光 */
  body::before{
    content:""; position:fixed; inset:0; pointer-events:none;
    background:radial-gradient(42rem 30rem at 24% 18%, rgba(255,180,84,.06), transparent 70%);
  }
  main{
    width:100%; max-width:40rem; margin:0 auto; padding:2rem;
    animation:fade .7s ease-out both;
  }
  @keyframes fade{from{opacity:0; transform:translateY(6px)} to{opacity:1; transform:none}}
  @media (prefers-reduced-motion:reduce){ main{animation:none} }

  .name{margin:0; font-size:2rem; font-weight:normal; letter-spacing:.04em}
  .meta{margin:.4rem 0 2.4rem; color:var(--dim); font-size:.9rem}
  nav{display:flex; flex-wrap:wrap; gap:.4rem 1.8rem; font-size:.9rem}
  a{
    color:var(--dim); text-decoration:none;
    border-bottom:1px dotted var(--line); padding-bottom:1px;
    transition:color .15s, border-color .15s;
  }
  a:hover{color:var(--amber); border-color:var(--amber)}
  ::selection{background:var(--amber); color:var(--bg)}
  .cursor{
    display:inline-block; width:.55em; height:1em; margin-left:.3em;
    background:var(--amber); vertical-align:-.08em;
    animation:blink 1.2s steps(1) infinite;
  }
  @keyframes blink{50%{opacity:0}}
  .sr{position:absolute; width:1px; height:1px; overflow:hidden; clip:rect(0 0 0 0)}
</style>
</head>
<body>
<main>

  <h1 class="sr">董旭 — Dong Xu</h1>
  <p class="name">董旭<span class="cursor"></span></p>
  <p class="meta">Dong Xu · Chengdu, China</p>

  <nav>
    <a href="mailto:hey@uxgnod.com">hey@uxgnod.com</a>
    <a href="https://github.com/uxgnod">github</a>
    <!-- 改动 2：blog 链接。博客（步骤 7）上线后指向 /blog/，在那之前可先留 /writing/ -->
    <a href="/writing/">blog</a>
  </nav>

</main>
<!--
█   █ █   █  ████ █   █  ███  ████
█   █  █ █  █     ██  █ █   █ █   █
█   █   █   █  ██ █ █ █ █   █ █   █
█   █  █ █  █   █ █  ██ █   █ █   █
█████ █   █  ████ █   █  ███  ████
uxgnod = dongxu, spelled backwards.
-->
</body>
</html>
```

---

## 4. about 页：`about/index.html` → `src/pages/about.astro`

方法同上：新建 `src/pages/about.astro`，把 `_bak/about/index.html` 的 `<html>` 整块拷进去，`<style>` 改成 `<style is:global>`。构建后 URL 不变，还是 `/about/`。

> 文件名和 URL 的关系：`src/pages/about.astro` → `/about`（配合 auto-trailing-slash 即 `/about/`）。
> 旧的目录式 `about/index.html` 不需要了 —— Astro 替你拼路径。

---

## 5. 404 页：`404.html` → `src/pages/404.astro`

同样方法搬到 `src/pages/404.astro`。Astro 会把它构建成 `dist/404.html`，正好被 `wrangler.jsonc` 的 `"not_found_handling": "404-page"` 接住。

---

## 6. writing 页（过渡方案）

博客还没写好之前，把 `writing/index.html` 也照步骤 4 搬成 `src/pages/writing.astro`，保证首页那个 blog 链接不死。
等步骤 7 的博客上线后：删掉 `writing.astro`，把首页链接改成 `/blog/`。

---

## 7. 加上博客（Content Collections）

Astro 的博客 = **Markdown 文件夹 + 一个字段定义 + 两个页面**。全部照抄即可。

### 7.1 定义文章的字段 —— `src/content.config.ts`

新建文件（路径必须是 `src/content.config.ts`，Astro 约定）：

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  // 扫描 src/content/blog/ 下的 .md 文件
  loader: glob({ pattern: '**/*.md', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    description: z.string().optional(),
    date: z.coerce.date(),          // "2026-09-01" 这种字符串自动转 Date
    draft: z.boolean().default(false), // true 的文章不会出现在列表和 RSS
  }),
});

export const collections = { blog };
```

> 好处：写文章时 frontmatter 写错（漏 title、日期格式不对），**构建直接报错**，而不是上线了才发现。

### 7.2 写第一篇文章 —— `src/content/blog/hello-world.md`

```markdown
---
title: First Light
description: 本站的第一次观测。
date: 2026-09-01
---

正文用 Markdown 随便写。
```

### 7.3 博客列表页 —— `src/pages/blog/index.astro`

新建 `src/pages/blog/` 目录，里面放 `index.astro`（样式沿用全站变量，无需新 CSS）：

```astro
---
import { getCollection } from 'astro:content';

// 取所有非草稿文章，按日期倒序
const posts = (await getCollection('blog', ({ data }) => !data.draft))
  .sort((a, b) => b.data.date.valueOf() - a.data.date.valueOf());

const fmt = (d: Date) => d.toISOString().slice(0, 10);
---

<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="dark">
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
<title>blog — 董旭</title>
<style is:global>
  :root{
    --bg:#0f0e0c; --ink:#d8cdb6; --dim:#8f8570;
    --amber:#ffb454; --line:#3a352b;
  }
  *{box-sizing:border-box}
  body{
    margin:0; background:var(--bg); color:var(--ink);
    font:16px/1.9 "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Noto Sans SC", sans-serif;
  }
  main{width:100%; max-width:40rem; margin:0 auto; padding:4rem 2rem}
  h1{font-size:1.4rem; font-weight:normal; letter-spacing:.04em; margin:0 0 2rem}
  ul{list-style:none; margin:0; padding:0}
  li{margin:0 0 1.1rem}
  time{color:var(--dim); font-size:.85rem; margin-right:1rem}
  a{color:var(--ink); text-decoration:none; border-bottom:1px dotted var(--line); padding-bottom:1px}
  a:hover{color:var(--amber); border-color:var(--amber)}
  .back{display:inline-block; margin-top:3rem; font-size:.9rem}
</style>
</head>
<body>
<main>
  <h1>blog</h1>
  <ul>
    {posts.map((post) => (
      <li>
        <time>{fmt(post.data.date)}</time>
        <a href={`/blog/${post.id}/`}>{post.data.title}</a>
      </li>
    ))}
  </ul>
  <a class="back" href="/">← 回到首页</a>
</main>
</body>
</html>
```

### 7.4 文章详情页 —— `src/pages/blog/[...slug].astro`

方括号文件名是 Astro 的**动态路由**：一 个文件生成 N 个页面，每篇文章一个。

```astro
---
import { getCollection, render } from 'astro:content';

// 构建时：为每篇非草稿文章生成一个路径
export async function getStaticPaths() {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  return posts.map((post) => ({ params: { slug: post.id }, props: { post } }));
}

const { post } = Astro.props;
const { Content } = await render(post);   // 把 Markdown 渲染成组件
const fmt = (d: Date) => d.toISOString().slice(0, 10);
---

<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="dark">
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
<title>{post.data.title} — 董旭</title>
<style is:global>
  :root{
    --bg:#0f0e0c; --ink:#d8cdb6; --dim:#8f8570;
    --amber:#ffb454; --line:#3a352b;
  }
  *{box-sizing:border-box}
  body{
    margin:0; background:var(--bg); color:var(--ink);
    font:16px/1.9 "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Noto Sans SC", sans-serif;
  }
  main{width:100%; max-width:40rem; margin:0 auto; padding:4rem 2rem}
  h1{font-size:1.5rem; font-weight:normal; letter-spacing:.02em; margin:0 0 .4rem}
  time{color:var(--dim); font-size:.85rem}
  article{margin-top:2.4rem}
  article a{color:var(--amber); text-decoration:underline dotted; text-underline-offset:3px}
  article a:hover{color:#ffd08a}
  article code{font-size:.9em; background:#1a1712; padding:.1em .35em; border-radius:3px}
  article pre{background:#14120f; border:1px solid var(--line); padding:1rem 1.2rem; overflow-x:auto}
  article pre code{background:none; padding:0}
  .back{display:inline-block; margin-top:3rem; font-size:.9rem;
        color:var(--dim); text-decoration:none}
  .back:hover{color:var(--amber)}
</style>
</head>
<body>
<main>
  <h1>{post.data.title}</h1>
  <time>{fmt(post.data.date)}</time>
  <article><Content /></article>
  <a class="back" href="/blog/">← 回到 blog</a>
</main>
</body>
</html>
```

> `post.id` 就是文件名：`hello-world.md` → `/blog/hello-world/`。想改 URL 就改文件名。

### 7.5 收尾

- 首页 `index.astro` 里的 blog 链接改成 `<a href="/blog/">blog</a>`，删除 `src/pages/writing.astro`。

### 7.6 RSS 和 sitemap（可选但建议）

```bash
npx astro add rss       # 安装 @astrojs/rss
npx astro add sitemap   # 安装 @astrojs/sitemap 并自动写进 astro.config.mjs
```

新建 `src/pages/rss.xml.js`：

```js
import rss from '@astrojs/rss';
import { getCollection } from 'astro:content';

export async function GET(context) {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  return rss({
    title: '董旭 · Dong Xu',
    description: '董旭的博客',
    site: context.site,   // 读 astro.config.mjs 里的 site
    items: posts.map((post) => ({
      title: post.data.title,
      description: post.data.description,
      pubDate: post.data.date,
      link: `/blog/${post.id}/`,
    })),
  });
}
```

sitemap 集成装好后自动生效（这就是步骤 1.1 配 `site` 的回报），产物是 `dist/sitemap-index.xml`。

---

## 8. 本地验证

```bash
npm run dev        # 开发服务器 http://localhost:4321，改文件即时热更
npm run build      # 正式构建 → dist/，有错误会在这里暴露
npx wrangler dev   # 用 Cloudflare 的运行时本地预览 dist/（最接近线上，验证 404/_headers）
```

上线前 checklist：

- [ ] `/` 首页视觉和旧站一致（字体、光标动画、辉光）
- [ ] `/about/` 正常
- [ ] `/plan.txt` 是纯文本且中文不乱码（`_headers` 生效）
- [ ] 随便访问一个不存在的路径 → 显示 404 页
- [ ] （博客上线后）`/blog/` 列表、文章页、`/rss.xml` 都正常

---

## 9. 部署

```bash
npm run deploy
```

第一次部署时会弹浏览器让你登录 Cloudflare 授权，之后一路回车。

**关键点（务必看一眼）：**

1. **同名覆盖**：Worker 名是 `uxgnod`，和旧的一样 → 新站直接顶替旧站，自定义域名、DNS 什么都不用动。
2. **检查 Cloudflare 是否挂了 Git 自动构建**：如果旧的 Worker 在 Cloudflare 后台配的是「连接 GitHub 仓库自动部署」（Workers Builds），它认的是仓库 ID 而不是名字 —— 改名后它依然连着 `uxgnod.com_bak`。以后你往 `_bak` push，会把**旧站重新部署上去**盖掉新站。去 Cloudflare 后台 → Workers → `uxgnod` → Settings → Build，把它断开或改指向新的 `uxgnod/uxgnod.com` 仓库。如果你一直是手动 `wrangler deploy`，忽略这条。
3. 部署完跑一遍步骤 8 的 checklist。

---

## 10. 日常流程（迁移完成后）

```bash
# 写博客
$EDITOR src/content/blog/新文章.md
npm run dev                    # 本地看效果
git add -A && git commit -m "post: 新文章"
npm run deploy                 # 上线

# 改首页/其他页面同理：改 .astro → dev 预览 → commit → deploy
```

Git 提交和部署是两回事：`push` 只进 GitHub（备份），`npm run deploy` 才更新线上。以后想自动化，可以加一个 GitHub Actions 在 push 到 main 时自动 `deploy`（需要时再说）。

---

## 附：常见问题

| 症状 | 原因 / 解法 |
|---|---|
| 页面中文乱码 | 确认 `<meta charset="utf-8">` 还在；`plan.txt` 乱码是 `_headers` 没生效，确认它在 `public/` 且重新 build 过 |
| 访问 `/about` 404 | 确认 wrangler.jsonc 里有 `"html_handling": "auto-trailing-slash"`，或直接访问 `/about/` |
| 改了 Markdown 列表没变 | Content Collections 的类型信息缓存在 `.astro/`，重启 `npm run dev` 即可 |
| frontmatter 报错构建失败 | 好事：字段写错了，按 `content.config.ts` 里的 schema 改 |
| 部署后还是旧站 | 十有八九是第 9 节第 2 条：Cloudflare 的 Git 集成还在部署 `_bak` |
