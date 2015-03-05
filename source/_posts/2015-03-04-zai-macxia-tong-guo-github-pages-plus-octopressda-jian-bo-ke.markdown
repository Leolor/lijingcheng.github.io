---
layout: post
title: "在Mac下通过Github Pages + Octopress搭建博客"
date: 2015-03-04 14:51:02 +0800
comments: true
categories: 
---
---

{% img left http://7x00ed.com1.z0.glb.clouddn.com/github-octopress.png 230 250 "" "" %}
>**[Github Pages](https://pages.github.com) - *Websites for you and your projects.***   
可用来搭建静态网站，它提供免费的域名、空间、无限流量，并且在世界各地都有较好的访问速度。不过网站也会轻易被人Clone，如果在意的话可以付费给github，然后将相关版本库建成私有的。
<br/>  
>**[Octopress](http://octopress.org) - *A blogging framework for hackers.***  
作为一款开源的静态博客系统，可以用来为我们的静态网站提供所需的HTML页面。
<!-- more -->
<br/>
#准备工作
- 需要git环境并且已经将创建好的ssh keys添加到github帐户下。
```
cd ~/.ssh (检查本地有没有ssh keys)
ssh-keygen -t rsa -C "注册github时用的email" (创建ssh keys)
用记事本打开id_rsa.pub，复制里面的东西到github帐户下
```
- 在github上新建名为“yourname.github.io”的版本库，之后可通过“yourname.github.io”域名来访问，如果你有自己的域名，可通过配置Octopress的CNAME文件进行映射。  
- 需要有1.9.3及以上版本的ruby环境。(可通过ruby --version来查看版本，如需升级可用rvm或rbenv)
- 更改ruby的源为淘宝，可提高下载速度  
```
gem sources -a http://ruby.taobao.org/  (添加)
gem sources -r http://rubygems.org/  (删除)
gem sources -l  (检查一下)
``` 

<br/>

#Octopress环境搭建
- 安装  
```
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```
- 安装所需依赖  
```
sudo gem install bundler 
rbenv rehash 
bundle install
```
- 安装默认模板，也可以安装[第三方模板](https://github.com/shashankmehta/greyshade)
```
rake install
```
- 配置网站
```
通过"/octopress/_config.yml"文件配置，参考：http://octopress.org/docs/configuring
自己修改html和css，将twitter以及google相关的内容删掉，可以提高访问速度
```
- 关联github pages
```
rake setup_github_pages (然后输入 git@github.com:yourname/yourname.github.com.git)
```

<br/>

#写博客
- 新建博客，会在"\octopress\source\\_posts"目录下生成扩展名为"markdown"的文件，推荐使用["Mou"](http://25.io/mou/)编辑器编写markdown文件并尽量采用[Markdown语法](http://wowubuntu.com/markdown/)。
```
rake new_post["new_blog_title"]
```
- 将octopress生成的html等文件提交到master分支
```
rake generate (生成静态网站)
rake preview (通过http://localhost:4000预览，修改markdown文件后直接刷新即可)
rake deploy push git
```
- 将octopress的相关文件提交到source分支
```
git add .
git commit -m ‘your commit message’
git push origin source
```
<br/>

#添加评论系统
- 通过在多说网[新建站点](http://duoshuo.com/create-site)来获取网站的short_name  
- 在_config.yml中添加以下内容，并替换"your_short_name"
```
# duoshuo comments 
duoshuo_comments: true 
duoshuo_short_name: your_short_name
```
- 新建"source/_includes/post/duoshuo.html"并添加下面内容，还需要替换"your_short_name"
``` js
<div class="ds-thread"></div>
<script type="text/javascript">
var duoshuoQuery = {short_name:"your_short_name"};
(function() {
    var ds = document.createElement('script');
    ds.type = 'text/javascript';
    ds.async = true;
    ds.src = 'http://static.duoshuo.com/embed.js';
    ds.charset = 'UTF-8';
    (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(ds);
})();
</script>
```
- 在"source/_layouts/post.html"中嵌入刚才新建的duoshuo.html  
``` js
{% if site.duoshuo_short_name and site.duoshuo_comments == true and page.comments == true %}
<section>
    <h1>评论</h1>
    <div id="comments" aria-live="polite">{% include post/duoshuo.html %}</div>
</section>
{% endif %}
```