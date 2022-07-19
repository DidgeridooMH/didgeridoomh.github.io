Computer, art, and video game enthusiast.

<ul>
  {% for post in site.posts %}
      <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
      <div style="color: #a0a0a0"><i>Written on {{ post.published_at | date: "%B %e, %Y" }}</i></div>
      {{ post.excerpt }}
  {% endfor %}
</ul>
