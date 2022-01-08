---
layout: post
title:	"Using Include Statements in WGSL"
wgpu-version: 0.12
---
# The Problem
As of writing this post, [WGSL has no include functionality](https://www.reddit.com/r/rust_gamedev/comments/rxnufk/why_does_wgsl_support_include_functionality/).

By default, a single `.wgsl` file will always be mapped to a single `wgpu::ShaderModule`, which makes writing a long compute / graphics pipeline either cumbersome (when using a single file) or repetetive (like in the case of [sharing structs between shader stage](https://elyshaffir.github.io/Taiga-Blog/2021/12/31/using_compute_shaders_to_modify_vertex_data.html)).

[According to the community](https://www.reddit.com/r/rust_gamedev/comments/rxnufk/why_does_wgsl_support_include_functionality/) the developer is expected to write their own include functionality, and this post is about how to do such a thing.

# The Solution

|❗ This solution is a modified version of [kvark's solution](https://github.com/kvark/vange-rs/blob/master/src/render/mod.rs) |

```rust
// wgsl_utils.rs
use std::{fs, io, io::Read, path};

pub fn load_shader(name: &str) -> Result<wgpu::ShaderModuleDescriptor, io::Error> {
	let shader_code = load_shader_module(name)?;

	Ok(wgpu::ShaderModuleDescriptor {
		label: Some(name),
		source: wgpu::ShaderSource::Wgsl(shader_code.into()),
	})
}

fn load_shader_module(name: &str) -> Result<String, io::Error> {
	let base_path = path::PathBuf::from("src");
	let module_path = base_path.join(name).with_extension("wgsl");
	if !module_path.is_file() {
		panic!("Shader not found: {:?}", module_path);
	}

	let mut module_source = String::new();
	io::BufReader::new(fs::File::open(&module_path)?).read_to_string(&mut module_source)?;
	let mut module_string = String::new();

	let first_line = module_source.lines().next().unwrap();
	if first_line.starts_with("//!include") {
		for include in first_line.split_whitespace().skip(1) {
			module_string.push_str(&*load_shader_module(include).unwrap());
		}
	}

	module_string.push_str(&module_source);
	Ok(module_string)
}
```
```rust
// main.rs
mod wgsl_utils;

let shader = device.create_shader_module(&wgsl_utils::load_shader("shader").unwrap());
```

|❗ `load_shader` receives the name of a shader without the `.wgsl` postfix |

|❗ Shaders are searched under the `src` folder |


This code allows you to use an `//!include` comment in the beginning of the shader file (only works in the first line) like so:
```wgsl
//!include to_include to_include2
...
```
which would take the contents of `src/to_include.wgsl` and `src/to_include2.wgsl` and insert them in the beginning of the shader file (in that order) recursively (`src/to_include.wgsl` could theoretically include other shader files).

|❗ `//!include` is a comment so it doesn't interfere with regular compilation |


That's it, good luck and keep on the fasten!
