# Sui Derived Objects

A practical showcase of **Derived Objects** on Sui — deterministic, parallelizable, top-level objects whose IDs can be computed before they even exist on-chain.

This repository contains a Move smart contract that demonstrates four distinct derivation strategies and a TypeScript test suite that proves off-chain ID computation matches on-chain results.

---

## Why Derived Objects?

Every Sui object gets a unique ID at creation time through `object::new(ctx)`, which pulls a fresh address from the `TxContext`. Derived objects flip this model: you **choose** the ID ahead of time by mapping a *parent object* to a *key*. The result is deterministic and unique — the same `(parent, key)` pair always produces the same address, on-chain or off-chain, even before the object is created.

A major advantage is that creating a derived object **does not require `object::new(ctx)`**. You only need a mutable reference to the parent's `UID` — `derived_object::claim(&mut uid, key)` returns the derived `UID` directly. This means object creation is decoupled from the transaction context and tied solely to the parent, enabling patterns where many objects can be spawned from a single known parent without ever touching `TxContext` for ID generation.

Under the hood, derived objects use dynamic fields for their on-chain storage of the `ClaimedStatus`, but they are **not** dynamic fields themselves. Once created, a derived object is a fully independent, top-level object — it has its own ownership, can be shared or transferred, and is accessed directly without routing through the parent.

> Claiming a derived ID requires a `&mut UID` reference to the parent. This is a deliberate security constraint: only code with mutable access to the parent's UID can mint derived objects from it, preventing unauthorized modules from squatting on your key space.

This unlocks patterns that are impossible (or painfully slow) with dynamic fields:

| Aspect | Derived Objects | Dynamic Fields |
|---|---|---|
| Address predictability | Yes | Yes |
| Parent required at runtime? | Only at creation | Always |
| Ownership | Any (shared, owned, wrapped, frozen) | Always owned by parent |
| Receives transfers before existing? | Yes | No |
| Parallel execution | Yes — independent keys run in parallel | Limited — writes are sequenced through the parent |
| Loading | Direct access after creation | Dynamic, loaded through parent |

Because derived objects are **not children** of the parent, transactions touching different keys execute in parallel with zero contention on the parent. The parent only exists to guarantee uniqueness.

### Off-Chain Superpowers

Since IDs are deterministic, you can **derive object IDs off-chain without making any API call**. The derivation is a pure, synchronous function — given the parent ID, key type, and key value, the SDK computes the address immediately in-memory. No network round-trip, no async, no RPC. This means your client code can:

- Compute any derived object's address instantly — no sequential queries through a parent.
- Bundle multiple lookups into a single `multiGetObjects` request.
- Index objects by simply iterating over known key spaces (see the [reverse-search test](#reverse-search-on-incremental-keys)).

---

## Repository Structure

```
sui-derived-objects/
├── contracts/
│   └── derived_objects/           # Move smart contract package
│       ├── sources/
│       │   ├── parent.move        # ParentObject definition & init
│       │   └── objects.move       # Four derived object types + creation fns
│       ├── tests/
│       │   └── derived_objects_tests.move  # On-chain unit tests
│       ├── Move.toml
│       └── Published.toml         # Testnet deployment metadata
│
└── ts/                            # TypeScript integration tests
    ├── src/
    │   ├── suiClient.ts           # Sui gRPC client setup
    │   └── helpers/
    │       ├── deriveObjectID.ts   # Off-chain ID derivation wrapper
    │       ├── getSigner.ts       # Ed25519 keypair helper
    │       └── typeTags.ts        # BCS type tag constants
    ├── tests/
    │   ├── derive.test.ts         # End-to-end derivation tests
    │   └── derive.utils.ts        # Shared test utilities
    ├── .env.example               # Required environment variables
    ├── package.json
    ├── vitest.config.ts
    └── tsconfig.json
```

---

## The Four Derivation Strategies

The contract showcases four ways to derive an object ID from a parent, each suited to different use cases.

### 1. Incremental (`u64` key)

Best for **ordered collections** — think auto-incrementing database IDs. The parent maintains a counter, and each new object claims the next index.

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

Off-chain, you can compute the ID of the *next* object by reading the current counter:

```typescript
const derivedKey = await testUtils.getCurrentIncrementalCounter();
const expectedID = deriveObjectIDFromParent(TYPE_TAGS.U64, derivedKey, bcs.U64);
```

### 2. Address key

Best for **per-address singletons** — users, objects, ... The address serves as the key, guaranteeing one object per address.  
Addresses can easily be obtained by object IDs using `id.to_address()`;

```move
public fun new_derived_address(parent: &mut ParentObject, key: address): DerivedObjectAddress {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), key);

    DerivedObjectAddress {
        id: derived_id,
        derivation_id: object::id(parent),
    }
}
```

```typescript
const derivedKey = new Ed25519Keypair().getPublicKey().toSuiAddress();
const expectedID = deriveObjectIDFromParent(TYPE_TAGS.ADDRESS, derivedKey, bcs.Address);
```

### 3. String key

Best for **named slots** — human-readable registries, config entries, DNS-like mappings.

```move
public fun new_derived_string(parent: &mut ParentObject, key: String): DerivedObjectString {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), key);

    DerivedObjectString {
        id: derived_id,
        derivation_id: object::id(parent),
    }
}
```

```typescript
const derivedKey = "my_unique_name";
const expectedID = deriveObjectIDFromParent(TYPE_TAGS.STRING, derivedKey, bcs.String);
```

### 4. Struct key

Best for **type-namespaced derivation** — when you want the key type itself to carry semantic meaning. The struct key uses `copy`, `drop`, and `store` abilities and wraps an address.

```move
public struct DerivedObjectStructKey(address) has copy, drop, store;

public fun new_derived_struct(parent: &mut ParentObject, addr: address): DerivedObjectStruct {
    let derived_id = derived_object::claim(parent.uid_mut_ref(), create_derived_struct_key(addr));

    DerivedObjectStruct {
        id: derived_id,
        derivation_id: object::id(parent),
        addr: addr,
    }
}
```

The off-chain type tag includes the full struct path:

```typescript
const keyType = `${PACKAGE_ID}::objects::DerivedObjectStructKey`;
const expectedID = deriveObjectIDFromParent(keyType, derivedKey, bcs.Address);
```

---

## Off-Chain ID Derivation

The TypeScript helper wraps the `@mysten/sui` SDK's `deriveObjectID` utility:

```typescript
import { deriveObjectID } from "@mysten/sui/utils";

export const deriveObjectIDFromParent = <T extends BcsType<any, any, any>>(
    keyType: string,
    key: any,
    bcsHelper: T
): string => {
    return deriveObjectID(
        process.env.PARENT_OBJECT_ID!,
        keyType,
        bcsHelper.serialize(key).toBytes()
    );
};
```

Three ingredients: the **parent object ID**, the **key's type tag**, and the **BCS-serialized key value**.

Notice that `deriveObjectID` is a **synchronous function** — it returns a `string`, not a `Promise`. The derivation is pure math (hashing the parent ID, type tag, and BCS-encoded key), so it runs entirely in-memory with zero network calls. You can derive thousands of object IDs in a tight loop without awaiting anything or opening a single connection.

---

## Reverse Search on Incremental Keys

One powerful pattern enabled by deterministic IDs is **reverse iteration** over incremental objects. Because you know the key space (`0..counter`), you can compute IDs in descending order and batch-fetch them:

```typescript
const currentCounter = await testUtils.getCurrentIncrementalCounter();
const maxItems = Math.min(10, currentCounter);
const lastItem = currentCounter - 1;

const derivedKeys = Array.from({ length: maxItems }, (_, k) => {
    return deriveObjectIDFromParent(TYPE_TAGS.U64, lastItem - k, bcs.U64);
});

const queriedObjects = await suiClient.getObjects({ objectIds: derivedKeys });
```

This gives you paginated, reverse-chronological access with a single `multiGet` call — no indexer, no cursor, no sequential parent reads.

---

## Getting Started

### Prerequisites

- [Sui CLI](https://docs.sui.io/guides/developer/getting-started/sui-install) for contract compilation and deployment
- [Bun](https://bun.sh) (or Node.js) for TypeScript tests

### Run Move Tests

```bash
cd contracts/derived_objects
sui move test
```

### Run TypeScript Tests

```bash
cd ts
cp .env.example .env
# Fill in USER_SECRET_KEY with a funded testnet keypair
bun install
bun run test
```

The tests execute real transactions against testnet. Each test creates a derived object on-chain and asserts that the object ID matches the off-chain prediction.

### Environment Variables

| Variable | Description |
|---|---|
| `NETWORK` | Sui network (`testnet`, `devnet`, `localnet`, `mainnet`) |
| `PACKAGE_ID` | Published package ID of the `derived_objects` contract |
| `PARENT_OBJECT_ID` | ID of the shared `ParentObject` created at deploy |
| `USER_SECRET_KEY` | Ed25519 secret key for signing transactions |
| `DERIVED_STRUCT_KEY_SIGNATURE` | Struct key name without positional args (e.g. `DerivedObjectStructKey`) |

---

## Deployed Module

An educational deployment of this contract is already live. Check `contracts/derived_objects/Published.toml` for the published package ID and chain details if you want to run the TypeScript tests without deploying your own.

---

## License

This project is provided as an educational showcase of Sui derived objects.
