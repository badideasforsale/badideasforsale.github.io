---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<h1>Latest Posts</h1>

<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>

<h1>About Me</h1>
<a href="/about">If you're really interested,</a> a link to my bio and some presentations I've given. You can also find me on <a href="https://social.securitytheater.net/@spaceinvader">Mastodon</a> or the Cloud Security Forum Slack. Nothing I say is official, I don't speak for my employer, I am not a lawyer, no warranty express or implied, allow eight weeks for shipping and handling, etc.