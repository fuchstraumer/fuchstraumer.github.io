---
layout: post
date: 2018-9-2
title: "Creating and Designing a Shader Toolchain"
img: aurora.jpg
published: true
tags: [C++, tools, Vulkan, Shaders, SPIR-V, Lua]
---

As an early pre-article note: I have written various versions of this article about three times now. ShaderTools has evolved *so* much from it's initial versions, and is constantly improving, so it was hard to ever feel it was "ready" to share. Or, I'd write half the content I needed then realize that (usually) I had written some new feature or fix that solved a problem I mentioned in the writing. So, here goes nothing: and hopefully, this ages well.

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
// descriptor set's resources
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

And the gift


