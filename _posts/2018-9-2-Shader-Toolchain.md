---
layout: post
date: 2018-9-2
title: "Creating and Designing a Shader Toolchain"
img: aurora.jpg
published: true
tags: [C++, tools, Vulkan, Shaders, SPIR-V, Lua]
---

As an early pre-article note: I have written various versions of this article about three times now. ShaderTools has evolved *so* much from it's initial versions, and is constantly improving, so it was hard to ever feel it was "ready" to share. Or, I'd write half the content I needed then realize that (usually) I had written some new feature or fix that solved a problem I mentioned in the writing - like the resource group system. That has gone through so many variations and iterations that I could write a whole article *just* on that content. So, here goes nothing: and hopefully, this ages well.

# Toolchains: the Reality of Engine Development

The title just above is not a pessimistic statement - but it is the truth! Before I really dove into the challenge of attempting to make my own graphics engine, I had little idea of creating things like toolchains. My previous work projects had never required them, or weren't large enough to benefit from them. But a few early experiments with trying to implement clustered forward rendering (and looking at [volumetric tiled forward]() made it even more urgent I get this sort of system working) made it clear that I needed some help - in the form of code designed to help bridge the gap and make my life easier.

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
{% endhighlight %}

Further, by tracking this data at a slightly higher level and between multiple invocations of this "reflection" system, one could get an accurate count of the resources required - so setting up your `VkDescriptorPool` to perfectly match the requirements of the current suite of shaders became nearly trivial! It had the effect of reducing code complexity *and* making things much more flexible - effectively, we had incidentally moved to a data-driven design where the "data" was our shaders. This data was now affecting the behavior of our code, and our code adapted to the data instead of the other way around (or just a lack of adaptation whatsoever).

### Potential Issues

However, there is one problem I have not yet solved. One still has to make sure that compatability between shader resources is maintained between all resources that may use a given set of resources: for example, let's say I perform the binding shuffling I briefly mentioned earlier (where I suggested adding a uniform buffer). I can't just modify a single shader doing that - I have to then modify all shaders that bind to that particular descriptor set. Sure, I could create an all-new unique descriptor set just for that single shader: but, descriptor set binding is one of the more expensive operations we can perform in Vulkan so it's more ideal for use to keep our descriptors pooled, to reduce binding changes. But copying and pasting code around, and remembering to always do so, is less than ideal as well. So, how do we fix this?

###### A Brief Aside: spirv-cross

I'd like to take a moment to quickly speak positively of `spirv-cross` and it's maintainers, especially Hans: I created [an issue](https://github.com/KhronosGroup/SPIRV-Cross/issues/551) explaining a lack of documentation or examples on using the API. I had personally figured out how to use the library for reflection, but it had taken a fair bit of experimentation and laborious crawling through the code to do so. I mentioned this on reddit and Hans suggested I create the issue I did, and after doing so he got right on it and created the rather immensely useful [examples for reflection](https://github.com/KhronosGroup/SPIRV-Cross/wiki/Reflection-API-user-guide) that one can now find in the wiki. Point of this sidebar being: take the time to speak to people about their libraries or projects! If you feel its lacking documentation or examples, the best way you can let someone know is by creating an issue on github in my opinion. Give it a shot!

## Shader Generation

In order to get around this problem, we can move to a generative shader approach: this makes plenty of sense, and is one that most shader toolchain software packages end up using anyways. As I am singular person, however, I'll avoid the highly abstracted approach of Unity and Unreal Engine: as great as that can be, the scope of work required is simply far too immense for me to manage among all my other work.

So, my goal became generating just the resource declarations per-shader: users will still have to write the `main()` block as they would otherwise, but dealing with the intricacies and irritations of Vulkans resource binding model would be effectively removed. Additionally, we could take this chance to add simple things like simpler handling of specialization constants, `#include` support, and other various pre-processor features (as we are now writing a GLSL preprocessor). So, let's get to work.

#### Specialization Constants

I'm going to skip over detailing how I implemented `#include` support - it's nothing too shocking to anyone who's been doing development work for a while, so I'll just skip right into something more interesting. Vulkan has these interesting objects called "specialization constants" - constant values in the shader that are bound to specified locations, a bit like descriptor resources. What makes them unique, however, is that the value can be modified at pipeline creation time. This is fairly powerful, as it lets you write one shader then potentially vary the behavior shortly before you use it: potentially allowing for the "generation" of shader permutations and variations at runtime. I personally tend to use it for holding values like the screen size, and other environmental constants that don't frequently change: when they do change (e.g, during a swapchain recreation event) we would have to recreate our pipelines anyways - allowing us to update the value to reflect the new screen size, for example.

However, it can be a bit annoying and tedious to type out `layout (constant_id = (idx))` for each of the specialization constants we intend to use. Additionally, I figured (correctly!) that practicing on this feature would help me prepare for the more difficult world of descriptors and those resources. 

#### Resource Groups, v1.0

#### Resource Groups - Lua version

