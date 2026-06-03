# Anchor Guidelines (Anchor Programs)

Anchor-specific rules. Read these alongside the general rules in [SKILL.md](SKILL.md) and the shared Rust + onchain-math rules in [RUST.md](RUST.md) (checked arithmetic, financial math, PDA bumps, project structure, no magic numbers — those apply to Anchor too).

## Anchor Version

- Write all code like the latest stable Anchor (currently 1.0.2 but there may be a newer version by the time you read this)
- Use LiteSVM and Rust tests for new Anchor programs. `anchor init` uses LiteSVM by default.
- Do not use unnecessary macros that are not needed in the latest stable Anchor

### Anchor 1.0 specifics

- **`CpiContext::new()` takes a `Pubkey` directly**, not an `AccountInfo`. Use `self.token_program.key()` rather than `self.token_program.to_account_info()` when constructing a `CpiContext`.
- **Use `transfer_checked` for all SPL token CPIs.** Plain `transfer` is deprecated. `transfer_checked` requires the mint and decimals, which you should be passing anyway.

## Anchor has silly defaults

Every project will need an IDL, and al,ost every project will use tokens, so ensure that features are correct...

```toml
[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]

## Token program compatibility

- **For token program compatibility, use `anchor_spl::token_interface`** (`InterfaceAccount<TokenAccount>`, `InterfaceAccount<Mint>`, `Interface<TokenInterface>`). The same code then works against both the Classic Token Program and the Token Extensions Program. Pair it with `transfer_checked` (see RUST.md).

## Data Structures

- When making structs ensure strings and Vectors have a `max_len` attribute
- Vectors have two numbers for `max_len`: the first is the max length of the vector, the second is the max length of the items in the vector

## Space Calculation (CRITICAL - NO MAGIC NUMBERS)

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

## Bumps

- Use `context.bumps.foo` not `context.bumps.get("foo").unwrap()` - the latter is outdated

## System Functions

- When you get the time via Clock, use `Clock::get()?;` rather than `anchor_lang::solana_program::clock`

## Testing (Rust + LiteSVM)

Anchor 1.0+ ships Rust + LiteSVM tests by default — `anchor init` now scaffolds a Rust integration test under `programs/<name>/tests/`, and `Anchor.toml` sets `test = "cargo test"`. Use this as the sole test pattern for Anchor programs. Do not write TypeScript tests for Anchor programs.

### How to initialise a new project

Always initialise new Anchor projects with both flags pinned explicitly:

```sh
anchor init <name> --package-manager npm --test-template litesvm
```

- `--package-manager npm` — `anchor init`'s default is `yarn`, which this skill bans. Pin npm at init time so you don't have to fix `Anchor.toml` afterwards.
- `--test-template litesvm` — currently the default in `anchor-cli`, but pin it explicitly so the project doesn't break if the default changes. The other templates (`mocha`, `jest`, `rust`, `mollusk`) are not used for new Anchor programs in this skill.

The `--template` flag defaults to `multiple` (multi-file program layout with `instructions/`, `state.rs`, `error.rs`); keep that default. `--template single` is a single `lib.rs` and Anchor itself flags it as "not recommended for production".

### What `anchor init` gives you

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

### Two scaffold fixes to apply immediately after `anchor init`

`anchor init`'s defaults conflict with this skill's rules. Fix them straight away:

1. **Set `package_manager = "npm"` in `Anchor.toml`** — `anchor init` defaults to yarn, but yarn is banned in this skill. If you used `--package-manager npm` at init time you can skip this step.

   ```toml
   [toolchain]
   package_manager = "npm"
   ```

2. **Delete `ts-mocha`, `mocha`, `chai` (and their `@types`) from `package.json`** — `--package-manager npm` does not remove the JS test dev-dependencies; you still need this step. The default JS test scaffold is stale. Anchor programs (since 1.0.0) use Rust + LiteSVM instead of TypeScript, not Mocha. If you keep a `package.json` at all (for offchain client code or scripts), it should not pull in Mocha-era dependencies.

### Minimal bare-bones test

The `anchor init` scaffold above is already the minimal pattern — `litesvm` plus the `solana-*` primitives, no extra dependencies. Use this when you want zero indirection and complete control over the transaction. New tests can follow the same shape: build an `Instruction`, wrap in a `Message` with the latest blockhash, sign as a `VersionedTransaction`, and call `svm.send_transaction(tx)`.

### Optional ergonomic helpers via solana-kite

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

### Account deserialisation

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

### Re-expiring blockhash between repeated identical transactions

LiteSVM, like a real validator, will reject a second transaction with the same blockhash + signer + message because the signature is identical to one it has already processed. If a test sends the *same* instruction twice (for example, calling `increment` in a loop), call `svm.expire_blockhash()` between sends so the next transaction picks up a fresh blockhash and is treated as new:

```rust
send_transaction_from_instructions(&mut svm, &[increment.clone()], &payer, &[&payer]).unwrap();
svm.expire_blockhash();
send_transaction_from_instructions(&mut svm, &[increment], &payer, &[&payer]).unwrap();
```

This is only needed when the message bytes would otherwise be byte-identical. Different instructions, different accounts, or different signers do not need it.

### Do not use

- `solana-test-validator` — slow, stateful, replaced by LiteSVM for tests.
- `anchor test --validator legacy` — same reason; the default `anchor test` runs `cargo test` against LiteSVM.
- `anchor.setProvider`, `anchor.AnchorProvider.env()` — TS Anchor client wiring, no longer used for tests.
- `program.methods.X().rpc()`, `program.methods.X().sendAndConfirm()` — the TS `@coral-xyz/anchor` client; do not use it for tests.
- `ts-mocha`, `mocha`, `chai` — the stale `anchor init` JS test scaffold.
- `tsx`-based `node:test` for Anchor program tests — fine for offchain scripts, not for testing programs.
- `@solana/web3.js` v1 — legacy in any context.
- `@coral-xyz/anchor` — Anchor's old TS client; not used in this test pattern.
- `kit-plugin-litesvm` (the TypeScript LiteSVM plugin) — superseded by using the `litesvm` Rust crate directly.
