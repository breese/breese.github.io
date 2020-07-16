---
layout: default
---

<p/>

<center style="width:55%">
<quote class="verse">I saw the immense beauty of the twinkling stars;<br/>
like countless diamonds on an inky cloak<br/>
of inaccessible and inscrutable depths.<br/>
The sparkling stars dim and gracefully recede.<br/>
The morning star preach about the comming light;<br/>
an epoch of uproar and change,<br/>
an epoch of progress and enlightenment.<br/>
The dark night on a furious retreat from a battle lost.<br/>
The divine morning sun rise in a magnificent battledress<br/>
of smouldering red, orange and yellow.<br/></quote>
-- Stellifer by Bj&oslash;rn Reese, 1995.
</center>

## Under Development Blog

<table>
{% for post in site.posts limit:10 %}
<tr>
<td><a href="{{ post.url }}">{{ post.title }}</a></td>
<td>{{ post.date | date: "%Y-%M-%d" }}</td>
</tr>
{% endfor %}
<tr>
<td><a href="blog/">Archive</a></td>
</tr>
</table>
