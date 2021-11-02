+++
title = "GPUs, Part 1: The Feynman Technique"
date = "2021-11-02"
+++

There's a lot of writing about the Feynman Technique, but essentially it boils
down to the idea that if you really understand something, you will be able to
explain it to a 12 year old. There are two problems with GPUs - I don't
understand them, and I don't know of any 12-year-old-level resources to learn
with. Here, I'll try to kill two birds with one stone.
<!-- more -->

## Huh?
GPUs are becoming more relevant as new people come up with new work to do, and
Moore's law fails to deliver on its promise of increasing single thread
performance to do it with. GPUs sidestep Moore's law by adding more threads.
Their main strength is in tasks where some operation is applied over
many inputs independently (like with SIMD - more on that later), but with some
clever tricks they can beat CPUs in tasks that at a glance don't seem parallel
at all. It's hard to turn a traditional sequential program into one that plays
nicely on a GPU, but depending on the workload, it can really pay off.

{{ quote(text="They do, like, images and shit, right?", author="My friend Drew") }}

Although they are big and scary and foreign, I think that with some time and
some honest work even a first year like me could tame the beast. I want to,
because they're really fast, and it would make me feel powerful and smart. Your
motives may be different.

To be clear, I'm not too bothered about the traditional 3D graphics pipeline
with its ROPs and TMUs and geometry. Although it would be nice to understand
that stuff, I'll be focusing more on GPUs as tools for general computation.

## Goals
My personal goal for this series is to be able to understand
[piet-gpu](https://github.com/linebender/piet-gpu) from start to finish.
piet-gpu is a 2D vector graphics renderer designed to leverage GPU compute as
much as possible. Whether this goal is reasonable will probably become clear in
a few months, but really it's not about the treasure, it's about the friends we
made along the way.

To get to piet-gpu, by my estimates, we'll need to understand a few things about
GPUs:
* CPU-side APIs - how to tell a GPU to run a program
* I/O - moving data between GPU memory and main memory
* Program structure - how parallelism can be achieved
* Algorithmic techniques - taking tasks that look sequential and reframing them
  until they can be done efficiently in parallel
