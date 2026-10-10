---
layout: default
title: Home
---

# Welcome to my Security Writeups

Here is a list of my latest research and posts:

<ul>
  {% for post in site.posts %}
    <li>
      {{ post.date | date: "%Y-%m-%d" }} — <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
