# Solution

## Overview
This solution implements a TCP server in Rust that exchanges Protocol Buffers messages with a simple test client. The server:
- Decodes **`ClientMessage`** requests and replies with **`ServerMessage`** responses.
- Supports **multiple clients concurrently** (thread-per-connection).
- Handles **multiple back‑to‑back messages on the same connection** by buffering reads and decoding one message at a time.
- Shuts down gracefully using an atomic `running` flag.

All provided tests pass:
```
running 5 tests
.....
test result: ok. 5 passed; 0 failed; 0 ignored
```

## How to Run
```bash
# Windows (PowerShell)
cargo test -- --test-threads=1
```
> The tests bind to `localhost:8080`. Running with a single test thread avoids port collisions.

## Key Fixes & Rationale
1. **Decode the correct wrapper type**
   - The `.proto` defines a `oneof` inside **`ClientMessage`** and **`ServerMessage`**.
   - The server must decode **`ClientMessage`**, not the generated inner enum `client_message::Message` (which does not implement `prost::Message`).

2. **Handle coalesced TCP reads**
   - TCP does not preserve message boundaries; multiple protobuf messages can arrive in one read.
   - Implemented a small **read buffer** and a `try_decode_one()` helper that decodes **one `ClientMessage`** from the front and returns the **consumed** byte count. Leftover bytes remain for the next decode.

3. **Concurrency model**
   - **Non‑blocking `accept`** loop on the main thread.
   - **Thread‑per‑connection** handlers for simplicity and to satisfy the tests’ multi‑client scenarios.
   - `Arc<AtomicBool>` controls **graceful shutdown**.

4. **I/O and error handling**
   - Treat `0` bytes read as **clean disconnect**.
   - Differentiate between **truncated/incomplete** protobuf vs. **invalid** protobuf and read more bytes when truncated.
   - Use `saturating_add` for `AddRequest` to avoid overflow panics.

## Protocol Notes
- Tests send **raw protobuf** (no length prefix). The server **encodes/decodes raw messages**, not length‑delimited frames.
- To be production‑ready across arbitrary message boundaries, a length‑delimited protocol is recommended. The current implementation achieves robustness by buffering and incremental decode while staying compatible with the test client.

## File(s) You Changed
- **`src/server.rs`**
  - Added buffered decoding loop (`inbuf`).
  - Implemented `try_decode_one()` using `prost::Message::decode` with `Bytes` to compute consumed bytes.
  - Wrote responses using `ServerMessage::encode` to raw bytes.
  - Non‑blocking `accept`, per‑connection handler threads, graceful `stop()`.

## Development Environment
- **Rust** (stable, MSVC toolchain on Windows)
- **Protobuf (`protoc`)** installed and on `PATH`
- Visual C++ Build Tools for the MSVC linker (`link.exe`)

Quick checks:
```powershell
rustc --version
cargo --version
protoc --version
```

## Testing
Run all tests sequentially to avoid `AddrInUse` on port 8080:
```powershell
cargo test -- --test-threads=1
```

Run only ignored tests:
```powershell
cargo test -- --test-threads=1 --ignored
```

## Trade‑offs & Future Improvements
- **Framing**: Switch to `encode_length_delimited`/`decode_length_delimited` on both client and server for explicit message boundaries.
- **Async**: Replace thread‑per‑connection with `tokio` + async I/O for scalability beyond this exercise.
- **Backpressure**: Introduce read limits / timeouts and connection‑level buffering caps.
- **Structured errors**: Map protobuf/IO errors to typed server errors for logging and metrics.
