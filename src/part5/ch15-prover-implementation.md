# Chapter 15: Prover Implementation

## Prerequisites

- Sigma protocols ([Chapter 11](../part4/ch11-sigma-protocols.md))
- Evaluation model ([Chapter 12](./ch12-evaluation-model.md))
- Verifier implementation ([Chapter 14](./ch14-verifier-implementation.md))

## Learning Objectives

- Understand the 10-step proving algorithm
- Master the unproven tree data structure and transformations
- Learn challenge flow through composite sigma structures
- Implement the hint system for distributed signing
- Trace proof serialization

## Prover Overview

The prover generates cryptographic proofs for sigma propositions[^1][^2]:

```
Proving Pipeline
─────────────────────────────────────────────────────

Step 0:  SigmaBoolean ─────> convertToUnproven()
                             │
                             ▼
Step 1:  Mark real nodes (bottom-up)
                             │
                             ▼
Step 2:  Check root is real (abort if simulated)
                             │
                             ▼
Step 3:  Polish simulated (top-down)
                             │
                             ▼
Steps 4-6: Simulate/Commit
         - Assign challenges to simulated children
         - Simulate simulated leaves
         - Compute commitments for real leaves
                             │
                             ▼
Step 7:  Serialize for Fiat-Shamir
                             │
                             ▼
Step 8:  Compute root challenge = H(tree || message)
                             │
                             ▼
Step 9:  Compute real challenges and responses
                             │
                             ▼
Step 10: Serialize proof bytes
```

## Tree Data Structures

### Node Position

Position encodes path from root[^3]:

```zig
const NodePosition = struct {
    /// Position bytes (e.g., [0, 2, 1] for "0-2-1")
    positions: []const u8,

    pub const CRYPTO_TREE_PREFIX: NodePosition = .{ .positions = &[_]u8{0} };

    pub fn child(self: NodePosition, idx: usize, allocator: Allocator) !NodePosition {
        var new_pos = try allocator.alloc(u8, self.positions.len + 1);
        @memcpy(new_pos[0..self.positions.len], self.positions);
        new_pos[self.positions.len] = @intCast(idx);
        return .{ .positions = new_pos };
    }
};
```

```
Position Encoding
─────────────────────────────────────────────────────

            0           (root)
          / | \
         /  |  \
       0-0 0-1 0-2      (children)
               /|
              / |
            0-2-0 0-2-1 (grandchildren)

Prefix "0" = crypto-tree (vs "1" = ErgoTree)
```

### Unproven Tree

During proving, the tree undergoes transformations[^4][^5]:

```zig
const UnprovenTree = union(enum) {
    unproven_leaf: UnprovenLeaf,
    unproven_conjecture: UnprovenConjecture,

    pub fn isReal(self: UnprovenTree) bool {
        return !self.simulated();
    }

    pub fn simulated(self: UnprovenTree) bool {
        return switch (self) {
            .unproven_leaf => |l| l.simulated,
            .unproven_conjecture => |c| c.simulated(),
        };
    }

    pub fn withChallenge(self: UnprovenTree, challenge: Challenge) UnprovenTree {
        return switch (self) {
            .unproven_leaf => |l| .{ .unproven_leaf = l.withChallenge(challenge) },
            .unproven_conjecture => |c| .{ .unproven_conjecture = c.withChallenge(challenge) },
        };
    }

    pub fn withSimulated(self: UnprovenTree, sim: bool) UnprovenTree {
        return switch (self) {
            .unproven_leaf => |l| .{ .unproven_leaf = l.withSimulated(sim) },
            .unproven_conjecture => |c| .{ .unproven_conjecture = c.withSimulated(sim) },
        };
    }
};
```

### Unproven Leaf Nodes

```zig
const UnprovenLeaf = union(enum) {
    unproven_schnorr: UnprovenSchnorr,
    unproven_dh_tuple: UnprovenDhTuple,

    // ... accessor methods
};

const UnprovenSchnorr = struct {
    proposition: ProveDlog,
    commitment_opt: ?FirstDlogProverMessage,
    randomness_opt: ?Scalar,  // Secret r for commitment
    challenge_opt: ?Challenge,
    simulated: bool,
    position: NodePosition,

    pub fn withChallenge(self: UnprovenSchnorr, c: Challenge) UnprovenSchnorr {
        return .{
            .proposition = self.proposition,
            .commitment_opt = self.commitment_opt,
            .randomness_opt = self.randomness_opt,
            .challenge_opt = c,
            .simulated = self.simulated,
            .position = self.position,
        };
    }

    pub fn withSimulated(self: UnprovenSchnorr, sim: bool) UnprovenSchnorr {
        return .{
            .proposition = self.proposition,
            .commitment_opt = self.commitment_opt,
            .randomness_opt = self.randomness_opt,
            .challenge_opt = self.challenge_opt,
            .simulated = sim,
            .position = self.position,
        };
    }
};

const UnprovenDhTuple = struct {
    proposition: ProveDhTuple,
    commitment_opt: ?FirstDhTupleProverMessage,
    randomness_opt: ?Scalar,
    challenge_opt: ?Challenge,
    simulated: bool,
    position: NodePosition,
};
```

### Unproven Conjecture Nodes

```zig
const UnprovenConjecture = union(enum) {
    cand_unproven: CandUnproven,
    cor_unproven: CorUnproven,
    cthreshold_unproven: CthresholdUnproven,

    pub fn simulated(self: UnprovenConjecture) bool {
        return switch (self) {
            .cand_unproven => |c| c.simulated,
            .cor_unproven => |c| c.simulated,
            .cthreshold_unproven => |c| c.simulated,
        };
    }

    pub fn children(self: UnprovenConjecture) []ProofTree {
        return switch (self) {
            .cand_unproven => |c| c.children,
            .cor_unproven => |c| c.children,
            .cthreshold_unproven => |c| c.children,
        };
    }
};

const CandUnproven = struct {
    proposition: Cand,
    challenge_opt: ?Challenge,
    simulated: bool,
    children: []ProofTree,
    position: NodePosition,
};

const CorUnproven = struct {
    proposition: Cor,
    challenge_opt: ?Challenge,
    simulated: bool,
    children: []ProofTree,
    position: NodePosition,
};

const CthresholdUnproven = struct {
    proposition: Cthreshold,
    challenge_opt: ?Challenge,
    simulated: bool,
    k: u8,                        // Threshold
    children: []ProofTree,
    polynomial_opt: ?Gf2_192Poly, // For challenge distribution
    position: NodePosition,
};
```

## The Proving Algorithm

### Prover Trait

```zig
const Prover = struct {
    secrets: []const PrivateInput,

    pub fn prove(
        self: *const Prover,
        tree: *const ErgoTree,
        ctx: *const Context,
        message: []const u8,
        hints_bag: *const HintsBag,
    ) ProverError!ProverResult {
        const reduction = try reduceToCrypto(tree, ctx);
        const proof = try self.generateProof(
            reduction.sigma_prop,
            message,
            hints_bag,
        );
        return .{
            .proof = proof,
            .extension = ctx.extension,
        };
    }

    pub fn generateProof(
        self: *const Prover,
        sigma_bool: SigmaBoolean,
        message: []const u8,
        hints_bag: *const HintsBag,
    ) ProverError!ProofBytes {
        return switch (sigma_bool) {
            .trivial_prop => |b| blk: {
                if (b) break :blk ProofBytes.empty();
                return error.ReducedToFalse;
            },
            else => |sb| blk: {
                const unproven = try convertToUnproven(sb);
                const unchecked = try proveToUnchecked(self, unproven, message, hints_bag);
                break :blk serializeSig(unchecked);
            },
        };
    }
};
```

### Step 0: Convert to Unproven

Transform SigmaBoolean to UnprovenTree[^6]:

```zig
fn convertToUnproven(sigma_tree: SigmaBoolean) ProverError!UnprovenTree {
    return switch (sigma_tree) {
        .c_and => |and_node| blk: {
            var children = try allocator.alloc(ProofTree, and_node.children.len);
            for (and_node.children, 0..) |child, i| {
                children[i] = .{ .unproven_tree = try convertToUnproven(child) };
            }
            break :blk .{
                .unproven_conjecture = .{
                    .cand_unproven = .{
                        .proposition = and_node,
                        .challenge_opt = null,
                        .simulated = false,
                        .children = children,
                        .position = NodePosition.CRYPTO_TREE_PREFIX,
                    },
                },
            };
        },
        .c_or => |or_node| blk: {
            // Similar conversion for OR
            // ...
        },
        .c_threshold => |th| blk: {
            // Similar conversion for THRESHOLD
            // ...
        },
        .prove_dlog => |pk| .{
            .unproven_leaf = .{
                .unproven_schnorr = .{
                    .proposition = pk,
                    .commitment_opt = null,
                    .randomness_opt = null,
                    .challenge_opt = null,
                    .simulated = false,
                    .position = NodePosition.CRYPTO_TREE_PREFIX,
                },
            },
        },
        .prove_dh_tuple => |dht| .{
            .unproven_leaf = .{
                .unproven_dh_tuple = .{
                    .proposition = dht,
                    .commitment_opt = null,
                    .randomness_opt = null,
                    .challenge_opt = null,
                    .simulated = false,
                    .position = NodePosition.CRYPTO_TREE_PREFIX,
                },
            },
        },
        else => error.Unexpected,
    };
}
```

### Step 1: Mark Real Nodes

Bottom-up traversal to mark what prover can prove[^7][^8]:

```zig
fn markReal(
    prover: *const Prover,
    tree: UnprovenTree,
    hints_bag: *const HintsBag,
) ProverError!UnprovenTree {
    return rewriteBottomUp(tree, struct {
        fn transform(node: ProofTree, p: *const Prover, hints: *const HintsBag) ?ProofTree {
            return switch (node) {
                .unproven_tree => |ut| switch (ut) {
                    .unproven_leaf => |leaf| blk: {
                        // Leaf is real if prover has secret OR hint shows knowledge
                        const secret_known = hints.realImages().contains(leaf.proposition()) or
                            p.hasSecretFor(leaf.proposition());
                        break :blk leaf.withSimulated(!secret_known);
                    },
                    .unproven_conjecture => |conj| switch (conj) {
                        .cand_unproven => |cand| blk: {
                            // AND is real only if ALL children are real
                            const simulated = anyChildSimulated(cand.children);
                            break :blk cand.withSimulated(simulated);
                        },
                        .cor_unproven => |cor| blk: {
                            // OR is real if AT LEAST ONE child is real
                            const simulated = allChildrenSimulated(cor.children);
                            break :blk cor.withSimulated(simulated);
                        },
                        .cthreshold_unproven => |ct| blk: {
                            // THRESHOLD(k) is real if AT LEAST k children are real
                            const real_count = countRealChildren(ct.children);
                            break :blk ct.withSimulated(real_count < ct.k);
                        },
                    },
                },
                else => null,
            };
        }
    }.transform, prover, hints_bag);
}
```

### Step 2: Check Root

```zig
fn proveToUnchecked(
    prover: *const Prover,
    unproven: UnprovenTree,
    message: []const u8,
    hints_bag: *const HintsBag,
) ProverError!UncheckedTree {
    // Step 1
    const step1 = try markReal(prover, unproven, hints_bag);

    // Step 2: If root is simulated, prover cannot prove
    if (!step1.isReal()) {
        return error.TreeRootIsNotReal;
    }

    // Steps 3-9...
}
```

### Step 3: Polish Simulated

Top-down traversal to ensure correct structure[^9]:

```zig
fn polishSimulated(tree: UnprovenTree) ProverError!UnprovenTree {
    return rewriteTopDown(tree, struct {
        fn transform(node: ProofTree) ?ProofTree {
            return switch (node) {
                .unproven_tree => |ut| switch (ut) {
                    .unproven_conjecture => |conj| switch (conj) {
                        .cand_unproven => |cand| blk: {
                            // Simulated AND: all children simulated
                            if (cand.simulated) {
                                break :blk cand.withChildren(
                                    markAllChildrenSimulated(cand.children),
                                );
                            }
                            break :blk cand;
                        },
                        .cor_unproven => |cor| blk: {
                            if (cor.simulated) {
                                // Simulated OR: all children simulated
                                break :blk cor.withChildren(
                                    markAllChildrenSimulated(cor.children),
                                );
                            } else {
                                // Real OR: keep ONE child real, mark rest simulated
                                break :blk makeCorChildrenSimulated(cor);
                            }
                        },
                        .cthreshold_unproven => |ct| blk: {
                            if (ct.simulated) {
                                break :blk ct.withChildren(
                                    markAllChildrenSimulated(ct.children),
                                );
                            } else {
                                // Real THRESHOLD(k): keep only k children real
                                break :blk makeThresholdChildrenSimulated(ct);
                            }
                        },
                    },
                    else => null,
                },
                else => null,
            };
        }
    }.transform);
}

fn makeCorChildrenSimulated(cor: CorUnproven) CorUnproven {
    // Find first real child, mark all others simulated
    var found_real = false;
    var new_children = allocator.alloc(ProofTree, cor.children.len);
    for (cor.children, 0..) |child, i| {
        const ut = child.unproven_tree;
        if (ut.isReal() and !found_real) {
            new_children[i] = child;
            found_real = true;
        } else if (ut.isReal()) {
            new_children[i] = ut.withSimulated(true);
        } else {
            new_children[i] = child;
        }
    }
    return cor.withChildren(new_children);
}
```

### Steps 4-6: Simulate and Commit

Combined traversal for challenges, simulation, and commitments[^10][^11]:

```zig
fn simulateAndCommit(
    tree: UnprovenTree,
    hints_bag: *const HintsBag,
    rng: std.rand.Random,
) ProverError!ProofTree {
    return rewriteTopDown(tree, struct {
        fn transform(node: ProofTree, hints: *const HintsBag, random: std.rand.Random) ?ProofTree {
            return switch (node) {
                .unproven_tree => |ut| switch (ut) {
                    // Step 4: Real conjecture assigns random challenges to simulated children
                    .unproven_conjecture => |conj| blk: {
                        if (conj.isReal()) {
                            break :blk assignChallengesFromRealParent(conj, random);
                        } else {
                            break :blk propagateChallengeToSimulatedChildren(conj, random);
                        }
                    },
                    // Steps 5-6: Simulate or commit at leaves
                    .unproven_leaf => |leaf| blk: {
                        if (leaf.simulated()) {
                            // Step 5: Simulate
                            break :blk simulateLeaf(leaf);
                        } else {
                            // Step 6: Compute commitment
                            break :blk commitLeaf(leaf, hints, random);
                        }
                    },
                },
                else => null,
            };
        }
    }.transform, hints_bag, rng);
}

/// Simulate a leaf: pick random z, compute commitment backwards
fn simulateLeaf(leaf: UnprovenLeaf) UncheckedTree {
    return switch (leaf) {
        .unproven_schnorr => |us| blk: {
            const challenge = us.challenge_opt orelse return error.SimulatedLeafWithoutChallenge;
            const sim = DlogProver.simulate(us.proposition, challenge);
            break :blk .{
                .unchecked_leaf = .{
                    .unchecked_schnorr = .{
                        .proposition = us.proposition,
                        .commitment_opt = sim.first_message,
                        .challenge = challenge,
                        .second_message = sim.second_message,
                    },
                },
            };
        },
        .unproven_dh_tuple => |ud| blk: {
            // Similar for DHT
        },
    };
}

/// Commit at a real leaf: pick random r, compute a = g^r
fn commitLeaf(
    leaf: UnprovenLeaf,
    hints: *const HintsBag,
    rng: std.rand.Random,
) UnprovenTree {
    return switch (leaf) {
        .unproven_schnorr => |us| blk: {
            // Check hints first
            if (hints.findCommitment(us.position)) |hint| {
                break :blk us.withCommitment(hint.commitment);
            }
            // Generate fresh commitment
            const first = DlogProver.firstMessage(rng);
            break :blk .{
                .unproven_leaf = .{
                    .unproven_schnorr = .{
                        .proposition = us.proposition,
                        .commitment_opt = first.message,
                        .randomness_opt = first.r,
                        .challenge_opt = null,
                        .simulated = false,
                        .position = us.position,
                    },
                },
            };
        },
        // Similar for DHT
    };
}
```

### Steps 7-8: Fiat-Shamir

Serialize tree and compute root challenge[^12]:

```zig
fn computeRootChallenge(tree: ProofTree, message: []const u8) Challenge {
    // Step 7: Serialize tree structure + propositions + commitments
    var buf = std.ArrayList(u8).init(allocator);
    fiatShamirTreeToBytes(&tree, buf.writer());

    // Step 8: Append message and hash
    buf.appendSlice(message);
    return fiatShamirHashFn(buf.items);
}
```

### Step 9: Compute Real Challenges and Responses

Top-down traversal for real nodes[^13][^14]:

```zig
fn proving(
    prover: *const Prover,
    tree: ProofTree,
    hints_bag: *const HintsBag,
) ProverError!ProofTree {
    return rewriteTopDown(tree, struct {
        fn transform(node: ProofTree, p: *const Prover, hints: *const HintsBag) ?ProofTree {
            return switch (node) {
                .unproven_tree => |ut| switch (ut) {
                    .unproven_conjecture => |conj| blk: {
                        if (!conj.isReal()) break :blk null;

                        switch (conj) {
                            .cand_unproven => |cand| blk: {
                                // Real AND: all children get same challenge
                                const challenge = cand.challenge_opt.?;
                                break :blk cand.withChildren(
                                    propagateChallenge(cand.children, challenge),
                                );
                            },
                            .cor_unproven => |cor| blk: {
                                // Real OR: real child gets XOR of root and simulated
                                const root_challenge = cor.challenge_opt.?;
                                const xored = xorChallenges(root_challenge, cor.children);
                                break :blk cor.withRealChildChallenge(xored);
                            },
                            .cthreshold_unproven => |ct| blk: {
                                // Real THRESHOLD: polynomial interpolation
                                break :blk computeThresholdChallenges(ct);
                            },
                        }
                    },
                    .unproven_leaf => |leaf| blk: {
                        if (!leaf.isReal()) break :blk null;

                        // Compute response z = r + e*w mod q
                        const challenge = leaf.challenge_opt orelse
                            return error.RealUnprovenTreeWithoutChallenge;

                        switch (leaf) {
                            .unproven_schnorr => |us| blk: {
                                const secret = p.findSecret(us.proposition) orelse
                                    hints.findRealProof(us.position)?.unchecked.second_message orelse
                                    return error.SecretNotFound;

                                const z = DlogProver.secondMessage(
                                    secret,
                                    us.randomness_opt.?,
                                    challenge,
                                );
                                break :blk .{
                                    .unchecked_leaf = .{
                                        .unchecked_schnorr = .{
                                            .proposition = us.proposition,
                                            .commitment_opt = null,
                                            .challenge = challenge,
                                            .second_message = z,
                                        },
                                    },
                                };
                            },
                            // Similar for DHT
                        }
                    },
                },
                else => null,
            };
        }
    }.transform, prover, hints_bag);
}
```

### Step 10: Serialize Proof

```zig
fn serializeSig(tree: UncheckedTree) ProofBytes {
    var buf = std.ArrayList(u8).init(allocator);
    var w = SigmaByteWriter.init(buf.writer());

    sigWriteBytes(&tree, &w, true);

    return .{ .bytes = buf.items };
}

fn sigWriteBytes(node: *const UncheckedTree, w: *SigmaByteWriter, write_challenge: bool) void {
    if (write_challenge) {
        w.writeBytes(&node.challenge());
    }

    switch (node.*) {
        .unchecked_leaf => |leaf| switch (leaf) {
            .unchecked_schnorr => |us| {
                w.writeBytes(&us.second_message.z.toBytes());
            },
            .unchecked_dh_tuple => |dh| {
                w.writeBytes(&dh.second_message.z.toBytes());
            },
        },
        .unchecked_conjecture => |conj| switch (conj) {
            .cand_unchecked => |cand| {
                // Children's challenges equal parent's - don't write
                for (cand.children) |child| {
                    sigWriteBytes(&child, w, false);
                }
            },
            .cor_unchecked => |cor| {
                // Write all except last (computed via XOR)
                for (cor.children[0 .. cor.children.len - 1]) |child| {
                    sigWriteBytes(&child, w, true);
                }
                sigWriteBytes(&cor.children[cor.children.len - 1], w, false);
            },
            .cthreshold_unchecked => |ct| {
                // Write polynomial coefficients
                w.writeBytes(ct.polynomial.toBytes(false));
                for (ct.children) |child| {
                    sigWriteBytes(&child, w, false);
                }
            },
        },
    };
}
```

## Response Computation

### Schnorr Response

```zig
const DlogProver = struct {
    /// First message: a = g^r
    pub fn firstMessage(rng: std.rand.Random) struct { r: Scalar, message: FirstDlogProverMessage } {
        const r = Scalar.random(rng);
        const a = DlogGroup.exponentiate(&DlogGroup.generator(), &r);
        return .{ .r = r, .message = .{ .a = a } };
    }

    /// Second message: z = r + e*w mod q
    pub fn secondMessage(
        private_key: DlogProverInput,
        r: Scalar,
        challenge: Challenge,
    ) SecondDlogProverMessage {
        const e = Scalar.fromBytes(&challenge.bytes);
        const z = r.add(e.mul(private_key.w));
        return .{ .z = z };
    }

    /// Simulation: pick random z, compute a = g^z * h^(-e)
    pub fn simulate(
        proposition: ProveDlog,
        challenge: Challenge,
    ) struct { first_message: FirstDlogProverMessage, second_message: SecondDlogProverMessage } {
        const z = Scalar.random(rng);
        const e = Scalar.fromBytes(&challenge.bytes);
        const minus_e = e.negate();

        const gz = DlogGroup.exponentiate(&DlogGroup.generator(), &z);
        const h_neg_e = DlogGroup.exponentiate(&proposition.h, &minus_e);
        const a = gz.multiply(&h_neg_e);

        return .{
            .first_message = .{ .a = a },
            .second_message = .{ .z = z },
        };
    }
};
```

## Hint System

### Hint Types

For distributed signing[^15]:

```zig
const Hint = union(enum) {
    real_secret_proof: RealSecretProof,
    simulated_secret_proof: SimulatedSecretProof,
    own_commitment: OwnCommitment,
    real_commitment: RealCommitment,
    simulated_commitment: SimulatedCommitment,
};

const RealSecretProof = struct {
    image: SigmaBoolean,
    challenge: Challenge,
    unchecked_tree: UncheckedTree,
    position: NodePosition,
};

const OwnCommitment = struct {
    image: SigmaBoolean,
    secret_randomness: Scalar,  // Private!
    commitment: FirstProverMessage,
    position: NodePosition,
};

const RealCommitment = struct {
    image: SigmaBoolean,
    commitment: FirstProverMessage,
    position: NodePosition,
};

const HintsBag = struct {
    hints: []const Hint,

    pub fn realImages(self: *const HintsBag) []const SigmaBoolean {
        // Collect public images from real proofs and commitments
    }

    pub fn findCommitment(self: *const HintsBag, pos: NodePosition) ?CommitmentHint {
        for (self.hints) |hint| {
            switch (hint) {
                .own_commitment, .real_commitment => |c| {
                    if (c.position.eql(pos)) return c;
                },
                else => {},
            }
        }
        return null;
    }

    pub fn findRealProof(self: *const HintsBag, pos: NodePosition) ?RealSecretProof {
        for (self.hints) |hint| {
            if (hint == .real_secret_proof and hint.real_secret_proof.position.eql(pos)) {
                return hint.real_secret_proof;
            }
        }
        return null;
    }
};
```

### Distributed Signing Protocol

```
Distributed Signing (2-of-2 AND)
─────────────────────────────────────────────────────

Round 1: Generate commitments
  Party 1 (sk1) ─────> OwnCommitment(pk1, r1, g^r1)
  Party 2 (sk2) ─────> OwnCommitment(pk2, r2, g^r2)

Exchange: Share RealCommitment (NOT OwnCommitment!)
  Party 1 ─────> RealCommitment(pk1, g^r1) ─────> Party 2
  Party 2 ─────> RealCommitment(pk2, g^r2) ─────> Party 1

Round 2: Sign sequentially
  Party 1:
    combined = hints1 ++ RealCommitment(pk2)
    partialProof = prove(tree, msg, combined)

  Extract hints from partial:
    hintsFromProof = bagForMultisig(partialProof, ...)

  Party 2:
    combined = hints2 ++ hintsFromProof
    finalProof = prove(tree, msg, combined)
```

## Prover Errors

```zig
const ProverError = error{
    ErgoTreeError,
    EvalError,
    Gf2_192Error,
    ReducedToFalse,
    TreeRootIsNotReal,
    SimulatedLeafWithoutChallenge,
    RealUnprovenTreeWithoutChallenge,
    SecretNotFound,
    Unexpected,
    FiatShamirTreeSerializationError,
};
```

## Summary

The prover transforms a sigma-tree through multiple stages:
1. **Mark real** (bottom-up): Identify what prover can prove
2. **Polish simulated** (top-down): Ensure correct structure
3. **Simulate and commit**: Generate commitments for all leaves
4. **Fiat-Shamir**: Compute root challenge from commitments + message
5. **Prove** (top-down): Compute challenges and responses for real nodes
6. **Serialize**: Output compact proof format

Key principles:
- Simulated transcripts are indistinguishable from real (zero-knowledge)
- Challenge flow depends on composition type (AND/OR/THRESHOLD)
- GF(2^192) polynomials enable efficient threshold challenge distribution
- Hints enable distributed signing without sharing secrets

---

*Next: [Chapter 16: ErgoScript Parser](../part6/ch16-ergoscript-parser.md)*

[^1]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverInterpreter.scala:1-100`

[^2]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs:1-100`

[^3]: Rust: `ergotree-interpreter/src/sigma_protocol/unproven_tree.rs` (NodePosition)

[^4]: Scala: `interpreter/shared/src/main/scala/sigmastate/UnprovenTree.scala`

[^5]: Rust: `ergotree-interpreter/src/sigma_protocol/unproven_tree.rs:27-88`

[^6]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs` (convert_to_unproven)

[^7]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverInterpreter.scala` (markReal)

[^8]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs:243-305`

[^9]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs:367-400`

[^10]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverInterpreter.scala` (simulateAndCommit)

[^11]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs` (simulate_and_commit)

[^12]: Rust: `ergotree-interpreter/src/sigma_protocol/fiat_shamir.rs`

[^13]: Scala: `interpreter/shared/src/main/scala/sigmastate/interpreter/ProverInterpreter.scala` (proving)

[^14]: Rust: `ergotree-interpreter/src/sigma_protocol/prover.rs` (proving)

[^15]: Rust: `ergotree-interpreter/src/sigma_protocol/prover/hint.rs`
