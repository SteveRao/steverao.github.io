---
layout:     post
title:      "IDEA编辑器配置"
subtitle:   "IDEA配置"
date:       2019-09-30
author:     "ZihaoRao"
catalog: true
header-img: "img/in-post/bg/night-sky.jpg"
tags: 开发工具
      配置环境
---





### 总体概述
---
> This article is written in Chinese. If necessary, please consider using [Google Translate](http://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https://steverao.github.io/2019/09/30/IDEA-settings/)
>
> 作为Java研发，目前业界最主流的编辑器IDEA确实为我们工作提供了极大便利。但要定制出满足个人使用的最佳开发工具一般还是要花费一定时间完成一定的配置与插件安装。本文主要介绍IDEA中一些对提升开发效率十分有益的插件或配置流程。
>
> **本文遵循[CC-BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)开源分享协议，转载文章内容请注明出处。**                                                                                                           



### 配置
---
#### 新项目配置

&emsp;&emsp;由于IDEA每次创建新项目时，在一些系统配置方面都会重新选择默认模式，所以为了避免每次开启新项目时因某些配置不当而导致出问题，可以一次性将个人的默认配置项在`Other Settings`中配置好。

&emsp;&emsp;打开`File > Other Settings` 可进行常见的Maven仓库信息，运行时服务器模版等信息的配置。



#### Mybatis Plugin

&emsp;&emsp;当需要使用Mybatis框架时，一般不能通过点击**接口方法定义**跳转到配置文件中对应**接口实现**，从而当需要定位问题或修改相关SQL时都多有不便。但在安装了Mybatis Plugin这款插件后就可以完美的解决该问题。

&emsp;&emsp;插件地址：[Mybatis Plugin](https://plugins.jetbrains.com/plugin/7293-mybatis-plugin/)



#### Mybatis Generation

&emsp;&emsp;当使用Mybatis进行持久层管理时，为每一张数据库表手动生成对应的Java Bean，接口定义和对应的xml文件非常麻烦和费时，当拥有了Mybatis Generation后仅需选择对应的数据库就可以自动的完成上述相关文件的生成，极大的提升新项目初始化效率。

&emsp;&emsp;由于该工具暂时还没发现合适所有人需求的版本，所以也无法推荐，有需要的可以去网上找适合自己的插件形式即可。



#### 其他

&emsp;&emsp;最后，在网上找到一篇该主题写得还比较详细的文章，因此便不必累述，剩于的配置或插件可参考下文，也在此对作者表示感谢。

&emsp;&emsp;[其他配置或插件](https://blog.mythsman.com/post/5d29f247cc343d1901c61d11/)


### 参考资料
---
- [IntelliJ常用配置备忘](https://blog.mythsman.com/post/5d29f247cc343d1901c61d11/)