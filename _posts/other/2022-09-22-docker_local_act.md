---
title: github action自动将drawio转图片
tagline: ""
last_updated: null
category : code-server
layout: post
tags : [https, code-server]
---
可以将在https://app.diagrams.net/画的图的原始.drawio文件保存到github仓库,    
通过配置github工作流自动生成图片
### github workflows
在.github/workflows/deploy_action.yml配置rlespinasse/drawio-export-action
in
master
```
name: Keep draw.io export synchronized
on:
  push:
    branches:
      master
    paths:
      "*.drawio"
jobs:
  drawio-export:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout sources
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Export drawio files to asciidoctor and png files
        uses: rlespinasse/drawio-export-action@v2
        with:
          format: png
          transparent: true

      - name: Push to GitHub
        uses: EndBug/add-and-commit@v7.2.1
        with:
          branch: master
          message: 'Convert .Drawio to PNG'

```

### 使用act进行github action本地调试
#### act安装
```
cd /home/code_github/
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
```
#### 列出可用的actions
```
/home/code_github/bin/act  -l
```
#### 本地运行action

```
/home/code_github/bin/act -n
```