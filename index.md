Computer, art, and video game enthusiast.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="/github-pages-with-jekyll{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
