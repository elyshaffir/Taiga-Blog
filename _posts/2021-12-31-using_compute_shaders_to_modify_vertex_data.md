---
layout: post
title:	"Using Compute Shaders to Modify Vertex Data"
wgpu-version: 0.11
---
# Prerequisites
- A functioning render pipeline.
- `wgpu::Features::VERTEX_WRITABLE_STORAGE` should be enabled.

The plan is to create a compute pipeline that modifies a storage buffer which is read back in the vertex shader.

# The Compute Shader
```wgsl
[[block]]
struct ComputeData {
	data: array<vec4<f32>>;
};

[[group(0), binding(0)]]
var<storage, read_write> compute_data: ComputeData;

[[stage(compute), workgroup_size(4, 1, 1)]]
fn cp_main([[builtin(local_invocation_id)]] param: vec3<u32>) {
	compute_data.data[param.x] = vec4<f32>(1.0 / f32(param.x + 1u), 1.0 / f32(param.x + 1u), 1.0 / f32(param.x + 1u), 1.0);
}

[[stage(vertex)]]
fn vs_main(..., [[builtin(vertex_index)]] vertex_index: u32) -> ... {
	...
	out.color = compute_data.data[vertex_index];
	return out;
}

[[stage(fragment)]]
fn fs_main ...
```

In this specific case the compute shader doesn't do too much, and the entire pipeline supports 4 vertices since `cp_main` runs 4 times (as indicated by the [`workgroup_size` attribute](https://www.w3.org/TR/WGSL/#attribute-workgroup_size)), each time filling a [different](https://www.w3.org/TR/WGSL/#local-invocation-id) color vector in the `compute_data` buffer.

|❗ Notice the type of `data` |

[Originally](https://www.reddit.com/r/rust_gamedev/comments/rozcgr/wgsl_alignment_workarounds/), `data` was an array of `vec3<f32>`s which meant that because array elements (and probably other things) need to be 16 byte alligned - an extra float would magically appear at the end of each element in the array as padding (with 0 value).

|❗ Notice the `[[block]]` attribute |

_As of WGPU 0.12 `[[block]]` was removed from the WGSL spec._

The attribute _"indicates this structure type represents the contents of a buffer resource occupying a single binding slot in the shader’s resource interface"_.

Not using it will yield the following meaningful error message:
```console
Global variable [1] 'compute_data' is invalid
Type flags DATA | COPY | INTERFACE | HOST_SHARED do not meet the required DATA | HOST_SHARED | TOP_LEVEL
```

# The Compute Pipeline
```rust
let compute_bind_group_layout =
	device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
		label: Some("Compute Bind Group Layout"),
			entries: &[wgpu::BindGroupLayoutEntry {
				binding: 0,
				visibility: wgpu::ShaderStages::COMPUTE | wgpu::ShaderStages::VERTEX,
				ty: wgpu::BindingType::Buffer {
					ty: wgpu::BufferBindingType::Storage { read_only: false },
					has_dynamic_offset: false,
					min_binding_size: None,
				},
				count: None,
			}],
		});

let compute_buffer = device.create_buffer(&wgpu::BufferDescriptor {
	label: Some("Compute Buffer"),
	size: 2048,
	usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::VERTEX,
	mapped_at_creation: false,
});

let compute_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
	label: Some("Compute Bind Group"),
	layout: &compute_bind_group_layout,
	entries: &[wgpu::BindGroupEntry {
		binding: 0,
		resource: wgpu::BindingResource::Buffer(wgpu::BufferBinding {
			buffer: &compute_buffer,
			offset: 0,
			size: wgpu::BufferSize::new(2048),
		}),
	}],
});

let compute_pipeline_layout =
	device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
		label: Some("Compute Pipeline Layout"),
		bind_group_layouts: &[&compute_bind_group_layout],
		push_constant_ranges: &[],
	});

let compute_pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
	label: Some("Compute Pipeline"),
	layout: Some(&compute_pipeline_layout),
	module: &shader,
	entry_point: "cp_main",
});
```

|❗ Notice the of `visibility` and `usage` fields |

# The Render Pipeline
In your render `BindGroupLayout` (if you don't already have one, creating an _empty_ one is pretty straight forward) you shoud add an `entry` for the newly created `compute_buffer`, since the vertex shader will be reading from it.

```rust
wgpu::BindGroupLayoutEntry {
	binding: 0,
	visibility: wgpu::ShaderStages::COMPUTE | wgpu::ShaderStages::VERTEX,
	ty: wgpu::BindingType::Buffer {
		ty: wgpu::BufferBindingType::Storage { read_only: false },
		has_dynamic_offset: false,
		min_binding_size: None,
	},
	count: None,
}
```

Even though the vertex shader will only be reading from the `compute_buffer`, since in the shader it is already defined as `read_write` we still need to specify `read_only: false`.

In your render `BindGroup` (again if you didn't already have one, creating an _empty_ one is pretty straight forward) you should add an `entry` for the newly created `compute_buffer`:
```rust
wgpu::BindGroupEntry {
	binding: 0,
	resource: wgpu::BindingResource::Buffer(wgpu::BufferBinding {
		buffer: &compute_buffer,
		offset: 0,
		size: wgpu::BufferSize::new(2048),
	}),
}
```

That's it, good luck and keep on the fasten!
