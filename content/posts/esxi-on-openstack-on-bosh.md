---
title: "ESXi on OpenStack on BOSHでの注意"
date: 2019-02-16T00:42:31+09:00
tags: ["bosh", "openstack"]
categories: ["tips"]
draft: false
---
OpenStackのインストールからは書ける気がしないので割愛  
主に設定についての部分だけ抜粋  
ちなみにOpenStackはESXi上に[Mirantis OpenStack](https://www.mirantis.com/software/openstack/download/)で  
Controller/Compute・Cinder/Kibana・Grafanaプラグイン  
の3台構成

## ESXi on OpenstackでVMが起動しない
ESXi本体の設定ファイル更新(要SSHログイン)
```diff
vi /etc/vmware/config

+ vhv.enable = "TRUE"

/etc/init.d/hostd restart
```

## OpenStack上のVMが鬼のように遅い
BOSHのデプロイだけで4時間とかうせやろ…と思ったが、  
ここはOpenStackのバージョンとか導入方法によって変わるのかな…  
(packstackでnewtonインストールした時はそんな事なかったはず)  
どうやら(少なくともMOS9.2は)仮想化方法の初期設定はqemuを使うっぽい…  
なのでkvmに設定してあげればおk  
MOSだとGUIからEnvitonment->Settings->Computeで変更出来る  
*nova.conf* を書き換える場合、
```diff
[libvirt]
- virt_type=qemu
+ virt_type=kvm
+ hw_machine_type = "x86_64=pc-i440fx-trusty,i686=pc-i440fx-trusty"
```

で大丈夫そう。  
**hw_machine_type** についてはアーキテクチャとOSを  
`virsh capabilitie`  
の出力から探すとよさそう。

## OpenStack上のVolume削除が鬼のように(ry
やっとこさ作ったBOSH環境をなんかのアレで消して作りなおしたい時に
`bosh delete-env ...`  
とやる訳なんですがvolumeの削除に時間がかかり過ぎて…
```
...

  Deleting disk 'a383dda4-76ce-4928-ac57-1a5e17d5e7e7'... Failed (00:05:07)
Failed deleting deployment (00:05:43)

Stopping registry... Finished (00:00:00)
Cleaning up rendered CPI jobs... Finished (00:00:00)

Deleting deployment:
  Deleting disk in the cloud:
    CPI 'delete_disk' method responded with error: CmdError{"type":"Bosh::Clouds::CloudError","message":"Timed out waiting for Volume `a383dda4-76ce-4928-ac57-1a5e17d5e7e7' to be deleted","ok_to_retry":false}

Exit code 1
```

*んあ゛あ゛あ゛ぁああ* てなもんですよ。  
実はOpenStackではVolumeの削除時にデフォルトで **zero overwrite(volumeをzeroで全て上書く)** するらしい。  
なので65Gのvolume消すのに3時間もかかる。  
これも *cinder.conf* を書き換える。
```diff
vi /etc/cinder/cinder.conf

[LVM-backend]
+ volume_clear=none

service cinder-volume restart
```

上の例では *[LVM-backend]* の下に追記したが、Cinderで使用しているバックエンドの配下に設定を入れる必要がある。  
(NFSとかネットワーク越しのディスクだとトラフィックやばそうだけどどうなんだろ…)  

### まとめ
OpenStackムズカシイ
