---
slug: /reference/development/wasm-sdk
title: Wasm SDK contract
---

# WebAssembly SDK contract

This document defines the Application Binary Interface (ABI) and host capabilities that you need to build a custom `fluxrig` WebAssembly (Wasm) Gear. fluxrig relies on a language-agnostic Wasm execution environment ([Wazero](https://wazero.io/)). You can author your business logic in any language that compiles to `wasm32-freestanding` or `wasm32-wasi`. Examples are Zig, Rust, C++, Go, and AssemblyScript. Adhere to this contract.

## Filter architecture & implicit ports

> [!NOTE]
> **Current topology role**: The project currently implements the Wasm gear strictly as a **Filter** gear (1-to-1 or 1-to-0). It receives exactly one input message, processes it synchronously, and returns exactly one output message (or drops it).

Because of this strict filter pattern, the Wasm gear uses **implicit ports**. You do not need to define a `ports` block in its configuration.
- **Input (`.in`)**: The router automatically routes incoming messages to the gear's `.in` port, which triggers the Wasm `process()` function.
- **Output (`.out`)**: The gear automatically emits the result that `process()` returns on the `.out` port.

*(The project plans support for Source/Split Wasm gears that generate spontaneous messages via `env.emit` for a future release).*

## The memory interface

WebAssembly restricts execution to an isolated linear memory sandbox. To pass data between the `fluxrig` host (the Rack) and the Wasm guest (your Gear), you must export basic memory allocation functions.

### Required exports

Your Wasm module **must** export the following three functions:

#### `alloc`
```text
alloc(len: i32) -> i32
```
The host calls it to allocate memory inside the guest's linear memory.
- **`len`**: The number of bytes the guest allocates.
- **Returns**: It returns an `i32` pointer to the allocated memory block.

#### `free`
```text
free(ptr: i32, len: i32)
```
The host calls it to free memory that the guest previously allocated.
- **`ptr`**: It gives the starting pointer of the memory block.
- **`len`**: It gives the length of the memory block in bytes.

#### `process`
```text
process(ptr: i32, len: i32) -> i64
```
The main execution entrypoint. The host calls it when a new message arrives at the Gear.
- **`ptr`**: A pointer to the CBOR-encoded `fluxMsg` payload that the host already wrote to guest memory.
- **`len`**: It gives the length of the CBOR payload in bytes.
- **Returns**: It returns a packed 64-bit integer (`i64`). The host uses bitwise shifting to unpack this into two 32-bit integers:
  - **High 32 bits**: It gives the return pointer to the modified CBOR payload.
  - **Low 32 bits**: It gives the return length of the modified payload.

If you wish to drop a message, return `0`. 

## Host imports (capabilities)

To interact with the host system (e.g., logging or state access), the host exposes a set of functions under the `env` namespace. You must import these explicitly in your language.

### `env.log`
```text
env.log(level: i32, ptr: i32, len: i32)
```
It emits a structured log message to the Rack's central telemetry stream.
- **`level`**: It gives the severity level (e.g., `1`=Debug, `2`=Info, `3`=Warn, `4`=Error).
- **`ptr`**: It gives the pointer to the UTF-8 encoded log message.
- **`len`**: It gives the length of the log message.

### `env.emit` [Roadmap / Future]
```text
env.emit(ptr: i32, len: i32)
```
*(The project plans it for a future release)*. It allows the Wasm gear to spontaneously generate new messages or split a single message into multiple messages (Fan-out). The Wasm module allocates memory, writes the CBOR payload, and calls `env.emit`. The host reads the memory and pushes the message to the wire asynchronously.

*(The host will also expose future capabilities like `flux_kv_get` or `flux_req_http` under the `env` module, and scenario-level permissions will govern them).*

## Signal format (CBOR)

For performance, the `fluxrig` host does not pass strings or JSON. Input payloads are **CBOR encoded** (RFC 8949) representations of the `fluxMsg` struct.

Your module must:
1. Decode the CBOR payload from the input pointer.
2. Apply business logic to the `fluxMsg` fields (e.g., routing headers, raw payload, metadata).
3. Encode the modified data back into CBOR.
4. Allocate space for the new CBOR data.
5. Return the packed pointer/length.

## Memory lifecycle

1. **Host receives a message** on the bus.
2. **Host calls `alloc(len)`** on the guest to reserve space.
3. **Host writes** the CBOR payload into guest memory at the returned pointer.
4. **Host calls `process(ptr, len)`**.
5. **Guest processes** the message, allocates new memory for the result, writes the new CBOR, and returns the packed `u64`.
6. **Host reads** the new CBOR from the returned pointer.
7. **Host calls `free`** on both the original input pointer and the new output pointer to prevent memory leaks.
