---
layout: post
date: 2018-1-27
title: "Simplifying Vulkan Pipeline Creation"
img: cascades.jpg
published: true
tags: [Vulkan,C++,Design,Libraries,Modern C++]
---

Creating a graphics pipeline in Vulkan is definitely one of the more involved tasks
one can perform in this API: creating the literal `vkPipeline` is easy enough, but filling 
all the files of `vkGraphicsPipelineCreateInfo` is... well, it's not always fun.

In my current trials of attempting to build a highly data-driven and flexible renderer, one
of the problems I ran into was finding ways to generate one of the most important members of
the `vkGraphicsPipelineCreateInfo` structure - the `layout` member, specifying a handle to the
pipeline layout that's going to be used for rendering the current target.

The reason this is so important is related to my plans for my renderer: there are "feature renderers"
and then "features". Features are the individual objects - e.g, a box or crate model in a scene
that been loaded from an `.obj` file. A feature renderer would then be an object that takes care 
of rendering (as an example) all `.obj` files, since I can apply similar techniques to all of these
(like consolidating the parts of a model together and generating indirect draw commands ahead of time).

However, lets say one feature uses a full set of PBR textures, and another only uses a reduced set. They 
can both share most of the same pipeline state (nearly all of it, really), but require slightly
different shaders and a different `vkPipelineLayout`. This pipeline layout is going to be mostly
influenced by our `vkDescriptorSetLayout` object, and up to this point I had been manually specifying
(and thus compiling-in) the `vkDescriptorSetLayoutBinding` members of this object in my code. 

This was a lot of boilerplate, but it also made things fixed and not terribly flexible. So I knew
there had to be a better way.

## Enter, ShaderTools!

Besides working on some clustered-forward stuff and being occupied with work, I've been spending
a sizable chunk of time working on my library "ShaderTools". It's definitely still a bit of a 
work-in-progress, but it's done enough that I felt I could write an article explaining how it works
and what it's done for my rendering code. 

ShaderTools has 3 main elements: a shader compiler, a shader "parser" / binding generator, and a 
generative shader system. The latter will be covered in another article, as the former two are
what's mainly relevant here (but they play VERY nicely with generated shaders!). 

It's also designed to be compiled to a DLL: compiling and reflecting on GLSL/SPIR-V requires
quite a few third-party libraries, and I didn't want to require recompiliation of clients of 
this library since I update it fairly frequently. I also wanted a chance to see how `PImpl` could
play with Modern C++. 

Lets start diving in, though.

### Compiling shaders

There's nothing terribly shocking or surprising here - `ShaderCompiler` does what it says on the tin,
compiling shaders either given a file path or given a raw source string.

{% highlight cpp %}
ShaderCompiler compiler;
Shader handle = compiler.Compile("my_vertex_shader.vert");
{% endhighlight %}

The returned object is really just a structure wrapping a 64-bit unsigned integer used to store some
important info. The upper 32 bits store a hash of the path to the saved shader file (either what was 
passed in, or a saved copy of the raw source string in a temp directory) and the lower 32 bits store
the shader's stage (vital for some internal operations).

## Shader Reflection

The real magic happens with shader reflection: here, we take a compiled binary and extract some important
data from it, namely:

    - Descriptor usages: types, bindings, quantity, etc
    - Inputs/outputs between stages: primarily useful to extract vertex shader inputs
    - Push constant ranges

Parsing binaries is done by adding all the shaders used with a given pipeline (so usually vertex + fragment)
to a `BindingGenerator` object and calling `CollateBindings()`, which sorts internal bindings to get them all
pleasantly ordered and removes duplicate bindings found during parsing.

#### Reflecting on the compiled binaries

spirv-cross is a tool that serves several purposes - reflecting back on generated binaries being
one of the many things it can do, but this is all I use it for. We perform reflection by creating a 
spirv-cross `Compiler` object, passing it the vector of binary data that corresponds to the shader we
seek information on. From here, it analyzes the bytecode and extracts what data it can.

{% highlight cpp %}
void BindingGeneratorImpl::parseImpl(const std::vector<uint32_t>& binary_data) {
        using namespace spirv_cross;
        Compiler compiler(binary_data);
        // parse for each type of resource
}
{% endhighlight %}

From this Compiler we can extract the following `ShaderResources` structure:

{% highlight cpp %}
struct ShaderResources {
	std::vector<Resource> uniform_buffers;
	std::vector<Resource> storage_buffers;
	std::vector<Resource> stage_inputs;
	std::vector<Resource> stage_outputs;
	std::vector<Resource> subpass_inputs;
	std::vector<Resource> storage_images;
	std::vector<Resource> sampled_images;
	std::vector<Resource> atomic_counters;

	// There can only be one push constant block,
	// but keep the vector in case this restriction is lifted in the future.
	std::vector<Resource> push_constant_buffers;

	// For Vulkan GLSL and HLSL source,
	// these correspond to separate texture2D and samplers respectively.
	std::vector<Resource> separate_images;
	std::vector<Resource> separate_samplers;
};
{% endhighlight %}

From this structure we can get all of the data we need to help us make pipeline setup quite a bit easier:
`stage_inputs` and `stage_outputs` are our input/output attributes, and in the case of the Vertex shader
retrieving the data from `stage_inputs` lets us generate the input attribute objects we need to create a pipeline.

The rest of the fields represent our various resource types, and usually a different `VkDescriptorType` value.
When generating bindings, we have a fairly simple series of actions for retrieving the stuff we want - I'll 
avoid making a particular example of that here, though, and instead just LINK to the section of my code dedicated to 
doing just that. Partially because things are about to get fairly detailed - turns out storing all the information
we can possibly need is actually quite a bit of work!

The parent object representing a single set and all of it's objects is the `DescriptorSetInfo` structure, which is fairly simple:
{% highlight cpp %}
struct DescriptorSetInfo {
    uint32_t Index = std::numeric_limits<uint32_t>::max();
    std::vector<DescriptorObject> Members = std::vector<DescriptorObject>();
};
{% endhighlight %}

Each `DescriptorObject` stored in `Members` describes a single unique resource - and that structure itself also stores
a bunch of important data:

{% highlight cpp %}
struct DescriptorObject {
    std::string Name;
    uint32_t Binding, ParentSet;
    VkShaderStageFlags Stages;
    VkDescriptorType Type = VK_DESCRIPTOR_TYPE_MAX_ENUM;
    std::vector<ShaderDataObject> Members;

    bool operator==(const DescriptorObject& other);
    bool operator<(const DescriptorObject& other);

    explicit operator VkDescriptorSetLayoutBinding() const;
    std::string GetType() const;
    void SetType(std::string type_str);
};
{% endhighlight %}

The fields mostly match the fields required to create a `VkDescriptorSetLayoutBinding` - `Name` is just a convienient thing
to have when serializaing to JSON (more on that later!) and `Members` is used for objects like uniform buffers that have
multiple unique data members (as for these, we can benefit from knowing their sizes and offsets).

Okay, so we've reflected on our shaders and retrieved a bunch of useful info about stuff going on in those shaders.
But there's still one last important step before we can get to using our generated data.

#### Collating reflection data

Initially, our `DescriptorSetInfo` objects are stored in a multimap (`std::unordered_multimap<VkShaderStageFlagBits, DescriptorSetInfo>`),
allow for multiple descriptor sets per shader stage. We now want to reduce this map into a single vector of `DescriptorSetInfo` objects.

Our collation loop body then looks a little something like so:

{% highlight cpp %}
for (const auto& entry : descriptorSets) {
    for (const auto& object : entry.second.Members) {
        // Now iterating through each `DescriptorObject` stored in a 
        // single `DescriptorSetInfo` structure
    }
}
{% endhighlight %}

Completely standard and nothing shocking here, but here comes the first little "gotcha" I encountered working on this: sometimes we can
have descriptor sets bound at bindings `0` and `2`, but have nothing at index `1`. And our goal is to get everything into a single linear
vector, so this requires carefully checking our binding index to make sure we won't index outside of the container (and if we would, we
need to resize appropriately to make room):

{% highlight cpp %}
const uint32_t& set_idx = obj.ParentSet;
if (set_idx + 1 > sortedSets.size()) {
    sortedSets.resize(set_idx + 1);
}
{% endhighlight %}

Okay, that's taken care of. But the same issue can occur again with the `Members` vector in `DescriptorSetInfo` - besides having a gap 
in descriptor *set* bindings, we can also potentially have a gap in the *object* bindings too. So we get to perform the same check again:

{% highlight cpp %}
const uint32_t& binding_idx = obj.Binding;
{% endhighlight %}