---
layout: default
title: Home
---

<meta name="google-site-verification" content="JlqtL3H6cdgMdqRdWX6q59UAVYyZBTci-s5k5fICA-8" />

<img src="profile.jpeg" alt="Hamza Adaoud" width="150" style="border-radius: 50%;">

# Hamza Adaoud - Security Research & Writeups
Cybersecurity researcher documenting vulnerability assessments, CTF walkthroughs, and security tooling.

[About Me](/about/)

# Welcome to my Security Writeups

Here is a list of my latest research and posts:

<ul>
  {% for post in site.posts %}
    <li>
      {{ post.date | date: "%Y-%m-%d" }} — <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
