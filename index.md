---
title: "Dan Ryan's tech blog"
---

## Welcome to My Blog

Here are my latest articles:

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <p>{{ post.excerpt }}</p>
      <small>Published on {{ post.date | date: "%B %-d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>