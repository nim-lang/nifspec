NIF data format
===============

The NIF data format is a text based file format designed for compiler frontend/backend
communication, configuration files, data exchange between different programming languages
and similar use cases. The design works on language agnostic ASTs and is so extensible
that many programming languages work well with it.

A NIF file corresponds to a "module" in the source language. The module is stored as an AST.
The AST consists of "atoms" and "compound nodes".


Extensibility is primarily achieved by using two different namespaces. One namespace
is used for "node kinds" / "tags" and a different one for source level identifiers. This does
away with the notion of a fixed set of "keywords". In NIF new "keywords" ("tags")
can be introduced without breaking any code.

There is also an optional **index structure** that maps symbols to offsets within a NIF file.
This makes NIF a hybrid between a binary and a text file.


Version 2027
------------

This document describes the **2027** version of NIF. Differences to the 2026 version:

- The `(.nif26)` directive was changed to `(.nif27)`.
- Line information moved from a node *prefix* to a node *suffix*. It is introduced
  by `@`, written immediately after an atom or after a tag name with **no** intervening
  whitespace. Examples: `"abc"@5,3`, `123@5,3,foo.nim`, `(tag@5,3 child child)`.
- A leading negative `~` may introduce a line-info suffix on its own without `@`,
  saving one byte for the very common infix case: `~3` is shorthand for `@~3`.
- Line-information numbers are encoded in **base62** (`0-9A-Za-z`), not decimal.
  Filenames are unaffected.
- Comments also became suffixes. They follow the line information (if any) with no
  whitespace in between. The general suffix shape is `<atom-or-tag>@<info>#<comment>#`,
  where both parts are optional.
- Numbers no longer require a leading `+`. Bare `12` is the integer twelve. Negative
  numbers still require a leading `-` (`-12`). The `~` sign is reserved for line-info
  diffs.
- `@` was added to the set of control characters.
- New escape shortcuts `\n`, `\t`, `\r`, `\|`, `\^` are recognized in addition to
  the canonical `\xx` form. The `\|` shortcut for a literal backslash is preferred
  over `\5C`; importantly, `\\` is **not** an escape sequence in NIF.


Example NIF module
------------------

In order to get a feeling for how a NIF file can look, here is a complete example:

```nif
(stmts
(imp@2,5,sysio.nim (type :File (object . .)))
(imp (proc :write.1.sys . (pragmas varargs) (params (param f File)) .))
(call write.1.sys "Hello World!\0A")
)
```

<div style="page-break-after: always;"></div>

Encoding
--------

A NIF file is stored as a sequence of bytes ("octets"). No Unicode validation steps are
required; parsers operate on raw bytes. While UTF-8 is commonly used, it is not mandated.
Importantly, any byte with value >= 128 may be used directly in identifiers and
literals without escaping — the set of control characters that must be escaped is
restricted to ASCII characters only (see "Control characters" below).


Whitespace
----------

Whitespace is used to separate tokens from each other but apart from that carries no
meaning and a NIF parser is supposed to ignore whitespace. Editors and other tools
can format and layout NIF code to be pleasing to look at.

Whitespace is the set `{' ', '\t', '\n', '\r'}`.


Control characters
------------------

NIF uses a small set of ASCII control characters (for example `(`, `)` and `~`) to describe
AST structure. These characters **must not** occur literally in string literals, char
literals, identifiers or symbols because a parser relies on them to find matching delimiters
and suffix introducers. They may, however, be represented inside literals or comments or
identifiers or symbols when escaped using the hex escape `\xx` (see "Escape sequences").

The control characters are the following ASCII bytes:

```
( )  [ ]  { }  ~  #  '  "  \  :  @
```

Escape sequences
----------------

Grammar:

```
HexChar    ::= [0-9A-F]
ShortChar  ::= 'n' | 't' | 'r' | '|' | '^'
Escape     ::= '\' (HexChar HexChar | ShortChar)
```

String and character literals, comments, identifiers and symbols support escape sequences
via backslashes. The primary form is `\xx` where `xx` is two upper-case hexadecimal digits
that spell the ASCII value of the encoded byte. For example, a binary zero is `\00`.

In addition, the following two-byte shortcuts are accepted:

| Shortcut | Decoded byte | Meaning             |
|----------|--------------|---------------------|
| `\n`     | `\x0A`       | newline             |
| `\t`     | `\x09`       | tab                 |
| `\r`     | `\x0D`       | carriage return     |
| `\|`     | `\x5C`       | a literal backslash |
| `\^`     | `\x22`       | a literal `"`       |

*Caution*: The commonly used `"\\"` in other languages that escapes the backslash itself
is **not** supported. Use `\|` or `\5C`.

*Rationale*: Each shortcut body byte (`n`, `t`, `r`, `|`, `^`) is neither a hexadecimal
digit nor a NIF control character, so a single byte of look-ahead after `\` unambiguously
selects between "two-hex-digit form" and "one-byte shortcut form". This keeps simple
metacharacter scanners (regex-based or hand-written) easy to write: an escape sequence is
always `\` plus *exactly* one or two more bytes, decided by inspecting the byte after `\`
in isolation.


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
IdentStart ::= <ascii_letter> | '_' | NonAscii | Escape
IdentChar  ::= IdentStart | [_0-9]
Identifier ::= IdentStart IdentChar*
NonAscii   ::= byte value >= 128
```

Identifiers cannot start with a digit, so a number and an identifier are never ambiguous.
Identifiers have no real meaning; in particular it **cannot** be assumed that two identifiers
with the same sequence of bytes (for example `abc`) refer to the same entity.

Bytes with value >= 128 may appear directly in identifiers and do not require escaping.
Identifiers that contain characters that are neither ASCII letters nor digits nor bytes >= 128
must be escaped using backslashes `\xx`, the same escape form used for string and char literals.


### Symbols

A "symbol" is a name that refers to an entity unambiguously. A symbol must adhere to the grammar:

```
Symbol    ::= IdentStart IdentChar* '.' (IdentChar | '.')*
SymbolDef ::= ':' Symbol
```


Roughly speaking, that is a "word" that must contain a dot but cannot start with a dot.

For example, the 2nd proc named `foo` in a Nim module `m` would typically become `foo.2.m` in NIF.

Symbols that contain characters that are neither letters nor digits must be escaped via
backslashes, `\xx` much like it is used in string and character literals.

A `SymbolDef` is a symbol annotated with a leading ':'. It indicates that the parent node is
the node introducing this symbol. Thus a tool can implement a feature like "goto definition"
in a language agnostic way without having to know which node kinds introduce new symbols.

There are two kinds of symbols: local and global symbols. A local symbol is of the form `<ident>.<disamb>` where
`disamb` is a list of digits. For example, a name like `foo.0` where the `0` implies it is the first symbol
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
Digit             ::= [0-9]
FloatingPointPart ::= ('.' Digit+ ('E' ('+' | '-')? Digit+)? ) | 'E' ('+' | '-')? Digit+
Number            ::= '-'? Digit+ (FloatingPointPart | 'u')?
```

Numbers use decimal notation only; for example, Nim's `0xff` becomes `255`.

Positive numbers do **not** require a leading `+`; bare `12` is the integer twelve.
Negative numbers carry a leading `-` (e.g. `-12`).

Unsigned numbers always have a `u` suffix. Floating point numbers must contain a dot or `E`.
Every other number is interpreted as a signed integer.

Because identifiers cannot start with a digit and line information is now introduced by
`@` or a leading `~` (see "Line information"), a leading digit unambiguously starts a number.


### Char literals

Grammar:

```
VisibleChar ::= ASCII value >= 32 but not a control character | byte value >= 128
CharLiteral ::= '\'' (VisibleChar | Escape) '\''
```

Char literals are enclosed in single quotes. Escapes are as defined under **Escape sequences**
(the canonical `\xx` form and the same `\n`, `\t`, `\r`, `\|`, `\^` shortcuts).


### String literals


Grammar:

```
EscapedData   ::= (VisibleChar | Escape | Whitespace)*
StringLiteral ::= '"' EscapedData '"'
```

String literals are enclosed in double quotes. Escapes are as defined under **Escape sequences**
(`\xx` plus the `\n`, `\t`, `\r`, `\|`, `\^` shortcuts). Note again that `\\` is **not** an escape.
Whitespace, even including newlines, can be part of the string literal without having to
escape it.

For example, the following single string literal contains an escaped byte plus an actual newline:

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
B62Digit ::= [0-9A-Za-z]
LineDiff ::= B62Digit* | '~' B62Digit+
LineInfo ::= '@' LineDiff (',' LineDiff (',' EscapedData)?)?
           | '~' B62Digit+ (',' LineDiff (',' EscapedData)?)?
Comment  ::= '#' EscapedData '#'
Suffix   ::= LineInfo? Comment?

Atom     ::= ( Empty | Identifier | Symbol | SymbolDef | Number
             | CharLiteral | StringLiteral ) Suffix
NodeKind ::= Identifier
TagHead  ::= NodeKind Suffix

Node     ::= Atom | CompoundNode
CompoundNode ::= '(' TagHead Node* ')'
```

In `LineInfo`, the second alternative is the **leading-`~` shorthand**: when the first column
diff is negative, `@` may be omitted because `~` already begins the suffix (see **Line information**).

The general syntax for a compound node is `(nodekind child1 child2 child3)`. `nodekind`
is also called the "tag". An optional line-information and/or comment suffix may appear
*directly* after the tag name (no whitespace).

That means NIF is a Lisp with some extensions:

- The ability to annotate (line, column, filename) information for any atom or compound node.
- The ability to annotate any atom or compound node with a comment.

Unlike in Lisp a function `call` is not implied so what is usually just `(f a b c)` in Lisp
becomes `(call f a b c)` in NIF. The first item in a list (`call` in the example) is called the "tag".
There are many different tags and the set of tags is extensible.

However, usually at least the following tags exist and ensure a minimum of compatibility between
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
| `expr`    | A list of statements ending in an expression. |
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


Every tag belongs to a "language". A language is a fixed predefined set of tags. The `(.lang)` directive can be used to nest one language in another:

```
(.nif27)
(.lang "html")
(html
  (a (kv (href) "https://some.url"))
  (.lang "css"
    (style (kv (color) "red") (kv (size) "12px"))
  )
)
```

In the context of compilers this can model inline assembler (assuming the assembler uses NIF syntax too) or Nim's `emit` pragma.


Line information
----------------

Grammar:

```
B62Digit ::= [0-9A-Za-z]
LineDiff ::= B62Digit* | '~' B62Digit+
LineInfo ::= '@' LineDiff (',' LineDiff (',' EscapedData)?)?
           | '~' B62Digit+ (',' LineDiff (',' EscapedData)?)?
```

Any atom and any tag name can be followed (with **no** intervening whitespace) by line
information of the form `@<col-diff>` or `@<col-diff>,<line-diff>` or
`@<col-diff>,<line-diff>,<filename>`. Examples:

```
123@5
"hello"@5,3
foo@5,3,foo.nim
(call@5,3 a b c)
```

The `diff` portions are values relative to the parent node. For example `5` means that the
node is at the same position as the parent node except that its column is `+5` characters.
Negative numbers carry a leading `~` (e.g. `~3` for "column - 3"). Negative leading column
diffs are typical for operands that appear **before** the syntactic construct named by the
parent tag—for instance `x + y` might become `(infix add x~3 y@2)` (equivalently
`x@~3`): each operand carries its column diff relative to the `infix` node.

**Diff numbers are written in base 62** using the digits `0-9A-Za-z`, where `A` = 10,
`Z` = 35, `a` = 36, `z` = 61. This shrinks line-information bytes by roughly 45% over
decimal. Filenames are not affected — they are arbitrary `EscapedData`.

**Shorthand for negative leading diffs**: when the very first diff is negative, the leading
`@` may be omitted, because `~` already marks the start of a line-information suffix.
So `atom~3` is shorthand for `atom@~3`. The shorthand only applies to the first segment;
to write a line info whose first diff is positive one must use `@`.

The AST root node can only be annotated with the form `<col,line,filename>` as it has no
parent node that column and line could refer to. Place this annotation directly after the
root tag name: `(stmts@1,1,foo.nim …)`.

Since line information includes both lines and columns it can easily take up 10-20% of the
file size for compiler-emitted ASTs. Therefore base62 is used and the leading `@` can be
omitted when the first diff is already negative.


Comments
--------

Grammar:

```
Comment ::= '#' EscapedData '#'
```

Any atom and any tag name can be followed (with **no** intervening whitespace) by a comment.
If both a line-information suffix and a comment are present, line information must come first
and the comment immediately follows it (still no whitespace).

Examples:

```nif
(add#performs an addition# x y)
123#answer#
foo@5,3#why this token is here#
```

The comment ends at the next `#`. This is not ambiguous because any control character within
a comment would have to be escaped via `\xx`.


Modules
-------

A complete NIF module consists of a list of directives followed by other CompoundNodes.
Typically, there is a single root node of kind `stmts`.

Formally a module is simply a non-empty list of `Node`:

```
NifModule ::= Node+
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

A directive looks like `(.directive ...)`. This is not ambiguous because a node kind cannot
start with a dot. The existing directives are:

- `.nif<version>`: Should be `.nif27`.
- `.indexat`: Defines the byte offset at which the index structure starts.
- `.index`: Defines the index structure for random-access of symbols.
- `.unusedname`: Defines the first available symbol for a code generator that does not occur in the current file.
- `.vendor`: Defines the vendor of the NIF file. For example `(.vendor "Nifler")`.
- `.platform`: Defines the platform of the NIF file. For example `(.platform "x86_64")`.
- `.config`: Defines the configuration of the NIF file. For example `(.config "release")`.
- `.lang`: Defines the language the used tags belong to.
- `.dialect`: Defines the dialect of the NIF file. For example `(.dialect "nim-parsed")`. Obsolete: Use `.lang` instead.

All directives except `.index` and `.lang` must be at the start of the file, before the module's AST. `.index` can also be at the end of the file. `.lang` can be used anywhere to influence the meaning of the wrapped tags.

Directives that are unknown or unsupported by a parser should be ignored.


### Version directive

The version directive looks like `(.nif<version>)`.

For example:

```nif
(.nif27)
```

There must be no whitespace before the version directive so that it also functions as a
"magic cookie" for tools that use these to determine file types.


Conformance
-----------

A conformant NIF parser should:

- Accept the module as a sequence of bytes and tolerate non-UTF-8 content.
- Allow bytes with value >= 128 in identifiers and string/char literals without requiring escapes.
- Support the `\xx` escape form for representing arbitrary byte values (including escaping control characters and `\` as `\5C`).
- Parse base62 line-information diffs and the leading-`~` shortcut form.
- Parse and ignore unknown directives and tolerate optional indexes.
- Expand trailing-dot global symbols (e.g., `foo.0.`) to include the module suffix when required.


Indexes
-------

The `.indexat` directive announces the existence of an index structure in the NIF file.
The index itself always uses the directive `.index` and contains pairs using the tags `x` (for "eXported") or `h` (for "Hidden"). It must be at the end of the NIF file. For example:


```
(.nif27)
(.indexat 1234)
(stmts
  (proc :foo.0.suffix ...)
  (var :bar.0.suffix ...)
)
(.index
  (x foo.0.suffix 12)
  (x bar.0.suffix 23)
)
```

The offsets are **diff**-based, to keep the resulting numbers shorter. The first entry is relative to +0
and then the absolute value of entry N is the value of N-1 plus the current entry. In other words, for the
above example the offset of `foo.0.suffix` is `12` and the offset of `bar.0.suffix` is `12 + 23 == 35`.

Only symbols that have at least two dots have entries in the index. The idea is that only these symbols are top level entries that are interesting to jump to from outside the current module.

Indexes are optional and can be recomputed. The recomputation can also be used for validation. The implementation ships with such a tool called `nifindex`.

**Implementation note**: The `.indexat` offset can be patched in place, without reallocations, by exploiting the fact that whitespace is a separator and can be of variable length. In other words, emit `(.indexat      )` with enough spaces between the directive name and the closing paren to accommodate the final offset, and overwrite those spaces with the actual offset (e.g., `1234`) once it is known.


Unused name hints
-----------------

The `.unusedname` directive looks like `(.unusedname <symbol>)`.
For example `(.unusedname tmp.14)` would tell the NIF processor that
the names `tmp.14`, `tmp.15`, `tmp.16`... do not occur in the NIF file and can be used for non-ambiguous temporary local names.


NIF trees as identifiers
------------------------

In many cases it is useful to turn a NIF tree into a canonical string representation that also
forms a valid identifier (for code generation or otherwise). The following encoding scheme
accomplishes this task:

1. Line information and comments (the `Suffix` parts) are ignored.
2. The substring of trailing `)` is removed as there is nothing interesting about `))))`.
3. Whitespace is canonicalized to a single space.
4. The space after `)` and before `(` is removed.


`(` is turned into `A`.
`)` is turned into `Z`.
A space that separates the children of a compound node is turned into `_`.

The empty node (`.`) is encoded as `E`.

Note that `.` within a symbol is **not** escaped!

The N-th (where N > 1) occurrence of a symbol or identifier
is encoded as `R<x>` where `x` refers to the symbol or identifier that has already
been emitted. But only if `R<x>` is still shorter than the symbol/identifier.

For example:

`(tag abcdef . abcdef)` is encoded as `Atag_abcdef_E_R0`.

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
| `_`   | space; separator between a node's children |
| `O`   | encodes the colon in a SymbolDef |
| `U`   | encodes the `"` that is used to delimit string literals |
| `X` | used to escape the letters used in this encoding and in general for characters that should not be used in an identifier |
| `R` | reference to an identifier or a symbol that has already occurred |
| `K` | reference to a node kind that has already occurred |

For example:

`(array (range 0 9) (array (range 0 4) (i 8))))`

Becomes:

`(array(range 0 9)(array(range 0 4)(i 8`

Which then is turned into:

`AarrayArange_0_9ZAK0AK1_0_4ZAi_8`


Paths/URLs as NIF trees
-----------------------

The `a` tag is used for absolute paths, and `p` for relative paths. A relative path is followed by a number
indicating the level of required parent directory navigations: 0 is `./`, 1 is `../`, 2 is `../../` and
so on.

The individual path components become NIF identifiers. They are subject to the general escape mechanism via the backslash-hex-hex notation. A file extension is represented as the NIF empty node dot followed by the pure extension
as a NIF identifier. This means `file.txt` becomes `file . txt` in NIF.

Likewise a URL can be encoded as `(protocol url parts)`.

These NIF trees can then turned into identifiers via the "NIF trees as identifiers" algorithm.

Examples:

| path |  NIF representation | as identifier |
| ---- | ------------------- | --------------|
| `/usr/bin/foo` | `(a usr bin foo)` | `Aa_usr_bin_foo` |
| `foobar` | `(p 0 foobar)` | `Ap_0_foobar` |
| `./foobar` | `(p 0 foobar)` | `Ap_0_foobar` |
| `file.txt` | `(p 0 file . txt)` | `Ap_0_file_E_txt` |
| `../../foo/bar.txt` | `(p 2 foo bar . txt)` | `Ap_2_foo_bar_E_txt` |
| `https://github.com/nifspec` | `(https github . com nifspec)` | `Ahttps_github_E_com_nifspec` |
