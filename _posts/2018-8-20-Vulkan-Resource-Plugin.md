---
layout: post
date: 2018-7-20
title: "Making an Asset Loader Plugin in C++"
img: cascades.jpg
published: false
tags: [C++, Engine Development, Vulkan, Plugins]
---

# Creating a Vulkan Resource-Management plugin

Last time, I covered the creation of a plugin for parallelizing the loading of assets from disk - this time, I'll be covering a plugin that plays very well with that system. Management of Vulkan API resources can be a tough topic: I've attempted to solve this problem about a dozen different ways by now, but I'm most content with this recent approach. When I refer to Vulkan resources I'm considering primarily `VkBuffer` and `VkImage`, but I also decided to integrate and support `VkSampler` in this system as well. 

Like last time, this will be integrated as a plug-in for `Caelestis`, which can be found here. This is the `resource_context` plugin.

## Why do we need to manage these resources?

I'm going to briefly assume that the reader doesn't have any real familiarity with Vulkan: but in Vulkan, we have to do a considerable quantity of work in managing our resources. Not only do they have to be manually created and destroyed (making sure to destroy them before the parent device is destroyed), we also have to manage the backing memory attached to them, plus potential expansions to their functionality in the form of `VkBufferView` or `VkImageView` (and the lifetimes of *those* objects too).

Up to this point, I had mostly stuck with what I was using in `VulpesRender` up to this point: a Vulkan device (a software abstraction over the GPU) is an object in my library (`vpr::Device`), and this created and managed a single `vpr::Allocator` instance - an allocator being an in-library class and system for allocating and managing memory for Vulkan. One could retrieve the allocator by calling a function attached to `vpr::Device`. Then, objects like `vpr::Buffer` and `vpr::Image` were able to request allocation and deallocating through this interface. Things worked quite well - for the most part!

