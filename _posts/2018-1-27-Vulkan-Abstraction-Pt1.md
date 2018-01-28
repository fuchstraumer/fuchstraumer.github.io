---
layout: post
date: 2018-1-27
title: "Abstracting Vulkan with C++ Part 1: Background"
img: yosemite1.jpg
published: true
tags: [Vulkan,C++,Design,Libraries, Modern C++]
---

Over the past several months I have been working on a Vulkan abstraction layer - in this
post, I'd like to go through the process by which I've built this library: from it's 
barest beginnings, to it's awkward "lets do everything in one library" middle-stage, to
what I believe is now a well-designed and clearly defined abstraction library.

This will be a series of sorts, and this first post is designed to run through almost the 
entire Vulkan API as fast I possibly can. 

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

## Vulkan objects - common traits

Most objects in Vulkan take some kind of `createInfo` structure pointer in their creation functions. These structures specify 
various attributes of the object that won't (usually) be allowed to change during use. Some of these are simple attributes that 
only the implementation really cares about - like transient and one-time-submit command buffer flags - or are more important, like 
image tiling used for an image object or what a buffer objects usage will be. `sType` and `pNext` members are universal to nearly every single structure in Vulkan. The `pNext` structure is useful for extensions, though: in the case of
the ones I've used thus far, you supply the vanilla structure to a function but set `pNext` to point to the relevant structure from the 
extension you're using. For 99% of use cases though, this can be set to `nullptr`.

Creation and destruction functions also take a `VkAllocationCallbacks*` parameter, which allows a user to define their own custom 
memory allocation functions to use for CPU-side resources. I've not yet seen a major need for this, and honestly have it forced to `nullptr`
in most of my code (though that's something I'd like to change soon). If anyone has seen a major benefit to using these, please let me know.

## The Core Vulkan objects

### The Instance

In Vulkan, we have 3 objects that I considered the "core" objects: `VkInstance`, `VkPhysicalDevice`, and `VkDevice`. A Vulkan
instance is what contains the total "state" of all Vulkan stuff going on. Objects can't be easily shared or passed across
instance boundaries. `VkInstanceCreateInfo` is the first create info structure most people will encounter - here we set some application 
info (name, version, engine name, engine version, API version), enable some instance-level extensions, and set our debug layers up.

The latter bit is one of the best things Vulkan provides - clear and obvious debug messages that can output messages via whatever method the user
specifies, if you choose to provide a custom debug callback function to Vulkan. In my case, it just calls a function that uses easylogging++ to
get the log messages to the console and to a log file with various levels of logging severity. 

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

It's probably not hard to divine what exactly a `VkPhysicalDevice` is supposed to represent. What's really especially nice compared to OpenGL is that
it lets us iterate over a system's available graphics devices and decide on which one to use. We can also check to make sure that a device supports all
the features that we want to use, get info on the queue family layout, check for limits on texture sizes and types, and so on. It helps to get a lot of
setup and check code done and out of the way early, instead of trying to check it with branching logic in the hot path we can probably find ways to make
these choices much earlier.

`VkPhysicalDevice`'s don't actually require a `createInfo` structure. Instead, we do everything we need to get our available devices like so:

{% highlight cpp %}
uint32_t num_devices = 0;
VkResult result = vkEnumeratePhysicalDevices(our_instance, &num_devices, nullptr);
std::vector<VkPhysicalDevice> devices(num_devices);
result = vkEnumeratePhysicalDevices(our_instance, &num_devices, devices.data());
{% endhighlight %}

The vector shown now contains all of our available physical devices, and we can use it to query features, memory properties, and physical device limits by
calling the relevant functions with any of the available device handles in that vector. From there, we can compare and "score" our devices how we choose
and ultimately pick whatever one we believe is best for the job.

##### Queue Families

I earlier hinted about Vulkan exposing us to a devices physical work queues - and indeed it does. `vkGetPhysicalDeviceQueueFamilyProperties` gets us most 
of what we want to know. First, a queue is sorta akin to a conventional CPU's concept of a thread: it represents objects of similar capabilities, but that
can _potentially_ run in parallel. The "potentially" bit is important, as it's what the specification says: there's no guarantee of parallel execution, but 
I personally feel that it should be seen as the general case. In Vulkan, there are also queue families, of which there are four of thus far:

- Graphics queues: best seen as the base queue type, with the most capabilities. We'll submit drawing commands to these.
- Transfer queues: often an interface to the DMA system, or the most efficient queue through which we can perform memory transfers to the device
- Compute queues: unsurprisingly, what we submit compute jobs to
- Sparse binding queues: sparse binding is the topic I know the least about, but these are the queues that are the most suited to that sort of work

Despite their being four unique queue families, it's often more common that a device has a handful of queues capable of doing a bit of everything (or actually everything),
then a few dedicated queues per family. Lets take a GTX1070 as an example: it has 16 queues capable of all four types of tasks, 8 compute queues, and 1 transfer queue. We
can submit transfer commands to the 16 general queues, but we should try to use the dedicated transfer if possible - and similarly for compute jobs.

### Logical Devices

Logical devices are a software abstraction of a physical device, just configured in a way we specify. Logical devices are what a user of Vulkan will spend
most of their time interacting with, and it is ultimately what nearly every object and resource in Vulkan is a "child" of. It's also what actually submits to the 
queues we just discussed.

The configuration of the physical device is set by the logical device as we decide what features the physical device supports we wish to actually enable - and it's
also another place where we enable certain extensions at a device-level scope. Additionally, we decide how many of each queue to actually "create" (i.e, enable) for
the logical device we're creating (still limited by the physical device's maximum) and create an array of priorities for each queue - letting us break down in even more 
detail how queue submissions are handled.

### How does this all affect threading?

In short, positively. In OpenGL, threading required complex juggling of OpenGL contexts - somewhat akin to the previous three classes all tied together. But resources on one thread 
really belonged to a given context so ownership had to be handled carefully and usually required extensions like direct state access and bindless textures (do correct 
me if I'm wrong, I moved to Vulkan before exploring this). In Vulkan, we don't even have to leave the scope of the logical device to get powerful with multithreading:
memory allocation, command submission to queues, resource formatting, command recording, and more can all be multithreaded.

Whereas OpenGL is a series of streets with a sophisticated array of traffic lights, checks, and traffic management systems Vulkan feels like the open road. That does, however, mean that
the burden of constructing all this synchronization and safety infrastructure now falls on us as users of the API. If nothing else, the best news is that we don't need
to fight the APIs entire history, legacy, and design to use multithreading.

## Vulkan resources 

In OpenGL resource management entailed tracking state, like what vertex buffer or element buffer we had bound at any one moment and making sure we had the right
binding set before adjusting a resources parameters or variables. Thankfully, that's all gone in Vulkan. Unfortunately, we gain a new set of responsibilities
as we have to manage lifetime, memory, and things like usage flags instead.

### Images and Buffers

Unlike OpenGL, there are no explicit texture objects in Vulkan - 


### Buffers