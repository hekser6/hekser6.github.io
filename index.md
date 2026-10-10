---
layout: default
title: Home
---
<img src="profile.jpeg" alt="Hamza Adaoud" width="150" style="border-radius: 50%;">

# Hamza Adaoud - Security Research & Writeups
Cybersecurity researcher documenting vulnerability assessments, CTF walkthroughs, and security tooling.

# Welcome to my Security Writeups

Here is a list of my latest research and posts:

<ul>
  {% for post in site.posts %}
    <li>
      {{ post.date | date: "%Y-%m-%d" }} — <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
