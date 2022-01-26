+++
title = "GPUs, Part 3: An Example Program"
date = "2022-01-11"
draft = true
+++

Now that we've seen some of the theory behind GPUs, it's time to see how it
works in practice. How can a lowly mortal like me speak to a Titan (heh)? What
do we need to know before going in? Comes with free wgpu tutorial.
<!-- more -->

## Shaders and the OpenGL model

There are some differing opinions on what to call a program that runs on a GPU.
It seems that people from graphics land call them shaders, because they are
generally used to calculate the colour or light level of an object. There are a
few types of shader and each has its own constraints, but the most generic one
and the one that we are interested in is the compute shader. In compute land,
they are called kernels, but there's already a thing called kernel and "shader"
sounds cooler to me so we're going with that.

Since there are so many types of GPU, programs tend to be stored in some
intermediate format and compiled to target a certain architecture at runtime.
This compilation is performed by the driver, which allows it to be
device-specific. We can see this reflected in OpenGL, where the API ingests
shaders written in GLSL. This means there is no ahead-of-time compilation step
needed if you're happy to write your shaders in GLSL. Vulkan takes a slightly
different approach by ingesting SPIR-V which is a binary format, one step below
the human-friendly GLSL, but still above the hardware in terms of abstraction.

### `gl_GlobalInvocationID`

Things get a little funky in this section, so hold on tight. Work done by
compute shaders tends to be intuitively divisible into chunks which exist in
some n-dimensional space. For example, a shader which modifies every pixel of an
800x600 image in some way can have its work divided into 800 * 600 = 480000
chunks, one for each pixel. These chunks can be named using the coordinates of
the pixel they work on, and so the chunks exist in 2-dimensional space.

In this case, the programmer can write a shader that operates on one pixel, and
in OpenGL terms, use `glDispatchCompute(800, 600, 1)` to run the compute shader.
The arguments to this function specify how many instances of the shader will be
run in each dimension of x, y, and z. Here, z is 1 because the image is
2-dimensional. This size is called `gl_NumWorkGroups`. But wait, I hear you say,
why is one unit of work called a work group?

Enter the funk, stage left. Recall that `gl_NumWorkGroups` tells us how many
*instances of the shader* will be run. Well, the shader itself has a
`gl_WorkGroupSize`, which is another 3-dimensional size that represents how many
times the code will be run, per instance of the shader.

This means that your code runs inside a... 6-orthotope? 🤔

I can't think of a great example for this, but suppose the image processing
shader actually operated on 4x4 groups of pixels. This is when you could use a
4x4x1 `gl_WorkGroupSize` and a 200x150x1 `gl_NumWorkGroups`.

All of these `gl_*` variables are accessible from within the shader, along with
a couple that I haven't mentioned. `gl_LocalInvocationID` is a 3D representation
of which unit within a work group is currently being run, and `gl_WorkGroupID`
represents which work group within the entire dispatch is being run. The most
important one is probably `gl_GlobalInvocationID`, which is a 3D identifier that
is unique across all invocations of your code. It is defined as
`gl_WorkGroupID * gl_WorkGroupSize + gl_LocalInvocationID`, which has the nice
effect of meaning that you can use it as an index into some data. In the image
example, the shader code could use `gl_GlobalInvocationID` directly to access
its respective pixel from the image. Nice!

## wgpu

APIs for working with GPUs are scary, they have a lot of long words, and
worst of all are usually designed for C programmers. wgpu is an exception to at
least one of these rules, because it provides a relatively nice Rust interface
while retaining design choices of things like Vulkan that allow you closer
access to GPU (or driver) behaviour. It also supports multiple backends, so you
can write code in terms of wgpu and run it on systems supporting Vulkan, Metal,
and a couple of versions of DirectX, among others.

Before anything can be done with wgpu, you must start with an `Instance`. While
creating this object, a backend is chosen. Then, this instance can be used to
create an `Adapter`, which is like a physical device. At this point, you can
specify whether to use a low power adapter or a high perfomance one, and provide
information about window systems if you intend to render to a window. The next
link in the chain is called `Device`, which is more like a session on a device
than the device itself. `Device` is part of a package deal with `Queue` - the
former allows control over resources and the latter allows control over actions.

When shaders are running, they need to have access to some objects to work on.
There are a few types of these, but the simplest is the buffer, which is a
contiguous region in memory. Objects like these are the main way for GPUs to
communicate with the rest of a computer, and they can be described using the
methods on `Device`.

To start with, these methods can be used to create a `BindGroupLayout` which is
a template for a group of objects and their properties. For example, a
`BindGroupLayout` might describe two read-only buffers which are visible to
compute shaders and are at least 4096 bytes in size. Then, a `BindGroup` can be
made which has this layout, by naming two buffers with those properties.

To use a compute shader, you need a "compute pipeline", which is an object that
holds the shader and some bind group layouts. Telling a GPU to run a shader
(after you've done all the setup) is done using command buffers, which are built
using command encoders. These commands have semantics like "run this compute
pipeline using these bind groups", and such a command also contains a size in x,
y, and z, which are equivalent to the arguments we saw being passed to
`glDispatchCompute` earlier. The command buffer can be submitted to the GPU via
the `Queue`.
