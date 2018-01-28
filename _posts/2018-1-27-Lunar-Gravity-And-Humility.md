---
layout: post
date: 2018-01-28
title: "Lunar Gravity Models and a lesson in Humility"
published: false
img: moon.jpg
tag: [Space, NASA, Legacy Code, C++, Spherical Harmonics]
---

My current work project has involved performing some fascinating and _really_ hard math
and modelling work for Goddard Space Flight Center, in support of the potential [BOLAS](https://www.nasa.gov/feature/goddard/2017/nasa-studies-tethered-cubesat-mission-to-study-lunar-swirls)
mission. This has entailed diving into a codebase written primarily from 1990-1993, giving
it a facelift, and expanding it's functionality. The whole "legacy code" part of my trip into
this codebase could and probably will make up an entire post of it's own someday, but 
for this post I'll primarily be talking about the gravity model we're using, how I built and 
iterated upon it, why we're studying the moon with a tethered spacecraft, and finishing with a 
recent and thoroughly eviscerating (for me, of course) lesson in humility.

## The Lunar Gravity field

<img src="{{site.baseurl}}/assets/img/lunar_g_accel.jpg" alt="The difference is deceptively small, but it's really quite severe" />


... is _sorta kinda_ weird. As in really weird. It was known shortly before the Apollo missions
that things weren't quite "right" after anaylsis of the Lunar Orbiter tracking data - itself a 
sort of navigation test to prepare for the upcoming Apollo missions. It was understood then that
the largest anomalies were due to mass concentrations on the Lunar maria, and this does make some
degree of sense. These regions are primarily made of very dense (relative to most of the lunar regolith)
basaltic lava.

> Interestingly, the moon's geometric center and center-of-mass are not the same: the center of mass is ~2 kilometers closer to the Earth

_However_, this alone can not describe the magnitude of the anomalies. So the composition
of these regions is only part of the answer, and the rest is up for debate and discovery. We do, however,
know enough to build an accurate model allowing us to compute accurate trajectories for low-lunar orbits.

## Astrodynamic Models, or, "close enough"

It might seem that absolute precision is required to implement feasible models for computing spacecraft
trajectories - but that's only partially correct. There's a balance between computability (or at least, 
the speed at which computations proceed) and the obvious need for accuracy since an incorrect trajectory
differing only in meters could potentially spell disaster for a mission.

But the amount of factors one must consider when computing a trajectory are rather dazzling, as we can
include:

- solar pressure, or the force of the solar wind on sunward elements of a spacecraft

- minute atmospheric drag forces - even the space station, for example, experiences some atmospheric drag at it's altitude in LEO

- as an additional note on the above, the sun's relative activity level can also cause the thermosphere to expand, increasing atmospheric drag

- drag due to the local magnetic field, itself also affected by the sun

- tidal effects, due to the shifting of mass with the tides

- even the lower-atmospheric tides have recently been added to models!

- in the case of tethered spacecraft, the temperature of the Tether changing the tether material's properties also affects orbit propagation

- n-body perturbation effects - e.g, the Moon for objects in LEO, or most of the solar system for craft in deep space

- and of course, the gravity field of the primary body influencing the simulation

If you'd like to read more on the complex topics of atmospheric drag (it's kinda fascinating!), I found [this
excellent presentation](https://ccmc.gsfc.nasa.gov/RoR_WWW/SWREDI/2015/SatDrag_YZheng_060415.pdf) while verifying 
things I've written in this article. Clearly, though, modelling a trajectory is going to require quite a bit 
of computation and careful inclusion of myriad external influences.

Because of this complexity, though, it's not really possible to compute an exact trajectory for _any_ mission. 
Often, "close enough" is more than good enough as most spacecraft will feature maneuvering thrusters. In the 
case of satellites only living in LEO with no maneuvering abilities, they benefit from our experience in modelling 
Low-Earth orbits and usually are given orbits that will last throughout their mission lifetime.

We can make simplifying assumptions and adjustments to our models though, usually based on the mission requirements
or the particular bodies the mission is focusing on. In the case of my work, we don't have to worry about magnetospheric,
ionospheric, atmospheric, etc sources of drag. Tidal effects also aren't a consideration we need. Things like solar 
pressure can be discarded after a bit of thought (at least for initial models), as it's likely the lunar gravity field
is going to be much more chaotic and much more important to consider than the effects of solar pressure (also, we're
using a thin rope tied between two 3U cubesats - not a huge area for solar pressure to act upon).

So our primary considerations become the gravity field, a tiny contribution from Earth's tug on a lunar satellite, and 
a whole bunch of work relating to the dynamics of tethered spacecraft. 

## "Nonspherical Gravitational Perturbation Model"

Or, translated from a wicked complex (but useful) document published by Goddard - how to model the gravity field of nonspherical
bodies (most of them) and account for their own unique mass-density anomalies in said model. A copy of the Goddard document I'm currently 
referencing can be found [here](http://www.ltas-vis.ulg.ac.be/cmsms/uploads/File/GoddardTrajectorySystem.pdf), for those so inclined 
to cause what is probably just short of mental self-harm.

Unsurprisingly, most of the planets and moons in the solar system aren't really that close to being perfectly spherical. Besides that, 
they too have non-uniform densities throughout their interiors. The Earth isn't just somewhat oblong - there's also a higher density
region just under the crust (in the mantle) near the equator. 

> In Martian models, consideration is given (via extra parameters) to seasonal variations in mass-density at the poles caused by the sublimation-condensation cycle of carbon dioxide at the Martian poles

These various anomalies in gravity fields have resulted in most models using spherical harmonics. This contrasts with what most people
learn and use in highschool physics for calculating gravitational forces:


## Tethered Spacecraft Dynamics

Tethered spacecraft exploit gravity and centrifugal force as a core element of their stability and use - the tether itself
often has a length measured in kilometers (my current model uses 25 kilometers), leaving the two end elements separated
by no small distance. Over this distance, there will be a difference in the gravitational force experienced at each end. 
This will cause the two ends to be somewhat "pulled" apart, keeping the tether taut and stopping it from going slack (at which
point things get delicate, bad, and difficult _right_ quick). This force will cause the tether to rotate around it's own center
of mass once per orbit period, keeping the lowest element closest to the body being orbited.

<img src="{{site.baseurl}}/assets/img/tether_equilibrium.jpg" />

This isn't the difficult part of the simulation, ultimately. What's more difficult is modelling the propagation of forces
up and down the tether. Lets say we use gyroscopes to point the upper spacecraft, in order to transmit data back 