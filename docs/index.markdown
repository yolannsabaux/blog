---
layout: default
---

<div id="home">
  <section id="archive">
    {% for post in site.posts %}
      {% unless post.next %}
        <ul class="this">
      {% else %}
        {% capture month %}{{ post.date | date: '%B %Y' }}{% endcapture %}
        {% capture nmonth %}{{ post.next.date | date: '%B %Y' }}{% endcapture %}
        {% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
        {% capture nyear %}{{ post.next.date | date: '%Y' }}{% endcapture %}
        {% if year != nyear %}
        </ul>
        <h2 style="text-align:left;">{{ post.date | date: '%Y' }}</h2>
        <ul class="past">
        {% endif %}
        {% if month != nmonth %}
        <h3 style="text-align:left;">{{ post.date | date: '%B %Y' }}</h3>
        {% endif %}
      {% endunless %}
      {% unless post.is_part %}
      <p>
        <b><a href="{{ site.baseurl }}{{ post.url }}">
          {% if post.title and post.title != "" %}
            {{ post.title }}
          {% else %}
            {{ post.excerpt | strip_html }}
          {% endif %}
        </a></b> - {% if post.date and post.date != "" %}{{ post.date | date: "%e %B %Y" }}{% endif %}
      </p>
      {% if post.has_parts %}
        <ul>
          {% for part in post.parts %}
            <li><a href="{{ site.baseurl }}{{ post.url }}{{ part.number | slugify }}">{{ post.title }} (Part {{ part.number }} )</a></li>
          {% endfor %}
        </ul>
      {% endif %}
     {% endunless %}
    {% endfor %}
    </ul>
  </section>
</div>
