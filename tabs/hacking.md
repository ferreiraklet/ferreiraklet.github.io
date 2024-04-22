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



<div id="post-list">

{% for post in site.posts %}

  <div class="post-preview p-3 mb-2 bg-light rounded floating-box" onclick="javascript:location.href='{{ post.url | relative_url }}'" style="background-color: rgba(65, 65, 65, 0.3) !important;">
    <div class="d-flex justify-content-between pr-xl-2">
    <article class="entry-header">
    <header class="entry-header">

      <h2 class="entry-hint-parent">{{ post.title }}</h2>
      </header>
      <footer class="entry-footer">Paper</footer>
      <a class="entry-link" href="{{ post.url | relative_url }}"></a>
      </article>
      {% if post.pin == true %}
        <i class="fas fa-thumbtack fa-fw text-muted mt-1 ml-2 mt-xl-2" data-toggle="tooltip" data-placement="left"
        title="Pinned"></i>
      {% endif %}
    </div>
    <br>
    <div class="post-content">
      <div class="post-img-cnt" style="text-align: center">
        <a href="{{ post.url | relative_url }}">
          <img src="{{post.image}}" alt="{{post.title}} image" class="post-img rounded" width="500" align="center" />
          <br>
        </a>
      </div>
      <div class="floating-box">
        {% include no-linenos.html content=post.content %}
        {{ content | markdownify | strip_html | truncate: 200 }}
      </div>
    </div>

    <div class="post-meta text-muted">
      <!-- posted date -->
      <i class="far fa-clock fa-fw"></i>
      {% include timeago.html date=post.date tooltip=true %}

      <!-- page views -->
      {% if site.google_analytics.pv.enabled %}
      <i class="far fa-eye fa-fw"></i>
      <span id="pv_{{-post.title-}}" class="pageviews">
        <i class="fas fa-spinner fa-spin fa-fw"></i>
      </span>
      {% endif %}
    </div>
  </div> <!-- .post-review -->

{% endfor %}

</div> <!-- #post-list -->

{% if paginator.total_pages > 0 %}
  {% include post-paginator.html %}
{% endif %}

