+++
title = "Netlify+Hexo实现持续部署最佳实践"
date = "2018-12-21 06:14:00"
url = "archives/114"
tags = ["Hexo"]
categories = ["杂项"]
+++

Hexo被称为是最佳的静态博客程序之一，然而其繁琐的环境搭建、构建，发布过程，让很多人望之却步，转而使用了传统的`WordPress`等博客程序，抛开维护成本不说，本着折腾一切的心态，最终研究出了一套完善的自动部署方案。

### Hexo特色 ###

 *  **超快速度:** Node.js 所带来的超快生成速度，让上百个页面在几秒内瞬间完成渲染。
 *  **支持 Markdown:** Hexo 支持 GitHub Flavored Markdown 的所有功能，甚至可以整合 Octopress 的大多数插件。
 *  **一键部署:** 只需一条指令即可部署到 GitHub Pages, Heroku 或其他网站。
 *  **丰富的插件:** Hexo 拥有强大的插件系统，安装插件可以让 Hexo 支持 Jade, CoffeeScript。

> 目前市面上还存在很多类似的静态网站生成器，详情请查看：[https://www.staticgen.com/][https_www.staticgen.com]

### 如何实现优雅发布 ###

就目前而言，Hexo发布的方式有以下几种：

 *  原始方式，即在本地搭建相关环境，编写md文件后，手动`hexo g`生成静态文件，然后通过`hexo deploy`发布到`Github Pages`；
 *  利用Github + Webhook来实现自动发布（这个需要一台自己服务器）详见：[http://www.swiftyper.com/2016/04/17/deploy-hexo-with-git-hook/][http_www.swiftyper.com_2016_04_17_deploy-hexo-with-git-hook]
 *  使用第三方的Hexo-Client、Hexo-Admin等程序，详见：[https://github.com/search?q=hexo-client][https_github.com_search_q_hexo-client]
 *  使用`Travis CI`持续部署Hexo，详见：[https://www.jianshu.com/p/5691815b81b6][https_www.jianshu.com_p_5691815b81b6]
 *  使用`Netlify`进行优雅地持续部署。详见：[https://www.netlify.com][https_www.netlify.com]

### 简要流程 ###

1.  使用`Github`登陆`Netlify`。
2.  使用`StaticGen`一键初始化Hexo仓库。
3.  将Hexo源码仓库Clone到本地，调整网站配置，编写文章。
4.  本地无需`Nodejs`、`NPM`、`Hexo`环境，修改完成后`Push到Github`，`Netlify`检测到仓库变更后实现`自动部署`。

在`Netlify`整个部署过程中, 你只需要提交代码, 其余的master部署预览(包括MR的预览), HTTPS证书, 静态资源的优化与CDN加速, 部署消息通知, 等等都不用再操心. 真的是太优雅了!

### 创建项目 ###

#### 在StaticGen上选择Hexo ####

![36aabd38-f45b-48db-9a00-58cbaa554611.png][]

#### 使用Github登陆 ####

![3d4a66e2-5a1a-47e0-930e-cc5d84bd3a90.png][]

#### 设置一个Github仓库名 ####

![d8175852-6d21-472c-9e89-8170e0261d75.png][]

### 轻点3步，轻松实现网站上线 ###

![906e5096-f4fb-42d7-b76b-2152ddf9fa34.png][]

#### 第一步：自动部署 ####

不用做任何设置, 每次master分支有更新代码, Netlify就会帮你自动部署代码. 图为部署记录

![9a942e08-dbbf-46c0-b402-2c24aa700820.png][]

实时看到部署的日志:

![99848415-3c09-483a-ae40-9f3acfa1476b.png][]

#### 第二步：自定义域名 ####

默认情况下，Netlify为我们分配了一个随机域名（可以自定义二级域名、独立域名）

![ee0bf0d2-8370-417f-af68-d20abc7a0ac3.png][]

#### 第三步：开启Https ####

自动生成`Let’s Encrypt`的证书, 也支持上传自己的证书，详见：[https://www.netlify.com/docs/ssl/][https_www.netlify.com_docs_ssl]

![4232829e-7d2f-42a2-9189-c3d5bff89ac5.png][]

### 其他：Netlify的优缺点 ###

优点：

 *  提供webhook的形式触发部署
 *  提供Html代码注入
 *  自动优化
 *  自动部署通知

缺点：不能检测到`git submodule`的变更

### 关于Markdown编辑器 ###

现在我们已经完成了Hexo的持续部署，将Hexo源码项目Clone到本地后，可以使用IDEA导入，IDEA内置的Markdown编辑器正好用来写文章，而IDEA内置的Git版本管理工具也不赖，哈哈，如此一来，书写博客就如同写代码一般，写完提交到Git即可。

此外、IDEA内置的Markdown编辑器不支持插入图片，我这里写了个轻量级的Markdown编辑器扩展程序, 支持粘贴图片文件然后上传到七牛云存储, 然后生成Markdown图片标记插入到文章中。

> 详见[https://gitee.com/wuwenze/markdown-support-qiniu][https_gitee.com_wuwenze_markdown-support-qiniu]

### 效果图 ###

![d8dfdcb0-fa4e-4f25-aa38-da90873a8678.png][]![f5becb7a-347b-4f34-b217-e02736236af4.png][]


[https_www.staticgen.com]: https://www.staticgen.com/
[http_www.swiftyper.com_2016_04_17_deploy-hexo-with-git-hook]: http://www.swiftyper.com/2016/04/17/deploy-hexo-with-git-hook/
[https_github.com_search_q_hexo-client]: https://github.com/search?q=hexo-client
[https_www.jianshu.com_p_5691815b81b6]: https://www.jianshu.com/p/5691815b81b6
[https_www.netlify.com]: https://www.netlify.com/
[36aabd38-f45b-48db-9a00-58cbaa554611.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/36aabd38-f45b-48db-9a00-58cbaa554611.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[3d4a66e2-5a1a-47e0-930e-cc5d84bd3a90.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/3d4a66e2-5a1a-47e0-930e-cc5d84bd3a90.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d8175852-6d21-472c-9e89-8170e0261d75.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/d8175852-6d21-472c-9e89-8170e0261d75.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[906e5096-f4fb-42d7-b76b-2152ddf9fa34.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/906e5096-f4fb-42d7-b76b-2152ddf9fa34.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[9a942e08-dbbf-46c0-b402-2c24aa700820.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/9a942e08-dbbf-46c0-b402-2c24aa700820.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[99848415-3c09-483a-ae40-9f3acfa1476b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/99848415-3c09-483a-ae40-9f3acfa1476b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ee0bf0d2-8370-417f-af68-d20abc7a0ac3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/ee0bf0d2-8370-417f-af68-d20abc7a0ac3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_www.netlify.com_docs_ssl]: https://www.netlify.com/docs/ssl/
[4232829e-7d2f-42a2-9189-c3d5bff89ac5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/4232829e-7d2f-42a2-9189-c3d5bff89ac5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_gitee.com_wuwenze_markdown-support-qiniu]: https://gitee.com/wuwenze/markdown-support-qiniu
[d8dfdcb0-fa4e-4f25-aa38-da90873a8678.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/d8dfdcb0-fa4e-4f25-aa38-da90873a8678.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[f5becb7a-347b-4f34-b217-e02736236af4.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181221/f5becb7a-347b-4f34-b217-e02736236af4.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg