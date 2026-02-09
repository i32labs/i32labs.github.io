+++
title = 'Nebula'
date = 2026-02-06T00:00:00-00:00
draft = false
+++

## Introduction

![Nebula Logo Dark](nebula_logo_dark.png)

Nebula is a D3D12 based GPU driven renderer using modern bindless techniques.

## Minimum Requirements

Nebula makes use of dynamic resources and D3D12 Shader Model 6.6. This places a hard minimum system requirements for the engine.
The recommended requirements bring in support for Enhanced Barriers. These are not currently used, but may become minimum requirements in the future.

| GPU | Minimum | Recommended |
|---|---|---|
| AMD | GCN3 | RDNA1 |
| Nvidia | Maxwell1 | Maxwell1 |
| Intel | Gen11 | Xe-HPG |
| Qualcomm | X1 | X1 |

CPUs make use of the .NET 11 baseline.

| CPU | Minimum |
|---|---|
| x86_64 | x86-64-v2 | 
| arm64 | armv8.0-a + LSE |

Windows 10, 1909 is required (or Windows Server 2022). Please pay attention to the required revision numbers.

| Windows | Version | Revision |
|---|---|---|
| Windows 10 | 1909 | 1350 |
| Windows 10 | 2004 | 789 |
| Windows 10 | 20H2 | 789 |

## Useful Resources

- https://asawicki.info/articles/state_of_gpu_hardware_2025.php


## Design Goals

### Two Root Signatures

`Nebula.Gpu` was originally born as a simple D3D12 wrapper mimicking the wgpu api. As we move away from this system of "bind groups" to fully bindless, we are making a few changes. Namley, only two root signatures generated at startup used for all shaders. This follows a similar approach to Frostbite and Cyberpunk 2077.

#### Compute Root Signature

#### Graphics Root Signature

What would a graphics root signature hold? Usually this is data that is not bindless, like uniform / constant buffers (view, proj, pos, time etc.)

### GPU Only Descriptor Heaps

As we're a bindless engine, CPU only staging descriptor heaps are no used, resources are allocated directly to GPU visible heaps, and indexed within the shader.

This does not apply to RTV and DSVs.

It's important that we don't free these resources while the GPU is using them, otherwise we will encounter undefined behavior. It's also especially important that we don't mess up what type of index we point towards. This is done by reference counting usages.

We may need to pair UAV and SRV descriptors together!

### 100% GPU Driven

Scenes are culled on the GPU using a compute shader, draw calls are setup in draw buffers.


## Bindless

> test

### Descriptor Heaps & Descriptor Handles

Whenever a resource is created in Nebula (buffer, texture or sampler), one or more descriptors are assigned to the resource based on its usage. These descriptors are stored in a universal GPU visible descriptor heap - and are cleaned up when safe.

You can obtain the internal index to a resource descriptor by calling the `GetDescriptorIndex(DescriptorType type)` method on the resource. 

Depending on the value of `DescriptorType` will will retrieve the following:

- **ConstantBuffer**: CBV
- **Read**: SRV
- **Write**: UAV

If the resource has not been setup with the appropriate usages, an exception will be thrown. An exception will also be thrown if attempting to access `ConstantBuffer` on textures.

It's important to note that this index is only valid as long as the resource is alive, descriptor indices are reused as needed.


Resources are accessed within GPU shaders via descriptor handles. Nebula provides two different types of descriptor handles depending on your goals.

###  `DescriptorHandle<T>`

DescriptorHandle's are a one-to-one match with slangs DescriptorHandle. They provide type safety both in c# and slang itself, with the expense of double memory usage (2 uints vs 1 uint). In most cases you should be fine to use this, and the extra type safety is a massive benefit.

If you are short for space (e.g. a push/root constant), then Nebula provides `LiteDescriptorHandle<T>`.

Usage:

1. First define the descriptor in a struct (in this example we are creating a push constant)

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 4)]
private struct ComputeConstants
{
    public DescriptorHandle<GpuBuffer> IndirectArgsBufferIndex;
    public DescriptorHandle<GpuBuffer> IndirectCountBufferIndex;
}
```

2. Next grab the descriptor indices from your resources you wish to access in the shader

```csharp
var computeConstants = new ComputeConstants
{
    IndirectArgsBuffer = indirectArgsBuffer.GetDescriptorIndex(DescriptorType.Write),
    IndirectCountBuffer = indirectCountBuffer.GetDescriptorIndex(DescriptorType.Write),
};
pass.PushConstant(ref computeConstants);
```

3. In your shader you can access as following

```hlsl
struct ComputeConstants
{
    DescriptorHandle<RWStructuredBuffer<DrawIndexedArguments>> indirectArgsBuffer;
    DescriptorHandle<RWBuffer<uint>> indirectCountBuffer;
};

[[vk::push_constant]]
ComputeConstants constant;

[shader("compute")]
[numthreads(1, 1, 1)]
void main(uint3 dispatchThreadID : SV_DispatchThreadID)
{
    DrawIndexedArguments args;
    args.IndexCountPerInstance = 3;
    args.InstanceCount = 1;
    args.StartIndexLocation = 0;
    args.BaseVertexLocation = 0;
    args.StartInstanceLocation = 0;

    constant.indirectArgsBuffer[0] = args;
    constant.indirectCountBuffer[0] = 1;
}
```

###  `LiteDescriptorHandle<T>`

A LiteDescriptorHandle is simply a wrapper around a single uint32 value. Within your shader code you'll specify uint within structs and use `.view<T>` to access the underlying resource. No type checking is done!

Usage:

1. First define the descriptor in a struct (in this example we are creating a push constant)

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 4)]
private struct ComputeConstants
{
    public LiteDescriptorHandle<GpuBuffer> IndirectArgsBufferIndex;
    public LiteDescriptorHandle<GpuBuffer> IndirectCountBufferIndex;
}
```

2. Next grab the descriptor indices from your resources you wish to access in the shader

```csharp
// Implicit conversion from DescriptorHandle to LiteDescriptorHandle
// C# type validation still available
var computeConstants = new ComputeConstants
{
    IndirectArgsBufferIndex = indirectArgsBuffer.GetDescriptorIndex(DescriptorType.Write),
    IndirectCountBufferIndex = indirectCountBuffer.GetDescriptorIndex(DescriptorType.Write),
};
pass.PushConstant(ref computeConstants);
```

3. In your shader you can access as following

```hlsl
struct ComputeConstants
{
    // Note how these are raw uints
    uint indirectArgsBufferIndex;
    uint indirectCountBufferIndex;
};

[[vk::push_constant]]
ComputeConstants constant;

[shader("compute")]
[numthreads(1, 1, 1)]
void main(uint3 dispatchThreadID : SV_DispatchThreadID)
{
    DrawIndexedArguments args;
    args.IndexCountPerInstance = 3;
    args.InstanceCount = 1;
    args.StartIndexLocation = 0;
    args.BaseVertexLocation = 0;
    args.StartInstanceLocation = 0;

    // Note the usage of .view<T>()
    constant.indirectArgsBufferIndex.view<RWStructuredBuffer<DrawIndexedArguments>>()[0] = args;
    constant.indirectCountBufferIndex.view<RWBuffer<uint>>()[0] = 1;
}
```