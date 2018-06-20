---
layout:     post
title:      
subtitle:   Spring容器与Spring MVC容器之间的关系
date:       2018-06-19
author:     Logic
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
    - spring mvc
---

Spring 容器和Spring MVC容器虽然是父容器与子容器的关系，但二者之间具有一定的独立性。具体来说，两个容器基于各自的配置文件分别进行初始化，只有在子容器中找不到对应的Bean的时候，才会去父容器查询并加载。