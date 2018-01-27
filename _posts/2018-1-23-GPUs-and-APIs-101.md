---
layout: post
date: 2018-01-25
title: "Graphics Hardware, Graphics APIs, and Graphics Programming 101"
published: true
img: sweden.jpg
tag: [Vulkan, C++, Programming, Graphics, Graphics API, General]
---

I have a number of family members, friends, and associates who either:

- Don't program at all
- Don't do graphics programming
- Know only surface-level details about graphics topics

In this post, I'm hoping to do a broad walk-through of graphics "stuff", 
from the hardware itself to the ways we as programmers interface with and
use that hardware. I'll also spend a bit of time at the end addressing 
GPU computing and how it relates to some terms that are entering the mainstream
more and more, like cryptocurrency and machine learning. 

For those that are interested, I'll probably eventually write another post 
that will be an even deeper dive into GPU hardware: potentially including
comparisons between design philosophies. I still have lots to learn
before I'm ready for that though, so this'll have to do for now!

### Graphics Hardware - GPUs

GPUs - or graphics processing units - are the powerhouse engines that cause
everything on a computer's (or a phone, laptop, tablet, etc) screen. You have
probably heard of the a "CPU" before, as the central processing unit. The names
here are quite transparent, but the GPU is a whole different beast than a CPU.

A CPU is designed for brains *and* brawn, with the ability not just to perform
rather advanced computations but to do so with relatively high speed. CPU's also
feature multiple cores - each core generally representing a parallel execution 
"thread". Threads are a common concept in programming and hardware - if we analogize
the CPU as a kitchen, each additional thread is another cook. They each share most of 
the same resources and workspace, usually have some unique resources relevant to their
immediate task at hand, and there's a limit on how many chef's we can have in a kitchen
while still being productive. Common thread counts nowadays are 4-8 for most desktop
CPUs, and 4 is even becoming extremely common in mobile hardware.

GPUs are really quite different, and it's hard to comprehend just *how* different 
they really are. While CPUs (up until AMD's Ryzen series) tended to not feature 
more than 4-8 threads, GPUs can have thousands of cores representing thousands of
threads. Unlike the CPU though, these individual threads are much *much* less powerful.

The kitchen metaphor can still help us understand them, though. Whereas a CPU features 
several chefs that are probably jumping around each performing their own complex individual
tasks, a GPU is closer to a much bigger industrial kitchen with a number of stations,
wherein each chef is much less skilled and is instead specialized at performing a few 
specific (and much simpler) tasks extremely quickly.

All of these less specialized chefs work in parallel as well, processing huge batches of 
tasks at once. Imagine having one group of GPU chefs specialized for preparing meat, another
for cutting vegetables, and yet another for making soups. Each group will be given ingredients
and then all the groups and all the chefs will get to work, attempting to finish together.

This whole parallel simple tasks model is important, as a GPU is responsible for making
the thousands of pixels on your screen update (seemingly) all at once. A CPU could split
the work among it's threads, but the CPU's threads/cores are overqualified for this task
and would still take longer. A GPU can very nearly assign a single pixel and all the tasks
required to set it's color to a single thread, or a few threads chained in series. Effectively,
GPU's execute thousands of instructions per cycle (a GTX 1080 can fire out 2560) and 
CPUs are limited to a few orders of magnitude fewer instructions per cycle.

### Graphics APIs

These first few paragraphs will probably be redundant for people who have programmed 
before, so feel free to skip ahead. For those who haven't, let's get into it - 

An API is an application programming interface, or a way for programmers to interact
with something. This something could be an external "library" of functions and methods
of doing a task, an improved method of accessing a database, or even a way of controlling
hardware. The latter is what graphics APIs serve to do - making it easier for programmers
to control and talk to graphics hardware.

It probably goes without saying that directly talking to hardware in it's own language is
prohibitive if not impossible, and talking to it even via some middle layer of abstraction
is still going to be very verbose and difficult. Plus, it would change based on the hardware:
different manufacturers, versions, and series would all have slightly different ways of communicating.
So, an API helps to remove this burden by standardizing the interface to the hardware. This way,
code written via an API will work on any device that implements a "driver" for the API in question.