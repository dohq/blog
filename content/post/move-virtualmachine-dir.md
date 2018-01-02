+++
title = "VirtualMachineディレクトリを移動する"
data = 2018-01-03T03:08:22+09:00
tags = ["virtualbox", "archlinux"]
categories = ["virtualbox"]
draft = false
+++

# 既存仮想マシンがある状態でVirtualMachineディレクトリを移動したい
VirtualMachineディレクトリを移動してvirtualboxからaddしようとすると  

```
Failed to open virtual machine located in /mnt/ssd/VirtualMachine/Windows7/Windows7.vbox.

Cannot register the hard disk '<新VDIパス>' {UUID} because a hard disk '<旧VDIパス>' with UUID {UUID} already exists.

Result Code: NS_ERROR_INVALID_ARG (0x80070057)
Component: VirtualBoxWrap
Interface: IVirtualBox {9570b9d5-f1a1-448a-10c5-e12f5285adad}
```
って言われてつらかった。  
自分ではこれで上手くいったって手順なのであしからず

### 手順
`VBoxManage internalcommands sethduuid <新VDIパス>`

新しいUUIDをコピー  
vboxファイルの *HardDisk* セクションに上記UUIDで置換  
これでどうですかね
