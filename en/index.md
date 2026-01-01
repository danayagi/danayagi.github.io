---
layout: default
title: English Posts
permalink: /en/
---

<div class="home">
  <h1 class="page-heading">English Posts</h1>

  <ul class="post-list">
    {%- for post in site.posts -%}
      {%- if post.url contains '/en/' -%}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
      </li>
      {%- endif -%}
    {%- endfor -%}
  </ul>

</div>
