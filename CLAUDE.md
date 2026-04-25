# Claude 初始化文档

## 项目概览
这是一个基于 `Hexo 8.1.1` 的静态博客仓库，主题使用 `hexo-theme-butterfly`。站点主配置在 `_config.yml`，主题相关配置在 `_config.butterfly.yml`。仓库主要工作内容是维护博客文章、页面、主题配置和少量构建脚本。

## 目录约定
- `source/_posts/`：博客文章 Markdown。
- `source/_data/`：站点数据文件，如友链、组件数据、Bangumi 缓存。
- `source/img/`、`source/static/`：静态图片、字体和额外资源。
- `source/about/`、`source/link/`、`source/tags/`、`source/categories/`：独立页面。
- `scaffolds/`：Hexo 新建文章/页面模板。
- `scripts/`：本地辅助脚本，目前主要用于替换配置中的敏感占位符。
- `.github/workflows/`：GitHub Pages、仓库镜像、IndexNow 自动化流程。

## 常用命令
- `npm install`：安装依赖，建议使用与 CI 一致的 Node 18。
- `npm run server`：启动本地开发服务器。
- `npm run preview`：执行 `clean + generate + server`，用于完整预览。
- `npm run build`：生成静态站点到 `public/`，这是最基本的变更校验步骤。
- `node scripts/index.js`：用环境变量 `GITALK_TOKEN` 替换 `_config.butterfly.yml` 中的占位符。

## Claude 工作约束
- 优先修改源文件，不要提交 `public/`、`db.json`、`node_modules/` 等生成产物。
- 修改文章时，保持现有 front matter 风格，常见字段包括 `title`、`date`、`tags`、`categories`、`keywords`、`cover`、`description`。
- 修改 YAML 时使用 2 空格缩进，尽量保持现有字段顺序和注释。
- `scripts/` 目录中的脚本使用 CommonJS 风格，尽量保持简单直接，避免引入额外依赖。
- 这个仓库没有单元测试框架，默认用 `npm run build` 作为最低验证标准；涉及页面或样式时，再配合 `npm run server` 做人工检查。

## 提交与变更风格
提交信息通常较短，内容更新多为简洁中文摘要，依赖升级多为 `Bump ...`。单次提交应尽量只处理一个主题，避免把文章、配置、脚本改动混在一起。

## 部署与安全
推送到 `main` 会触发 GitHub Pages 构建发布。不要把密钥直接写入配置文件，敏感信息应通过环境变量注入，例如 `GITALK_TOKEN`。涉及工作流或部署配置的修改，需要特别谨慎并说明影响范围。
