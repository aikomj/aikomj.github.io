---
layout: page
title: 工作中用到的工具
titlebar: Tool
subtitle: <span class="mega-octicon octicon-organization"></span>&nbsp;&nbsp; 盘点下我用的顺手的那些工具！
menu: Tool
css: ['blog-page.css']
permalink: /tool
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='tool' or post.tag == 'tool' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}" target="_blank">{{ post.title }}</a>
                        <p style="color:#666">{{ post.excerpt }}</p>
                        <span class='circle'></span>
                    </div>
                </li>
                {% endif %}
            {% endfor %}
        </ul> 

        <!-- Pagination -->
        {% include pagination.html %}

        <!-- Comments -->
       <div class="comment">
         {% include comments.html %}
       </div>
    </div>

</div>
<script>
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });

    });
</script>