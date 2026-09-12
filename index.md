---
layout: default
---

<div class="home-layout">

  <aside class="home-banner">
    <img
      src="{{ '/assets/ms-kernelkar-banner.png' | relative_url }}"
      alt="Ms. Kernelkar"
    >
  </aside>

  <div class="home-content">

    {% for post in site.posts %}
      <article class="post-preview">

        <h2 class="post-title">
          <a href="{{ post.url | relative_url }}">
            {{ post.title }}
          </a>
        </h2>

        <p class="post-date">
          {{ post.date | date: "%B %-d, %Y" }}
        </p>

        <p class="post-excerpt">
          {{ post.excerpt | strip_html | truncatewords: 45 }}
        </p>

        <a class="read-more" href="{{ post.url | relative_url }}">
          Read more →
        </a>

      </article>
    {% endfor %}

  </div>

</div>