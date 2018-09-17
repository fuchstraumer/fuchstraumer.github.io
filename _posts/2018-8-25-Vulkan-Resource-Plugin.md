---
layout: post
date: 2018-8-25
title: "Making a Vulkan Resource Plugin in C++"
img: river_mountain.jpg
published: true
tags: [C++, Engine Development, Vulkan, Plugins]
---

# Creating a Vulkan Resource-Management plugin

Last time, I covered the creation of a plugin for parallelizing the loading of assets from disk - this time, I'll be covering a plugin that plays very well with that system. Management of Vulkan API resources can be a tough topic: I've attempted to solve this problem about a dozen different ways by now, but I'm most content with this recent approach. When I refer to Vulkan resources I'm considering primarily `VkBuffer` and `VkImage`, but I also decided to integrate and support `VkSampler` in this system as well. 

Like last time, this will be integrated as a plug-in for `Caelestis`, which can be found [here](https://github.com/fuchstraumer/Caelestis/commit/a0a486f6c9618e69b0061bd1a78d861de2424b0f). This is the `resource_context` plugin.

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
- Transferring data to the resources should be performed asynchronously and without user involvement
- Cleaning up and managing the lifetime of things like staging buffers and their data must be done by the system, not the user

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

The parameters that I haven't directly addressed yet are those used to set the initial contents of the buffers - `gpu_resource_data_t` is a structure that holds a pointer to some data along with the size of said data, along with some memory alignment and pitch information members as required. I unfortunately found out that a seperate, and more detailed, structure `gpu_image_resource_data_t` is effectively required in order for us to be able to set the initial data of images, however. We'll return to that shortly, however. The rest of the parameters should be fairly apparent - if we pass a `nullptr` for a `view_info` parameter, that resource's `ViewHandle` field won't be set as the view object won't be created (this will be pretty common with buffers, for example, unless we're using them as texel buffers).

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

But there are issues when this is applied to images: we can't, for example, figure out how many MIP levels we have. Even if we passed in an array of these initial data structures, we couldn't back out this sort of info. Backing out things like the width and height is doable, but we'd still be missing other fields like "Array Layers" and so on: so it's easiest to just add this unique structure for handling image data uploads:

{% highlight cpp %}
struct gpu_image_resource_data_t {
    const void* Data = nullptr;
    size_t DataSize{ 0 };
    uint32_t Width{ 0 };
    uint32_t Height{ 0 };
    uint32_t ArrayLayer{ 0 };
    uint32_t NumLayers{ 1 };
    uint32_t MipLevel{ 0 };
};
{% endhighlight %}

Via usage of this structure, we are now able to rather easily set the contents of everything from simple 1D linearly-tiled images made up of barely thousands of pixels, to uploading complex images with multiple mip levels like cubemaps or array textures. While it isn't *great* that we have two barely-differing structures that serve effectively the same purpose, the advantages of this approach are rather apparent and absolutely worth any minor code issues they may cause down the road. This brings us into our next major chunk of functionality though - which will be used to transfer data into our resources.

### Step 3: Transferring Resource Data to the GPU

Another area that can become potentially tricky is the transferring of data to the GPU: in Vulkan, we have to manage these operations at a level of complexity that is effectively unheard of in earlier DirectX versions and in OpenGL. First, there are two types of memory in Vulkan:

- Host visible
- Device local

Our logic for handling transfers is split based on these memory types (which is set by the `memory_type` parameter of our creation functions): host-visible memory is much easier to transfer into, as it's usually a designated region of RAM that the CPU and GPU share access to, and writing into it requires much less work. Effectively, it becomes: (assuming we're not using the `HOST_COHERENT` flag here, for performance reasons)

- If we're not worried about readbacks (can be parsed from usage flags), skip mapped memory region invalidation and just map the region we wish to copy into
- Perform a simply `memcpy` to copy into the desired memory region, potentially using a simple `for` loop + an offset if we have an array of `gpu_resource_data_t` structures
- Make sure to unmap the region once our transfer is complete
- Lastly, call a Vulkan "flush" command to ensure our writes from the CPU are visible to the GPU as well

This copy process is not complex at all - but the process of copying into device-local memory is considerably more involved. 

#### Transfer System Infrastructure

First, we have to create "staging" buffers for our data: these are literal `VkBuffer` objects that we will create purely for transferring data out of (into the device-local buffers we plan to actually use). In order to wrap this functionality up in a clean way, so that I could track these staging buffers more easily, I created the following `UploadBuffer` structure:

{% highlight cpp %}
struct UploadBuffer {
    UploadBuffer(const vpr::Device* _device, vpr::Allocator* alloc, VkDeviceSize sz);
    void SetData(const void* data, size_t data_size, size_t offset);
    VkBuffer Buffer;
    vpr::Allocation alloc;
    const vpr::Device* device;
};
{% endhighlight %}

We use our constructor to quickly create `vpr::Allocation` and `VkBuffer` objects for use as staging buffers - and since we will so clearly know what the various flags and parameters we need to construct these will be, the only parameter that really varies is the size of the allocation itself. From there, we call `SetData` with a data pointer, a size, and an offset into the staging buffer as many times as we need to fully populate it's contents.

There's no destructor explicitly defined, but the copy constructor and copy assignment operator have been explicitly deleted - creating a copy of these objects could prove disastrous, and having these implicitly destroyed is not quite what we want (as unfortunate as it is, because it does make things a bit messy). We store all of our upload buffers in a single container, and these are cleared per-frame after the various transfers they are used for have been completed. By doing these all at once we are also able to make sure that the memory attached is properly freed, and that the created `VkBuffer` gets destroyed too.

The transferring of data from the `UploadBuffer` structures into the actual destination resources is done by the `TransferSystem` - itself a thin wrapper over a `vpr::CommandPool` and some thread-safety related items (addressed later). Once we have our upload buffer populated, we use the `TransferSystem` to retrieve a `VkCommandBuffer` that we can record transfer commands into like so:

{% highlight cpp %}
auto cmd = transfer_system.TransferCmdBuffer();        
const VkBufferCreateInfo* p_info = 
    reinterpret_cast<VkBufferCreateInfo*>(resource->Info);
const VkBufferMemoryBarrier memory_barrier0 {
    VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER,
    nullptr,
    0,
    VK_ACCESS_TRANSFER_WRITE_BIT,
    VK_QUEUE_FAMILY_IGNORED,
    VK_QUEUE_FAMILY_IGNORED,
    reinterpret_cast<VkBuffer>(resource->Handle),
    0,
    p_info->size
};
vkCmdPipelineBarrier(cmd, VK_PIPELINE_STAGE_ALL_COMMANDS_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 
    0, 0, 0, 1, &memory_barrier0, 0, nullptr);
std::vector<VkBufferCopy> buffer_copies(num_data);
VkDeviceSize offset = 0;
for (size_t i = 0; i < num_data; ++i) {
    buffer_copies[i].size = initial_data[i].DataSize;
    buffer_copies[i].dstOffset = offset;
    buffer_copies[i].srcOffset = offset;
    offset += initial_data[i].DataSize;
}
vkCmdCopyBuffer(cmd, upload_buffer->Buffer, reinterpret_cast<VkBuffer>(resource->Handle), 
    static_cast<uint32_t>(buffer_copies.size()), buffer_copies.data());
const VkBufferMemoryBarrier memory_barrier1 {
    VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER,
    nullptr,
    VK_ACCESS_TRANSFER_WRITE_BIT,
    accessFlagsFromBufferUsage(p_info->usage),
    VK_QUEUE_FAMILY_IGNORED,
    VK_QUEUE_FAMILY_IGNORED,
    (VkBuffer)resource->Handle,
    0,
    p_info->size
};
vkCmdPipelineBarrier(cmd, VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_ALL_COMMANDS_BIT, 
    0, 0, nullptr, 1, &memory_barrier1, 0, nullptr);
{% endhighlight %}

There's a lot to process here, so lets go through it step by step. First, we retrieve the `VkBufferCreateInfo` belonging to our destination resource: from this, we get the size of the buffer so that we can create an appropriate `VkBufferMemoryBarrier` to protect the resource from incorrect or improper writes. After executing/using this memory barrier, we then proceed to generate the series of `VkBufferCopy` structures we shall need to perform a full copy between the staging resource and the destination resource: in this case, we just need to specify the size of each "chunk" of data being uploaded, and our offset into both the source and destination buffers is just how much we've transferred thus far.

From there, we execute `vkCmdCopyBuffer` to finalize the data transfer. We close out with another `VkBufferMemoryBarrier` structure being created - with a minor change. We need to specify the "access flags" in each memory barrier, so that the API knows how we plan to access this buffer after the transfer is complete (which can be important to the driver, for one thing). We don't have any easy way to have the user specify these access flags however, and we can't just settle on some massive bitfield of all potential uses (could be a nightmare for performance, like using any of the `GENERAL` usage/access flags). Instead, we `accessFlagsFromBufferUsage` to back out our likely access flags from the usage flags set in the `VkBufferCreateInfo` structure we retrieved earlier.

Luckily, identifying our access flags using the `VK_BUFFER_USAGE_` series of flags is really quite easy - there's a nearly 1-to-1 mapping between the two in this case, so a simple if-else if series of statements is all we ultimately need. For our default case, we just `VK_ACCESS_MEMORY_READ_BIT` - and if we're incorrect, it's likely that the Vulkan validation layers will help us identify a correct setting and amend the issue quickly.

#### Copying Into Device-Local Memory for Images

The above showed most of the process, and some of the infrastructure required, for copying data into `VkBuffer`s. The process for images is nearly the same with only a few minor tweaks: first, we use a `VkBufferImageCopy` structure (unsurprisingly) in place of our `VkBufferCopy` structure we previously used. These structures require the mip level and array layer of a given copy - meaning that our addition of a `gpu_image_resource_data_t` structure containing these fields was just as important as we initially thought. 

Second, using memory barriers becomes more important, especially using a barrier to complete the transfer. In Vulkan, images have specified "layouts" that affect how they may be used by the GPU and API: oftentimes, this information can be used by the driver to change where it places an image in memory or how it prepares it for optimal access. Memory barriers are used on images to transfer them between layouts, and it is ultimately required that we get an image into a transfer-optimal layout (`VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`) before trying to write into it: in which case, the write may fail entirely or we may see bizarre results in the final image (and the validation layers will *absolutely* make us aware of our mistake). Once done transferring, then, it is vital that we transfer the image out of a transfer-only layout and into the layout that reflects how it will ultimately be used.

{% highlight cpp %}
const VkImageMemoryBarrier barrier1{
    VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER,
    nullptr,
    VK_ACCESS_TRANSFER_WRITE_BIT,
    accessFlagsFromImageUsage(info->usage),
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    imageLayoutFromUsage(info->usage),
    device->QueueFamilyIndices.Transfer,
    device->QueueFamilyIndices.Graphics,
    reinterpret_cast<VkImage>(resource->Handle),
    VkImageSubresourceRange{ VK_IMAGE_ASPECT_COLOR_BIT, 0, 
        info->mipLevels, 0, info->arrayLayers }
};
{% endhighlight %}

`accessFlagsFromImageUsage` is the same as our similarly-named method for buffers - we just use the `VkImageUsageFlags` field of the resource's `VkCreateInfo` structure to back out the likely access flags, and hope that any errors can be caught by the validation layers for fixes down-the-road. `imageLayoutFromUsage` performs a similar task, and thankfully it ends up being even simpler than the access flags - there are only a few possibilities, and in worst-case scenarios we can fall back on the `VK_IMAGE_LAYOUT_GENERAL` flag. However, this layout comes with potential performance implications - so we make sure to log a relatively high-priority warning to the console when this occurs, so developers or users can amend whatever cause the code to choose this as a valid setting.

### Step 4: Destroying Vulkan Resources

With our resources created and their initial data set, we can then freely proceed to use them - and once done, destroy them. Thankfully, this is probably one of the simplest things we're going to implement in this entire system - the signature of the function is literally just:

{% highlight cpp %}
void DestroyResource(VulkanResource* rsrc);
{% endhighlight %}

Internally, we use a switch case to read the `type` field of our resource structure, and then call a unique function for destroying our various types. In the case of buffers and images, we also perform additional checks to see if we created and `view` members and make sure these get destroyed, and lastly finish be making sure to call `allocator->FreeMemory()` to free and release the memory used by the resource as well. The only potential issue is making sure we aren't erasing out of containers while other threads are adding new entries, so we quickly nab the mutex and lock it via a `std::lock_guard` so we can safely erase without invalidating iterators elsewhere. 

Lastly, resource destruction isn't *required*, precisely. It's really really recommended, but someone forgetting to destroy a resource won't break the plugin or whatever system is using it. The resource plugin subscribes to swapchain resize and recreation notifications from the renderering context plugin, and as such has to destroy all of it's resources when that occurs (meaning clients have to take care of re-creating them). Additionally, we just clear out our container on shutdown too. I've considered changing this design, and would be open to input or suggestions on it - should I log warnings when resources are left remaining, thus effectively making it "bad" behavior to not delete resources? I'm not quite sure.

### Thread-Safety

Speaking of issues with threading, though, lets tackle the various ways we can protect ourselves from these problems and the various systems that had their design affected. First, the `TransferSystem` mentioned earlier comes to mind: within it, I implemented a basic spinlock based on work I saw in [WickedEngine](https://github.com/turanszkij/WickedEngine/blob/bb10f92b7cb6269c8a46edd2dcd9abe00d25d267/WickedEngine/wiSpinLock.h). My version is effectively the same, but with a minor twist: instead of explicitly using the `lock()` and `unlock()` functions, a dangerous proposition for my absent-minded self, I created a `transferSpinLockGuard` structure. This structure serves the same purpose as a `std::lock_guard` - attempting to acquire the spinlock on construction, and releasing it upon destruction. The structure itself is *remarkably* simple and I feel much more confident and safe using it than the raw object itself:

{% highlight cpp %}
struct transferSpinLockGuard {
    transferSpinLock& lck;
    transferSpinLockGuard(transferSpinLock& _lock);
    ~transferSpinLockGuard();
};

transferSpinLockGuard::transferSpinLockGuard(transferSpinLock& _lock) : 
    lck(_lock) {
    lck.lock();
}

transferSpinLockGuard::~transferSpinLockGuard() {
    lck.unlock();
}
{% endhighlight %}

Barely a dozen lines of code allows us to use our spinlock in a nice and safe manner: we then acquire the spinlock from the transfer system itself right before we begin recording transfer commands:

{% highlight cpp %}
// Leverage RAII to our advantage more: use 
// brackets to create a new scope. Record 
// transfer commands in brackets, spinlock
// released ASAP when we exit brackets again!
{
    auto& transfer_system = 
        ResourceTransferSystem::GetTransferSystem();
    auto guard = transfer_system.AcquireSpinLock();
    auto cmd = transfer_system.TransferCmdBuffer(); 
    // record transfer commands as per the usual
}
{% endhighlight %}

Ultimately, I will admit that the choice to use a spinlock was a personal one - I had yet to use one at all, and it seemed well-suited to my purpose. Using a mutex would've gotten the job done as well, but since I expect our locking times (and thus waiting times for other threads) to remain short, I believe that it won't end up being a *bad* choice, per-se, in the long run. 

Thread-safety for container insertion (for containers used for storing our `VulkanResource` objects, and more) is done in the more standard manner using `std::mutex` and `std::lock_guard`. Where possible, however, the code was structure so that container modifications - i.e, adding or removing entries - was done in batches. Then, pointers are quickly retrieved to the added elements so that we can release the mutex and proceed onwards with our work. This so far has allowed for the fluent creation and destruction of resources from multiple threads - and, the `vpr::Allocator` and memory allocation system therein is already thread-safe as well. 

### Transfer System, Continued

So far I've mentioned the transfer system briefly and not in terrible detail - but there are a few more details worth going into. Initial testruns showed that I was spending quite a bit of CPU time calling `vkQueueSubmit` through the transfer system - even though there weren't any actual transfers occuring, as the singular `VkCommandBuffer` we were using didn't have any transfers recorded into it. This occured as our plugin has a function called `LogicalUpdate` that is called every frame, at the beginning of the frame, without any delta-time information: it's intended for internal plugin state updating and the like, so in the case of this plugin I had placed a `TransferSystem::CompleteTransfers` call and a call to flush our staging buffers within it (so, we transfer all our data then clear all the `UploadBuffer` structures we created for these transfers).

The fix ended up being really quite simple: set a flag to mark the attached command buffer as dirty, which is only set if `TransferSystem::TransferCmdBuffer()` is called. Then, when `TransferSystem::CompleteTransfers` we can check to see if this flag has been set - if so, we have work to submit and we proceed with a call to `vkQueueSubmit`. After that, we reset our flag to mark the command buffer as unused. This small change helped save a considerable amount of CPU time - it may not have been that large of an issue, but it seemed like something worth addressing regardless.

### Conclusion

Hopefully this post helps with insight into how to design a similar system - managing resources in the new explicit graphics APIs is not the simplest challenge to confront, and it's fairly easy to back oneself into a corner (as I did, with VulpesRender initially) without considering potential issues down the road. As always, I welcome criticism, questions, and comments: I already have a fairly large suite of improvements I'd love to make, and I'm sure there are things I've done that could be done better (so please, let me know!). The source code of the relevant plugin can be found as part of the `resource_context` plugin in Caelestis, [here](https://github.com/fuchstraumer/Caelestis/tree/master/plugins/resource_context). The relevant source files will be "TransferSystem", "ResourceTypes", and "ResourceContext". If you're curious how the API for this plugin looks, that will simply be in `ResourceContextAPI.hpp`.

### Potential Further Work

The following are just various ideas I've had for improving this library, but haven't gotten to yet - in practice, the current version works more than well enough for me. Additionally, I've been pressed for time lately and just haven't had the ability to tune this plugin to it's peak as much as I tune everything else I write:

- See if things like [Slim Reader/Writer locks](https://docs.microsoft.com/en-us/windows/desktop/Sync/slim-reader-writer--srw--locks), or critical sections and the corresponding Unix utilities, could be used to improve performance of the thread-safety related code
- Interface this plugin with ShaderTools, so that this plugin can automatically create the backing resources required for a shader or shader group in one go
- Make sure that submitting transfer work to a unique/dedicated transfer queue actually works, and doesn't cause ownership issues with the Vulkan API when said resources are used by the graphics queue
