---
title: "SSHポートフォワーディングを多段で行う"
date: 2019-01-25T09:46:03+09:00
type: posts
tags: ["linux", "ssh"]
categories: ["tips"]
draft: false
---

# なんで
外出先から自宅にあるESXiのWEB管理画面が見たいとか…ニッチすぎるか

## 図
```
┌────────┐                 ┌──────┐                   ┌──────┐          ┌──────────┐
│手元のPC│                 │踏み台│                   │自宅PC│          │自宅サーバ│
└───┬────┘                 └──┬───┘                   └──┬───┘          └────┬─────┘
    │ ポート8080(なんでもいい)│                          │                   │
    │ ────────────────────────>                          │                   │
    │                         │                          │                   │
    │                         │ ポート44444(なんでもいい)│                   │
    │                         │ ─────────────────────────>                   │
    │                         │                          │                   │
    │                         │                          │     ポート443     │
    │                         │                          │ ──────────────────>
┌───┴────┐                 ┌──┴───┐                   ┌──┴───┐          ┌────┴─────┐
│手元のPC│                 │踏み台│                   │自宅PC│          │自宅サーバ│
└────────┘                 └──────┘                   └──────┘          └──────────┘
```

## 結論
基本的には普通のポートフォワードコマンドに則る  
例として  
手元のPC: localhost  
踏み台: bastion  
自宅PC: home  
自宅サーバ: server  
で繋げられるものとする

`ssh -L 8080:localhost:44444 bastion ssh -4 -L 44444:server:443 home`

これで `https://127.0.0.1/` にてESXiの管理画面にアクセス出来るはず。  
ちょっと困ったのが
```
bind [::1]:44444: Cannot assign requested address
```
のエラーが出た時。
ipv6がONになってる鯖を踏み台にした際に起きるっぽいので `-4` オプションでipv4のみ使用するようにした。
