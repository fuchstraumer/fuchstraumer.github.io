---
layout: post
date: 2018-8-25
title: "Making a Vulkan Resource Plugin"
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

### Pitfalls of Pre-Existing Systems

But then I had to begin adding more implicit structures and functionality, like staging buffers - in Vulkan, if we want to put data in GPU-only memory (i.e, not visible or modifiable by the CPU) we first have to stage it in some CPU-side buffers: then, we can schedule a transfer (potentially on the transfer queue, i.e through the DMA bus) to copy this staged data onto the GPU. What I had begun to do, when the user tried to create a `vpr::Buffer` or `vpr::Image` that would have it's data in GPU-exclusive memory was implicitly create the staging buffers required, and place them in a container so they stayed around until we transferred them. 

But this brings up a question - when can we release these staging buffers? They will represent some actual memory usage for a user (so just occupying RAM), and we don't want to just be cluttering our `vpr::Allocator` system with blocks of now-useless data. I ended up settling on exposing a static method one could call called `vpr::ReleaseStagingBuffers(vpr::Device* dvc)`, which worked - but was not really ideal, of course. Now I had put the burden on users remembering to call this frequently enough, and the whole system just felt unwieldy when scaled up.

In addition, I was almost always implicitly creating `VkImageView` objects - singular, per image - implicitly when creating a `vpr::Image`. But it's not *too* uncommon of a scenario for one to need more than `VkImageView` per image, so that was another restriction of my current system. Further, delayed initialization (mostly by delaying when we upload or set the contents of a buffer/image) was not very easy at all - you couldn't create the parent buffers or images then delay backing memory assignment at all, and there are situations that come up fairly frequently where this is exactly what you want to do.

### Goals for New System

Okay, so we clearly have some room for improvement when it comes to creating, destroying, modifying, and generally managing our Vulkan resources. Before I began, I laid out a few clear requirements I had in mind:

    - Creating resources should avoid as many implicit hidden steps as possible
    - Creating resources with initial contents should be as easy as creating them *without* any initial contents
    - Destroying resources should be able to be performed simply and explicitly
    - Creating and destroying resources from multiple threads must be possible
    - Transferring data from staging buffers should be performed asynchrously and without user involvement
    - Cleaning up and managing the lifetime of these staging buffers and their data must be done by the system, not the user

This gives us a fairly clear picture of what capabilities we need to support, so I began to move forward with the design.

## Implementing Our Resource System

### Step 1: Considering the Nature of our Plugin System

As mentioned briefly in the opening paragraph, this resource system will be part of the plugin-based paradigm I am pursuing in my engine code. So that is going to make things immediately difficult: as in, passing complex classes like the old `vpr::Buffer` and `vpr::Image` across plugin boundaries is going to become impossible. But this is also probably for the best - these complex classes resulted in a lot of behind-the-scenes implicit behavior and a number of assumptions based on the user's intent.

Now I won't be able to - or have to, depending on your perspective - make assumptions about the user's desires. Second, all the actual functionality and work is going to be part of the system. I have, in effect, run into something akin to the Entity-Component-System approach to design merely by necessity. I have begun to see many upsides in this sort of software design, however, as it forces one to design pragmatic and functional interfaces and encourages careful thought througout the process of writing a new subsystem.

So, we're going to keep our object encapsulating a Vulkan resource simple - so I settled on the following `VulkanResource` structure:

{% highlight cpp %}
struct VulkanResource {
    resource_type Type{ resource_type::INVALID };
    uint64_t Handle{ 0 };
    void* Info{ nullptr };
    uint64_t ViewHandle{ 0 };
    void* ViewInfo{ nullptr };
    const char* Name{ "" };
    void* UserData{ nullptr };
};
{% endhighlight %}

There's nothing too surprising here: the nice thing about the handles to Vulkan API objects is that they can (thus far) be safely stored as `uint64_t` values (which is actually quite true, if I am interpreting the specification right). The leading field `resource_type` can be used internally to verify we're interpreting something correctly before doing things with it, and can be used externally by users as well for the same purpose. Our `void*` for `Info` and `ViewInfo` will be used to point to the Vulkan `CreateInfo` structs used for the current resource type, as appropriate - these structures can be relatively large, and vary in size between resource types so this is the only way to really do this. Keeping them around can be useful, however, as higher-level systems may want to do a bit of reflection or inquiry into the metadata attached to a resource. By simply casting to our desired types (e.g, `Handle` to `VkBuffer` with `(VkBuffer)Handle`) we are able to use the structures fields directly in Vulkan functions.

`Name` and `UserData` have been left for later use: named resources can be quite useful when working with shader reflection and rendergraph systems (*cough*, ShaderTools), and `UserData` has become something I've grown to appreciate - allowing for later expansion or the attachment of further metadata as required even if it falls outside the scope of our original plans.

### Step 2: Creating VulkanResource objects

In order to avoid having the user copy a bunch of data (mostly pointers and 64-bit handles, to be fair) they don't probably need a copy of, we're going to have all of our creation functions just return `VulkanResource*`. Users can then store and use these pointers as they see fit. When creating a resource, we'll give users optional parameters by having most parameters be provided as pointers, so that they can selectively pass `nullptr` for the pieces they don't expect to use. Our two main creation functions will then be:

{% highlight cpp %}
VulkanResource* CreateBuffer(const VkBufferCreateInfo*, const VkBufferViewCreateInfo*, const size_t num_data, 
    const gpu_resource_data_t* initial_data, const uint32_t mem_type, void* user_data);
VulkanResource* CreateImage(const VkImageCreateInfo*, const VkImageViewCreateInfo*, const size_t num_data, 
    const gpu_image_resource_data_t* initial_data, const uint32_t _memory_type, void* user_data);
{% endhighlight %}

The parameters that I haven't directly addressed yet are those used to se the initial contents of the buffers - `gpu_resource_data_t` is a structure that holds a pointer to some data along with the size of said data, along with some memory alignment and pitch information members as required. I unfortunately found out that a seperate, and more detailed, structure `gpu_image_resource_data_t` is effectively required in order for us to be able to set the initial data of images, however. We'll return to that shortly, however. The rest of the parameters should be fairly apparent - if we pass a `nullptr` for a `view_info` parameter, that resource's `ViewHandle` field won't be set as the view object won't be created (this will be pretty common with buffers, for example, unless we're using them as texel buffers).

#### Setting Initial Resource Data

This is where things got to be a little bit tricky: previously, a lot of this logic had been made more implicit or had been embedded fairly deeply into the individual resource types themselves. At first, I thought I could get away with using this simple structure for setting the contents of buffers *and* images:

{% highlight cpp %}
struct gpu_resource_data_t {
    const void* Data;
    size_t DataSize;
    size_t DataAlignment;
    uint32_t Pitch;
    uint32_t SlicePitch;
};
{% endhighlight %}
