---
layout: post
date: 2018-12-22
title: "Writing A Better Descriptor Abstraction For Vulkan"
img: spider_meadows.jpg
published: false
tags: [Vulkan, C++, Tools, Shaders, SPIR-V, Lua]
---

Cover the following:
    - Holy crap writing this article is always hard because the software is evolving
    - Difficulties of maintaining data relationships and heirarchies (excessive blurring of responsibilities)
    - Serialization attempts that resulted in larger-than-desired files
    - Repeated work to generate and compile shaders
    - Poor interface to users

Rewrite goals:
    - Use shader processor to thread the bulk of the work 
    - For expensive "recompile" step, also bounce off to threads
    - Reduce binary size by removing duplicated data
    - Lay the path for future improvements, by allowing for cooperation between the compiler and generator
