NIF data format
===============

The NIF data format is a text based file format designed for compiler frontend/backend
communication or communication between different programming languages. The design is
heavily tied to Nim's requirements. However, the design works on language agnostic ASTs
and is so extensible that other programming languages work well with it too.

A NIF file corresponds to a "module" in the source language. The module is stored as an AST.
The AST consists of "atoms" and "compound nodes".


Extensibility is primarily achieved by using two different namespaces. One namespace
is used for "node kinds" / "tags" and a different one for source level identifiers. This does
away with the notion of a fixed set of "keywords". In NIF new "keywords" ("tags")
can be introduced without breaking any code.

There is also an optional **index structure** that maps symbols to offsets within a NIF file.
This makes NIF a hybrid between a binary and a text file.


Version 2026
------------

This document describes the **2026** version of NIF. Differences to the original version from 2024:

- The `(.nif24)` directive and other directives have been removed.
- The index structure became an official part of the spec.
- How symbol names must be formed is more refined.
- Global symbol names can be shortened by a trailing dot.


Example NIF module
------------------

In order to get a feeling for how a NIF file can look, here is a complete example:

```nif
(stmts
(imp 2,5,sysio.nim(type :File (object ..)))
(imp (proc :write.1.sys . (pragmas varargs) (params (param f File)).))
(call write.1.sys "Hello World!\0A")
)
```

<div style="page-break-after: always;"></div>

Encoding
--------

A NIF file is stored as a mere sequence of bytes ("octets"). No Unicode validation
steps are performed and no BOM-prefix can be used.


Whitespace
----------

Whitespace is used to separate tokens from each other but apart from that carries no
meaning and a NIF parser is supposed to ignore whitespace. Editors and other tools
can format and layout NIF code to be pleasing to look at.

Whitespace is the set `{' ', '\t', '\n', '\r'}`.


Control characters
------------------

NIF uses some characters like `(`, `)` and `~` to describe the AST. As such these characters
**must not** occur in string literals, char literals and comments so that a NIF parser can
skip to the enclosing `)` without complex logic.

The control characters are:

```
( )  [ ]  { }  ~  #  '  "  \  :
```

Escape sequences
----------------

Grammar:

```
HexChar ::= [0-9A-F]
Escape ::= '\' HexChar HexChar
```

String and character literals support escape sequences via backslashes quite like in other
languages. Unlike other languages only `\xx` where `xx` stands for the ASCII value that is
encoded is supported. Characters of a value < 32 (space) have to be encoded as `\xx` too.

For example, a binary zero in a string literal is written as `"\00"`.

*Caution*: The commonly used `"\\"` in other languages that escapes the backslash itself
is not supported either and must be written as `\5C`!

*Rationale*: Ease of implementation.


Atoms
-----

### Empty

```
Empty ::= '.'
```

As a special syntactic extension, the "empty" or "missing" node is written as a single dot `.`.
Empty nodes are frequently used because a construct like `let` can have optional parts like
pragmas, a type annotation or an initial value. The empty node is then used to ensure that a
`let` node (for example) always has a fixed number of children and that the N-th child is
always a type or empty.

Empty nodes do not require whitespace in between them: `...` is a list of 3 empty nodes.


### Identifiers

The most common atom is the "identifier". Its spelling must adhere to the grammar:

```
IdentStart ::= <unicode_letter> | '_' | Escape
IdentChar ::= IdentStart | [_0-9]
Identifier ::= IdentStart+ IdentChar*
```


Identifiers have no real meaning, in particular it **cannot** be assumed that two identifiers
of the same string (for example `abc`) refer to the same entity.

Identifiers that contain characters that are neither letters nor digits must be escaped via
backslashes, `\xx` much like it is used in string and character literals.


### Symbols

A "symbol" is a name that refers to an entity unambiguously. A symbol must adhere to the grammar:

```
Symbol ::= IdentStart IdentChar* '.' (IdentChar | '.')*
SymbolDef ::= ':' Symbol
```


Roughly speaking that is a "word" that must contain a dot but cannot start with a dot.

For example the 2nd proc named `foo` in a Nim module `m` would typically become `foo.2.m` in NIF.

Symbols that contain characters that are neither letters nor digits must be escaped via
backslashes, `\xx` much like it is used in string and character literals.

A `SymbolDef` is a symbol annotated with a leading ':'. It indicates that the parent node is
the node introducing this symbol. Thus a tool can implement a feature like "goto definition"
in a language agnostic way without having to know which node kinds introduce new symbols.

There are two kinds of symbols: local and global symbols. A local symbol is of the form `<ident>.<disamb>` where
`disamb` is a list of digits. For example a name like `foo.0` where the `0` implies it is the first symbol
originally named `foo`. The `0` is also called a "disambiguation number". Local symbols are not part of the
optional lookup index structure.

A global symbol is of the form `<ident>.<disamb>.<moduleSuffix>`, or `<ident>.<disamb>.<key>.<moduleSuffix>` where `key`
usually is the result from a generic instantiation.

A global symbol can leave out the `moduleSuffix` but then a trailing dot must be present to distinguish between
local and global symbols: `foo.0` is a **local symbol** in the current module, `foo.0.` is a **global symbol** that
is immediately expanded during parsing to `foo.0.modname` assuming the file being processed is `modname.nif`.


### Numbers

Grammar:

```
Digit ::= [0-9]
FloatingPointPart ::= ('.' Digit+ ('E' ('+' | '-')? Digit+)? ) | 'E' ('+' | '-')? Digit+
Number ::= ('+' | '-') Digit+ (FloatingPointPart | 'u')?
```

Numbers must start with a plus or a minus and only their decimal notation is supported.
For example Nim's `0xff` would become `256`.

Unsigned numbers always have a `u` suffix. Floating point numbers must contain a dot or `E`.
Every other number is interpreted as a signed integer.

Note that numbers that do not start with a plus nor a minus are interpreted as "line information". See
the corresponding section for more details.


### Char literals

Grammar:

```
VisibleChar ::= ASCII value >= 32 but not a control character
CharLiteral ::= '\'' (VisibleChar | Escape) '\''
```

Char literals are enclosed in single quotes. The only supported escape sequence is `\xx`.


### String literals


Grammar:

```
EscapedData ::= (VisibleChar | Escape | Whitespace)*
StringLiteral ::= '"' EscapedData '"'
```

String literals are enclosed in double quotes. The only supported escape sequence is `\xx`.
Whitespace, even including newlines, can be part of the string literal without having to
escape it.

For example:

```nif
  "This is a single\20
  literal string"
```

Produces: `"This is a single \n  literal string"`.


<div style="page-break-after: always;"></div>


Compound nodes
--------------

Grammar:

```
Atom ::= Empty | Identifier | Symbol | SymbolDef | Number | CharLiteral |
         StringLiteral

NodeKind ::= Identifier

Node ::= NodePrefix (Atom | CompoundNode)
NodePrefix ::= LineInfo? Comment?
CompoundNode ::= '(' NodeKind Node* ')'
```

The general syntax for a compound node is `(nodekind child1 child2 child3)`.

That means NIF is a Lisp with some extensions:

- The ability to annotate (line, column, filename) information for a node.
- The ability to annotate a node with a comment. (In Lisp comments are not attached to a node.)

Unlike in Lisp a function `call` is not implied so what is usually just `(f a b c)` in Lisp
becomes `(call f a b c)` in NIF. The first item in a list (`call` in the example) is called the "node kind".
There are many different node kinds and the set of node kinds is extensible.

However, usually at least the following node kinds exist and ensure a minimum of compatibility between
programming languages.

| Node kind | Description                                                                 |
| --------- | --------------------------------------------------------------------------- |
| `nil`     | A nil/null pointer. Note: This is not an atom so that it does not conflict with an identifier named `nil` and so that it can get a type annotation. |
| `false`   | The boolean value `false`. Note: This is not an atom so that it does not conflict with an identifier named `false`. |
| `true`    | The boolean value `true`. Note: This is not an atom so that it does not conflict with an identifier named `true`. |
| `nan`     | The floating point value `nan`. Note: This is not an atom so that it does not conflict with an identifier named `nan`. |
| `inf`    | The floating point value `infinity`. Note: This is not an atom so that it does not conflict with an identifier named `inf`. |
| `neginf`    | The floating point value `-infinity`. Note: This is not an atom so that it does not conflict with an identifier named `neginf`. |
| `stmts`   | A list of statements. |
| `expr`    | A list of statements but ending in an expression. |
| `imp`     | An import of a declaration from a different module. |
| `proc`    | A proc declaration. Note: For Nim `func`, `iterator` etc. are also used. |
| `type`    | A type declaration. |
| `params`  | Wraps a list of parameters. |
| `param`    | A parameter declaration. |
| `var`    | A var declaration. |
| `let`    | A let declaration. |
| `fld`    | An object field declaration. |
| `const`    | A const declaration. |
| `if`    | An `if` statement. |
| `elif`    | An `elif` section inside an `if` statement. |
| `else`    | An `else` section within an `if` statement. |
| `while` | A `while` loop. |
| `ret`   | A return statement. |
| `brk`   | A break statement. |
| `and`   | Logical `and` operator. |
| `or`    | Logical `or` operator. |
| `not`    | Logical `not` operator. |
| `addr`    | Address-of operator. |
| `deref`    | Pointer dereference operation. |
| `asgn`    | Assignment statement. |
| `at`    | Array index operation. |
| `dot`    | Object field selection. |
| `add`    | Arithmetic add instruction. Usually the first child is a type like `i32` to specify an `i32` addition. |
| `sub`    | Arithmetic sub instruction. Usually the first child is a type like `i32` to specify an `i32` subtraction. |
| `mul`    | Multiplication. Takes a type like `add`. |
| `div`    | Division. Takes a type like `add`. |
| `mod`    | Modulo operator. Takes a type like `add`. |
| `shr`    | Bit shift to the right. Takes a type like `add`. |
| `shl`    | Bit shift to the left. Takes a type like `add`. |
| `bitand`    | Bitwise and. Takes a type like `add`. |
| `bitor`    | Bitwise or. Takes a type like `add`. |
| `bitnot`   | Bitwise not. Takes a type like `add`. |
| `eq`    | Testing for equality. Takes a type like `add`. |
| `neq`   | Testing for "not equals" ("!="). Takes a type like `add`. |
| `le`    | Less than or equals ("<="). Takes a type like `add`. |
| `lt`    | Strictly less than ("<"). Takes a type like `add`. |
| `i` N | Where N can be 8, 16, ... The signed integer type that uses N bits. |
| `u` N | Where N can be 8, 16, ... The unsigned integer type that uses N bits. |
| `f` N | Where N can be 8, 16, ... The floating point type that uses N bits. |
| `array`    | Type constructor that produces an `array` type. |
| `object`    | Type constructor that produces an `object` type. |
| `ptr`    | Type constructor that produces a pointer type. |
| `proctype`    | Type constructor that produces a proc type. |
| `pragmas`    | List of pragmas. |
| `kv`    | A single (key, value) pair. The `ExprColonExpr` node kind in Nim. |
| `vv`    | A (value, value) pair. The `ExprEqExpr` node kind in Nim. |
| `par`    | Wraps an expression inside parentheses. |
| `cons`    | An object/array/etc. constructor. First child is a type. |
| `lab`    | A label declaration (target of a `jmp`). |
| `jmp`    | A jump or goto instruction. |


Line information
----------------

Grammar:

```
LineDiff ::= Digit* | '~' Digit+
LineInfo ::= LineDiff (',' LineDiff (',' EscapedData)?)?
```

Every node can be prefixed with a digit or `~` or `,` to add source code information.
("This node originates from file.nim(line,col).")
There are 3 forms:

1. `<column-diff>`
2. `<column-diff, line-diff>`
3. `<column, line, filename>`

The `diff` means that the value is relative to the parent node. For example `8` means that the node is at
the same position as the parent node except that its column is `+8` characters. Negative numbers use the tilde
and not the minus. Negative numbers are usually required for "infix" nodes where the left hand operand
preceeds the parent (`x + y` becomes
`(infix add ~3 x 2 y)` because `x` is written before the `+` operator).

The AST root node can only be annotated with the form `<column, line, filename>` as it has no parent node
that column and line could refer to.

Note that numeric literals in NIF have to start with `+` or `-` and thus cannot cause ambiguity with line
information.

Since the information includes both lines and columns it can easily take up 10-20% of the file size.
Therefore a mere digit starts a line information and not a numeric literal. Numeric literals are not
nearly as frequent in practice.


Comments
--------

Grammar:

```
Comment ::= '#' EscapedData '#'
```

Every node can be prefixed with `#` to add a comment to the particular node. The comment
also has to end with a `#`.

For example:

```nif
# This performs an add.#(add x y)
```

Note how the comment ends at `#`. This is not ambiguous as any control character within a
comment would have to be escaped via `\xx`.

If a node is annotated both with line information and a comment the line information has
to come first.


Modules
-------

A complete NIF module consists of a list of directives followed by other CompoundNodes.
Typically, there is a single root node of kind `stmts`.

Formally a module is simply a non-empty list of `CompoundNode`:

```
NifModule ::= CompoundNode+
```

### Module suffixes

A module is a file on disk. The filename typically has the structure `<suffix>.<pipeline-step>.nif`
where `<pipeline-step>` typically is short form  like `p` (for "parsed" file) or `s` (for "semchecked" file).

For example in `sysma2dyk.s.nif`:

- `sysma2dyk` is the module's unique name. This is also the used suffix for global symbols that end in a dot: `foo.0.` is expanded to `foo.0.sysma2dyk` by a NIF parser.
- `s` is the "pipeline-step". Here `s` indicates that the file was checked for semantics, in other words identifiers have been looked up and translated to symbols and that type-checking has been performed. The pipeline-step is optional.
- `nif` is the file extension. Every NIF file should have this file extension.


Directives
----------

A directive looks like an atom like a string literal, an integer literal or a symbol.

Directives must be at the start of the file, before the module's AST. Directives that are unknown
to a parser should be ignored.


Indexes
-------

If the first directive is an integer literal it specifies a byte offset into the current file
that describes where to find the index structure. The index always uses the tag `index` and `kv`
pairs. For example:


```
+1234
(stmts
  (proc :foo.0.suffix ...)
  (var :bar.0.suffix ...)
)
(index
  (kv foo.0.suffix +12)
  (kv bar.0.suffix +23)
)
```

The offsets are **diff**-based, to keep the resulting numbers shorter! The first entry is relative to +0
and then the absolute value of entry N is the value of N-1 plus the current value. In other words for the
above example the offset of `foo.0.suffix` is `12` and the offset of `bar.0.suffix` is `12 + 23 == 35`.

Only symbols that have at least two dots have entries in the index. The idea is that only these symbols are top level entries that are interesting to jump to from outside the current module.

Indexes are optional and can be recomputed. The recomputation can also be used for validation. The implementation ships with such a tool called `nifindex`.


Unused name hints
-----------------

If the second directive is a symbol it indicates the first available symbol for a code generator
that does not occur in the current file. For example `tmp.14` would tell the NIF processor that
the names `tmp.14`, `tmp.15`, `tmp.16`... do not occur in the NIF file and can be used for non-ambiguous
temporary local names.



NIF trees as identifiers
------------------------

In many cases it is useful to turn a NIF tree into a canonical string representation that also
forms a valid identifier (for code generation or otherwise). The following encoding scheme
accomplishes this task:

1. Line information and comments are ignored.
2. The unary `+` for numbers is removed.
3. The substring of trailing `)` is removed as there is nothing interesting about `))))`.
4. Whitespace is canonicalized to a single space.
5. The space after `)` and before `(` is removed.


`(` is turned into `A`.
`)` is turned into `Z`.
A space that separates the children of a compound node is turned into `S`.

The empty node (`.`) is encoded as `E`.

Note that `.` within a symbol is **not** escaped!

The N-th (where N > 1) occurrence of a symbol or identifier
is encoded as `R<x>` where `x` refers to the symbol or identifier that has already
been emitted. But only if `R<x>` is still shorter than the symbol/identifier.

For example:

`(tag abcdef . abcdef)` is encoded as `AtagSabcdefSESR0`.

Characters like `A` and `Z` that are used in the encoding must be
escaped should they occur. The encoding is `X<xx>` where `xx` is the
hexadecimal value of the character's byte value. Other characters that are not valid
identifiers such as the space character or a newline are encoded as `X<xx>` too.

In summary:

| Letter | Used for |
| --------- | -------------- |
| `A`    | begin of a compound node `(` |
| `Z`   | end of a compound node `)`  |
| `E`   | the empty node |
| `S`   | space; separator between a node's children |
| `O`   | encodes the colon in a SymbolDef |
| `U`   | encodes the `"` that is used to delimit string literals |
| `X` | used to escape the letters used in this encoding and in general for characters that should not be used in an identifier |
| `R` | reference to an identifier or a symbol that has already occurred |
| `K` | reference to a node kind that has already occurred |

For example:

`(array (range +0 +9) (array (range +0 +4) (i +8))))`

Becomes:

`(array(range 0 9)(array(range 0 4)(i 8`

Which then is turned into:

`AarrayArangeS0S9ZAK0AK1S0S4ZAiS8`
