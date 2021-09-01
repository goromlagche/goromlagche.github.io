---
title:  "Blog"
layout: archive
permalink: /Blog/
author_profile: true
comments: true
---

Most of the posts are technical, written mainly for my own reference. Collected throughout my day to day work. Most of them are TILs, not long form blogs.

I'd be happy if any of you find them useful too.

<ul>
  {% for post in site.posts %}
    {% unless post.next %}
      <font color="#778899"><h2>{{ post.date | date: '%Y %b' }}</h2></font>
    {% else %}
      {% capture year %}{{ post.date | date: '%Y %b' }}{% endcapture %}
      {% capture nyear %}{{ post.next.date | date: '%Y %b' }}{% endcapture %}
      {% if year != nyear %}
        <font color="#778899"><h2>{{ post.date | date: '%Y %b' }}</h2></font>
      {% endif %}

    {% endunless %}
   {% include archive-single.html %}
  {% endfor %}
</ul>
