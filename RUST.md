# Rust Guidelines (Anchor Programs)

These guidelines apply to Anchor programs and any Rust crates that use Solana dependencies. Read this alongside the general rules in [SKILL.md](SKILL.md).

## Anchor Version

- Write all code like the latest stable Anchor (currently 1.0.2 but there may be a newer version by the time you read this)
- Use LiteSVM and Rust tests for new Anchor programs. `anchor init` uses LiteSVM by default.
- Do not use unnecessary macros that are not needed in the latest stable Anchor
- Don't implement instruction handlers as methods on account structs. There's no reason to tie state to functions, the function is not modifying the state (if we did like OOP, which we don't), and the functions and structs work without doing this, so there's no reason to implement instruction handlers as methods on account structs.

### Anchor 1.0 specifics

- **`CpiContext::new()` takes a `Pubkey` directly**, not an `AccountInfo`. Use `self.token_program.key()` rather than `self.token_program.to_account_info()` when constructing a `CpiContext`.
- **Use `transfer_checked` for all SPL token CPIs.** Plain `transfer` is deprecated. `transfer_checked` requires the mint and decimals, which you should be passing anyway.

## Anchor has silly defaults

Every project will need an IDL.

```toml
[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
```

and if it uses Tokens (like almost every Anchor project) it will need this dependency (insert whatever version is applicable):

```toml
[dependencies]
anchor-spl = "1.0.2"
```

## Project Structure

- **Never modify the program ID** in `lib.rs` or `Anchor.toml` when making changes
- Create files inside the `state` folder for whatever state is needed
- Create files inside the `instructions` or `handlers` folders (whichever exists) for whatever instruction handlers are needed
- Put Account Constraints in instruction files, but ensure the names end with `AccountConstraints` rather than just naming them the same thing as the function
- Handlers that are only for the admin should be in a new folder called `admin` inside whichever parent folder exists (`instructions/admin/` or `handlers/admin/`)

## Account Constraints

- Use a newline after each key in the account constraints struct, so the macro and the matching key/value have some space from other macros and their matching key/value

## Bumps

- Use `context.bumps.foo` not `context.bumps.get("foo").unwrap()` - the latter is outdated

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

## Error Handling

- Return useful error messages
- Write code to handle common errors like insufficient funds, bad values for parameters, and other obvious situations
- All arithmetic in onchain code is `checked_*` — never raw `+ - * /`. Solana's BPF doesn't trap on overflow in release builds; silent wraps are how hacks happen. `checked_*` returns `Option`; force the error with `.ok_or(MyError::MathOverflow)?`. Reserve `saturating_*` for cosmetic/UX display values, never for balances.

## Onchain Financial Math

Applies to any code touching money, balances, prices, shares, fees, or token amounts. These rules are non-negotiable.

- **Integers only — no floats, no fixed-point libraries.** Floats are non-deterministic across platforms (different validators could disagree on state). `fixed::types::I64F64`, `rust_decimal`, `bnum`-fixed-point and similar are also out — they add audit surface, burn compute, and hide the rounding/precision decisions you should be making explicitly. Token amounts are integers (base units), prices are ratios of integers. The system is discrete. Production Solana AMMs (Orca, Raydium, Meteora, Saber, Phoenix) all use raw `u128`. If you find yourself reaching for a decimal type, stop — the right tool is `u128` with discipline.
- **Multiply before you divide.** `a * b / c`, not `(a / c) * b`. Division truncates; dividing first throws away precision permanently.
- **Use `u128` (or wider) for intermediate products.** `u64 * u64` overflows at ~1.8e19. Cast both operands to `u128` _before_ multiplying, then narrow the final result with `try_into().map_err(|_| MyError::MathOverflow)?`.
- **For price × quantity × rate, u128 may not be enough.** `price (u64) × quantity (u64) × rate (u64)` can reach ~5.4e58, which overflows u128 (~3.4e38). If your hot path multiplies three u64-scale values together, either bound your inputs tightly and document the proof, or hand-roll a U256 type (~200 lines of Newton-based arithmetic, as Uniswap V2 in Solidity and Saber in Rust do). Don't reach for a crate just for this.
- **Round in the protocol's favour, never the user's.** Value-to-share and share-to-value conversions: user gets floor, protocol gets ceil. Otherwise you leak 1 base unit per transaction forever, and attackers will industrialise it.
- **Validate ranges before doing the math.** Reject zero inputs, `amount > balance`, ratios that would mint zero shares. Cheap, prevents the inflation/donation attack on empty pools and other whole bug classes.
- **Check invariants after the math, not just before.** "K must not decrease" on a swap, "total LP shares == sum of holdings", "reserves >= owed fees". Compute, then `require!()` the invariant.
- **Assert token conservation on every instruction that moves funds.** Before returning from any instruction that transfers tokens between vaults or accounts, verify that total value in == total value out. Sum all debits and credits and `require!` they balance. A one-line conservation check catches an entire class of drain bugs before they reach mainnet.
- **Decimals are tracked, not assumed.** USDC=6, SOL=9, SPL tokens vary. Use `transfer_checked` (carries decimals in the CPI). Reserves hold raw base units; the UI does cosmetic conversion. Never hard-code `* 10^9`.
- **Oracle/price freshness is part of the math.** Check `last_updated_slot` and reject if older than N slots. A stale price means the calculation is wrong.
- **Checks-effects-interactions.** Update state before the token transfer CPI, not after.
- **Treat client-supplied values as adversarial.** If a handler takes `(amount_a, amount_b)`, verify each against onchain state, not against each other.
- **Test the branch the bug lives in.** Standard AMM/lending bugs sit in the _non-empty pool_, _post-swap_, _post-fee_, _rounding-edge_ branches. The happy path almost always works. Write the test that exercises the branch where the bug actually lives.
- **LP shares use different formulas for first deposit vs subsequent.** First deposit: shares = `sqrt(amount_a * amount_b)` (geometric mean bootstraps the pool). Subsequent deposits: shares = `min(amount_a * supply / reserve_a, amount_b * supply / reserve_b)` (proportional to share-of-pool). Using the geometric mean for every deposit is a real, repeated bug — test both branches separately.
- **For integer sqrt, hand-code Newton's method on `u128`** (~15 lines, as Uniswap V2 in Solidity / Saber in Rust do). Don't reach for a fixed-point crate for one sqrt.
- **Slippage protection: accept a `min_output_*` from the user and verify before the CPI.** Swaps, deposits, and withdraws all need it. Without it, sandwich attackers steal value across the price gap they create.
- **Never silently clamp user input to balance.** If a user asks to swap 100 and you clamp to 80 because that's the balance, the user's slippage check passes against the wrong amount. Either fail the instruction or return the actual amount so the client can validate.
- **Use `transfer_checked`, never raw `transfer`.** `transfer_checked` carries the mint and decimals through the CPI, so a wrong-mint or wrong-decimals account causes a CPI failure instead of a silent miscalculation.
- **For token program compatibility, use `anchor_spl::token_interface`** (`InterfaceAccount<TokenAccount>`, `InterfaceAccount<Mint>`, `Interface<TokenInterface>`). The same code then works against both the Classic Token Program and the Token Extensions Program.
- **Oracle freshness uses slots, not unix time.** Slot count is what the runtime guarantees; `Clock::get()?.unix_timestamp` is validator-influenced. Check `last_updated_slot` against `Clock::get()?.slot` and reject if older than N slots. If you must use a unix timestamp (because the oracle only exposes one), state why in a comment.
- **Canonical pubkey ordering for two-asset pools.** Order mints so `mint_a.key() < mint_b.key()` (lexicographic on the 32-byte key). Same pool whether the user passes `(USDC, SOL)` or `(SOL, USDC)`. Enforce in the constraint, don't rely on the client.

### Escrows, Vaults, and Escape Hatches

- **Every escrow needs a cancel/withdraw instruction.** An escrow with no cancel locks abandoned offers forever — funds become unrecoverable when the counterparty disappears. The cancel must be callable by the maker (and only the maker) at any time before the trade settles.
- **Don't use `init_if_needed` for an account the wrong party would pay rent for.** Common bug: the taker's instruction lazily creates the maker's destination ATA via `init_if_needed`, so the taker pays the maker's rent. Either require the maker to pre-create their ATA or pass the rent payer explicitly.
- **Update state before the CPI.** Already in the list above, but worth repeating in the vault context: write the new balance/share count first, then transfer. A CPI that re-enters (rare on Solana but possible via callbacks) sees current state, not stale state.

**Pattern to copy when ratio-clamping (Uniswap V2 style):**

```rust
let pool_a = pool_a_amount as u128;
let pool_b = pool_b_amount as u128;
let amount_a_u128 = amount_a as u128;
let amount_b_u128 = amount_b as u128;

// Multiply before divide; u128 prevents overflow.
let amount_b_required = amount_a_u128
    .checked_mul(pool_b).ok_or(ErrorCode::MathOverflow)?
    .checked_div(pool_a).ok_or(ErrorCode::MathOverflow)?;

let (final_a, final_b) = if amount_b_required <= amount_b_u128 {
    (amount_a_u128, amount_b_required)
} else {
    let amount_a_required = amount_b_u128
        .checked_mul(pool_a).ok_or(ErrorCode::MathOverflow)?
        .checked_div(pool_b).ok_or(ErrorCode::MathOverflow)?;
    (amount_a_required, amount_b_u128)
};

let final_a: u64 = final_a.try_into().map_err(|_| ErrorCode::MathOverflow)?;
let final_b: u64 = final_b.try_into().map_err(|_| ErrorCode::MathOverflow)?;
```

## Config Validation

When an admin instruction accepts a configuration struct, don't just check that the fields are well-formed — check that they are financially safe. Write a `validate_config` function that rejects configurations which would make the program exploitable even if each field is individually valid:

- Verify that fee rates, margins, and limits don't combine to produce impossible states (e.g. liquidation fee + protocol fee > maintenance margin means the protocol can be insolvent by design)
- Verify that numeric bounds don't overflow the hot-path calculations that use them
- `require!` these properties when the config is accepted, not in every handler that reads it

This is the difference between "is this a valid u64" and "does this configuration make the program safe". Reject unsafe configs at init time.

## Cargo hygiene

- Run `cargo clean` after finishing with a Rust project. Anchor `target/` directories accumulate fast (multi-GiB per project).
- If disk usage hits 85%, clean before doing more work.

## PDA Management

- Add `pub bump: u8` to every struct stored in PDA
- Save the bumps inside each when the struct inside the PDA is created
- **Include an instance discriminator in accounts that belong to a specific instance.** If your program manages multiple independent pools, markets, or vaults, store the instance's pubkey (e.g. the pool or market address) in each subordinate account and validate it on every instruction. Without this, an attacker can pass an account from one instance into an instruction for another — the types match, Anchor won't catch it, and the math silently operates on the wrong state.

## System Functions

- When you get the time via Clock, use `Clock::get()?;` rather than `anchor_lang::solana_program::clock`

## Editing Quasar Programs

Quasar is a Solana program framework that uses Anchor-like syntax (`#[program]`, `#[derive(Accounts)]`, `#[account]`) but with better runtime performance: zero-copy account access, `no_std`, dense instruction discriminators, and compiled output roughly an order of magnitude smaller than equivalent Anchor binaries. The surface is familiar to Anchor developers, but several conveniences work differently or are absent.

- **Use the same program ID as the Anchor build.** Offchain tooling derives PDAs from the program ID; matching IDs across Anchor and Quasar binaries means clients work against either build unchanged.
- **Account state numeric fields are Pod-wrapped.** Read with `let value: u64 = self.field.into();` and write with `self.field = PodU64::from(value);`. Same for `PodU32`, `PodI64`, etc. Forgetting the conversion compiles but produces wrong values.
- **`log()` takes a static string only.** No format strings, no interpolation. To log a value, issue multiple `log()` calls with separate static strings or log the raw bytes.
- **Account structs need explicit lifetimes.** Use `<'info>` on the struct and `&'info` / `&'info mut` on each reference field. Quasar will not infer these.
- **No `realloc` constraint.** Pick the maximum size up front and use fixed-capacity inline storage like `PodString<N>` or `PodVec<T, N>`. Plan the layout before writing the state struct.
- **No `close` constraint.** To close an account, zero the lamports and clear the data manually inside the handler.
- **No `CpiContext`.** SPL CPIs use the helper style on the program handle: `self.token_program.transfer(...).invoke()`. Generic CPIs use `BufCpiCall::new(...).invoke()`. Anchor's `CpiContext::new(program, accounts).invoke()` pattern does not exist in Quasar.

### Testing Quasar programs (QuasarSVM)

Quasar programs use **QuasarSVM** as the test harness — not LiteSVM. It is an in-process SVM with no validator required. Run tests with `quasar test` (or `cargo test -- --nocapture` to see CU output).

Add to `[dev-dependencies]` in the program's `Cargo.toml`:

```toml
[dev-dependencies]
quasar-svm = { git = "https://github.com/blueshift-gg/quasar-svm" }
solana-account = "3.4.0"
solana-address = { version = "2.2.0", features = ["decode"] }
solana-instruction = { version = "3.2.0", features = ["bincode"] }
solana-pubkey = "4.1.0"
```

The basic test shape:

```rust
#[cfg(test)]
mod tests {
    use quasar_svm::{ExecutionStatus, QuasarSvm};
    use solana_account::Account;
    use solana_pubkey::Pubkey;

    fn setup() -> QuasarSvm {
        let elf = include_bytes!("../target/deploy/my_program.so");
        QuasarSvm::new()
            .with_program(&Pubkey::from(crate::ID), elf)
    }

    #[test]
    fn test_initialize() {
        let svm = setup();
        let payer = Pubkey::new_unique();

        let result = svm.process_transaction(
            &[instruction],
            &[(payer, Account::new(10_000_000_000, 0, &system_program))],
        );

        match result.status() {
            ExecutionStatus::Success => {}
            ExecutionStatus::Err(e) => panic!("failed: {e}"),
        }
    }
}
```

Key differences from LiteSVM:
- Load the program with `QuasarSvm::new().with_program(id, elf)` instead of `svm.add_program(id, bytes)`
- Pass initial account state as `(Pubkey, Account)` tuples directly to `process_transaction` — no separate airdrop step
- `result.status()` returns `ExecutionStatus::Success` or `ExecutionStatus::Err`, not a `Result`
- Check CU consumption with `result.compute_units_consumed` to assert performance budgets
- Chain multi-step tests by feeding `resulting_accounts` from one call into the next

Run with `anchor build` first so the `.so` is on disk before the `include_bytes!` compiles.
