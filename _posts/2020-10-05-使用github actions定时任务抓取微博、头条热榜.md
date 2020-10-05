---
layout: post
title:  使用github actions定时任务抓取微博、头条热榜
date:   2020-10-05
---

## 使用方法

* 在项目中secrets设置GH_TOKEN
* 将hotboard.yml加入到项目的actions中，此action为定时任务，每五分钟运行一次(github规则最短5分钟)！
* 实际生效时间一般在30分钟左右，如未运行不防多等等

## 爬虫源码地址：
> https://github.com/shawze/hotboard

## action代码

```
name: hotboard
on:
  schedule:
    - cron:  '*/5 * * * *'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
    - name: set env
      run: |
        sudo pip3 install requests
        sudo pip3 install lxml
    - name: git clone
      run: |
        cd /tmp
        sudo git clone https://github.com/shawze/hotboard.git
    - name: run hotboard
      run: |
        sudo su
        cd /tmp/hotboard
        python3  githubaction.py ${{secrets.GH_TOKEN}}
```
