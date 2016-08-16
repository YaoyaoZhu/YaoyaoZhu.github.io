---
layout: post
title: The manual of building a static blog website like this.
categories: [blog]
tags: [Efficiency]
description: 
---



# The manual of building a static blog.

1. ## Get the theme repository and rename.

   ```
    git clone https://github.com/Azeril/azeril.github.io.git
    mv azeril.github.io AlexanderWangsgithub.github.io
   ```

   ​

2. ## Edit your personal profile.

   ```
    edit _config.yml
    git add .
    git commit -m"init config"
   ```

   ​

3. ## Remote the local repository to your repo on github.

    ```
   git remote set-url origin https://github.com/AlexanderWangsgithub/AlexanderWangsgithub.github.io.git
    ```

   ​

4. ## Push changes to github.

   ```
    git push -u origin master
   ```

   ​


