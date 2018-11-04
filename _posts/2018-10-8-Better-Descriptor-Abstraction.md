---
layout: post
date: 2018-9-14
title: "Better Descriptor Abstraction In Vulkan"
img: spider_meadows.jpg
published: false
tags: [Vulkan, C++, Tools, Graphics]
---

While trying to write a merge sort compute shader for my ongoing work to implement Volumetric Tiled Forward rendering, I noticed a potential scenario that I had absolutely no infrastructure or real ability to deal with: ping-pong bouncing of descriptor set bindings for usage in sorting shaders. The use case has us swapping the underlying buffers bound to the "input" and "output" locations of our sort shaders - so that we iteratively keep sorting the same dataset, consistently writing the ultimate results and converging to a solution.

This isn't exactly a simple operation in Vulkan though, especially not with how my `vpr::DescriptorSet` abstraction is setup. That and trying to make resource creation and binding easier for your average user provided insight that I definitely needed a better descriptor abstraction - not just for the sets, but also for the layouts, most likely. It's going to sit upon the existing `vpr` objects, however, as I don't believe the functionality and implementation I have in mind belongs at the level of that library.

## Registering Resources

- pass `VulkanResource*` 
- have to assume some stuff for image resources

## Updating With A Descriptor Update Template

- required linear array of data entries per descriptor entry
- have to use a union since there's no tidy way to do this