# Summarizing Solana Programs

Rules for explaining a Solana program clearly and truthfully — in a README, a walkthrough, a video script, or an answer.

**These build on [SKILL.md](SKILL.md)** and apply whenever you explain a program, not only when you write one — in particular its _Fight for Truth_, _Terminology_, and _Writing About Financial Software_ sections. This file adds only the explanation-specific guidance not covered there.

## Explain through people and motivations

- **Cast the walkthrough with named people, each with a concrete motive.** A program becomes legible when you follow real actors doing real things, usually with a profit motive. Eg, nobody "wants to liquidate bad positions" - they want to "make money by liquidating bad positions".
- **Naming convention.** End _users_ take names from the start of the alphabet — **Alice, Bob, Carol, Dave** — and are ordinary users, never Solana developers. _Admins / operators_ take a name from later in the alphabet — **Maria**, or another late letter.
- **Every actor needs a motivation — including the operator.** If a participant has no reason to take part, the explanation (and probably the program) is incomplete. A canonical lending cast:
  - **Alice** wants to earn yield on her USDC.
  - **Bob** wants to stay long NVIDIA, so he borrows USDC against his NVDAx instead of selling it.
  - **Carol** wants to short SpaceX, so she borrows SPCXx and sells it.
  - **Dave** liquidates unhealthy positions for the discount.
  - **Maria** runs the market and earns the interest spread (the reserve factor).
- **Show opposing cases when the program is symmetric** (a long that wins and a short that gets liquidated teaches more than one happy path).

## Use real, recognizable assets

- **Only reference tokens that actually exist** — verify before naming one.
- **Prefer well-known real-world assets** the audience already understands, and gloss each on first use: **USDC** (US dollars), **NVDAx** (NVIDIA stock), **SPCXx** (SpaceX stock).
- **No memecoins**, and **avoid SOL** wherever possible — its price is meaningless to most people, whereas a dollar or a familiar stock is not.
- **Numbers are real and current, or clearly hypothetical.** Cite a price you state (real token, today's value, with a source); label any invented or future figure as illustrative. Keep token math concrete with base units and a worked example ("100 SPCXx at \$170 = \$17,000 collateral; 75% LTV ⇒ borrow up to \$12,750").

## Ensure math is correct

Use your preferred tool (Rust or Python) to ensure the math used during the scenarios described in the summary is correct.

## Get the mechanism and incentives right

- **Explain incentives correctly, and correct plausible-but-wrong mental models.** Example: liquidation is not a "crank fee" — the liquidator is a principal who repays debt from their own pocket and takes the borrower's collateral at a discount; the program can't sell collateral itself, so it pays outsiders to show up.
- **Contrast onchain vs offchain on what actually differs:** collateral instead of credit, rules-as-deployed-bytecode, algorithmic (utilization-based) rates, permissionless liquidation, oracle dependence, and the finality of bugs.
- **Get the account hierarchy right** — say what owns what (market → reserves → vault + share mint; obligations; per-obligation collateral vaults) rather than hand-waving "the program stores it."

## Show the state changes, step by step

A walkthrough is only legible if the reader can see what each call _does_ to onchain state. Render the program as a ledger, one instruction at a time.

- **Go call by call, in the order a real user invokes them, across the whole lifecycle.** Create → register → act → match → cancel → settle → withdraw. One instruction per step; don't skip the unglamorous setup or the closing settle/withdraw — those are where custody and incentives become real.
- **For each step, list the accounts it touches as `ADDED` (created here) or `UPDATED` (mutated here); for `UPDATED`, show only the fields that change, as `before → after`.** Don't reprint unchanged state — the diff is the point.
- **Label every account's address model and authority.** Mark it _off curve_ (a PDA — list its seeds) or _on curve_ (a plain keypair, client-generated), and name who signs for it. This is what makes custody legible: a vault keypair whose authority is a market PDA, or an order book held as a client keypair rather than a PDA, each tell the reader who can move what.
- **End every step with an explicit token-movement block** — `FROM → TO`, amount, and the reason (collateral lock, fee sweep, settlement payout). When nothing moves, write `none` and say why. Readers assume a match pays out; show them when it doesn't.
- **Separate accounting from custody.** A match or fill usually moves _numbers_ — credited/owed (unsettled) balances — while the tokens stay in the vaults until a later settle. State plainly where value is _owed_ versus where it has actually _moved_; conflating the two is the most common way a walkthrough lies.
- **Annotate the fee on every step, including the ones that generate none.** Show the arithmetic by hand (e.g. `250 × 3 = 750; 750 × 25 bps ÷ 10,000 = 1.875 → 1`) and state the rounding direction, deferring to _Onchain Financial Math → round in the protocol's favour_ in [RUST.md](RUST.md). "Fee generated: none — nothing filled" on a resting order is as informative as the one line where a fee is born.
- **Explain each non-obvious design decision the moment it first bites, in a sentence or two — not in a separate architecture dump.** Why an order book is a client keypair, not a PDA (it can be ~180 KB; a program can't allocate something that large inside a transaction, so the client creates it and the program verifies it is empty and program-owned); why a resting maker sets the fill price (the taker gets price improvement); why N fills sweep one fee transfer, not N.
- **Keep numbers small and hand-verifiable, and reconcile at the end.** Use quantities a reader can multiply in their head, so they can check each vault balance and confirm tokens-in == tokens-out — the same conservation check [RUST.md](RUST.md) requires of the program itself (_Assert token conservation_).

A single step in this format:

```
### Carol asks to sell 3 NVDAx at 245 — and matches Bob's resting 250 bid

ADDED — Order #3 (Carol)   [off curve — PDA, seeds: "order" + market + order_id]
    side: ask   price: 245 (her limit, NOT the 250 fill)   qty: 3   status: Filled

UPDATED — fee_vault        [on curve — keypair; authority = Market PDA]
    balance: 0 → 1 USDC                 ← the fee is born here

UPDATED — Bob MarketUser   [off curve — PDA]
    unsettled_base: 0 → 3               (owed to Bob, not yet paid out)

TOKEN MOVEMENT:
    Carol base ATA → base_vault   3 NVDAx   (taker collateral lock)
    quote_vault    → fee_vault    1 USDC    (one sweep after all fills)

Fee generated: 1 USDC  (250 × 3 = 750; 750 × 25 bps ÷ 10,000 = 1.875 → 1, rounded down)
```

## Name peers, kept current

Reference real comparable programs (e.g. **Kamino Lend**, **MarginFi**, **Save**) and use **current** names (Save, not Solend). Don't cite dead, hacked, or irrelevant projects, and don't reach to other ecosystems for the comparison.

## Push back on an incomplete program — don't disclaim it

- **If a program isn't production-ready, fix it, don't paper over it.** When a feature is missing or a participant has no incentive to take part (e.g. the market operator earns nothing), say so and offer to add it — a disclaimer is not a substitute for a working design.
- **Deliberate test-only scaffolding is different** and should simply be labelled as such — e.g. a mock price account standing in for a real Switchboard/Pyth feed in tests. Distinguish "missing functionality" (fix it) from "test stand-in" (document it).
- Ensure each step makes sense given previous steps - eg, Maria can't claim interest paid by Bob if Bob hasn't paid any interest yet. Add missing steps.

## Finish the summary with where everyone ended up

For example: "Alice earned a yield, Bob stayed long and won, Carol's short went against her, Dave got paid to clean it up, and Maria earned the spread on every dollar of interest the market produced."

## Shape

Lead with the outcome, then support it. Respect a length or time budget, cut anything that doesn't earn its place, and match the depth to the audience.
