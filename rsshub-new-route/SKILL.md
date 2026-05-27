---
name: rsshub-new-route
description: |
  Complete guide for creating a new RSSHub RSS route from scratch and submitting a clean PR to DIYgod/RSSHub.
  Covers: website analysis, WordPress REST API detection, namespace + route file creation,
  local testing, linting, PR template (exact format required), and CI troubleshooting.
  Includes all common pitfalls and their fixes encountered in real submissions.
  Use when: user wants to add a new RSS source to RSSHub, create a route, submit a PR to RSSHub,
  or mentions RSSHub + new route / new RSS source / 新路由 / 添加RSS源 / 制作RSS.
author: panqingjie00
version: 1.0.0
date: 2026-05-27
user-invocable: true
---

# RSSHub 新路由创建完整指南

## 前置条件

- 已 fork [RSSHub](https://github.com/DIYgod/RSSHub) 并部署到 Vercel
- 本地已安装 pnpm：`npm install -g pnpm`
- 已克隆 fork 仓库到本地，并执行 `pnpm install`

---

## 第一步：分析目标网站

1. 访问目标网站，判断技术栈：
   - **WordPress**：检查 `/wp-json/wp/v2/posts` API
   - **静态页面**：需用 cheerio 解析 HTML
   - **SPA / 反爬**：可能需要 Puppeteer
2. 确认数据字段：标题、内容、日期、链接等
3. 如果有分页，确认分页参数

---

## 第二步：创建路由文件

### 2.1 命名空间文件 `lib/routes/<namespace>/namespace.ts`

```ts
import type { Namespace } from '@/types';

export const namespace: Namespace = {
    name: '站点中文名',
    url: 'example.com',
    lang: 'zh-CN',
};
```

### 2.2 路由文件（如 `lib/routes/<namespace>/latest.ts`）

```ts
import type { Route } from '@/types';
import cache from '@/utils/cache';
import got from '@/utils/got';
import { parseDate } from '@/utils/parse-date';

export const route: Route = {
    path: '/latest',              // ⚠️ 相对路径，namespace 自动拼接
    categories: ['new-media'],
    example: '/example/latest',   // ✅ 完整路径，含 namespace
    url: 'example.com',
    name: '最新文章',
    maintainers: ['你的GitHub用户名'],
    features: {
        requireConfig: false,
        requirePuppeteer: false,
        antiCrawler: false,
        supportBT: false,
        supportPodcast: false,
        supportScihub: false,
    },
    radar: [{
        source: ['example.com'],
        target: '/latest',
    }],
    handler,
};

async function handler() {
    const response = await cache.tryGet('example:latest', async () => {
        const { data } = await got('https://example.com/wp-json/wp/v2/posts', {
            searchParams: { per_page: 20 },
        });
        return data;
    });

    const items = response.map((item) => ({
        title: item.title.rendered,
        description: item.content.rendered,
        pubDate: parseDate(item.date_gmt),
        link: item.link,
    }));

    return {
        title: '站点名 - 最新文章',
        link: 'https://example.com',
        item: items,
    };
}
```

### ⚠️ 关键易错点

| 错误 ❌ | 正确 ✅ | 说明 |
|----------|----------|------|
| `ctx.cache.tryGet(...)` | `import cache from '@/utils/cache'` → `cache.tryGet(...)` | Hono Context 没有 `cache` 属性，会导致 500 |
| `async function handler(ctx)` 不用 ctx | `async function handler()` | 省略未用参数 |
| `async function handler(_ctx)` | `async function handler()` | 下划线前缀被 oxlint 禁止 |
| `throw new Error(...)` 类型校验后 | `throw new TypeError(...)` | oxlint unicorn/prefer-type-error 规则 |
| `path: '/namespace/latest'` | `path: '/latest'` | path 是相对于 namespace 的 |
| `example: '/latest'` | `example: '/namespace/latest'` | example 需要完整路径 |

---

## 第三步：本地测试

```bash
pnpm dev
curl http://localhost:1200/<namespace>/latest
```

确认返回完整 RSS XML，包含 title、link、item 等字段。

---

## 第四步：Lint 检查

```bash
npx oxlint --type-aware lib/routes/<namespace>/latest.ts
```

常用 oxlint 规则违反：
- `eslint(no-unused-vars)` — 未用参数
- `unicorn(prefer-type-error)` — 类型校验后应用 TypeError
- `eslint-js(no-restricted-syntax)` — `_ctx` 等被禁语法

---

## 第五步：提交 PR

### 分支操作

```bash
git checkout -b feat/<namespace>-latest
git add lib/routes/<namespace>/
git commit -m "add: example.com RSS 路由"
git push origin feat/<namespace>-latest
```

### PR 正文 — 严格模板

**四个 `## ` 标题缺一不可**，`routes` 代码块必须带 `routes` 语言标识：

````markdown
## Involved Issue / 该 PR 相关 Issue

Close #

## Example for the Proposed Route(s) / 路由地址示例

```routes
/example/latest
```

## New RSS Route Checklist / 新 RSS 路由检查表

- [x] New Route / 新的路由
  - [x] Follows [Script Standard](https://docs.rsshub.app/joinus/advanced/script-standard)
- [x] Anti-bot or rate limit / 反爬/频率限制
  - [x] If yes, do your code reflect this sign?
- [x] [Date and time](https://docs.rsshub.app/joinus/advanced/pub-date)
  - [x] Parsed / 可以解析
  - [x] Correct time zone / 时区正确
- [ ] New package added / 添加了新的包
- [ ] `Puppeteer`

## Note / 说明

简短说明数据来源和路由功能。
````

### 提 PR 链接

```
https://github.com/DIYgod/RSSHub/compare/master...<你的用户名>:<分支名>?quick_pull=1
```

- **base repository**：`DIYgod/RSSHub`
- **base branch**：`master`
- ⚠️ 不要点错创建到自己的 fork

---

## 第六步：CI 问题排查

| CI 检查 | 失败现象 | 原因 | 解决 |
|---------|----------|------|------|
| Fetch affected routes | `auto: route no found`、PR 被关闭 | PR 正文格式不对 | 检查 4 个 `## ` 标题 + routes 代码块语言标识 |
| Route test | `auto: not ready to review` | RSS 输出异常 / 运行时错误 | 本地复现，检查 handler 返回值 |
| Docker build | cancelled 或 failure | 代码编译错误 | 检查 TypeScript 语法 |
| oxlint | 2 new alerts | 代码规范 | `npx oxlint --type-aware` 本地检查 |
| Vercel | Authorization required | **fork PR 正常现象** | ⚠️ 不用管，不影响合并 |

---

## 完整文件结构

```
lib/routes/<namespace>/
├── namespace.ts          # 站点名称、URL、语言
└── latest.ts             # 路由处理器（或按功能命名）
```
