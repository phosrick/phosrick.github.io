# Phosrick's Blog

基于 [Astro Pure](https://github.com/cworld1/astro-theme-pure) 主题构建的个人博客，使用 Astro + UnoCSS 打造，静态部署在 GitHub Pages。

## 简介

这是我的个人博客，用于记录技术笔记、思考与日常。线上地址：[https://phosrick.github.io](https://phosrick.github.io)。

### 内容结构

- **Blog**：文章列表与正文
- **About**：关于我

## 本地开发

环境要求：

- [Node.js](https://nodejs.org/)：18.0.0+（或 [Bun](https://bun.sh/)）

常用命令（使用 Bun）：

```shell
# 安装依赖
bun install
# 启动开发服务器
bun dev
# 构建项目
bun run build
# 本地预览（构建后）
bun preview
# 新建一篇文章
bun pure new
```

如果你使用 npm / pnpm，可将上面的 `bun` 替换为对应包管理器的命令，例如 `npm install`、`npm run dev`。

## 部署

本项目输出为静态站点（`output: 'static'`），已在 `astro.config.ts` 中配置 `site: 'https://phosrick.github.io'`，可直接通过 GitHub Pages 部署，构建产物位于 `dist/` 目录。

如需部署到 Vercel、Netlify 等其他平台，可在 `astro.config.ts` 中启用对应的 Adapter。详见 [Astro Pure 部署文档](https://astro-pure.js.org/docs/setup/deployment)。

## 配置

站点的标题、作者、菜单、社交链接、评论、搜索等选项都集中在 `src/site.config.ts` 中，按需修改即可。

## 许可证

本项目基于 Apache 2.0 License 开源；所使用的 Astro Pure 主题同样遵循 Apache 2.0 License。
