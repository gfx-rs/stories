---
layout: post
title:  "Franzplot: a teaching software (re)written in Rust"
author: Francesco Cattoglio
categories: [ stories ]
---
# Franzplot: a teaching software (re)written in Rust

## A bit of background
My name is Francesco Cattoglio and I have been a research assistant at Politecnico di Milano for a few years. Since 2018 I have taken up the role of teaching assistant for a class titled "Curves and Surfaces for Design". The goal of this class is to explain some 3D math concepts to Design students. As with everything related to mathematics, grasping some abstract concepts without practical examples can be tricky, so we spend about 15-20 hours doing computer-based exercises. As soon as I started my lessons I saw the students struggling with the tool we were using, so I proposed coding a new one from scratch. And we did! In just four months of part-time working I managed to scrap together the first version of the new teaching software. Since my nickname at the office was Franz, my supervisor kept referring to it as "FranzPlot". Even if it did sound a bit funny, in the end we kept that name (and my supervisor still tells me she is very sorry for that ridiculous choice! 😄)

## First version
To use this software the student creates a graph of nodes which contain some equations describing a given set of geometrical objects (parametric curves and surfaces) and some transformation matrices. Then, when the user is done, the CPU computes those shapes and the scene can be inspected to see what those objects look like.
The first version was written in C++, and used OpenGL via the [Magnum Engine](https://magnum.graphics/), which is a really amazing piece of work considering the very small team that put it together. I would really like to thank mosra (Vladimír Vondruš) for making the first version of the Franzplot possible! Imgui was used to create the node editor interface which, as far as teaching goes, is the most important part of the whole project.

Here is the node graph editor:
![Node graph editing](https://github.com/francesco-cattoglio/stories/raw/main/assets/img/franzplot_nodes.png)

And here is the scene rendering: (both screenshots are from the new Rust version, not the C++ one)
![Node graph editing](https://github.com/francesco-cattoglio/stories/raw/main/assets/img/franzplot_scene.png)

## RIIR! (Rewrite It In Rust)
If the first version was C++, why was the second version built in Rust?  All the people involved saw the potential and decided to turn this side project into a "real" software. However, to take things to the next level many things needed addressing. First and foremost, the amount of technical debt was rather high. Second, OpenGL has already been deprecated by Apple and we could not afford the risk of having nearly half of our students being unable to run the software in the future if Apple decided to remove support altogether. Third, we wanted to make the tool more powerful, flexible and available as a web app as well.

To enhance the interactivity of the software we needed to be able to recompute *everything* at least 30 times per second, so the formula interpreter that we were using would not cut it anymore. After realizing that even a cheap laptop iGPU has a huge amount of flops, I decided to go for a compute-based solution, but how can one have compute shaders on a webpage? The answer would be WebGPU, a new graphics programming API that aims to be a future standard to expose modern hardware capabilities to the web. But wait, there is more! Although the standard was initially created for the web, the API itself is a very modern and ergonomic one, and implementations for native platforms are being written right now. This means that **you can have a single code base that can easily target the web as well as Windows, MacOS and Linux!**

I first read of WebGPU and its Rust implementation _wgpu_ around March 2020. It was clear that everything was still a work in progress, but I already knew the amazing work made by the gfx-rs team and I was confident that wgpu would have been the right bet. More than a year later I firmly stand by that assertion.
Up until this point I had very little Rust experience, but I was already in love with the language. During a meeting I explained my supervisors that it would not have been an easy nor a quick job, but I was sure that the Rust+wgpu combo would have been the right choice going forward: by using a modern language and a modern gpu API we would be able to deliver a native-only version for all the desktop platforms while keeping the road open for a web target in the future.

After many months of learning, opening github issues, getting help and advice from Kvark, Cwfitzgerald and many others from the community, we finally went live with the new version. The core of the software is Rust, while the UI is still powered by `imgui`, using [`cxx`](https://cxx.rs/) as a bridge to the [`imnodes`](https://github.com/Nelarius/imnodes) library for the node graph editor. Most of the puzzle pieces fell right into their place, only requiring a minimal nudge to work together.
The new version was a success: even though some bugs needed ironing, the final product has been very solid so far. There are even a couple students that are unable to run the *old* OpenGL version on their laptops, but the new one runs just fine!

## Technical details
Franzplot uses wgpu for both compute and visualization purposes. Every time the "Generate Scene" button is clicked, a few things happen:
- the software checks that the graph does not contain any error nor cycles, and nodes get sorted according to their dependency
- for each node, the memory requirements for its output are computed and a wgpu buffer gets allocated
- each node's equations are turned into a compute shader that reads from the input buffers and writes to the output one

Just as an example, imagine you have this graph in which node 2 depends on both 1 and 4, node 5 only depends on 4 and node 3 depends on both 2 and 5.

![Node graph editing](https://github.com/francesco-cattoglio/stories/raw/main/assets/img/nodes_graph.png)

This will get converted into a vector containing the following objects, which I call "compute blocks"

![Node graph editing](https://github.com/francesco-cattoglio/stories/raw/main/assets/img/processed_nodes.png)

You can see from the above drawing that each block owns a unique compute shader (created from the student's equations) and its output buffer, but only has *bindings* to the buffers it uses as inputs. This is because a wgpu buffer has strict ownership rules. Thanks to bindings however we can create a handle for reading from a buffer whenever we like (to be precise, a compute block does not store individual bindings, but a bind group, and not just a shader, but a compute pipeline!).
Since we sorted the compute blocks based on dependency, we just need to iterate over the vector and submit all the computations to a wgpu queue.

The whole conversion process takes very little time, less then one second for a reasonably-sized node graph. In a sense, FranzPlot is "just" a compiler: it turns mathematical formulas into GLSL. Everything else is just some glue code with a super simple 3D scene visualization on top: basic triangle meshes with a matcap shader and a black & white texture on top!

When the user switches to the scene visualization all the shaders are run in the correct order, and this updates all the buffers that make up the displayed meshes. When the student changes the value of a few pre-defined global variables (stored in a uniform buffer) all the shaders are run again and the scene is updated in real time! Here is a gif of FranzPlot in action:

![Node graph editing](https://github.com/francesco-cattoglio/stories/raw/main/assets/img/franzplot_scene.gif)


## Current state and future developments
FranzPlot is currently closed source, since we are still trying to figure out what the next steps should be. It might become open source in the future, but there are many things that we need to consider first. Once a decision is made I hope we manage to spark some interest into this software and keep expanding its features. It would be nice to find some universities with similar classes that need an easy-to-use tool to help the students understand advanced geometry concepts.

W.r.t. the actual code, there are still many things to do: even though the new code was written from scratch, I still feel like some technical debt crept in and there are a few changes I would like to make to the internals. I have already planned a few features that will be fun and challenging to add, and I would *love* to ditch GLSL completely and move to WGSL (WebGPU Shading Language), since it has matured a lot.
Finally, even though I enjoyed working with imgui-rs & imnodes, I would like to find a rust-only solution for the UI. Right now I am investigating [`egui`](https://github.com/emilk/egui) as a possible alternative, but this is kind of low-priority.
