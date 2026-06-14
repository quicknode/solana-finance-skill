# Summarizing Solana Programs

Rules for explaining a Solana program clearly and truthfully — in a README, a
walkthrough, a video script, or an answer. Derived from explaining a
Kamino/Solend-style lending program end to end.

## Ground everything in the code

- **Name only real identifiers.** Every handler, account, struct, and field you
  name must exist in the source — grep before you write it. "I'm pretty sure
  that's the name" is not good enough. A wrong name is worse than no name.
- **Walk the lifecycle by handler.** Map each step a user takes to the actual
  instruction handler and the key accounts it touches (e.g. "Alice calls
  `deposit_reserve_liquidity`; it moves USDC into the reserve's `liquidity_vault`
  and mints her share tokens"). Plain-English mechanics with no handler names
  leave the reader unable to connect the story to the program.
- **Get the account hierarchy right.** Spell out what owns what (market →
  reserves → vault + share mint; obligations; per-obligation collateral vaults)
  rather than hand-waving "the program stores it."

## Use the right words

- **Solana terminology, not Ethereum.** "Programs" not "smart contracts",
  "transaction fees" not "gas". No mempools. Don't explain Solana by analogy to
  chains the reader may not care about.
- **Cut needless jargon.** If a plain phrase works, use it. Drop terms like
  "money market" or "loan officer" when "lenders earn yield, borrowers borrow
  against assets" says it more clearly.
- **One name per role, enforced throughout.** Pick a single term for each party
  (lender/borrower/liquidator/owner) and never switch mid-explanation.
- **Describe what is, not what was.** Document current behavior, not history. No
  "no longer uses X" or "previously did Y" in a summary.

## Explain through people and motivations

- **Give each actor a name and a concrete reason to act.** "Alice wants yield on
  idle USDC"; "Bob won't sell his SpaceX, so he borrows against it"; "Dave
  liquidates for the discount." Personas make a protocol legible.
- **Every actor needs a motivation — including the operator.** If the party who
  runs the protocol has no way to make money, the economic model is incomplete;
  say how they earn (e.g. the reserve-factor spread) or flag that they don't.
- **Show opposing cases when the program is symmetric.** A long that wins and a
  short that gets liquidated teaches more than one happy path.

## Be honest about mechanism and incentives

- **Explain incentives correctly, and correct common misconceptions.** Don't let
  a plausible-but-wrong mental model stand. Example: liquidation is not a "crank
  fee" — the liquidator is a principal who repays debt from their own pocket and
  takes the borrower's collateral at a discount; the program can't sell
  collateral itself, so it pays outsiders to show up.
- **Contrast onchain vs offchain on what actually differs:** collateral instead
  of credit, rules-as-deployed-bytecode, algorithmic (utilization-based) rates,
  permissionless liquidation, oracle dependence, and the finality of bugs.

## Numbers and external facts

- **Real, current, cited — or clearly hypothetical.** If you cite a token or a
  price, verify it's real and current and cite the source. Mark any invented or
  future figure as illustrative; never present it as fact.
- **Keep token math concrete.** Use base units, decimals, and a worked example
  ("100 SPCXx at $170 = $17,000 collateral; 75% LTV ⇒ borrow up to $12,750")
  rather than vague proportions.

## Name peers, kept current

- Reference real comparable programs (e.g. Kamino Lend, MarginFi, Save) and use
  **current** names (Save, not Solend). Don't cite dead, hacked, or irrelevant
  projects, and don't reach to other ecosystems for the comparison.

## Scope, custody, and disclaimers

- **State simplifications plainly.** Distinguish the example from production: a
  test oracle vs a real Switchboard/Pyth feed, one market vs many, no fee tier,
  no audit. Don't let a reader mistake a teaching example for production-ready.
- **Describe custody precisely; avoid loaded words.** Don't claim "non-custodial"
  or "trustless" loosely. Say what's true: funds sit in program-owned vault PDAs,
  the owner can change risk params and collect fees but has no path to user
  funds, and the deployed rules can't be bypassed.

## Shape

- **Lead with the outcome**, then support it. Respect a length/time budget, cut
  anything that doesn't earn its place, and match the depth to the audience.
