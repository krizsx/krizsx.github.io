---
title: hugo使用shortcode引入音乐
date: 2018-07-08T09:06:00+08:00
lastmod: 2018-07-08T09:06:00+08:00
cover: "/img/2018/07/music.jpg"
draft: false
categories: ["blog"]
tags: ["music"]
---

{{< music "/music/2018/07/wakeup.m4a" "wakeup-倪安东" >}}

最近在找歌,然后就突然想到,hugo 中可不可以引入音乐播放器?于是去网上搜索了一下,发现并没有现成的

于是去看了看官网,发现了关于[shortcode](https://gohugo.io/content-management/shortcodes/#readout)的介绍,就想到了用这个来引入音乐

<!--more-->

# shortcode 介绍

可以在 markdown 中插入一段自定义的 html,目录应该是在 layouts 的 shortcodes 下

# 创建 music.html

在项目根目录创建 layouts 和 shortcodes 目录,然后创建 music.html 文件

# 编辑 music.html

```html
<h2>{{.Get 1}}</h2>
<audio controls autoplay loop preload="none" src="{{.Get 0}}">
  <p>Your browser does not support the <code>audio</code> element.</p>
</audio>
```

这里用到的 "{{ .Get 0}}"是 hugo 中 shortcode 的语法,在 markdown 中传递参数给 html
第一个参数传递歌曲的位置,第二个参数传递歌名

# 上传歌曲

我把音乐放到了这个目录

`/music/2018/07/wakeup.m4a`

# 编辑 markdown 文件,引入 shortcode

像下面这样,在 markdown 任何地方引入都可以,然后传递参数

```html
<!-- 两个大括号之间有个空格,否则这个也会被识别为shortcode -->
{ {< music "/music/2018/07/wakeup.m4a" "wakeup-倪安东" >} }
```

完成!
