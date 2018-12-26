---
title: "BOSH Lite on cf-deployment"
date: 2018-12-05T18:51:47+09:00
tags: ["bosh", "cloudfoundry"]
categories: ["tips"]
draft: true
---

# BOSH-lite にOSS版cloudfoundryをデプロイしてオレオレローカルPaaSを構築する
## ネタが無い！
やっつけ感が半端無くて心苦しいのですが、ローカル環境bosh-lite(VirtualBox on BOSH)を立てて
さらにその上にOSS版CFを構築した時の知見と躓いたとこを書いていこうかなと
**Linux上でしか試してません！**

## 構築環境
```bash
                   -`
                  .o+`
                 `ooo/                 OS: Arch Linux
                `+oooo:                Kernel: x86_64 Linux 4.14.86-1-lts
               `+oooooo:               Shell: vim
               -+oooooo+:              CPU: Intel Core i5-2520M @ 4x 3.2GHz [46.0°C]
             `/:-:++oooo+:             GPU: intel
            `/++++/+++++++:            RAM: 546MiB / 15927MiB
           `/++++++++++++++:
          `/+++ooooooooooooo/`
         ./ooosssso++osssssso+`
        .oossssso-````/ossssss+`
       -osssssso.      :ssssssso.
      :osssssss/        osssso+++.
     /ossssssss/        +ssssooo/-
   `/ossssso+/:-        -:/+osssso+-
  `+sso+:-`                 `.-/+oso:
 `++:.                           `-/+/
 .`                                 `/

$ VBoxManage --version
5.2.22r126257

$ pacman -Qi bosh-cli
Name            : bosh-cli
Version         : 3.0.1-1
Description     : BOSH command line interface tool
Architecture    : x86_64
URL             : https://bosh.io/
Licenses        : Apache2
Groups          : None
Provides        : None
Depends On      : None
Optional Deps   : openssh: bosh ssh [installed]
Required By     : None
Optional For    : None
Conflicts With  : None
Replaces        : None
Installed Size  : 15.68 MiB
Packager        : Unknown Packager
Build Date      : Tue 31 Jul 2018 08:31:48 AM JST
Install Date    : Tue 31 Jul 2018 08:32:13 AM JST
Install Reason  : Explicitly installed
Install Script  : No
Validated By    : None
```

## てじゅーん
つらつらと書いていきます

### BOSH-liteデプロイ
```bash
mkdir ~/workspace
cd ~/workspace
git init
git submodule add https://github.com/cloudfoundry/bosh-deployment
```

[ドキュメント](https://bosh.io/docs/bosh-lite/) に書いてある

1. `bosh create-env` コマンドを実行して
2. 構築されたBOSH-liteへのaliasを張って
3. ip routeで構築されたbosh-liteへのルーティング追加して

をまとめて実行してくれるスクリプトがあります。
`bosh-deployment/virtualbox/create-env.sh`
便利ですね！

**ただ、** 今回は自分で構築用スクリプトを書いていきます。
詳しい理由は後述しますが、この後対象のスクリプトに手を入れる事になる為gitでの管理がちと面倒になる為です。

という訳で上のドキュメントに沿ってやっていきます。

#### 構築用スクリプト
```bash
vim bosh-deploy.sh

bosh create-env bosh-deployment/bosh.yml \
  --state state.json \
  --vars-store bosh-lite-creds.yml \
  -o bosh-deployment/virtualbox/cpi.yml \
  -o bosh-deployment/virtualbox/outbound-network.yml \
  -o bosh-deployment/bosh-lite.yml \
  -o bosh-deployment/uaa.yml \
  -o bosh-deployment/credhub.yml \
  -o bosh-deployment/jumpbox-user.yml \
  -v director_name=bosh-lite \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork

      ※ドキュメント上には `bosh-deployment/bosh-lite-runc.yml` がありますが、
      中身は空で互換性の為と書かれているので今回は外しました。


chmod +x bosh-deploy.sh
./bosh-deploy.sh
```
さて `bosh-deploy.sh` を実行すると必要なファイルのダウンロードとコンパイルが実行されます。
最後に
```bash
Finished deploying (00:04:36)

Stopping registry... Finished (00:00:00)
Cleaning up rendered CPI jobs... Finished (00:00:00)

Succeeded
```
と出てればデプロイ成功です。
さて、構築したbosh-lite環境(VirtualBox上に出来ているVM)ですが実はデフォルトだと
```
CPU: 2
RAM: 4G
ディスク: 16G
```
となり、今回立てたいCloudFoundry環境にはちと足りません(主にメモリ)
なのでこのVMをスペックアップします。

#### bosh-liteのスケールアップ
VirtualBox上に構築されているVMなので設定からポチーでええやん!
と思いきやこれは出来ません。
BOSH-liteに関しては、VMをシャットダウンするとBOSHを再デプロイする必要があります。
なのでその際にbosh-deploymentで設定されているVMスペックに戻ってしまいます。
そこで [OpsFiles](https://bosh.io/docs/cli-ops-files/) を使用してVMスペックを変更します。
```bash
$ mkdir ops-files
$ vim ops-files/bosh-lite-scale-up.yml

- type: replace
  path: /resource_pools/name=vms/cloud_properties?
  value:
    cpus: 2
    memory: 8192
    ephemeral_disk: 32_768

bosh create-env bosh-deployment/bosh.yml \
  --state state.json \
  --vars-store bosh-lite-creds.yml \
  -o bosh-deployment/virtualbox/cpi.yml \
  -o bosh-deployment/virtualbox/outbound-network.yml \
  -o bosh-deployment/bosh-lite.yml \
  -o bosh-deployment/uaa.yml \
  -o bosh-deployment/credhub.yml \
  -o bosh-deployment/jumpbox-user.yml \
  -o ops-files/bosh-lite-scale-up.yml \ <- 追加！
  -v director_name=bosh-lite \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork

./deploy-bosh.sh
```
VMを作りなおすので時間がボチボチかかります。 :cofee:
```
~~~
Guest OS:        Ubuntu (64-bit)
Memory size:     8192MB
Page Fusion:     off
VRAM size:       16MB
CPU exec cap:    100%
HPET:            off
Chipset:         piix3
Firmware:        BIOS
Number of CPUs:  2
~~~
```
大丈夫そうです。

#### routeを追加する
BOSH-lite上にデプロイされるコンポーネントに対してホスト上からアクセス出来るようにrouteを追加します。  
これはドキュメントの内容まんまですね。

`sudo ip route add   10.244.0.0/16 via 192.168.50.6`

#### bosh-cliを使えるように
boshのcliを通してBOSH-liteに接続出来るよういくつかの環境変数を設定します。
```bash
$ vim .envrc

export BOSH_CLIENT=admin
export BOSH_ENVIRONMENT=192.168.50.6
export BOSH_CA_CERT=$(bosh int bosh-lite-creds.yml --path /director_ssl/ca)
export BOSH_CLIENT_SECRET=$(bosh int bosh-lite-creds.yml --path /admin_password)

export CREDHUB_SERVER=https://192.168.50.6:8844
export CREDHUB_CA_CERT="$(bosh int bosh-lite-creds.yml --path=/credhub_tls/ca)
$(bosh int bosh-lite-creds.yml --path=/uaa_ssl/ca)"
export CREDHUB_CLIENT=credhub-admin
export CREDHUB_SECRET=$(bosh int bosh-lite-creds.yml --path=/credhub_admin_client_secret)

source .envrc または
direnv allow(direnvを使用してるなら)
```
これで構築したBOSH-liteに対してコマンドが発行出来るようになりました。
```bash
$ bosh env

bosh-lite
1b4517e6-e819-4ef1-9cfc-074ee8e663ac
268.3.0 (00000000)
warden_cpi
compiled_package_cache: disabled
config_server: enabled
local_dns: enabled
power_dns: disabled
snapshots: disabled
admin
```
やったぜ

#### stemcellのアップロード
BOSHを通してコンポーネントをデプロイする際は、ベースとなるOSが必要になります。(これを[stemcell](https://bosh.io/stemcells/)といいます)  
そのOSは環境毎に違うので、BOSH-lite用のものをアップロードします。  
※BOSH-liteの場合はWarden (BOSH Lite)を、更にはtrustyとxenialどちらも必要です。  
基本は最新がいいので、bosh.io/stemcells/を見るといいです。
```bash
$ bosh upload-stemcell --sha1 2976b5db401617ce92941e4228a37b6b87a31de1 \
  https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-xenial-go_agent\?v=170.13

$ bosh upload-stemcell --sha1 40ac356e7153e8f070a9dddf0863dba5b39ffb0b \
  https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent\?v=3586.60


$ bosh stemcells

Using environment '192.168.50.6' as client 'admin'

Name                                         Version  OS             CPI  CID
bosh-warden-boshlite-ubuntu-trusty-go_agent  3586.60  ubuntu-trusty  -    9a0c76ce-7b3d-41d6-70e0-6fc4c0bfe4c4
bosh-warden-boshlite-ubuntu-xenial-go_agent  170.13   ubuntu-xenial  -    83216c54-c7ca-4a16-4f07-3419284517ed

(*) Currently deployed

2 stemcells

Succeeded
```

#### cloud-configのアップデート
BOSH-liteに限った話ではないですが、BOSHが動いている環境の設定はcloud-configというものによって管理されています。(AWSとかGCPとかVirtualBoxとかvSphereとか…)
なのでVirtualBox用cloud-configを適用します。
```bash
$ bosh update-cloud-config bosh-deployment/warden/cloud-config.yml

y
```

#### ちょっと試す
以上でBOSH-liteの基本的な構築は完了なので、ちょっと動作確認してみます。  
試しにnode-exporterをデプロイしてみましょ
```bash
$ bosh -d node-exporter deploy <(wget -O- https://raw.githubusercontent.com/bosh-prometheus/node-exporter-boshrelease/master/manifests/node-exporter.yml)

Redirecting output to ‘wget-log.7’.
Using environment '192.168.50.6' as client 'admin'

Using deployment 'node-exporter'

Release 'node-exporter/4.1.0' already exists.

+ azs:
+ - name: z1
+ - name: z2
+ - name: z3

+ vm_types:
+ - name: default

+ compilation:
+   az: z1
+   network: default
+   reuse_compilation_vms: true
+   vm_type: default
+   workers: 5

+ networks:
+ - name: default
+   subnets:
+   - azs:
+     - z1
+     - z2
+     - z3
+     dns:
+     - 8.8.8.8
+     gateway: 10.244.0.1
+     range: 10.244.0.0/24
+     reserved: []
+     static:
+     - 10.244.0.34
+   type: manual

+ disk_types:
+ - disk_size: 1024
+   name: default

+ stemcells:
+ - alias: default
+   os: ubuntu-xenial
+   version: '170.13'

+ releases:
+ - name: node-exporter
+   sha1: bc4f6e77b5b81b46de42119ff1a9b1cf5b159547
+   url: https://github.com/bosh-prometheus/node-exporter-boshrelease/releases/download/v4.1.0/node-exporter-4.1.0.tgz
+   version: 4.1.0

+ update:
+   canaries: 1
+   canary_watch_time: 1000-30000
+   max_in_flight: 32
+   serial: false
+   update_watch_time: 1000-30000

+ instance_groups:
+ - azs:
+   - z1
+   instances: 1
+   jobs:
+   - name: node_exporter
+     release: node-exporter
+   name: node-exporter
+   networks:
+   - name: default
+   stemcell: default
+   vm_type: default

+ name: node-exporter

+ vm_extensions: []

Continue? [yN]: y

Task 9

Task 9 | 22:59:16 | Preparing deployment: Preparing deployment (00:00:00)
Task 9 | 22:59:16 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 9 | 22:59:16 | Creating missing vms: node-exporter/dc319609-355e-4748-a9a6-f9bd9e9e413f (0) (00:00:09)
Task 9 | 22:59:25 | Updating instance node-exporter: node-exporter/dc319609-355e-4748-a9a6-f9bd9e9e413f (0) (canary) (00:00:10)

Task 9 Started  Thu Dec 13 22:59:16 UTC 2018
Task 9 Finished Thu Dec 13 22:59:35 UTC 2018
Task 9 Duration 00:00:19
Task 9 done

Succeeded
```
デプロイは出来たので動作確認も
```bash
$ bosh vms
node-exporter/dc319609-355e-4748-a9a6-f9bd9e9e413f	running	z1	10.244.0.2	75a1ba31-ea73-4f8b-4049-e3f0e1f2c64c	default	true	
Using environment '192.168.50.6' as client 'admin'

Task 13. Done

Deployment 'node-exporter'

Instance                                            Process State  AZ  IPs         VM CID                                VM Type  Active
node-exporter/dc319609-355e-4748-a9a6-f9bd9e9e413f  running        z1  10.244.0.2  75a1ba31-ea73-4f8b-4049-e3f0e1f2c64c  default  true

1 vms

Succeeded

$ curl http://10.244.0.2:9100
<html>
                        <head><title>Node Exporter</title></head>
                        <body>
                        <h1>Node Exporter</h1>
                        <p><a href="/metrics">Metrics</a></p>
                        </body>
                        </html>%
```

やったぜ(2回目)

### cloudfoundry構築
こんな長い記事書いたこと殆どないのでもう疲れたよパトラッシュな感じですが、頑張っていきまっしょい。  

```bash
git submodule add https://github.com/cloudfoundry/cf-deployment
```

### 
