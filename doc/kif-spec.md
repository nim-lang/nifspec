KIF data format
===============

KIF is a *k*ompression method for NIF that is particularly easy to implement
and that allows for all algorithms to be run directly on the compressed format
without a full decompression step. The decompression can happen on-the-fly.

A KIF file can be transformed lossless into NIF and vice versa. A KIF file is a binary
file. The format is:

```
header consisting of a 32 bit cookie: Always the value 0x00CEFCEF (in big endian)
offset to token table: 32bit unsigned integer
length of list of token indexes: varint
list of token indexes: Every index is a "varint" (as implemented by Nim's stdlib `varint` module)
table of unique tokens: Every token is prefixed by its length as a varint. Tokens are decoded completely (no escape sequences are used) and can contain binary zeros. Numbers, however remain in their ASCII format.
```

Tokens
------

The tokens in the compressed token stream do not directly correspond to NIF tokens. Line information like `0,1,file.nim` is considered to be the separate tokens: `(%`, `0`, `1`, `file.nim`, `)`. (This exploits the fact that `%` is not a valid NIF tag name.)

Likewise a series of three closing `)))` is considered to be a single token as is a series
of two closing `))`. This improves the compression rate somewhat.

