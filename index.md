---
layout: page
title: Home
tagline: the metric gear always turns
order: 1
---
{% include JB/setup %}

<div class="intro">
Hello, my name is Paul Deardorff.  I'm a business student at UC Berkeley interested in tech and entrepreneurship.  I've done some <a href="pages/projects.html">projects</a> and helped other teams on <a href="pages/projects.html#consulting">their projects</a>.  
</div>

## Posts 
<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>




