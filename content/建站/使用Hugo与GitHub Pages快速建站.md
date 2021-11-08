## 前言
作为一名程序员，并且是一名有追求的程序员，**技术影响力** 对我们不可或缺。今天介绍一下我是如何利用 Hugo 与 GitHub Pages 快速搭建个人内容输出平台。
## Hugo 与 GitHub Pages 简介
[Hugo](https://github.com/gohugoio/hugo): 世界上最快的网站建设框架，将 Markdown 文件生成 HTML 静态页面。

[GitHub Pages](https://docs.github.com/cn/pages/getting-started-with-github-pages/about-github-pages) GitHub 提供的静态网页托管服务，不需要额外的服务器来部署你的博客系统了。
## 搭建过程
1. [安装 Hugo](https://gohugo.io/getting-started/installing/)

Mac 直接使用以下命令二进制安装
```bash
brew install hugo
```
2. GitHub 创建一个仓库，如 heganghuan-blog，并且 git clone 到本地

3. 使用 Hugo 初始化 Clone 下来的仓库
```bash
hugo new site heganghuan-blog --force
```
生成文件目录应该如下:
```
├── archetypes  // 文章默认模版
│   └── default.md
├── config.toml // 配置文件
├── content     // Markdown 文件
├── data        // 
├── layouts     // html 文件
├── static      // 图片, CSS, JS...
└── themes      // hugo 主题
```
4. [选择一款主题](https://themes.gohugo.io/)

这里为了快速搭建，使用默认主题，后期根据个人需要去修改主题
```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke

echo theme = \"ananke\" >> config.toml
```

5. 发布一篇博客
```bash
hugo new posts/my-first-post.md
```
你也可以之间在 /contents 文件夹下直接新建一个 markdown 文件

6. 本地启动测试
```bash
hugo server -D
```
打开命令行给出的地址，修改 markdown 文件，网页还会实时变化

7. GitHub Pages 设置

Github 仓库首页  -> Settings -> Pages -> main -> docs -> save

会出现一个网站地址: user-name.github.io/repo-name/

意思是打开这个地址就会展现仓库 /docs 下的内容。

接下来我们的工作就是：把 Hugo 生成的静态网页生成文件放在 /docs 文件夹下，再 push 到 GitHub。
8. 修改 config.toml
示例如下，请自行修改
```
baseURL = 'https://he2121.github.io/heganghuan-blog/'   // 设置成你自己的
languageCode = 'zh-cn'
title = '小贺的博客'
theme = "ananke"
publishDir = 'docs'
```

9. 生成静态网页，push 到 GitHub
```
hugo -D
git add .
git commit -m "first commit"
git push
```
打开 GitHub 提供的页面链接 https://he2121.github.io/heganghuan-blog/
测试是否成功

 

