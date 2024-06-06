---
title: 基于ASTRO的个人博客配置
published: 2024-06-06
description: 搁置了许久的一次博客更新
tags:
  - 折腾
category: 无用之用
draft: true
---
## 为什么选择ASTRO？
>[ASTRO](https://astro.build/)是最适合构建像博客、营销网站、电子商务网站这样的**以内容驱动的网站**的 Web 框架。

以上内容摘自 ASTRO 官网。

笔者之前也使用过 [Hexo](https://hexo.io/zh-cn/) 框架，Hexo 的优势在于只需要做一些简单的配置填空就可以直接部署了，它的定位也因此是博客框架，而 ASTRO 的定位是前端框架，笔者体验下来，感到对一些 ASTRO 主题进行魔改相比 Hexo 主题来说更容易，这和与之适配的`.astro`语言有一定关系，以**复杂是可选的**为旨，也提供了很大地再开发空间。

笔者个人目前想快速建立起（或者说修整好）个人的博客，同时希望保留一些可扩展性，为以后的个性化修改做铺垫，与上文中提到的 ASTRO 的宗旨不谋而合，因此最终选择它作为框架。

## 主题魔改
博客修改的主题是 [fuwari](https://github.com/saicaca/fuwari) ，从 demo 可以看出来基本与本站是一个模子里刻出来的：
![](基于ASTRO的个人博客配置/image-20240606213636357.png)

但是本站更倾向于使用绿色作为主题色，第一件事情就是把右上角主题色调整的滑动条关闭，同时固定主题色为绿色（hue:160），效果大致如下：
![](基于ASTRO的个人博客配置/image-20240606213901109.png)

可以看到在深色模式下原主题的背景包括各组件都做了和主题色的适配处理，整个画面看起来泛绿，和本人的头像相性很差，因此笔者又魔改了配色来协调。

本主题中颜色模型用到的是 oklch ，预设颜色定义在`src\components\GlobalStyles.astro`中，修改如下：
```css
color_set({
  --primary: oklch(0.70 0.14 var(--hue)) oklch(0.75 0.14 var(--hue))
  --page-bg: oklch(0.95 0.01 var(--hue)) oklch(0.1 0 0)
  --card-bg: white oklch(0.2 0 0)
  ...
})
```

原主题定义如下：
```css
color_set({
  --primary: oklch(0.70 0.14 var(--hue)) oklch(0.75 0.14 var(--hue))
  --page-bg: oklch(0.95 0.01 var(--hue)) oklch(0.16 0.014 var(--hue))
  --card-bg: white oklch(0.23 0.015 var(--hue))
  ...
})
```

修改后深色模式下的背景颜色不再和主题色相关。

除此之外，还将中文字体修改为了宋体，原主题中未为中文预设字体，默认中文字体观感很差，宋体在间距和对比度足够的情况下效果不差，因此暂时先选择宋体作为主力中文字体。

最后是一些配置填空和细节的修改，包括调低圆角等。

接下来的目标是更大程度地修改主题，使其外观上偏向终端风，绿色也会比目前更亮一些。本站预计不会放学术相关的内容，所以画风不会走向严肃规整型。

## 博客部署

本站最终部署在 Github Pages 上，利用了 Github Actions ，用到的 workflow(.github/workflow/deploy.yml) 如下：
```yaml
name: Deploy to GitHub Pages

on:
  # 每次推送到 `main` 分支时触发这个“工作流程”
  # 如果你使用了别的分支名，请按需将 `main` 替换成你的分支名
  push:
    branches: [ main ]
  # 允许你在 GitHub 上的 Actions 标签中手动触发此“工作流程”
  workflow_dispatch:

# 允许 job 克隆 repo 并创建一个 page deployment
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout your repository using git
        uses: actions/checkout@v4

      - name: Setup Node.js environment
        uses: actions/setup-node@v3
        with:
          node-version: 20

      - name: Install pnpm
        run: npm install -g pnpm@latest

      - name: Install dependencies
        run: pnpm install

      - name: Build your site
        run: pnpm run build

      - name: Upload your site
        uses: withastro/action@v2
        with:
          path: . # 存储库中 Astro 项目的根位置。（可选）
          node-version: 20 # 用于构建站点的特定 Node.js 版本，默认为 20。（可选）
          package-manager: pnpm@latest # 应使用哪个 Node.js 包管理器来安装依赖项和构建站点。会根据存储库中的 lockfile 自动检测。（可选）

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

利用 GPT 结合报错和官方提供的[工作流](https://docs.astro.build/zh-cn/guides/deploy/github/)进行了修改。

在博客内容写作上，使用 Obsidian 的 Templater 插件和 Custom Attachment Location 插件作为复制，前者可以保证博客文件在模板规则限制上过关，后者可以更好地管理附件，目前还没有对博客底层的部署逻辑做修改去完善自动化部署服务，因此只能在写时多下些功夫。

由于 Obsidian 仓库和博客文件夹分离，选择使用如下脚本完成从前者到后者的手动单向同步：
```powershell
robocopy  D:\obsidian D:\page\src\content\posts\ /MIR /R:0 /W:0 /LOG:SyncLog.txt /TEE
```

## 总结
拖了小半年终于把这事给做了，接下来可以多写多总结些东西了，这对于整理思路和所学都是大有裨益的。