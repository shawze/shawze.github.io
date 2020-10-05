---
layout: post
title:  Github Actions上传到腾讯COS
date: 2020-10-05
---

背景及用处：使用github pages创建自己的网站，但github速度过慢，同步腾讯cos设置静态托管加速访问；
使用了官方的代码和其它cos同步代码未成功，通过摸索以下代码可正常运行。

## 设置密钥、存储桶等信息
在Github项目中settings-->Secrets-->new Secret
设置以下信息：
* COS_BUCKET
* COS_REGION
* SECRETID
* SECRETKEY

## 上传代码
* 在Github项目中Actions-->set up a workflow yourself；
* 代码最后一个step根据自己的实际情况设置；
* coscmd config config 配置请使用$ { {  } },配置参数，去掉花括号内空格；

```
name: sync cos
on: 
  push:
    branches: 
    - main
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
    - name: install coscmd
      run: |
        pip3 install setuptools
        cd /tmp
        sudo git clone https://github.com/tencentyun/coscmd.git
        cd coscmd
        sudo python3 setup.py install
    - name: config coscmd
      run: | 
        coscmd config -a ${{ secrets.SECRETID }} -s ${{ secrets.SECRETKEY }} -b ${{ secrets.COS_BUCKET }} -r ${{ secrets.COS_REGION }}
    - name: Sync COS
      run: |
        cd /tmp
        sudo git clone https://github.com/shawze/shawze.github.io.git
        cd shawze.github.io
        # sudo rm -rf .git
        # coscmd delete -r -f /
        # coscmd upload -r ./ ./
        coscmd delete -r -f _posts
        coscmd upload -r _posts _posts
        
```
