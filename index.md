---
layout: page
title: 大早网络
tagline: dazao
---
{% include JB/setup %}
<div class="row">
<div class="col-md-9">
这是杭州大早网络科技有限公司&&杭州只有一佳网络科技有限公司所维护的blog，主要记录一些计算机、互联网相关的内容。
 {% for post in site.posts limit:3 %}
  <div class="post">
      <h3 class="title"><a href="{{ post.url }}">{{ post.title }}</a>
	<span class="date">{{ post.date | date_to_string }}</span>
	</h3>
	{{ post.content }}
    	<div class="more">
         <a href="{{ post.url }}"  class="btn btn-sm btn-default center-block">Comments</a>
    	</div>
  </div>
  {% endfor %}

<center><a href='archive.html'>~~ Browse Archive ~~</a></center>

</div>

<div class="col-md-3">
<h3>Search</h3>
<form  action="http://www.google.com/search" method="get"><input type="hidden" name="sitesearch" value="hangzhoudazao.github.io" /> <input type="hidden" name="hl" value="zh-CN" /><input type="hidden" name="ie" value="UTF-8" />
<div class="row">
<div class="col-md-8">
<input id="query"  onfocus="if( this.value=='search the blog in google') {this.value='' };" type="text" name="q" value="search the blog in google" class="form-control"/>
</div>
<div class="col-md-2">
<button type="submit"  class="btn btn-sm btn-default">go~~</button>
</div>
</div>
</form>
<h3>Subscribe</h3>
<a href="rss.xml"><img src="img/feed.png" height="32"  /></a>

<h3>New Blogs</h3>
<ul class="posts">
  {% for post in site.posts limit:20 %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<h3>Links</h3>
<ul>
<li>
<span><a href="http://www.cplusplus.com">www.cplusplus.com</a></span>
</li>
</ul>
</div>
</div>
