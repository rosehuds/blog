+++
title = "GPUs, Part 1: The Feynman Technique"
date = "2021-11-12"
+++

There's a lot of writing about the Feynman Technique, but essentially it boils
down to the idea that if you want to learn something, you should try to
explain it to a 12-year-old. I'm not a 12-year-old, but I am a first year
compsci, which is basically the same thing. I have two problems with GPUs.
Firstly, I don't understand them. Secondly, I don't know of any
12-year-old-level resources to learn with. Here, I'll try to kill two birds with
one stone.
<!-- more -->

In this series I'll explore GPUs as general-purpose compute engines, their
characteristics, and how one can learn to harness them for good. During this
journey I'll try to document what I find in a way that allows any curious
computer scientist to follow along at home. We can worry about the 12-year-olds
later.

My favourite language is Rust and I don't really know how to write anything else
so I'll be aiming to use only Rust, at least on the CPU side. If possible, I'll
use [rust-gpu](https://github.com/EmbarkStudios/rust-gpu) for GPU-side code as
well.

## Why bother
GPUs are becoming more relevant as new people come up with new work to do, and
Moore's law fails to deliver on its promise of increasing single thread
performance to do it with[^1]. GPUs sidestep this problem by adding more
threads. Their main strength is in tasks where some operation is applied over
many inputs independently (like with SIMD - more on that later), but with some
clever tricks they can beat CPUs in tasks that at a glance don't seem parallel
at all. It's hard to turn a traditional sequential program into one that plays
nicely on a GPU, but depending on the workload, it can really pay off. This
transformation is what I aim to get familiar with.

{{ quote(text="They do, like, images and shit, right?", author="My friend Drew") }}

Although they are big and scary and foreign, I think that with some time and
some honest work even a first year like me could tame the beast. I want to,
because they're really fast, and it would make me feel powerful and smart. Your
motives may be different, and that's OK.

To be clear, I'm not too bothered about the traditional 3D graphics pipeline
with its ROPs and TMUs and geometry. Although it would be nice to understand
that stuff, I'll be focusing more on general computation to start with.

## Goals
My personal goal for this series is to be able to understand
[piet-gpu](https://github.com/linebender/piet-gpu) from start to finish.
piet-gpu is a 2D vector graphics renderer designed to leverage GPU compute to do
what has historically been done on CPUs. Whether I'll be able to do this will
probably become clear in a few months, but really it's not about the treasure,
it's about the friends we made along the way.

To get to piet-gpu, by my estimates, we'll need to understand a few things about
GPUs:
* GPU architecture - how do they run programs?
* CPU-side APIs - how to tell a GPU to run a program
* I/O - moving data between GPU memory and main memory
* Algorithmic techniques - taking tasks that look sequential and reframing them
  until they can be done efficiently in parallel

## Conclusion
This will be a fun exercise in self-motivation I guess. I've learned some cool
stuff so far, so I'm optimistic about how far we'll get. The next post, a first
look into GPU architecture, should come quite soon. I hope to see you there!

[^1]: This isn't really what Moore's law is, but it's true that single thread
  performance isn't growing in the same way it used to.
  [Here](https://preshing.com/20120208/a-look-back-at-single-threaded-cpu-performance/)
  is an article from 2012 showing the effect already being visible.

***

{{ prev_next(prev="", next="@/gpus/2-architecture.md") }}
