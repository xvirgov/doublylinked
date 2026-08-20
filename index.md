---
layout: default
title: home
---

# xvirgov@blog:~$ ls -la posts/

Notes on computer security — exploits, defenses, crypto, and whatever else catches my attention.

---

{% for post in site.posts %}
- **{{ post.date | date: "%Y-%m-%d" }}** &nbsp;&nbsp; [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}

---

## about

Hi, I'm xvirgov. This is where I write up things I learn about computer security. See the [about page](/about/) for more.

```
$ whoami
xvirgov
$ cat /etc/motd
welcome. mind the segfaults.
```
