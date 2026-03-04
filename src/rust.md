# Rust and Q: FFI, IPC, and High-Performance Integration

Rust and Q are natural partners in the sense that both communities believe in doing things the right way even when doing it wrong would be faster. They differ in that the Rust compiler will tell you exactly what you did wrong, while Q will silently coerce your type and let you discover the problem three joins later.

There are two ways to connect Rust to kdb+:

1. **IPC over TCP**: Connect to a running kdb+ process, speak the wire protocol. This is the same protocol everything else uses.
2. **FFI via the C API**: Load a shared library into Rust and call kdb+ functions directly. Faster, more complex, requires the C API headers.

For 95% of use cases, IPC is the right choice. For the other 5% — where you're embedding kdb+ in a Rust application, or writing a C extension for kdb+ in Rust — FFI is the path.

## IPC: The `kdbplus` Crate

The `kdbplus` crate provides both IPC client functionality and FFI bindings. Start with IPC.

```toml
# Cargo.toml
[dependencies]
kdbplus = { version = "0.5", features = ["ipc"] }
tokio = { version = "1", features = ["full"] }
```

### Basic Connection

```rust
use kdbplus::ipc::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Connect to a running kdb+ process
    let mut client = QStream::connect(ConnectionMethod::TCP, "localhost", 5001, "").await?;

    // Execute a q expression
    let result = client.send_sync_message(&"til 10").await?;
    println!("{}", result);  // 0 1 2 3 4 5 6 7 8 9

    Ok(())
}
```

The empty string in `connect` is the auth credentials (`"username:password"` if needed). The `send_sync_message` call blocks until the remote q process responds — same as a synchronous IPC call in Q itself.

### Type System

This is where the THAT'S-how-you-do-it moment lives. kdbplus represents Q values with a `K` type that mirrors Q's type hierarchy:

```rust
use kdbplus::ipc::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = QStream::connect(ConnectionMethod::TCP, "localhost", 5001, "").await?;

    // Integer atom -> K with type QLONG
    let x: K = client.send_sync_message(&"42").await?;
    println!("Type: {:?}", x.get_type());   // QLONG
    println!("Value: {}", x.get_long()?);   // 42

    // Float list
    let xs: K = client.send_sync_message(&"1.0 2.5 3.7").await?;
    let floats: Vec<f64> = xs.get_float_list()?;
    println!("{:?}", floats);   // [1.0, 2.5, 3.7]

    // Symbol list
    let syms: K = client.send_sync_message(&"`AAPL`MSFT`GOOG").await?;
    let symbols: Vec<&str> = syms.get_symbol_list()?;
    println!("{:?}", symbols);   // ["AAPL", "MSFT", "GOOG"]

    // Table
    let table: K = client.send_sync_message(&"([] sym:`AAPL`MSFT; price:185.5 375.2)").await?;
    // Tables are dictionaries of lists in Q; iterate columns
    let col_names = table.get_keys()?;
    for name in col_names {
        println!("Column: {}", name);
    }

    Ok(())
}
```

### Querying a Table

The pattern for real data retrieval — execute a Q select, get a table back, process it:

```rust
use kdbplus::ipc::*;
use std::collections::HashMap;

#[derive(Debug)]
struct Trade {
    sym: String,
    price: f64,
    size: i64,
}

async fn get_trades(
    client: &mut QStream,
    date: &str,
    syms: &[&str],
) -> Result<Vec<Trade>, Box<dyn std::error::Error>> {
    let sym_list = syms.iter()
        .map(|s| format!("`{}", s))
        .collect::<Vec<_>>()
        .join("");

    let query = format!(
        "select sym, price, size from trade where date={}, sym in {}",
        date, sym_list
    );

    let result = client.send_sync_message(&query.as_str()).await?;

    // Extract columns from the table
    let sym_col: Vec<&str> = result.get_column("sym")?.get_symbol_list()?;
    let price_col: Vec<f64> = result.get_column("price")?.get_float_list()?;
    let size_col: Vec<i64> = result.get_column("size")?.get_long_list()?;

    let trades = sym_col.into_iter()
        .zip(price_col.into_iter())
        .zip(size_col.into_iter())
        .map(|((sym, price), size)| Trade {
            sym: sym.to_string(),
            price,
            size,
        })
        .collect();

    Ok(trades)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = QStream::connect(ConnectionMethod::TCP, "localhost", 5001, "").await?;

    let trades = get_trades(&mut client, "2024.01.15", &["AAPL", "MSFT"]).await?;

    for trade in &trades {
        println!("{:?}", trade);
    }

    println!("Total trades: {}", trades.len());
    Ok(())
}
```

### Sending Data to Q

Pushing Rust data into a Q process requires constructing `K` objects:

```rust
use kdbplus::ipc::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = QStream::connect(ConnectionMethod::TCP, "localhost", 5001, "").await?;

    // Build a Q table from Rust data
    let syms = K::new_symbol_list(vec!["AAPL".to_string(), "MSFT".to_string()], qattribute::NONE);
    let prices = K::new_float_list(vec![185.5_f64, 375.2_f64], qattribute::NONE);
    let sizes = K::new_long_list(vec![100_i64, 200_i64], qattribute::NONE);

    let table = K::new_table(
        &["sym", "price", "size"],
        &[syms, prices, sizes]
    )?;

    // Set it as a variable in the Q process
    let set_query = K::new_mixed_list(vec![
        K::new_string("set", qattribute::NONE),
        K::new_symbol("`rust_data"),
        table,
    ]);
    client.send_sync_message(&set_query).await?;

    // Verify
    let count = client.send_sync_message(&"count rust_data").await?;
    println!("Rows inserted: {}", count.get_long()?);  // 2

    Ok(())
}
```

### Async Streaming

For continuous data feeds — subscribing to a tickerplant, for example:

```rust
use kdbplus::ipc::*;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = QStream::connect(ConnectionMethod::TCP, "localhost", 5001, "").await?;

    // Subscribe to a tickerplant
    // Standard tick subscriber pattern: .u.sub[`trade; `]
    client.send_sync_message(&".u.sub[`trade; `]").await?;
    println!("Subscribed to trade feed");

    // Listen for incoming messages
    loop {
        // receive_message blocks until a message arrives
        let msg = client.receive_message().await?;

        // Tickerplant sends: (`upd; `trade; table_data)
        if let Ok(msg_list) = msg.get_mixed_list() {
            if msg_list.len() == 3 {
                let func_name = msg_list[0].get_symbol()?;
                let table_name = msg_list[1].get_symbol()?;
                let data = &msg_list[2];

                if func_name == "upd" && table_name == "trade" {
                    let price_col: Vec<f64> = data.get_column("price")?.get_float_list()?;
                    let sym_col: Vec<&str> = data.get_column("sym")?.get_symbol_list()?;

                    for (sym, price) in sym_col.iter().zip(price_col.iter()) {
                        println!("UPDATE: {} @ {:.2}", sym, price);
                    }
                }
            }
        }
    }
}
```

This is the real-time subscriber pattern. The kdb+ tickerplant calls `.u.upd` on all subscribers when new data arrives; the Rust side receives these as incoming messages without polling.

## FFI: The C API

When you need to go deeper — writing kdb+ extensions in Rust, embedding kdb+ in a Rust process, or implementing a custom feed handler — you need the C API.

The `kdbplus` crate includes FFI bindings in its `api` feature:

```toml
[dependencies]
kdbplus = { version = "0.5", features = ["api"] }
```

### Writing a kdb+ Extension in Rust

kdb+ extensions are shared libraries loaded with `2:`. The extension exports C-compatible functions that take and return `K` values.

```rust
// src/lib.rs
use kdbplus::api::*;
use kdbplus::api::native::*;

// Function signature must match kdb+ C API: K func(K x)
#[no_mangle]
pub extern "C" fn rust_add(x: K, y: K) -> K {
    // Extract values
    let a = unsafe { x.get_long() };
    let b = unsafe { y.get_long() };

    // Compute result
    let result = a + b;

    // Return a new K long atom
    unsafe { kj(result) }
}

// More complex: process a list
#[no_mangle]
pub extern "C" fn rust_squares(x: K) -> K {
    unsafe {
        let n = x.get_count();
        let result = ktn(KJ as i32, n as i64);  // allocate long list

        for i in 0..n {
            let val = kJ(x).add(i).read();
            kJ(result).add(i).write(val * val);
        }

        result
    }
}
```

Build as a dynamic library:

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]
```

```bash
cargo build --release
# Output: target/release/libmyextension.dylib (macOS) or .so (Linux)
```

Load from Q:
```q
/ Load the shared library
mylib: `:/path/to/libmyextension 2: (`rust_add; 2)
squares: `:/path/to/libmyextension 2: (`rust_squares; 1)

/ Call the Rust functions
mylib[3; 4]       / 7
squares til 10    / 0 1 4 9 16 25 36 49 64 81
```

### Memory Management in the C API

This is the part where the Rust compiler cannot save you. Q manages memory with reference counting. When a Rust function returns a `K` value to Q, Q owns it. The rules:

- Arguments passed into your function have their reference count incremented by Q before the call. You don't need to increment them again.
- The return value from your function becomes Q's property. Don't free it.
- If you create a `K` value inside your function and it becomes an argument to another function (not the return value), you may need to call `r0` to decrement its reference count.

```rust
#[no_mangle]
pub extern "C" fn rust_filter_positive(x: K) -> K {
    unsafe {
        let n = x.get_count();
        let mut positives: Vec<i64> = Vec::new();

        for i in 0..n {
            let val = kJ(x).add(i).read();
            if val > 0 {
                positives.push(val);
            }
        }

        let result = ktn(KJ as i32, positives.len() as i64);
        for (i, &val) in positives.iter().enumerate() {
            kJ(result).add(i).write(val);
        }

        // x is owned by Q, we don't free it
        // result will be returned to Q, which takes ownership
        result
    }
}
```

Memory bugs here won't show up as Rust compilation errors — they'll show up as kdb+ crashes or memory leaks. Write tests, use valgrind on Linux, and check your reference counts carefully.

## Connection Pooling

For high-throughput Rust services consuming kdb+ data, managing connections efficiently matters:

```rust
use kdbplus::ipc::*;
use std::sync::Arc;
use tokio::sync::Mutex;

struct QConnectionPool {
    connections: Vec<Arc<Mutex<QStream>>>,
    size: usize,
    next: std::sync::atomic::AtomicUsize,
}

impl QConnectionPool {
    async fn new(host: &str, port: u16, auth: &str, size: usize)
        -> Result<Self, Box<dyn std::error::Error>>
    {
        let mut connections = Vec::with_capacity(size);
        for _ in 0..size {
            let conn = QStream::connect(
                ConnectionMethod::TCP, host, port, auth
            ).await?;
            connections.push(Arc::new(Mutex::new(conn)));
        }
        Ok(Self {
            connections,
            size,
            next: std::sync::atomic::AtomicUsize::new(0),
        })
    }

    fn get(&self) -> Arc<Mutex<QStream>> {
        let idx = self.next.fetch_add(1, std::sync::atomic::Ordering::Relaxed) % self.size;
        self.connections[idx].clone()
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = Arc::new(
        QConnectionPool::new("localhost", 5001, "", 4).await?
    );

    // Spawn concurrent queries
    let mut handles = Vec::new();
    for sym in &["AAPL", "MSFT", "GOOG", "AMZN"] {
        let pool = pool.clone();
        let sym = sym.to_string();
        let handle = tokio::spawn(async move {
            let conn = pool.get();
            let mut guard = conn.lock().await;
            let query = format!("last select price from trade where sym=`{}", sym);
            guard.send_sync_message(&query.as_str()).await
        });
        handles.push(handle);
    }

    for handle in handles {
        let result = handle.await??;
        println!("{}", result);
    }

    Ok(())
}
```

## TLS Connections

If your kdb+ process is behind a TLS-terminating proxy or configured with TLS directly:

```rust
use kdbplus::ipc::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // TLS connection
    let mut client = QStream::connect(
        ConnectionMethod::TLS,
        "myserver.example.com",
        5001,
        "user:pass"
    ).await?;

    let result = client.send_sync_message(&"til 5").await?;
    println!("{}", result);

    Ok(())
}
```

## When to Use IPC vs FFI

Use **IPC** when:
- You have a running kdb+ process you want to query
- You're building a service that consumes or publishes kdb+ data
- You want the kdb+ process to remain independently manageable
- You want Rust compilation errors instead of runtime crashes

Use **FFI** when:
- You're writing a kdb+ extension (`.so`/`.dylib` loaded with `2:`)
- You need zero-copy access to Q data structures
- You're embedding a compute kernel that Q will call directly
- You genuinely enjoy reading about reference counting

The IPC path is the more Rust-idiomatic one — clean separation, async-friendly, proper error types. The FFI path gives you more power and more ways to segfault. Start with IPC.
