+++
title = '快速开始'
linkTitle = '快速开始'
description = '克隆、安装依赖、启动开发服务器，三步看到页面。'
date = 2026-01-10
weight = 10
difficulty = 'beginner'
estimatedTime = 10
prerequisites = ['/docs']
outcomes = ['在本地跑起这个站点', '知道改哪个文件会改到哪一页']
tags = ['入门']
+++

## 克隆并启动

```bash
git clone --recurse-submodules https://github.com/hencter/hugo-scratch.git
cd hugo-scratch
hugo server
```

## 确认改对了地方

打开 `content/docs/quick-start.md`，改一句话，保存，浏览器会自动刷新。
