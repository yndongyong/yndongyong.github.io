# yndongyong's Blog

个人技术博客源码仓库，基于 [Hexo](https://hexo.io/) 静态博客框架与 [NexT](https://theme-next.org/) 主题搭建。

- **线上地址**：[https://yndongyong.github.io](https://yndongyong.github.io)
- **分支规范**：
  - `hexo_source`：博客源文件分支（Markdown 文章、主题配置、站点配置）
  - `master`：静态站点分支（GitHub Pages 自动展示的编译后 HTML/CSS/JS 文件）

---

## 常用命令速查表

| 操作 | 完整命令 | 简写命令 / 快捷命令 | 说明 |
|---|---|---|---|
| **创建文章** | `hexo new post "标题"` | `hexo n "标题"` | 在 `source/_posts/` 下生成新的 Markdown 文件 |
| **本地预览** | `hexo server` | `hexo s` | 启动本地服务，访问 `http://localhost:4000` |
| **清理缓存** | `hexo clean` | - | 清除 `public/` 目录与 `db.json` 编译缓存 |
| **生成静态文件** | `hexo generate` | `hexo g` | 渲染 Markdown 并生成静态网页到 `public/` |
| **一键部署** | `hexo deploy` | `hexo d` | 将 `public/` 静态网页发布到 GitHub `master` 分支 |
| **生成并部署** | `hexo generate -d` | `hexo g -d` | 编译静态页面并立即部署到线上 |
| **全自动发布** | - | `npm run run` | 自动执行清理、构建发布，并同步推送源码至 `hexo_source` 分支 |

> **提示**：如果未全局安装 `hexo-cli`，命令前可添加 `npx`（例如 `npx hexo s`）。

---

## 详细操作指南

### 1. 创建新文章

```bash
hexo new post "你的文章标题"
# 简写
hexo n "你的文章标题"
```

执行后将在 `source/_posts/` 目录下生成同名 `.md` 文件，文件头部包含默认 Front-matter 元数据：

```yaml
---
title: 你的文章标题
date: 2026-09-27 15:00:00
categories:
  - - 分类名
tags:
  - - 标签名
---
```

---

### 2. 本地实时预览

修改或编写文章时，在本地启动预览服务：

```bash
hexo server
# 简写
hexo s
```

- 默认访问地址：`http://localhost:4000`
- 支持热更新（保存 Markdown 文件后刷新浏览器即可看到效果）。
- **如果端口 4000 被占用**，可使用 `-p` 参数指定其他端口：
  ```bash
  hexo s -p 5000
  ```
- **如果修改没有生效**，建议先清理缓存后再启动：
  ```bash
  hexo clean && hexo s
  ```

---

### 3. 生成静态文件

将 Markdown 渲染为可在浏览器独立运行的 HTML 页面：

```bash
hexo generate
# 简写
hexo g
```

生成的文件存放在 `public/` 目录中。

---

### 4. 发布上线与源码同步

日常发布文章包含两个步骤：**发布静态网站** 与 **保存博客源码**。

#### 方式 A：一键自动化发布（推荐）

直接在项目根目录下执行 `package.json` 中配置好的命令：

```bash
npm run run
```

该命令会自动顺序执行：
1. `hexo clean`（清空旧缓存）
2. `hexo g -d`（重新编译并推送到线上 GitHub Pages）
3. `git add . && git commit -m '同步新文章'`（将 Markdown 等源码纳入 Git）
4. `git push origin hexo_source`（推送源码到 GitHub 备份）

---

#### 方式 B：手动分布执行

1. **部署静态页面到线上**：
   ```bash
   hexo clean
   hexo g -d
   ```
2. **提交并推送源码**：
   ```bash
   git add .
   git commit -m "feat: 新增文章 xxx"
   git push origin hexo_source
   ```

---

## 常见问题与技巧

1. **修改主题或配置后页面未更新**：
   先执行 `hexo clean` 清空缓存，再重新执行 `hexo s` 或 `hexo g`。
2. **草稿（Draft）功能**：
   - 创建草稿：`hexo new draft "草稿标题"`（保存在 `source/_drafts/` 目录）
   - 本地预览包含草稿：`hexo s --drafts`
   - 发布草稿转为正式文章：`hexo publish "草稿标题"`
