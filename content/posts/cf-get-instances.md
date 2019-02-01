---
title: "omコマンドでCFコンポーネントのインスタンス数を取得する"
date: 2019-02-01T15:55:23+09:00
type: posts
tags: ["cloudfoundry", "PCF"]
categories: ["tips"]
draft: true
---
# 注
商用版CloudFoundry(Pivotal CloudFoundry)且つOpsManagerを使用している事が前提です。

## What's om command?
![om](https://camo.githubusercontent.com/016aa89165564423565b7ea6e32dde5e0f54f670/687474703a2f2f692e67697068792e636f6d2f336f37714451356977316f587944654a416b2e676966)  
[pivotal-cf/om](https://github.com/pivotal-cf/om)  
マントラが云々  
  

実際のところは多分OpsManagerコマンドってなもんで、READMEにも  
`magical tool that helps you configure and deploy tiles to an Ops-Manager 1.8`  
と書いてある。  

## 何に使うん？
OpsManagerにはapiいろいろ出来るAPIが生えているのですが、
1. uaacでトークンゲット
2. curlでHeaderやらいろいろ付与してAPIを叩く
という手順が必要になります。  
omコマンドはこれをいつもOpsManagerのログインに使用しているID:PASSを使ってAPIを叩く事が出来ます(自分はこの使い方くらいしかしてません…)  

```
uaac target https://YOUR_OPSMAN_IP/uaa

uaac token owner get
Client name: opsman
Client secret: JUST_PRESS_ENTER
User name: YOUR_USERNAME_HERE
Password: YOUR_PASSWORD_HERE

uaac context

curl "https://example.com/api/v0/staged/director/properties" \
    -X GET \
        -H "Authorization: Bearer UAA_ACCESS_TOKEN"
```
これが

```
om -t https://example.com -u username -p password curl -p /api/v0/staged/director/properties
```
こうじゃ  
…スッキリしましたね？  

多分、ベリー便利な使い方としてはomのサブコマンドである  
**aply-changes** とか **upload-product** とか **stage-product**  
ですはい。
