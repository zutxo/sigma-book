# Chapter 29: Soft-Fork Mechanism

## Prerequisites

- ErgoTree structure ([Chapter 3](../part1/ch03-ergotree-structure.md))
- Serialization ([Chapter 7](../part2/ch07-serialization.md))
- Transaction validation ([Chapter 24](../part8/ch24-transaction-validation.md))

## Learning Objectives

- Understand version context and script versioning
- Implement validation rules with configurable status
- Master unknown opcode handling for soft-forks
- Work with the AOT to JIT interpreter transition

## Version Context Architecture

The soft-fork mechanism enables protocol upgrades without breaking consensus[^1][^2]:

```
Soft-Fork Version Architecture
══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    Block Header                                  │
│                                                                  │
│   Block Version: 1, 2, 3, 4                                     │
│                                                                  │
│   Activated Script Version = Block Version - 1                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ErgoTree Header                               │
│                                                                  │
│   7   6   5   4   3   2   1   0                                 │
│   ├───┼───┼───┼───┼───┼───┼───┤                                 │
│   │ M │ G │ C │ S │ Z │ V │ V │ V                               │
│   └───┴───┴───┴───┴───┴───┴───┘                                 │
│   M = More bytes follow                                         │
│   G = GZIP (reserved)                                           │
│   C = Context costing (reserved)                                │
│   S = Constant segregation                                      │
│   Z = Size included                                             │
│   V = Version (0-7)                                             │
└─────────────────────────────────────────────────────────────────┘
```

## ErgoTree Version

Script version is encoded in header bits 0-2[^3][^4]:

```zig
const ErgoTreeVersion = struct {
    value: u3, // 0-7

    const VERSION_MASK: u8 = 0x07;

    /// Version 0 - Initial mainnet (v3.x)
    pub const V0 = ErgoTreeVersion{ .value = 0 };
    /// Version 1 - Height monotonicity (v4.x)
    pub const V1 = ErgoTreeVersion{ .value = 1 };
    /// Version 2 - JIT interpreter (v5.x)
    pub const V2 = ErgoTreeVersion{ .value = 2 };
    /// Version 3 - Sub-blocks, new ops (v6.x)
    pub const V3 = ErgoTreeVersion{ .value = 3 };

    /// Maximum supported script version
    pub const MAX_SCRIPT_VERSION = V3;

    /// Parse version from header byte
    pub fn parseVersion(header_byte: u8) ErgoTreeVersion {
        return .{ .value = @truncate(header_byte & VERSION_MASK) };
    }

    pub fn toU8(self: ErgoTreeVersion) u8 {
        return @as(u8, self.value);
    }
};
```

## ErgoTree Header

Header byte encoding with flags[^5][^6]:

```zig
const ErgoTreeHeader = struct {
    version: ErgoTreeVersion,
    is_constant_segregation: bool,
    has_size: bool,

    const CONSTANT_SEGREGATION_FLAG: u8 = 0b0001_0000;
    const HAS_SIZE_FLAG: u8 = 0b0000_1000;

    /// Parse header from byte
    pub fn parse(header_byte: u8) !ErgoTreeHeader {
        return .{
            .version = ErgoTreeVersion.parseVersion(header_byte),
            .is_constant_segregation = (header_byte & CONSTANT_SEGREGATION_FLAG) != 0,
            .has_size = (header_byte & HAS_SIZE_FLAG) != 0,
        };
    }

    /// Serialize header to byte
    pub fn serialize(self: *const ErgoTreeHeader) u8 {
        var header_byte: u8 = self.version.toU8();
        if (self.is_constant_segregation) {
            header_byte |= CONSTANT_SEGREGATION_FLAG;
        }
        if (self.has_size) {
            header_byte |= HAS_SIZE_FLAG;
        }
        return header_byte;
    }

    /// Create v0 header
    pub fn v0(constant_segregation: bool) ErgoTreeHeader {
        return .{
            .version = ErgoTreeVersion.V0,
            .is_constant_segregation = constant_segregation,
            .has_size = false,
        };
    }

    /// Create v1 header (size is mandatory)
    pub fn v1(constant_segregation: bool) ErgoTreeHeader {
        return .{
            .version = ErgoTreeVersion.V1,
            .is_constant_segregation = constant_segregation,
            .has_size = true,
        };
    }
};
```

## Version Context

Thread-local context tracks activated and tree versions[^7][^8]:

```zig
const VersionContext = struct {
    activated_version: u8,
    ergo_tree_version: u8,

    /// JIT costing activation version (v5.0)
    const JIT_ACTIVATION_VERSION: u8 = 2;
    /// v6.0 soft-fork version
    const V6_SOFT_FORK_VERSION: u8 = 3;

    pub fn init(activated: u8, tree: u8) !VersionContext {
        // ergoTreeVersion must never exceed activatedVersion
        if (activated >= JIT_ACTIVATION_VERSION and tree > activated) {
            return error.InvalidVersionContext;
        }
        return .{
            .activated_version = activated,
            .ergo_tree_version = tree,
        };
    }

    /// True if JIT costing is activated (v5.0+)
    pub fn isJitActivated(self: *const VersionContext) bool {
        return self.activated_version >= JIT_ACTIVATION_VERSION;
    }

    /// True if v6.0 protocol is activated
    pub fn isV6Activated(self: *const VersionContext) bool {
        return self.activated_version >= V6_SOFT_FORK_VERSION;
    }

    /// True if v3+ ErgoTree version
    pub fn isV3OrLaterErgoTree(self: *const VersionContext) bool {
        return self.ergo_tree_version >= V6_SOFT_FORK_VERSION;
    }
};

/// Thread-local version context
threadlocal var current_context: ?VersionContext = null;

pub fn withVersions(
    activated: u8,
    tree: u8,
    comptime block: fn (*VersionContext) anyerror!void,
) !void {
    const ctx = try VersionContext.init(activated, tree);
    const prev = current_context;
    current_context = ctx;
    defer current_context = prev;
    try block(&ctx);
}

pub fn currentContext() !*const VersionContext {
    return &(current_context orelse return error.VersionContextNotSet);
}
```

## Version History

```
Protocol Version History
══════════════════════════════════════════════════════════════════

┌─────────────┬────────────────┬──────────────┬────────────────────┐
│ Block Ver   │ Script Ver     │ Protocol     │ Features           │
├─────────────┼────────────────┼──────────────┼────────────────────┤
│ 1           │ 0              │ v3.x         │ Initial mainnet    │
│ 2           │ 1              │ v4.x         │ Height monotonicity│
│ 3           │ 2              │ v5.x         │ JIT interpreter    │
│ 4           │ 3              │ v6.x         │ Sub-blocks, new ops│
└─────────────┴────────────────┴──────────────┴────────────────────┘

Relation: activated_script_version = block_version - 1
```

## Rule Status

Validation rules have configurable status[^9][^10]:

```zig
const RuleStatus = union(enum) {
    /// Default: rule is active and enforced
    enabled,
    /// Rule is disabled (via voting)
    disabled,
    /// Rule replaced by new rule
    replaced: struct { new_rule_id: u16 },
    /// Rule parameters changed
    changed: struct { new_value: []const u8 },

    const StatusCode = enum(u8) {
        enabled = 1,
        disabled = 2,
        replaced = 3,
        changed = 4,
    };

    pub fn statusCode(self: RuleStatus) StatusCode {
        return switch (self) {
            .enabled => .enabled,
            .disabled => .disabled,
            .replaced => .replaced,
            .changed => .changed,
        };
    }
};
```

## Validation Rules

Rules define validation behavior with soft-fork support[^11][^12]:

```zig
const ValidationRule = struct {
    id: u16,
    description: []const u8,
    soft_fork_checker: SoftForkChecker,
    checked: bool = false,

    pub fn checkRule(self: *ValidationRule, settings: *const ValidationSettings) !void {
        if (!self.checked) {
            if (settings.getStatus(self.id) == null) {
                return error.ValidationRuleNotFound;
            }
            self.checked = true;
        }
    }

    pub fn throwValidationException(
        self: *const ValidationRule,
        cause: anyerror,
        args: []const u8,
    ) ValidationError {
        return ValidationError{
            .rule = self,
            .args = args,
            .cause = cause,
        };
    }
};

const ValidationError = struct {
    rule: *const ValidationRule,
    args: []const u8,
    cause: anyerror,
};
```

## Core Validation Rules

```zig
const ValidationRules = struct {
    const FIRST_RULE_ID: u16 = 1000;

    /// Check primitive type code is valid
    pub const CheckPrimitiveTypeCode = ValidationRule{
        .id = 1007,
        .description = "Check primitive type code is supported or added via soft-fork",
        .soft_fork_checker = .code_added,
    };

    /// Check non-primitive type code is valid
    pub const CheckTypeCode = ValidationRule{
        .id = 1008,
        .description = "Check non-primitive type code is supported or added via soft-fork",
        .soft_fork_checker = .code_added,
    };

    /// Check data can be serialized for type
    pub const CheckSerializableTypeCode = ValidationRule{
        .id = 1009,
        .description = "Check data values of type can be serialized",
        .soft_fork_checker = .when_replaced,
    };

    /// Check reader position limit
    pub const CheckPositionLimit = ValidationRule{
        .id = 1014,
        .description = "Check Reader position limit",
        .soft_fork_checker = .when_replaced,
    };
};
```

## Soft-Fork Checkers

Detect soft-fork conditions from validation failures[^13][^14]:

```zig
const SoftForkChecker = enum {
    none,
    when_replaced,
    code_added,

    pub fn isSoftFork(
        self: SoftForkChecker,
        settings: *const ValidationSettings,
        rule_id: u16,
        status: RuleStatus,
        args: []const u8,
    ) bool {
        return switch (self) {
            .none => false,
            .when_replaced => switch (status) {
                .replaced => true,
                else => false,
            },
            .code_added => switch (status) {
                .changed => |c| std.mem.indexOf(u8, c.new_value, args) != null,
                else => false,
            },
        };
    }
};
```

## Validation Settings

Configurable settings from blockchain state[^15][^16]:

```zig
const ValidationSettings = struct {
    rules: std.AutoHashMap(u16, struct { rule: *ValidationRule, status: RuleStatus }),

    pub fn getStatus(self: *const ValidationSettings, id: u16) ?RuleStatus {
        if (self.rules.get(id)) |entry| {
            return entry.status;
        }
        return null;
    }

    pub fn updated(self: *const ValidationSettings, id: u16, new_status: RuleStatus) !ValidationSettings {
        var new_rules = try self.rules.clone();
        if (new_rules.getPtr(id)) |entry| {
            entry.status = new_status;
        }
        return .{ .rules = new_rules };
    }

    /// Check if exception represents a soft-fork condition
    pub fn isSoftFork(self: *const ValidationSettings, ve: ValidationError) bool {
        const entry = self.rules.get(ve.rule.id) orelse return false;

        // Don't tolerate replaced v5.0 rules after v6.0 activation
        switch (entry.status) {
            .replaced => {
                const ctx = currentContext() catch return false;
                if (ctx.isV6Activated() and
                    (ve.rule.id == 1011 or ve.rule.id == 1007 or ve.rule.id == 1008))
                {
                    return false;
                }
                return true;
            },
            else => return entry.rule.soft_fork_checker.isSoftFork(
                self,
                ve.rule.id,
                entry.status,
                ve.args,
            ),
        }
    }
};
```

## Soft-Fork Execution Wrapper

Execute code with soft-fork fallback:

```zig
pub fn trySoftForkable(
    comptime T: type,
    settings: *const ValidationSettings,
    when_soft_fork: T,
    block: fn () anyerror!T,
) T {
    return block() catch |err| {
        if (@errorCast(ValidationError, err)) |ve| {
            if (settings.isSoftFork(ve)) {
                return when_soft_fork;
            }
        }
        return err;
    };
}

// Usage: handling unknown opcodes
fn deserializeValue(
    reader: *Reader,
    settings: *const ValidationSettings,
) !Value {
    return trySoftForkable(
        Value,
        settings,
        // Soft-fork fallback: return unit placeholder
        Value.unit,
        // Normal deserialization
        struct {
            fn f() !Value {
                const op_code = try reader.readByte();
                const serializer = getSerializer(op_code) orelse
                    return error.UnknownOpCode;
                return serializer.parse(reader);
            }
        }.f,
    );
}
```

## AOT to JIT Transition

```
Script Validation Rules Across Versions
══════════════════════════════════════════════════════════════════

Rule │ Block │ Block Type │ Script │ v4.0 Action     │ v5.0 Action
─────┼───────┼────────────┼────────┼─────────────────┼─────────────
  1  │ 1,2   │ candidate  │ v0/v1  │ AOT-cost,verify │ AOT-cost,verify
  2  │ 1,2   │ mined      │ v0/v1  │ AOT-cost,verify │ AOT-cost,verify
  3  │ 1,2   │ candidate  │ v2     │ skip-pool-tx    │ skip-pool-tx
  4  │ 1,2   │ mined      │ v2     │ skip-reject     │ skip-reject
─────┼───────┼────────────┼────────┼─────────────────┼─────────────
  5  │ 3     │ candidate  │ v0/v1  │ skip-pool-tx    │ JIT-verify
  6  │ 3     │ mined      │ v0/v1  │ skip-accept     │ JIT-verify
  7  │ 3     │ candidate  │ v2     │ skip-pool-tx    │ JIT-verify
  8  │ 3     │ mined      │ v2     │ skip-accept     │ JIT-verify

Actions:
  AOT-cost,verify  Cost estimation + verification using v4.0 AOT
  JIT-verify       Verification using v5.0 JIT interpreter
  skip-pool-tx     Skip mempool transaction (can't handle)
  skip-accept      Accept block without verification (trust majority)
  skip-reject      Reject transaction/block (invalid version)
```

## Consensus Equivalence Properties

For safe transition between interpreter versions:

```zig
// Property 1: AOT-verify preserved between releases
// forall s:ScriptV0/V1, R4.0-AOT-verify(s) == R5.0-AOT-verify(s)

// Property 2: AOT-cost preserved
// forall s:ScriptV0/V1, R4.0-AOT-cost(s) == R5.0-AOT-cost(s)

// Property 3: JIT can replace AOT
// forall s:ScriptV0/V1, R5.0-JIT-verify(s) == R4.0-AOT-verify(s)

// Property 4: JIT cost bounded by AOT
// forall s:ScriptV0/V1, R5.0-JIT-cost(s) <= R4.0-AOT-cost(s)

// Property 5: ScriptV2 rejected before soft-fork
// forall s:ScriptV2, if not SF active => reject
```

## Version-Aware Interpreter

```zig
pub fn verify(
    ergo_tree: *const ErgoTree,
    ctx: *const Context,
) !bool {
    const script_version = ergo_tree.header.version;
    const activated_version = ctx.activatedScriptVersion();

    // Execute with proper version context
    var version_ctx = try VersionContext.init(
        activated_version.toU8(),
        script_version.toU8(),
    );

    const prev = current_context;
    current_context = version_ctx;
    defer current_context = prev;

    // Version-specific execution
    if (version_ctx.isJitActivated()) {
        return verifyJit(ergo_tree, ctx);
    } else {
        return verifyAot(ergo_tree, ctx);
    }
}

fn verifyJit(tree: *const ErgoTree, ctx: *const Context) !bool {
    const reduced = try fullReduction(tree, ctx);
    return verifySignature(reduced, ctx.messageToSign());
}

fn verifyAot(tree: *const ErgoTree, ctx: *const Context) !bool {
    // Legacy AOT interpreter path
    const result = try aotEvaluate(tree, ctx);
    return verifySignature(result, ctx.messageToSign());
}
```

## Block Extension Voting

Rule status changes via blockchain extension voting:

```
Extension Voting Flow
══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    Block Extension Section                       │
│                                                                  │
│  Key (2 bytes)    │    Value                                    │
│  ─────────────────┼──────────────────────────────────────────── │
│  Rule ID          │    RuleStatus + data                        │
│  0x03EF (1007)    │    ChangedRule([0x5A, 0x5B])               │
│                   │    (new opcodes 0x5A, 0x5B allowed)        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Voting Epoch                                  │
│                                                                  │
│  Epoch 1:    □ □ □ ■ □ ■ ■ □ ■ ■   (5/10 = 50%)               │
│  Epoch 2:    ■ ■ □ ■ ■ ■ □ ■ ■ ■   (8/10 = 80%)               │
│  Epoch 3:    ■ ■ ■ ■ ■ □ ■ ■ ■ ■   (9/10 = 90%) → ACTIVATED   │
└─────────────────────────────────────────────────────────────────┘

New Opcode Addition:
  1. Before soft-fork: Unknown opcode → ValidationException
  2. Extension update: ChangedRule(Array(newOpcode)) for rule 1001
  3. After activation: Old nodes recognize soft-fork via SoftForkWhenCodeAdded
  4. Result: Old nodes skip verification; new nodes execute new opcode
```

## Unknown Opcode Handling

```zig
fn handleUnknownOpcode(
    reader: *Reader,
    settings: *const ValidationSettings,
    op_code: u8,
) !Expr {
    // Check if this is a soft-fork condition
    const rule = &ValidationRules.CheckTypeCode;
    const status = settings.getStatus(rule.id) orelse return error.RuleNotFound;

    switch (status) {
        .changed => |c| {
            // Check if opcode was added via soft-fork
            if (std.mem.indexOfScalar(u8, c.new_value, op_code) != null) {
                // Soft-fork: skip remaining bytes, return placeholder
                reader.skipToEnd();
                return Expr{ .constant = Constant.unit };
            }
        },
        else => {},
    }

    // Not a soft-fork condition - fail hard
    return rule.throwValidationException(error.UnknownOpCode, &[_]u8{op_code});
}
```

## Summary

- **ErgoTreeVersion** encodes script version in 3-bit header field (0-7)
- **VersionContext** tracks activated protocol and tree versions
- **RuleStatus** can be Enabled, Disabled, Replaced, or Changed
- **SoftForkChecker** detects soft-fork conditions from validation failures
- **trySoftForkable** provides graceful fallback for unknown constructs
- **AOT→JIT transition** demonstrated soft-fork for major interpreter change
- **Block extension voting** enables parameter changes via miner consensus
- **Old nodes** remain compatible by trusting majority on unverifiable blocks

---

*Next: [Chapter 30: Cross-Platform Support](./ch30-cross-platform-support.md)*

[^1]: Scala: `core/shared/src/main/scala/sigma/VersionContext.scala:17-35`

[^2]: Rust: `ergotree-ir/src/chain/context.rs:46-53`

[^3]: Scala: `core/shared/src/main/scala/sigma/ast/ErgoTree.scala` (header)

[^4]: Rust: `ergotree-ir/src/ergo_tree/tree_header.rs:122-145`

[^5]: Scala: `core/shared/src/main/scala/sigma/ast/ErgoTree.scala:57-84`

[^6]: Rust: `ergotree-ir/src/ergo_tree/tree_header.rs:27-109`

[^7]: Scala: `core/shared/src/main/scala/sigma/VersionContext.scala:47-56`

[^8]: Rust: `ergotree-ir/src/chain/context.rs:12-54`

[^9]: Scala: `core/shared/src/main/scala/sigma/validation/RuleStatus.scala:4-53`

[^10]: Rust: Not directly present in sigma-rust; validation handled at higher level

[^11]: Scala: `core/shared/src/main/scala/sigma/validation/ValidationRules.scala:13-51`

[^12]: Rust: Validation rules embedded in deserializer implementations

[^13]: Scala: `core/shared/src/main/scala/sigma/validation/SoftForkChecker.scala:4-42`

[^14]: Rust: Soft-fork handling at application level (ergo-lib)

[^15]: Scala: `core/shared/src/main/scala/sigma/validation/SigmaValidationSettings.scala:45-69`

[^16]: Rust: `ergo-lib/src/chain/parameters.rs` (blockchain parameters)
