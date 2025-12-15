# Appendix D: Method Reference

Complete reference for all methods available on each type[^1][^2].

## Method Organization

```
Method System Architecture
══════════════════════════════════════════════════════════════════

                    ┌────────────────────────┐
                    │    STypeCompanion      │
                    │  type_code: TypeCode   │
                    │  methods: []SMethod    │
                    └──────────┬─────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       ▼                       ▼                       ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  SNumeric    │       │    SBox      │       │   SColl      │
│  methods.len │       │  methods.len │       │ methods.len  │
│     = 13     │       │     = 10     │       │    = 20+     │
└──────────────┘       └──────────────┘       └──────────────┘

Method Lookup:
─────────────────────────────────────────────────────────────────
  receiver.methodCall(type_code=99, method_id=1)
       │
       ▼
  STypeCompanion::Box.method_by_id(1)
       │
       ▼
  SMethod { name: "value", tpe: Box => Long, cost: 10 }
```

## Zig Method Descriptors

```zig
const SMethod = struct {
    name: []const u8,
    method_id: u8,
    tpe: SFunc,
    cost_kind: CostKind,
    min_version: ?ErgoTreeVersion = null,

    pub fn isV6Only(self: *const SMethod) bool {
        return self.min_version != null and
            @intFromEnum(self.min_version.?) >= 3;
    }
};

const SFunc = struct {
    t_dom: []const SType,  // Domain (receiver + args)
    t_range: SType,        // Return type

    pub fn unary(recv: SType, ret: SType) SFunc {
        return .{ .t_dom = &[_]SType{recv}, .t_range = ret };
    }

    pub fn binary(recv: SType, arg: SType, ret: SType) SFunc {
        return .{ .t_dom = &[_]SType{ recv, arg }, .t_range = ret };
    }
};
```

## Numeric Types (SByte, SShort, SInt, SLong)[^3]

| ID | Method | Signature | v5 | v6 | Cost |
|----|--------|-----------|----|----|------|
| 1 | toByte | T → Byte | ✓ | ✓ | 10 |
| 2 | toShort | T → Short | ✓ | ✓ | 10 |
| 3 | toInt | T → Int | ✓ | ✓ | 10 |
| 4 | toLong | T → Long | ✓ | ✓ | 10 |
| 5 | toBigInt | T → BigInt | ✓ | ✓ | 30 |
| 6 | toBytes | T → Coll[Byte] | - | ✓ | 5 |
| 7 | toBits | T → Coll[Boolean] | - | ✓ | 5 |
| 8 | bitwiseInverse | T → T | - | ✓ | 5 |
| 9 | bitwiseOr | (T, T) → T | - | ✓ | 5 |
| 10 | bitwiseAnd | (T, T) → T | - | ✓ | 5 |
| 11 | bitwiseXor | (T, T) → T | - | ✓ | 5 |
| 12 | shiftLeft | (T, Int) → T | - | ✓ | 5 |
| 13 | shiftRight | (T, Int) → T | - | ✓ | 5 |

## SBigInt[^4]

| ID | Method | Signature | v5 | v6 | Cost |
|----|--------|-----------|----|----|------|
| 1-5 | toXxx | Conversions | ✓ | ✓ | 10-30 |
| 6-13 | bitwise | Bitwise ops | - | ✓ | 5-10 |
| 14 | toUnsigned | BigInt → UnsignedBigInt | - | ✓ | 5 |
| 15 | toUnsignedMod | (BigInt, UBI) → UBI | - | ✓ | 10 |

## SUnsignedBigInt (v6+)[^5]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 14 | modInverse | (UBI, UBI) → UBI | 50 |
| 15 | plusMod | (UBI, UBI, UBI) → UBI | 10 |
| 16 | subtractMod | (UBI, UBI, UBI) → UBI | 10 |
| 17 | multiplyMod | (UBI, UBI, UBI) → UBI | 15 |
| 18 | mod | (UBI, UBI) → UBI | 10 |
| 19 | toSigned | UBI → BigInt | 5 |

## SGroupElement[^6]

| ID | Method | Signature | v5 | v6 | Cost |
|----|--------|-----------|----|----|------|
| 2 | getEncoded | GE → Coll[Byte] | ✓ | ✓ | 250 |
| 3 | exp | (GE, BigInt) → GE | ✓ | ✓ | 900 |
| 4 | multiply | (GE, GE) → GE | ✓ | ✓ | 40 |
| 5 | negate | GE → GE | ✓ | ✓ | 45 |
| 6 | expUnsigned | (GE, UBI) → GE | - | ✓ | 900 |

## SSigmaProp[^7]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | propBytes | SigmaProp → Coll[Byte] | 35 |
| 2 | isProven | SigmaProp → Boolean | 10 |

## SBox[^8]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | value | Box → Long | 1 |
| 2 | propositionBytes | Box → Coll[Byte] | 10 |
| 3 | bytes | Box → Coll[Byte] | 10 |
| 4 | bytesWithoutRef | Box → Coll[Byte] | 10 |
| 5 | id | Box → Coll[Byte] | 10 |
| 6 | creationInfo | Box → (Int, Coll[Byte]) | 10 |
| 7 | getReg[T] | (Box, Int) → Option[T] | 50 |
| 8 | tokens | Box → Coll[(Coll[Byte], Long)] | 15 |

### Register Access

```zig
const BoxMethods = struct {
    // R0-R3: mandatory registers
    pub const R0 = makeRegMethod(0);  // monetary value
    pub const R1 = makeRegMethod(1);  // guard script
    pub const R2 = makeRegMethod(2);  // tokens
    pub const R3 = makeRegMethod(3);  // creation info
    // R4-R9: optional registers
    pub const R4 = makeRegMethod(4);
    pub const R5 = makeRegMethod(5);
    pub const R6 = makeRegMethod(6);
    pub const R7 = makeRegMethod(7);
    pub const R8 = makeRegMethod(8);
    pub const R9 = makeRegMethod(9);

    fn makeRegMethod(comptime idx: u8) SMethod {
        return .{
            .method_id = 7,  // getReg opcode
            .name = "R" ++ &[_]u8{'0' + idx},
            .cost_kind = .{ .fixed = .{ .cost = .{ .value = 50 } } },
        };
    }
};
```

## SAvlTree[^9]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | digest | AvlTree → Coll[Byte] | 15 |
| 2 | enabledOperations | AvlTree → Byte | 15 |
| 3 | keyLength | AvlTree → Int | 15 |
| 4 | valueLengthOpt | AvlTree → Option[Int] | 15 |
| 5 | isInsertAllowed | AvlTree → Boolean | 15 |
| 6 | isUpdateAllowed | AvlTree → Boolean | 15 |
| 7 | isRemoveAllowed | AvlTree → Boolean | 15 |
| 8 | updateOperations | (AvlTree, Byte) → AvlTree | 20 |
| 9 | contains | (AvlTree, key, proof) → Boolean | dynamic |
| 10 | get | (AvlTree, key, proof) → Option[Coll[Byte]] | dynamic |
| 11 | getMany | (AvlTree, keys, proof) → Coll[Option[...]] | dynamic |
| 12 | insert | (AvlTree, entries, proof) → Option[AvlTree] | dynamic |
| 13 | update | (AvlTree, operations, proof) → Option[AvlTree] | dynamic |
| 14 | remove | (AvlTree, keys, proof) → Option[AvlTree] | dynamic |
| 15 | updateDigest | (AvlTree, Coll[Byte]) → AvlTree | 20 |

## SContext[^10]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | dataInputs | Context → Coll[Box] | 15 |
| 2 | headers | Context → Coll[Header] | 15 |
| 3 | preHeader | Context → PreHeader | 10 |
| 4 | INPUTS | Context → Coll[Box] | 10 |
| 5 | OUTPUTS | Context → Coll[Box] | 10 |
| 6 | HEIGHT | Context → Int | 26 |
| 7 | SELF | Context → Box | 10 |
| 8 | selfBoxIndex | Context → Int | 20 |
| 9 | LastBlockUtxoRootHash | Context → AvlTree | 15 |
| 10 | minerPubKey | Context → Coll[Byte] | 20 |
| 11 | getVar[T] | (Context, Byte) → Option[T] | dynamic |

## SHeader[^11]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | id | Header → Coll[Byte] | 10 |
| 2 | version | Header → Byte | 10 |
| 3 | parentId | Header → Coll[Byte] | 10 |
| 4 | ADProofsRoot | Header → Coll[Byte] | 10 |
| 5 | stateRoot | Header → AvlTree | 10 |
| 6 | transactionsRoot | Header → Coll[Byte] | 10 |
| 7 | timestamp | Header → Long | 10 |
| 8 | nBits | Header → Long | 10 |
| 9 | height | Header → Int | 10 |
| 10 | extensionRoot | Header → Coll[Byte] | 10 |
| 11 | minerPk | Header → GroupElement | 10 |
| 12 | powOnetimePk | Header → GroupElement | 10 |
| 13 | powNonce | Header → Coll[Byte] | 10 |
| 14 | powDistance | Header → BigInt | 10 |
| 15 | votes | Header → Coll[Byte] | 10 |
| 16 | checkPow | Header → Boolean (v6+) | 500 |

## SPreHeader[^12]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | version | PreHeader → Byte | 10 |
| 2 | parentId | PreHeader → Coll[Byte] | 10 |
| 3 | timestamp | PreHeader → Long | 10 |
| 4 | nBits | PreHeader → Long | 10 |
| 5 | height | PreHeader → Int | 10 |
| 6 | minerPk | PreHeader → GroupElement | 10 |
| 7 | votes | PreHeader → Coll[Byte] | 10 |

## SGlobal[^13]

| ID | Method | Signature | v5 | v6 | Cost |
|----|--------|-----------|----|----|------|
| 1 | groupGenerator | Global → GroupElement | ✓ | ✓ | 10 |
| 2 | xor | (Coll[Byte], Coll[Byte]) → Coll[Byte] | ✓ | ✓ | PerItem |
| 3 | serialize[T] | T → Coll[Byte] | - | ✓ | dynamic |
| 4 | fromBigEndianBytes[T] | Coll[Byte] → T | - | ✓ | 10 |
| 5 | encodeNBits | BigInt → Long | - | ✓ | 20 |
| 6 | decodeNBits | Long → BigInt | - | ✓ | 20 |
| 7 | powHit | (Int, ...) → BigInt | - | ✓ | 500 |

## SCollection[^14]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 1 | size | Coll[T] → Int | 14 |
| 2 | apply | (Coll[T], Int) → T | 14 |
| 3 | getOrElse | (Coll[T], Int, T) → T | dynamic |
| 4 | map[R] | (Coll[T], T → R) → Coll[R] | PerItem(20,1,10) |
| 5 | exists | (Coll[T], T → Bool) → Bool | PerItem(20,5,10) |
| 6 | fold[R] | (Coll[T], R, (R,T) → R) → R | PerItem(20,1,10) |
| 7 | forall | (Coll[T], T → Bool) → Bool | PerItem(20,5,10) |
| 8 | slice | (Coll[T], Int, Int) → Coll[T] | PerItem(10,2,100) |
| 9 | filter | (Coll[T], T → Bool) → Coll[T] | PerItem(20,5,10) |
| 10 | append | (Coll[T], Coll[T]) → Coll[T] | PerItem(20,2,100) |
| 14 | indices | Coll[T] → Coll[Int] | PerItem(20,2,128) |
| 15 | flatMap[R] | (Coll[T], T → Coll[R]) → Coll[R] | PerItem(20,5,10) |
| 19 | patch (v6) | (Coll[T], Int, Coll[T], Int) → Coll[T] | dynamic |
| 20 | updated (v6) | (Coll[T], Int, T) → Coll[T] | 20 |
| 21 | updateMany (v6) | (Coll[T], Coll[Int], Coll[T]) → Coll[T] | PerItem |
| 26 | indexOf | (Coll[T], T, Int) → Int | PerItem(20,1,10) |
| 29 | zip[U] | (Coll[T], Coll[U]) → Coll[(T,U)] | PerItem(10,1,10) |
| 30 | reverse (v6) | Coll[T] → Coll[T] | PerItem |
| 31 | startsWith (v6) | (Coll[T], Coll[T]) → Boolean | PerItem |
| 32 | endsWith (v6) | (Coll[T], Coll[T]) → Boolean | PerItem |
| 33 | get (v6) | (Coll[T], Int) → Option[T] | 14 |

## SOption[^15]

| ID | Method | Signature | Cost |
|----|--------|-----------|------|
| 2 | isDefined | Option[T] → Boolean | 10 |
| 3 | get | Option[T] → T | 10 |
| 4 | getOrElse | (Option[T], T) → T | 10 |
| 7 | map[R] | (Option[T], T → R) → Option[R] | dynamic |
| 8 | filter | (Option[T], T → Bool) → Option[T] | dynamic |

## STuple

Tuples support component access by position:

```zig
const TupleMethods = struct {
    /// Access tuple component by index (1-based like Scala)
    pub fn component(comptime idx: usize) SMethod {
        return .{
            .name = "_" ++ std.fmt.comptimePrint("{}", .{idx}),
            .method_id = @intCast(idx),
            .cost_kind = .{ .fixed = .{ .cost = .{ .value = 12 } } },
        };
    }
};

// Usage: tuple._1, tuple._2, ... up to tuple._255
```

---

*[Previous: Appendix C](./appendix-c-costs.md) | [Next: Appendix E](./appendix-e-serialization.md)*

[^1]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala`

[^2]: Rust: `ergotree-ir/src/types/smethod.rs:36-99`

[^3]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala` (SNumericTypeMethods)

[^4]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala` (SBigIntMethods)

[^5]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala` (SUnsignedBigIntMethods)

[^6]: Rust: `ergotree-ir/src/types/sgroup_elem.rs`

[^7]: Scala: `data/shared/src/main/scala/sigma/ast/methods.scala` (SSigmaPropMethods)

[^8]: Rust: `ergotree-ir/src/types/sbox.rs:29-92`

[^9]: Rust: `ergotree-ir/src/types/savltree.rs`

[^10]: Rust: `ergotree-ir/src/types/scontext.rs`

[^11]: Rust: `ergotree-ir/src/types/sheader.rs`

[^12]: Rust: `ergotree-ir/src/types/spreheader.rs`

[^13]: Rust: `ergotree-ir/src/types/sglobal.rs`

[^14]: Rust: `ergotree-ir/src/types/scoll.rs:22-266`

[^15]: Rust: `ergotree-ir/src/types/soption.rs`
