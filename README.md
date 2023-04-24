# TinyBuf

Optimised container of immutable owned bytes. Stores `Arc<dyn AsRef<[u8]>>`, `Box<[u8]>`, `Box<dyn AsRef<[u8]>>`, `Vec<u8>`, and `[u8; N]` for small `N` values in one single type (no generics) with a direct move—no additional heap allocation or copying.

All TinyBuf values are slices, so you get all of the [standard slice methods](https://doc.rust-lang.org/std/primitive.slice.html) directly. TinyBuf also implements:

- `AsRef<[u8]>`
- `Clone`
- `Hash`
- `PartialEq` and `Eq`
- `PartialOrd` and `Ord`
- `serde::Serialize` and `serde::Deserialize` (with optional `serde` feature)

## Table of contents

* [Usage](#usage)
* [Use case](#use-case)
  + [Accepting data](#accepting-data)
  + [Providing data](#providing-data)
* [Types](#types)
* [Cloning](#cloning)
* [Comparison](#comparison)
  + [`Vec<u8>`](#vecu8)
  + [`SmallVec` and `TinyVec`](#smallvec-and-tinyvec)
  + [`&[u8]`](#u8)
  + [`Cow<u8>`](#cowu8)

## Usage

Add it to your dependencies:

```
cargo add tinybuf
```

Replace usage of immutable `Vec<u8>` in type locations with `TinyBuf`.

## Use case

TinyBuf is optimised for these scenarios:

- You want to provide or accept immutable owned bytes of varying types and lengths.
- You want to provide or accept immutable `[u8; N]`, where `N <= 23`.

Here are some examples to demonstrate the use case for TinyBuf. For illustration purposes, let's assume we're building an in-memory key-value store, and focus on the values.

### Accepting data

Normally, if you wish to accept some owned bytes, you would use a `Vec`:

```rust
struct MyKV {
  entries: HashMap<&'static str, Vec<u8>>,
}

impl MyKV {
  pub fn set(&mut self, key: &'static str, value: Vec<u8>) {
    self.entries.insert(key, value);
  }
}
```

However, this is not optimal for callers who have their bytes in an array, boxed slice, etc. They'd have to reallocate into a new `Vec`:

```rust
let kv: MyKV = MyKV::new();
// Fast, direct move, no reallocation or copying.
kv.set("vec", vec![0, 4, 2]);
// Requires allocating a new Vec and copying into it.
kv.set("boxslice", Box::new(&[0, 4, 2]).to_vec());
// Requires allocating a new Vec and copying into it.
kv.set("array", [0, 4, 2].to_vec());
```

So the next approach might be to use generics:

```rust
struct MyKV<V: AsRef<[u8]>> {
  entries: HashMap<&'static str, V>,
}

impl<V: AsRef<[u8]>> MyKV<V> {
  pub fn set(&mut self, key: &'static str, value: V) {
    self.entries.insert(key, value);
  }
}
```

This works if the value type is always the same. But perhaps you want to accept different types of bytes—maybe different users or subsystems use different types, or sometimes arrays are used for small data to avoid heap allocation. If so, this isn't possible using the above generics approach. Instead, another approach might be to instead use boxed traits:

```rust
struct MyKV {
  entries: HashMap<&'static str, Box<dyn AsRef<[u8]>>>,
}

impl MyKV {
  pub fn get(&self, key: &'static str) -> Option<&Box<dyn AsRef<[u8]>>> {
    self.entries.get(key)
  }

  pub fn set(&mut self, key: &'static str, value: impl AsRef<[u8]>) {
    self.entries.insert(key, Box::new(value));
  }
}
```

While this approach allows storing different types of bytes without copying to a `Vec`, the boxing essentially reverses that optimisation, since calling `Box::new` will always move it to the heap, even for `Box<[u8]>` values. This may not be too bad for data already on the heap, since it's only the struct itself, not the data, that is heap allocated, but it still requires a heap allocation of a relatively small size.

This is where TinyBuf comes in:

```rust
struct MyKV {
  entries: HashMap<&'static str, TinyBuf>,
}

impl MyKV {
  pub fn get(&self, key: &'static str) -> Option<&TinyBuf> {
    self.entries.get(key)
  }

  pub fn set<D: Into<TinyBuf>>(&mut self, key: &'static str, value: D) {
    self.entries.insert(key, value);
  }
}

let kv: MyKV = MyKV::new();
// Fast, direct move, stored inline in TinyBuf, no heap allocation or copying of data.
kv.set("boxslice", Box::new(&[0, 4, 2]));
kv.set("static", b"my static value");
kv.set("vec", vec![0, 4, 2]);
// For small arrays: fast, direct move, stored inline in TinyBuf, no heap allocation or copying of data.
kv.set("smallarray", [0, 4, 2]);
// If an array is too big, heap allocation is required, so you must explicitly opt in.
kv.set("largearray", TinyBuf::from_slice(&[0u8; 24]));
// If you already have a Box<dyn AsRef[u8]> or Arc<dyn AsRef<[u8]>> value, TinyBuf will accept it too.
kv.set("boxdyn", my_boxed_asref);
kv.set("arc", my_arced_asref);
// If you have some other type that TinyBuf doesn't support and cannot convert it to one of the supported types, you can created a boxed trait value, which will require heap allocation.
kv.set("custom", Box::<dyn AsRef[u8]>::new(my_asrefable_value));
```

### Providing data

As shown in the previous `MyKV` example, one benefit of TinyBuf is to be able to accept and store a lot of different owned byte slice types while providing one single non-generic type, all without heap allocation or copying.

Another advantage is the ability to dynamically leverage small inline arrays when reading from a slice and wanting to convert it into an owned value:

```rust
fn read_from_vm_memory(&self, start: usize, end: usize) -> Vec<u8> {
  // This will always heap allocate, even if the length is tiny.
  self.mem[start..end].to_vec()
}

fn read_from_vm_memory(&self, start: usize, end: usize) -> TinyBuf {
  // This will only heap allocate if the length is large.
  TinyBuf::from_slice(&self.mem[start..end])
}
```

Note that in both cases, the data is copied because we want to own the data, but in the second TinyBuf variant, no heap allocation occurs if the length is less than or equal to 23. This optimisation is done automatically and transparently, with no additional conditional code required.

## Types

`From<T>` is implemented for these types for easy fast conversion.

Note that these will always perform a fast move operation. If you'd like TinyBuf to try and copy the data into an inline `TinyBuf::Array*` variant if the length is small enough, and do a standard `.into()` otherwise, use `TinyBuf::copy_if_small`.

|Source type|Notes|
|---|---|
|`[u8; N]` where `N <= 23`|Stored inline, no heap allocation or copying of data.|
|`Arc<dyn AsRef<[u8]> + Send + Sync + 'static>`||
|`Box<dyn AsRef<[u8]> + Send + Sync + 'static>`||
|`Box<[u8]>`|This is a separate variant as converting to `Box<dyn AsRef<u8>>` would require further boxing.|
|`&'static [u8]`||
|`Vec<u8>`|This is a separate variant as `into_boxed_slice` may reallocate. Capacity must be less than `2^56`.|

## Cloning

When you call `TinyVec::clone`, it will try to perform optimal copying:

- If the length is less than or equal to 23, the data will be cloned into a new `TinyBuf::Array*` variant, even if it's currently not a `TinyBuf::Array*`.
- If it's a `TinyBuf::Arc`, it will be cheaply cloned using `Arc::clone`.
- Otherwise, it will be cloned into a new `TinyBuf::BoxSlice` using boxing (i.e. heap allocation).

## Comparison

### `Vec<u8>`

TinyBuf is a container for immutable data, and does not support any mutation methods, unlike `Vec<u8>`.

TinyBuf will avoid a heap allocation if the length is small enough, whereas `Vec<u8>` will always do a heap allocation except for zero bytes.

### `SmallVec` and `TinyVec`

TinyBuf only holds `u8` values and does not support mutation.

TinyBuf doesn't require declaring the length or capacity upfront, and will dynamically optimise arrays of length less than or equal to 23.

TinyBuf converts from a wide variety of types, not just `Vec<u8>`, and does so using only a move, no data copying.

As a rule of thumb, TinyBuf is more of a replacement for an immutable `Box<dyn AsRef<[u8]>>` than a mutable `Vec<u8>`.

### `&[u8]`

If it's possible to use slices, it's definitely preferred. However, when it's not possible, usually due to lifetime constraints, `TinyBuf` provides close to identical functionality.

### `Cow<u8>`

`Cow` still has a lifetime as it could contain borrowed data, so it's not owned like `Vec<u8>` and TinyBuf.
