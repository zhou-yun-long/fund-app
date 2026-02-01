# 部署到 GitHub Pages

## 前置条件

- GitHub 账号
- 项目已推送到 GitHub 仓库

## 方式一：GitHub Actions 自动部署（推荐）

### 1. 修改 vite.config.ts

添加 `base` 配置（仓库名）：

```ts
export default defineConfig({
  base: '/fund-app/',  // 替换为你的仓库名
  // ... 其他配置
})
```

### 2. 创建 GitHub Actions 工作流

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 3. 启用 GitHub Pages

1. 进入仓库 Settings → Pages
2. Source 选择 "GitHub Actions"
3. 推送代码后自动部署

访问地址：`https://你的用户名.github.io/fund-app/`

## 方式二：手动部署到 gh-pages 分支

### 1. 安装 gh-pages

```bash
npm install -D gh-pages
```

### 2. 添加部署脚本

在 `package.json` 中添加：

```json
{
  "scripts": {
    "deploy": "npm run build && gh-pages -d dist"
  }
}
```

### 3. 修改 vite.config.ts

```ts
export default defineConfig({
  base: '/fund-app/',  // 替换为你的仓库名
})
```

### 4. 执行部署

```bash
npm run deploy
```

### 5. 配置 GitHub Pages

1. 进入仓库 Settings → Pages
2. Source 选择 "Deploy from a branch"
3. Branch 选择 `gh-pages`，目录选择 `/ (root)`

## 处理 SPA 路由

GitHub Pages 不支持 SPA 路由，刷新会 404。解决方案：

### 方案一：使用 Hash 路由

修改 `src/router.ts`：

```ts
import { createRouter, createWebHashHistory } from 'vue-router'

const router = createRouter({
  history: createWebHashHistory(import.meta.env.BASE_URL),
  routes
})
```

### 方案二：添加 404.html

在 `public` 目录创建 `404.html`：

```html
<!DOCTYPE html>
<html>
<head>
  <script>
    const path = window.location.pathname;
    const repo = '/fund-app';
    if (path.startsWith(repo)) {
      const route = path.slice(repo.length);
      sessionStorage.setItem('redirect', route);
      window.location.replace(repo + '/');
    }
  </script>
</head>
</html>
```

在 `index.html` 的 `<body>` 开头添加：

```html
<script>
  const redirect = sessionStorage.getItem('redirect');
  if (redirect) {
    sessionStorage.removeItem('redirect');
    window.history.replaceState(null, '', redirect);
  }
</script>
```

## 自定义域名（可选）

1. 在 `public` 目录创建 `CNAME` 文件，写入域名：
   ```
   fund.example.com
   ```

2. 在域名 DNS 添加 CNAME 记录指向 `你的用户名.github.io`

3. 修改 `vite.config.ts`：
   ```ts
   base: '/'  // 使用自定义域名时改为 /
   ```

## 常见问题

### 资源加载 404

确保 `base` 配置正确，与仓库名一致。

### 页面空白

检查浏览器控制台，通常是路径问题。

### API 请求失败

GitHub Pages 是 HTTPS，确保 API 也使用 HTTPS。天天基金 JSONP 接口支持 HTTPS。
