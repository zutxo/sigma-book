# Chapter 30: Cross-Platform Support

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- Elliptic curve cryptography ([Chapter 9](../part4/ch09-elliptic-curve-cryptography.md))
- SDK architecture ([Chapter 27](../part9/ch27-high-level-sdk.md))
- Build systems and compilation

## Learning Objectives

- Understand cross-compilation architecture
- Implement platform abstraction layers
- Master conditional compilation for targets
- Work with WASM and native targets

## Cross-Compilation Architecture

Zig provides native cross-compilation to any target from any host[^1][^2]:

```
Cross-Compilation Targets
══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    Host Build System                             │
│                                                                  │
│   zig build -Dtarget=<target>                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Native      │  │ WASM        │  │ Embedded    │
│             │  │             │  │             │
│ x86_64-linux│  │ wasm32-wasi │  │ arm-none    │
│ aarch64-macos│ │ wasm32-freestanding│ │ riscv32│
│ x86_64-windows│└─────────────┘  └─────────────┘
└─────────────┘

Common Targets:
  x86_64-linux-gnu       Linux desktop/server
  aarch64-linux-gnu      ARM64 Linux (Raspberry Pi, etc.)
  x86_64-macos           macOS Intel
  aarch64-macos          macOS Apple Silicon
  x86_64-windows-gnu     Windows
  wasm32-wasi            WebAssembly with WASI
  wasm32-freestanding    WebAssembly browser
```

## Platform Abstraction

Platform-specific code via conditional compilation:

```zig
const builtin = @import("builtin");
const std = @import("std");

const Platform = struct {
    pub const target = builtin.target;
    pub const os = target.os.tag;
    pub const arch = target.cpu.arch;

    pub const is_wasm = arch == .wasm32 or arch == .wasm64;
    pub const is_native = !is_wasm;
    pub const is_windows = os == .windows;
    pub const is_linux = os == .linux;
    pub const is_macos = os == .macos;

    /// Get platform-appropriate crypto implementation
    pub fn getCrypto() type {
        if (is_wasm) {
            return WasmCrypto;
        } else {
            return NativeCrypto;
        }
    }

    /// Get platform-appropriate allocator
    pub fn getDefaultAllocator() std.mem.Allocator {
        if (is_wasm) {
            return std.heap.wasm_allocator;
        } else {
            return std.heap.c_allocator;
        }
    }
};
```

## Crypto Abstraction Layer

Platform-agnostic cryptography interface[^3][^4]:

```zig
const CryptoFacade = struct {
    const Impl = Platform.getCrypto();

    pub const SECRET_KEY_LENGTH: usize = 32;
    pub const PUBLIC_KEY_LENGTH: usize = 33;
    pub const SIGNATURE_LENGTH: usize = 64;

    /// Create new crypto context
    pub fn createContext() CryptoContext {
        return Impl.createContext();
    }

    /// Normalize point to affine coordinates
    pub fn normalizePoint(p: Ecp) Ecp {
        return Impl.normalizePoint(p);
    }

    /// Negate point (y-coordinate)
    pub fn negatePoint(p: Ecp) Ecp {
        return Impl.negatePoint(p);
    }

    /// Check if point is infinity
    pub fn isInfinityPoint(p: Ecp) bool {
        return Impl.isInfinityPoint(p);
    }

    /// Point exponentiation: p^n
    pub fn exponentiatePoint(p: Ecp, n: *const Scalar) Ecp {
        return Impl.exponentiatePoint(p, n);
    }

    /// Point multiplication (addition in EC group): p1 + p2
    pub fn multiplyPoints(p1: Ecp, p2: Ecp) Ecp {
        return Impl.multiplyPoints(p1, p2);
    }

    /// Encode point (compressed or uncompressed)
    pub fn encodePoint(p: Ecp, compressed: bool) [PUBLIC_KEY_LENGTH]u8 {
        return Impl.encodePoint(p, compressed);
    }

    /// HMAC-SHA512
    pub fn hashHmacSha512(key: []const u8, data: []const u8) [64]u8 {
        return Impl.hashHmacSha512(key, data);
    }

    /// PBKDF2-HMAC-SHA512
    pub fn generatePbkdf2Key(
        password: []const u8,
        salt: []const u8,
        iterations: u32,
    ) [64]u8 {
        return Impl.generatePbkdf2Key(password, salt, iterations);
    }

    /// Secure random bytes
    pub fn randomBytes(dest: []u8) void {
        Impl.randomBytes(dest);
    }
};
```

## Native Crypto Implementation

Using Zig's standard library and optional C bindings[^5][^6]:

```zig
const NativeCrypto = struct {
    const std = @import("std");
    const crypto = std.crypto;

    pub fn createContext() CryptoContext {
        return CryptoContext.secp256k1();
    }

    pub fn normalizePoint(p: Ecp) Ecp {
        return p.normalize();
    }

    pub fn negatePoint(p: Ecp) Ecp {
        return p.negate();
    }

    pub fn isInfinityPoint(p: Ecp) bool {
        return p.isIdentity();
    }

    pub fn exponentiatePoint(p: Ecp, n: *const Scalar) Ecp {
        return p.mul(n.*);
    }

    pub fn multiplyPoints(p1: Ecp, p2: Ecp) Ecp {
        return p1.add(p2);
    }

    pub fn encodePoint(p: Ecp, compressed: bool) [33]u8 {
        if (compressed) {
            return p.toCompressedSec1();
        } else {
            var buf: [65]u8 = undefined;
            return p.toUncompressedSec1(&buf)[0..33].*;
        }
    }

    pub fn hashHmacSha512(key: []const u8, data: []const u8) [64]u8 {
        var hmac = crypto.auth.HmacSha512.init(key);
        hmac.update(data);
        return hmac.finalResult();
    }

    pub fn generatePbkdf2Key(
        password: []const u8,
        salt: []const u8,
        iterations: u32,
    ) [64]u8 {
        var result: [64]u8 = undefined;
        crypto.pwhash.pbkdf2(
            &result,
            password,
            salt,
            iterations,
            crypto.auth.HmacSha512,
        );
        return result;
    }

    pub fn randomBytes(dest: []u8) void {
        crypto.random.bytes(dest);
    }
};
```

## WASM Crypto Implementation

WebAssembly-specific implementation using imports:

```zig
const WasmCrypto = struct {
    // External functions imported from JavaScript host
    extern "env" fn crypto_random_bytes(ptr: [*]u8, len: usize) void;
    extern "env" fn crypto_hmac_sha512(
        key_ptr: [*]const u8,
        key_len: usize,
        data_ptr: [*]const u8,
        data_len: usize,
        out_ptr: [*]u8,
    ) void;
    extern "env" fn crypto_secp256k1_mul(
        point_ptr: [*]const u8,
        scalar_ptr: [*]const u8,
        out_ptr: [*]u8,
    ) void;

    pub fn createContext() CryptoContext {
        return CryptoContext.secp256k1();
    }

    pub fn randomBytes(dest: []u8) void {
        crypto_random_bytes(dest.ptr, dest.len);
    }

    pub fn hashHmacSha512(key: []const u8, data: []const u8) [64]u8 {
        var result: [64]u8 = undefined;
        crypto_hmac_sha512(
            key.ptr,
            key.len,
            data.ptr,
            data.len,
            &result,
        );
        return result;
    }

    pub fn exponentiatePoint(p: Ecp, n: *const Scalar) Ecp {
        var result: [33]u8 = undefined;
        crypto_secp256k1_mul(
            &p.toCompressedSec1(),
            &n.toBytes(),
            &result,
        );
        return Ecp.fromCompressedSec1(result) catch unreachable;
    }

    // ... other operations using WASM imports
};
```

## WASM JavaScript Host

JavaScript glue code for browser/Node.js:

```javascript
// wasm_host.js - JavaScript host for WASM crypto
const crypto = require('crypto');
const secp256k1 = require('secp256k1');

const imports = {
    env: {
        crypto_random_bytes: (ptr, len) => {
            const bytes = crypto.randomBytes(len);
            const mem = new Uint8Array(wasmMemory.buffer, ptr, len);
            mem.set(bytes);
        },

        crypto_hmac_sha512: (keyPtr, keyLen, dataPtr, dataLen, outPtr) => {
            const key = new Uint8Array(wasmMemory.buffer, keyPtr, keyLen);
            const data = new Uint8Array(wasmMemory.buffer, dataPtr, dataLen);
            const hmac = crypto.createHmac('sha512', key);
            hmac.update(data);
            const result = hmac.digest();
            const out = new Uint8Array(wasmMemory.buffer, outPtr, 64);
            out.set(result);
        },

        crypto_secp256k1_mul: (pointPtr, scalarPtr, outPtr) => {
            const point = new Uint8Array(wasmMemory.buffer, pointPtr, 33);
            const scalar = new Uint8Array(wasmMemory.buffer, scalarPtr, 32);
            const result = secp256k1.publicKeyTweakMul(point, scalar, true);
            const out = new Uint8Array(wasmMemory.buffer, outPtr, 33);
            out.set(result);
        }
    }
};
```

## Conditional Compilation

Target-specific code paths:

```zig
const builtin = @import("builtin");

pub fn getTimestamp() i64 {
    if (builtin.target.os.tag == .wasi) {
        // WASI clock_time_get
        var ts: std.os.wasi.timestamp_t = undefined;
        _ = std.os.wasi.clock_time_get(.REALTIME, 1, &ts);
        return @intCast(ts / 1_000_000_000);
    } else if (builtin.target.cpu.arch == .wasm32) {
        // Freestanding WASM - use imported function
        return wasmGetTimestamp();
    } else {
        // Native - use std
        return std.time.timestamp();
    }
}

pub fn allocate(comptime T: type, n: usize) ![]T {
    const allocator = if (Platform.is_wasm)
        std.heap.wasm_allocator
    else if (builtin.link_libc)
        std.heap.c_allocator
    else
        std.heap.page_allocator;

    return allocator.alloc(T, n);
}
```

## Build Configuration

build.zig for multi-target builds:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Native target (default)
    const native_target = b.standardTargetOptions(.{});
    const native_optimize = b.standardOptimizeOption(.{});

    const lib = b.addStaticLibrary(.{
        .name = "ergotree",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = native_target,
        .optimize = native_optimize,
    });

    // WASM target
    const wasm_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const wasm_lib = b.addStaticLibrary(.{
        .name = "ergotree",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = wasm_target,
        .optimize = .ReleaseSmall,
    });

    // Export for JavaScript
    wasm_lib.rdynamic = true;
    wasm_lib.export_memory = true;

    // WASI target (for Node.js)
    const wasi_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .wasi,
    });

    const wasi_lib = b.addExecutable(.{
        .name = "ergotree",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = wasi_target,
        .optimize = .ReleaseSmall,
    });

    // Install all targets
    b.installArtifact(lib);
    b.installArtifact(wasm_lib);
    b.installArtifact(wasi_lib);
}
```

## Memory Management

Platform-specific allocation strategies:

```zig
const Allocator = std.mem.Allocator;

const MemoryConfig = struct {
    /// Maximum memory for WASM (64KB pages)
    wasm_max_pages: u32 = 256, // 16MB

    /// Use arena for batch operations
    use_arena: bool = true,

    /// Pre-allocate constant pool
    constant_pool_size: usize = 4096,
};

pub fn createPlatformAllocator(config: MemoryConfig) Allocator {
    if (Platform.is_wasm) {
        // WASM uses linear memory with explicit growth
        return std.heap.WasmAllocator.init(.{
            .max_memory = config.wasm_max_pages * 65536,
        });
    } else {
        // Native uses page allocator with arena wrapper
        if (config.use_arena) {
            const backing = std.heap.page_allocator;
            var arena = std.heap.ArenaAllocator.init(backing);
            return arena.allocator();
        }
        return std.heap.page_allocator;
    }
}
```

## Type Representation

Consistent types across platforms:

```zig
/// Platform-independent big integer
pub const BigInt = struct {
    limbs: []u64,
    positive: bool,

    pub fn fromBytes(bytes: []const u8) BigInt {
        // Works on all platforms
        var limbs = std.ArrayList(u64).init(allocator);
        // ... conversion logic
        return .{ .limbs = limbs.items, .positive = true };
    }

    pub fn toBytes(self: *const BigInt, buf: []u8) []u8 {
        // Consistent byte representation
        // ... conversion logic
        return buf[0..written];
    }
};

/// Platform-independent scalar (256-bit)
pub const Scalar = struct {
    bytes: [32]u8,

    pub fn fromBigInt(n: *const BigInt) Scalar {
        var result: Scalar = undefined;
        _ = n.toBytes(&result.bytes);
        return result;
    }
};
```

## Endianness Handling

Consistent byte order across architectures:

```zig
pub fn readU32BE(bytes: []const u8) u32 {
    return std.mem.readInt(u32, bytes[0..4], .big);
}

pub fn writeU32BE(value: u32, buf: []u8) void {
    std.mem.writeInt(u32, buf[0..4], value, .big);
}

pub fn readU64LE(bytes: []const u8) u64 {
    return std.mem.readInt(u64, bytes[0..8], .little);
}

// Serialization always uses network byte order (big-endian)
pub fn serializeInt(value: anytype, writer: anytype) !void {
    const T = @TypeOf(value);
    var buf: [@sizeOf(T)]u8 = undefined;
    std.mem.writeInt(T, &buf, value, .big);
    try writer.writeAll(&buf);
}
```

## Performance Considerations

```
Platform Performance Characteristics
══════════════════════════════════════════════════════════════════

┌─────────────────┬───────────────────────────────────────────────┐
│ Platform        │ Characteristics                               │
├─────────────────┼───────────────────────────────────────────────┤
│ Native (x86_64) │ ✓ SIMD acceleration (AVX2/AVX512)            │
│                 │ ✓ Hardware AES-NI                             │
│                 │ ✓ Large memory, fast allocation              │
│                 │ ✓ Multi-threaded execution                   │
├─────────────────┼───────────────────────────────────────────────┤
│ Native (ARM64)  │ ✓ NEON SIMD                                   │
│                 │ ✓ Hardware crypto extensions                  │
│                 │ ✓ Power-efficient                             │
├─────────────────┼───────────────────────────────────────────────┤
│ WASM (browser)  │ ○ Single-threaded (mostly)                    │
│                 │ ○ Linear memory model                         │
│                 │ ✓ JIT compilation by browser                 │
│                 │ ○ No direct filesystem/network               │
├─────────────────┼───────────────────────────────────────────────┤
│ WASI (Node.js)  │ ○ Single-threaded                             │
│                 │ ✓ WASI syscalls for I/O                      │
│                 │ ✓ Sandboxed execution                        │
└─────────────────┴───────────────────────────────────────────────┘

Optimization Strategies:
  Native:   Use comptime for specialization, SIMD intrinsics
  WASM:     Minimize memory allocations, batch operations
  Both:     Profile-guided optimization, cache-friendly layouts
```

## Testing Cross-Platform

```zig
const testing = std.testing;

test "crypto operations consistent across platforms" {
    const key = "test_key";
    const data = "test_data";

    const result = CryptoFacade.hashHmacSha512(key, data);

    // Expected value computed externally
    const expected = [_]u8{
        0x8f, 0x9d, 0x1c, // ... full 64 bytes
    };

    try testing.expectEqualSlices(u8, &expected, &result);
}

test "point operations" {
    const ctx = CryptoFacade.createContext();
    const g = ctx.generator;

    // g + g = 2g
    const two_g_add = CryptoFacade.multiplyPoints(g, g);
    const scalar_2 = Scalar.fromInt(2);
    const two_g_mul = CryptoFacade.exponentiatePoint(g, &scalar_2);

    try testing.expect(two_g_add.eql(two_g_mul));
}
```

## Usage Example

Cross-platform wallet library:

```zig
const Wallet = struct {
    prover: Prover,
    allocator: Allocator,

    pub fn init(seed: []const u8) !Wallet {
        const allocator = Platform.getDefaultAllocator();

        // Platform-independent key derivation
        const master_key = CryptoFacade.generatePbkdf2Key(
            seed,
            "mnemonic",
            2048,
        );

        return .{
            .prover = try Prover.fromSeed(master_key, allocator),
            .allocator = allocator,
        };
    }

    pub fn signTransaction(
        self: *const Wallet,
        tx: *const Transaction,
    ) !SignedTransaction {
        // Works identically on all platforms
        return self.prover.sign(tx);
    }
};

// Same code runs on all targets:
// - Desktop app (native)
// - Browser extension (WASM)
// - Mobile wallet (ARM native or WASM)
// - Server-side validation (native)
```

## Summary

- **Zig cross-compiles** to any target from any host without external tools
- **Platform abstraction** uses `builtin.target` for conditional compilation
- **CryptoFacade** provides consistent API across native and WASM
- **WASM targets** use JavaScript imports for platform-specific crypto
- **Memory management** adapts to platform constraints
- **Type representation** ensures consistent behavior across architectures
- **Testing** verifies identical results on all platforms

---

*Next: [Chapter 31: Performance Engineering](./ch31-performance-engineering.md)*

[^1]: Scala: `core/shared/src/main/scala/sigma/crypto/CryptoFacade.scala` (abstraction)

[^2]: Rust: Platform-independent design in sigma-rust crate structure

[^3]: Scala: `core/jvm/src/main/scala/sigma/crypto/Platform.scala` (JVM impl)

[^4]: Rust: `ergotree-interpreter/src/sigma_protocol/` (crypto operations)

[^5]: Scala: `core/js/src/main/scala/sigma/crypto/Platform.scala` (JS impl)

[^6]: Rust: Feature flags in Cargo.toml for optional dependencies