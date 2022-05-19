---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---


# Posts

  {% for post in site.posts %}
  * <a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date|date:"%Y-%m-%d" }}{% endfor %}
