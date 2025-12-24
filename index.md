---
layout: default
title: Salvatore's Blog
---

<section class="blog-list">
  {% for post in site.posts %}
    <article style="margin-bottom: 40px; border-bottom: 1px solid #eee; padding-bottom: 20px;">
      <h2 style="margin-bottom: 5px;">
        <a href="{{ post.url | relative_url }}" style="text-decoration: none; color: #159957;">{{ post.post_title }}</a>
      </h2>
      <p style="color: #606c71; font-size: 0.9em; margin-top: 0;">
        Pubblicato il {{ post.date | date: "%d/%m/%Y" }}
      </p>
      <div class="excerpt">
        {{ post.excerpt | strip_html | truncatewords: 30 }}
      </div>
      <a href="{{ post.url | relative_url }}" style="font-weight: bold; font-size: 0.9em;">Leggi tutto →</a>
    </article>
  {% endfor %}
</section>