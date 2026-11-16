# QUICHE: Simulated-Time Support

This repository contains a modified QUICHE crate that introduces support for externally controlled simulated time. It is intended for integration with discrete-event network simulators such as ns-3.

## 1. Problem Statement

QUICHE uses Rust's `Instant::now()` to obtain timestamps for internal operations, including packet processing, recovery, and congestion control.

In a discrete-event simulator such as ns-3, time is controlled by the simulator rather than the operating system. QUICHE's internal clock must therefore be synchronized with the simulator's notion of time.

This implementation introduces a configurable clock abstraction that allows QUICHE to operate using externally supplied simulated time.

## 2. Implementation

The implementation introduces the following changes:

### New clock module

A new file, `src/clock.rs`, implements the clock abstraction with three main functions:

- `now()` – Returns the current time according to the selected clock mode.
- `set_simulated_time_us(time_us)` – Enables simulated-clock mode and updates the simulated time in microseconds.
- `use_real_clock()` – Switches back to the normal system-clock mode.

### Module registration

The new module is registered in `src/lib.rs`, alongside the existing module declarations:

```rust
mod clock;
```

The module does not need to be declared `pub` because it is accessed internally within the same Rust crate.

### C FFI interface

The simulated-time function is exposed through QUICHE's C FFI in `src/ffi.rs`:

```rust
#[no_mangle]
pub extern "C" fn quiche_set_simulated_time_us(time_us: u64) {
    crate::clock::set_simulated_time_us(time_us);
}
```

Its declaration is added to `include/quiche.h`:

```c
void quiche_set_simulated_time_us(uint64_t time_us);
```

This allows external applications to update QUICHE's internal clock through its C interface.

### Replacing direct system-clock access

Existing production-code calls to `Instant::now()` are replaced with `crate::clock::now()` so that internal QUICHE operations use the configurable clock.

For the source revision used in this implementation, **19 replacements across 8 Rust files** were made.

To identify direct system-clock accesses in another QUICHE revision, use:

```bash
grep -rn "Instant::now()" . --include="*.rs"
```

Inspect the results before applying replacements, since line numbers and source locations may differ between revisions.

## 3. Using This Repository

This repository contains the modified `quiche/` crate, not the complete Cloudflare QUICHE workspace.

To integrate it:

1. Clone the original Cloudflare QUICHE repository.
2. Clone this repository separately.
3. Back up or rename the original `quiche/` directory.
4. Place the contents of this repository into the original workspace's `quiche/` directory.

The original workspace provides the Cargo configuration and supporting dependencies required to build the crate.

**If you use this repository, the simulated-time changes are already integrated.** You do not need to repeat the implementation steps.

Original upstream repository: https://github.com/cloudflare/quiche

## 4. Build

From the original QUICHE workspace root:

```bash
cd quiche
cargo build --release --features ffi
```

The build produces the QUICHE libraries in the workspace's `target/release/` directory.

Compiler warnings may appear; the important check is that the build completes successfully.

## 5. Verify the Exported C Function

From the `quiche/` directory, run:

```bash
nm -D ../target/release/libquiche.so | grep simulated_time
```

The output should contain the symbol:

```text
T quiche_set_simulated_time_us
```

The `T` indicates that the function is defined in the library's text section. Its presence confirms that the simulated-time function is exported in the shared library.

## 6. Build Outputs

| File | Description |
|---|---|
| `libquiche.a` | Static library |
| `libquiche.so` | Shared (dynamic) library |

The shared library can be inspected using `nm -D` to verify dynamically exported symbols.

## 7. Scope

This repository covers the QUICHE-side simulated-clock implementation only.

The ns-3 wrapper, simulator event scheduling, and network topology configuration are maintained separately.