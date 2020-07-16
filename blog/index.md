---
layout: default
title: Under Development
---

<h1>{{ page.title }}</h1>
<p style="font-style: italic;">by Bj&oslash;rn Reese</p>

{% for post in site.posts %}
<article>
  {% for tag in post.tags %}
  <p><span class='marginnote'>{{ tag | capitalize }}</span></p>
  {% endfor %}
  <center class='postdate'>{{ post.date | date: "%d %B %Y" }}</center>
  <span class='postdateseparator'></span>
  <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
  <blockquote>
  {{ post.excerpt }}
  </blockquote>
</article>
{% endfor %}
