# How Rust implements multithreading

- **Upstream commit:** `4f8ea98e`
- **Area:** `library/std/src/thread`, `library/std/src/sync`, `library/core/src/marker.rs`
- **Date:** 2026-06-15

## Question

How does `std` turn `thread::spawn(|| ...)` into a real OS thread, and what in the type system (`Send`/`Sync`) makes sharing data across threads safe?

## Short answer

`std::thread::spawn` wraps `Builder::spawn`, which boxes the closure and hands it to a per-platform `imp::Thread::new` that calls `libc::pthread_create` on Unix (or `CreateThread` on Windows). The returned `JoinHandle` owns the OS handle; `join()` blocks via `pthread_join`. Safety is enforced **at the type level**: `spawn` requires `Send + 'static`, where `Send`/`Sync` are compiler-derived **auto traits** in `library/core/src/marker.rs`. `Rc` opts out with `impl !Send`; thread-safe sharing uses `Arc<Mutex<T>>` whose `Mutex` wraps an OS/futex primitive via `sys::sync`.

## 1. From `spawn` to an OS thread

`thread::spawn` (`library/std/src/thread/functions.rs:125`) is just `Builder::new().spawn(f).expect(...)` with `F: FnOnce() -> T, F: Send + 'static, T: Send + 'static`. `Builder::spawn` (`thread/builder.rs:185`) forwards to the `unsafe` `spawn_unchecked` (`builder.rs:254`) → the free `spawn_unchecked` in `thread/lifecycle.rs:17`, which:
- Allocates an `Arc<Packet<T>>` to carry the result back (`lifecycle.rs:56`).
- Runs the closure inside `panic::catch_unwind` so a panic doesn't unwind across the FFI boundary (`lifecycle.rs:68`).
- Calls `imp::Thread::new(stack_size, init)` (`lifecycle.rs:116`).

`Packet` is manually `unsafe impl Sync` (`lifecycle.rs:169`) — the soundness argument (only spawner/joiner touch it) is beyond compiler inference.

**Unix backend** (`library/std/src/sys/thread/unix.rs:43`): `Thread::new` builds a `pthread_attr_t`, sets the stack size, then `libc::pthread_create(&mut native, attr, thread_start, data)` (`unix.rs:96`). `thread_start` (`:107`) reconstructs the box and runs the closure; `Thread::join` calls `libc::pthread_join` (`:124`). All `libc::pthread_*` come from the `libc` crate (see [[24-libc-dependency]]). **Windows** mirrors this with `CreateThread` (`sys/thread/windows.rs:31`). `JoinHandle<T>` (`thread/join_handle.rs:71`) owns the handle; `join` (`:149`) blocks and returns `T` or the panic payload; dropping it *detaches* (`:9`). TLS is `thread_local!` / `LocalKey` (`thread/local.rs:11`).

## 2. Send / Sync

In `library/core/src/marker.rs`:

```rust
pub unsafe auto trait Send { }   // marker.rs:92
pub unsafe auto trait Sync { }   // marker.rs:657
```

- **Auto traits**: auto-implemented when every field is `Send`/`Sync` (`marker.rs:71`); you rarely write `impl` yourself.
- **`unsafe`**: hand impls need `unsafe impl` (a soundness claim).
- **Meaning**: `Send` = movable to another thread; `Sync` = `&T` shareable, i.e. `T: Sync ⟺ &T: Send` (`marker.rs:537`).
- **Negative impls**: raw pointers are `!Send`/`!Sync` (`marker.rs:97`); `unsafe impl<T: Sync> Send for &T` ties them together (`:105`).

**Why `spawn` is safe**: the `Send + 'static` bounds mean the closure (and captures) can legally move to the new thread with no borrowed data; the borrow checker rejects unsound captures at compile time. **`!Send` example**: `Rc` has `impl !Send for Rc` (`library/alloc/src/rc.rs:330`) because its refcount is non-atomic — capturing an `Rc` in a `spawn` closure fails to compile, steering you to `Arc` (atomic, `unsafe impl Send/Sync for Arc where T: Send + Sync`, `sync.rs:279`).

## 3. Synchronization primitives (`library/std/src/sync`)

Re-exported from `sync/mod.rs`: `Arc`/`Weak` (atomic refcount), `Mutex`/`RwLock` (poison-aware guards; non-poison variant under `sync::nonpoison`), `Condvar`, `mpsc` channels, `Once`/`OnceLock`/`LazyLock`/`Barrier`. Atomics come straight from `core` (`pub use core::sync::atomic`, `mod.rs:177`) — no OS support needed.

**Mutex wraps an OS/futex primitive**: `Mutex<T>` (`sync/poison/mutex.rs:227`) holds `inner: sys::Mutex`; the backend (`sys/sync/mutex/mod.rs`) is a Linux/futex impl or a `pthread_mutex_t` wrapper. The public `Mutex` adds poisoning + `UnsafeCell<T>` and is `unsafe impl Sync where T: Send` (`mutex.rs:257`) — `Sync` needs only `T: Send` because the lock serializes access.

## 4. Example

```rust
use std::sync::{Arc, Mutex};
use std::thread;
let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];
for _ in 0..10 {
    let counter = Arc::clone(&counter);       // bump the atomic refcount
    handles.push(thread::spawn(move || {      // `move` transfers the Arc clone (Send)
        *counter.lock().unwrap() += 1;        // OS/futex lock via sys::sync
    }));
}
for h in handles { h.join().unwrap(); }        // pthread_join under the hood
assert_eq!(*counter.lock().unwrap(), 10);
```

Swapping `Arc` for `Rc` fails to compile thanks to `impl !Send for Rc`.

## 5. Compiler-side parallelism (cross-ref)

The above is std-level. Separately, **rustc itself** has a parallel frontend on its own work-stealing pool, `compiler/rustc_thread_pool` (vendored Rayon) — a different concurrency layer; see [[10-performance-and-robustness]].

## See also

- [[24-libc-dependency]] · [[10-performance-and-robustness]] · [[40-smart-pointers]] · [[07-glossary]]
