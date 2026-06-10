---
title: Posts | Articles | Papers
type: articles
# The Archives of posts and articles.
# v2.0
# https://github.com/cotes2020/jekyll-theme-chirpy
# © 2017-2019 Cotes Chung
# MIT License
---

{% assign pinned = site.posts | where_exp: "item", "item.pin == true"  %}
{% assign default = site.posts | where_exp: "item", "item.pin != true"  %}
{% assign posts = "" | split: "" %}

<!-- Get pinned posts -->

{% assign offset = paginator.page | minus: 1 | times: paginator.per_page %}
{% assign pinned_num = pinned.size | minus: offset %}

{% if pinned_num > 0 %}
  {% for i in (offset..pinned.size) limit: pinned_num %}
    {% assign posts = posts | push: pinned[i] %}
  {% endfor %}
{% else %}
  {% assign pinned_num = 0 %}
{% endif %}


<!-- Get default posts -->

{% assign default_beg = offset | minus: pinned.size %}

{% if default_beg < 0 %}
  {% assign default_beg = 0 %}
{% endif %}

{% assign default_num = paginator.posts | size | minus: pinned_num  %}
{% assign default_end = default_beg | plus: default_num | minus: 1 %}

{% if default_num > 0 %}
  {% for i in (default_beg..default_end) %}
    {% assign posts = posts | push: default[i] %}
  {% endfor %}
{% endif %}



<div class="hacking-section-header">
  <h1><i class="fas fa-stream mr-2" style="color:#f07020;font-size:0.95rem;"></i>Articles &amp; Papers</h1>
  <div class="hacking-header-line"></div>
  <span class="post-count-badge">{{ site.posts | size }} posts</span>
</div>

<div class="post-card-grid">
{% for post in site.posts %}
  <a class="post-card" href="{{ post.url | relative_url }}">
    {% if post.image %}
    <div class="post-card-img-wrap">
      <img src="{{ post.image }}" alt="{{ post.title }}">
      <div class="post-card-img-overlay"></div>
    </div>
    {% else %}
    <div class="post-card-img-wrap" style="background: linear-gradient(135deg, rgba(200,80,16,0.15), rgba(25,25,32,1)); height:80px;">
      <div class="post-card-img-overlay"></div>
    </div>
    {% endif %}
    <div class="post-card-content {% unless post.image %}post-card-no-image{% endunless %}">
      <div class="post-card-cat">
        {% if post.categories.size > 0 %}{{ post.categories | first }}{% else %}Research{% endif %}
        {% if post.pin == true %}&nbsp;<i class="fas fa-thumbtack" style="font-size:0.6rem;opacity:0.6;"></i>{% endif %}
      </div>
      <div class="post-card-title">{{ post.title }}</div>
      <div class="post-card-excerpt">{{ post.content | strip_html | truncate: 110 }}</div>
    </div>
    <div class="post-card-footer">
      <span class="post-card-date">
        <i class="far fa-clock"></i>
        {% include timeago.html date=post.date %}
      </span>
      <span class="post-card-arrow"><i class="fas fa-arrow-right"></i></span>
    </div>
  </a>
{% endfor %}
</div>

