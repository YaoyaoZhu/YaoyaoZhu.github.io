---
layout: post
title: GitLab & GitHub同时存SSH Key
categories: [blog]
tags: [Efficiency]
description: 
---

# GitLab & GitHub同时存SSH Key

## Add SSHKey(for github)

### 1. Generate a new SSHKey

```ssh-keygen -t rsa -b 4096 -C "wg1033755123@gmail.com"```

### 2. Check ssh agent

```eval "$(ssh-agent -s)"```

### 3. Add SSH key to ssh-agent 

```ssh-add ~/.ssh/id_rsa_github```

### 4. Add SSHKey to github accout

```pbcopy < ~/.ssh/id_rsa_github.pub```

github link: https://github.com/settings/ssh

## Manage SSHKey for different git

**vi  ~/.ssh/config**



    Host *github.com*
    	User git
    	IdentityFile ~/.ssh/id_rsa_github
    	
    Host *git.elenet.me*
    	User git
    	IdentityFile ~/.ssh/id_rsa
