# Repository Guidelines

## Project Structure & Module Organization
This repository is a Hexo blog. Main site settings live in `_config.yml`, while Butterfly theme overrides live in `_config.butterfly.yml`. Content is under `source/`: posts in `source/_posts/`, shared data in `source/_data/`, static images in `source/img/`, extra assets in `source/static/`, and standalone pages such as `source/about/` and `source/link/`. Reusable post templates are in `scaffolds/`. Small maintenance utilities live in `scripts/`. GitHub Actions for Pages deploy, mirroring, and IndexNow are in `.github/workflows/`.

## Build, Test, and Development Commands
Use Node 18 to match CI.

- `npm install`: install Hexo and theme dependencies.
- `npm run server`: start the local Hexo dev server.
- `npm run preview`: clean, generate, and serve the site for a full local check.
- `npm run build`: clean and generate the static site into `public/`; treat this as the required pre-PR verification step.
- `npm run deploy`: run Hexo deploy if deployment config is set locally.
- `node scripts/index.js`: replace `{GITALK_TOKEN}` in `_config.butterfly.yml` from `GITALK_TOKEN` when needed.

## Coding Style & Naming Conventions
Keep changes small and follow the surrounding style. Use YAML with 2-space indentation in config and data files. JS utilities in `scripts/` are CommonJS; prefer clear function names and minimal side effects. New posts should stay in `source/_posts/` as descriptive Markdown filenames, usually matching the article title, and include front matter like `title`, `date`, `tags`, `categories`, `keywords`, `cover`, and `description`. Do not commit generated output such as `public/`, `db.json`, or `node_modules/`.

## Testing Guidelines
There is no dedicated unit test suite in this repository. Validate changes by running `npm run build`, then inspect the affected pages with `npm run server` or `npm run preview`. For content updates, verify front matter renders correctly, links resolve, and images load from the intended `source/` path.

## Commit & Pull Request Guidelines
Recent history uses short, direct commit subjects: concise Chinese summaries for content/theme updates and `Bump ...` messages for dependency upgrades. Keep commits focused on one concern. PRs should include a brief description, note any config or workflow changes, link the relevant issue if one exists, and attach screenshots for theme, layout, or page-level visual changes. Confirm `npm run build` passes before requesting review.

## Security & Configuration Tips
Do not hardcode secrets in theme config. Use environment variables such as `GITALK_TOKEN` and review any workflow edits carefully, since pushes to `main` trigger GitHub Pages deployment.
