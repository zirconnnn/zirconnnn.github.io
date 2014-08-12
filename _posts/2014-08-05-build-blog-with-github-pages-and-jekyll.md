---
layout: post
title: "使用GitHub Pages搭建博客"
modified: 2014-08-05 23:44:32 +0800
tags: [github,jekyll,tool,tutorial,disqus]
image:
  feature: abstract-1.jpg
  credit: 
  creditlink: 
comments: true
share: true
---


博客搭了好久了，前几天重新换了个响应式布局的主题，还挺清爽简洁的，顺便记录下搭建过程吧。

## 1. 域名
前段日子看到[Godaddy][1] `.me`域名做活动就心血来潮买了个域名，荒了一段时间，心想还是搭个blog吧，最后还是选择搭在[GitHub Pages][2]，折腾就折腾一些吧。

首先呢，得把NS换到[DNSPOD][3], 可以参考[官方帮助][4]。

## 2. GitHub Pages
不熟悉git和github？可以戳这篇[史上最浅显易懂的Git教程][5]。

要在[GitHub Pages][2]搭建blog需要选择**User or organization site**，需要注意你的blog仓库名必须和你的**github username**相同，然后follow页面上剩余的步骤就算基本搭建完成，浏览器中输入`username.github.io`就可以看到效果。

### 绑定域名
下面是绑定域名到刚才搭建的pages，官方指导在[这里][6]。
简单说下步骤：

- 在repository下新建一个`CNAME`文件，里面写入你的域名，如`example.com`
- 到DNSPOD你域名的配置页面，如果是顶级域名`example.com`则配置两个`A`记录分别指向：

> 192.30.252.153

> 192.30.252.154

- 配置顶级域名的话同时还要给`www.example.com`子域名配置一个`CNAME`记录，指向`username.github.io` ,这样别人输入`www.example.com`，github服务器会301重定向到`example.com`。

- 如果你是单个子域名，如`blog.example.com`只需配置一个`CNAME`记录即可，github推荐使用子域名，一来如果github服务器ip变了对于顶级域名你得去修改它的`A`记录，二来可以更有效抵御DoS和利用CDN加速。
- 配置完成你可以dig一下查看解析情况

## 3. Jekyll
[Jekyll][7] 是一个静态页面生成模板引擎，GitHub Pages内置了对Jekyll的支持，所以你只要建立符合Jekyll规范的目录文件就可以被渲染成一个静态站点。

### 环境准备
先来搭建一下Jekyll本地环境，Mac自带了ruby，但是要装一下[RubyGems][8]，然后更改一下gem源：
{% highlight sh %}
gem sources --remove https://rubygems.org/
gem sources -a http://ruby.taobao.org/
gem sources -l
sudo gem update  —-system
{% endhighlight %}
一开始我是通过`gem install jekyll`单独安装jekyll等所需要的库，但是现在有更简单更易于管理的方式，官方指导在[这里][9]，首先需要安装一下bundler：
{% highlight sh %}
gem install bundler
{% endhighlight %}
[Bundler][10]通过Gemfile和Gemfile.lock来管理你整个project所依赖的RubyGems，你可以把需要的gems写到Gemfile中，然后执行`bundle install`，就会自动安装这些gems以及所依赖的其他库，并将所有的依赖信息保存到Gemfile.lock。别人可以通过copy你的Gemfile和Gemfile.lock来保持其环境的一致性。Gemfile看起来像这样：

> source 'http://ruby.taobao.org/'

> gem 'github-pages'

> gem 'coderay'

而[the GitHub Pages Gem][11]这个项目将所有GitHub Pages所依赖的RubyGems整合成一个gem：`github-pages`，所以你只要在Gemfile中加上`gem 'github-pages'`，就可以让你的本地环境保持和GitHub Pages[环境][20]一致，如果Jekyll等版本有更新，只需要简单的跑一下`bundle update github-pages`即可。

### 熟悉Jekyll
熟悉Jekyll最好就是clone一个用Jekyll搭建的站点到本地，[这儿][12]有很多，接着对照Jekyll的[官方文档][13]进行学习，列一下几个比较重要的点：

- 目录结构
- _config.yml，整个project的Jekyll配置文件
- [YAML][14]头信息，Jekyll会特别处理带有这种头信息的文件
- [liquid][15]模板语言，Jekyll对其进行了一些扩展，如使用[Pygments][18]语法高亮的tags写法：{% raw %}`{% highlight java %}{% endhighlight %}`{% endraw %}

### Markdown
Jekyll支持使用Markdown撰写blog，Markdown语法相当简单，若不熟悉，[这儿][16]有快速入门，Jekyll 2.0之后使用[Kramdown][17]作为默认的Markdown解析器，其他还有像[Discount][19]等，这些解析器都对标准Markdown有一些扩展与改进。Mac下较好的Markdown编辑器推荐[mou][23]，另外这个[在线的编辑器][24]感觉也不错。

## 4. Disqus
静态blog没有数据库，无法自己增加评论功能，好在有第三方的选择，如[Disqus][21]，[多说][22]，个人更喜欢Disqus的整体风格，虽然国内速度较慢。配置也很简单，需要注意几个方面：

- setup的时候选择`Universal Code`，然后它会提供给你两段code，一段是`load Disqus`，一段是`display comment count`，你可以将这两段写到一个html中然后放到`_include`目录下，在需要加载Disqus的页面中通过{% raw %}`{% include xxx.html %}`{% endraw %}方式来引用，当然，要加载Disqus的页面还需要你在合适的位置加上`<div id="disqus_thread"></div>`，这个id就代表了Disqus的放置位置。
- 对于显示评论数具体可参考[这里][25]，显示评论数的样式可以在Disqus的`settings->general->Comment Count Link`中配置，如我的配置：

>  ![count](/images/postimgs/disqus_count.png)

- `settings->advanced->Trusted Domains`该设置最好填上你的域名，它代表只有该域名及其子域名才能加载你的Disqus，置空则代表任何域名。

## 5. 其他选择
Jekyll的theme有很多选择，比如[这里][26]还有[这里][27]，你可以直接clone一个进行修改，除了Jekyll还可以选择[Octopres][28]，[hexo][29]，[Ruhoh][30]等，GitHub上还有其他一些类似的静态blog系统。

## 6. TODO
- 同时托管到[gitcafe][31]
- 加个toc
- Jekyll 2.0新特性
- 保持热情写blog吧

[1]: http://www.godaddy.com/
[2]: https://pages.github.com/
[3]: https://www.dnspod.cn/
[4]: https://support.dnspod.cn/Kb/showarticle/tsid/42/
[5]: http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000
[6]: https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages
[7]: http://jekyllrb.com/
[8]: http://rubygems.org/pages/download/
[9]: https://help.github.com/articles/using-jekyll-with-pages
[10]: http://bundler.io/
[11]: https://github.com/github/pages-gem
[12]: https://github.com/jekyll/jekyll/wiki/Sites
[13]: http://jekyllrb.com/docs/home/
[14]: http://yaml.org/
[15]: http://docs.shopify.com/themes/liquid-documentation/basics
[16]: http://wowubuntu.com/markdown/basic.html
[17]: http://kramdown.gettalong.org/index.html
[18]: http://pygments.org/
[19]: http://dafoster.net/projects/rdiscount/
[20]: https://pages.github.com/versions/
[21]: https://disqus.com/
[22]: http://duoshuo.com/
[23]: http://mouapp.com/
[24]: https://www.zybuluo.com/mdeditor
[25]: https://help.disqus.com/customer/portal/articles/565624
[26]: http://jekyllthemes.org/
[27]: http://www.zhanxin.info/themes.html
[28]: http://octopress.org/
[29]: http://hexo.io/
[30]: http://ruhoh.com/
[31]: https://gitcafe.com/
