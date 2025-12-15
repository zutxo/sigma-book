# Chapter 16: ErgoScript Parser

> **PRE-ALPHA WARNING**: This is a pre-alpha version of The Sigma Book. Content may be incomplete, inaccurate, or subject to change. Do not use as a source of truth. For authoritative information, consult the official repositories:
> - [sigmastate-interpreter](https://github.com/ScorexFoundation/sigmastate-interpreter) — Reference Scala implementation
> - [sigma-rust](https://github.com/ergoplatform/sigma-rust) — Rust implementation
> - [ergo](https://github.com/ergoplatform/ergo) — Ergo node


## Prerequisites

- AST nodes ([Chapter 4](../part2/ch04-value-nodes.md))
- Type system ([Chapter 2](../part1/ch02-type-system.md))
- Basic understanding of parser combinators

## Learning Objectives

- Understand parser combinator and Pratt parsing techniques
- Master parser module structure (lexer, grammar, expressions, types)
- Learn operator precedence via binding power
- Trace expression parsing from source to AST
- Handle source position tracking for errors

## Parser Architecture

ErgoScript source code transforms to AST through lexing and parsing[^1][^2]:

```
Parsing Pipeline
─────────────────────────────────────────────────────

Source Code
    │
    ▼
┌──────────────────────────────────────────────────┐
│                    LEXER                         │
│                                                  │
│  Characters ─────> Tokens                        │
│  "val x = 1 + 2"                                 │
│  ─────>  [ValKw, Ident("x"), Eq, Int(1),        │
│           Plus, Int(2)]                          │
└──────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────┐
│                   PARSER                         │
│                                                  │
│  Tokens ─────> AST                               │
│  Grammar rules, precedence, associativity        │
│  ─────>  ValDef("x", BinOp(Int(1), +, Int(2)))  │
└──────────────────────────────────────────────────┘
    │
    ▼
Untyped AST (SValue)
```

## Lexer (Tokenizer)

Converts character stream to tokens[^3]:

```zig
const TokenKind = enum {
    // Literals
    int_number,
    long_number,
    string_literal,

    // Keywords
    val_kw,
    def_kw,
    if_kw,
    else_kw,
    true_kw,
    false_kw,

    // Operators
    plus,
    minus,
    star,
    slash,
    percent,
    eq,
    neq,
    lt,
    gt,
    le,
    ge,
    and_and,
    or_or,
    bang,

    // Punctuation
    l_paren,
    r_paren,
    l_brace,
    r_brace,
    l_bracket,
    r_bracket,
    dot,
    comma,
    colon,
    semicolon,
    arrow,

    // Identifiers
    ident,

    // Special
    whitespace,
    comment,
    eof,
    err,
};

const Token = struct {
    kind: TokenKind,
    text: []const u8,
    range: Range,
};

const Range = struct {
    start: usize,
    end: usize,
};
```

### Lexer Implementation

```zig
const Lexer = struct {
    source: []const u8,
    pos: usize,

    pub fn init(source: []const u8) Lexer {
        return .{ .source = source, .pos = 0 };
    }

    pub fn nextToken(self: *Lexer) Token {
        self.skipWhitespaceAndComments();

        if (self.pos >= self.source.len) {
            return .{ .kind = .eof, .text = "", .range = .{ .start = self.pos, .end = self.pos } };
        }

        const start = self.pos;
        const c = self.source[self.pos];

        // Single-character tokens
        const single_char_token: ?TokenKind = switch (c) {
            '(' => .l_paren,
            ')' => .r_paren,
            '{' => .l_brace,
            '}' => .r_brace,
            '[' => .l_bracket,
            ']' => .r_bracket,
            '.' => .dot,
            ',' => .comma,
            ':' => .colon,
            ';' => .semicolon,
            '+' => .plus,
            '-' => .minus,
            '*' => .star,
            '/' => .slash,
            '%' => .percent,
            else => null,
        };

        if (single_char_token) |kind| {
            self.pos += 1;
            return .{ .kind = kind, .text = self.source[start..self.pos], .range = .{ .start = start, .end = self.pos } };
        }

        // Multi-character tokens
        if (c == '=' and self.peek(1) == '=') {
            self.pos += 2;
            return .{ .kind = .eq, .text = "==", .range = .{ .start = start, .end = self.pos } };
        }

        if (c == '=' and self.peek(1) == '>') {
            self.pos += 2;
            return .{ .kind = .arrow, .text = "=>", .range = .{ .start = start, .end = self.pos } };
        }

        if (c == '&' and self.peek(1) == '&') {
            self.pos += 2;
            return .{ .kind = .and_and, .text = "&&", .range = .{ .start = start, .end = self.pos } };
        }

        // Numbers
        if (std.ascii.isDigit(c)) {
            return self.scanNumber(start);
        }

        // Identifiers and keywords
        if (std.ascii.isAlphabetic(c) or c == '_') {
            return self.scanIdentifier(start);
        }

        // Unknown character
        self.pos += 1;
        return .{ .kind = .err, .text = self.source[start..self.pos], .range = .{ .start = start, .end = self.pos } };
    }

    fn scanIdentifier(self: *Lexer, start: usize) Token {
        while (self.pos < self.source.len) {
            const c = self.source[self.pos];
            if (std.ascii.isAlphanumeric(c) or c == '_') {
                self.pos += 1;
            } else {
                break;
            }
        }

        const text = self.source[start..self.pos];
        const kind: TokenKind = if (keywords.get(text)) |kw| kw else .ident;
        return .{ .kind = kind, .text = text, .range = .{ .start = start, .end = self.pos } };
    }

    fn scanNumber(self: *Lexer, start: usize) Token {
        // Check for hex
        if (self.source[self.pos] == '0' and self.pos + 1 < self.source.len and
            (self.source[self.pos + 1] == 'x' or self.source[self.pos + 1] == 'X'))
        {
            self.pos += 2;
            while (self.pos < self.source.len and std.ascii.isHex(self.source[self.pos])) {
                self.pos += 1;
            }
        } else {
            while (self.pos < self.source.len and std.ascii.isDigit(self.source[self.pos])) {
                self.pos += 1;
            }
        }

        // Check for L suffix (long)
        var kind: TokenKind = .int_number;
        if (self.pos < self.source.len and (self.source[self.pos] == 'L' or self.source[self.pos] == 'l')) {
            kind = .long_number;
            self.pos += 1;
        }

        return .{ .kind = kind, .text = self.source[start..self.pos], .range = .{ .start = start, .end = self.pos } };
    }

    const keywords = std.ComptimeStringMap(TokenKind, .{
        .{ "val", .val_kw },
        .{ "def", .def_kw },
        .{ "if", .if_kw },
        .{ "else", .else_kw },
        .{ "true", .true_kw },
        .{ "false", .false_kw },
    });
};
```

## Parser Structure

Event-based parser using markers[^4][^5]:

```zig
const Event = union(enum) {
    start_node: SyntaxKind,
    add_token,
    finish_node,
    err: ParseError,
    placeholder,
};

const Parser = struct {
    source: Source,
    events: std.ArrayList(Event),
    expected_kinds: std.ArrayList(TokenKind),
    allocator: Allocator,

    pub fn init(allocator: Allocator, tokens: []const Token) Parser {
        return .{
            .source = Source.init(tokens),
            .events = std.ArrayList(Event).init(allocator),
            .expected_kinds = std.ArrayList(TokenKind).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn parse(self: *Parser) []Event {
        grammar.root(self);
        return self.events.toOwnedSlice();
    }

    fn start(self: *Parser) Marker {
        const pos = self.events.items.len;
        try self.events.append(.placeholder);
        return Marker.init(pos);
    }

    fn at(self: *Parser, kind: TokenKind) bool {
        try self.expected_kinds.append(kind);
        return self.peek() == kind;
    }

    fn bump(self: *Parser) void {
        self.expected_kinds.clearRetainingCapacity();
        _ = self.source.nextToken();
        try self.events.append(.add_token);
    }

    fn expect(self: *Parser, kind: TokenKind) void {
        if (self.at(kind)) {
            self.bump();
        } else {
            self.err();
        }
    }
};

const Marker = struct {
    pos: usize,

    pub fn init(pos: usize) Marker {
        return .{ .pos = pos };
    }

    pub fn complete(self: Marker, p: *Parser, kind: SyntaxKind) CompletedMarker {
        p.events.items[self.pos] = .{ .start_node = kind };
        try p.events.append(.finish_node);
        return .{ .pos = self.pos };
    }

    pub fn precede(self: Marker, p: *Parser) Marker {
        const new_marker = p.start();
        p.events.items[self.pos] = .{ .start_node_at = new_marker.pos };
        return new_marker;
    }
};
```

## Pratt Parsing (Binding Power)

Expression parsing uses Pratt parsing for operator precedence[^6][^7]:

```
Binding Power Concept
─────────────────────────────────────────────────────

Expression:   A       +       B       *       C
Power:           3       3       5       5

The * has higher binding power, holds B and C tighter.
Result: A + (B * C)

Associativity via asymmetric power:
Expression:   A       +       B       +       C
Power:     0     3      3.1     3      3.1     0

Right power slightly higher → left associativity
Result: (A + B) + C
```

### Expression Grammar

```zig
const grammar = struct {
    pub fn root(p: *Parser) CompletedMarker {
        const m = p.start();
        while (!p.atEnd()) {
            stmt(p);
        }
        return m.complete(p, .root);
    }

    pub fn expr(p: *Parser) ?CompletedMarker {
        return exprBindingPower(p, 0);
    }

    /// Pratt parser core
    fn exprBindingPower(p: *Parser, min_bp: u8) ?CompletedMarker {
        var lhs = lhs(p) orelse return null;

        while (true) {
            const op: ?BinaryOp = blk: {
                if (p.at(.plus)) break :blk .add;
                if (p.at(.minus)) break :blk .sub;
                if (p.at(.star)) break :blk .mul;
                if (p.at(.slash)) break :blk .div;
                if (p.at(.percent)) break :blk .mod;
                if (p.at(.lt)) break :blk .lt;
                if (p.at(.gt)) break :blk .gt;
                if (p.at(.le)) break :blk .le;
                if (p.at(.ge)) break :blk .ge;
                if (p.at(.eq)) break :blk .eq;
                if (p.at(.neq)) break :blk .neq;
                if (p.at(.and_and)) break :blk .and_;
                if (p.at(.or_or)) break :blk .or_;
                break :blk null;
            };

            if (op == null) break;

            const bp = op.?.bindingPower();
            if (bp.left < min_bp) break;

            // Consume operator
            p.bump();

            // Parse right operand with right binding power
            const m = lhs.precede(p);
            const parsed_rhs = exprBindingPower(p, bp.right) != null;
            lhs = m.complete(p, .infix_expr);

            if (!parsed_rhs) break;
        }

        return lhs;
    }

    /// Left-hand side (atoms and prefix expressions)
    fn lhs(p: *Parser) ?CompletedMarker {
        if (p.at(.int_number)) return intNumber(p);
        if (p.at(.long_number)) return longNumber(p);
        if (p.at(.ident)) return ident(p);
        if (p.at(.true_kw) or p.at(.false_kw)) return boolLiteral(p);
        if (p.at(.minus) or p.at(.bang)) return prefixExpr(p);
        if (p.at(.l_paren)) return parenExpr(p);
        if (p.at(.l_brace)) return blockExpr(p);
        if (p.at(.if_kw)) return ifExpr(p);

        p.err();
        return null;
    }

    fn intNumber(p: *Parser) CompletedMarker {
        const m = p.start();
        p.bump();
        return m.complete(p, .int_literal);
    }

    fn ident(p: *Parser) CompletedMarker {
        const m = p.start();
        p.bump();
        return m.complete(p, .ident);
    }

    fn prefixExpr(p: *Parser) ?CompletedMarker {
        const m = p.start();
        const op_bp = UnaryOp.fromToken(p.peek()).?.bindingPower();

        p.bump(); // operator
        _ = exprBindingPower(p, op_bp.right);

        return m.complete(p, .prefix_expr);
    }

    fn parenExpr(p: *Parser) CompletedMarker {
        const m = p.start();
        p.expect(.l_paren);
        _ = expr(p);
        p.expect(.r_paren);
        return m.complete(p, .paren_expr);
    }

    fn ifExpr(p: *Parser) CompletedMarker {
        const m = p.start();
        p.expect(.if_kw);
        p.expect(.l_paren);
        _ = expr(p);
        p.expect(.r_paren);
        _ = expr(p);
        if (p.at(.else_kw)) {
            p.bump();
            _ = expr(p);
        }
        return m.complete(p, .if_expr);
    }
};
```

### Binary Operators

```zig
const BinaryOp = enum {
    add,
    sub,
    mul,
    div,
    mod,
    lt,
    gt,
    le,
    ge,
    eq,
    neq,
    and_,
    or_,

    const BindingPower = struct { left: u8, right: u8 };

    pub fn bindingPower(self: BinaryOp) BindingPower {
        return switch (self) {
            .or_ => .{ .left = 1, .right = 2 },      // ||
            .and_ => .{ .left = 3, .right = 4 },     // &&
            .eq, .neq => .{ .left = 5, .right = 6 }, // ==, !=
            .lt, .gt, .le, .ge => .{ .left = 7, .right = 8 },
            .add, .sub => .{ .left = 9, .right = 10 },
            .mul, .div, .mod => .{ .left = 11, .right = 12 },
        };
    }
};

const UnaryOp = enum {
    neg,
    not,

    pub fn bindingPower(self: UnaryOp) struct { right: u8 } {
        return switch (self) {
            .neg, .not => .{ .right = 13 }, // Higher than all binary
        };
    }
};
```

## Operator Precedence Table

```
Operator Precedence (lowest to highest)
─────────────────────────────────────────────────────
 1-2    ||                 Logical OR
 3-4    &&                 Logical AND
 5-6    == !=              Equality
 7-8    < > <= >=          Comparison
 9-10   + -                Addition, Subtraction
11-12   * / %              Multiplication, Division
  13    - ! ~              Prefix (unary)
  14    . ()               Postfix (method call, index)
```

## Type Parsing

```zig
const TypeParser = struct {
    const predef_types = std.ComptimeStringMap(SType, .{
        .{ "Boolean", .s_boolean },
        .{ "Byte", .s_byte },
        .{ "Short", .s_short },
        .{ "Int", .s_int },
        .{ "Long", .s_long },
        .{ "BigInt", .s_big_int },
        .{ "GroupElement", .s_group_element },
        .{ "SigmaProp", .s_sigma_prop },
        .{ "Box", .s_box },
        .{ "AvlTree", .s_avl_tree },
        .{ "Context", .s_context },
        .{ "Header", .s_header },
        .{ "PreHeader", .s_pre_header },
        .{ "Unit", .s_unit },
    });

    pub fn parseType(p: *Parser) ?SType {
        if (p.at(.ident)) {
            const name = p.currentText();

            // Check predefined types
            if (predef_types.get(name)) |t| {
                p.bump();
                return t;
            }

            // Generic types: Coll[T], Option[T]
            p.bump();
            if (p.at(.l_bracket)) {
                p.bump();
                const inner = parseType(p) orelse return null;
                p.expect(.r_bracket);

                if (std.mem.eql(u8, name, "Coll")) {
                    return .{ .s_coll = inner };
                } else if (std.mem.eql(u8, name, "Option")) {
                    return .{ .s_option = inner };
                }
            }

            // Type variable
            return .{ .s_type_var = name };
        }

        // Tuple type: (T1, T2, ...)
        if (p.at(.l_paren)) {
            p.bump();
            var items = std.ArrayList(SType).init(p.allocator);
            while (!p.at(.r_paren)) {
                const t = parseType(p) orelse return null;
                try items.append(t);
                if (!p.at(.r_paren)) p.expect(.comma);
            }
            p.expect(.r_paren);
            return .{ .s_tuple = items.toOwnedSlice() };
        }

        // Function type: T1 => T2
        const domain = parseType(p) orelse return null;
        if (p.at(.arrow)) {
            p.bump();
            const range = parseType(p) orelse return null;
            return .{ .s_func = .{ .args = &[_]SType{domain}, .ret = range } };
        }

        return domain;
    }
};
```

## Statement Parsing

```zig
fn stmt(p: *Parser) ?CompletedMarker {
    if (p.at(.val_kw)) {
        return valDef(p);
    }
    if (p.at(.def_kw)) {
        return defDef(p);
    }
    return expr(p);
}

fn valDef(p: *Parser) CompletedMarker {
    const m = p.start();
    p.expect(.val_kw);
    p.expect(.ident);

    // Optional type annotation
    if (p.at(.colon)) {
        p.bump();
        _ = TypeParser.parseType(p);
    }

    p.expect(.eq);
    _ = expr(p);

    return m.complete(p, .val_def);
}

fn defDef(p: *Parser) CompletedMarker {
    const m = p.start();
    p.expect(.def_kw);
    p.expect(.ident);

    // Parameters
    if (p.at(.l_paren)) {
        p.bump();
        while (!p.at(.r_paren)) {
            p.expect(.ident);
            p.expect(.colon);
            _ = TypeParser.parseType(p);
            if (!p.at(.r_paren)) p.expect(.comma);
        }
        p.expect(.r_paren);
    }

    // Return type
    if (p.at(.colon)) {
        p.bump();
        _ = TypeParser.parseType(p);
    }

    p.expect(.eq);
    _ = expr(p);

    return m.complete(p, .def_def);
}
```

## Source Position Tracking

Every AST node carries source position for error messages[^8]:

```zig
const SourceContext = struct {
    index: usize,
    line: u32,
    column: u32,
    source_line: []const u8,

    pub fn fromIndex(index: usize, source: []const u8) SourceContext {
        var line: u32 = 1;
        var col: u32 = 1;
        var line_start: usize = 0;

        for (source[0..index], 0..) |c, i| {
            if (c == '\n') {
                line += 1;
                col = 1;
                line_start = i + 1;
            } else {
                col += 1;
            }
        }

        // Find end of current line
        var line_end = index;
        while (line_end < source.len and source[line_end] != '\n') {
            line_end += 1;
        }

        return .{
            .index = index,
            .line = line,
            .column = col,
            .source_line = source[line_start..line_end],
        };
    }
};

const ParseError = struct {
    expected: []const TokenKind,
    found: ?TokenKind,
    span: Range,

    pub fn format(self: ParseError, ctx: SourceContext) []const u8 {
        // Format error message with source context
    }
};
```

## Syntax Tree Construction

Events convert to concrete syntax tree[^9]:

```zig
const SyntaxKind = enum {
    // Nodes
    root,
    val_def,
    def_def,
    if_expr,
    block_expr,
    infix_expr,
    prefix_expr,
    paren_expr,
    lambda_expr,
    apply_expr,
    select_expr,

    // Literals
    int_literal,
    long_literal,
    bool_literal,
    string_literal,
    ident,

    // Error
    err,
};

const SyntaxNode = struct {
    kind: SyntaxKind,
    range: Range,
    children: []SyntaxNode,
    text: ?[]const u8,
};

fn buildTree(events: []const Event, tokens: []const Token) SyntaxNode {
    var builder = TreeBuilder.init();

    for (events) |event| {
        switch (event) {
            .start_node => |kind| builder.startNode(kind),
            .add_token => builder.addToken(tokens[builder.token_idx]),
            .finish_node => builder.finishNode(),
            .err => |e| builder.addError(e),
            .placeholder => {},
        }
    }

    return builder.finish();
}
```

## Parsing Example

```
Input: "val x = 1 + 2 * 3"

Tokens:
  [val_kw, ident("x"), eq, int(1), plus, int(2), star, int(3)]

Events:
  start_node(val_def)
    add_token(val_kw)
    add_token(ident)
    add_token(eq)
    start_node(infix_expr)       // 1 + (2 * 3)
      add_token(int)             // 1
      add_token(plus)
      start_node(infix_expr)     // 2 * 3
        add_token(int)           // 2
        add_token(star)
        add_token(int)           // 3
      finish_node
    finish_node
  finish_node

AST:
  ValDef
    name: "x"
    rhs: InfixExpr(+)
           lhs: IntLiteral(1)
           rhs: InfixExpr(*)
                  lhs: IntLiteral(2)
                  rhs: IntLiteral(3)
```

## Error Recovery

```zig
const RECOVERY_SET = [_]TokenKind{ .val_kw, .def_kw, .r_brace };

fn err(p: *Parser) void {
    const current = p.source.peekToken();
    const range = if (current) |t| t.range else p.source.lastTokenRange();

    try p.events.append(.{
        .err = .{
            .expected = p.expected_kinds.toOwnedSlice(),
            .found = if (current) |t| t.kind else null,
            .span = range,
        },
    });

    // Skip tokens until recovery point
    if (!p.atSet(&RECOVERY_SET) and !p.atEnd()) {
        const m = p.start();
        p.bump();
        _ = m.complete(p, .err);
    }
}
```

## Summary

- **Lexer** converts characters to tokens with position tracking
- **Parser** uses event-based architecture with markers
- **Pratt parsing** handles operator precedence via binding power
- **Left associativity**: right power slightly higher than left
- **Source positions** enable accurate error messages
- **Error recovery** skips to synchronization points
- Output is untyped AST; semantic analysis comes next

---

*Next: [Chapter 17: Semantic Analysis](./ch17-semantic-analysis.md)*

[^1]: Scala: `parsers/shared/src/main/scala/sigmastate/lang/SigmaParser.scala`

[^2]: Rust: `ergoscript-compiler/src/parser.rs`

[^3]: Rust: `ergoscript-compiler/src/lexer.rs`

[^4]: Scala: `parsers/shared/src/main/scala/sigmastate/lang/parsers/Basic.scala`

[^5]: Rust: `ergoscript-compiler/src/parser/marker.rs`

[^6]: Scala: `parsers/shared/src/main/scala/sigmastate/lang/parsers/Exprs.scala`

[^7]: Rust: `ergoscript-compiler/src/parser/grammar/expr.rs:1-60`

[^8]: Scala: `parsers/shared/src/main/scala/sigmastate/lang/SourceContext.scala`

[^9]: Rust: `ergoscript-compiler/src/parser/sink.rs`