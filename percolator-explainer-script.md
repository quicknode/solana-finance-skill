# Percolator — Explainer Script

**Format:** single-narrator video / podcast script
**Target length:** ~12 minutes (≈1,900 words at ~160 wpm)
**Audience:** developers comfortable with Solana finance; perps experience is helpful but not assumed — the script includes a short primer
**Source:** `github.com/aeyakovenko/percolator` — README + `spec.md` (v16.9.0)

> Bracketed lines are delivery/on-screen cues, not spoken.

---

### [0:00] Cold open

What happens to a perpetual futures exchange when the market melts down, positions go bankrupt, and there isn't enough money on the books to pay everyone who's in profit?

Most exchanges answer that with an *auto-deleveraging queue* — an ADL queue. When someone blows up and the insurance fund can't cover the loss, the exchange reaches into the order book, picks specific profitable traders on the other side, and force-closes their positions to absorb the damage. It works. But it's unpredictable. If you're the trader getting picked, you can't reason in advance about whether it'll be you, or how much you'll lose.

Percolator is a research project from Anatoly Yakovenko that asks: what if we replaced that queue with something you could actually reason about, audit, and run with no human in the loop?

### [0:45] First, the primer — perps, the players, and why this is hard

[on-screen: "Perpetual future = leveraged bet, no expiry date"]

Quick grounding, in case you don't live in this corner of the market.

A *perpetual future* — a "perp" — is a leveraged bet on a price that never expires. A normal futures contract settles on a fixed date. A perp just keeps going. You post a little collateral — say a few hundred dollars — and you control a much larger position, long or short. To stop the perp's price from drifting away from the real spot price, the exchange charges a small recurring payment called *funding*: when too many traders are long, longs pay shorts; when the crowd is short, shorts pay longs. That funding tether is the trick that lets a contract with no expiry date still track the underlying.

Offchain, this product is enormous. The perpetual swap was invented by BitMEX back in 2016, and today the biggest venues by volume are centralized exchanges — Binance Futures, Bybit, OKX. Perps are the single most-traded product in all of crypto.

Onchain, the marquee names mostly live on their own chains or rollups — dYdX, GMX, Hyperliquid. On Solana specifically, the well-known perps platforms are **Drift Protocol** and **Jupiter Perps**, with **Zeta** also in the mix. These are real programs, holding real money, matching real leverage right now.

So why is running one of these so hard? Because a perp exchange is three dangerous machines bolted together: a continuous market that never closes, a clearinghouse that guarantees every trade, and a risk engine that has to stay solvent straight through a crash. Three things go wrong, over and over.

One — *oracles*. The exchange doesn't magically know the "real" price; it trusts a price feed. Manipulate that feed and you can mint paper profit out of nothing. This isn't hypothetical on Solana: in 2022, an attacker pushed around the price of a thinly-traded token on Mango Markets and walked away with more than \$100 million.

Two — *leverage outruns liquidations*. When the market gaps, a leveraged position can go underwater faster than anyone can close it. Now the position owes more than its collateral is worth, and somebody else has to absorb that loss.

Three — *who absorbs it?* Every serious exchange keeps an insurance fund for exactly this. But when the fund is drained and there's still a hole in the balance sheet, the exchange has to claw the loss back from the profitable traders on the other side. That's the auto-deleveraging queue from the cold open — and that pick-a-victim mechanism is exactly what Percolator is trying to replace.

Hold onto those three problems — oracles, underwater positions, and who pays. Everything Percolator does is aimed squarely at them.

### [3:00] What it is

[on-screen: repo banner + "EDUCATIONAL RESEARCH — NOT AUDITED"]

First, the honest disclaimer, straight from the repo: Percolator is an educational research project. It is not production ready, it has not been audited, and you should not point it at real funds. And be precise about what it actually is: in the repo's own words, it's a "pure risk-engine library." It is *not* a deployed Solana program — there are no instruction handlers, no program id, no account decoder, no deployment manifest. It's the risk *math*, written in Rust and formally proven, meant to be embedded inside a wrapper program that you would write. But those ideas map directly onto the perps programs people build on Solana, which is why it's worth understanding.

Here's the framing. A perp exchange has two separate fairness problems, and Percolator solves each with its own mechanism:

One — *exit fairness*. When the vault is stressed and there isn't enough to go around, who gets paid, and how much?

Two — *overhang clearing*. When positions go bankrupt, how does the other side absorb the leftover loss without the market deadlocking?

The first is handled by a single number called **H**, the haircut ratio. The second is handled by two per-side indices called **A** and **K**. The two mechanisms — H and A/K — are independent and compose cleanly.

### [4:15] The core idea — profit is a junior claim

[on-screen: "Capital is senior. Profit is junior."]

Before the mechanisms, the one mental shift everything rests on.

Stop treating profit like money. On a stressed exchange, your unrealized profit isn't cash sitting in a drawer — it's a *claim* against a shared balance sheet, and that balance sheet might not be able to honor it in full.

So Percolator splits the vault into two tiers. Deposited capital is *senior*. Profit is *junior*. The non-negotiable invariant is one sentence:

No user can ever withdraw more value than actually exists on the exchange balance sheet.

That's the whole game. Senior capital is protected. Junior profit is only as real as the money behind it. Everything else is the machinery that enforces this without ever picking favorites.

### [5:05] Mechanism one — H, fair exits

[on-screen: the H formula]

Start with **H**, the haircut ratio. It answers "how much profit is real right now?" with a single global number that applies to everybody equally.

Roughly: take the vault's total value, subtract everyone's senior capital and the insurance fund. Whatever's left over is the residual that can back profit. Divide that residual by the total matured profit in the system, and you get `h`, a number between zero and one.

If the exchange is fully backed, `h` equals one — your profit is fully real, withdraw it all. If the exchange is stressed, `h` drops below one, and *every* profitable account sees the same fraction of its profit honored. No queue, no priority, no first-come-first-served advantage. The rounding is deliberately conservative, so the sum of everyone's haircut profit can never exceed what's actually in the vault.

There's a second piece that makes this an oracle-manipulation defense. Fresh profit doesn't count immediately. It lands in a per-account reserve and only "matures" into withdrawable profit after a warm-up period. So if an attacker spikes a price to mint themselves a huge paper gain, that gain is locked out of the haircut math and out of their withdrawable balance until the warm-up passes — by which point the manipulation is gone.

And critically: H only ever gates *profit*. It never touches deposited capital. If you're flat — no open position — your deposit is always fully yours. As losses settle and the buffer recovers, `h` rises again on its own. It's self-healing.

### [6:45] Mechanism two — A/K, fair overhang clearing

[on-screen: "A scales positions. K accumulates PnL."]

Now the harder problem. An account goes bankrupt. Two things have to happen: its position quantity has to come out of open interest, and any uncovered loss has to be spread across the traders on the opposite side.

A traditional ADL queue does this by *naming* counterparties and closing them. Percolator refuses to name anyone. Instead, each side of each market carries two global coefficients.

**A** is a scaling factor for everyone's position. When a liquidation pulls quantity out of open interest, `A` ticks down, and every account on that side shrinks by the exact same ratio. Nobody is singled out — the whole side deleverages a hair.

**K** is an accumulator. Every profit-and-loss event on that side — mark price moves, funding, and socialized bankruptcy losses — gets folded into this one running index. Each account reads its share off the index whenever it's next touched. So when a deficit gets socialized, `K` shifts, and every account absorbs the same loss per unit of position.

The beautiful part is the cost. Settling any single account is constant-time — just read the current `A` and `K` against the snapshot from when you last touched it. And because it's all global indices, the result is completely independent of what order accounts get processed in.

[on-screen: DrainOnly → ResetPending → Normal]

There's also a guarantee that the market always heals, through a three-phase cycle. When `A` shrinks past a precision floor, the side goes **DrainOnly** — no new positions, existing ones can only close. Once open interest hits zero, it goes **ResetPending** — the engine snapshots `K`, bumps the epoch, and resets `A` back to one. Then it returns to **Normal** and reopens for trading at full precision. No governance vote, no admin button. The state machine just makes progress.

### [9:05] How they compose

[on-screen: the two-column comparison table]

So you've got two independent tools. H handles exits, using pro-rata profit scaling, and it's triggered whenever someone withdraws or converts. A/K handles bankruptcy overhang, using pro-rata position and deficit scaling, triggered when a bankrupt account is liquidated. H recovers automatically as the residual improves; A/K recovers through that deterministic three-phase reset.

Put them together and you get the four properties the project is really selling: no user can withdraw more than exists, no user is ever singled out for forced closure, markets always recover, and flat accounts always keep their deposits.

One honest caveat the repo states plainly: the A/K fairness is *exact* for open-position economics, while H's fairness is exact only for the realized claims currently stored — not the theoretical "true" number you'd get if you force-settled every account in the world at once.

### [10:05] The deeper engine

[on-screen: spec.md scrolling — "Realizable Full Shared Cross-Margin v16.9.0"]

That clean two-mechanism story is the README. The `spec.md` goes much deeper, and it's worth knowing it exists. The current spec describes a "realizable full shared cross-margin" engine, and it adds one more idea on top of H and A/K.

The problem it solves: cross-margin lets profit on one position support another position in the same account — great for capital efficiency, the kind of UX you'd want. But it's dangerous if that profit came from a manipulated market. So the spec adds a *source-domain* rule. Profit from a given market is only usable up to the real, reserved money that the *losing* side of that same market can actually pay. The core invariant is: usable credit from a source can't exceed the realizable backing reserved for that source.

Translation: if an attacker pumps one asset, the paper profit they create there can never turn into spendable buying power across the rest of their portfolio. The fake gain stays trapped in the market that created it.

And this isn't just prose. The repo ships formal verification — on the order of 265 machine-checked proofs using Kani, including a "no-steal" theorem. You run them yourself with `cargo kani`. That's the real point of the project: a risk engine whose safety properties are *proven*, not just argued.

### [11:15] Close

So that's Percolator. Two simple, predictable mechanisms — a global haircut for fair exits, two global indices for fair bankruptcy clearing — sitting on one rule: profit is a junior claim, and you can never withdraw more than the exchange actually holds. No queues, no chosen victims, no admin intervention, and the math is formally verified.

It's research, not a product — read the disclaimer, don't trade on it. But if you're building perps risk logic on Solana, it's one of the clearest mental models out there for keeping an exchange solvent and fair when everything goes wrong.

[on-screen: repo URL + "Apache-2.0 · cargo kani"]

---

*Terminology note: this script follows the Quicknode Solana finance skill conventions — "programs" (not "smart contracts"), "onchain"/"offchain" unhyphenated, "instruction handlers" vs. "instructions", "transaction fees" (not "gas"), and no claims of being audited or production-ready, per the repo's own disclaimer. Identifiers named here (H, h, A, K, the per-account reserve, DrainOnly/ResetPending/Normal, `cargo kani`) are drawn directly from the repo's README and spec.md, and the "pure risk-engine library — no program id, no instruction handlers, no deployment manifest" framing is the README's own. The primer's external references are background context, not from the Percolator repo: the perpetual swap's origin at BitMEX (2016); the centralized venues Binance Futures, Bybit, OKX; the onchain venues dYdX, GMX, Hyperliquid; the Solana perps platforms Drift Protocol, Jupiter Perps, and Zeta; and the 2022 Mango Markets oracle-manipulation exploit (widely reported at more than \$100 million). Verify these names, figures, and the current Solana-platform landscape against a primary source before recording. These guidelines were applied and they do apply to this project.*
