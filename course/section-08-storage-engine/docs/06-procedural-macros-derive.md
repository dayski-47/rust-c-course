# Doc 06 - Procedural Macros and Custom Derive

🟡 When you write `#[derive(Serialize)]`, a program runs at compile time that reads your struct's type definition and generates an implementation. That program is a procedural macro. This doc teaches you how to write one.

Most Rust developers use proc macros every day (serde, thiserror, tokio::main) but treat them as black boxes. For ironkv, you'll write a `#[derive(Storable)]` macro that generates binary serialization code for any struct that implements it - the same pattern used by serde, but for your custom binary format.

---

## The Three Types of Macros

**Declarative macros (`macro_rules!`)** - pattern-match on syntax, expand to code. Good for reducing repetitive call-site boilerplate:

```rust
// Declarative macro - matches a pattern and expands
macro_rules! assert_entry_eq {
    ($engine:expr, $key:expr, $expected:expr) => {
        assert_eq!(
            $engine.get($key).unwrap(),
            Some($expected.to_vec()),
            "entry mismatch for key {:?}",
            $key
        );
    };
}

// Use in tests:
assert_entry_eq!(db, b"user:1", b"alice");
assert_entry_eq!(db, b"user:2", b"bob");
```

**Procedural macros** - a Rust program that takes a `TokenStream` (the parsed source code) as input and returns a `TokenStream` (the generated code). Three subtypes:

| Type | Syntax | Use Case |
|------|--------|----------|
| Custom derive | `#[derive(MyTrait)]` | Generate trait impls |
| Attribute macro | `#[my_attribute]` | Transform functions/structs |
| Function-like | `my_macro!(...)` | Custom syntax |

Custom derive is the most common and the one you'll write for ironkv.

---

## Declarative Macros for Test Boilerplate

Before writing a proc macro, check if `macro_rules!` is enough. For test boilerplate, it usually is:

```rust
// Generate multiple test cases from a table
macro_rules! test_roundtrip {
    ( $( $name:ident: ($key:expr, $val:expr) ),* $(,)? ) => {
        $(
            #[test]
            fn $name() {
                let dir = tempfile::tempdir().unwrap();
                let mut db = IronKV::open(dir.path()).unwrap();
                db.put($key, $val).unwrap();
                assert_eq!(db.get($key).unwrap(), Some($val.to_vec()));
            }
        )*
    };
}

test_roundtrip! {
    roundtrip_empty_key:   (b"", b"value"),
    roundtrip_empty_value: (b"key", b""),
    roundtrip_binary:      (b"\x00\x01\x02", b"\xFF\xFE\xFD"),
    roundtrip_large_value: (b"big", &[0u8; 1024 * 1024][..]),  // 1 MB value
}
// Generates 4 separate test functions
```

This pattern eliminates copy-pasted test functions while keeping the test table readable.

### Fragment Types

The fragment types you'll use most often:

```rust
macro_rules! example {
    ($e:expr)    => { /* any expression: 42, a + b, foo() */ };
    ($t:ty)      => { /* any type: i32, Vec<String>, impl Trait */ };
    ($i:ident)   => { /* an identifier: my_var, Config */ };
    ($b:block)   => { /* a block: { let x = 1; x } */ };
    ($l:literal) => { /* a literal: 42, "hello", true */ };
    ($tt:tt)     => { /* any token tree - most flexible */ };
}

// Repetition patterns:
// $( ... ),*   - zero or more, comma-separated
// $( ... );+   - one or more, semicolon-separated
// $( ... )?    - zero or one (optional)
```

---

## Writing a Custom Derive Macro

A custom derive macro lives in its own crate with `proc-macro = true`. This is a hard requirement - proc macros use a special ABI that can only appear in proc-macro crates.

### Crate setup

```toml
# ironkv-derive/Cargo.toml
[package]
name = "ironkv-derive"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

In the workspace:

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["ironkv-core", "ironkv-derive", "ironkv-cli"]

[workspace.dependencies]
ironkv-derive = { path = "ironkv-derive" }
```

### The `Storable` derive macro

The goal: `#[derive(Storable)]` on a struct generates `impl Storable for MyStruct` with `to_bytes()` and `from_bytes()` methods.

```rust
// ironkv-derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

/// Derives `Storable` for a struct, generating binary serialization methods.
///
/// The generated format: for each field, write its size as a u64 LE,
/// then the field's bytes. This supports variable-length fields.
#[proc_macro_derive(Storable)]
pub fn derive_storable(input: TokenStream) -> TokenStream {
    // Parse the input tokens into a syntax tree
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;  // The struct name

    // Only support structs with named fields
    let fields = match &input.data {
        Data::Struct(s) => match &s.fields {
            Fields::Named(f) => &f.named,
            _ => panic!("Storable only supports structs with named fields"),
        },
        _ => panic!("Storable only supports structs"),
    };

    // Generate the serialization code for each field
    let serialize_fields = fields.iter().map(|f| {
        let field_name = f.ident.as_ref().unwrap();
        quote! {
            // Write the field as: [8-byte length][field bytes]
            let field_bytes = self.#field_name.to_storable_bytes();
            bytes.extend_from_slice(&(field_bytes.len() as u64).to_le_bytes());
            bytes.extend_from_slice(&field_bytes);
        }
    });

    // Generate the deserialization code for each field
    let deserialize_fields = fields.iter().map(|f| {
        let field_name = f.ident.as_ref().unwrap();
        let field_type = &f.ty;
        quote! {
            // Read: [8-byte length][field bytes] → field value
            let len = u64::from_le_bytes(
                cursor[..8].try_into().map_err(|_| "truncated length")?
            ) as usize;
            cursor = &cursor[8..];
            let #field_name = <#field_type as StorableField>::from_storable_bytes(&cursor[..len])
                .map_err(|_| "field deserialization failed")?;
            cursor = &cursor[len..];
        }
    });

    let field_names: Vec<_> = fields.iter().map(|f| f.ident.as_ref().unwrap()).collect();

    // The generated impl
    let expanded = quote! {
        impl Storable for #name {
            fn to_bytes(&self) -> Vec<u8> {
                let mut bytes = Vec::new();
                #(#serialize_fields)*
                bytes
            }

            fn from_bytes(data: &[u8]) -> Result<Self, &'static str> {
                let mut cursor = data;
                #(#deserialize_fields)*
                Ok(#name {
                    #(#field_names),*
                })
            }
        }
    };

    TokenStream::from(expanded)
}
```

### Using the macro

```rust
// ironkv-core/src/lib.rs
use ironkv_derive::Storable;

pub trait Storable: Sized {
    fn to_bytes(&self) -> Vec<u8>;
    fn from_bytes(data: &[u8]) -> Result<Self, &'static str>;
}

#[derive(Debug, Storable)]
pub struct IndexEntry {
    pub key: Vec<u8>,
    pub offset: u64,
    pub length: u32,
}

// The macro generates:
// impl Storable for IndexEntry {
//     fn to_bytes(&self) -> Vec<u8> { ... }
//     fn from_bytes(data: &[u8]) -> Result<Self, &'static str> { ... }
// }

fn example() {
    let entry = IndexEntry {
        key: b"user:1".to_vec(),
        offset: 4096,
        length: 128,
    };

    let bytes = entry.to_bytes();
    let restored = IndexEntry::from_bytes(&bytes).unwrap();
    assert_eq!(entry.key, restored.key);
    assert_eq!(entry.offset, restored.offset);
}
```

---

## Debugging Proc Macros with `cargo-expand`

The hardest thing about proc macros is that errors can be cryptic - a bug in the generated code appears as an error in code you didn't write. `cargo-expand` shows you the generated code:

```bash
cargo install cargo-expand

# Show all macro expansions in the crate
cargo expand

# Show expansion for a specific module
cargo expand storage::index
```

When `#[derive(Storable)]` generates broken code, `cargo expand` lets you see exactly what the generated `impl Storable for IndexEntry { ... }` looks like. Most macro bugs are visible immediately in the output.

---

## Attribute Macros for Test Setup

Function-like attribute macros can transform async functions - the same pattern used by `#[tokio::test]`:

```rust
// Usage (what you'd write):
#[ironkv::test]
async fn test_put_and_get() {
    let value = db.get(b"key").await.unwrap();
    assert_eq!(value, Some(b"val".to_vec()));
}

// What the macro generates (conceptually):
#[tokio::test]
async fn test_put_and_get() {
    // Setup: create a temp directory and open the DB
    let _dir = tempfile::tempdir().unwrap();
    let mut db = IronKV::open_for_testing().await.unwrap();
    
    // The original test body
    let value = db.get(b"key").await.unwrap();
    assert_eq!(value, Some(b"val".to_vec()));
    
    // Teardown: DB is closed when dropped
}
```

This eliminates the setup/teardown boilerplate from every test. The macro injects it.

---

## When to Reach for Macros

**Reach for macros when:**

- Reducing boilerplate that requires AST-level transformation (derive traits, code from type structure)
- Generating code that varies structurally based on its arguments (test tables, DSLs)
- Compile-time DSLs that the compiler should validate (`sql!`, `html!`, format strings)

**Don't reach for macros when:**

- Generics and traits can express the same thing - they're clearer and produce better error messages
- The repetition is in call sites, not in definitions - use functions
- The "boilerplate" is actually clarifying - making something implicit is not always an improvement

The test table pattern (`test_roundtrip!`) is a good use: it removes 10+ lines of identical setup per test case while keeping the table readable. The `#[derive(Storable)]` macro is a good use: there's no other way to inspect a struct's fields at compile time. But adding a macro to avoid writing three related functions is almost always wrong - a trait with a default implementation is simpler.
