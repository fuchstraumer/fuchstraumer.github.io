---
layout: post
date: 2018-9-14
title: "Introduction to Vulkan"
img: yosemite1.jpg
published: true
tags: [Vulkan]
---

Over the past several months I have been learning about and using Vulkan, from writing a basic abstraction over the
Vulkan primitives and objects, to my SPIR-V shader tools library (designed with Vulkan in mind), and now with my work
on something resembling (yet another) Vulkan rendering engine. I'd like to get to those projects as well, as they've all 
taught me a ton: but those projects also were just general programming learning experiences, and I've got no degree/background 
in this field and coding is the only way I've been able to improve.

Regardless, in this post I'd like to walk through Vulkan a bit. Think of it as a supremely abridged version of 
"The Vulkan Programming Guide" (and indeed, I consulted this often while writing this piece) intended to familiarize
and educate those who haven't used this API about it's various elements, realities, and details. It's not substitute
for actually using the API, but I hope it might help prime one's brain with some introductory knowledge and ideas!

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
vector classes (e.g `float3`, `int4`, etc) but no implemented operators for them. These instead had to be
acquired by using a third party header. Which is less than ideal, even if its fairly easy to do.

But what really drew me in was the promise of the complexity of Vulkan. Y'see, I'm the sort of person that 
adores games like the DCS series and gets addons like the PMDG-made series of commercial airliners for 
flight sims. So while some see complex interfaces and arrays of switches and say "__NOPE__", I find myself
resisting the urge to start fiddling with all the dials, knobs, and switches just to figure out how it all works.
And to see what happens when things break. And even maybe finding out _how_ to break things.

Additionally, a compelling argument can now be made for cross-platform support with Vulkan. MoltenVK is now offically
part of the core SDK, and is being maintained even more actively - so using Vulkan on Mac (with a few limitations)
is now absolutely feasible. The Vulkan Portability Initiative is ongoing as well, and Khronos has hinted that a DX12
shim is probably next (and may appear sooner than we expect).

Also, I probably have an issue with self-loathing. I started programming by learning C++ and then decided I was 
going to learn Vulkan. Healthy people don't do these things ;)

## Vulkan objects - common traits

Most objects in Vulkan take some kind of `createInfo` structure pointer in their creation functions. These structures specify 
various attributes of the object that won't (usually) be allowed to change during use. Some of these are simple attributes that 
only the implementation really cares about - like transient and one-time-submit command buffer flags - or are more important, like 
image tiling used for an image object or what a buffer objects usage will be. `sType` and `pNext` members are universal to nearly 
every single structure in  Vulkan. The `pNext` structure is useful for extensions, though: in the case of the ones I've used thus far, 
you supply the vanilla structure to a function but set `pNext` to point to the relevant structure from the extension you're using. 
For 99% of use cases though, this can be set to `nullptr`. Next will be a `flags` field: for some objects this supplies important
additional information (e.g, saying an image will be a cubemap by setting `VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT`). For most objects,
this field is reserved for future use and doesn't yet have any flags/enums defined for it - thus, one can just set it to `0`. 

I found it beneficial to have a header filled with `constexpr` "base" `createInfo` structures though, because if nothing else it'll save
some work in typing the `sType`, `pNext`, and `flags` fields for most of your objects. For others, I originally filled out fields with sane default
values: though as of recently I've chosen to replace potentially dangerous values (or ones that the user should *really* be explicitly setting themselves) to be incorrect from the get-go. This forces a user to then correct the error, as the validation layers (more on those later!) will quickly warn them of their wrongdoing.

Creation and destruction functions also take a `VkAllocationCallbacks*` parameter, which allows a user to define their own custom 
memory allocation functions to use for CPU-side resources. I've not yet seen a major need for this, and honestly have it forced to `nullptr`
in most of my code (though that's something I'd like to change soon). If anyone has seen a major benefit to using these, please let me know. 

## The Core Vulkan objects

### The Instance

In Vulkan, we have 3 objects that I considered the "core" objects: `VkInstance`, `VkPhysicalDevice`, and `VkDevice`. A Vulkan instance is what contains the total "state" of all Vulkan stuff going on. Objects can't be easily shared or passed across instance boundaries (without extensions used for things like cross-API resource sharing, at least). `VkInstanceCreateInfo` is the first create info structure most people will encounter - here we set some application info (name, version, engine name, engine version, API version), enable some instance-level extensions, and set our debug layers up.

The latter bit is one of the best things Vulkan provides - clear and obvious debug messages that can output messages via whatever method the user specifies, if you choose to provide a custom debug callback function to Vulkan. In my case, it just calls a function that uses easylogging++ to get the log messages to the console and to a log file with various levels of logging severity. One can also enable the debug layers through the usage of an environment variable, and a recent extension to this core functionality also makes it possible to make debug callbacks even more powerful - by attaching user data (such as names) to objects, so that they can be more easily tracked.

Specifying the extensions on instance creation (effectively ahead of time), in the words of the the Vulkan Programming Guide, is done because:

>As with OpenGL, Vulkan supports extensions as a central part of the API. However, in OpenGL, we would
>create a context, query the supported extensions, and then start using them. This meant that drivers would
>need to assume that your application might suddenly start using an extension at any time and be ready for it.
>Further, it couldn’t tell which extensions you were looking for, which made the process even more difficult.
>In Vulkan, applications are required to opt in to extensions and explicitly enable them. This allows drivers to
>disable extensions that aren’t in use and makes it harder for applications to accidentally start using
>functionality that’s part of an extension they weren’t intending to enable.

If you've ever had to check for and then use extensions in OpenGL, avoiding this sort of pain probably sounds splendid. Thus far, I'd have to agree.

### Physical Devices

It's probably not hard to divine what exactly a `VkPhysicalDevice` is supposed to represent. What's really especially nice compared to OpenGL is that it lets us iterate over a system's available graphics devices and decide on which one to use. We can also check to make sure that a device supports all the features that we want to use, get info on the queue family layout, check for limits on texture sizes and types, and so on. It helps to get a lot of setup and check code done and out of the way early, instead of trying to check it with branching logic in the hot path we can probably find ways to make these choices much earlier.

`VkPhysicalDevice`'s don't actually require a `createInfo` structure. Instead, we do everything we need to get our available devices like so:

{% highlight cpp %}
uint32_t num_devices = 0;
VkResult result = vkEnumeratePhysicalDevices(our_instance, &num_devices, nullptr);
std::vector<VkPhysicalDevice> devices(num_devices);
result = vkEnumeratePhysicalDevices(our_instance, &num_devices, devices.data());
{% endhighlight %}

The vector shown now contains all of our available physical devices, and we can use it to query features, memory properties, and physical device limits by calling the relevant functions with any of the available device handles in that vector. From there, we can compare and "score" our devices how we choose and ultimately pick whatever one we believe is best for the job. Generally this involves finding the dedicated card on the system (if it exists).

Vulkan also allows for multi-GPU usage - however, without extensions this will involve a lot of work by the developer in manually manage work and resources across multiply physical devices. The `VK_KHR_device_group` and `VK_KHR_device_group_creation` extensions however allow the developer to interface with multiple physical devices as if they were a single device: likely incurring some overhead, but it's almost certainly worth it given how difficult one's work can be otherwise.

##### Queue Families

I earlier hinted about Vulkan exposing us to a devices physical work queues - and indeed it does. `vkGetPhysicalDeviceQueueFamilyProperties` gets us most of what we want to know. First, a queue is sorta akin to a conventional CPU's concept of a thread: it represents objects of similar capabilities, but that can _potentially_ run in parallel. The "potentially" bit is important, as it's what the specification says: there's no guarantee of parallel execution, but I personally feel that it should be seen as the general case. In Vulkan, there are also queue families, of which there are four of thus far:

- Graphics queues: best seen as the base queue type, with the most capabilities. We'll submit drawing commands to these.
- Transfer queues: often an interface to the DMA system, or the most efficient queue through which we can perform memory transfers to the device
- Compute queues: unsurprisingly, what we submit compute jobs to
- Sparse binding queues: used to bind sparse resources (resources that don't have to be backed by memory on the GPU all the time)

Despite their being four unique queue families, it's often more common that a device has a handful of queues capable of doing a bit of everything (or actually everything), then a few dedicated queues per family. Lets take a GTX1070 as an example: it has 16 queues capable of all four types of tasks (the graphics queues), 8 compute queues, and 1 transfer queue. We can submit transfer commands to the 16 general queues, but we should try to use the dedicated transfer queue for large transfers or streaming in resources between frames when possible. 

I also mentioned that the `vkPhysicalDeviceQueueFamilyProperties` struct for a GTX1070 exposes multiple "general" graphics queues, but this is a bit misleading. There currently isn't any hardware I know of that actually supports multiple graphics command queues, so using more than one results in some kind of driver multiplexing and thus some form of overhead. Generally, one should only create a single graphics queue to avoid forcing the driver to multiplex things. 

_However_, multiple compute command queues and can do exist. So feel free to create and use as many of these as you can: its already well-established by IHVs that most applications drastically underutilize the compute capabilities of most dedicated graphics cards anyways. 

### Logical Devices

Logical devices are a software abstraction of a physical device, just configured in a way we specify. Logical devices are what a user of Vulkan will spend most of their time interacting with, and it is ultimately what nearly every object and resource in Vulkan is a "child" of. It's also what actually submits to the queues we just discussed.

The configuration of the physical device is set by the logical device as we decide what features the physical device supports we wish to actually enable - and it's also another place where we enable certain extensions at a device-level scope. Additionally, we decide how many of each queue to actually "create" (i.e, enable) for the logical device we're creating (still limited by the physical device's maximum) and create an array of priorities for each queue - letting us break down in even more detail how queue submissions are handled. This usually isn't necessary, however.

## How well can Vulkan be multithreaded and parallelized?

In short, easily- compared to previous APIs. In OpenGL, threading required complex juggling of OpenGL contexts - somewhat akin to the previous three classes all tied together. But resources on one thread really belonged to a given context so ownership had to be handled carefully and usually required extensions like direct state access and bindless textures (do correct me if I'm wrong, I moved to Vulkan before *really* exploring this). In Vulkan, we don't even have to leave the scope of the logical device to get powerful with multithreading: memory allocation, command submission to queues, resource formatting, command recording, and more can all be multithreaded.

Whereas OpenGL is a series of streets with a sophisticated array of traffic lights, checks, and traffic management systems Vulkan feels like the open road. That does, however, mean that the burden of constructing all this synchronization and safety infrastructure now falls on us as users of the API. If nothing else, the best news is that we don't need to fight the APIs entire history, legacy, and design to use multithreading. 

We'll cover synchronization primitives at the end, and some of those can be used to guard against threading issues and hiccups. I'd like to write another post,however, on the topic as I need to explore it more thoroughly myself. The most apparent (and easiest) way to multithread is just recording rendering commands on a number of threads: one can have "secondary" command buffers that are then executed by a "primary" command buffer. So, a number of threads can be dispatched each with their own secondary command buffer to record into. Once these threads are finished and have all been joined back together, we can gather these secondary command buffers and have them be executed + submitted to a graphics queue from the main rendering thread.

## Vulkan resources 

In OpenGL resource management entailed tracking state, like what vertex buffer or element buffer we had bound at any one moment and making sure we had the right binding set before adjusting a resources parameters or variables. Thankfully, that's all gone in Vulkan. Unfortunately, we gain a new set of responsibilities as we have to manage lifetime, memory, and things like usage flags instead. Memory management is on another level, compared to what one is probably used to coming from OpenGL. And I'm not even really going to be able to dive into sparse resources here, either (resources that don't have to be backed by memory at all times, only when requested/bound).

### Buffers and Images

`VkBuffer` and `VkImage` will most likely be the resources one uses the most, and they are quite different (in good ways!) from how OpenGL handles the notion of these kinds of resources. First, both resources can have their usage precisely specified through mixtures of the [`VkBufferUsageFlags`](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkBufferUsageFlagBits.html)  and [`VkImageUsageFlags`](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkImageUsageFlagBits.html), respectively. A buffer that we're going to use as a vertex buffer gets created with the `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` flag set, and an image we're going to use for sampling in a shader gets `VK_IMAGE_USAGE_SAMPLED_BIT` set.

These items are still "stored" mostly in unsigned integer handles, but there's still a lot more to each resource: instead of selecting and modifying these objects by flipping levers and pressing buttons in the state machine though, we are able to set most of these properties up-front.

#### VkBuffer

Buffers are the simplest type of Vulkan resource we can create, which is reflected in the complexity of their particular `createInfo` structure. They are used to store linear or unstructured data, and don't really have a set format (though they can be viewed as a particular format - more on that later). This is what you will load data into for use in shaders, usually: be it vertex, index, uniform, or mutable compute datasets (or just empty buffers to output compute results into).

#### VkImage

Images are considerably more complex: they require a lot more information to be passed into Vulkan upon creation, and so we'll take the time to break this down and explain a bit about some of the fields in their `createInfo` structure:

{% highlight cpp %}
typedef struct VkImageCreateInfo {
    VkStructureType sType;
    const void* pNext;
    VkImageCreateFlags flags;
    VkImageType imageType;
    VkFormat format;
    VkExtent3D extent;
    uint32_t mipLevels;
    uint32_t arrayLayers;
    VkSampleCountFlagBits samples;
    VkImageTiling tiling;
    VkImageUsageFlags usage;
    VkSharingMode sharingMode;
    uint32_t queueFamilyIndexCount;
    const uint32_t* pQueueFamilyIndices;
    VkImageLayout initialLayout;
};
{% endhighlight %}

There's loads more information in this structure than we're normally used to even having attached to most objects in OpenGL: this is for the best though, as it helps the driver a ton and I've come to appreciate the clarity and verbosity the API requires when setting up objects.

Things like `imageType`, `format`, and `extent` shouldn't be too hard to understand: we effectively already have these in older APIs. `tiling` is a new field,though: it specifies whether the image is stored in an "optimally tiled" format in memory, or in a "linearly tiled" format. Linearly tiled images and data are simple and can be interpreted by the host - it's just good ol fashioned data packing. Optimally tiled images are likely to be packed or stored in a special fashion, based on the current hardware - this can be especially beneficial with textures and image data though, since this storage tiling method may result in increased cache coherency or reduced texel fetch latency. Tiling depends on the format and our planned usage (the `usage` field, set as one or a number of  `VkImageUsageFlags`): we can then use a function like this one in VulpesRender to return the tiling for our image (we always try to return `VK_IMAGE_TILING_OPTIMAL`, since it can be so beneficial).

`sharingMode` (also present in `VkBufferCreateInfo`, but we skipped that structure) is used to potentially allow the usage of this image across multiple queue families, concurrently - `VK_SHARING_MODE_EXCLUSIVE` says we will explicitly handle sharing across queue families ourself (thus not requiring any API work + overhead). `VK_SHARING_MODE_CONCURRENT` says multiple families may be using this resource, in which case we need to specify how many queue families can be using this resource and pass their indices as well.

The last new field is `initialLayout`. 

### Buffer and Image views

While a `VkBuffer` or `VkImage` handle represents the more "raw" object itself, and is usually tied to some sort of backing memory as well, `VkBufferView` and `VkImageView` objects are used 

### Samplers

### Shader Modules

## Device Memory

### Memory residency and types

### Allocating and Binding memory

Allocating and using memory in Vulkan has two distinct steps: the creation step, wherein we request a new chunk of memory from the GPU and "attach" it (so-to-speak) to a new `VkDeviceMemory` handle, and the binding step in which we tie together a resource (`VkBuffer`/`VkImage`) to a region of memory.

### Best Practices (briefly)

## Using Resources - Descriptors

In order to use Vulkan resources in shaders, we must attach them to "descriptors". This ends up acting like a more static version of OpenGL's getting/setting of uniform locations and variables, and it's probably one of the hardest concepts to grasp and begin working with as a new user of the Vulkan API. 

### DescriptorSets

### DescriptorSetLayouts

### DescriptorPools

## Pipelines

### Graphics Pipeline Creation

### Pipeline Cache Objects

A `VkPipelineCache` can be passed to the creation function for pipelines, and will be used to accelerate pipeline creation when possible. This is *especially* important on Mac OSX w/ MoltenVK: the pipeline creation step usually involves some kind of code generation and shader compiliation, but when using MoltenVK its when the SPIR-V binary blobs attached to a pipeline are converted to Metal shader code. This generated shader code is then stored/attached to the pipeline cache object, and can be used again when generating similar pipelines.

One can take the mechanism further, caching results of pipeline creation across application executions by [saving the pipeline data to file](https://github.com/fuchstraumer/VulpesRender/blob/master/src/resource/PipelineCache.cpp#L128). Then, one can later load this pipeline cache data back from the stored file data - passing through a verification step ([example](https://github.com/fuchstraumer/VulpesRender/blob/master/src/resource/PipelineCache.cpp#L60)) to verify the pipeline cache is valid according to a few important qualifiers - and the pre-existing data can be used as a base to build from or mutate from. You'll need some kind of UUID for this: I personally just pass a `size_t` "hash" code to the constructor. Currently I generate this UUID by hashing the `typeid` of whatever object is generating the pipeline object, but this is a dated idea based on my previous thoughts on renderer design. Ultimately, the UUID can be generated however the user desires.

If you check my personal implementation of a pipeline cache object, you'll see the simplest way implement this behavior in C++: have the destructor of the object automatically dump the pipeline cache contents to a file, and have the constructor search for pre-existing file to initialize the cache with. One can also merge multiple caches together into one new `VkPipelineCache` handle, which can be especially useful if one is threading the generation of multiple subtly varying pipeline objects.

## The Swapchain

One of the other entirely new responsibilities required of the application writer when using Vulkan is swapchain management - a responsibility that is usually not a consideration at all with other graphics APIs. 

### Creation and Management

#### Further reading

### Presentation Surfaces

### Framebuffers

## RenderPasses

### Subpasses

## Queries

## Synchronization

There are three 
