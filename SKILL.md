---
name: solana-anchor-claude-skill
description: "Use when working on Solana software, including one or more of: Solana client code using TypeScript, Rust libraries that use Solana crates, Anchor programs, including Rust program files, Rust tests using LiteSVM, and Anchor.toml configuration. Designed to create minimal, reusable code without unnecessary duplication."
---

# Coding Guidelines

Apply these rules to ensure code quality, maintainability, and adherence to project standards.

## Do the whole thing

The marginal cost of completeness is near zero with AI. Do the whole thing.

Do it right. Do it with tests. Do it with documentation. Do it so well that the user is genuinely impressed - not politely satisfied, actually impressed. Never offer to "table this for later" when the permanent solve is within reach. Never leave a dangling thread when tying it off takes five more minutes. Never present a workaround when the real fix exists.

The standard isn't "good enough" - it's "holy shit, that's done." Search before building. Test before shipping.

Ship the complete thing. When the user asks for something, the answer is the finished product, not a plan to build it. Time is not an excuse. Fatigue is not an excuse. Complexity is not an excuse. Boil the ocean.

## Success Criteria

- Before declaring success, declaring that work is complete, or celebrating, run `anchor test`. If the tests fail, there is more work to do. Don't stop until `anchor test` passes on the code you have made.
- Do not write placeholder tests. Placeholder tests don't count as tests, placeholder tests passing does not achieve your task.
  - Tests that just do `assert.ok(true)` or similar are placeholder tests and do not count as tests
  - Tests that do not call the program's instruction handlers are placeholder tests and do not count as tests
  - Tests must: initialize accounts, send transactions, verify state changes, check balances
  - If you find yourself writing placeholder tests, stop and write real integration tests instead
  - DO NOT mark "Write tests" as complete until tests actually call the program instructions
  - DO NOT ask "should I write real tests now?" - if the tests are placeholders, write real ones immediately

- Do not stop until documentation like `README.md` and `CHANGELOG.md` are also updated with your changes. If you have made a feature, and it is not documented in the README or changelog, there is more work to do and you must continue working.

- When summarizing your work, show the work items you have achieved with this symbol '✅' and if there is any more work to do, add a '❌' for each remaining work item.

## Documentation Sources

Use these official documentation sources:

- **Anchor**: https://www.anchor-lang.com/docs
- **Anchor Error Codes**: https://raw.githubusercontent.com/coral-xyz/anchor/master/lang/src/error.rs
- **Solana Kite**: https://solanakite.org
- **Solana Kit**: https://solanakit.com
- **Agave (Solana CLI)**: https://docs.anza.xyz/ (Anza makes the Solana CLI and Agave).
- **Switchboard** (if used): https://docs.switchboard.xyz/docs-by-chain/solana-svm
- **Arcium** (if used): https://docs.arcium.com/developers

## Do not use

- Do not use 'Solana Labs' documentation. The company has been replaced by Anza.

- Do not use 'Coral XYZ' documentation. Coral used to maintain Anchor, but Anchor is now maintained by the Solana Foundation (solana.org)

- Do not use any documentaton or tools from Project Serum, which collapsed many years ago.

- Do not use yarn. Use npm. Yarn has no reason to exist and only adds unnecessary dependencies. Replace Yarn with npm everywhere you see it.

- Do not use **Switchboard Functions** - this product is dead and no longer maintained. (Note: Switchboard oracles are still active and usable.)

- Do not use **Clockwork** - this product is dead. For scheduled instruction handler invocation, use [TukTuk](https://github.com/helium/tuktuk/tree/main/typescript-examples) instead.

## Library versions

Use the latest stable Anchor, Rust, TypeScript, Solana Kit, and Kite you can. If a bug occurs, favor updating rather than rolling back.

## Project Documentation

Every project must have a `README.md` file in the project root that includes:

- **Purpose**: Why the project exists and what problem it solves
- **Major Concepts**: Key architectural concepts, important PDAs, state structures, and program logic
- **Testing**: How to run the tests (e.g., `anchor test`)
- **Setup**: Any prerequisites or setup steps needed to work with the project
- **Usage**: Basic usage examples or deployment instructions if applicable

Keep the README focused and practical. Avoid generic boilerplate - write documentation that would actually help someone understand and work with this specific project.

## General Coding Guidelines

### You are a deletionist

Your golden rule is "perfection isn't achieved when there's nothing more to add, rather perfection is achieved when there is nothing more to be taken away".

Remove:

- Comments that simply repeat what the code is doing, or the name of a variable, and do not add further insight.
- Repeated code that should be turned into a named function

### Communication Style

- Do not make disclaimers about being a "complete project" or state what works
- It is expected that work is complete and functional - no need to state this explicitly
- Avoid phrases like "This is a complete implementation" or "All features are working"
- Just deliver the work without meta-commentary about its completeness

### Working with Generated or Unfamiliar Code

**CRITICAL - Verify Before Use:**

- Before calling ANY function whose signature you don't know with certainty, read the actual source code/type definitions first
- NEVER guess or assume what parameters a function accepts based on what seems logical
- Don't invent convenience parameters that don't exist
- Generated code, third-party libraries, and unfamiliar codebases often have different APIs than you expect
- Common mistake: Assuming a function accepts high-level parameters → WRONG. Check the actual signature in the source files first

### Code Honesty and Clarity

- It's important not to deceive anyone reading this code. Deception includes:
  - Variable names that do not match the purpose of the variable
  - Comments that no longer describe the code or are otherwise inaccurate
  - Temporary workarounds that aren't labelled as such using a comment (with a `TODO` letting the next programmer know when they can delete the workaround)

### Variable Naming

Ensure good variable naming. Rather than add comments to explain what things are, give them useful names.

**Don't do this:**

```typescript
// Foo
const shlerg = getFoo();
```

**Do this instead:**

```typescript
const foo = getFoo();
```

**Naming conventions:**

- Arrays should be plurals (`shoes`), items within arrays should be the singular (`shoes.forEach((shoe) => {...})`)
- Functions should be verby, like `calculateFoo` or `getBar`
- Avoid abbreviations, use full words (e.g., use `context` rather than `ctx`). Never use `e` for something thrown, use `thrownObject`, never use `v` when you mean `value`. There is almost no case where a single character variable is a good idea outside maths (eg `p` and `q` for cryptography).
- Name a transaction some variant of `transaction`. Name instructions some variant of `instruction`. Name signatures some variant of `signature`. Do not confuse them - eg if the type looks like an instruction, you should not call it a 'transaction' because that is deceptive.

You can still add comments for additional context, just be careful to avoid comments that are explaining things that would be better conveyed by good variable naming.

### Code Quality

- Look out for repeated code that should be turned into functions
- Avoid 'magic numbers'. Make numbers either have a good variable name, a comment explaining why they are that value, or a reference to the URL you got the value from. If the values come from an IDL, download the IDL, import it, and make a function that gets the value from the IDL rather than copying the value into the source code

This is a magic number. Don't do this:

```ts
const FINALIZE_EVENT_DISCRIMINATOR = new Uint8Array([
  27, 75, 117, 221, 191, 213, 253, 249,
]);
```

Instead do this:

```ts
const FINALIZE_EVENT_DISCRIMINATOR = getEventDiscriminator(
  arciumIdl,
  "FinalizeComputationEvent",
);
```

- The code you are making is for production. You shouldn't have comments like `// In production we'd do this differently` or `**Implementation incomplete** - Needs program config handling and proper PDA derivations` or `**WORK IN PROGRESS**` in the final code you produce, or functions that return placeholder data. Instead: do the fucking work.
- Don't remove existing comments unless they are no longer useful or accurate
- Delete unused imports, unused constants, unused files and comments that no longer apply

## TypeScript Guidelines

These guidelines apply to browser code, scripts, offchain client code, and any other places where TypeScript is used in the project. Anchor programs (since 1.0.0) use Rust + LiteSVM instead of TypeScript — see the **Testing (Rust + LiteSVM)** section under the Rust guidelines.

### General TypeScript

Use `type: module` in `package.json` files.

Avoid using a `tsconfig.json` unless it's needed, as we use `tsx` to run most typescript and it doesn't usually need one. If you do need a `tsconfig.json`, state why at the top of the file, and you can use the most modern version of ECMAScript/JavaScript you want - up to say 2023.

### Async/await

Favor `async`/`await` and `try/catch` over `.then()` or `.catch()` or using callbacks for flow control. `tsx` has top level `await` so you don't need to wrap top level `await` in IIFEs.

### Type System

- **Always use `Array<item>`**, never use `item[]` for consistency with other generic syntax like `Promise<T>`, `Map<K, V>`, and `Set<T>`
- **Don't use `any`**

### Comments

- Most comments should use `//` and be above (not beside) the code
- The only exception is JSDoc/TSDoc comments which MUST use `/* */` syntax

### Solana-Specific TypeScript

- Don't make new `@solana/web3.js` version 1 code. Do not make new code using `@coral-xyz/anchor` package. Don't replace Solana Kit with web3.js version 1 code. web3.js version 1 is legacy and should be eventually removed. Solana Kit used to be called web3.js version 2. Use Solana Kit, preferably via Solana Kite.
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

### Thrown object handling

- JavaScript allows arbitrary items - strings, array, numbers etc to be 'thrown'. However you can assume that any non-Error item that is thrown is an programmer error. Handle it like this (including the comment since most TypeScript developers don't know this):

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

## Rust Guidelines (Anchor Programs)

### Terminology

- Remember this is Solana not Ethereum. Ethereum is not relevant to any documentation you write. Do not assume people know or care about Ethereum.
  - Don't tell me about 'smart contracts' or 'protocols' (use 'programs' instead)
  - Don't tell me about 'gas' (use 'transaction fees' instead)
  - There are no 'mempools'.

- Token program terminology:
  - Use 'Token Extensions Program' or 'Token extensions' for the newer token program (not 'Token 2022' which is just a code name)
  - Use 'Classic Token Program' for the older token program

- Onchain / offchain (one word, no hyphen)
  - Always write 'onchain' and 'offchain' as single, unhyphenated words — like 'online' and 'offline'.
  - Never write 'on-chain' or 'off-chain'. The hyphenated forms are wrong.
  - Apply the same rule to related terms: 'crosschain' (not 'cross-chain'), etc.
  - Sources:
    - [Solana Foundation style guide](https://solana.com/docs/references/terminology)
    - [US Government usage](https://www.sec.gov/files/rules/interp/2026/33-11412.pdf)
    - [Cat (catmcgee) will make fun of you if you write 'on-chain'](https://x.com/catmcgee/status/2028153588715761825)

- Some tools in Solana unfortunately use the same word 'instructions' for both the input and the functions. To avoid confusion, use 'instruction handlers' for the functions that handle instructions, and 'instructions' for the input to those functions.

### Anchor Version

- Write all code like the latest stable Anchor (currently 0.32.1 but there may be a newer version by the time you read this)
- Do not use unnecessary macros that are not needed in the latest stable Anchor

### Anchor has silly defaults

Every project will need an IDL.

```toml
[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
```

and if it uses SPL Tokens (like almost every Anchor project) it will need this dependency (insert whatever version is applicable):

```toml
[dependencies]
anchor-spl = "0.32.1"
```

### Project Structure

- **Never modify the program ID** in `lib.rs` or `Anchor.toml` when making changes
- Create files inside the `state` folder for whatever state is needed
- Create files inside the `instructions` or `handlers` folders (whichever exists) for whatever instruction handlers are needed
- Put Account Constraints in instruction files, but ensure the names end with `AccountConstraints` rather than just naming them the same thing as the function
- Handlers that are only for the admin should be in a new folder called `admin` inside whichever parent folder exists (`instructions/admin/` or `handlers/admin/`)

### Account Constraints

- Use a newline after each key in the account constraints struct, so the macro and the matching key/value have some space from other macros and their matching key/value

### Bumps

- Use `context.bumps.foo` not `context.bumps.get("foo").unwrap()` - the latter is outdated

### Data Structures

- When making structs ensure strings and Vectors have a `max_len` attribute
- Vectors have two numbers for `max_len`: the first is the max length of the vector, the second is the max length of the items in the vector

### Space Calculation (CRITICAL - NO MAGIC NUMBERS)

- **Do not use magic numbers anywhere**. I don't want to see `8 + 32` or whatever.
- **Do not make constants for the sizes of various data structures**
- For `space`, use syntax like: `space = SomeStruct::DISCRIMINATOR.len() + SomeStruct::INIT_SPACE,`
- All structs should have `#[derive(InitSpace)]` added to them, to get the `INIT_SPACE` trait
- **DO NOT use magic numbers**

**Example:**

```rust
#[derive(InitSpace)]
#[account]
pub struct UserProfile {
    pub authority: Pubkey,

    #[max_len(50)]
    pub username: String,

    pub bump: u8,
}

#[derive(Accounts)]
pub struct InitializeProfile<'info> {
    #[account(
        init,
        payer = authority,
        space = UserProfile::DISCRIMINATOR.len() + UserProfile::INIT_SPACE,
        seeds = [b"profile", authority.key().as_ref()],
        bump
    )]
    pub profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

### Error Handling

- Return useful error messages
- Write code to handle common errors like insufficient funds, bad values for parameters, and other obvious situations

### PDA Management

- Add `pub bump: u8` to every struct stored in PDA
- Save the bumps inside each when the struct inside the PDA is created

### System Functions

- When you get the time via Clock, use `Clock::get()?;` rather than `anchor_lang::solana_program::clock`

### Testing (Rust + LiteSVM)

Anchor 1.0+ ships Rust + LiteSVM tests by default — `anchor init` now scaffolds a Rust integration test under `programs/<name>/tests/`, and `Anchor.toml` sets `test = "cargo test"`. Use this as the sole test pattern for Anchor programs. Do not write TypeScript tests for Anchor programs.

#### How to initialise a new project

Always initialise new Anchor projects with both flags pinned explicitly:

```sh
anchor init <name> --package-manager npm --test-template litesvm
```

- `--package-manager npm` — `anchor init`'s default is `yarn`, which this skill bans. Pin npm at init time so you don't have to fix `Anchor.toml` afterwards.
- `--test-template litesvm` — currently the default in `anchor-cli`, but pin it explicitly so the project doesn't break if the default changes. The other templates (`mocha`, `jest`, `rust`, `mollusk`) are not used for new Anchor programs in this skill.

The `--template` flag defaults to `multiple` (multi-file program layout with `instructions/`, `state.rs`, `error.rs`); keep that default. `--template single` is a single `lib.rs` and Anchor itself flags it as "not recommended for production".

#### What `anchor init` gives you

A fresh `anchor init` produces these test-related defaults:

`Anchor.toml`:

```toml
[toolchain]
package_manager = "yarn"

[features]
resolution = true
skip-lint = false

[scripts]
test = "cargo test"

[hooks]
```

`programs/<name>/Cargo.toml` `[dev-dependencies]`:

```toml
[dev-dependencies]
litesvm = "0.10.0"
solana-message = "3.0.1"
solana-transaction = "3.0.2"
solana-signer = "3.0.0"
solana-keypair = "3.0.1"
```

`programs/<name>/tests/test_initialize.rs`:

```rust
use {
    anchor_lang::{solana_program::instruction::Instruction, InstructionData, ToAccountMetas},
    litesvm::LiteSVM,
    solana_message::{Message, VersionedMessage},
    solana_signer::Signer,
    solana_keypair::Keypair,
    solana_transaction::versioned::VersionedTransaction,
};

#[test]
fn test_initialize() {
    let program_id = anchor_scaffold_probe::id();
    let payer = Keypair::new();
    let mut svm = LiteSVM::new();
    let bytes = include_bytes!("../../../target/deploy/anchor_scaffold_probe.so");
    svm.add_program(program_id, bytes).unwrap();
    svm.airdrop(&payer.pubkey(), 1_000_000_000).unwrap();

    let instruction = Instruction::new_with_bytes(
        program_id,
        &anchor_scaffold_probe::instruction::Initialize {}.data(),
        anchor_scaffold_probe::accounts::Initialize {}.to_account_metas(None),
    );

    let blockhash = svm.latest_blockhash();
    let msg = Message::new_with_blockhash(&[instruction], Some(&payer.pubkey()), &blockhash);
    let tx = VersionedTransaction::try_new(VersionedMessage::Legacy(msg), &[payer]).unwrap();

    let res = svm.send_transaction(tx);
    assert!(res.is_ok());
}
```

Before the program binary exists, run `anchor build` so `target/deploy/<name>.so` is on disk; the test loads it via `include_bytes!`.

#### Two scaffold fixes to apply immediately after `anchor init`

`anchor init`'s defaults conflict with this skill's rules. Fix them straight away:

1. **Set `package_manager = "npm"` in `Anchor.toml`** — `anchor init` defaults to yarn, but yarn is banned in this skill. If you used `--package-manager npm` at init time you can skip this step.

   ```toml
   [toolchain]
   package_manager = "npm"
   ```

2. **Delete `ts-mocha`, `mocha`, `chai` (and their `@types`) from `package.json`** — `--package-manager npm` does not remove the JS test dev-dependencies; you still need this step. The default JS test scaffold is stale. Anchor programs (since 1.0.0) use Rust + LiteSVM instead of TypeScript, not Mocha. If you keep a `package.json` at all (for offchain client code or scripts), it should not pull in Mocha-era dependencies.

#### Minimal bare-bones test

The `anchor init` scaffold above is already the minimal pattern — `litesvm` plus the `solana-*` primitives, no extra dependencies. Use this when you want zero indirection and complete control over the transaction. New tests can follow the same shape: build an `Instruction`, wrap in a `Message` with the latest blockhash, sign as a `VersionedTransaction`, and call `svm.send_transaction(tx)`.

#### Optional ergonomic helpers via solana-kite

[`solana-kite`](https://crates.io/crates/solana-kite) is an optional thin layer on top of `litesvm` that removes most of the manual transaction wiring. Used in the wild by [`quiknode-labs/solana-program-examples/basics/counter/anchor`](https://github.com/quiknode-labs/solana-program-examples/tree/main/basics/counter/anchor).

Add to `[dev-dependencies]`:

```toml
[dev-dependencies]
litesvm = "0.10.0"
solana-kite = "0.3.0"
borsh = "1.6.1"
```

The same test, rewritten with kite:

```rust
use {
    anchor_lang::{solana_program::instruction::Instruction, InstructionData, ToAccountMetas},
    litesvm::LiteSVM,
    solana_kite::{create_wallet, send_transaction_from_instructions},
};

#[test]
fn test_initialize() {
    let program_id = anchor_scaffold_probe::id();
    let mut svm = LiteSVM::new();
    let bytes = include_bytes!("../../../target/deploy/anchor_scaffold_probe.so");
    svm.add_program(program_id, bytes).unwrap();

    let payer = create_wallet(&mut svm, 1_000_000_000).unwrap();

    let instruction = Instruction::new_with_bytes(
        program_id,
        &anchor_scaffold_probe::instruction::Initialize {}.data(),
        anchor_scaffold_probe::accounts::Initialize {}.to_account_metas(None),
    );

    send_transaction_from_instructions(&mut svm, &[instruction], &payer, &[&payer]).unwrap();
}
```

`create_wallet` replaces the `Keypair::new()` + `svm.airdrop(...)` pair, and `send_transaction_from_instructions` replaces the `Message` / `VersionedMessage` / `VersionedTransaction` construction. Bare `litesvm` is still the baseline — reach for kite when you have repeated boilerplate worth removing.

#### Account deserialisation

Anchor account data is `[8-byte discriminator][borsh-serialised struct]`. To read state from a LiteSVM test, fetch the account, skip the first 8 bytes, and `borsh`-decode the rest. Define a mirror struct (or import the program's own) that derives `BorshDeserialize`.

```rust
use borsh::BorshDeserialize;

#[derive(BorshDeserialize)]
struct CounterAccount {
    pub count: u64,
}

let account = svm.get_account(&counter_pda).unwrap();
let counter = CounterAccount::try_from_slice(&account.data[8..]).unwrap();
assert_eq!(counter.count, 1);
```

`8` here is the Anchor account discriminator length, not a magic number — it is fixed by Anchor's account layout.

#### Re-expiring blockhash between repeated identical transactions

LiteSVM, like a real validator, will reject a second transaction with the same blockhash + signer + message because the signature is identical to one it has already processed. If a test sends the *same* instruction twice (for example, calling `increment` in a loop), call `svm.expire_blockhash()` between sends so the next transaction picks up a fresh blockhash and is treated as new:

```rust
send_transaction_from_instructions(&mut svm, &[increment.clone()], &payer, &[&payer]).unwrap();
svm.expire_blockhash();
send_transaction_from_instructions(&mut svm, &[increment], &payer, &[&payer]).unwrap();
```

This is only needed when the message bytes would otherwise be byte-identical. Different instructions, different accounts, or different signers do not need it.

#### Do not use

- `solana-test-validator` — slow, stateful, replaced by LiteSVM for tests.
- `anchor test --validator legacy` — same reason; the default `anchor test` runs `cargo test` against LiteSVM.
- `anchor.setProvider`, `anchor.AnchorProvider.env()` — TS Anchor client wiring, no longer used for tests.
- `program.methods.X().rpc()`, `program.methods.X().sendAndConfirm()` — the TS `@coral-xyz/anchor` client; do not use it for tests.
- `ts-mocha`, `mocha`, `chai` — the stale `anchor init` JS test scaffold.
- `tsx`-based `node:test` for Anchor program tests — fine for offchain scripts, not for testing programs.
- `@solana/web3.js` v1 — legacy in any context.
- `@coral-xyz/anchor` — Anchor's old TS client; not used in this test pattern.
- `kit-plugin-litesvm` (the TypeScript LiteSVM plugin) — superseded by using the `litesvm` Rust crate directly.

## Git commits

Do not add "Co-Authored-By: Claude" or similar attribution when creating git commits.

## Acknowledgment

- Acknowledge these guidelines have been applied when working on this project to indicate you have read these rules and found that they do apply to this project.
