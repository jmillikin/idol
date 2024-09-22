# Idol Syntax

## General syntax

An Idol schema is written as UTF-8 text. The preferred file extension is `.idol`.

```idol
# A simple example Idol schema.

namespace "acme.example/hello-world"

const FRIENDLY_GREETING: text = "Hello, world!"

message Hello {
	greeting@1: text
}
```

The order of productions within a schema is: namespace, imports, exports, options, and declarations. The namespace is required; other productions are optional.

## Permitted characters

All Unicode scalar values are permitted except for non-space ASCII control characters. Forbidden characters are:
- `%x00-08` (ASCII `NUL` to `BS`)
- `%x0B` (ASCII `VT`)
- `%x0C` (ASCII `FF`)
- `%x0E-1F` (ASCII `SO` to `US`)
- `%x7F` (ASCII `DEL`)

Space characters `%x09` (horizontal tab), `%x20` (space), and `%xA0` (non-breaking space) are permitted between most tokens (the exceptions are noted).

Newlines `%x0A` (line feed) and `%x0D.0A` (`\r\n`, carriage return + line feed) are permitted between some tokens. The scalar value `%x0D` (carriage return) is only permitted as part of a `%x0D.0A` sequence.

```abnf
SP = %x09 / %x20 / %xA0

NL = %x0A / %x0D.0A
```

## Comments

Comments start with `%x23` (`#`), and continue until the next newline or end-of-file. Comments starting with `%x23.23` are documentation comments ("doc comments"), which may be extracted by documentation generators or included in generated files.

```idol
# This is a comment.
## This is a doc comment.
```

The Idol schema parser reports comment text as-is, without removing the `#` prefix or trimming spaces.

```abnf
COMMENT = %x23 *(%x09 / %x20-7E / %x80-D7FF / %xE000-10FFFF)
```

### Identifiers

Identifiers must start with `%x61-7A / %x41-5A` (`[a-zA-Z]`), contain `%x61-7A / %x41-5A / %x30-39 / %x5F` (`[a-zA-Z0-9_]`), and end with `%x61-7A / %x41-5A / %x30-39` (`[a-zA-Z0-9]`).

In other words, they must start with an ASCII letter, contain ASCII alphanumeric characters plus underscores, and must not end with an underscore.

```abnf
IDENT = ALPHA *( ALPHA-NUM / %x5F ALPHA-NUM )

ALPHA = %x61-7A / %x41-5A
ALPHA-NUM = ALPHA / %x30-39
```

There are no keywords or reserved identifiers in the Idol schema syntax. The following schema is permitted (though not recommended):

```idol
# Define a structure type named `struct`.
struct struct {
	const: bool
}

# Define a message type named `message`, containing a field named `struct`,
# which is of the type `struct` defined above.
message message {
	struct@1: struct
}
```

Source code generators may transform or reject identifiers that don't conform to the rules of their target languages, for example by capitalizing type names or adding underscores to names that are language keywords.

The Idol schema compiler may reject schemas in which shadowing of a built-in type renders the meaning ambiguous to a human reader:

```idol
# Permitted
enum bool: u8 {
	true = 0
	false = 1
}

# Error: Built-in type `bool` shadowed by user-defined type.
const A: bool = .true
```

### Built-in types

| Type name                 | Description                                              |
| ------------------------- | -------------------------------------------------------- |
| `bool`                    | Booleans (`.true` or `.false`)                           |
| `u8`, `u16`, `u32`, `u64` | Unsigned integers                                        |
| `i8`, `i16`, `i32`, `i64` | Signed integers                                          |
| `f32`, `f64`              | IEEE-754 floating-point                                  |
| `text`                    | UTF-8 encoded text without `NUL`                         |
| `asciz`                   | Arbitrary bytes without `NUL`                            |
| `handle`                  | Handle to ancillary data passed via an IPC/RPC protocol. |

Array types are defined by appending `[]` (for a dynamically-sized array) or `[<int literal>]` for a fixed-size array. For example, a field containing arbitrary bytes might be typed as `u8[]`, and a field containing an XYZ coordiate might be `f32[3]`.

Handles are unsigned 32-bit integers that represent an index into an array of values propagated separately from Idol messages, within the context of an IPC/RPC protocol.
* A Unix process would use a `handle` to identify the index of a file descriptor passed via `SCM_RIGHTS`.
* A distributed system performing network calls would use a `handle` to identify a cryptographic credential passed via an HTTP header.

Handles are not directly written to encoded Idol data structures, but are obtained from a list passed to the message decoder.

### Integer literals

Integer literals have an optional base prefix, and may be negative:
* `0`, `42`, and `-42` are integer literals. The digit `0` is only permitted by itself, not as a leading zero (`00` and `01` are invalid).
* `0b101010` is an integer literal in base 2 (**b**inary), and may contain leading zeroes.
* `0o52` is an integer literal in base 8 (**o**ctal), and may contain leading zeroes.
* `0d42` is an integer literal in base 10 (**d**ecimal), and may contain leading zeroes.
* `0x2a` or `0x2A` are integer literals in base 16 (he**x**), and may contain leading zeroes.

```abnf
INT-LIT =  %x30
INT-LIT =/ [%x2D] %x31-39 *(%x30-39)

BIN-INT-LIT =  %x30.62 1*(%x30-31)
BIN-INT-LIT =/ %x2D %x30.62 *(%x30) %x31 *(%x30-31)

OCT-INT-LIT = %x30.6F 1*(%x30-37)
OCT-INT-LIT =/ %x2D %x30.6F *(%x30) %x31-37 *(%x30-37)

DEC-INT-LIT = %x30.64 1*(%x30-39)
DEC-INT-LIT =/ %x2D %x30.64 *(%x30) %x31-39 *(%x30-39)

HEX-INT-LIT = %x30.78 1*(%x30-39 / %x41-46 / %x61-66)
HEX-INT-LIT =/ %x2D %x30.78 *(%x30) (%x31-39 / %x41-46 / %x61-66) *(%x30-39 / %x41-46 / %x61-66)

ANY-INT-LIT = INT-LIT / IN-INT-LIT / OCT-INT-LIT / DEC-INT-LIT / HEX-INT-LIT
```

Some locations in the syntax that accept integer literals forbid base prefixes,
typically when the use of a non-decimal base would be unexpected by the reader.

### Text literals

Text literals are quoted with `%x22` (double quote), and may contain Unicode scalar values. The following escape sequences are allowed:
* `\\` decodes to `%x5C` (backslash)
* `\"` decodes to `%x22`
* `\n` decodes to `%x0A`
* `\xNN` decodes to the Unicode scalar value `U+00NN`, with exactly two hex digits allowed.
* `\u{NNNN}` decodes to the Unicode scalar value `U+NNNN`, with 1 to 6 hex digits allowed.

```abnf
TEXT-LIT = %x22 *(
             %x09 / %x20 / %x21 /
             %x23-5B /
             %x5D-7E /
             %x80-D7FF / %xE000-10FFFF
             (%x5C (
                %x22 /
                %x5C /
                %x6E /
                %x78 2(%x31-39 / %x41-46 / %x61-66) /
                %x75.7B 1*6(%x31-39 / %x41-46 / %x61-66) %x7D
            ))
           ) %x22
```

As the source is in UTF-8, non-ASCII characters may be used freely in text literals. Note that the `text` primitive type forbids the scalar value `U+0000`.

### Bytes literals

Bytes literals are quoted with `%x22` (double quote), and may contain scalar values in the range `%x01-FF`. The following escape sequences are allowed:
* `\\` decodes to `%x5C` (backslash)
* `\"` decodes to `%x22`
* `\n` decodes to `%x0A`
* `\xNN` decodes to the byte `%xNN`, with exactly two hex digits allowed.
* `\u{NNNN}` decodes to the UTF-8 encoding of the Unicode scalar value `U+NNNN`, with 1 to 6 hex digits allowed.

As the source is in UTF-8, non-ASCII characters in bytes literals will be represented as their UTF-8 encoding. Note that the `asciz` primitive type forbids the byte `0x00`.

```abnf
BYTES-LIT = %x22 *(
             %x09 / %x20 / %x21 /
             %x23-5B /
             %x5D-7E /
             %x80-D7FF / %xE000-10FFFF
             (%x5C (
                %x22 /
                %x5C /
                %x6E /
                %x78 2(%x31-39 / %x41-46 / %x61-66) /
                %x75.7B 1*6(%x31-39 / %x41-46 / %x61-66) %x7D
            ))
           ) %x22
```

Syntatically, text and bytes literals are the same. They are distinguished within the schema compiler's type checker at the location of use.

### Boolean literals

There is no dedicated literal syntax for boolean values. Instead, they use the same syntax as enum item references.

```idol
const A: bool = .false
const B: bool = .true
```

## Header

The productions that can only occur before declarations are collectively called the "header".

### Namespace

Each schema must have a namespace, which provides scoping within the set of schemas being compiled together. The namespace is declared with the `namespace` production.

```idol
namespace "acme.example/hello-world"
```

The namespace is a text literal, which may contain any Unicode scalar value except for non-space ASCII control characters. The set of excluded characters is the same as for Idol schema source code.

It is recommended, but not required, that namespaces use some sort of hierarchical convention rooted under a DNS domain name controlled by the authors of the schema.

The prefix `"idol/"` is reserved for use by the Idol schema compiler.

```abnf
namespace = %s"namespace" *SP TEXT-LIT
```

### Imports

Declarations from other schema files may be imported with the `import` production. Imported declarations may be used like any 

```idol
namespace "acme.example/hello-world"

import "acme.example/i10n" { Language }

message Hello {
	greeting@1: text

	@{optional}
	language@2: Language
}
```

Instead of importing specific declarations, the entire namespace can be given a local name and used to qualify type names.

A type name such as `i10n.Language` may not contain spaces before or after the dot.

```idol
namespace "acme.example/hello-world"

import "acme.example/i10n" as i10n

message Hello {
	greeting@1: text

	@{optional}
	language@2: i10n.Language
}
```

Importing declarations from the same namespace is permitted, and is useful for splitting up a pre-existing schema file into separate smaller files.

```idol
namespace "acme.example/hello-world"

import "acme.example/hello-world" { Greeting }

message SayHello {
	greeting@1: Greeting
}
```

```abnf
import = %s"import" *SP TEXT-LIT *SP (imports / import-as)

imports = %x7B
          *(SP / NL / COMMENT NL)
          *(IDENT *(SP / NL / COMMENT NL))
          %x7D

import-as = %s"as" 1*SP IDENT
```

### Exports

Declarations defined in the current file are always exported. Use the `export` production to re-export declarations imported from other namespace.

```idol
namespace "acme.example/i10n/v2"

import "acme.example/i10n" { Language }

export { Language }
```

Dotted type names can also be used:

```idol
namespace "acme.example/i10n/v2"

import "acme.example/i10n" as i10n

export { i10n.Language }
```

Exported declarations can be renamed:

```idol
namespace "acme.example/i10n/v2"

import "acme.example/i10n" as i10n

export i10n.Language as LanguageCode
```

```abnf
export = %s"export" (*SP exports / 1*SP export-as)

exports = %x7B
          *(SP / NL / COMMENT NL)
          *(type-name *(SP / NL / COMMENT NL))
          %x7D

export-as = type-name 1*SP %s"as" 1*SP IDENT

type-name = [ IDENT %x2E ] IDENT
```

### Options

The `options` production can be used to configure how the Idol schema compiler and other tooling processes the current schema file.


```idol
namespace "acme.example/hello-world"

options {
	some_idolc_option = "a special value"
}
```

By default, options are validated against the schema used by the Idol schema compiler itself. Options intended for other tools should specify an options schema:


```idol
namespace "acme.example/hello-world"

import "acme.example/idol-codegen-cxx" {
	CxxCodegenOptions
}

options: CxxCodegenOptions {
	namespace = "acme::hello_idl"
}
```

At present the Idol schema compiler has no options defined, so the only use for top-level `options` is to configure external tools via an imported schema.

```abnf
options = %s"options" [ *SP %x3A *SP type-name ] *SP %x7B
          *(SP / NL / COMMENT NL)
          *(options-option *(SP / NL / COMMENT NL))
          %x7D

options-option = IDENT *SP %x3D *SP option-value

option-value = ANY-INT-LIT / TEXT-LIT / %x2E IDENT
```

## Declarations

Types and values defined within an Idol schema file are known collectively as "declarations". They may be declared in any order, so long as they come after the productions found in the header.

### Constants

Use `const` to declare a constant value of some primitive type.

```idol
# 1 second in milliseconds
const TIMEOUT_MSEC: u32 = 1000

# default TLS server authority
const TLS_AUTHORITY = "api.example.com" 
```

The following types are supported for constant declarations:
* `bool`, `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`
* `f32` and `f64`, which accept integer integers.
* `text`, which accepts text literals.
* `asciz`, which accepts bytes literals. The trailing `NUL` byte is automatically appended.
* `u8[]`, which accepts bytes literals.

```abnf
const = %s"const" *SP IDENT *SP %x3A *SP type-name *SP %x3D *SP const-value

const-value = ANY-INT-LIT / TEXT-LIT / %x2E IDENT / const-name

const-name = [ IDENT %x2E ] IDENT
```

#### Const options

Constants may have options -- see the section on struct fields for details on the syntax.

Supported constant options are:
* `deprecated: bool`: whether the constant is deprecated.

### Enums

Use `enum` to declare an enumeration of named integers. The enum must be derived from an integer type (`u8`, `i8`, `u16`, `i16`, `u32`, `i32`, `u64`, or `i64`).

```idol
enum HttpStatus: u16 {
	OK = 200
	ERR_NOT_FOUND = 404
	ERR_FORBIDDEN = 403
	ERR_INTERNAL_ERROR = 500
}
```

Enums derived from a signed integer may have items assigned negative values.

```idol
enum errno: i8 {
	EPERM = -1
	ENOENT = -2
	EINTR = -4
}
```

Integer literals assigned to enum items may have a base prefix.

```idol
enum FcntlFlags: u32 {
	O_CREAT  =  0o100
	O_EXCL   =  0o200
	O_NOCTTY =  0o400
	O_TRUNC  = 0o1000
}
```

```abnf
enum = %s"enum" *SP IDENT *SP %x3A *SP IDENT *SP %x7B
          *(SP / NL / COMMENT NL)
          *(enum-item *(SP / NL / COMMENT NL))
          %x7D

enum-item = IDENT *SP %x3D *SP ( ANY-INT-LIT / const-name )
```

#### Enum options and enum item options

Enums and enum items may have options -- see the section on struct fields for details on the syntax.

Supported enum and enum item options are:
* `deprecated: bool`: whether the enum / enum item is deprecated.

### Structs

Use `struct` to declare a fixed-size record type. Struct fields may be of any type that has a known size -- note that this includes other structs, but does not include `text`, `asciz`, or messages.

```abnf
struct = %s"struct" *SP IDENT *SP %x7B
          *(SP / NL / COMMENT NL)
          *(struct-field *(SP / NL / COMMENT NL))
          %x7D

struct-field = IDENT *SP %x3A *SP type-name
```

Structs are typically used when the set of fields is small and stable, because they are more efficient than messages but can be difficult or impossible to change later without breaking binary compatibility.

```idol
# More convenient than `coordinate: f32[3]`
struct Coordinate {
	x: f32
	y: f32
	z: f32
}
```

Structs can also be used to provide names and type-safety to types that represent some other form of serialized data.

```idol
struct Sha256Checksum {
	bytes: u8[32]
}
```

Struct fields follow the same alignment and padding system as C:
* Numeric types `u*`, `i*`, `f*`, and `handle` are aligned to their size.
* Arrays have the same alignment as their value type.
* Structures are aligned to their most-aligned field.
* Padding is implicitly added between structure fields to ensure alignment.

In general adding new fields to a struct will break binary compatibility, because all encoders and decoders must agree on the struct's size and alignment. The exceptions are:
* A new field may replace padding.
  * The declaration `struct { a: u8 b: u16 }` may be modified to `struct { a: u8 c: u8 b: u16 }`.
* A new field may replace existing fields if doing so would not change the struct's alignment.
  * The declaration `struct { a: u32 reserved: u8[4] }` may be modified to `struct { a: u32 b: u16 c: u16 }`.

If a struct is expected to have new fields added in the future, it is recommended to reserve some empty space within the struct for that purpose.

#### Struct options and struct field options

Structs and struct fields may have options, specified with one of (1) `@options { }`, (2) the single-option shorthand syntax `@{ option_name = "value" }`, or (3) the even shorter `bool`-specific syntax `@{option_name}`:

```idol
@{deprecated = .true}
struct OldStruct {
	dont_use_me: u32
}

struct SomeStruct {
	a: u32

	@{deprecated}
	b: u32
}
```

Supported struct and struct field options are:
* `deprecated: bool`: whether the struct / struct field is deprecated.

Struct and struct field options may also have imported schemas:

```idol
import "acme.example/idol-codegen-go" {
	GoCodegenFieldOptions
}

struct SomeStruct {
	@options: GoCodegenFieldOptions { accessor_name = "IsNew" }
	new: bool
}
```

### Messages

Use `message` to declare a dynamically-sized record type. A message may contain fields of any type, including itself. Fields may be optional, in which case generated code will allow detecting whether the field value was present in the message data.

```idol
message Hello {
	greeting@1: text
	language_id@2: u32
}
```

Each field has a tag such as `@1`, which is an integer in the range `[1, 65535]`. The message size depends on the largest tag that is present within the message data: A message containing a `u32` with tag `N` will be `8 + (N * 8)` bytes.

```abnf
message = %s"message" *SP IDENT *SP %x7B
          *(SP / NL / COMMENT NL)
          *(message-field *(SP / NL / COMMENT NL))
          %x7D

message-field = IDENT *SP tag *SP %x3A *SP type-name

tag = %x40 %x31-39 *(%x30-39)
```

A single dynamically-sized field may contain up to `0x7FF00000 - 16` bytes of data if it is the only field in the message and has a tag of `1`.

#### Message options and message field options

Messages and message fields may have options -- see the section on struct options for details on the syntax.

Supported message options are:
* `deprecated: bool`: whether the message is deprecated.

Supported message field options are:
* `deprecated: bool`: whether the field is deprecated.
* `optional: bool`: Whether generated code should provide API for testing field presence.

### Unions

Use `union` to declare a dynamically-sized tagged union. A union is similar to a message, but it contains at most a single value (identified by field tag).

```idol
union DivisionResult {
	result@1: f32
	error@2: ErrorCode
}
```

```abnf
union = %s"union" *SP IDENT *SP %x7B
          *(SP / NL / COMMENT NL)
          *(union-field *(SP / NL / COMMENT NL))
          %x7D

union-field = IDENT *SP tag *SP %x3A *SP type-name
```

Union fields may also be optional, in which case the union header specifies which field _might_ be set.

```idol
# Code review might succeed without any comments from the reviewer.
union CodeReviewResult {
	@{optional}
	comments@1: text

	error@2: ErrorCode
}
```

#### Union options and field options

Unions and union fields may have options -- see the section on struct options for details on the syntax.

Supported union options are:
* `deprecated: bool`: whether the union is deprecated.

Supported union field options are:
* `deprecated: bool`: whether the field is deprecated.
* `optional: bool`: Whether generated code should provide API for testing field presence, in addition to testing the union's field tag.

### Protocols

Use `protocol` to declare the protocol of an RPC-style service.

```idol
protocol Greeter {
	rpc Greet(GreetRequest): GreetResponse # can also `: (GreetResponse)`
}
```

```abnf
protocol = %s"protocol" *SP IDENT *SP %x7B
          *(SP / NL / COMMENT NL)
          *(protocol-method *(SP / NL / COMMENT NL))
          %x7D

protocol-method = protocol-rpc / protocol-event

protocol-rpc = %s"rpc" 1*SP IDENT *SP %x28 *SP type-name [ 1*SP %s"stream" ] *SP %x29 *SP %x3A *SP rpc-response

rpc-response = type-name / %x28 *SP [ type-name [ 1*SP %s"stream" ] *SP ] %x29

protocol-event = %s"event" 1*SP IDENT *SP %x28 *SP type-name [ 1*SP %s"stream" ] *SP %x29
```


#### Protocol RPCs

```idol
protocol Greeter {
	rpc Greet(GreetRequest stream): GreetResponse
}
```

```idol
protocol Greeter {
	rpc Greet(GreetRequest): (GreetResponse stream)
}
```

```idol
protocol Greeter {
	rpc Greet(GreetRequest stream): (GreetResponse stream)
}
```

```idol
protocol Greeter {
	rpc Greet(GreetRequest): ()
}
```

```idol
protocol Greeter {
	rpc Greet(GreetRequest stream): ()
}
```

#### Protocol Events

```
protocol FileWatcher {
	event SomethingHappened(SomeEventPayload)
}
```

#### Protocol options

Protocols, protocol RPCs, and protocol events may have options -- see the section on struct options for details on the syntax.

Supported options for all three productions are:
* `deprecated: bool`: whether the protocol / rpc / event is deprecated.
