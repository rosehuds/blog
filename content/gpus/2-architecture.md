+++
title = "GPUs, Part 2: Architecture"
date = "2021-11-14"
+++

What kind of maths can we do on a GPU? How does it get done so quickly? What the
hell is a "warp"? Find out all of this and more in today's episode.
<!-- more -->

## Parallel processing
In CPU land, you can sometimes speed up data processing by splitting the data into
parts, and processing multiple parts at the same time. There are two ways of
doing this.

### Multiprocessing

The most common way is multiprocessing, where there are multiple CPUs within a
computer, and code can be run on all of them at the same time. In this case,
it's as if there are multiple computers all working on a portion of the task, except
the CPUs in a multiprocessing system share resources such as main memory to make
communication between them much faster than if they were in separate computers.

Writing programs to take advantage of such an architecture is fairly easy
because once the task can be split into parts, the development process feels
like writing separate programs to handle each part. Sometimes it's the same
program that handles every part, in which case you only have to write one
program, and life is good.

### SIMD

On the other end of the spectrum, many modern CPUs support a method of processing
called SIMD - Single Instruction (stream) Multiple Data (streams). In a CPU with
SIMD, there is still only one control flow path. That is, when the CPU reaches
some control flow in the program such as an if statement or loop, the CPU either
is executing the body or it isn't. Hence, "single instruction stream".

Although there can only be one instruction executing at a time, these
instructions can perform the same operation on multiple sets of operands
simultaneously. A processor that can operate on 8 sets of operands in this way
is said to have 8 SIMD lanes. For example, the `vmulps` instruction from x86 can
be used to multiply 8 pairs of numbers, producing 8 results. Each pair is
processed independently of the others, and they are processed in parallel.

This style of computation is very well suited to a select few applications. For
example, to change the volume of a buffer of audio samples, each sample must be
multiplied by some scale factor. In this case, SIMD can be used without much
thought, and if you're lucky your compiler will transform the naïve for loop
implementation into machine code that makes use of SIMD instructions. This
transformation is called auto-vectorisation, and it can be demonstrated with
the following Rust function:

```rust
pub fn scale(buffer: &mut [f32; 8]) {
    for value in buffer.iter_mut() {
        *value *= 4.0;
    }
}
```

Compiling with `-Ctarget-cpu=haswell` to enable AVX2 gives the following
assembly:

```
<simd::scale>:
c4 e2 7d 18 05 00 00    vbroadcastss 0x0(%rip),%ymm0
00 00
c5 fc 59 07             vmulps (%rdi),%ymm0,%ymm0
c5 fc 11 07             vmovups %ymm0,(%rdi)
c5 f8 77                vzeroupper
c3                      ret
```

Here we can see only one instruction actually doing any multiplication
(`vmulps`) because this instruction can operate on registers containing 8 values
each. Since the function only operates on 8 values, there aren't even any
branches in the body.

However, auto-vectorisation is really hard, and you kind of have to cross your
fingers when compiling if you want to take advantage of it. In GPU land, the
processors themselves are structured a little differently, and this makes the
parallelism easier for the compiler to take advantage of. It can also make
programming these processors easier, as we'll see further down.

## How GPUs do it

Now, we meet something called predication. I don't think many people use this
term, and I had to reverse engineer it from some poorly written Wikipedia
articles, but the concept was introduced to me fairly clearly in Alyssa
Rosenzweig's talk The Occult and the Apple GPU. Thanks to Alyssa not only for
the talk but also for responding to my email with a very detailed explanation of
how GPU architects are constantly battling each other to redefine words in
different ways. We'll meet some of these later.

The idea is that SIMD is pretty cool, but since it doesn't allow for diverging
control flow across lanes, it's not cool enough. Predication allows each lane in
a SIMD-like processor to independently be enabled or disabled, and this can be
done using an execution mask, which is a set of bits with one bit per lane; if a
bit is zero, its corresponding lane does not write to any registers or memory,
so it is effectively doing no work at all.

Since we're in GPU land now, as promised, we have some new words to learn. The
lanes in a GPU are more powerful than traditional SIMD lanes, so we call them
"threads", to indicate that they're cooler and more flexible. The threads come
in groups called "wavefronts" or "warps". All threads in a warp share a program
counter, meaning threads in a warp cannot execute different instructions at the
same time. The execution mask for a 32-thread warp can either be seen as 1 bit
per thread, or a single 32-bit mask per warp, depending on which way you look at
it.

Here's a visual example of divergence in a program running on a
hypothetical 4-thread warp. The lines down the side represent whether that
thread is enabled at that point in the program.

```
0 1 2 3
| | | |     let mut array = [0, 1, 2, 3];
| | | |
| | | |     if array[thread_id] % 2 == 0 {
|   |           array[thread_id] += 1;
|   |       }
| | | |
| | | |     // at this point the array is [1, 1, 3, 3]
```

Because there is only one program counter per warp, if the condition of an if
statement evaluates to false in a given thread, that thread has to wait it out
by staying disabled until the whole warp gets to the end of the statement. This
means that if you have to use an if statement, it's best to keep the body short
and/or to have a condition that is usually true, to maximise how many threads
are enabled at once.

### But why?

By adding predication to SIMD, we give ourselves the ability to write programs
in a way that looks much more like the single-threaded scalar processing that we
are all used to. The program from above, written using SIMD, might look like
this:

```rust
let mut array = u32x4::new(0, 1, 2, 3);
let one = u32x4::splat(1);
let mask = (array & one).eq(one);
array = mask.select(array, array + 1);
```

This differs from the GPU-style program above in one striking way: this program
describes calculations in terms of vectors (`u32x4`, i.e. a group of four
integers) whereas the GPU-style code describes the calculations individually,
achieving parallelism implicitly by using the `thread_id` to index into the
data, which splits the data into independent chunks. This means that the
GPU-style code can be used on any size of array without modifying the code
itself.

Another important difference is that the GPU-style code looks to me to be far
easier to write (and it was), partially due to this scalar/vector code style
distinction. Furthermore, the resulting machine code of the SIMD program is
probably quite inefficient due to having to save all of the results of `array +
1` into registers even though it doesn't end up using them. So, SIMD is good
for programs that need to do the same thing a bunch of times, but predicated
SIMD is good for programs that need to do *nearly* the same thing a bunch of
times, which is way more programs.

## Instruction sets

At the beginning of this post, I asked "what kind of maths can we do on a GPU?".
The answer will vary depending on the GPU, so I looked at the manual for AMD's
GCN 3 architecture[^1], because I have a GCN 4 card, and apparently they're similar
enough that AMD didn't release a new manual.

GCN 3 provides all the arithmetic stuff you'd expect; add, sub, mul, div on
integers and single-precision floats are there along with some logical
operators, but it also has loads of others. There are a few convenient
instructions for graphics programming like trig functions, square root,
reciprocal square root, and even cubemap helpers. As if we weren't spoiled
already, there are some logs and exps as well.

Skimming some documents for Apple's G13[^2] and Arm's Bifrost[^3] instruction
sets indicates that most of these instructions are fairly standard.

## Conclusion

CPUs have some ways to achieve parallelism, but they are restricted by the way
people expect to be able to write programs for them. GPUs had no such
restriction, so their designs can lean into parallelism as much as they want. As
we've seen, this leads to an unfamiliar but powerful architecture.

In the next few posts, I'll explore writing and running programs on a GPU, and
some real-world examples of the architectural concepts we've met so far.

[^1]: <https://developer.amd.com/wordpress/media/2013/12/AMD_GCN3_Instruction_Set_Architecture_rev1.1.pdf>

[^2]: <https://dougallj.github.io/applegpu/docs.html>

[^3]: <https://cgit.freedesktop.org/mesa/mesa/tree/src/panfrost/bifrost/ISA.xml?id=07a5ec83fb09de861d940fea69b49cefb08fda75>

***

{{ prev_next(prev="@/gpus-1.md", next="") }}
