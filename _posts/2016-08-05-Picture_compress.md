---
layout: post
title: 图片压缩命令
categories: [blog]
tags: [Efficiency]
description: 
---

## 图片压缩命令

### png

`brew install pngcrush`

```
pngcrush -brute jmeter_shot.png jmeter_small.png
```
```
for file in *.png ; do pngcrush "$file" "${file%.png}-crushed.png" && mv "${file%.png}-crushed.png" "$file" ; done
```
### jpg

`brew install jpegoptim`

```
jpegoptim jmeter_shot.jpg
```

**alias**

```
alias pngc="pngcrush -brute"
alias jpgc="jpegoptim"
```

