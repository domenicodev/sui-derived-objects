# Derived Objects — Move Contract

The on-chain half of the showcase. This Move package defines a shared `ParentObject` and four derived object types, each illustrating a different key strategy for `sui::derived_object::claim`.

None of the creation functions use `object::new(ctx)` — derived IDs are claimed entirely from the parent's `UID` via `derived_object::claim(&mut uid, key)`, decoupling object creation from the `TxContext`. The claim requires a `&mut UID` reference to the parent, which is a deliberate security constraint: only code with mutable access to the parent can mint derived objects from it.

While derived objects use dynamic fields internally to track `ClaimedStatus` on-chain, they are **not** dynamic fields. Once created, each derived object is a fully independent, top-level object with its own ownership.

## Modules

### `parent`

Owns the `ParentObject` struct — the anchor from which all derived IDs are computed. A single instance is created and shared during `init`, so every user can call into it without ownership restrictions.

```move
public struct ParentObject has key {
    id: UID,
    incremental_counter: u64,
}
```

Key points:

- `uid_mut_ref` is `public(package)` — only modules in this package can claim derived IDs from the parent, preventing external packages from squatting on the key space.
- `incremental_counter` tracks the next available index for incremental derivation. It is bumped after each claim.
- `create_parent_object` is exposed as a public function for flexibility (e.g. tests, multi-parent setups), while `init` shares one automatically at deploy.

### `objects`

Contains all four derived object types and their creation functions.

#### `DerivedObjectIncremental`

Derived from `(parent_id, u64)`. The counter lives on the parent, so keys are guaranteed to be sequential and non-repeating.

```move
public fun new_derived_incremental(parent: &mut ParentObject): DerivedObjectIncremental {
    let item_index = parent.get_incremental_counter();
    let derived_id = derived_object::claim(parent.uid_mut_ref(), item_index);
    parent.increment_counter();

    DerivedObjectIncremental {
        id: derived_id,
        derivation_id: object::id(parent),
    }
}
```

#### `DerivedObjectAddress`

Derived from `(parent_id, address)`. Natural fit for per-user objects — pass the sender's address as the key and you get a soulbound slot.

```move
public fun new_derived_address(parent: &mut ParentObject, key: address): DerivedObjectAddress {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), key);

    DerivedObjectAddress {
        id: derived_id,
        derivation_id: object::id(parent),
    }
}
```

#### `DerivedObjectString`

Derived from `(parent_id, String)`. Useful for human-readable keys — names, labels, slugs.

```move
public fun new_derived_string(parent: &mut ParentObject, key: String): DerivedObjectString {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), key);

    DerivedObjectString {
        id: derived_id,
        derivation_id: object::id(parent),
    }
}
```

#### `DerivedObjectStruct`

Derived from `(parent_id, DerivedObjectStructKey)`. The key is a custom struct with `copy`, `drop`, `store` abilities, wrapping an address:

```move
public struct DerivedObjectStructKey(address) has copy, drop, store;
```

This strategy gives you **type-level namespacing** — the full struct path (including package ID) becomes part of the derivation hash. Two packages using the same parent and the same inner address will produce different derived IDs because their key types differ.

```move
public fun new_derived_struct(parent: &mut ParentObject, addr: address): DerivedObjectStruct {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), create_derived_struct_key(addr));

    DerivedObjectStruct {
        id: derived_id,
        derivation_id: object::id(parent),
        addr: addr,
    }
}
```

### The `derivation_id` Field

Every derived object stores a `derivation_id: ID` field pointing back to the parent. This is optional from the protocol's perspective but useful for:

- Off-chain indexing — given any derived object, you can trace which parent it belongs to.
- On-chain assertions — future functions can verify a derived object came from an expected parent.

## Tests

The test module (`derived_objects_tests.move`) covers all four strategies:

| Test | Key type | Notes |
|---|---|---|
| `test_derived_incremental` | `u64` | Uses `#[random_test]` to fuzz 1–5 sequential claims and verify each derived address. |
| `test_derived_address` | `address` | Claims with a fixed address (`@0xA`) and checks determinism. |
| `test_derived_string` | `String` | Claims with `"my_string"` and checks determinism. |
| `test_derived_struct` | `DerivedObjectStructKey` | Creates a struct key from `@0xB`, claims, and checks determinism. |

Every test follows the same pattern:

1. Create a parent.
2. Compute the expected derived address with `derived_object::derive_address`.
3. Claim the derived ID with `derived_object::claim`.
4. Assert the object's ID matches the prediction.

Run them with:

```bash
sui move test
```

## Build & Deploy

```bash
# Build
sui move build

# Deploy to testnet
sui client publish --gas-budget 100000000

# The init function automatically shares a ParentObject.
# Grab its ID from the transaction output for use in the TS tests.
```

## Published

An educational deployment of this contract is already live. See `Published.toml` for the published package ID, chain ID, and upgrade capability details.
