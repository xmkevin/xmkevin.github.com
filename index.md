---
layout: page
title: Hello World!
tagline: Supporting tagline
---
{% include JB/setup %}

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
      <P>{{ post.excerpt }}</P>
    </li>
  {% endfor %}
</ul>


