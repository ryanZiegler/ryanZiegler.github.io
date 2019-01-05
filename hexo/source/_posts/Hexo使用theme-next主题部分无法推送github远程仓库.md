---
title: Hexo使用theme/next主题部分无法推送github远程仓库
date: 2019-01-05 15:08:01
tags:
---
## Hexo使用theme/next主题部分无法推送github远程仓库

#### 关键词: Hexo, next, github, git submodule, data files, next.yml 

#### 前置步骤

1. 你想要用hexo+github搭建自己的博客(master构建部署分支/hexo编辑分支)

   ~~这种方法你的博客在github上是透明的哦~~~

2. 使用`git clone`的方式下载了theme/next的库包

3. 在hexo分支上推送博客内容, 换台电脑`git clone`自己的博客仓库  

   ​

#### 产生的问题

​	在新电脑上`git clone`的仓库hexo分支中theme/next文件是空的, 在github上查看next文件发现为submodule形式. 也就是你如果想要在新电脑上编辑自己的博客, 那么你需要重新`git clone`官方next远程仓库.也就是说你此前next主题相关的主题配置`_config.yml`, 你已经丢失了!!!~~除非你去旧电脑上copy(手动滑稽.ipg~~

~~我不信就我一个人会有这个问题,所以我就去next的github的issue查了一下,果然有跟我一样问题的大兄弟,不过人家在2016年提出问题并得到解决了~~



#### 解决方案

此时分两种情况:

如果你已经对next主题进行自己的定制化修改, 那么你需要方案①;

如果你只是对next主题文件`_config.yml`进行了简单的修改, 那么你需要方案②;

1. fork + submodule
2. 使用Hexo data files配置主题 Theme configurations using Hexo data files

~~因为目前我只需求对主题简单配置,并且能够在公司or家里电脑能够同步配置,所以我使用的方案②,具体操作如下 很简单;~~

方案②:

通过这个Data Files，你可以将所有的主题配置放置在站点的 `source/_data/next.yml` 文件中。原先放置在 站点配置文件 中的选项可以迁移到新的位置，同时，主题配置文件可以不用做任何修改。若后续版本有配置相关的改动时，你仅需在 `next.yml` 中做相应调整即可

1. 请先确保你所使用的 Hexo 版本在 3 以上
2. 在站点的 `source/_data` 目录下新建 `next.yml` 文件（`_data`目录可能需要新建）
3. 迁移站点配置文件和主题配置文件中的配置到 `next.yml` 中
4. 使用 `--config source/_data/next.yml` 参数启动服务器, 生成或者部署。
     例如: `hexo clean --config source/_data/next.yml && hexo g --config source/_data/next.yml`。



#### 写在最后

这样你就可以愉快的开始自己的hexo博客之旅啦~

希望可以帮助到更多的人!



ps: 本篇文章参考了[官方文档](https://github.com/iissnan/hexo-theme-next/blob/master/README.cn.md)和[官方issue#932](https://github.com/iissnan/hexo-theme-next/issues/932)


