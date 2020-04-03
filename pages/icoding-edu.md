---
layout: page
title: 艾编程Java架构师课程笔记
titlebar: icoding-edu
subtitle: <span class="mega-octicon octicon-globe"></span>&nbsp;&nbsp;日均亿级访问系统架构设计训练营，<a href="https://www.icodingedu.com/" target="_blank">更多精选课程 ，<font color="#EB9439">点我</font>查看！</a>
menu: icoding-edu
css: ['blog-page.css']
permalink: /icoding-edu
keywords: icoding
---
<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='icoding-edu' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}" target="_blank">{{ post.title }}</a>
                        <p style="color:#666">
                            {{ post.excerpt }}
                        </p>
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