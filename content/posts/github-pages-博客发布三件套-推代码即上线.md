---
title: GitHub Pages 博客发布三件套：推代码即上线
date: 2026-08-18
draft: false
---

发布 GitHub Pages 博客，两大痛点：草稿误发、构建产物污染源码。三个小技巧一次解决。

## 一、用 Actions 自动化发布

GitHub 官方推荐非 Jekyll 场景用 GitHub Actions 发布，三件套 configure-pages、upload-pages-artifact、deploy-pages 即可；注意 GITHUB_TOKEN 推送不触发重建。

## 二、main 分支当唯一发布源

源码只存 .md 与 Hugo 配置，构建产物交给 Actions，不必维护 gh-pages 静态分支；若走分支模式，/docs 目录是源码与站点共存的折中。

## 三、front matter 控制发布状态

Hugo 里 draft: true 默认不渲染，本地用 hugo server -D 预览，线上永不误发；publishDate、expiryDate 还能定时上下线。

## 小结

推代码即上线，草稿永不误发。

## 参考链接

1. [https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
2. [https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
3. [https://github.com/actions/deploy-pages](https://github.com/actions/deploy-pages)
4. [https://gohugo.io/content-management/front-matter/](https://gohugo.io/content-management/front-matter/)
5. [https://til.simonwillison.net/github-actions/github-pages](https://til.simonwillison.net/github-actions/github-pages)
6. [https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
