---
layout: post
date: 2018-9-2
title: "Creating and Designing a Shader Toolchain"
img: aurora.jpg
published: true
tags: [C++, tools, Vulkan, Shaders, SPIR-V, Lua]
---

**Note: This article is a WIP! If you see it, it's probably because I'm working on it at this very moment...**

As an early pre-article note: I have written various versions of this article about three times now. ShaderTools has evolved *so* much from it's initial versions, and is constantly improving, so it was hard to ever feel it was "ready" to share. Or, I'd write half the content I needed then realize that (usually) I had written some new feature or fix that solved a problem I mentioned in the writing - like the resource group system. That has gone through so many variations and iterations that I could write a whole article *just* on that content. So, here goes nothing: and hopefully, this ages well.

# Toolchains: the Reality of Engine Development

The title just above is not a pessimistic statement - but it is the truth! Before I really dove into the challenge of attempting to make my own graphics engine, I had little idea of creating things like toolchains. My previous work projects had never required them, or weren't large enough to benefit from them. But a few early experiments with trying to implement clustered forward rendering (and looking at [volumetric tiled forward](https://www.3dgep.com/volume-tiled-forward-shading/) made it even more urgent I get this sort of system working) made it clear that I needed some help - in the form of code designed to help bridge the gap and make my life easier.

With Vulkan shaders, one has to declare all the resources in a much more explicit manner. It's not just specifying the range of limited locations you might bind textures to: no, it's specifying where you bind all resources to. And not just a singular `binding` field either: since Vulkan resources are declared in "sets", that's another field that has to be specified. For some of the more complicated shaders used in the fancier rendering pipelines nowadays, you get fairly ridiculous resource binding declarations like this:

{% highlight glsl %}
// Resource block: Lights
layout (set = 1, binding = 0, rgba8) readonly restrict uniform imageBuffer lightColors;
layout (set = 1, binding = 1, rgba32f) restrict uniform imageBuffer positionRanges;
// End resource block

// Resource block: ClusteredForward
layout (set = 2, binding = 0, r32ui) restrict uniform uimageBuffer lightCountTotal;
layout (set = 2, binding = 1, r32ui) restrict uniform uimageBuffer lightBounds;
layout (set = 2, binding = 2, r8ui) restrict uniform uimageBuffer Flags;
layout (set = 2, binding = 3, r32ui) restrict uniform uimageBuffer lightList;
layout (set = 2, binding = 4, r32ui) restrict uniform uimageBuffer lightCounts;
layout (set = 2, binding = 5, r32ui) restrict uniform uimageBuffer lightCountOffsets;
layout (set = 2, binding = 6, r32ui) restrict uniform uimageBuffer lightCountCompare;
// End resource block

// Resource block: ObjMaterials
layout (set = 3, binding = 0) uniform _Material_ {
    // whole bunch of members
} Material;

layout (set = 3, binding = 1) uniform sampler2D normalMap;
layout (set = 3, binding = 2) uniform sampler2D metallicMap;
layout (set = 3, binding = 3) uniform sampler2D roughnessMap;
layout (set = 3, binding = 4) uniform sampler2D diffuseMap;
// End resource block
{% endhighlight %}

This is extremely gross. We don't want to write this, and we're (assumedly) people who are fairly used to writing shaders at a lower-level like this. This is *absolutely* not the kind of interface we want to begin exposing to artists and effects designers, and is not the sort of detail we want to them worrying about anyways. And it gets worse: since Vulkan uses so much just *more* information and data about it's resources and where they're bound, we have to do things like the following when creating the API objects that correspond to shader resource bindings:

{% highlight cpp %}
// texelBuffersLayout is a vpr::DescriptorSetLayout, used to describe the layout of a single 
// descriptor set's resources. This layout is for the "ClusteredForward" resource block.
constexpr static VkShaderStageFlags fc_flags = VK_SHADER_STAGE_FRAGMENT_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 0);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 1);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 2);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 3);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 4);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 5);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 6);
texelBuffersLayout->Create();
{% endhighlight %}

And the gift keeps giving: Vulkan descriptor sets are allocated from `VkDescriptorPool`s. Creating these pools effectively requires forecasting, ahead of time, how many of each of the Vulkan `VK_DESCRIPTOR_TYPE_` resources you plan to use: if you try to allocate for 12 texel buffers, but you only specified room for 6, the API will throw errors and things will break really quick. And what if you change some of the bindings around? The above code has bindings 0-6 for texel buffers - but if we modify the shader to add a uniform buffer at binding 0, we have to make sure our code reflects that (both by changing the above bindings, and making sure our `VkDescriptorPool` has room). It's a lot of work to manage, and it's just not easy to keep sane with all this going on. 

So these issues motivated the initial work into ShaderTools: let's take our pre-existing shaders, and just reflect back upon them to generate all this binding metadata at runtime (as no such mechanism exists in Vulkan, like it did in OpenGL). 

## Shader Reflection - the Beginnings

If one wants to perform reflection on SPIR-V shaders, you're going to have to use the [spirv-cross](https://github.com/KhronosGroup/SPIRV-Cross) library. By feeding it a SPIR-V binary blob, one can query the library and get access to *all* of the resources used by a shader. From here, it's only a matter of collating this data across multiple stages (usually just vertex + fragment) and generating the relevant data: in initial versions of my library, the literally meant just generating the arrays of `VkDescriptorSetLayoutBinding`s one would require for a given combination of shaders. 

{% highlight cpp %}
const size_t num_descriptor_sets = reflectionSystem->GetNumDescriptorSets();
std::vector<std::vector<VkDescriptorSetLayoutBinding>> setBindings(num_descriptor_sets);
for (size_t i = 0; i < num_descriptor_sets; ++i) {
    size_t num_bindings = 0;
    reflectionSystem->GetLayoutBindings(i, &num_bindings, nullptr);
    setBindings[i].resize(num_bindings);
    reflectionSystem->GetLayoutBindings(i, &num_bindings, setBindings[i].data());
}
// assuming we have some VkDescriptorSetLayout wrapper class, we then just add the bindings:
ourDescriptorSetLayout.AddSetLayoutBindings(setBindings[0]);
// (repeat the above for each descriptor set and set layout we retrieved data for)
{% endhighlight %}

Further, by tracking this data at a slightly higher level and between multiple invocations of this "reflection" system, one could get an accurate count of the resources required - so setting up your `VkDescriptorPool` to perfectly match the requirements of the current suite of shaders became nearly trivial! It had the effect of reducing code complexity *and* making things much more flexible - effectively, we had incidentally moved to a data-driven design where the "data" was our shaders. This data was now affecting the behavior of our code, and our code adapted to the data instead of the other way around (or just a lack of adaptation whatsoever).

### Potential Issues

However, there is one problem I have not yet solved. One still has to make sure that compatability between shader resources is maintained between all resources that may use a given set of resources: for example, let's say I perform the binding shuffling I briefly mentioned earlier (where I suggested adding a uniform buffer). I can't just modify a single shader doing that - I have to then modify all shaders that bind to that particular descriptor set. Sure, I could create an all-new unique descriptor set just for that single shader: but, descriptor set binding is one of the more expensive operations we can perform in Vulkan so it's more ideal for use to keep our descriptors pooled, to reduce binding changes. But copying and pasting code around, and remembering to always do so, is less than ideal as well. So, how do we fix this?

#### A Brief Aside: spirv-cross

I'd like to take a moment to quickly speak positively of `spirv-cross` and it's maintainers, especially Hans: I created [an issue](https://github.com/KhronosGroup/SPIRV-Cross/issues/551) explaining a lack of documentation or examples on using the API. I had personally figured out how to use the library for reflection, but it had taken a fair bit of experimentation and laborious crawling through the code to do so. I mentioned this on reddit and Hans suggested I create the issue I did, and after doing so he got right on it and created the rather immensely useful [examples for reflection](https://github.com/KhronosGroup/SPIRV-Cross/wiki/Reflection-API-user-guide) that one can now find in the wiki. Point of this sidebar being: take the time to speak to people about their libraries or projects! If you feel its lacking documentation or examples, the best way you can let someone know is by creating an issue on github in my opinion. Give it a shot!

## Shader Generation

In order to get around this problem, we can move to a generative shader approach: this makes plenty of sense, and is one that most shader toolchain software packages end up using anyways. As I am singular person, however, I'll avoid the highly abstracted approach of Unity and Unreal Engine: as great as that can be, the scope of work required is simply far too immense for me to manage among all my other work.

So, my goal became generating just the resource declarations per-shader: users will still have to write the `main()` block as they would otherwise, but dealing with the intricacies and irritations of Vulkans resource binding model would be effectively removed. Additionally, we could take this chance to add simple things like simpler handling of specialization constants, `#include` support, and other various pre-processor features (as we are now writing a GLSL preprocessor) all to reduce the difficulty of writing shaders. So, let's get to work.

#### Specialization Constants

I'm going to skip over detailing how I implemented `#include` support - it's nothing too shocking to anyone who's been doing development work for a while, so I'll just skip right into something more interesting. Vulkan has these interesting objects called "specialization constants" - constant values in the shader that are bound to specified locations, a bit like descriptor resources. What makes them unique, however, is that the value can be modified at pipeline creation time. This is fairly powerful, as it lets you write one shader then potentially vary the behavior shortly before you use it: potentially allowing for the "generation" of shader permutations and variations at runtime. I personally tend to use it for holding values like the screen size, and other environmental constants that don't frequently change: when they do change (e.g, during a swapchain recreation event) we would have to recreate our pipelines anyways - allowing us to update the value to reflect the new screen size, for example.

However, it can be a bit annoying and tedious to type out `layout (constant_id = (idx))` for each of the specialization constants we intend to use. Additionally, I figured (correctly!) that practicing on this feature would help me prepare for the more difficult world of descriptors and those resources. Currently, one declares a specialization constant quite simply: `SPC (TYPE) (NAME) = (VALUE)` will do the trick. From there, the shader generation system adds the requisite `layout (constant_id = (idx))` prefix as appropriate, keeping a running tally of the current index to use per-shader. 

#### Simplifying Resources With Resource Groups

So as was noted earlier, Vulkan resources are attached to descriptor sets: these can be thought of as a sort of semantic or logical "grouping" of these resources as well. First, it's a good idea to keep resources with similar update frequencies together in descriptor sets: this way, for example, you could bind your global per-frame data to descriptor set 0 for example, then switch out higher bindings per rendering type, then per-material, then (not ideally, but you could) per-object if you had to. But you might also group together all your resources used for compute-type tasks - e.g, clustered forward lighting and the like - into a descriptor set. I decided to use "resource groups" as my abstraction or alias over a descriptor set, and began with a fairly explicit setup like so:

{% highlight glsl %}
#pragma BEGIN_RESOURCES VOLUMETRIC_FORWARD
UNIFORM_BUFFER cluster_data {
    uvec3 GridDim;
    float ViewNear;
    uvec2 Size;
    float Near;
    float LogGridDimY;
} ClusterData;

U_IMAGE_BUFFER r8ui ClusterFlags;
U_IMAGE_BUFFER r32ui PointLightIndexList;
U_IMAGE_BUFFER r32ui SpotLightIndexList;
U_IMAGE_BUFFER rg32ui PointLightGrid;
U_IMAGE_BUFFER rg32ui SpotLightGrid;
U_IMAGE_BUFFER r32ui UniqueClustersCounter;
U_IMAGE_BUFFER r32ui UniqueClusters;

#pragma END_RESOURCES VOLUMETRIC_FORWARD
{% endhighlight %}

The above declares a resource group. During the shader generation process, I search the current subfolder being used for a `Resources.glsl` file and parse it for all the resource groups I can find. I then parse from this the appropriate metadata I will need to generate the final bindings, so that when a user writes `#pragma USE_RESOURCES VOLUMETRIC_FORWARD` we insert the above resource block into their shader. This worked well enough, and combined with my pre-existing reflection infrastructure it gave me most of the capabilities I needed.

## Taking the Library Further

I began wondering, however, if I could take this library further. It was about this time that I was doing more reading into rendergraphs and that concept, like in [this excellent article](http://themaister.net/blog/2017/08/15/render-graphs-and-vulkan-a-deep-dive/) (from Hans Kristian, who helped me with his excellent work on spirv-cross earlier in this very article!). Effectively, a rendergraph is us dealing with the manual synchronization required in modern APIs, and has some things in common even with the AST that is created during the compiliation of something like C/C++. We atttempt to traverse a series of rendering commands, rendertargets, and resources - from this, we attempt to generate scheduling. Here's a helpful graphic I created some time ago for when I was writing a software design document explaining this concept to a crowd of embedded developers at work:

<img src="{{site.baseurl}}/assets/img/RenderGraph.png" alt="I am eternally proud of this infographic, honestly" />

The rendergraph really needs to know when resources are read, and when they are written to. And if a certain shader only performs pure reads, or pure writes, this can be *immensely* helpful for scheduling the various steps in our rendering process (as shown). Additionally, I believed that with a little work I should be able to automate the resource creation process as well - so that we could use our shaders to hook into something like a [Vulkan resource plugin](https://fuchstraumer.github.io/Vulkan-Resource-Plugin/). Through this, not only are we generating the descriptor sets, descriptor set layouts, pipeline layouts, and descriptor pools (a huge chunk of work we would otherwise compile in to our code!) - we can now generate the requisite resources automatically and based on some input data, too! This was an exciting prospect so I got right to work.

#### Resource Groups - Attempt #2

First, I knew I would need to improve how I handled resources. We would now have to start tracking resources 

#### Resource Groups - Lua version

#### Automating Resource Creation and Population (Sometimes!)

#### Extracting Even More Metadata

##### Getting Strings Across A DLL Boundary

I didn't explicitly note it earlier, but due to the large amount of dependencies ShaderTools requires I was rather determined to make this project work as a DLL/shared library - so that clients wouldn't have to also link to our huge pile of dependencies. This creates some restrictions though: for example, as I noted above we often want to retrieve the names of our resources and even the resource group they're in as strings. But how do we pass or retrieve strings across a DLL boundary? Just writing to an array of `const char*` pointers can be dangerous and expensive, as then we have to 

- make sure the pointers remain valid by storing copies of the strings somewhere
- potentially copy what would've been temporary data into more permanent storagea

Instead, through some cheeky usage of RAII and `strdup` we can get a nice and safe way to retrieve strings across a DLL, while retaining a C ABI. The solution is a structure like this:

{% highlight cpp %}

{% endhighlight %}

In case you were unaware, `strdrup` copies the strings and requires eventually calling `free` on the destination string pointers. I didn't want to enforce users to have to remember this step though, so by making the destructor of the `dll_retrieved_strings_t` structure call this (along with deleting it's copy constructor + assignment operator) we can effectively avoid leaking memory when passing these strings around. The easiest way to use it is by declaring a new scope with brackets (`{}`), copying the strings into local storage, then exiting the brackets and letting the retrieved strings be cleaned up. It's an idea I got again from `std::lock_guard`, and it works splendidly! And in case you can't tell, I'm fairly proud of my clever little trick :)

## ShaderPacks, Shaders, and ShaderStages

Now with a full-featured resource system that allows for fully automated 

## Potential Improvements 

- Better handling of vertex interfaces and inter-stage interfaces
- Improved Lua integration: fairly "singleton"-esque right now
- Move specialization constants to Lua system as well
- Lots of opportunities for cleanup and performance improvements
- Run a multithreaded compile step for shader groups
- Implement caching, by loading files back up from a specified directory
