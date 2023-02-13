---
layout: page
title: k8s(Kubernetes) 系列文章
titlebar: Docker
subtitle: <span class="mega-octicon octicon-flame"></span>&nbsp;&nbsp; 相信付出的力量
menu: docker
css: ['blog-page.css']
permalink: /k8s
keywords: k8s
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='k8s' or post.tags contains 'k8s' or post.keywords contains 'k8s' %}
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