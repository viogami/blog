---
title: 【开发问题】linux启动docker镜像成功，而win启动该镜像mysql报错
subtitle: 
slug: docker-mysql-error
tags: 
  - docker
  - mysql
  - linux
  - windows
  - 开发问题
summary: 
date: 2024-04-27
authors:
  - viogami: author.png
---

- dockers报错：`Starting MySQL database server mysqld ...failed`
- 进入容器内部，查看mysql错误日志：`cat /var/log/mysql/error.log`
- mysql具体报错：`Different lower_case_table_names settings for server ('2') and data dictionary ('0').`

<!--more-->

**出现该问题的重要前提是：**

- docker启动mysql并不是根据官方mysql镜像，而是在一个基础php镜像上安装的mysql服务。
- 我的mysql数据通过docker的卷挂载在外部，通过一个mysql文件夹管理。

当我把linux上可以运行的镜像和mysql外部卷都发送给win机器运行，由于**mysql数据是外部挂载的**，所以会出现同一个镜像，换了操作系统却无法运行的问题。本质上是mysql在linux和win上的环境变量`lower_case_table_names`默认值不同导致的。由于数据库数据在linux上初始化，该值已经为0，而在win上需要为2，所以报错。

> Docker镜像确实是操作系统无关的，但是MySQL的lower_case_table_names设置是与操作系统有关的。这是因为lower_case_table_names设置决定了MySQL如何存储和比较表名和数据库名。
>
> 在Windows和Mac系统中，文件系统通常不区分大小写，因此MySQL默认将lower_case_table_names设置为2。在这种设置下，MySQL将表名存储为小写，并且比较时不区分大小写。
>
> 在Linux系统中，文件系统通常区分大小写，因此MySQL默认将lower_case_table_names设置为0。在这种设置下，MySQL将表名存储为给定的大小写，并且比较时区分大小写。
>
> 当你在一个系统（例如Windows）上创建MySQL数据，然后尝试在另一个系统（例如Linux）上使用这些数据时，可能会出现这个错误。这是因为数据字典（存储了表名和数据库名的元数据）的lower_case_table_names设置与服务器的设置不一致。

## 解决方案

解决个蛋，一般都是放在linux上运行的，mysql在哪初始化就在哪部署。将镜像和外部卷都放在linux机器上部署就好了。win上报错要启动mysql前修改那个环境变量的值。

或者有一种麻烦的思路，就是将其mysql服务从一个容器中剥离出来，新开一个容器启动，这样方便设置环境变量。如果要修改一个非mysql基础镜像的mysql环境变量还是比较繁琐的。新开一个容器，用docker-compose启动即可。

<script src="https://giscus.app/client.js"
        data-repo="viogami/blog"
        data-repo-id="R_kgDOORWDyA"
        data-category="Announcements"
        data-category-id="DIC_kwDOORWDyM4Conxc"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
