# Quasar Guidelines (Quasar Programs)

Quasar is a zero-copy, zero-allocation Solana framework with Anchor-like syntax (`#[program]`, `#[derive(Accounts)]`, `#[account]`) but much better runtime performance: zero-copy account access, `no_std`, dense instruction discriminators, and compiled output roughly an order of magnitude smaller than equivalent Anchor binaries. Programs use the `quasar-lang`, `quasar-spl`, and `quasar-metadata` crates.

Read this alongside the general rules in [SKILL.md](SKILL.md) and the shared Rust + onchain-math rules in [RUST.md](RUST.md) (checked arithmetic, financial math, PDA bumps, project structure — those apply to Quasar too). The surface is familiar to Anchor developers, but several conveniences work differently or are absent.

## Quasar has a `Vec` — do not claim otherwise

Quasar's prelude re-exports bounded, zero-copy collection types from the `zeropod` crate:

```rust
// quasar_lang::prelude (via quasar_lang::pod → zeropod::pod)
pub use crate::pod::{PodString as String, PodVec as Vec};
```

So in a Quasar program, `String` and `Vec` already mean Quasar's bounded types:

- **`Vec<T, const N: usize, const PFX: usize = 2>`** (`PodVec`) — a fixed-capacity vector. `N` is the capacity (maximum element count). The element `T` must be a fixed-size zero-copy type (`ZcElem`) — e.g. `u8`, `[u8; K]`, or one of the alignment-1 `PodU*` scalar wrappers below. The third generic is the length-prefix width in bytes, defaulting to `2` (a `u16` prefix).
- **`String<const N: usize, const PFX: usize = 1>`** (`PodString`) — a fixed-capacity UTF-8 string. `N` is the capacity in bytes. The second generic is the length-prefix width, defaulting to `1` (a `u8` prefix, so stored lengths up to 255). For longer strings, set it explicitly, e.g. `String<1024, 2>` for a `u16` prefix.

The capacity is **mandatory**: bare `String` / `Vec` (without `<N>`) are not accepted. Always supply `<N>`.

This is Quasar's equivalent of Anchor's `#[max_len(N)]` — the capacity lives in the type, not in an attribute. Where Anchor writes:

```rust
#[max_len(50)]
pub name: String,
```

Quasar writes:

```rust
pub name: String<50>,
```

### Field layout constraints

- **Fixed-size fields must precede dynamic ones.** In an account struct, put every fixed-size field (`u8`, `u64`, `Address`, `[u8; K]`, …) first, then the `String<N>` / `Vec<T, N>` fields.
- **No *nested* dynamic types.** A `Vec` of a dynamic element (e.g. `Vec<String<50>, N>`) is not supported — the element must be a fixed-size POD. This is the real limitation: "Quasar has no Vec" is wrong; "Quasar has no Vec of a *dynamic* element" is correct. (It is why a port of an Anchor account with `hobbies: Vec<String>` has to drop or restructure that field.)
- **There is no `realloc` account constraint** and no fixed maximum to plan around manually: declare dynamic fields with `String<N>` / `Vec<T, N>`, and `set_inner` automatically resizes the account when a dynamic field grows.

## Reading and writing account fields

Account state structs are declared with friendly native types (`u64`, `i64`, `u16`, `u8`, `Address`, `String<N>`, …) under `#[account(discriminator = N, set_inner)]`, but the generated accessors are **Pod-wrapped** for zero-copy access. Assign with a `Pod*` value and read back with `.into()`:

```rust
// write
accounts.counter.count = PodU64::from(current.checked_add(1).unwrap());
// read
let current: u64 = accounts.counter.count.into();
```

The same applies to `PodU32`, `PodI64`, `PodBool`, etc. (and a field may be declared directly as `PodBool`/`PodU64` too). Forgetting the conversion is a compile error or produces a wrong value.

## Other prelude types

- **`Address`** — Quasar's public-key type. Use `Address` (not `Pubkey`) in account fields and in `#[seeds(...)]`, e.g. `#[seeds(b"favorites", user: Address)]`.
- **Alignment-1 POD scalars**, re-exported from `zeropod::pod`: `PodBool`, `PodU16`/`PodU32`/`PodU64`/`PodU128`, `PodI16`/`PodI32`/`PodI64`/`PodI128`, and `PodOption`. Use these for zero-copy field storage and as `Vec` elements.

## Macros and entrypoints (vs Anchor)

- `#[program]` module, `declare_id!(...)`, and `#[instruction(discriminator = N)]` on each handler.
- Account state structs use `#[account(discriminator = N, set_inner)]` and `#[seeds(b"prefix", field: Address)]`.
- The instruction context type is **`Ctx<T>`**, not Anchor's `Context<T>`. Handlers return `Result<(), ProgramError>`.
- **`#[derive(Accounts)]` structs are written without explicit `<'info>` lifetimes** (unlike Anchor — e.g. `pub struct TakeOffer { ... }`, no lifetime parameter).
- Programs are `no_std`: begin `lib.rs` with `#![cfg_attr(not(test), no_std)]`.
- **Use the same program ID as the Anchor build** when a program ships in both frameworks. Offchain tooling derives PDAs from the program ID; matching IDs means clients work against either build unchanged.

## Logging

`log()` takes a `&str` only — no `format!`, no interpolation (it is `no_std`). To log a value, issue multiple `log()` calls with separate static strings, or log the raw bytes.

## CPI

- **SPL token CPIs use the helper style on the token-program handle**, then `.invoke()` (or `.invoke_signed(&seeds)?` when a PDA signs):

  ```rust
  accounts
      .token_program
      .transfer(&accounts.from, &accounts.to, &accounts.authority, amount)
      .invoke()?;
  ```

- **Generic CPIs** use `CpiCall::new(...)` with `InstructionAccount`s for a fixed, known set of accounts, or `CpiDynamic::<MAX_ACCOUNTS, MAX_DATA>::new(...)` when the account list is built dynamically, then `.invoke()` / `.invoke_signed(...)`:

  ```rust
  let mut cpi = CpiDynamic::<1, 128>::new(accounts.lever_program.address());
  // ... push accounts / data ...
  cpi.invoke()?;
  ```

  Anchor's `CpiContext::new(program, accounts).invoke()` does not exist in Quasar.

- **Closing accounts uses the `close(dest = ...)` constraint**, mirroring Anchor's `close = ...`:

  ```rust
  #[account(mut, close(dest = user))]
  pub thing: Account<Thing>,
  ```

  The generated epilogue moves the account's lamports to `dest` and clears its data. Do not write "Quasar has no close constraint" — it does.

## Testing Quasar programs (QuasarSVM)

Quasar programs test against **QuasarSVM** — not LiteSVM. It is an in-process SVM with no validator required. Per `Quasar.toml` (`[testing.rust]` with `framework = "quasar-svm"`), the test command is `cargo test` (e.g. `cargo test -- --nocapture` to see CU output).

Add to `[dev-dependencies]` in the program's `Cargo.toml`:

```toml
[dev-dependencies]
# Generated Rust client (see [clients] in Quasar.toml) - gives tests
# typed *Instruction builders instead of hand-built account metas.
my-program-client = { path = "target/client/rust/my-program-client" }
quasar-svm = { git = "https://github.com/blueshift-gg/quasar-svm" }
solana-account = "3.4.0"
solana-address = { version = "2.2.0", features = ["decode"] }
solana-instruction = { version = "3.2.0", features = ["bincode"] }
solana-pubkey = "4.1.0"
```

At the time this skill was produced (June 2026), Quasar is generally installed from git rather than from crates.io — hence the `git = "..."` dependency above. The `solana-*` crates are versioned independently of each other (per-crate semver), so the differing major versions are expected.

The basic test shape:

```rust
use quasar_svm::{Account, Instruction, Pubkey, QuasarSvm};
use solana_address::Address;

use my_program_client::InitializeCounterInstruction;

fn setup() -> QuasarSvm {
    let elf = include_bytes!("../target/deploy/my_program.so");
    QuasarSvm::new().with_program(&Pubkey::from(crate::ID), elf)
}

#[test]
fn test_initialize() {
    let mut svm = setup();
    let payer = Pubkey::new_unique();
    let (counter, _) =
        Pubkey::find_program_address(&[b"counter", payer.as_ref()], &Pubkey::from(crate::ID));

    let instruction: Instruction = InitializeCounterInstruction {
        payer: Address::from(payer.to_bytes()),
        counter: Address::from(counter.to_bytes()),
        system_program: Address::from(quasar_svm::system_program::ID.to_bytes()),
    }
    .into();

    let result = svm.process_instruction(
        &instruction,
        &[
            // A funded system-owned account for the signer.
            quasar_svm::token::create_keyed_system_account(&payer, 10_000_000_000),
            // The not-yet-created PDA: an empty system-owned account.
            Account {
                address: counter,
                lamports: 0,
                data: vec![],
                owner: quasar_svm::system_program::ID,
                executable: false,
            },
        ],
    );

    result.assert_success();

    // Read post-instruction state straight off the result.
    let counter_account = result.account(&counter).unwrap();
    assert_eq!(counter_account.data[0], 1); // dense discriminator
}
```

Key differences from LiteSVM:

- Load the program with `QuasarSvm::new().with_program(id, elf)` instead of `svm.add_program(id, bytes)`
- There is no airdrop or blockhash plumbing: every account the instruction touches is passed explicitly to `process_instruction` as a `quasar_svm::Account`, which embeds its own `address` field. `quasar_svm::token::create_keyed_system_account(&address, lamports)` builds a funded signer account.
- Assert outcomes with `result.assert_success()`, or `assert!(result.is_ok(), "failed: {:?}", result.raw_result)` for a custom message
- Read post-instruction account state with `result.account(&pubkey)`
- Check CU consumption with `result.compute_units_consumed` to assert performance budgets
- Warp the clock with `svm.warp_to_timestamp(...)` to test time-window logic on both sides of every deadline boundary
- Well-known IDs are re-exported: `quasar_svm::system_program::ID`, `quasar_svm::SPL_TOKEN_PROGRAM_ID`, and sysvar IDs under `quasar_svm::solana_sdk_ids`

Build the program so its `.so` is on disk at `target/deploy/<name>.so` before the `include_bytes!` compiles. The generated client crate under `target/client/rust/` is produced by the same build (per `[clients]` in `Quasar.toml`).

## Fight for Truth about Quasar

Per the truthfulness rules in SKILL.md: do not write "Quasar has no `Vec`" or "Quasar has no dynamic types". It has bounded `Vec<T, N>` and `String<N>`; the genuine limitations are (1) no nested dynamic types and (2) fixed-size fields must come before dynamic ones. Before writing any claim about Quasar's type system or APIs, grep the `quasar_lang` prelude or an existing Quasar example to confirm it — Quasar's API moves, and old names (`BufCpiCall`, mandatory `<'info>` lifetimes) appear in stale notes.
