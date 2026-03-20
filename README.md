# Torque

High-performance JSON library for Elixir via [Rustler](https://github.com/rustler-magic/rustler) NIFs, powered by [sonic-rs](https://github.com/cloudwego/sonic-rs) (SIMD-accelerated).

Torque provides the fastest JSON encoding and decoding available in the BEAM ecosystem, with a selective field extraction API for workloads that only need a subset of fields from each document.

## Features

- SIMD-accelerated decoding (AVX2/SSE4.2 on x86, NEON on ARM)
- Ultra-low memory encoder (64 B per encode vs ~4 KB for OTP `json`/jason)
- Parse-then-get API for selective field extraction via JSON Pointer (RFC 6901)
- Batch field extraction (`get_many/2`) with single NIF call
- Automatic dirty CPU scheduler dispatch for large inputs
- jiffy-compatible `{proplist}` encoding

## Installation

Add to your `mix.exs`:

```elixir
def deps do
  [
    {:torque, "~> 0.1.3"}
  ]
end
```

Precompiled binaries are available for common targets. To compile from source, install a stable Rust toolchain and set `TORQUE_BUILD=true`.

## Usage

### Decoding

```elixir
{:ok, data} = Torque.decode(~s({"name":"Alice","age":30}))
# %{"name" => "Alice", "age" => 30}

data = Torque.decode!(json)
```

### Selective Field Extraction

Parse once, extract many fields without building the full Elixir term tree:

```elixir
{:ok, doc} = Torque.parse(json)

{:ok, "example.com"} = Torque.get(doc, "/site/domain")
nil = Torque.get(doc, "/missing/field", nil)

# Batch extraction (single NIF call, fastest path)
results = Torque.get_many(doc, ["/id", "/site/domain", "/device/ip"])
# [{:ok, "req-1"}, {:ok, "example.com"}, {:ok, "1.2.3.4"}]
```

### Encoding

```elixir
# Maps with atom or binary keys
{:ok, json} = Torque.encode(%{id: "abc", price: 1.5})
# "{\"id\":\"abc\",\"price\":1.5}"

# Bang variant
json = Torque.encode!(%{id: "abc"})

# iodata variant (fastest, no {:ok, ...} tuple wrapping)
json = Torque.encode_to_iodata(%{id: "abc"})

# jiffy-compatible proplist format
{:ok, json} = Torque.encode({[{:id, "abc"}, {:price, 1.5}]})
```

## API

| Function | Description |
|----------|-------------|
| `Torque.decode(binary)` | Decode JSON to Elixir terms |
| `Torque.decode!(binary)` | Decode JSON, raising on error |
| `Torque.parse(binary)` | Parse JSON into opaque document reference |
| `Torque.get(doc, path)` | Extract field by JSON Pointer path |
| `Torque.get(doc, path, default)` | Extract field with default for missing paths |
| `Torque.get_many(doc, paths)` | Extract multiple fields in one NIF call |
| `Torque.get_many_nil(doc, paths)` | Extract multiple fields, `nil` for missing |
| `Torque.length(doc, path)` | Return length of array at path |
| `Torque.encode(term)` | Encode term to JSON binary |
| `Torque.encode!(term)` | Encode term, raising on error |
| `Torque.encode_to_iodata(term)` | Encode term, returns binary directly (fastest) |

## Type Conversion

### JSON to Elixir

| JSON | Elixir |
|------|--------|
| object | map (binary keys) |
| array | list |
| string | binary |
| integer | integer |
| float | float |
| `true`, `false` | `true`, `false` |
| `null` | `nil` |

For objects with duplicate keys, the last value wins.

### Elixir to JSON

| Elixir | JSON |
|--------|------|
| map (atom/binary keys) | object |
| list | array |
| binary | string |
| integer | number |
| float | number |
| `true`, `false` | `true`, `false` |
| `nil` | `null` |
| atom | string |
| `{keyword_list}` | object |

## Errors

Functions return `{:error, reason}` tuples (or raise `ArgumentError` for bang/iodata variants). Possible `reason` atoms:

### Decode / Parse

| Atom | Returned by | Meaning |
|------|-------------|---------|
| `:nesting_too_deep` | `decode/1`, `get/2`, `get_many/2` | Document exceeds 512 nesting levels |

`parse/1` and `decode/1` also return `{:error, binary}` with a message from sonic-rs for malformed JSON.

### Encode

| Atom | Returned by | Meaning |
|------|-------------|---------|
| `:unsupported_type` | `encode/1` | Term has no JSON representation (PID, reference, port, …) |
| `:invalid_utf8` | `encode/1` | Binary string or map key is not valid UTF-8 |
| `:invalid_key` | `encode/1` | Map key is not an atom or binary (e.g. integer key) |
| `:malformed_proplist` | `encode/1` | `{proplist}` contains a non-`{key, value}` element |
| `:non_finite_float` | `encode/1` | Float is infinity or NaN (unreachable from normal BEAM code) |
| `:nesting_too_deep` | `encode/1` | Term exceeds 512 nesting levels |

## Benchmarks

Apple M2 Pro, OTP 28, Elixir 1.19:

### Decode (1.2 KB)

| Library | ips | mean | median | p99 | memory |
|---|---|---|---|---|---|
| torque | **258.5K** | **3.87 μs** | **3.54 μs** | **8.29 μs** | **1.56 KB** |
| simdjsone | 185.0K | 5.41 μs | 4.96 μs | 11.50 μs | 1.59 KB |
| jiffy | 152.0K | 6.58 μs | 5.83 μs | 17.83 μs | **1.56 KB** |
| otp json | 130.9K | 7.64 μs | 7.17 μs | 15.54 μs | 7.73 KB |
| jason | 108.0K | 9.26 μs | 8.50 μs | 20.21 μs | 9.54 KB |

### Parse + Get

| Library | ips | mean | median | p99 | memory |
|---|---|---|---|---|---|
| torque parse+get_many_nil | **193.4K** | **5.17 μs** | **4.50 μs** | 10.92 μs | 1.59 KB |
| torque parse+get_many | 187.4K | 5.34 μs | 4.54 μs | **10.63 μs** | **1.58 KB** |
| torque parse+get | 152.6K | 6.55 μs | 5.83 μs | 15.63 μs | 2.80 KB |
| simdjsone parse+get | 119.6K | 8.36 μs | 6.13 μs | 34.88 μs | 2.28 KB |

### Encode (1.2 KB)

| Library | ips | mean | median | p99 | memory |
|---|---|---|---|---|---|
| otp json: map => iodata | **1055.0K** | **0.95 μs** | **0.88 μs** | 1.63 μs | 3928 B |
| torque: proplist => iodata | 843.0K | 1.19 μs | 1.08 μs | **1.54 μs** | **64 B** |
| torque: proplist => binary | 834.0K | 1.20 μs | 1.08 μs | **1.54 μs** | 88 B |
| torque: map => iodata | 756.5K | 1.32 μs | 1.21 μs | 1.75 μs | **64 B** |
| torque: map => binary | 744.7K | 1.34 μs | 1.21 μs | 1.75 μs | 88 B |
| jason: map => iodata | 606.2K | 1.65 μs | 1.50 μs | 3.08 μs | 3488 B |
| jiffy: proplist => iodata | 562.0K | 1.78 μs | 1.54 μs | 3.33 μs | 120 B |
| otp json: map => binary | 548.8K | 1.82 μs | 1.54 μs | 5.25 μs | 3992 B |
| jiffy: map => iodata | 462.5K | 2.16 μs | 1.87 μs | 6.00 μs | 824 B |
| simdjsone: proplist => iodata | 445.6K | 2.24 μs | 2.04 μs | 3.50 μs | 184 B |
| jason: map => binary | 381.0K | 2.62 μs | 2.33 μs | 7.17 μs | 3552 B |
| simdjsone: map => iodata | 370.9K | 2.70 μs | 2.42 μs | 6.04 μs | 888 B |

### Decode (750 KB)

| Library | ips | mean | median | p99 | memory |
|---|---|---|---|---|---|
| torque | **486.5** | **2.06 ms** | **1.58 ms** | **3.23 ms** | **1.56 KB** |
| simdjsone | 436.3 | 2.29 ms | 1.82 ms | 3.51 ms | **1.56 KB** |
| otp json | 193.0 | 5.18 ms | 5.20 ms | 6.17 ms | 2.49 MB |
| jason | 137.8 | 7.26 ms | 7.09 ms | 8.66 ms | 3.55 MB |
| jiffy | 107.9 | 9.27 ms | 9.47 ms | 10.62 ms | 5.53 MB |

### Encode (750 KB)

| Library | ips | mean | median | p99 | memory |
|---|---|---|---|---|---|
| torque: proplist => iodata | **613.7** | **1.63 ms** | **1.62 ms** | 1.79 ms | **64 B** |
| torque: proplist => binary | 613.1 | **1.63 ms** | **1.62 ms** | **1.78 ms** | 88 B |
| torque: map => iodata | 572.4 | 1.75 ms | 1.73 ms | 2.03 ms | **64 B** |
| torque: map => binary | 570.8 | 1.75 ms | 1.74 ms | 1.93 ms | 88 B |
| otp json: map => iodata | 532.6 | 1.88 ms | 1.83 ms | 2.24 ms | 5.40 MB |
| jiffy: proplist => iodata | 314.3 | 3.18 ms | 2.87 ms | 4.21 ms | 37.7 KB |
| jiffy: map => iodata | 282.8 | 3.54 ms | 3.27 ms | 4.61 ms | 1.06 MB |
| jason: map => iodata | 281.0 | 3.56 ms | 3.55 ms | 4.57 ms | 4.96 MB |
| otp json: map => binary | 259.0 | 3.86 ms | 3.71 ms | 5.62 ms | 5.40 MB |
| simdjsone: proplist => iodata | 239.5 | 4.18 ms | 3.88 ms | 5.18 ms | 37.7 KB |
| simdjsone: map => iodata | 214.9 | 4.65 ms | 4.41 ms | 5.78 ms | 1.06 MB |
| jason: map => binary | 190.5 | 5.25 ms | 5.28 ms | 6.59 ms | 4.96 MB |

Run benchmarks locally:

```bash
MIX_ENV=bench mix run bench/torque_bench.exs
```

## Limitations

- **Nesting depth**: JSON documents nested deeper than 512 levels return `{:error, :nesting_too_deep}` from `decode/1`, `get/2`, `get_many/2`, and `encode/1` rather than crashing the VM. Real-world documents are never this deep; the limit exists to prevent stack overflow in the NIF.

## License

MIT
