---
weight: 1
title: "个人博客搭建之搭建篇-(Hugo + Github Pages + Github Action自动发布)"
date: 2025-08-18T15:35:59+08:00
lastmod: 2025-08-18T15:35:59+08:00
draft: false
author: "Stef"
authorLink: ""
description: "这篇文章利用Hugo + Github Pages搭建个人博客，并用Github Action实现自动发布."
images: []
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Hugo", "Github Pages", "Github Action"]
categories: ["个人博客搭建"]

lightgallery: true
---

> 利用Hugo + Github Pages搭建个人博客，并用Github Action实现自动发布。

# 0. 使用到的工具
* Hugo: 静态页面生成工具。
* Github Pages: 静态网站托管平台。
* Github Action: 自动部署工作流，将`Hugo`生成的静态页面发布到`Github Pages`上。

# 1. 常见做法
* 做法1: `发布仓库（Pages站点仓库）` 和 `内容仓库（markdown源码）` 分为两个仓库，在`内容仓库（markdown源码）`提交文章后，自动触发`内容仓库`预先配置的Actions,执行对应的action构建打包并发布到`Github Pages站点仓库`,随后访问`https://<username>.github.io/`即可看到博客。
* 做法2(本文做法):	一个仓库，多个分支模式，`main`分支存放`内容源码(如markdown文章)`， `gh-pages`分支存放通过`Github Action`生成的静态页面。

# 2. 开始搭建
### 2.1 hugo环境准备
```bash
# 安装hugo
brew install hugo
# 🔥:安装的是最新版的hugo，版本v0.148,博主用的主题LoveIt在v0.148报错，所以吧hugo版本降为0.145.0 后正常
# 查看版本
hugo version
# 创建site,即:博客
hugo new site blog
```
### 2.2 为博客添加一个主题
```bash
# 进入到blog目录下
cd blog
# 🔥 注意:以下命令都是在`blog目录`下执行，`blog目录`是 `site`的根目录 
# 需要本地安装git
git init
# 以`git submodule`的方式添加主题，比`git clone`灵活，方便版本管理与后续更新
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
# 下载后的`LoveIt`主题在 `themes`下，确认下
ls -al themes/LoveIt
```

### 2.3 查看`LoveIt theme`自带`exampleSite`的效果并创建一篇文章
拷贝exampleSite到`site`的根目录下
```bash
# 在`site`的根目录下执行
cp -rf  themes/LoveIt/exampleSite/*  ./
```
调整`hugo.toml`配置文件
```yml
baseURL = "https://<username>.github.io/"

# theme
# 🔥主题，必须存在
theme = "LoveIt"
# themes directory
# 🔥主题目录，需要注释掉
# themesDir = "../.."

# website title
# 网站标题
title = "<username>'s Blog"
# 🔥是否使用 git 信息，此处需要设置为false，否则在`hugo server -D`时会报错
enableGitInfo = false

```
查看`LoveIt theme`自带`exampleSite`的效果
```bash
hugo server -D
```
在浏览器打开 `http://localhost:1313/` 即可查看，接下来创建一个测试文章
```bash
hugo new posts/my-first-post.md
```
随后在`content/posts`下可以找到`my-first-post.md`文件，用自己的熟悉的markdown编辑器编辑文章即可
### 2.4 使用`github action`自动部署到`github pages`
登录自己的github,创建`github pages`仓库，仓库名应为 `<username>/<username>.github.io`,
创建`.gitignore`文件，防止无用文件提交到仓库
```
# Hugo 构建输出（由 Actions 生成）
/public/
/resources/_gen/
.hugo_build.lock

# 操作系统缓存
.DS_Store
Thumbs.db
desktop.ini

# 开发环境配置
.idea/            # JetBrains IDE
.vscode/          # VS Code
.env              # 敏感环境变量
*.swp             # Vim 临时文件

# 主题子模块处理（推荐方式）
themes/*          # 忽略主题内容
!themes/          # 保留目录（允许子模块存在）
themes/**/*       # 递归忽略主题内文件
!themes/*/        # 保留每个主题的目录名
```
初始化git config信息(可选)，
```bash
git config --local user.email  your@gmail.com
git config --local user.name your_name
#查看git config信息
git config --list
```
与blog的仓库进行关联
```bash
# 二选一
git remote add origin https://github.com/<username>/<username>.github.io.git
# ssh方式，git push时相对稳定些，需要把自己`id_rsa.pub`添加到github中
git remote set-url origin  git@github.com:<username>/<username>.github.io.git
```
(🔥注意)启用 GitHub Pages 的 Actions 写入权限,进入你的 Git Pages仓库,操作路径:Settings → Actions → General,在 Workflow permissions 里选择：`Read and write permissions`（Read-only）, → Save. 然后上传blog代码到github。
```
git branch -M main
git add .
git push -u origin main
```
添加自动部署脚本文件，在`.github/workflows`下创建`deploy.yml`文件
```yml
name: Deploy Hugo site to Pages

on:
  push:
    branches:
      - main   # 🚨 这里写你 Hugo 源码所在的分支，比如 main 或 master

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.145.0'   # 固定版本的话可以写具体版本号

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public   # Hugo 默认输出目录
          # 下面两个选项会自动生成 .nojekyll，防止 GitHub Pages 当成 Jekyll
          enable_jekyll: false
          force_orphan: true
```
将`deploy.yml`文件提交后，会开始部署，不出意外会构建失败，需要在GitHub Pages 选择`gh-pages`分支，进入你的 Git Pages仓库,操作路径: Settings → GitHub Pages -> Page,在 Source 里选择: gh-pages 分支, Save。 等待20+秒，构建并部署成功后，即可在浏览器访问`https://<username>.github.io/`

> 会自动创建`gh-pages`分支,好多博客用的是`PERSONAL_TOKEN`, 其实用`GITHUB_TOKEN`即可，`GITHUB_TOKEN`是github提供的，不像`PERSONAL_TOKEN`需要自己手动生成，并在仓库中设置，比较繁琐。

# 3. 总结
自此，博客已成功自动构建&部署到github pages上，但还有很多优化事项没做，如 图床方案，主题调整，添加评论插件，添加访问量统计等等，后面有时间再把待优化的事项完成。

# 4. 遇到的错误
## 4.1 GitHub Actions时报错。
异常:
```
failed to extract shortcode: template for shortcode "style" not found
```
解决方法:
```yml
# 1. 调整hugo-version: '0.145.0'
- name: Setup Hugo
  uses: peaceiris/actions-hugo@v3
  with:
    hugo-version: '0.145.0'  # 👈
# 2.在GitHub Actions的deploy.yml文件中，中启用 submodule
- name: Checkout source
  uses: actions/checkout@v4
  with:
    submodules: true   # 👈 开启子模块
    fetch-depth: 0     # 👈 获取全部历史记录
```
## 4.2 构建时尝试读取git log异常。
异常:
```
Failed to read Git log: fatal: your current branch 'main' does not have any commits yet”
```
解决方法:
```yml
# 调整`hugo.toml`配置文件
# 🔥是否使用 git 信息，此处需要设置为false，否则在`hugo server -D`时会报错
enableGitInfo = false
```

# 5. 参考文章
1. [用Hugo构建我的Blog](https://weilanjin.github.io/posts/%E6%88%91%E7%9A%84hugo/)
2. [利用GitHub Action实现Hugo博客在GitHub Pages自动部署](https://juejin.cn/post/7399982698854891583)
3. [基于 Github Action 自动构建 Hugo 博客](https://www.lixueduan.com/posts/blog/01-github-action-deploy-hugo/)