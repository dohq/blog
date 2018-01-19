---
title: "EC2でntpの設定が効かない"
date: 2018-01-19T14:29:46+09:00
type: posts
draft: False
---

### EC2上のLinux(Centos5)でntp時刻同期が出来ない
現行のサービスにて、EC2上でCentOS5のシステムがあります。  
ところが最近このマシンの時刻が早まってきてまして、いろいろと不都合が…  
ntpサービスの導入をしていなかった事から  
「まぁNTPの設定すればええやろ」  
と軽い考えだったのですが、ntpの設定しようがnptdateコマンド打とうが時刻が合ってくれない…  
なんていうことがありましたとさ


### 原因
EC2インスタンスはXen環境で動いているようで、 **Domain-0 の時刻に引っぱられる** ようです。  
なので、カーネルパラメータからこれを無効にすればOKです。


### 対策
```
/etc/sysctl.conf


xen.independent_wallclock = 1  <- 追加


sysctl -p
```

即反映させる場合は、  
```
echo '1' > /proc/sys/xen/independent_wallclock
```

これで、`ntpdate ntp.nict.jp`でok
