---
title: 在obsidian中写博客

date: 2023-08-26 21:58:34

tags:
---

### 目前写博客的工作流

1. 维护一个markdown库（这里要注意markdown的语法差异）
2. 通过框架处理成html
3. 一个服务器，放到上面即可

### 使用obsidian来写博客

1. 在obsidian中书写只是想让整体的写作体验更好一些
   1. 可以借助社区丰富的插件来做一些自动化工作或者有意思的工作
2. 和目前的日常工作流统一（日常工作在另一个vault）

### 博客如何自动更新

1. 博客内容放到一个git仓库，我自己是对应了两个远端
   1. 自己服务器上一个git仓库
   2. 一个在github用来做备份
2. 自己服务器的git仓库写了hooks,也就是仓库更新后会自动执行脚本，处理markdown，当前使用的hexo来作为处理博客的框架，当然也可以选择别的
3. 本地如何来操作呢？在obsidian中进行写，然后使用插件调用shell脚本推送仓库（在command palette 调用即可）
