---
title: FavAni--从0开始的web开发日志
slug: "FavAni"
subtitle: 
summary:
date: 2023-09-30
cardimage: 
featureimage: 
caption: 
authors:
  - viogami: author.png
---
{{< figArray subfolder="`<subfoldername>`" figCaption="" numCols=2 >}}

## [演示DEMO地址](http://fa.viogami.tech/)

### 技术栈：gin+gorm+mysql+jwt+redis+grpc+pytorch+vue3+vite+tailwind

<!--more-->

## 前言

~~一方面为了完成毕设(毕设大概率是造前人的轮子了)，一方面也是~~毕设导师要求用WP了，那纯兴趣使然了。
这个项目的开发很是缓慢，因为今年大部分时间在考研，而且前后端技术栈有很多空白，如果类似bangumi这样的网站，数据库的知识也不能缺乏。我coding效率一般，好在有chatgpt的强力支持，bangmui的网站api是开放的，但我可能没精力实现全部功能。~~我的目标其实是一个纯粹的个人打分网站(去社交化，但提供社交软件导入)~~

随着对web开发的认知逐步完善，我决定大改需求，原来的目标推翻重做，造轮子实在比较蠢。

我是bangumi网站的用户，兴趣之余也想利用本人能力仿制一个功能类似的项目，添加个人认为有必要的其他功能，使用不同于bangumi站点的设计框架，设计一个可以给用户类似使用需求的web应用。

-----时间线总览-----

- 确认了需求和使用什么前后端的框架,今天从0搭建了vue+elementui框架 [2023/9/30]
- 使用vite+vue3+elements-plus开发前端，之前vue-cil太多是基于vue2的，要做就做新的
  [2023/10/10]
- 没时间搞，今天想了一下看，把需求推翻重做，不是以仿bangumi的web应用为目标，bangui有自己正在开发的react应用，多劳无益，还不如去fork。现在摸清楚了bangumi的api怎么调用，数据怎么get，post，做完一个完成的自助数据查询网站也不错了。没多余精力搞前端，u3d还在水深火热之中[2023/11/16]
- 决定使用vue3+go的前后端框架，完成了对用户和条目信息的get和分析。
  [2023/11/16]
- 花了两天，做了界面，用[weave][3]的界面做了一定的参考，可以想象，css是复杂精密的，界面制作是个细活，还是挺难上手的。完成了功能
- 查询用户，用户页面，用户页面进行用户收藏查询(一次显示8个，有下一页按钮)
- 导航，页脚，侧边栏设计
- 时间线，关于，以及重点的文档页面，通过调用github的api获取仓库的README.md文件
- 查询条目
  [2023/11/27]
- 实现了前后端基本交互，完善了登陆和注册功能，线上连接了mysql和redis
  [2024/1/4]
- 终极完善了登录注册功能，使用了中间件鉴权，jwt的方式，给两种登录方式各设置了不同的token。
  [2024/2]
- 实现了收藏的大部分功能，推荐算法的执行部分使用python编写，部署为一个微服务，通过**grpc**的方式调用。
  [2024/3]

bangumi开源项目，相关链接：
**[bangumi API][4]**
**[bangmui 用户授权机制][5]**

自助数据检索功能

- [X] 在login页面，输入用户名，返回用户相关信息
- [X] 输入相关条目，返回需要的条目信息

## 原型设计

参考bangumi各种辅助工具的UI逻辑，但是界面设计可以更现代化一点，可能会花费很多时间，会经济的考虑设计样式。
项目参考：[weave][6]

## 前端

个人作全栈开发，用vue这种较为轻量的前端开发框架。目前决定使用vue3+element-plus做前端。
框架搭建：下载node.js，配置环境。创建第一个vue单页应用。根据elementUI的官方步骤，搭建框架

---

今天发现之前是基于vue2的，用的是vue官方vue-cli脚手架，现在更推荐使用vite，仅支持vue3，遂改之。
elementui也更新为element-plus仅用于支持vue3。vite不用全局安装，较为轻量。有了以前摸索的基础，下载了路由组件和element-plus，迅速就搭建好了空项目。

花了一下午搞清楚了bangumi的API是怎么在vite中用axios的get和post请求调用的，记录了。输入用户名或者各种搜索条件可以获取到相应的数据了，下一步做一个数据爬取，然后完善功能吧。

---

前端基本首页做完了，现在就是功能逻辑了，用户的功能也大差不大了，项目准备build发布一次，后续开发一下检索的功能，并开发后端和数据库，进行数据处理。

基本完善了99%的页面设计。

## 后端

后端用golang编写，使用GIN的框架。
Restful API，使用gorm接入mysql，接入redis。

目前实现了一个基本的登陆和注册功能，并把用户数据存放到数据库。处理前端POST的数据进行登陆和注册验证。
使用jwt，构造鉴权中间件，分组路由实现。

数据库中目前只有一个用户表。同时新建了条目表和用户收藏表，用于建立两表之间的联系。
添加了tag表，gorm中用many2many的方式和条目表构建一个关系，同时用于对用户标签化，便于进行推荐算法。

使用grpc服务和python后端通信。由于神经网络算法的代码过于依赖python，我将其微服务化，可以使得该代码给多个项目使用。通过定义protoc文件，生成protoc文件，编写grpc客户端和服务端代码，算法后端已经成熟的运行了。

## 总结

vue基本官方文档就能解决90%的问题，构建后大小也很惊人， 只有几MB。
其实完成了这个，vue的使用倒没有熟练多少，更多是了解了很多前端技术栈的知识，刚起步的时候，连vue和vite是什么都分不清，还一直以为我用的是vite框架呢🤣(导致项目名字也叫digbgm_**vite**了，笑)

走完从开发到生产环境部署的全流程，实践一遍可以学到很多以为简单但并没那么简单的事情。比如api的调用跨域(需要nginx使用反代理，把指定的地址反代到跨域的地址)，刷新404问题，都是典中典，新手是这样的

前端最大的问题，我觉得是配css，这是最折磨的，特别我还有一点强迫，还好我也不是主营前端，css凑合吧…
现在我觉得说不定用tailwind会好一点。。。😥

---

后端的搭建基于我对go的逐渐熟练以后，在有了go的相关基础和gin的一定使用经验后，我快速搭建了该项目的后端，并进行了线上部署。go打包后为linux可执行文件，上线是很快的，一键部署。在使用配置文件启动项目后，也更加了灵活。

  [3]: https://github.com/qingwave/weave
  [4]: https://bangumi.github.io/api/#/
  [5]: https://github.com/bangumi/api/blob/master/docs-raw/How-to-Auth.md
  [6]: https://github.com/qingwave/weave

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