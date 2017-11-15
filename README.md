## Take a look at my blog posts

{% for post in site.posts %}	
[{{ post.title }}]({{ post.url }}) ({{ post.date }})
{% endfor %}	