---
layout: post
title: OSX小功能
categories: [blog]
tags: [Efficiency]
description: 
---

OSX小功能

### **输出带声调字母**

在英文输入法环境下，长按字母键不放，同时按tab键。

### **输出特殊符号**

```
option键＋字母键

100˚C ——> option+k

∞ ——> option+5

Sim•Fancis ——> option+8

≈ ——> option+x

≤ ——> option+<

≥ ——> option+>

≠ ——> option+=

¥ ——> option+y

```

### **快捷方式**

选中文件，command＋L

### **Chrome**

Ctrl+Shift+T 快捷键可以打开上一个被关闭的选项卡。
Ctrl+Shift+N 在隐身模式下打开新窗口

###  [Mac 挂载NTFS移动硬盘进行读写操作](http://blog.csdn.net/sunbiao0526/article/details/8566317)

1. diskutil info /Volumes/YOUR_NTFS_DISK_NAME 

找到 Device Node

Device Node:              /dev/disk1s1

2. hdiutil eject /Volumes/YOUR_NTFS_DISK_NAME

"disk1" unmounted.
"disk1" ejected.

弹出你的硬盘

3. 创建一个目录，稍后将mount到这个目录 

sudo mkdir /Volumes/MYHD

4. 将NTFS硬盘 挂载 mount 到mac

sudo mount_ntfs -o rw,nobrowse /dev/disk1s1 /Volumes/MYHD/



