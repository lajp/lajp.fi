---
permalink: /blog
title: blog
layout: default.liquid
---
## Welcome to the blog of Luukas Pörtfors

Below is a listing of articles

{% for post in collections.posts.pages %}
<div>
<h3><a href="/{{ post.permalink }}">{{ post.title }}</a></h3>

{{ post.description }}

Published: {{ post.published_date | date: "%Y-%m-%d" }}

Tags: {{ post.tags | join: ", " }}
</div>
{% endfor %}
