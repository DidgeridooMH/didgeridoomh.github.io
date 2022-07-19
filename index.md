Computer, art, and video game enthusiast.

<ul>
  {% for post in site.posts %}
    <li>
      <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
      <div style="font-color: #346eeb;font-weight: bold">{{ post.excerpt }}</div>
    </li>
  {% endfor %}
</ul>
