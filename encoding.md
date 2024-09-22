# Encoding

Idol is intended for fast zero-copy IPC, and its encoding is designed to approximate the properties of hand-written C structures used in traditional Unix IPC protocols. On top of this basic goal it adds vtable-like `message` types, which are similar to [Protocol Buffers](https://protobuf.dev/) in that new fields can be added without breaking the ABI.

The Idol encoding is expected to be stable, but its documentation is still very much a work in progress. Readers may find it easier to understand the current documentation if they are already familiar with other IPC protocols such as [D-Bus](https://dbus.freedesktop.org/doc/dbus-specification.html), Fuchsia's [FIDL](https://fuchsia.dev/fuchsia-src/reference/fidl/language/wire-format), and/or Linux's [netlink](https://docs.kernel.org/userspace-api/netlink/intro.html).

## Numbers and enums

Numeric values are encoded in little-endian, with signed integers in two's complement. `bool` is encoded as `0x00` for `false`, and `0x01` for `true`.

`handle` is encoded as `0x00000000` if absent, and `0xFFFFFFFF` if present.

An enum value has the same encoding (including alignment) as the numeric type the enum is derived from.

## Structs

Struct fields follow the same alignment and padding system as C:
* Numeric types `u*`, `i*`, `f*`, `handle`, and enums are aligned to their size.
* Fixed-size arrays have the same alignment as their value type.
  * Structs may not contain dynamically-sized arrays.
* Structs are aligned to their most-aligned field.
* Padding is implicitly added between structure fields to ensure alignment. Padding must contain only `0x00` bytes.

## Arrays

Fixed-length arrays of fixed-size items are encoded as the concatenation of their encoded items. Padding of `0x00` is inserted between items if necessary for alignment.
* The alignment of the array is the alignment of the item type.

Variable-length arrays of fixed-size items are encoded as the concatenation of their encoded items in the same format as fixed-length arrays. The total length of the array in bytes (stored separately) is divided by the padded item size to compute the array length.
* The alignment of the array is the alignment of the item type.

Fixed-length arrays of length `N` containing variable-size values are encoded as a `u32[N]` array of item sizes, followed by optional padding to the item alignment, then the concatenation of their encoded items. Padding of `0x00` is inserted between items if necessary for alignment.
* The alignment of the array is 4 (alignment of `u32`).

Variable-length arrays of variable-size values are encoded as a `u32[]` array of item sizes, followed by optional padding to the item alignment, then the concatenation of their encoded items. Padding of `0x00` is inserted between items if necessary for alignment.
* The alignment of the array is 4 (alignment of `u32`).

## Messages

A message is serialized as a fixed-size "message header", followed by an array of "thunks", and then the values of dynamically-sized or large fields (if present).

```
MessageHeader (8 bytes)

thunks[]
  thunk[0] (8 bytes)
  thunk[1] (8 bytes)
  thunk[2] (8 bytes)
  ...
  thunk[thunk_count-1]

[value data ...]
```

Thunks for small fixed-size fields are called "inline thunks" because the field value is stored directly in the thunk. Dynamically-sized or large fields are called "indirect thunks".

A serialized message may be in either "encoded" or "decoded" format.
* The portable format is resistent to malicious messages, but incurs extra validation overhead on each field access. This also means that accessing a field of an encoded message is a failable operation.
* The decoded format is more efficient for random field access, but decoded messages should only be inspected if they came from a trusted source (such as the kernel, or host process of an interpreter runtime).

A message can be encoded or decoded in-place without additional allocation if the field types are known. If decoding was successful, then the resulting decoded message can be trusted to provide valid data for the fields for which type information was available.

Generated code is expected to combine decoding with an API that only permits access to known fields as the appropriate type.

### Message header

The message header contains the size of the message, the size of its thunks array, and a flags bitset.

```idol
struct MessageHeader {
	size: u32
	flags: u16
	thunk_count: u16
}
```

Field semantics:
* `size` is the size of the message in bytes, including the message header and any trailing padding.
  * Message sizes are always a multiple of 8.
  * The maximum size of a message is `0x7FF00000` bytes (2047 MiB).
* `flags` is `%x00.00`.
* `thunk_count` is the number of thunks in the message's thunks array.
  * Due to the relationship between field tags and thunks, this is also the largest tag value of the present fields.

If `thunk_count` is `N`, then the message contains tags `T <= N`, and the thunk for tag `T` is in `message.thunks[T-1]`.

If testing the presence of a field with tag `T`, then if `T > thunk_count` then that field is not present in the message.

### Inline thunks

The layout of an inline thunk is:

```idol
struct InlineThunk {
	handle_count: u16
	flags: u16
	value: u32
}
```

Field semantics:
* `handle_count` is `%x00.00` except for `handle` fields, for which it is `%x01.00`.
* `flags` is `%x00 %b10000000` if present, `%x00.00` if absent.
* `value` is the encoded value. Unused bytes, including struct padding, must be `0x00`.

### Encoded indirect thunks

The layout of an encoded indirect thunk is:

```idol
struct EncodedIndirectThunk {
	handle_count: u16
	flags: u16
	value_size: u32
}
```

Field semantics:
* `handle_count` is the total number of `handle` values contained within the indirect field's data. It is an encoding error for this count to overflow `0xFFFF`.
* `flags` is `%x00 %b11000000` if present, `%x00.00` if absent.
* `value_size` is the size of the value, in bytes.

To locate the value data within the message's dynamically-sized data segment, the message's thunks must be iterated through and their value sizes (with padding) summed up.
* Each value's data is aligned to 8 bytes, so a value with a `value_size` of `14` would be followed by two bytes of `0x00` padding.
* Note that value data is aligned regardless of the alignment of the field type. A field containing `u8[12]` has an alignment of `1`, but its value data is still aligned to 8 bytes.
  * Similarly, a variable-length `u8[]` array containing a single item will have its data aligned to 8 bytes.

### Decoded indirect thunks

The layout of a decoded indirect thunk is:

```idol
struct DecodedIndirectThunk {
	# { flags: u4 offset: u28 }
	offset_and_flags: u32
	value_size: u32
}
```

Field semantics:
* `offset` is the offset of the value data, relative to the start of the message's dynamically-sized data segment, right-shifted by three.
  * To obtain the actual offset, compute `(thunk.offset_and_flags & 0x0FFFFFFF) << 3`, then add it to the data segment base address.
* `flags` is `%b1000` if present, `%b0000` if absent.
* `value_size` is the unpadded size of the value, in bytes.

In memory, the `flags` and `offset` fields are combined into a single `u32`. They're in little-endian layout regardless of the host's native endianness, so the layout looks like this:
* `thunk.bytes[0]` is `offset[0:8]`
* `thunk.bytes[1]` is `offset[8:16]`
* `thunk.bytes[2]` is `offset[16:24]`
* `thunk.bytes[3]` is `offset[24:28] | (flags << 4)`

### Empty value optimization

Dynamically-sized values that are present but "empty" are optimized to avoid having their headers occupy space in the data segment. These values must have their thunk's `size` field set to `0x00000000`. Field accessor functions should special-case this size and return a synthetic empty value.

* The empty value for `u64`, `i64`, and `f64` is `%x00.00.00.00.00.00.00.00`.
* The empty value for `text` and `asciz` is `%x00`.
* The empty value for variable-length arrays of fixed-size items naturally has `size: 0`, and so does not need to be optimized.
* The empty value for variable-length arrays of variable-size items is `%x00.00.00.00`.
* The empty value for a message or union is `%x08.00.00.00.00.00.00.00`.

## Unions

The format of thunks (inline and indirect, encoded and decoded) is the same for unions as it is in messages.

### Union header

The union header contains the size of the union, which field might be set, and a flags bitset.

```idol
struct UnionHeader {
	size: u32
	flags: u16
	field_tag: u16
}
```

Field semantics:
* `size` is the size of the union in bytes, including the union header and any trailing padding.
  * Union sizes are always a multiple of 8.
  * The maximum size of a union is `0x7FF00000` bytes (2047 MiB).
* `flags` is `%x00.00`.
* `field_tag` is the tag of the field that the union is set to.
