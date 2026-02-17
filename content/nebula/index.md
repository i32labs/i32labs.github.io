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

## Technical Limits

The below table describes the technical limits of Nebula, you will most likely encounter GPU / CPU bottlenecks before you reach these limits.

| Area | Limit | Notes |
|---|---|---|
| Meshes | 16,384 | |
| Mesh LODs | 65,536 | |
| Vertices | 128 MB | |
| Indices | 128 MB | |
| Instances | 65,536 | |
| Skinned Instances | 4096 | |
| Materials | 65,536 | |
| Unique Pipelines | 128 | |

## Useful Resources

- https://asawicki.info/articles/state_of_gpu_hardware_2025.php


## Design Goals

Nebula is designed as a modern bindless gpu driven rendering engine. What does this mean?

All scene resources are stored in persistent large buffers on the GPU. This includes things like meshes, lods, materials, instances etc. Resources interact and depend on each other via indices into the global buffers. Notably Nebula uses a global vertex and index buffers for all meshes (inspired by ID tech and other modern renderers). This allows the GPU to access all vertex data without the need of cpu side commands.

Scenes are culled on the GPU using a compute shader, draw calls are setup in draw indirect buffers. The CPU does not issue individual draw calls, but instead grouped calls based on batched passes using MultiDrawIndirect. All render pass data is sent to the GPU via a root constant (indices pointing towards the srv descriptors of the buffers).

We keep pipeline count low and keep root signatures even lower at only two root signatures which are generated at startup used for all shaders. This follows a similar approach to Frostbite and Cyberpunk 2077. By default we allocate 128 byte for a dynamic push / root constant, and provide CBV access to shaders that may need it (such as camera position data).

### Descriptor Heaps & Descriptor Handles

CPU only staging descriptor heaps are no used, resources are allocated directly to GPU visible heaps, and indexed within the shader. This does not apply to RTV and DSVs.

It's important that we don't free these resources while the GPU is using them, otherwise we will encounter undefined behavior. It's also especially important that we don't mess up what type of index we point towards. This is done by reference counting usages.

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

## Assets

### Classes & Types

#### Assets<T

The assets storage is a generic collection that contains all assets for a specifid asset type `T`.
Assets are ref counted and accessed via a `Handle{T}`. We also support the concept of a default asset for
cases where an asset cannot be found, is still loading or is invalid.

Assets can be added, removed and queried from the storage. Although if loading from disk, assets should be loaded
via the `AssetLoader`

#### Handle<T

A lightweight reference to an asset stored in `Assets{T}`. Instead of directly accessing and storing assets
we use handles. Handles also allow us to dynamically replace the asset backing the handle (such as hot-reading).

#### `AssetManager`

The AssetManager is responsible for loading assets from disk. If provided with the same uri+type it will
return the same `Handle{T}` ensuring resources are only loaded once.

Internally the asset loader will add the asset to an appropriate asset storage via `Assets{T}`.

#### AssetProvider<T

The asset provider is registered for each support asset type `T`. It is responsible for the actual loading of the asset.

Assets providers are also responsible for "cooking" assets, that is turning them into `.nasset` files which nebula can load without any processing.

For example: When loading a mesh, lots of work is done to process vertices, optimize, generate LODs etc. Doing this every time on startup is expensive, instead
Nebula will process an asset if it does not exist (or is outdated) and store it next to your executable for next launch and distribution.

Asset cooking is entirely optional and does not have to be implemented within an AssetProvider.

When providing a custom asset you will need to implement an asset provider for your type.


### Default Supported Assets

These assets are supported out of the box:

- Font
- Texture
- Audio

Generally speaking you'll have to add support for your own asset types such as Meshes, Materials, Shaders etc.

### Asset Storage

- Assets/


### Hot Reloading

AssetLoader will check the last modified time of the file on disk and if it has changed, it will reload the asset. While debugging
Nebula will load assets from within the project directory, otherwise it will load from the bin directory.

### TODO:

- Asset Dependencies (e.g., Material -> Shader, Texture)


## Scene Format & Prefabs

This section describes the in development scene format (and prefab format)

### Prefab

```json
{
  "version": 1,
  "prefab": {
    "name": "PlayerWithGun",
    "entities": [
      {
        "EntityName": "Player",
        // GlobalTransform is added via "required components"
        "Transform3D": {
          "Translation": [0, 1, 0],
          "Rotation": [0, 0, 0, 1],
          "Scale": [1, 1, 1]
        },
        // ViewVisibility and InheritedVisibility are added via "required components"
        "Visibility": "Inherited",
        // GpuInstanceData, GpuInstanceDirty, GpuInstanceId are added via "required components"
        "Renderable": {  
            // Link up assets / resources
            "Mesh": "$meshes.capsule",
            "Material": "$materials.player"
        }
      },
      {
        "EntityName": "PlayerGun",
        "Transform3D": {
          "Translation": [0.9, 0, 0.2],
          "Rotation": [0, 0.707, 0, 0.707],
          "Scale": [0.01, 0.01, 0.01]
        },
        "Visibility": "Inherited",
        "Renderable": {  
            "Mesh": "$meshes.m1911",
            "Material": "$materials.gun"
        },
        "ChildOf": { "parent": "@Player" }
      }
    ]
  },
  // All resources or 'assets' are defined here, the asset group name is registered in the engine
  "resources": {
    // Mesh
    "meshes": {
      // This mesh is defined inline, it'll be created dynamically based on the properties
      "capsule": { 
        "type": "inline", 
        // Properties is a custom object defined for each inline asset
        "properties": { 
          // A simple example, set the shape, but we could also pass extra params in here like size
          "shape": "capsule"
        } 
      },
      
      // Resources can be loaded directly from a file using the resource loader
      // We may also specify custom options
      "m1911": { "type": "file", "path": "Assets/M1911.obj", "options": {} }
    },
    // StandardMaterial
    "materials": {
      // Resources can be defined inline without loading from a file
      "gun": { 
        "type": "inline", 
        // These properties are specific to the material
        "properties": { 
          "color": [1, 0, 0, 0] 
        } 
      }
    }
  }
}

```

### Scene

```json
{
  "version": 1,
  "scene": {
    "entities": [
      {
        "EntityName": "Player",
        // GlobalTransform is added via "required components"
        "Transform3D": {
          "Translation": [0, 1, 0],
          "Rotation": [0, 0, 0, 1],
          "Scale": [1, 1, 1]
        },
        // ViewVisibility and InheritedVisibility are added via "required components"
        "Visibility": "Inherited",
        // GpuInstanceData, GpuInstanceDirty, GpuInstanceId are added via "required components"
        "Renderable": {  
            "Mesh": "$meshes.capsule",
            "Material": "$materials.player"
        }
      },
      {
        "EntityName": "PlayerGun",
        "Transform3D": {
          "Translation": [0.9, 0, 0.2],
          "Rotation": [0, 0.707, 0, 0.707],
          "Scale": [0.01, 0.01, 0.01]
        },
        "Visibility": "Inherited",
        "Renderable": {  
            "Mesh": "$meshes.m1911",
            "Material": "$materials.gun"
        },
        "ChildOf": { "parent": "@Player" }
      }
    ]
  },
  "resources": {
    "meshes": {
      "capsule": { "type": "procedural", "shape": "capsule" },
      "m1911": { "type": "file", "path": "Assets/M1911.obj" }
    },
    "materials": { 
        "gun": {  "type": "inline",  "properties": { "color": [1, 0, 0, 0] } } 
    }
  }
}

```

## Global Buffers

These core buffers are used throughout Nebula. A subset of these are passed to the graphics shader via a root constant.

### Meshes

Provides information necessary for the global vertices buffer to render a mesh (`VertexStride`, and `VertexByteOffset`). Also contains bounding information for compute based culling (`BoundingSphereCenter` and `BoundingSphereRadius`).

Points towards one or more mesh lods for actual index data  for rendering the mesh as various levels of detail (via `LodCount` and `LodBaseIndex`).

### MeshLODs

Provides information necessary for interpretation of the global index buffer (`IndexElementOffset` and `IndexElementCount`).

Also contains `ScreenSizeThreshold` used to determine when the LOD should be active.

### Vertices

Global byte buffer containing all vertices of varying strides. 

### Indices

Global `uint` buffer containing all indices.

### Instances

Every instance in the scene will have an item in this buffer. This determines where a mesh should be rendered alongside the material, optional skinning transforms, flags and the render pipeline.

The render pipeline can override the pbr material if needed (in that case, just set it to 0).

### Visible Instances

Used for compute based gpu culling.

### Materials

PBR based material properties. 

### Bone Matrices

Bone transforms for skinning.

### Indirect Arguments

### Indirect Count

### DrawIds