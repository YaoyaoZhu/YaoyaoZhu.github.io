---
layout: post
title: Mac下mysql忘记密码
categories: [blog]
tags: [Database]
description: 
---

Mac下mysql忘记密码

1. Stop mysql in system reference.

2. `sudo ./usr/local/mysql-5.7.13-osx10.11-x86_64/bin/mysqld_safe --skip-grant-tables`

3. Use command `mysql` login in.

4. Update password of root to null.

   ```
   update mysql.user set authentication_string="" where user="root";
   exit;
   ```
5. Set new password.
`/usr/local/mysql-5.7.13-osx10.11-x86_64/bin/mysqladmin -u root -p password wanggang`