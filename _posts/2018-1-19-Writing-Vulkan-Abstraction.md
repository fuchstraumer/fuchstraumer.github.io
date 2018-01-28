---
layout: post
date: 2018-1-19
title: "Designing, Building, and Using a Vulkan abstraction layer"
img: pinnacle.jpg
tags: [Vulkan,C++,Design,Libraries, Modern C++]
---

Over the past several months I have been working on a Vulkan abstraction layer - in this
post, I'd like to go through the process by which I've built this library: from it's 
barest beginnings, to it's awkward "lets do everything in one library" middle-stage, to
what I believe is now a well-designed and clearly defined abstraction library.

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
more established APIs like DX9 and OpenGL. The best resource is the red book equivalent for Vulkan - 