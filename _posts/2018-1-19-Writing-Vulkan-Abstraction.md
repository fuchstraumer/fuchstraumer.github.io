---
layout: post
date: 2018-1-27
title: "Designing, Building, and Using a Vulkan abstraction layer"
img: pinnacle.jpg
published: true
tags: [Vulkan,C++,Design,Libraries, Modern C++]
---

Over the past several months I have been working on a Vulkan abstraction layer - in this
post, I'd like to go through the process by which I've built this library: from it's 
barest beginnings, to it's awkward "lets do everything in one library" middle-stage, to
what I believe is now a well-designed and clearly defined abstraction library.

The first few major chunks on Vulkan itself will be written as if the reader has no previous 
knowledge of Vulkan, so feel free to skip into the middle of the article for more design-oriented
stuff.

## The Origin Story 

I chose to start learning Vulkan after I was having issues getting CUDA and OpenGL to play nice -
I was at the time generating procedural noise using CUDA, and even my (in hindsight) naive and suboptimal
implementation was quite capable of trouncing similar CPU implementations:

<img src="{{site.baseurl}}/assets/img/noise_benchmarks.png" width="768" />


<img src="{{site.baseurl}}/assets/img/noise_demo.jpg" width="512" height="512" />

I was trying to procedurally generate and render globes at the time, since that's _clearly_ a wise project
for someone who only started programming in C++ (and at all, really) 4-5 months prior. But I'm stubborn,
if nothing else, so I tried to make things happen.

Passing resource handles between OpenGL and CUDA wasn't the easiest though, and I was starting to grow 
more and more irritated with some of the implementation choices behind CUDA. Like the existence of 
vector classes (e.g `float3`, `int4`, etc) but no implemented operators for them. They instead had to be
acquired by using a third party header - seriously, NVidia? 

But what really drew me in was the promise of the complexity of Vulkan. Y'see, I'm the sort of person that 
adores games like the DCS series and gets addons like the PMDG-made series of commercial airliners for 
flight sims. So while some see complex interfaces and arrays of switches and say "__NOPE__", I find myself
resisting the urge to start fiddling with all the dials, knobs, and switches just to figure out how it all works.

Also, I probably have an issue with self-loathing. I started programming by learning C++ and then decided I was 
going to learn Vulkan. People who care about themselves don't do these things.

## (Volcano Memes), or, diving into Vulkan

One of the problems that is still somewhat present is the comparative lack of resources compared to the much
more established APIs like DX9 and OpenGL. The best resource is the red book equivalent for Vulkan, which
has good-enough examples of how to use everything and does go into a satisfying amount of depth on each individual
object and resource that Vulkan presents to a user of the API.

From the start, everything you do in Vulkan feels radically removed from that of doing the same in OpenGL. Recording
drawing commands into command buffers is one thing, but actually having some sort of direct interface to the hardware’s
command queues (via queue families) is an entirely different (and awesome!) thing.

### Vulkan objects - common traits

Most objects in Vulkan take some kind of `createInfo` structure pointer in their creation functions. These structures specify 
various attributes of the object that won't (usually) be allowed to change during use. Some of these are simple attributes that 
only the implementation really cares about - like transient and one-time-submit command buffer flags - or are more important, like 
image tiling used for an image object or what a buffer objects usage will be.

`sType` and `pNext` members are universal to nearly every single structure in Vulkan. I can't quite tell you
why the `sType` member exists, though I'm sure there's a reason. The `pNext` structure is useful for extensions, though: in the case of
the ones I've used thus far, you supply the vanilla structure to a function but set `pNext` to point to the relevant structure from the 
extension you're using. For 99% of use cases though, this can be set to `nullptr`.

Creation and destruction functions also take a `VkAllocationCallbacks*` parameter, which allows a user to define their own custom 
memory allocation functions to use for CPU-side resources. I've not yet seen a major need for this, and honestly have it forced to `nullptr`
in most of my code (though that's something I'd like to change soon). If anyone has seen a major benefit to using these, please let me know.

Enums are also everywhere. And some of them get nasty long - looking at you, `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_MULTIVIEW_PER_VIEW_ATTRIBUTES_PROPERTIES_NVX`
or the compressed texture format `VK_FORMAT_G12X4_B12X4_R12X4_3PLANE_444_UNORM_3PACK16_KHR`. Before you get too horrified though, know that 
I chose extreme cases and once you wrap these things away a bit the pain is eased pretty quickly.

### The Core Vulkan objects

In Vulkan, we have 3 objects that I considered the "core" objects: `VkInstance`, `VkPhysicalDevice`, and `VkDevice`. A Vulkan
instance is what contains the total "state" of all Vulkan stuff going on. Objects can't be easily shared or passed across
instance boundaries. 

### Vulkan Images and ImageViews
Remember creating OpenGL textures with the following few lines?

{%highlight cpp%}
GLuint tex;
glGenTextures(1, &tex);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 2, 2, 0, GL_RGB, GL_FLOAT, pixels);
{%endhighlight%}

Well, wave goodbye to that in Vulkan. First, the good news - storing a texture in an RAII style and even in a class at all
is much easier now. You don't have to call `glTexParameter...` to set texture options or variations - this is set easily on
a per-object basis upon creation, but it's also usually fixed upon creation. Also, there are no raw `VkTexture` objects and 
there's actually two objects you need to keep track of per instance of a Vulkan texture.

Vulkan uses `VkImage` as it's base Image object, and you're also almost certainly going to need to create a corresponding 
`VkImageView` object. Also, you have to explicitly give this image some `VkDeviceMemory` to use - for now, let's just assume
we'll give each memory-consuming object it's own unique object. We'll return to the better ways of doing this - via subregion 
binding - later, when I talk about the allocator system I'm using.

So yeah, we're already up to a few more moving bits and pieces than the single `GLuint` we used with OpenGL. But it gets worse, oh
so much worse. You see, every one of the somewhat detailed/advanced objects in Vulkan has a `CreateInfo` struct. So we are going to
have to fill out a `VkImageCreateInfo` structure and a `VkImageViewCreateInfo` struct. The `VkImageCreateInfo` struct isn't exactly
simple:

{% highlight cpp %}
typedef struct VkImageCreateInfo {
    VkStructureType          sType;
    const void*              pNext;
    VkImageCreateFlags       flags;
    VkImageType              imageType;
    VkFormat                 format;
    VkExtent3D               extent;
    uint32_t                 mipLevels;
    uint32_t                 arrayLayers;
    VkSampleCountFlagBits    samples;
    VkImageTiling            tiling;
    VkImageUsageFlags        usage;
    VkSharingMode            sharingMode;
    uint32_t                 queueFamilyIndexCount;
    const uint32_t*          pQueueFamilyIndices;
    VkImageLayout            initialLayout;
} VkImageCreateInfo;
{% endhighlight %}

Lets break this down bit by bit though:

- `flags`: another common occurance in Vulkan structure, used to specify some unique attributes of this object upon creation. 

- `VkImageType`: this is where we specify the format of the image, e.g 2D vs 3D vs cubemap. (`VK_IMAGE_TYPE_2D`, `VK_IMAGE_TYPE_3D`, `VK_IMAGE_TYPE_CUBE_ARRAY`)

- `VkFormat`: Oh god, this is admittedly one of the worst enums in Vulkan. Look upon the full list and weep - regardless, it's good to specify this ahead of time and so explicitly as it can rather
tremendously affect how Vulkan is able to optimize and layout this memory

- `mipLevels`, `arrayLayers`: does what it says on the tin. We can't generate mipmaps like in Vulkan, so one has to specify this ahead of time. `arrayLayers` is for array textures, which are less of a pain in the ass to setup and deal with than they were in OpenGL

- `samples` is for setting the level of anisotropic filtering. Support for this varies from device to device.