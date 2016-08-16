---
layout: post
title: OSX Environment
categories: [blog]
tags: [Efficiency]
description: 
---

OSX Environment

#### brew

```shell
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

[see more](http://brew.sh/index_zh-cn.html)

#### git

```shell
brew install git
```

#### oh-my-zsh

**auto install**

```shell
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
```

**manually operated**

```shell
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
chsh -s /bin/zsh
```

```shell
plugins=(git osx ruby autojump)
export JAVA8_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_73.jdk/Contents/Home"
export JAVA7_HOME="/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home"
alias ll="ls -al"
alias usb="cd /Volumes"
alias rf="rm -rf"
alias gall="git add \."
alias gpo="git push origin"
alias gpom="git push origin master"
alias gcmt="git commit -m"
alias aj="autojump"
alias jdk7="export JAVA_HOME=$JAVA7_HOME"
alias jdk8="export JAVA_HOME=$JAVA8_HOME"
```

use it at new terminal window

#### autojump

```shell
brew install autojump
vim ~/.zshrc
plugins=(git osx ruby autojump)
```



