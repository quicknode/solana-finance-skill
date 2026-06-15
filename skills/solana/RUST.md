# Rust Guidelines (Solana programs)

These guidelines apply to any Rust program or crate that uses Solana dependencies — whether you build with Anchor, Quasar, or the native Solana crates. Read this alongside the general rules in [SKILL.md](SKILL.md), plus the file for your framework:

- **Anchor** → [ANCHOR.md](ANCHOR.md)
- **Quasar** → [QUASAR.md](QUASAR.md)

If a task touches more than one, read each.

## Project Structure

- **Never modify the program ID** when making changes (in `lib.rs`, and in `Anchor.toml` for Anchor projects)
- Create files inside the `state` folder for whatever state is needed
- Create files inside the `instructions` or `handlers` folders (whichever exists) for whatever instruction handlers are needed
- Put Account Constraints (the `#[derive(Accounts)]` structs) in instruction files, and name them ending with `AccountConstraints` rather than naming them the same thing as the function
- Handlers that are only for the admin should be in a new folder called `admin` inside whichever parent folder exists (`instructions/admin/` or `handlers/admin/`)
- Don't implement instruction handlers as methods on account structs. The function isn't modifying the account, and the handlers and structs work fine without it — so there's no reason to tie state to functions.

## Account Constraints

- Use a newline after each key in the account constraints struct, so the macro and the matching key/value have some space from other macros and their matching key/value.

- Account constraints must not be given the same name as functions, because structs cannot do anything - `PlaceBet` would be a misleading and silly name for a struct because a struct cannot place a bet. If this struct is Account Constraints for an Anchor instruction handler called `place_bet` (which is a bad default from Anchor), name the struct `PlaceBetAccountConstraints` or similar.

## Error Handling

- Return useful error messages
- Write code to handle common errors like insufficient funds, bad values for parameters, and other obvious situations
- **Bounds-check client-supplied lengths and indices before slicing.** `split_at`, `copy_from_slice`, and indexing panic on out-of-range input; an attacker-controlled length must be `require!`d against the actual slice length first so the program returns a clean error instead of aborting.
- **Use a named error enum, not `ProgramError::Custom(4)`.** Bare numeric custom errors are magic numbers; the client can't tell what failed.
- **Never accept a parameter and ignore it.** A handler that takes `decimals: u8` and then hardcodes `9` silently gives the caller something different from what they asked for. Either use the parameter or remove it from the signature.
- All arithmetic in onchain code is `checked_*` — never raw `+ - * /`. Solana's BPF doesn't trap on overflow in release builds; silent wraps are how hacks happen. `checked_*` returns `Option`; force the error with `.ok_or(MyError::MathOverflow)?`. Reserve `saturating_*` for cosmetic/UX display values, never for balances.

## Onchain Financial Math

Applies to any code touching money, balances, prices, shares, fees, or token amounts. These rules are non-negotiable.

- **Integers only — no floats, no fixed-point libraries.** Floats are non-deterministic across platforms (different validators could disagree on state). `fixed::types::I64F64`, `rust_decimal`, `bnum`-fixed-point and similar are also out — they add audit surface, burn compute, and hide the rounding/precision decisions you should be making explicitly. Token amounts are integers (minor units), prices are ratios of integers. The system is discrete. Production Solana AMMs (Orca, Raydium, Meteora, Saber, Phoenix) all use raw `u128`. If you find yourself reaching for a decimal type, stop — the right tool is `u128` with discipline.
- **Multiply before you divide.** `a * b / c`, not `(a / c) * b`. Division truncates; dividing first throws away precision permanently.
- **Use `u128` (or wider) for intermediate products.** `u64 * u64` overflows at ~1.8e19. Cast both operands to `u128` _before_ multiplying, then narrow the final result with `try_into().map_err(|_| MyError::MathOverflow)?`.
- **For price × quantity × rate, u128 may not be enough.** `price (u64) × quantity (u64) × rate (u64)` can reach ~5.4e58, which overflows u128 (~3.4e38). If your hot path multiplies three u64-scale values together, either bound your inputs tightly and document the proof, or hand-roll a U256 type (~200 lines of Newton-based arithmetic, as Uniswap V2 in Solidity and Saber in Rust do). Don't reach for a crate just for this.
- **Round in the protocol's favour, never the user's.** Value-to-share and share-to-value conversions: user gets floor, protocol gets ceil. Otherwise you leak 1 minor unit per transaction forever, and attackers will industrialise it.
- **Validate ranges before doing the math.** Reject zero inputs, `amount > balance`, ratios that would mint zero shares. Cheap, prevents the inflation/donation attack on empty pools and other whole bug classes.
- **Check invariants after the math, not just before.** "K must not decrease" on a swap, "total LP shares == sum of holdings", "reserves >= owed fees". Compute, then `require!()` the invariant.
- **Assert token conservation on every instruction that moves funds.** Before returning from any instruction that transfers tokens between vaults or accounts, verify that total value in == total value out. Sum all debits and credits and `require!` they balance. A one-line conservation check catches an entire class of drain bugs before they reach mainnet.
- **Decimals are tracked, not assumed.** USDC=6, SOL=9, SPL tokens vary. Use `transfer_checked` (carries decimals in the CPI). Reserves hold raw minor units; the UI does cosmetic conversion. Never hard-code `* 10^9`.
- **Oracle/price freshness is part of the math.** Check `last_updated_slot` and reject if older than N slots. A stale price means the calculation is wrong.
- **Checks-effects-interactions.** Update state before the token transfer CPI, not after.
- **Treat client-supplied values as adversarial.** If a handler takes `(amount_a, amount_b)`, verify each against onchain state, not against each other.
- **Test the branch the bug lives in.** Standard AMM/lending bugs sit in the _non-empty pool_, _post-swap_, _post-fee_, _rounding-edge_ branches. The happy path almost always works. Write the test that exercises the branch where the bug actually lives. In particular, test *both directions* of any symmetric flow: a swap suite that only ever trades A→B will never catch a wrong-variable bug in the B→A branch.
- **One major unit is `10u64.pow(decimals)` minor units.** `1u64.pow(decimals)` is always 1, and `n.pow(decimals)` is n^decimals, not n major units — both are real shipped bugs. n major units is `n * 10u64.pow(decimals)`, with `checked_mul`.
- **Time windows: write the comparison in words first, then test with a nonzero duration.** "Contributions are allowed while `now < start + duration`" — code exactly that sentence. Inverted deadline comparisons (`duration <= elapsed` where `elapsed < duration` was meant) ship to mainnet because the tests use `duration: 0`, where both directions degenerate to equality. Every deadline needs a test that warps the clock past the boundary and one that stays inside it.
- **LP shares use different formulas for first deposit vs subsequent.** First deposit: shares = `sqrt(amount_a * amount_b)` (geometric mean bootstraps the pool). Subsequent deposits: shares = `min(amount_a * supply / reserve_a, amount_b * supply / reserve_b)` (proportional to share-of-pool). Using the geometric mean for every deposit is a real, repeated bug — test both branches separately.
- **For integer sqrt, hand-code Newton's method on `u128`** (~15 lines, as Uniswap V2 in Solidity / Saber in Rust do). Don't reach for a fixed-point crate for one sqrt.
- **Slippage protection: accept a `min_output_*` from the user and verify before the CPI.** Swaps, deposits, and withdraws all need it. Without it, sandwich attackers steal value across the price gap they create.
- **Never silently clamp user input to balance.** If a user asks to swap 100 and you clamp to 80 because that's the balance, the user's slippage check passes against the wrong amount. Either fail the instruction or return the actual amount so the client can validate.
- **Use `transfer_checked`, never raw `transfer`.** `transfer_checked` carries the mint and decimals through the CPI, so a wrong-mint or wrong-decimals account causes a CPI failure instead of a silent miscalculation.
- **Write token code that works against both token programs.** The same code should run unchanged against the Classic Token Program and the Token Extensions Program. Use your framework's token-interface types to achieve this (Anchor: `anchor_spl::token_interface` — see ANCHOR.md). Always pair this with `transfer_checked`.
- **Oracle freshness uses slots, not unix time.** Slot count is what the runtime guarantees; `Clock::get()?.unix_timestamp` is validator-influenced. Check `last_updated_slot` against `Clock::get()?.slot` and reject if older than N slots. If you must use a unix timestamp (because the oracle only exposes one), state why in a comment.
- **Canonical pubkey ordering for two-asset pools.** Order mints so `mint_a.key() < mint_b.key()` (lexicographic on the 32-byte key). Same pool whether the user passes `(USDC, SOL)` or `(SOL, USDC)`. Enforce in the constraint, don't rely on the client.

### Compound interest

For interest, fees that roll up, or any "grows by a rate each period" quantity. The same integer-fixed-point discipline above applies; these rules are specific to the compounding.

- **Compute `(1 + r)^n` by exponentiation-by-squaring on a scaled integer — never `powf`, `exp`, or anything touching `e`.** `e` and `powf` are floats: non-deterministic and banned by the integers-only rule above. Hold `1.0` as a WAD-scaled integer (`ONE_WAD = 1_000_000_000_000_000_000` = 1e18, in `u128`), form the per-period factor `(1 + rate_per_period)` in the same scale, and raise it to the integer power of periods elapsed with square-and-multiply. This is exactly what the SPL token-lending lineage ships (Save/Solend, Port: `(1 + rate/SLOTS_PER_YEAR).pow(slots_elapsed)`), and it is *exact* discrete compounding up to one truncation per multiply — error `O(log n)` ULPs, far below token granularity.

```rust
// 1.0 in WAD fixed point. factor and result are u128 in this scale.
const ONE_WAD: u128 = 1_000_000_000_000_000_000;

// Multiply two WAD-scaled values, rescaling back down. Floors (truncates).
// Use a wider intermediate (U192/U256) if either operand can exceed ~1e19.
fn wad_mul(a: u128, b: u128) -> Option<u128> {
    a.checked_mul(b)?.checked_div(ONE_WAD) // multiply before divide
}

// (1 + rate_per_period)^periods, exact discrete compounding, O(log n) muls.
fn compound_factor(mut factor: u128, mut periods: u64) -> Option<u128> {
    let mut result = ONE_WAD; // 1.0
    while periods > 0 {
        if periods & 1 == 1 {
            result = wad_mul(result, factor)?;
        }
        factor = wad_mul(factor, factor)?;
        periods >>= 1;
    }
    Some(result)
}
```

- **Compound per slot, not per unix second.** Periods are slots elapsed (`Clock::get()?.slot - last_update_slot`), and the per-period rate is `annual_rate / SLOTS_PER_YEAR` — same reasoning as *Oracle freshness uses slots, not unix time*. Early-return when zero slots have elapsed.
- **Accrue lazily into a stored cumulative index, not per position.** Keep one `cumulative_borrow_rate` (and `last_update_slot`) on the market; multiply it by the compound factor on any touch; each position's owed amount is a ratio of indices. This is the universal Solana pattern (Save, Port, Kamino) — it makes accrual permissionless and O(1) per user.
- **Round the *result* against the user, and decide direction explicitly.** Interest owed **by** a user (borrow, fee) rounds **up** (`ceiling`); interest credited **to** a user (supply yield) rounds **down** (`floor`) — per *Round in the protocol's favour*. The factor's per-multiply truncation is negligible; the load-bearing rounding is converting the index to a token amount, so apply `floor`/`ceil` there. Rounding the wrong way here is the exact bug class behind the SPL token-lending disclosure (round-trip extraction; the fix replaced `round` with `floor`) — a rounding-in-the-user's-favour error is a drain, not a cosmetic.
- **`spl-math`'s `PreciseNumber` is a correct *reference* for the algorithm, but don't depend on it.** Its `checked_pow` is a clean square-and-multiply and the crate has no `unsafe`. However: `solana-program-library` was archived (read-only) in 2025 and the crate is unmaintained; `PreciseNumber` rounds **half-up** (it adds `ONE/2` before dividing), which can round *in the user's favour* — wrong for money; and it is CU-heavy (U256 internals). If you want its math, vendor the ~15-line square-and-multiply (it's Apache-2.0) into your own WAD helpers and control the rounding yourself, rather than taking a dependency on an abandoned crate whose default rounding you have to fight.
- **A truncated binomial/Taylor series is an acceptable *compute* optimisation, not a default.** For large slot gaps, exact `pow` is `O(log n)` full-width multiplies; Aave and Kamino instead expand `(1 + b)^n ≈ 1 + nb + n(n-1)/2·b² + n(n-1)(n-2)/6·b³`, which is `O(1)` and accurate because the per-slot rate `b` is microscopic (so `b⁴+` vanishes). If you do this, keep the same scaled-integer/checked discipline and ensure the truncation error rounds *against* the user (Aave's comment notes it deliberately undercharges borrowers / underpays suppliers).
- **If you accrue simple interest per touch and call it "compound", say so.** Multiplying principal by `(1 + r·Δt)` once per interaction (marginfi, Compound v2) only compounds *across* settlement events; over a long un-touched gap it under-accrues versus true `(1+r)^n`. That's a legitimate design, but document it as simple-per-accrual — don't claim exact compounding the code doesn't do (*Fight for Truth*).

### Escrows, Vaults, and Escape Hatches

- **Every vault stores who may withdraw, and every withdraw verifies it.** A vault whose PDA signs for any caller is an open drain: if the only signer in the withdraw instruction is the fee payer and the recipient is client-supplied, anyone can withdraw anything. Record the depositor/authority when assets enter, `require!` it (against a `Signer`) when they leave. "The README admits there is no authority" does not make the program acceptable as a reference.
- **Every escrow needs a cancel/withdraw instruction.** An escrow with no cancel locks abandoned offers forever — funds become unrecoverable when the counterparty disappears. The cancel must be callable by the maker (and only the maker) at any time before the trade settles.
- **Don't lazily create an account the wrong party would pay rent for.** Common bug: the taker's instruction lazily creates the maker's destination ATA (e.g. via Anchor's `init_if_needed`), so the taker pays the maker's rent. Either require the maker to pre-create their ATA or pass the rent payer explicitly.
- **Close accounts to whoever paid their rent.** The mirror image of the bullet above: when the taker settles an offer, the offer account and vault rent were paid by the maker, so `close = maker` — not `close = taker`, and never a caller-chosen unchecked account. When closing manually, move *all* lamports; leaving `minimum_balance` behind strands it forever at a PDA nobody can sign for.
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

### Pre-signed Message Authorization

If a handler accepts an offchain signature (ed25519, secp256k1 "sign in with Ethereum", session keys) as authorization, the signed message must commit to everything the signature authorizes:

- The message the program verifies must be reconstructed *onchain* from the program ID, the specific action, the amount, the recipient, and a nonce stored in program state — never accepted as an opaque client-supplied hash.
- Increment the stored nonce after each successful use, so a signature authorizes exactly one execution.
- A signature over an arbitrary 32-byte value the client provides authorizes nothing in particular and everything in practice: any old signature can be replayed forever, for any amount, to any destination.
- The signature check supplements the Solana-side authority check, it does not replace it. Keep the `Signer` + stored-authority comparison too.

## Account Binding

Framework type checks (Anchor `Account<T>`, Quasar `Account<T>`) only verify owner and discriminator. They do NOT link accounts to each other. Every account in a constraint struct must be bound to something:

- State accounts: `seeds` / `address` derivation, or `has_one` from another bound account.
- Vaults and ATAs: `has_one` from the state account that recorded them, or `associated_token::*` constraints.
- Per-user records: seeds that include the signer's key, so user A cannot pass user B's record with their own token account.
- Mints a state account stores (`usdc_mint`, `asset_mint_a`, ...): `has_one` in EVERY instruction that reads balances denominated in them. An unbound mint lets a caller substitute a junk mint whose vault is empty and skew any NAV/share calculation.
- Stored program addresses (swap router, oracle program): either enforce them where the CPI happens or delete the field. A stored-but-never-checked address is worse than none: readers assume it is enforced.

An account with no constraint and no handler-side check is a finding, not a style issue. When a program ships in multiple frameworks, port the FULL constraint set: a twin variant missing one `has_one` or one authority comparison is a security bug, not a porting shortcut.

## Config Validation

When an admin instruction accepts a configuration struct, don't just check that the fields are well-formed — check that they are financially safe. Write a `validate_config` function that rejects configurations which would make the program exploitable even if each field is individually valid:

- Verify that fee rates, margins, and limits don't combine to produce impossible states (e.g. liquidation fee + protocol fee > maintenance margin means the protocol can be insolvent by design)
- Verify that numeric bounds don't overflow the hot-path calculations that use them
- `require!` these properties when the config is accepted, not in every handler that reads it

This is the difference between "is this a valid u64" and "does this configuration make the program safe". Reject unsafe configs at init time.

## Cargo hygiene

- Run `cargo clean` after finishing with a Rust project. Build `target/` directories accumulate fast (multi-GiB per project, Anchor especially).
- If disk usage hits 85%, clean before doing more work.

## PDA Management

- Add `pub bump: u8` to every struct stored in PDA
- Save the bumps inside each when the struct inside the PDA is created
- **Include an instance discriminator in accounts that belong to a specific instance.** If your program manages multiple independent pools, markets, or vaults, store the instance's pubkey (e.g. the pool or market address) in each subordinate account and validate it on every instruction. Without this, an attacker can pass an account from one instance into an instruction for another — the types match, the framework won't catch it, and the math silently operates on the wrong state.

## System Functions

- When you need the time, use the `Clock` sysvar via `Clock::get()?`. (Framework-specific import notes are in ANCHOR.md / QUASAR.md.)
