# nif

The NIF data format is a text based file format designed for compiler frontend/backend
communication or communication between different programming languages. The design is
heavily tied to Nim's requirements. However, the design works on language agnostic ASTs
and is so extensible that other programming languages work well with it too.

A NIF file corresponds to a "module" in the source language. The module is stored as an AST.
The AST consists of "atoms" and "compound nodes".


Highlights
----------

- Compact, readable S-expression format with optional diffed line information and an
  optional byte-index for fast random access.
- Simple parsing/serialization: a single `\xx` escape form and an ASCII-based control
  character set make implementations straightforward.
- Simple module system based on unique filenames: Distribute program modules as `.nif` files in a common directory; module identities are derived from filenames (including the optional pipeline-step suffix) and cross-module references use the symbol naming convention (e.g., `foo.0.module` can always be found in `module.nif`).
- Extensible and flexible: NIF could have been the foundation for C++'s module system (and it would be better at it than C++'s obscure custom binary formats).
- Used in production by the Nimony compiler, demonstrating practical applicability.


Universal Data Format via Language Contexts
--------------------------------------------

NIF's `.lang` directive allows different semantic domains to coexist in a single unified syntax.
Rather than maintaining separate formats (JSON for data, HTML for markup, HTTP for protocols, SQL for databases, etc.),
each domain can be represented as a different language context:

```nif
(.lang "html"
  (div (class "container")
    (p "Welcome")
    (.lang "css"
      (style
        (kv (background-color) "blue")))))

(.lang "json"
  (object
    (kv (name) "Alice")
    (kv (created) (isodate "2026-01-19"))))

(.lang "sql"
  (create-table users
    (column (id) (i +64))
    (column (name) string)))
```

With a single NIF parser, the same infrastructure provides:
- Type-safe representation (via extensible tags)
- Origin tracking (line/column/filename on every node)
- Unified module system (imports, namespacing)
- Composable embedding (languages can nest)
- Optional indexing for random access

This reduces implementation burden: instead of multiple parsers, type systems, and serialization formats,
different semantic domains use a shared structure with scoped semantics.


Documentation
-------------

Docs are here: https://github.com/nim-lang/nifspec/blob/master/doc/nif-spec.md


Design goals
------------

- Simple to parse.
- Simple to generate.
- Text representation that tends to produce small code.
- Extensible and easily backwards compatible with earlier versions.
- Lossless conversion back to the source code is possible.
- Can be as high level or as low level as required because statements, expressions
  and declarations can be nested arbitrarily.
- Lots of search&replace like operations can be performed reliably with pure text
  manipulation. No parsing step is required for these.
- Readable and writable for humans. However, it is not optimized for this task.

Extensibility is primarily achieved by using two different namespaces. One namespace
is used for "node kinds" and a different one for source level identifiers. This does
away with the notion of a fixed set of "keywords". In NIF new
"keywords" ("node kinds" / "tags") can be introduced without breaking any code.

Why yet another data format?
----------------------------

Other, comparable formats (LLVM Bitcode or text format, JVM bytecode, .NET bytecode, wasm)
have one or more of the following flaws:

- More complex to parse.
- Too low level representation that loses structured control flow.
- Too low level representation that implies the creation of temporary variables that are
  not in the original source code file.
- Binary formats that make it harder to create tools for.
- Less flexible.
- Cannot express Nim code well.
- Produces bigger files.
