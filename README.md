# Btz's blog repository

## 这个博客的作用：

 这个博客是我用来记录生活和学习的、我会将自己所学的知识加以总结，形成一个可阅读的文档再放在此博客中。

水平有限、如有错误，可在issue或者评论中指出，本人感激不尽。

## 这个博客的一些参考实现

### 博客模板：

#### This blog template is forked from [Hux Blog](https://github.com/Huxpro/huxpro.github.io).

##### This is the boilerplate of [Hux Blog](https://github.com/Huxpro/huxpro.github.io), all documents is over there!

##### [View Boilerplate &rarr;](http://huangxuan.me/huxblog-boilerplate/)

### 博客的评论系统

模板中有自带的两套评论系统——disqus和多说，但是多说早已经停止维护，disqus使得无法科学上网的童鞋无法评论。

所以我找到了还算好看的gitment评论系统，效果见此 [Demo](https://imsun.github.io/gitment/)。
> gitment如何配置请看官方文档 [gitment](https://github.com/imsun/gitment)

中间遇到的比较大的坑是，老是会弹出 object ProgressEvent 然后就登录不进去。
>原因是因为源代码中需要访问gh-oauth.imsun.net去GET一些资源、但是此网站的证书过期了。
##### 解决办法是：
1. 手动进入gh-oauth.imsun.net，然后点高级 -> 继续前往gh-oauth.imsun.net，这样在调试的过程中就可以登录了。
2. 更换资源的url，将源码中的url改为如下链接：
```html
<link rel="stylesheet" href="https://jjeejj.github.io/css/gitment.css">
<script src="https://jjeejj.github.io/js/gitment.js"></script>
```
> 其他一些问题的解决可以参考[Gitment评论功能接入踩坑教程](https://www.jianshu.com/p/57afa4844aaa)

### 基于模板上做的一些修改

除了换了评论系统，还做了一些更改：
1. 由于Mini-about中的css文件和js文件都是从http开头的url中获取的、因此有一些浏览器会自动拦截从这些不安全的url加载的组件。**解决办法是将所有被拦截的url的开头都替换成https**，目前有如下几个url需要替换：
head.html中的：
```html
    <link href="http://cdn.staticfile.org/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet" type="text/css">
    footer.html中的
```
footer.html中的：
```javascript
async("http://cdn.bootcss.com/fastclick/1.0.6/fastclick.min.js", function(){
        var $nav = document.querySelector("nav");
        if($nav) FastClick.attach($nav);
    })
```