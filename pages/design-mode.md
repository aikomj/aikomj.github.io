---
layout: page
title: 设计模式系列文章
titlebar: java
subtitle: <span class="mega-octicon octicon-globe"></span> &nbsp;&nbsp; 修炼java内功，23种设计模式
menu: design-mode
css: ['blog-page.css']
permalink: /design-mode
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='design-mode' or post.tags contains 'design-mode'%}
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