---
layout: post
date: 2018-6-3
title: "Creating a Vulkan Shader System"
img: aurora.jpg
published: true
tags: [Vulkan]
---

Over the past several months, I have been working on my [ShaderTools repository](). This project started incredibly simple,
and eventually evolved to including shader generation, Lua scripting, and more functionality than I initially envisoned
ever having.

Now, it's a fairly full-featured Vulkan-focused shader toolchain for generating, compiling, and reflecting on shaders. It simplifies
the process of writing shaders tremendously, and completely removes the need to worry about some of the more complicated realities of 
Vulkan. Further, it's proven easy to modify and expand - so expect even more progress on it yet!

### Humble Beginnings

As I mentioned, I initially had not planned to make this particular project very complex at all. I wanted to be able to compile shaders
at runtime, and this was all at first. So I began adding a simple shader compiler into my larger renderer project, [VulpesSceneKit](). 

But I got to thinking, especially after reading of [Kyle Halladay's articles]() - how could I use runtime shader compiliation and reflection
to simplify my usage of Vulkan? Right off the bat, being able to reflect on a shader would let me generate descriptor set layout information
and pipeline layout information for a set of shaders at runtime - instead of having to effectively compile-in this setup code as I currently
did. And besides being inflexible when done like this, it can also be a pain to write. Like this snippet adding a bunch of texel buffers
for use in clustered forward shading:

{% highlight cpp %}
constexpr static VkShaderStageFlags fc_flags = VK_SHADER_STAGE_FRAGMENT_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 0);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 1);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 2);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 3);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 4);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 5);
texelBuffersLayout->AddDescriptorBinding(VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER, fc_flags, 6);
{% endhighlight %}

Ew. And this is just for creating the layout information: later we have to also attach these same 7 bindings/objects to the descriptor set, using
another set of similar function calls. And I have to make sure to keep this code updated, so that it reflects any changes I make in the shader code.

Surely we can do better - so I aimed to do just that.

### Simplifying Pipeline Creation

So, in order to simplify our setup code and make sure we don't break compiled code by changing shader code around, we need to find some way to parse
a shaders used resources and generate `VkDescriptorSetLayoutBinding` structures based on this information. Enter `spirv-cross`, which allows us to
extract all of this information ourself. Now until recently `spirv-cross` wasn't terribly well-documented: so much of my early work was merely spent
experimenting with the API and figuring out what sort of information I could even hope to extract (after I chatted with Hans on reddit though, he got right to 
work making [this post](https://github.com/KhronosGroup/SPIRV-Cross/wiki/Reflection-API-user-guide) <3)

So by using `spirv-cross` appriopriately, we can now generate arrays of `VkDescriptorSetLayoutBinding`s per shader, with one array per descriptor set used
per shader. We are also able to find push constants, retrieve specialization constants, and find out what our vertex shader inputs look like so we can generate
an array of `VkVertexInputAttributeDescription`s too! Now we don't need to recompile or edit our rendering code to reflect changes in shaders, and the amount
of code required to perform this setup process has been sharply reduced as well. It looks something like the following:

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

I quickly realizing another problem though: in complex setups that use several passes/groups of shaders all sharing the same resources, we now have to make
sure to copy chunks of code like the following between each shader, ensuring they're all the same and always up-to-date:

{% highlight glsl %}
layout (set = 1, binding = 1, rgba32f) uniform imageBuffer positionRanges;
layout (set = 1, binding = 2, rgba8) uniform imageBuffer lightColors;
layout (set = 0, binding = 0, r8ui) uniform uimageBuffer flags;
layout (set = 0, binding = 2, r32ui) uniform uimageBuffer lightCounts;
layout (set = 0, binding = 4, r32ui) uniform uimageBuffer lightCountOffsets;
layout (set = 0, binding = 5, r32ui) uniform uimageBuffer lightList;
{% endhighlight %}

Oh, no. This isn't good. We'll get to how I got around this next, after I explain a bit about why using `spirv-cross` led me to decide on making this
project a shared library.

#### The DLL Decision

The downside of the SPIR-V ecosystem right now is that it kinda tends to drag in a whole boatload of dependencies: ShaderTools has something like 7 linked-in
dependencies if I recall correctly. This is quite a bit to link in, and they have a fairly large amount of headers attached to (so it begins to clutter the
browse/intellisense databases of your IDE) - so in order to hopefully compartmentalize things, I decided to make this my first proper attempt at making a 
shared library. That way, all the static libraries required for the SPIR-V stuff could be fully linked in without requiring anything of the client, and I could
use `PImpl` to its utmost to ensure clients also don't start including headers they don't need. `PImpl` probably incurred some shudders, though: it's not 
exactly the most enjoyable idiom to use or implement, but thankfully `std::unique_ptr` makes it a little less painful to deal with. 

One of the more legitimate complaints is the requirement to then comply to a C-style API: so no exposing/passing of standard library objects across DLL 
boundaries, and fairly simple tasks like retrieving arrays of strings [become tests in clever little RAII tricks](https://github.com/fuchstraumer/ShaderTools/blob/master/src/common/UtilityStructs.cpp#L9) as you try to avoid leaking memory from things like `strdup`.

In the end though, I was actually pretty surprised at how painless things were. Insulating clients from the maze of dependencies required felt nice, and it
really did help encourage me to design smarter and more intuitive interfaces. Being able to expand functionality or fix bugs without having to recompile
my rather heavyweight renderer is a considerable plus, too.

### Shader Generation - how not to do it

So, before our little sidebar I was just mentioning that maintaining parity of resource binding declarations across several shaders was really not a fun
thing to have to do. In order to get around it, we'll clearly need some kind of generative shader system: and I had been wondering about creating this
sort of system anyways, since I wanted to be able to generate shader permutations for different hardware and API capabilities.

So I began implementing a generative shader system - and while it worked, the initial method I used was anything but pretty. Things that would be processed
were all declared with `#pragma` - or sometimes an all-caps keyword prefixed with `$`. 

### Resource Groups

### Lua scripts as JSON 2.0

#### Unordered everythings

#### Local Objects and Script Dependencies oh my

### Separating Usage of a Resource from the Actual Object

### Conclusions

### Further goals

