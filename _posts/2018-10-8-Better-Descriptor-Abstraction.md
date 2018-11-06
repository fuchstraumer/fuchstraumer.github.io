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

## Back To A Slot-Based Approach?

A classic fixture of older APIs was the idea of declaring or having a "slot" for a resource in a shader or pipeline (or the abstract idea of a pipeline, in most cases). One would say "Hey, I'm going to use a texture at slot 0" or something, then bind data to that actual slot before drawing with it. The process I'm going to use here will be vaguely similar - we declare our slots through structures like a `VkDescriptorSetLayout`, which is attached to a `VkPipelineLayout` and used to build a pipeline. At this point, we've only declared what resources we're going to use and where they're going to be bound. 

The interface for this can be really simple: all we need to construct this layout information is an integer index and a `VkDescriptorType` value. Optionally, we can also pass a whole `VkDescriptorSetLayoutBindingValue`:

{% highlight cpp %}
void AddLayoutBinding(const size_t idx, const VkDescriptorType type);
void AddLayoutBinding(const VkDescriptorSetLayoutBinding& binding);
{% endhighlight %}

We'll call this repeatedly, most likely hooking into what our shader reflection system provides (for me, ShaderTools). This way we can reflect on our shaders being used and use that to construct layout info for our new `Descriptor` object - without needing to bind any resources quite yet. This is particularly nice for things like textures and samplers, and other cases where we aren't able to automate the construction of the backing resources for a given slot. 

Under the hood, this is really just adding bindings to a `vpr::DescriptorSetLayout` object. We'll construct this object when required to, i.e. when we try to retrieve the `VkDescriptorSetLayout` handle for constructing a `VkPipelineLayout` for a pipeline that wants to use our `Descriptor`. This should be fairly low-cost, and keeps things nice and simple without requiring an explicit creation/initialization function.

## Binding Resources Proper

To bind the *actual* resources to be used in a drawing operation though, we'll bind one of our `VkDescriptorSet`s. Resources are bound to the descriptor set, then, when we call `VkUpdateDescriptorSet`. Doing this requires that we have built an array of `VkWriteDescriptorSet`s - these objects hold binding information, descriptor type info, and more importantly the handles to the actual resources we are going to use with this set. We'll add resources to bind through the following interface, using the `VulkanResource` object I created for my resource system ([article about that here](https://fuchstraumer.github.io/Vulkan-Resource-Plugin/)):

{% highlight cpp %}
void BindResourceToIdx(const size_t idx, const VulkanResource* rsrc);
{% endhighlight %}

Unlike previous APIs, though, much of our data is still going to be baked into the pipeline - so this won't really be the same as a conventional binding operation. We will, however, need to occasionally change the contents of a descriptor set. This means that we might be calling this after already constructing the descriptor set once, or at some point during the rendering process. Consider the case of something like a descriptor set used for material data, if we're doing something like the ping-pong sorting setup mentioned previously, or we've had to destroy and reallocate our buffers (e.g, if we're defragging the GPU memory and things got moved a bit). 

This involves building an array of those write descriptors mentioned earlier, which can become relatively expensive since we *have* to (by decree of the specification) create one `VkWriteDescriptorSet` for each resource. This can ultimately become a bit wasteful due to things that aren't being updated, redundant state information being included (e.g what descriptor type is bound at location X), and so on. We can reduce the performance cost of updating these resources though - by using a `VkDescriptorUpdateTemplate`.

## Update Templates?

`VkDescriptorUpdateTemplate` has been around for a bit, but as of Vulkan 1.1 it's no longer a KHR extension and has been made a part of the core API. The premise of these update templates is 

## Registering Resources

- pass `VulkanResource*` 
- have to assume some stuff for image resources

## Updating With A Descriptor Update Template

- required linear array of data entries per descriptor entry
- have to use a union since there's no tidy way to do this
