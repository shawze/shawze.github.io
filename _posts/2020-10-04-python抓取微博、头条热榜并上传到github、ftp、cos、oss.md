---
layout: post
date:   2020-10-04
title:  python抓取微博、头条热榜并上传到github\ftp\cos\oss
---

```
# -*- coding: utf-8 -*-
#! /usr/bin/env python3

from datetime import datetime
import requests
from lxml import etree
import pprint
from qcloud_cos import CosConfig
from qcloud_cos import CosS3Client
import sys
import logging
from ftplib import FTP
import oss2
import json
import base64


class HotBrand():
    def __init__(self):
        self.weibo_url = 'https://s.weibo.com/top/summary?cate=realtimehot'
        self.toutiao_url = 'https://i.snssdk.com/hot-event/hot-board/?origin=hot_board'

    def fetch(self):
        data = []
        data += self.parse_toutiao()
        data += self.parse_weibo()
        self.html_format(data)

    def html_format(self,data):
        data_html = []
        for i,item in enumerate(data):
            text = f'''\
            <a href="{item['Url']}">
            <div class="card-v6-warpper" style="height: 52px;">
            <div class="card-index card-index-active">{i+1}</div>
            <div class="card-title">{item['Title'][:14]}</div>
            <div class="card-hot">{item['Site']}</div>
            </div>
            </a>
            '''
            data_html.append(text)
        data_html_text = ''.join(data_html)
        # pprint.pprint(data_html)
        # print(data_html_text)
        fetch_time = str(datetime.now().ctime())
        with open('hotboardtemplate.html','r',encoding='utf-8') as fb:
            html  = fb.read()
        html = html.replace('内容区域',data_html_text)
        html = html.replace('更新时间',f'更新时间：{fetch_time}')
        with open('hot.html','w',encoding='utf-8') as fb:
            fb.write(html)

    def uploadCOS(self):
        logging.basicConfig(level=logging.INFO, stream=sys.stdout)
        secret_id = ''  # 替换为用户的 secretId
        secret_key = ''  # 替换为用户的 secretKey
        region = 'ap-shanghai'  # 替换为用户的 Region
        token = None  # 使用临时密钥需要传入 Token，默认为空，可不填
        scheme = 'https'  # 指定使用 http/https 协议来访问 COS，默认为 https，可不填
        config = CosConfig(Region=region, SecretId=secret_id, SecretKey=secret_key, Token=token, Scheme=scheme)
        client = CosS3Client(config)
        file_name = 'hot.html'
        with open(file_name, 'rb') as fb:
            response = client.put_object(
                Bucket='',
                Body=fb,
                Key=file_name
            )
            print(response['ETag'])

    def uploadFtp(self):
        ftp = FTP(host='',user='', passwd='')  # connect to host, default port
        file_name = 'hot.html'
        with open(file_name, 'rb') as fb:
            rtn = ftp.storbinary(f'STOR {file_name}',fb)
            print(rtn)

    def uploadOSS(self):
        # 阿里云主账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM账号进行API访问或日常运维，请登录 https://ram.console.aliyun.com 创建RAM账号。
        auth = oss2.Auth('', '')
        # Endpoint以杭州为例，其它Region请按实际情况填写。
        bucket = oss2.Bucket(auth, '', '')

        file_name = 'hot.html'
        with open(file_name, 'rb') as fb:
            rtn = bucket.put_object(file_name, fb)
            print(rtn)

    def uploadGithub(self):
        file_name = 'hot.html'
        header = {"Accept": "application/vnd.github.v3+json",
                  "Authorization": 'token '+''}
        url = 'https://api.github.com/repos/shawze/shawze.github.io/contents/hot/index.html'
        with open(file_name, 'rb') as fb:
            fileDate = base64.b64encode(fb.read()).decode('utf-8')
        r = requests.get(url=url, headers=header)
        if r.status_code == 200:
            sha = r.json()['sha']
        else:
            sha = ''
        print(r.json())
        param ={"message":str(datetime.today()),
                "content":fileDate,
                "committer":
                    {"name": "",
                     "email": ""
                     },
                "sha": sha
                }
        rtn = requests.put(url,data=json.dumps(param),headers=header)
        print(rtn.text)

    def parse_toutiao(self):
        resp = requests.get(self.toutiao_url)
        data_resp = resp.json()
        if data_resp['status'] == "success":
            data = data_resp['data']
            data_lite =  [{
                           'Id': i+1,
                           'Title':item['Title'],
                           'Url':item['Url'],
                           'HotValue':item['HotValue'],
                           # 'Type': '',
                            'Site': '头条',
                           }
                          for i,item in enumerate(data)]
            # pprint.pprint(data_lite)
        return data_lite

    def parse_weibo(self):
        header = {
            'User-Agent': 'Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) '
                          'Chrome/85.0.4183.121 Mobile Safari/537.36',
            'Cookie': '',
            'DNT': '1',
            'Host': 's.weibo.com',
            'Sec-Fetch-Dest': 'document',
            'Sec-Fetch-Mode': 'navigate',
            'Sec-Fetch-Site': 'none',
            'Sec-Fetch-User': '?1',
            'Upgrade-Insecure-Requests': '1',
        }
        resp = requests.get(self.weibo_url,headers=header)
        resp = requests.get(self.weibo_url)
        resp_html = etree.HTML(resp.text)
        if resp_html != '':
            selector_id = resp_html.xpath('//td[@class="td-01 ranktop"]/text()')
            selector_title = resp_html.xpath('//td[@class="td-02"]/a/text()')
            selector_url = resp_html.xpath('//td[@class="td-02"]/a/@href')
            selector_hot_value = resp_html.xpath('//td[@class="td-02"]/span/text()')
            # selector_type = resp_html.xpath('//td[@class="td-03"]/i/text()')
            # print(len(selector_id))
            # print(len(selector_title))
            # print(len(selector_hot_value))
            # print(len(selector_type))

            data = zip(selector_id, selector_title, selector_url, selector_hot_value)
            data = list(data)
            data_lite = []
            for item in data:
                temp = {
                    'Id': int(item[0]),
                    'Title': item[1],
                    'Url': 'https://s.weibo.com' + item[2],
                    'HotValue': item[3],
                    'Site': '微博',
                }
                data_lite.append(temp)
            # pprint.pprint(data_lite)
        return data_lite


if __name__ == '__main__':
    hot_brand = HotBrand()
    hot_brand.fetch()
    hot_brand.uploadCOS()
    hot_brand.uploadFtp()
    hot_brand.uploadOSS()
    hot_brand.uploadGithub()


```
