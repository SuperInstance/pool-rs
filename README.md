# pool-rs

## Description
`pool-rs` is a Rust library that provides fast, safe, and ergonomic pool management utilities. It is a component of the **Cocapn Fleet** (github.com/SuperInstance).

## Usage
Add the crate to your project:

```toml
[dependencies]
pool-rs = "0.1.0"
```

Import and use it in Rust code:

```rust
use pool_rs::Pool;

fn main() {
    let mut pool = Pool::new();
    // pool operations here
}
```

## Related
- **Cocapn Fleet** – the larger project this library belongs to: https://github.com/SuperInstance  
- Documentation & design notes: [DOCKSIDE-EXAM.md](DOCKSIDE-EXAM.md)  
- Research materials: see the `research/` directory  

## License
Distributed under the terms of the [LICENSE](LICENSE) file.