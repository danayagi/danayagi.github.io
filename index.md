---
layout: default
title: Home
---

<div class="home">
  <h1 class="page-heading">Posts</h1>

  <ul class="post-list">
    {%- for post in site.posts -%}
      {%- unless post.url contains '/en/' -%}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
      </li>
      {%- endunless -%}
    {%- endfor -%}
  </ul>

</div>
