---
name: solana-claude-skill
description: "Use when working on Solana software, including one or more of: Solana client code using TypeScript, Rust libraries that use Solana crates, Anchor programs, Quasar programs, LiteSVM tests, including Rust program files, TypeScript tests, and Anchor.toml or Quasar.toml configuration. Designed to create minimal, reusable code without unnecessary duplication."
---

# Coding Guidelines

Apply these rules to ensure code quality, maintainability, and adherence to project standards.

## Fight for Truth

Don't write things that aren't currently true — anywhere. Chat, code comments, variable names, PR titles, READMEs, commit messages.

- Documentation and comments that do not match the code are considered untrue.
- Variable names that do not match the purpose of the variable are considered untrue. For example, a struct called `InitializeMarket` is not true because a struct cannot 'initialize a market' - structs do not do things, only functions can do things.
- Temporary workarounds that aren't labelled as such are lying through omission - there is an issue you aren't telling the next programmer about. Mark them with a `TODO` comment with a link to a git issue (if it exists) and telling the next programmer when they can delete the workaround.
- If unsure of something, say so. Bluffing is lying.
- **Ambiguity is a soft lie:** if a phrase could be read two ways and only one is true, it's misleading. Disambiguate before sending — pick the term that says exactly what's meant, name the antecedent of every "it"/"this"/"that".
- A wrong statement is worse than no statement.
- Separate scratch labels from real identifiers.

Actively fix untrue things when you see them. Don't let "close enough" wording stand in for the truthful one.

**Grep before naming.** Before sending any prose, walkthrough, README, comment, or commit message that names a specific identifier (function, struct, file, account, module, field, constant), grep the source for that exact identifier and confirm it exists. "I'm pretty sure that's the name" is not enough. If the identifier doesn't exist, either use the real name or apply the rename to the code first, then write the prose.

**Describe what is, not what was removed.** READMEs, doc-comments, and code comments document current state — not history. Lines like "no floats", "no longer uses X", "replaces the previous Y approach" belong in CHANGELOGs and PR descriptions, not source artefacts. A first-time reader has no history and "no longer uses I64F64" creates ambient confusion ("wait, should I be worried?"). Sweep before sending: grep for `no longer`, `removed`, `previously`, `used to`, `formerly`, `dropped`, `now uses`, `replaces the previous` — each hit is a candidate for deletion.

## Do the whole thing

The marginal cost of completeness is near zero with AI. Do the whole thing.

Do it right. Do it with tests. Do it with documentation. Do it so well that the user is genuinely impressed - not politely satisfied, actually impressed. Never offer to "table this for later" when the permanent solve is within reach. Never leave a dangling thread when tying it off takes five more minutes. Never present a workaround when the real fix exists.

The standard isn't "good enough" - it's "holy shit, that's done." Search before building. Test before shipping.

Ship the complete thing. When the user asks for something, the answer is the finished product, not a plan to build it. Time is not an excuse. Fatigue is not an excuse. Complexity is not an excuse. Boil the ocean.

## Success Criteria

- Before declaring success, declaring that work is complete, or celebrating, run the project's actual tests using the correct command for that project (for example: `anchor test` for Anchor workspaces, the project's TypeScript test command for TypeScript clients/tests, or `cargo test` for Rust crates). If the tests fail, there is more work to do. Don't stop until the relevant test command passes on the code you have made.
- Create all program state through the program's own instruction handlers in tests. Injecting pre-fabricated accounts (hand-built account data passed straight into the SVM) hides missing init instructions and missing constraints - a program can pass every test while being unusable onchain. Pre-fabricated accounts are only acceptable for accounts a foreign program would have created (an oracle's price account, a mainnet-dumped fixture).
- Tests that use a zero or degenerate value for a parameter (e.g. `duration: 0`) test only the boundary where opposite comparisons coincide. Use nonzero, asymmetric values and test both sides of every boundary.
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
- **LiteSVM**: https://www.anchor-lang.com/docs/testing/litesvm
- **Anchor Error Codes**: https://raw.githubusercontent.com/coral-xyz/anchor/master/lang/src/error.rs
- **Quasar**: https://quasar-lang.com/docs
- **Solana Kite**: https://solanakite.org
- **Solana Kit**: https://solanakit.com
- **Agave (Solana CLI)**: https://docs.anza.xyz/ (Anza makes the Solana CLI and Agave).
- **Switchboard** (if used): https://docs.switchboard.xyz/docs-by-chain/solana-svm
- **Arcium** (if used): https://docs.arcium.com/developers

## Terminology

- Remember this is Solana not Ethereum. Ethereum is not relevant to any documentation you write. Do not assume people know or care about Ethereum.
  - Don't tell me about 'smart contracts' or 'protocols' (use 'programs' instead)
  - Don't tell me about 'gas' (use 'transaction fees' instead)
  - There are no 'mempools'.
  - Do not tell me about other things that are not relevant to Solana.

- Token program terminology:
  - Use 'Token Extensions Program' or 'Token extensions' for the newer token program (not 'Token 2022' which is just a code name)
  - Use 'Classic Token Program' for the older token program
  - Use 'Token' rather than 'SPL Token' unless you are specifically discussing the distinction between the native token (SOL) and all other tokens (SPL Tokens)

- Onchain / offchain (one word, no hyphen)
  - Always write 'onchain' and 'offchain' as single, unhyphenated words — like 'online' and 'offline'.
  - Never write 'on-chain' or 'off-chain'. The hyphenated forms are wrong.
  - Apply the same rule to related terms: 'crosschain' (not 'cross-chain'), etc.
  - Sources:
    - [Solana Foundation style guide](https://solana.com/docs/references/terminology)
    - [US Government usage](https://www.sec.gov/files/rules/interp/2026/33-11412.pdf)
    - [Cat (catmcgee) will make fun of you if you write 'on-chain'](https://x.com/catmcgee/status/2028153588715761825)

- Token amount units: a **major unit** is the human-scale denomination — dollar, pound, yen, SOL. A **minor unit** is the smallest denomination, the raw integer programs operate on — cent, penny, sen, lamport. Use these terms; do not say 'base units'.

- Some tools in Solana unfortunately use the same word 'instructions' for both the input and the functions. To avoid confusion, use 'instruction handlers' for the functions that handle instructions, and 'instructions' for the input to those functions.

## Do not use

- Do not use 'Solana Labs' documentation. The company has been replaced by Anza.

- Do not use 'Coral XYZ' documentation. Coral used to maintain Anchor, but Anchor is now maintained by the Solana Foundation (solana.org)

- Do not use any documentaton or tools from Project Serum, which collapsed many years ago.

- Do not use yarn. Yarn has no reason to exist and only adds unnecessary dependencies and is not commonly used for new JS/TS projects in 2026. Replace Yarn with npm everywhere you see it. Use npm for new projects as it does not require additional dependencies. Keep using pnpm if the project already uses pnpm.

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

### Documentation style

- **No numbered headings.** Headings are words only — no `## 1. Overview` or `### 3.6 Liquidation`. Numbered headings break when a section is inserted or removed.
- **No preview paragraphs.** Don't open a README or section with "the sections below cover X, Y, and Z" — the headings already do that.
- **Integrate per-instruction reference into lifecycle prose.** Walk through the program's flow and inline each handler's mechanics (signers, accounts, token movements, errors) at the point it's first called. Don't keep a separate flat "Instruction Reference" section.
- **Bold canonical terms on first use**, plain everywhere after — like a textbook.
- **No ASCII art, no Mermaid diagrams, no markdown tables.** Use headings, nested bullet lists, or prose. Tables don't render well on chat surfaces.
- **No em-dashes.** Use a regular dash or rewrite the sentence. Em-dashes are an LLM-output tell. This applies to READMEs, code comments, commit messages, and doc strings.
- **Don't say "worked example" or "worked scenario".** Just "Example", "Scenario", or "Walkthrough".

## Writing About Financial Software

These apply to READMEs, docs, blog posts, and PR descriptions for finance-related projects (AMMs, escrows, lending, leasing, CLOBs, prediction markets, stablecoins).

- **"Non-custodial" is a loaded word.** If the program locks funds in vaults during its lifecycle (every escrow, lending, AMM, leasing program does), don't claim "non-custodial" — it contradicts itself. What you usually mean is "no admin override, the rules are the deployed bytecode". Say that directly, or just describe the custody arrangement (program-owned vault, PDA signers, no admin escape hatch).
- **Upgrade authority is normal on Solana** — programs are usually upgradable so authors can ship security fixes. Don't apologise for it or treat it as disqualifying for "trustless" claims. Trust in the author/multisig is baseline; "trustless" means the documented rules can't be bypassed, not "bytecode frozen forever".
- **"Token" not "mint" in economic prose.** A mint is the onchain account that controls supply; a token is the asset. In economic descriptions ("post token A as collateral, borrow token B"), say "token A" and "token B". Reserve "mint account" for technical descriptions of what gets passed to instructions.
- **Tokens are fungible by default — don't say so.** Don't write "fungible token" or sentences explaining that tokens are fungible. The reader knows. Only qualify when contrasting ("non-fungible token" / NFT). Same rule as not explaining what an integer is.
- **Don't be fascinated with "tokenization" — a tokenized asset is just an asset.** Drop the word "tokenized" from economic prose. A "basket of tokenized assets" is a "basket of assets"; "tokenized stocks like TSLAx and NVDAx" are "stocks like TSLAx and NVDAx". The fact that an asset is represented by a token onchain is the baseline assumption of everything here, not a notable property. Only mention tokenization when the act of representing an offchain asset onchain is itself the subject (e.g. explaining how an issuer mints a token backed by a real-world asset).
- **One name per role/concept, enforced everywhere.** Pick a single term for each party (lessor/lessee, maker/taker, long/short, borrower/lender) and use ONLY that term throughout. Mixing terminology mid-document is how readers lose track of who owes what to whom.
- **Don't conflate "long the collateral" with "long the trade".** Anyone who posts collateral wants it to hold value (otherwise margin call), so every borrower is long their collateral. The directional bet is on the _borrowed_ asset, separately. Be precise about which "long" you mean.
- **Be careful with the word "securities".** It's a legal term. SOL is not a security. Asset-leasing is not "securities lending" even when the mechanics are analogous. Prefer "asset lending", "token lending", or "directional token lending" — and ask before picking one.
- **Spell out two-asset flows with concrete examples.** "Posts collateral and takes delivery of borrowed tokens" reads circular. "Posts USDC as collateral, borrows NVDAx" makes the asymmetry obvious. Don't make the reader infer that mints A and B are different things.
- **Name the instruction handlers in lifecycle prose.** When walking through "what the user does" (open position, close position, liquidate), name the actual handler (`take_lease`, `return_lease`, `liquidate`). Plain-English mechanics without handler names leave the reader unable to connect the narrative to the code.

## General Coding Guidelines

### You are a deletionist

Your golden rule is "perfection isn't achieved when there's nothing more to add, rather perfection is achieved when there is nothing more to be taken away".

Remove:

- Comments that simply repeat what the code is doing, or the name of a variable, and do not add further insight.
- Repeated code that should be turned into a named function.
- Unused imports, unused constants, unused files, and comments that no longer apply.

Before deleting "stale" scaffolding, confirm it is actually dead: grep the CI workflows (`.github/workflows/`) and package scripts for references. Test files and package.json scripts that look like leftovers are sometimes exactly what CI runs.
- Doc-comments whose first line just paraphrases the identifier. `/// Pool authority PDA.` above `pub pool_authority` is noise. Either explain something the name doesn't (seed derivation, mutability rationale, type-choice reason, an invariant the reader can't see from the type) or delete the line.

Don't remove existing comments unless they are no longer useful or accurate.

### Communication Style

- Do not make disclaimers about being a "complete project" or state what works
- It is expected that work is complete and functional - no need to state this explicitly
- Avoid phrases like "This is a complete implementation" or "All features are working"
- Just deliver the work without meta-commentary about its completeness

### Config files: leave a comment explaining WHY

When you change a configuration value, or pin a version in any config file (`Anchor.toml`, `Cargo.toml`, `package.json`, CI workflows, `.gitignore`, `rust-toolchain.toml`), leave a comment explaining _why_. The next reader needs the rationale, not just the value.

- **Pinned versions:** what breaks without the pin? when can it be unpinned?
- **Non-default timeouts / limits:** why this number?
- **Removed sections:** what was it doing? why was it removed?
- **`.gitignore` exceptions:** why is this file tracked despite the rule?
- **Workarounds:** what's the proper fix? when can this be replaced? (mark with `TODO`)

Example:

```toml
# Pinned: 0.8.7 conflicts with litesvm's dep tree.
# Unpin when litesvm upgrades its ahash requirement.
ahash = "=0.8.6"
```

When you remove a section, only add why to the git commit, so the file is free of information that does not apply to its existing state.

### Working with Generated or Unfamiliar Code

**CRITICAL - Verify Before Use:**

- Before calling ANY function whose signature you don't know with certainty, read the actual source code/type definitions first
- NEVER guess or assume what parameters a function accepts based on what seems logical
- Don't invent convenience parameters that don't exist
- Generated code, third-party libraries, and unfamiliar codebases often have different APIs than you expect
- Common mistake: Assuming a function accepts high-level parameters → WRONG. Check the actual signature in the source files first

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
- Avoid abbreviations, use full words (e.g., use `context` rather than `ctx`). Never use `e` for something thrown, use `thrownObject`, never use `v` when you mean `value`. There is almost no case where a single character variable is a good idea outside math (eg `p` and `q` for cryptography).
- Name a transaction some variant of `transaction`. Name instructions some variant of `instruction`. Name signatures some variant of `signature`. Do not confuse them - eg if the type looks like an instruction, you should not call it a 'transaction' because that is deceptive.

You can still add comments for additional context, just be careful to avoid comments that are explaining things that would be better conveyed by good variable naming.

### Code Quality

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

## Language-Specific Guidelines

The rules above apply to every file in the project. In addition, read the file that matches the language you are editing:

- **TypeScript** (Solana Kit clients, Solana Kit tests, browser code, anything `.ts`): see [TYPESCRIPT.md](TYPESCRIPT.md)
- **Rust — any Solana program/crate** (financial math, checked arithmetic, project structure, cargo hygiene): see [RUST.md](RUST.md)
- **Rust — Anchor** (`.rs` files using `anchor_lang`, LiteSVM tests): see [ANCHOR.md](ANCHOR.md), plus RUST.md for the shared rules
- **Rust — Quasar** (`.rs` files using `quasar_lang`/`quasar_spl`): see [QUASAR.md](QUASAR.md), plus RUST.md for the shared rules

If a task touches more than one, read each.

For setting up a Solana toolchain in CI or a fresh remote container (Agave, platform-tools behind TLS-intercepting proxies, the Quasar CLI, building Anchor projects without the anchor CLI): see [ENVIRONMENT.md](ENVIRONMENT.md).

## Git commits

Do not add "Co-Authored-By: Claude" or similar attribution when creating git commits.

Use the Linux/Git style `scope: description` for commit titles. [Do not use 'Conventional commits'](https://sumnerevans.com/posts/software-engineering/stop-using-conventional-commits/).

## Acknowledgment

- Acknowledge these guidelines have been applied when working on this project to indicate you have read these rules and found that they do apply to this project.
