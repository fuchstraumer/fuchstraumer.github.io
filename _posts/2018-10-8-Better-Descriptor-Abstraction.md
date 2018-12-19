---
layout: post
date: 2018-9-14
title: "Writing A Better Descriptor Abstraction For Vulkan"
img: spider_meadows.jpg
published: true
tags: [Vulkan, C++, Tools, Graphics]
---

<meta property="og:url" content="https://fuchstraumer.github.io/Better-Descriptor-Abstraction/">
<meta property="og:type" content="article">
<meta property="og:title" content="Writing A Better Descriptor Abstraction For Vulkan">
<meta property="og:description" content="How can we write a performant and easy to use abstraction over Vulkan's descriptor system? We'll also explore using Descriptor Set Update Templates.">
<meta property="og:image" content="https://fuchstraumer.github.io/assets/img/spider_meadows.jpg">

While trying to write a merge sort compute shader for my ongoing work to implement Volumetric Tiled Forward rendering, I noticed a potential scenario that I had absolutely no infrastructure or real ability to deal with: ping-pong bouncing of descriptor set bindings for usage in sorting shaders. The use case has us swapping the underlying buffers bound to the "input" and "output" locations of our sort shaders - so that we iteratively keep sorting the same dataset, consistently writing the ultimate results and converging to a solution.

This isn't exactly a simple operation in Vulkan though, especially not with how my `vpr::DescriptorSet` abstraction is setup. That and trying to make resource creation and binding easier for your average user provided insight that I definitely needed a better descriptor abstraction - not just for the sets, but also for the layouts, most likely. It's going to sit upon the existing `vpr` objects, however, as I don't believe the functionality and implementation I have in mind belongs at the level of that library.

### Back To A Slot-Based Approach?

A classic fixture of older APIs was the idea of declaring or having a "slot" for a resource in a shader or pipeline (or the abstract idea of a pipeline, in most cases). One would say "Hey, I'm going to use a texture at slot 0" or something, then bind data to that actual slot before drawing with it. The process I'm going to use here will be vaguely similar - we declare our slots through structures like a `VkDescriptorSetLayout`, which is attached to a `VkPipelineLayout` and used to build a pipeline. At this point, we've only declared what type of resources we're going to use and where they're going to be bound. 

The interface for this is simple: all we need is an integer index and a `VkDescriptorType` value. Optionally, we can also pass a whole `VkDescriptorSetLayoutBindingValue`:

{% highlight cpp %}
void AddLayoutBinding(const size_t idx, const VkDescriptorType type);
void AddLayoutBinding(const VkDescriptorSetLayoutBinding& binding);
{% endhighlight %}

We'll call this repeatedly, most likely hooking into what our shader reflection system provides (for me, ShaderTools). This way we can reflect on our shaders being used and use that to construct layout info for our new `Descriptor` object - without needing to bind any resources quite yet. This is particularly nice for things like textures and samplers, and other cases where we aren't able to automate the construction of the backing resources for a given slot. 

Under the hood, this is really just adding bindings to a `vpr::DescriptorSetLayout` object. We'll construct this object when required to, i.e. when we try to retrieve the `VkDescriptorSetLayout` handle for constructing a `VkPipelineLayout` for a pipeline that wants to use our `Descriptor`. This should be fairly low-cost, and keeps things nice and simple without requiring an explicit creation/initialization function.

### Binding Resources Proper

To bind the *actual* resources to be used in a drawing operation though, we'll bind one of our `VkDescriptorSet`s. Resources are bound to the descriptor set, then, when we call `VkUpdateDescriptorSet`. Doing this requires that we have built an array of `VkWriteDescriptorSet`s - these objects hold binding information, descriptor type info, and more importantly the handles to the actual resources we are going to use with this set. We'll add resources to bind through the following interface, using the `VulkanResource` object I created for my resource system ([article about that here](https://fuchstraumer.github.io/Vulkan-Resource-Plugin/)):

{% highlight cpp %}
void BindResourceToIdx(const size_t idx, const VulkanResource* rsrc);
{% endhighlight %}

Unlike previous APIs, though, much of our data is still going to be baked into the pipeline - so this won't really be the same as a conventional binding operation. We will, however, need to occasionally change the contents of a descriptor set. This means that we might be calling this after already constructing the descriptor set once, or at some point during the rendering process. Consider the case of a descriptor set used for material data, or if we're doing something like the ping-pong sorting mentioned in the opening paragraph, or if we've had to destroy and reallocate our buffers (e.g, if we're defragging the GPU memory and things got moved a bit). 

This involves building an array of those write descriptors mentioned earlier, which can become relatively expensive since we *have* to (by decree of the specification) create one `VkWriteDescriptorSet` for each resource. This can ultimately become a bit wasteful due to things that aren't being updated, redundant state information being included (e.g what descriptor type is bound at location X), and so on. We can reduce the performance cost of updating these resources though - by using a `VkDescriptorUpdateTemplate`.

### Update Templates?

`VkDescriptorUpdateTemplate` has been around for a bit, but as of Vulkan 1.1 it's no longer a KHR extension and has been made a part of the core API. The premise of these update templates is to potentially reduce the overhead of updating what resources are bound to a descriptor, by reducing the size of the update data package. Instead of passing in a whole `VkWriteDescriptorSet`, we are effectively going to be only passing the smaller subset structure used to specify the handles to the resources we're using: `VkDescriptorBufferInfo`, `VkDescriptorImageInfo`, or just a plain `VkBufferView` for objects like texel buffers. 

We create the update template by creating an array of `VkDescriptorUpdateTemplateEntry` structures, primarily. This structure looks like so:

{% highlight cpp %}
typedef struct VkDescriptorUpdateTemplateEntry {
    uint32_t            dstBinding;
    uint32_t            dstArrayElement;
    uint32_t            descriptorCount;
    VkDescriptorType    descriptorType;
    size_t              offset;
    size_t              stride;
} VkDescriptorUpdateTemplateEntry;
{% endhighlight %}

Nothing here should be too surprising: `dstBinding` is what location we're specifying update data for, and the next three fields are for specifying the descriptor type and if it's an array descriptor. `offset` and `stride` can be a little confusing if you're just reading [the specification](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkDescriptorUpdateTemplateEntry.html), however: `offset` will be used later, when we make the call to update the descriptor set using our update template. It specifies how far into the raw data buffer passed to this function call we have to go to find the data for this particular update entry - whereas `stride` is going to be used with array descriptors. If we have an array descriptor, we need to know how far to step to read the next desciptor at offset `i` - in this case, it would be the size of our structure that is storing the update data. If you're not using array descriptors though, it'll just be zero. Here's how I add a new update entry:

{% highlight cpp %}
addUpdateEntry(idx, VkDescriptorUpdateTemplateEntry{ 
    uint32_t(idx),
    0,
    1,
    type,
    sizeof(rawDataEntry) * idx,
    0 // size not required for single-descriptor entries
});
{% endhighlight %}

This will insert a `VkDescriptorUpdateTemplateEntry` into the `std::vector` I'm using for these structures at the given idx, and in this case our `stride` field is the size of the structure with my update data (`rawDataEntry`) times the binding idx of our object. Nice and easy, this part is! I will briefly say that a `VkDescriptorSetUpdateTemplate` also has two modes - the one I'm using, and a mode that makes it function somewhat like a more advanced push constant. This is all set and configured in the corresponding `createInfo` structure for this object, so definitely be sure to give the [corresponding documentation](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkDescriptorUpdateTemplateCreateInfo.html) a thorough read. Moving on though, how do we go about storing the update data? And how do we perform this updating?

### Updating With A Descriptor Update Template

As mentioned, when we call `vkUpdateDescriptorSetWithTemplate` we're going to be reading from a raw array containing just the bare minimum of data we need to update our descriptor entries. This still means we have a potential of three different structure types being used to store the data, though: how can we store them contiguously and together as if they were one type? Figuring this out took me a few minutes, as the solution isn't one I use often in my land of higher-level and more modern C++: we use a `union`. A union stores only one of it's fields at a time, but has a consistent size (the size of the largest member type). So, I created a `rawDataEntry` object that looks like so:

{% highlight cpp %}
union rawDataEntry {
    VkDescriptorBufferInfo BufferInfo;
    VkDescriptorImageInfo ImageInfo;
    VkBufferView BufferView;
};
{% endhighlight %}

Nothing at all surprising here - now when we want to add an entry for a new image at index `i`, we just add to our vector of these objects (`rawEntries`) an instance of this union where we've initialized and filled out the field `ImageInfo`. The API knows from the `VkDescriptorUpdateTemplateEntry` we gave it earlier that when it goes to look at this index for the update data it should expect to find a `VkDescriptorImageInfo`, and so it interprets the data there as such and is able to use it to bind a specific `VkImageView` (and corresponding sampler, if it's a combined image-sampler) to a given location. Calling the update function itself is incredibly simple, now:

{% highlight cpp %}
vkUpdateDescriptorSetWithTemplate(device->vkHandle(), descriptorSet, updateTemplate, rawEntries.data());
{% endhighlight %}

Also, as an interesting note one may wish to consider for further refinements of this design and idea: an update template is *not* explicitly bound to a single descriptor set. It really is an update *template* that can be used with multiple descriptor sets, potentially allowing one to even more efficiently or effectively use it for updating multiple descriptor sets sharing the same layout throughout a frame. The way I use it here is the simplest case, and has already satisfied my requirements - but I'm interested in exploring this idea further and seeing how it can be refined (e.g, [what is suggested in this blog post](https://ourmachinery.com/post/vulkan-descriptor-sets-management/)). 

### Binding With Update Templates

As mentioned above, we bind an actual physical resource to a location with a call to `BindResourceToIdx`. This takes in a location and a pointer to my `VulkanResource` structure ([created and explained in this article](https://fuchstraumer.github.io/Vulkan-Resource-Plugin/)), from which it is able to get the necessary info. This function itself breaks down to be fairly simple:

{% highlight cpp %}
void Descriptor::BindResourceToIdx(size_t idx, VulkanResource* rsrc) {
    // set a flag so we make sure to call update() on next use in a frame
    if (!dirty) {
        dirty = true;
    }

    // no resource bound at that index yet, create the required data entries
    if (createdBindings.count(idx) == 0) {
        addDescriptorBinding(idx, rsrc);
    }
    else {
        // just update the handles used at idx
        updateDescriptorBinding(idx, rsrc);
    }
   
}
{% endhighlight %}

We keep a `dirty` flag so that when a user tries to grab the underlying `VkDescriptorSet` handle we are able to update the descriptor set, so that they always get the most up-to-date version of it with the resources bound that they expect. `createdBindings` is just an `unordered_set` of indices - while trying to use this object, I realized I wasn't effectively tracking which locations had already had their `VkDescriptorSetUpdateTemplateEntry` and `rawDataEntry` objects created properly. This becomes problematic, of course, when we go to try to bind a resource to an index - if we already have the objects created, we can potentially save some time by just doing some simpler tasks to update the handles in the `rawDataEntry` object corresponding to that index.

### Example and Conclusion

As a quick example to verify that my Descriptor implementation works ([header available here](https://github.com/fuchstraumer/DiamondDogs/blob/92b6243863708ed1ae40436ca615e82d2808c8cb/modules/rendergraph_module/include/Descriptor.hpp), [and source here](https://github.com/fuchstraumer/DiamondDogs/blob/b45fa1a9ab6fb224a66688fac96ddfeade61c215/modules/rendergraph_module/src/Descriptor.cpp)), here's a quick test I created for it: in it, pressing the `V` key toggles between two textures for the skybox. Switching between them seems to work without issues, and now that I've got my abstracted interface working it seems to be easier than the alternative:

<iframe width="400" height="300" 
src="https://giant.gfycat.com/FeminineFlawlessLacewing.webm">
</iframe>

It has also allowed me to proceed with my implementation of the sorting algorithm from that DX12 lighting method - and in hindsight, could allow one to more closely emulate how DX12 allows one to do fairly late binding of resources (at least compared to my bind-early approach implemented in `vpr::DescriptorSet`). The only caveat is that I don't store much metadata or info about what resource is bound to an index - so if I wanted to change the binding of the resource called "InputKeys" in a `DescriptorObject`, I'd have to rely on a higher level abstraction associating the names of the resources to an index (i.e, I use a `std::unordered_map<std::string, size_t>` in the object that manages the lifetime of the `Descriptor`s I create for a group of shaders).

This article was fairly short, but I'm trying to use more of this format from here on out: hopefully, it helped clarify some potential uses of descriptor set update templates (and how one can get them to work, e.g using a `union` and so on). If I'm lucky, it'll help give you some ideas for making a robust and easy-to-use abstraction for descriptors in Vulkan - without completely comprimising on performance while doing so. As always, feel free to contact me with questions or leave them as comments. Criticism, ideas, and clarification is always welcomed!
