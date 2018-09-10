---
layout: post
date: 2018-9-5
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

### Resource Groups from Lua Scripts

Despite the logic above, my first goal with the upgrades I had planned was to automate resource creation - and in order to do that, I was going to have to really upgrade my resource groups. First, the current system just felt sort of ungainly - I wasn't really a fan of it, and felt that it could have been designed better. While ShaderTools might not be at the level of being used by artists, I could see it being used as a middleware translation between some sort of material/node editor and the final shader code: so simplification is just going to make interfacing to higher-level abstractions easier.

At first, I thought I could use JSON to describe resource groups. But that quickly fell apart: in order to create a `VkBuffer`, we need to know the size of the buffer. And there are many cases where this size depends on some runtime parameters (usually any time you're doing compute/lighting work) like the current screens size or the depth range - or, it might just be a configurable value like a maximum number of lights that we adjust based on the user's graphics configuration. Regardless, there was no easy to handle this in JSON - I'm effectively describing a need for functions.

But then I had a realization: Lua is fast with key-value tables, especially LuaJIT, and Lua lets me use functions as table fields. This would be *perfect*. I could expose some environment functions like `GetWindowSizeX` to Lua, and call them to set fields of my resource descriptors. Further, I could be fairly certain that while executing a Lua script might not be nearly as fast as simply parsing some JSON it's unlikely that it'll be so slow as to massively impact runtime performance.

The format I settled on works like so - first, we can of course declare all our functions in a separate Lua script. Here, I call functions called "GetWindowX()" and "GetWindowY()" that get the size in pixels of our current rendertarget - and use that to calculate tile dimensions for clustered forward rendering. I also store a few other members, which could potentially be adjusted at runtime by reading from a configuration or the like:

{% highlight lua %}
local dimensions = {
    NumLights = 2048;
    TileWidth = 64;
    TileHeight = 64;
};

function dimensions.TileCountX()
    return (GetWindowX() - 1) / dimensionFunctions.TileWidth + 1;
end

function dimensions.TileCountY()
    return (GetWindowY() - 1) / dimensionFunctions.TileHeight + 1;
end

function dimensions.TileCountZ()
    return 256;
end

function dimensions.GetTileSizes()
    return dimensions.TileCountX(), dimensions.TileCountY(), dimensions.TileCountZ();
end

function dimensions.GetTotalTileCount()
    x, y, z = dimensions.GetTileSizes();
    return x * y * z;
end

return dimensions;
{% endhighlight %}

Our resource script is then able to use these functions by using the standard Lua "requires" format. A resource script is what we use to declare our resource groups now, instead of the rather ungainly `#pragma BEGIN_RESOURCES` format. It makes the specification of all our metadata loads easier: field names make it clear what we can write, and adding custom structured buffer types becomes easier too (more on that later!):

{% highlight lua %}
local dimensions = require("Functions")
-- Reduced for brevity, but you get the point
Resources = {
    GlobalResources = {
        UBO = {
            Type = "UniformBuffer",
            Members = {
                model = { "mat4", 0 },
                view = { "mat4", 1 },
                projectionClip = { "mat4", 2 },
                normal = { "mat4", 3 },
                viewPosition = { "vec4", 4 },
                depth = { "vec2", 5 },
                numLights = { "uint", 6 }
            }
        }
    },
    ClusteredForward = {
        flags = {
            Type = "StorageTexelBuffer",
            Format = "r8ui",
            Size = dimensions.TotalTileCount(),
            Qualifiers = "restrict"
        },
        bounds = {
            Type = "StorageTexelBuffer",
            Format = "r32ui",
            Size = dimensions.NumLights * 6,
            Qualifiers = "restrict"
        },
        Tags = { "InitializeToZero", "ClearAtEndOfPass" }
    },
    Lights = {
        positionRanges = {
            Type = "StorageTexelBuffer",
            Format = "rgba32f",
            Size = dimensions.NumLights,
            Qualifiers = "restrict readonly",
            Tags = { "HostGeneratedData" }
        }
        lightColors = {
            Type = "UniformTexelBuffer",
            Format = "rgba8",
            Size = dimensions.NumLights,
            Qualifiers = "restrict readonly",
            Tags = { "HostGeneratedData" }
        }
    }
}
{% endhighlight %}

During execution of the script, we simply iterate through the entries of the "Resources" table. Then, we recurse through their individual entries and extract all the data we need. We are then able to build both the information we need for generating the descriptor layout/binding data we had earlier, but we now also have some extras like the ability to add qualifers and a size field. That size field is really the only item we actually *need* to generate the backing resources though, as we can technically populate the fields of things like `VkBufferCreateInfo` (and `VkBufferViewCreateInfo` for our texel buffers) already.

#### Handling Edge Cases and Unique Behavior

You are probably wondering what the `Tags` field is about - this is a field that I added fairly late in the development process, after realizing I couldn't quite cover everything I wanted to do with the existing infrastructure. The intended use is to further specialize individual resources and groups using user-defined "tags": these tags are stored then on a per-group and per-shader basis as appropriate, and can be used by clients interacting with `ShaderTools` to mutate how they use the resources.

For example, lets consider my attachment of the "HostGeneratedData" tag to the Lights resources. Instead of using this tag to change behavior in the backend, the intent is for a user to do something like the following (using the `ShaderResource` object found [here](https://github.com/fuchstraumer/ShaderTools/blob/6f409a4ad66ecc886258b711589e967ca8b00566/include/core/ShaderResource.hpp)):

{% highlight cpp %}
void BufferResourceCache::createBuffer(const st::ShaderResource* rsrc) {
    VkBufferCreateInfo create_info{ default_buffer_info };
    create_info.usage = usageFlagsFromDescriptorType(rsrc->DescriptorType());
    create_info.size = rsrc->MemoryRequired();

    {
        st::dll_retrieved_strings_t tags = rsrc->GetTags();
        if (tags.NumStrings != 0) {
            for (size_t i = 0; i < tags.NumStrings; ++i) {
                if (strcmp(tags[i], "HostGeneratedData")) {
                    // Since it's host generated data, mutate usage flags
                    // so that we can copy into it from the host
                    create_info.usage |= VK_BUFFER_USAGE_TRANSFER_DST_BIT;
                }
            }
        }
    }

    // should be using a view_info param here but skipped for brevity
    VulkanResource* new_buffer = resourcePluginAPI->CreateBuffer(&create_info, nullptr,
        0, nullptr, memory_type::DEVICE_LOCAL, nullptr);
}
{% endhighlight %}

In this way, we can attach further metadata and then evolve unique behaviors based on that data without having to modify the backend library doing all this parsing. The above probably isn't the most ideal case and this probably is not the best or most performant way to handle this, but I didn't feel like it was wise to start modifying ShaderTools deeply to accomodate the premise of these tags. So in the end I think I leaving it to the user to do things like the above makes the most sense. In the case of the other tags, like the one attached to the `ClusteredForward` group, we would make note of this (on the rendegraph or client level, of course) and then be sure to fill the buffers in questions with zeros, and schedule another clear/fill to zero operation at the end of each renderpass. I do have other ideas, however: like potentially making the tags field cause the execution of a secondary script, which can be used to modify the resource as appropriate to include the appropriate fields (maybe by potentially reaching in and directly mutating a `VkBufferCreateInfo`?). Unfortunately, I haven't yet had time to further explore this idea. The farther I got was using the tags field to call a `std::function` object that was stored in a `std::unordered_map<std::string, std::function<void(st::ShaderResource*)>>`: this accomplishes the same end result, but is still less-than-ideal of course as the code for modifying these resources is still compiled-in (for now).

#### Handling Structured Buffers

The `UBO` structure in the above snippet of the shader resource script probably looked a little weird: first I had the buffers member name, and then the field was another Lua table containing a type string and an index. This combo looks messy, but the index is unfortunately required: Lua uses unordered tables, so the order of the members may otherwise end up scrambled and confused. This took me some time to even realize, however, as it wasn't until I compared a shader before and after processing that I even noticed the order of the members had changed.

Object sizes are read from a table mapping the object's type string to a size value - there exist a large quantity of default sizes baked into the ShaderTools backend, but a user can actually freely add more sizes by declaring a new `ObjectSizes` table. The key is the object's type string, then, and the value is just the size. This table is then appended to the pre-existing table in ShaderTools.

#### Textures?

Handling textures is a tough one - by textures, I'm referring to images we explicitly expect to load data for from a file, and that will be used when shading a primitive. We obviously will want to declare the binding locations of these objects ahead of time, but we won't be able to do anything more for them: format is unknown, size is unknown, and the backing data is unknown as well. To make it more complex, we also can't really assume that a single file will ever be enough for a texture "slot". It's likely that we will be changing out what the backing data in a texture slot is at runtime, as we change primitives and bind different descriptor sets. For that reason, I added a `FromFile` field. When specified as true, the parsing of further metadata is skipped - so there is no way to retrieve `VkImageCreateInfo`/`VkImageViewCreateInfo` for textures created in this manner. It's imperfect, but it's the best I could think of doing without specifying what our material system will look like at the level of this library (which I'd rather not do, as it makes it considerably less general-use). Thus, a texture field looks a bit like so:

{% highlight lua %}
Resources = {
    MaterialTextures = {
        diffuseMap = {
            Type = "CombinedImageSampler",
            FromFile = true
        },
        normalMap = {
            Type = "CombinedImageSampler",
            FromFile = true
        },
        roughnessMap = {
            Type = "CombinedImageSampler",
            FromFile = true
        }
        -- And so on...
    }
}
{% endhighlight %}

### Extracting Even More Metadata

As mentioned earlier, part of my goal in getting this system to work was so that I could use it to assist rendergraph construction. Now tha we can create the backing resources attached to shaders, we need to figure out how much data we can extract about the usage of resources by each individual shader (and sometimes, shader stages). The biggest thing we want to identify is if a resource is `readonly` or `writeonly` - which becomes rather hard to identify. At first, I thought this qualifier was being explicitly applied to a resource if it was at all able to: in GLSL recompiled from SPIR-V assembly the `readonly`/`writeonly` qualifiers were being applied to resources. But upon reading the `.spvasm` files myself, I couldn't find these qualifiers anywhere - and reading the SPIR-V spec revealed that this qualifier is only available when applied to images (and seemingly, not even texel buffers).

{% highlight text %}
OpDecorate %positionRanges DescriptorSet 1
OpDecorate %positionRanges Binding 1
OpDecorate %positionRanges Restrict
OpDecorate %lightBounds DescriptorSet 2
OpDecorate %lightBounds Binding 1
OpDecorate %lightBounds Restrict
OpDecorate %Flags DescriptorSet 2
OpDecorate %Flags Binding 2
OpDecorate %Flags Restrict
{% endhighlight %}

The only qualifier I could see - shown above in a snippet of SPIR-V assembly - was the "restrict" one that I was specifying be applied myself in the resource scripts. So how was the recompiler, when compiling back into GLSL, able to decide to apply these qualifiers? I created an issue in the `spirv-cross` repo, but diving further into the source code, learning more about SPIR-V, and [a clarifying answer by Hans](https://github.com/KhronosGroup/SPIRV-Cross/issues/606) (again, he is too good and I appreciate his work on `spirv-cross` immensely) revealed that SPIR-V doesn't store this information, conventionally. Indeed, our best bet is to make sure we use the right qualifiers from the get-go.

#### Memory Modifiers/Qualifiers

So, we know two things: retrieving the Memory (or, access) qualifiers from the SPIR-V isn't strictly possible, and setting the qualifiers initially is our best. The second point is why I added the `Qualifiers` field to the resource scripts: you'll notice nearly every resource uses `restrict`, which is a qualifier that you should (by the recommendation of numerous Khronos documents) use as often as possible. In other locations, I was able to specify things like `readonly` however.

At first, I thought I could then simply store these qualifiers in the shader resource meta-object used to associate metadata to a resource. But this really won't work: these access qualifiers may change in different shaders, and while we're able to specify some as invariant across multiple shader stages / pipelines this is not usually the case. So I had to rethink things.

Clearly, we needed to separate the abstraction representing a shader resource from it's _usage_. There is a clear separation here, and it's something we'll now need - we'll be using all of our resources multiple times, and each time we may have different qualifiers on it. This resulted in the creation of a `ResourceUsage` ([here](https://github.com/fuchstraumer/ShaderTools/blob/cd5910cb594fc6c1818eb09da9d0a3d194242d95/include/core/ResourceUsage.hpp)) object: `ShaderResource` ([here](https://github.com/fuchstraumer/ShaderTools/blob/cd5910cb594fc6c1818eb09da9d0a3d194242d95/include/core/ShaderResource.hpp)) represents a singular resource itself, but a `ResourceUsage` is a child object representing a singular use of the resource in a particular shader. We then store our access qualifiers in the resource usage, not the parent resource. This way, our rendergraph is able to also exploit this split in logic: it can see a resource is used multiple times, and by analyzing the qualifiers on it's various usages it is able to schedule passes, insert memory barriers, and ensure safe access/use of a resource.

##### Getting Strings Across A DLL Boundary

I didn't explicitly note it earlier, but due to the large amount of dependencies ShaderTools requires I was rather determined to make this project work as a DLL/shared library - so that clients wouldn't have to also link to our huge pile of dependencies. This creates some restrictions though: for example, as I noted above we often want to retrieve the names of our resources and even the resource group they're in as strings. But how do we pass or retrieve strings across a DLL boundary? Just writing to an array of `const char*` pointers can be dangerous and expensive, as then we have to

- make sure the pointers remain valid by storing copies of the strings somewhere
- potentially copy what would've been temporary data into more permanent storagea

Instead, through some cheeky usage of RAII and `strdup` we can get a nice and safe way to retrieve strings across a DLL, while retaining a C ABI. The solution is a structure like this:

{% highlight cpp %}
struct ST_API dll_retrieved_strings_t {
    dll_retrieved_strings_t(const dll_retrieved_strings_t&) = delete;
    dll_retrieved_strings_t& operator=(const dll_retrieved_strings_t&) = delete;
    dll_retrieved_strings_t();
    ~dll_retrieved_strings_t();
    dll_retrieved_strings_t(dll_retrieved_strings_t&& other) noexcept;
    dll_retrieved_strings_t& operator=(dll_retrieved_strings_t&& other) noexcept;
    void SetNumStrings(const size_t& num_names);
    const char* operator[](const size_t& idx) const;
    char** Strings{ nullptr };
    size_t NumStrings{ 0 };
};

// in use, in the DLL:
dll_retrieved_strings_t ShaderPack::GetShaderGroupNames() const {
    dll_retrieved_strings_t names;
    names.SetNumStrings(impl->groups.size());
    size_t i = 0;
    for (auto& group : impl->groups) {
        names.Strings[i] = strdup(group.first.c_str());
        ++i;
    }

    return names;
}
{% endhighlight %}

In case you were unaware, `strdrup` copies the strings and requires eventually calling `free` on the destination string pointers. I didn't want to enforce users to have to remember this step though, so by making the destructor of the `dll_retrieved_strings_t` structure call this (along with deleting it's copy constructor + assignment operator) we can effectively avoid leaking memory when passing these strings around. The easiest way to use it is by declaring a new scope with brackets, copying the strings into local storage, then exiting the brackets and letting the retrieved strings be cleaned up. It's an idea I got again from `std::lock_guard`, and it works splendidly! And in case you can't tell, I'm fairly proud of my clever little trick :)

## ShaderPacks, Shaders, and ShaderStages

Now with a full-featured resource system that allows for fully automated resource construction (in some cases), and that allows for our rendergraph to extract as much precious information as it can, it's time to move on to more of our higher-level design. This was an area I've struggled in, and I was only vaguely relieved to find a few other articles describing similar problems.

For one, naming our objects in an efficient way becomes difficult. What is a "Shader"? Is it a single pack of shader code representing a stage in the programmable graphics pipeline? Or is it the combined programmable shader stage source code snippets required for a whole pipeline? What do we call the combined set of resources, shader source code snippets / stages, and so on? For my project, after a bunch of iteration I settled on the following:

- ShaderStage describes a single stage of execution in the programmable graphics pipeline, and is our smallest distinct object
- A Shader is a combination of ShaderStages that will eventually be used by a single graphics pipeline
- A ShaderPack is a combination of Shaders, along with a resource script

#### Designing an Efficient Interface

#### Creating Descriptor Pools



## Potential Improvements

I've had a whole host of improvements in mind for ages now - and in many ways, ShaderTools' current status is a result of me actually implementing many of my ideas for improvements. If I was going to make a 2.0 version of this library, I'd probably use a more Unity-like system and implement abstract shaders with getters and setters and the like that are far removed from writing conventional GLSL: it opens up lots of chances for us to work some magic from the backend, ultimately. These ideas however are improvements I could be making to the *current* version of ShaderTools, however:

- Something I really need to do: **resources with same names may not actually be equivalent and shouldn't have their representations merged together**
- **Figure out how to handle textures.** Using ShaderTools to create these backing resources is a moot point, as we'll likely be swapping what textures are bound where fairly frequently
- Better handling of vertex interfaces and inter-stage interfaces for vertex attributes
- Implement tags being used to execute further scripts and mutate resources
- Improved Lua integration: fairly "singleton"-esque right now, can get weird if client is using Lua too
- **Move specialization constants to Lua system as well**
- Profiling! Need to test using this live to build a bunch of shaders so I can get some good performance data from it so as to direct my optimizations
- Run a multithreaded compile step for shader groups, since compiling with optimizations is rather expensive
- Implement caching, by loading files back up from a specified directory (or the temporary file dump directory we use now)
- Shader permutations! and actually enabling/using vendor extensions as appropriate. This falls more in line with my idea of a "ShaderTools 2.0", however
- Potentially build a C# GUI/interface over this library to make interacting and using it far easier, and so I can finally learn C#
