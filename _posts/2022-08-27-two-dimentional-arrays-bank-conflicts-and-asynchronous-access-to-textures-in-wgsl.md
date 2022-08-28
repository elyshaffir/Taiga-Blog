---
layout: post
title: "Two Dimentional Arrays, Bank Conflicts and Asynchronous Access to Textures in WGSL"
wgpu-version: 0.13
---
## Background
Taiga is a [falling sand game](https://en.wikipedia.org/wiki/Falling-sand_game) built around a cellular automata.

Since the logic of this cellular automata runs on the GPU (the code for each cell runs in a separate thread for performance), it makes sense to store the world data as a _grid of cells_ on the GPU.

This approach, while correct, raised all sorts of problems discussed (and finally solved) in this post.

## Problem #1: Bank Conflicts
[Bank conflicts](https://stackoverflow.com/a/3841968/5102607) (in a nutshell) cause serial execution of code.

I started by storing the _grid of cells_ (which from now on will be called **the world**) in a storage buffer:
```rust
// world.wgsl
struct World {
	materials: array<array<u32, WORLD_SIZE_X>, WORLD_SIZE_Y>; // each element is an ID of a material
};

@group(0) @binding(0) var<storage, read_write> world: World;
```
The logic of sand materials is fairly simple (pseudo code):
```python
if cell_below is empty:
	move_to_cell_below()
```
In "small" worlds (in this example 50 by 50), it worked just fine:

![50 by 50 works nicely]({{ site.baseurl }}/assets/images/50x50.gif)

As you can see, the sand materials (yello squares) fall nicely to the ground.

But in "big" worlds (in this example 500 by 500), the sand seems to fall instantly (or teleport) to the ground:

![500 by 500 sand falls instantly]({{ site.baseurl }}/assets/images/500x500 teleporting.gif)

What causes this behaviour is that threads that handle cells on the same column execute serially from top to bottom, causing a teleportation effect.

The reason the threads execute in that manner is of course bank conflicts.

Since the world is big and requires a big buffer, much of the data is stored in the same banks (and multiple accesses to the same bank are executed serially).

## Solution #1 and Problem #2: Textures and Race Conditions
For some reason, changing the world buffer to be a 2-dimentional texture solved the problem for big worlds:

```rust
@group(0) @binding(0) var world: texture_storage_2d<r32uint, read_write>;
```

![500 by 500 works nicely]({{ site.baseurl }}/assets/images/500x500.gif)

When thinking about creating new materials and especially materials that can move faster, a new problem appeared: what happens if two materials move to (or through) the same cell _at the same time_?

My solution for such parallelism problems (also called race conditions) is [atomic types](https://www.w3.org/TR/WGSL/#atomic-types).

Ideally, it would be possible to replace my `texture_storage_2d<r32uint, read_write>` with some sort of `texture_storage_2d<atomic<r32uint>, read_write>` that has the same guarantees as atomic types but in the form of a texture.

Unfortunately as of writing this post there is no such type in WGSL (GLSL has [atomic operations on some texture types](https://www.khronos.org/opengl/wiki/Image_Load_Store#Atomic_operations), as discussed in a question I posted on both [Reddit](https://www.reddit.com/r/rust_gamedev/comments/wwllee/wgpu_atomic_texture_operations/) and [Stack Overflow](https://stackoverflow.com/questions/73475506/rust-wgpu-atomic-texture-operations) while researching the topic).

## The Solution: Implementing Locks Using Atomic Types
Instead of writing to the texture in the game logic using the standard `textureStore` I wrote a wrapper around it that makes sure writing to the texture in a certain cell happens only in one thread at a time.

I achieved that guarantee using a simple locking system implemented with atomic types:
```rust
// locks.wgsl
struct Locks {
	locks: array<array<atomic<u32>, WORLD_SIZE_X>, WORLD_SIZE_Y>,
};

@group(0) @binding(1) var<storage, read_write> locks: Locks;

fn lock(location: vec2<u32>) -> bool {
    let lock_ptr = &locks.locks[location.y][location.x];
    let original_lock_value = atomicLoad(lock_ptr);
    if (original_lock_value > 0u) {
        return false;
    }
    return atomicAdd(lock_ptr, 1u) == original_lock_value;
}

fn unlock(location: vec2<u32>) {
    atomicStore(&locks.locks[location.y][location.x], 0u);
}
```
Ideally, I'd use `atomicCompareExchangeWeak` instead of that somewhat complex logic in `lock`, but `atomicCompareExchangeWeak` didn't seem to work on my machine so I created similar logic myself.

And so, while reading from the texture should be possible at any time, writing to the texture in some `location: vec2<u32>` should be done only if `lock(location)` returned `true` (don't forget to call `unlock` when finished writing).

Unlocking the locks between shader calls (in addition to calling `unlock` when finished writing) is also a good idea.

### Side Note
Using bank conflicts to your advantage is also possible, for example while writing this post I noticed that even just accessing the lock (calling nothing but `atomicLoad`) changes the behaviour of the program dramatically.

In big worlds, accessing the locks causes bank conflicts in the same way the original layout of the world caused them.

This makes sense, since access to the locks is serial it creates some sort of "queue" of threads lining up to access the same lock, which can be used to decrease the chances of a race condition dramatically (but probably not completely).

It's an interesting technique that might be useful some day, maybe even as an optimization to my implementation of an atomic texture.

That's it, good luck and keep on the fasten!
