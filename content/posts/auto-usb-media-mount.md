---
title: "USBストレージを自動でマウントする"
date: 2023-06-12T11:44:46+09:00
tags: ["linux", "ArchLinux"]
categories: ["tips"]
draft: false
---
# 背景
UbuntuとかManjaroみたいに最初からユーザフレンドリーな統合デスクトップ環境がセットアップされているOSの場合、USBストレージを刺した際に自動でマウントされファイラ等から容易にアクセスする事が出来る。

しかし、統合デスクトップ環境を利用せずに1から構築されたLinux環境の場合、殆どの環境でUSBストレージは自動でマウントされない。

`sudo mount /dev/sdxY ~/myusb` みたいにsudoしたくないし、fstabに書いてマウントもしたくない。

# 解決策
自分の環境がArchLinuxなのでそれに則る。
```
yay -S --needed udisks2 udiskie
```

.xinitrcとかi3のconfigで `udiskie --tray` とかして自動起動するようにしておくと、USBメディアを刺した際自動的に `/run/media/$USER/` 配下にマウントされるようになる。

自動でマウントしたくないとか通知いらないといった場合はudiskieのUsageを参照のこと。


# 参考
[https://wiki.archlinux.jp/index.php/Udisks](https://wiki.archlinux.jp/index.php/Udisks)

[https://github.com/coldfix/udiskie/wiki/Usage](https://github.com/coldfix/udiskie/wiki/Usage)
