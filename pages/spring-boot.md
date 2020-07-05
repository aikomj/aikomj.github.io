---
layout: page
title: Spring Boot 系列文章
titlebar: spring-boot
subtitle: <span class="mega-octicon octicon-package"></span> &nbsp;&nbsp; 锲而不舍的干到底，结果只能是成功
menu: spring-boot
css: ['blog-page.css']
permalink: /spring-boot
keywords: Spring Boot 教程,Spring Boot 示例,Spring Boot 学习,Spring Boot 资源,Spring Boot 2.0
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category =='springboot' or post.tags == 'springboot' or post.keywords contains 'springboot'%}
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