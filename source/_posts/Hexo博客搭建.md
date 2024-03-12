---
title: Hexo博客搭建
tags:
  - Hexo
  - 个人博客
categories: 杂项
cover: /img/cover2.png
abbrlink: b8f4bd70
date: 2023-05-16 10:33:29
---

# Hexo博客搭建

## 什麽是hexo?

對於一種東西的介紹與使用，它的官方文檔一定是最詳細最準確的。與其在網上到處搜別人的博客，不如直接看[hexo官方文檔](https://hexo.io/zh-cn/docs/setup)。在文檔中詳細的介紹了hexo的安裝與博客的挂載，這裏不在詳細介紹。而對於hexo我自身的理解就是，它就是一個靜態網頁生成器，通過markdown文本生產靜態網頁，並配合它自身的服務器模式或者其他靜態頁面挂載網站完成個人博客的搭建。

## 如何安裝hexo?

如上所説，詳細的安裝步驟在他的官方文檔中説的很清楚了。我的理解就是hexo其實就是一個依托npm管理的包，通過：

`npm install -g hexo-cli`

即可全局安裝hexo了。

現在只要選擇一個文件夾作爲你的hexo博客文件夾，就可以在本地運行hexo博客了。執行：

`hexo init <folder>`

現在進入你所選的文件夾，執行：

`npm install`

給你的項目安裝所需依賴。現在你的文件夾目錄結構應該是這樣的：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

至於每個文件夾的具體用途，在[官方文檔](https://hexo.io/zh-cn/docs/setup)也是用詳細説明的。

現在基本工作已經完成，執行：

```
# 生成博客靜態文件
hexo g 
# 運行hexo服務器
hexo s
```

在 `localhost:4000` 現在應該可以看到默認十分簡陋的博客主頁了。這個時候可以選擇一個你覺得好看的主題即可，博主選擇的是 [butterfly](https://butterfly.js.org/)這個主題。

**至於hexo詳細命令，也請參照官方文檔。**

## 如何挂載博客

我采用的方法是github pages的方案進行挂載博客。在github中新建一個倉庫，倉庫名為 `用戶名.girhub.io`，再將本地的hexo項目上傳即可。現在就要用到Github Actions，我的理解就是它就是Github提供的一個自動化服務，當它檢測到分支變化的時候，就會執行我們提供的~~自動化脚本~~。大概流程便是：

1. 本地新建博文，編寫markdown博文文本。
2. 通過git上傳到遠程github倉庫。
3. github actions檢測到分支變動，執行構建脚本，在另外一條分支生成靜態博客文件。
4. github pages監視靜態博客分支，博客内容更新。

從上面的流程可以發現，我們現在本地其實都可以不使用hexo了，只要將markdown文本通過git上傳就行了，同時我們還能在github網頁中直接編輯博文，這樣就脫離了複雜的hexo構建和環境依賴。説了這麽多，卻沒説如何使用Github Actions,其實，只需在倉庫中的`.github/`文件夾中新建`pages.yaml`,並寫入一下内容即可

```
.github/workflows/pages.yml
name: Pages

on:
  push:
    branches:
      - main # default branch

jobs:
  pages:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js 16.x
        uses: actions/setup-node@v2
        with:
          node-version: "16"
      - name: Cache NPM dependencies
        uses: actions/cache@v2
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

現在只需在GitHub储存库中前往 `Settings > Pages > Source`，并将 branch 改为 `gh-pages`即可。

現在訪問你的 `https`://<你的 GitHub 用户名>.github.io 即可查看你的博客了。

## CDN速度提升

github pages最大的問題就是在國内訪問速度實在太慢了，其實這個問題可以通過接入CDN解決，博主便是通過接入七牛的CND提升速度的，七牛的話每個月有十個G的免費流量，對於個人博客綽綽有餘了。接入七牛也不算複雜，最主要的就是還需要一個有備案的域名，有想法的可以自行研究一下，這裏也不再詳細説明。

效果圖：

![](https://cdn.jsdelivr.net/gh/Echo-xzp/Resource/img/blog-speed.png)



