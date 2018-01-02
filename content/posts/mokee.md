+++
title = "ArchLinuxでMokeeをビルドする"
date = "2017-01-11T02:09:46Z"
tags = ["android", "Mokee"]
categories = ["tips"]
+++

## 自分メモ Mokee(MM)のビルド
Xiaomi Mi5(gemini)のMokeeをビルドしたのでそのメモ
尚、対象デバイスはgemini(Xiaomi Mi5)とする。

開発ツール系はインストールしてあるよね？

### 環境

```
OS: Arch Linux
Kernel: x86_64 Linux 4.8.13-1-ARCH
Uptime: 4d 23h 59m
Packages: 836
Shell: zsh 5.3.1
CPU: Intel Core i5-2520M CPU @ 3.2GHz
RAM: 7292MiB / 15934MiB
```

### 以下雑なコマンド

```
yaourt -S repo imagemagick
mkdir ~/Mokee && cd $_
virtualenv2 venv
repo init -u https://github.com/MoKee/android.git -b mkm
repo sync -j16
. build/envsetup.sh
. venv/bin/activate
breakfast gemini
mka bacon
```


### その他メモ
selinux permissiveのkernelを作る

device/xiaomi/gemini/BoardConfig.mk

```
 # Kernel
 BOARD_KERNEL_BASE := 0x80000000
-BOARD_KERNEL_CMDLINE := console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom user_debug=31 msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 cma=32M@0-0xffffffff
+BOARD_KERNEL_CMDLINE := console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom user_debug=31 msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 cma=32M@0-0xffffffff androidboot.selinux=permissive
 BOARD_KERNEL_IMAGE_NAME := Image.gz-dtb
 BOARD_KERNEL_PAGESIZE := 4096
 BOARD_KERNEL_TAGS_OFFSET := 0x00000100
```
