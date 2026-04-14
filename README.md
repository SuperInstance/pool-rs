# pool-rs
A Rust library for efficient pool management.

## Description
Part of the Cocapn Fleet (github.com/SuperInstance), pool-rs provides a high‑performance pool management system designed for modern applications.

## Usage
Add `pool-rs` to your `Cargo.toml`:
```toml
[dependencies]
pool-rs = "0.1.0"
```
Use the pool in your Rust code:
```rust
use pool_rs::Pool;

fn main() {
    let pool = Pool::new(10);
    // ...
}
```

## Related
- [Cocapn Fleet](https://github.com/SuperInstance)
- [Documentation](DOCKSIDE-EXAM.md)

## License
This project is licensed under the terms of the [LICENSE](LICENSE) file.