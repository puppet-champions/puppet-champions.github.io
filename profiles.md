---
layout: default
tab: Profiles
title: Profiles of our Champions
tagline: It's our community that makes Puppet great.
weight: 3
hero:
    size: small
    image: profiles.jpg
---

# Extraordinary Puppeteers

<ul class="puppeteers">
  {% for profile in site.puppeteers %}
    <li>
      <a href="{{ profile.url }}">
        <img src="{{ profile.image }}" />
        <div class="name">{{ profile.name }}</div>
      </a>
      <p class="bio">{{ profile.bio }}</p>
    </li>
  {% endfor %}
</ul>

------

# Puppet Champions

<ul class="champions">
  {% for profile in site.champions %}
    <li>
      <a href="{{ profile.url }}">
        <img src="{{ profile.image }}" />
        <div class="name">{{ profile.name }}</div>
      </a>
    </li>
  {% endfor %}
</ul>
