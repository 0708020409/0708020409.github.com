---
layout: page
title: If i rest,i rust !
tagline: 
---
{% include JB/setup %}

## Post Lists

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
   
## To-Do

-	Post something about nginx or lua.
-	Vitual Memory Study.
-	Redis.
-	Hadoop.
   
     
	   
	     
