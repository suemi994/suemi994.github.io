---
layout: post
title: jekyll模板之Zedoc
category: tools
tags: jekyll
date: 2015-01-15
---

##前言

由于要为同学和自己开发的web框架搭一个website，所以想到用jekyll写一个模板，一方面是满足自己的需求，另一方面是为后来人提供方便。这篇博文将详细记录我的模板搭建过程。

首先，在这里要感谢@ yyx990803（尤小右）大神，里面大部分的布局都引用自他为vue.js搭的website，不过他提供的是一个hexo主题，我们给出了jekyll模板。

[模板请点这里](https://github.com/suemi994/Zedoc)
[live demo](http://zetajs.io)

##使用

###编写guide

~~~md
---
layout: guide
title: your title
---
~~~

将上面的加在文章开头，就可以编写guide，会自动在左侧生成目录栏，包含所有guide章节和本章节的结构（详情见效果展示），跳转将带有滑动加速特效。

###编写附加资料

~~~md
---
layout: gpost
title: your title
---
~~~

将附加材料加到blog/_posts文件夹下
将在左侧增加目录栏，包含本篇文章的所有结构

###编写其他

####API页
修改根目录下的api.md，所有h2和h3将在左侧栏生成目录。

####附加材料列表
修改blog/index.html，默认有分页功能，按时间顺序显示所有附加材料

####示例页
修改根目录下examples.md即可


##配置

~~~yml
# Site settings
title: Your awesome title
subtitle: Your awesome subtitle
email: your-email@domain.com
description: > # this means to ignore newlines until "baseurl:"
  Write an awesome description for your new site here. You can edit this
  line in _config.yml. It will appear in your document head meta (for
  Google search results) and in your feed.xml site description.
baseurl: "" # the subpath of your site, e.g. /blog/
url: "http://yourdomain.com" # the base hostname & protocol for your site
twitter_username: jekyllrb
github_username:  jekyll
permalink: /post/:year/:month/:day/:title.html
paginate: 1
paginate_path: "lib/page:num"

# Build settings
markdown: kramdown
collections:
  guide:
    output: true
    permalink: /guide/:path.html

~~~

##效果展示
- ![主页展示](/public/img/2015-01-15-home.png)


- ![guide展示](/public/img/2015-01-15-guide.png)
