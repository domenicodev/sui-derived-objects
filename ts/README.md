# Derived Objects — TypeScript Tests

The off-chain half of the showcase. This package proves that derived object IDs can be **computed client-side** before (or without) touching the network — then validates the prediction against real on-chain transactions on Sui testnet.

## What This Demonstrates

1. **Off-chain ID derivation** — Using `deriveObjectID` from the `@mysten/sui` SDK to compute the exact object address that the Move contract will produce.
2. **End-to-end correctness** — Each test creates a derived object on testnet, then asserts `createdObject.objectId === expectedDerivedObjectID`.
3. **Reverse search** — Iterating over incremental keys in descending order and batch-fetching objects with a single `getObjects` call.

## Project Structure

```
ts/
├── src/
│   ├── suiClient.ts               # SuiGrpcClient configured from env
│   └── helpers/
│       ├── deriveObjectID.ts       # Core: off-chain ID derivation wrapper
│       ├── getSigner.ts            # Ed25519 keypair from secret key
│       └── typeTags.ts             # BCS type tag constants (String, address, u64)
├── tests/
│   ├── derive.test.ts              # 5 test cases covering all derivation strategies
│   └── derive.utils.ts             # TestUtils class — tx building, signing, assertions
├── .env.example                    # Template for required env vars
├── package.json
├── vitest.config.ts
└── tsconfig.json
```

## Core: Off-Chain Derivation

The key helper is `deriveObjectIDFromParent` in `src/helpers/deriveObjectID.ts`:

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

Three inputs:

| Input | Example | Description |
|---|---|---|
| `keyType` | `"u64"`, `"address"`, `"0x1::string::String"` | Full type tag of the key used in `derived_object::claim` |
| `key` | `42`, `"0xA..."`, `"my_string"` | The actual key value |
| `bcsHelper` | `bcs.U64`, `bcs.Address`, `bcs.String` | BCS serializer matching the key type |

The parent object ID comes from the environment. With these three pieces you can predict any derived object's address without a network call.

## Type Tags

The `TYPE_TAGS` enum keeps BCS type strings in one place:

```typescript
export enum TYPE_TAGS {
    STRING = "0x1::string::String",
    ADDRESS = "address",
    U64 = "u64",
}
```

For struct keys, the type tag is dynamic and includes the full package path:

```typescript
const keyType = `${PACKAGE_ID}::objects::DerivedObjectStructKey`;
```

## Test Cases

All tests live in `tests/derive.test.ts` and run against a live testnet deployment.

| Test | Key type | What it does |
|---|---|---|
| **String** | `0x1::string::String` | Derives with a random string (`"RANDOM_STRING" + Date.now()`), creates on-chain, asserts ID match. |
| **Address** | `address` | Derives with a random Ed25519 address, creates on-chain, asserts ID match. |
| **Incremental (u64)** | `u64` | Reads the parent's current counter, derives the next ID, creates on-chain, asserts ID match. |
| **Struct key** | `{PACKAGE}::objects::DerivedObjectStructKey` | Derives with a custom struct key wrapping a random address, creates on-chain, asserts ID match. |
| **Reverse search** | `u64` | Computes up to 10 incremental IDs in descending order, batch-fetches them with `getObjects`, asserts all exist. |

### Test Flow

Every creation test follows the same pattern:

```
compute expected ID off-chain  →  build & execute transaction  →  find created object  →  assert IDs match
```

The reverse search test shows a practical indexing pattern:

```typescript
const derivedKeys = Array.from({ length: maxItems }, (_, k) => {
    return deriveObjectIDFromParent(TYPE_TAGS.U64, lastItem - k, bcs.U64);
});

const queriedObjects = await suiClient.getObjects({ objectIds: derivedKeys });
```

## Setup

### 1. Install dependencies

```bash
bun install
```

### 2. Configure environment

```bash
cp .env.example .env
```

Fill in the `.env` file:

| Variable | Description |
|---|---|
| `NETWORK` | `testnet` (or `devnet`, `localnet`, `mainnet`) |
| `PACKAGE_ID` | Published package address |
| `PARENT_OBJECT_ID` | Shared `ParentObject` ID from deployment |
| `USER_SECRET_KEY` | Ed25519 secret key for a funded account |
| `DERIVED_STRUCT_KEY_SIGNATURE` | Struct key name without positional args (e.g. `DerivedObjectStructKey`) |

### 3. Run tests

```bash
bun run test
```

Tests execute real transactions, so the signing account needs testnet SUI. You can get some from the [Sui faucet](https://docs.sui.io/guides/developer/getting-started/get-coins).

## Dependencies

| Package | Purpose |
|---|---|
| `@mysten/sui` | Sui TypeScript SDK — client, BCS, transaction building, `deriveObjectID` |
| `vitest` | Test runner |
