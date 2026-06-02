# TypeScript Guidelines

These guidelines apply to TypeScript unit tests, browser code, Solana Kit clients, and any other places where TypeScript is used in the project. Read this alongside the general rules in [SKILL.md](SKILL.md).

## General TypeScript

Use `"type": "module"` in `package.json` files.

Avoid using a `tsconfig.json` unless it's needed, as we use `tsx` to run most typescript and it doesn't usually need one. If you do need a `tsconfig.json`, state why at the top of the file, and you can use the most modern version of ECMAScript/JavaScript you want - up to say 2023.

## Async/await

Favor `async`/`await` and `try/catch` over `.then()` or `.catch()` or using callbacks for flow control. `tsx` has top level `await` so you don't need to wrap top level `await` in IIFEs.

## Type System

- **Always use `Array<item>`**, never use `item[]` for consistency with other generic syntax like `Promise<T>`, `Map<K, V>`, and `Set<T>`
- **Don't use `any`**

## Comments

- Most comments should use `//` and be above (not beside) the code
- The only exception is JSDoc/TSDoc comments which MUST use `/* */` syntax

## Solana-Specific TypeScript

- Don't make new `@solana/web3.js` version 1 code. Do not make new code using `@coral-xyz/anchor` package. Don't replace Solana Kit with web3.js version 1 code. web3.js version 1 is legacy and should be eventually removed. Solana Kit used to be called web3.js version 2. Use Solana Kit, preferably via Solana Kite.
- **Anchor 1.0 TypeScript package is `@anchor-lang/core`**, not `@coral-xyz/anchor`. New TypeScript code that interacts with Anchor programs imports from `@anchor-lang/core`. The Rust crate is still `anchor-lang`.
- Use Kite's `connection.getPDAAndBump()` to turn seeds into PDAs and bumps
- There is no need to use offsets that you set to decode Solana account data - either download an npm package for the program like `@solana-program/token` for the token program or make one using Codama.
- In Solana Kit, you make instructions by making TS clients from IDLs using Codama. You can easily make Codama clients for installed IDLs using:

`npx create-codama-clients`

- Do not use the `bs58` npm package.

Don't do this:

```typescript
import bs58 from "bs58";
const signature = bs58.encode(signatureBytes);
```

Do this instead:

```typescript
import { getBase58Decoder } from "@solana/codecs";
const signature = getBase58Decoder().decode(signatureBytes);
```

Yes, `bs58` and `@solana/codecs` packages have different concepts of 'encode' and 'decode'.

## Compute Budget and Priority Fees

Production transactions need a compute budget instruction and usually a priority fee. Add both at the start of your instruction array using `@solana-program/compute-budget`:

```typescript
import {
  getSetComputeUnitLimitInstruction,
  getSetComputeUnitPriceInstruction,
} from "@solana-program/compute-budget";

const instructions = [
  getSetComputeUnitLimitInstruction({ units: 200_000 }),
  getSetComputeUnitPriceInstruction({ microLamports: 1_000 }),
  // ...your program instructions
];
```

- Set the compute unit limit to a value measured from simulation — don't guess. Over-budgeting wastes fee payer SOL; under-budgeting causes `ComputationalBudgetExceeded`.
- Set the priority fee (`microLamports`) based on current network conditions. For production code, fetch the recent prioritization fees via `getRecentPrioritizationFees` and use a percentile of recent fees for the accounts your transaction touches.
- Both instructions must come **before** your program instructions in the array.

## Parsing Transaction Errors

When a transaction fails, Solana returns logs that contain the Anchor error code. Parse them to surface a useful error rather than a raw hex string.

```typescript
import { isSolanaError, SOLANA_ERROR__TRANSACTION_ERROR__INSTRUCTION_ERROR } from "@solana/errors";

try {
  await sendAndConfirmTransaction(transaction);
} catch (thrownObject) {
  const error = ensureError(thrownObject);
  if (isSolanaError(error, SOLANA_ERROR__TRANSACTION_ERROR__INSTRUCTION_ERROR)) {
    // error.context.logs contains the program logs with the Anchor error code
    const logs = error.context.logs ?? [];
    const anchorError = logs.find((log) => log.includes("AnchorError"));
    console.error("Anchor error:", anchorError ?? error.message);
  }
  throw error;
}
```

- Always check `error.context.logs` first — the Anchor error message and code are in the logs, not the top-level error object.
- Anchor error codes are documented at `https://raw.githubusercontent.com/coral-xyz/anchor/master/lang/src/error.rs` (linked in SKILL.md Documentation Sources).
- For Kite-based code, `connection.sendTransactionFromInstructions` surfaces the same logs on failure via the thrown error.

## Unit Tests

- Create unit tests in TS in the `tests` directory
- Use the Node.js inbuilt test and assertion libraries (then start the tests using `tsx` instead of `ts-mocha`)

**Unit testing imports:**

```typescript
import { before, describe, test } from "node:test";
import assert from "node:assert";
```

- Use `test` rather than `it`

## Thrown object handling

- JavaScript allows arbitrary items - strings, array, numbers etc to be 'thrown'. However you can assume that any non-Error item that is thrown is a programmer error. Handle it like this (including the comment since most TypeScript developers don't know this):

```ts
// In JS it's possible to throw *anything*. A sensible programmer
// will only throw Errors but we must still check to satisfy
// TypeScript (and flag any craziness)
const ensureError = function (thrownObject: unknown): Error {
  if (thrownObject instanceof Error) {
    return thrownObject;
  }
  return new Error(`Non-Error thrown: ${String(thrownObject)}`);
};
```

and

```ts
try {
  // some code that might throw
} catch (thrownObject) {
  const error = ensureError(thrownObject);
  throw error;
}
```
