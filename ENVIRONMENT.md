# Environment Setup (CI and remote execution)

How to get a working Solana toolchain in a fresh Linux container (CI runners, Claude Code on the web, devcontainers). Verified June 2026 against `quicknode/solana-program-examples`.

## Agave (Solana CLI) and cargo-build-sbf

```bash
curl -sSfL https://release.anza.xyz/stable/install -o /tmp/solana-install.sh
sh /tmp/solana-install.sh
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```

`cargo build-sbf` downloads its platform-tools toolchain on first use. In environments with a TLS-intercepting proxy this download fails with `invalid peer certificate: UnknownIssuer`, because `cargo-build-sbf` uses its own certificate store rather than the system's. Work around it by downloading with `curl` (which trusts the system store) and placing the archive where `cargo-build-sbf` caches it:

```bash
# Match the version cargo-build-sbf asks for in its error message (vX.YY).
PT_VERSION=v1.53
curl -sSfL "https://github.com/anza-xyz/platform-tools/releases/download/${PT_VERSION}/platform-tools-linux-x86_64.tar.bz2" -o /tmp/platform-tools.tar.bz2
mkdir -p ~/.cache/solana/${PT_VERSION}/platform-tools
tar -xjf /tmp/platform-tools.tar.bz2 -C ~/.cache/solana/${PT_VERSION}/platform-tools
```

Different tools pin different platform-tools versions (the Quasar CLI pins its own); repeat the download for each version requested in error messages.

## Anchor CLI

Install from crates.io (a long compile, roughly ten minutes):

```bash
cargo install anchor-cli --locked
```

`anchor test` reads the wallet path from `Anchor.toml` (`~/.config/solana/id.json` by default), so create a throwaway keypair once per container:

```bash
solana-keygen new --no-bip39-passphrase -s -o ~/.config/solana/id.json
```

In an existing repository the original program keypairs are usually not committed, so `anchor build` fails its program-ID check against the freshly generated keypair. Do NOT run `anchor keys sync` (it would change the program IDs, which this skill forbids); build with the check skipped, and test against the in-process SVM with no validator and no deploy:

```bash
anchor build --ignore-keys
anchor test --skip-local-validator --skip-deploy
```

(Verified: this runs the project's `test = "cargo test"` LiteSVM suite. Without `--skip-local-validator` anchor tries to spawn a `surfpool` validator; without `--skip-deploy` it tries to deploy to `localhost:8899`.)

### Without the anchor CLI

`anchor build` and `anchor test` are wrappers. If the anchor CLI is not installed, the equivalent is:

```bash
cargo build-sbf   # produces target/deploy/<name>.so
cargo test        # runs the LiteSVM tests, which include_bytes! the .so
```

Always rebuild the `.so` after changing program code, before re-running tests: the tests embed the binary at compile time via `include_bytes!`, so a stale `.so` silently tests old code.

## Quasar CLI

Quasar installs from git, not crates.io. Pin the exact commit your project's CI pins (branch tips move and regress):

```bash
git clone https://github.com/blueshift-gg/quasar /tmp/quasar
cd /tmp/quasar && git checkout <pinned-commit>
cargo install --path cli --locked
```

Then per project: `quasar build` (also regenerates the `target/client/rust/<name>-client` crate the tests import), `quasar test`. The CLI shells out to `cargo build-sbf --tools-version vX.YY`, so the platform-tools workaround above may be needed for that version too.

## Disk hygiene

Each Anchor build target is 1-2 GiB. Run `cargo clean` (and `quasar clean`) per project after its tests pass, especially when building many examples in one session.

## Repository session hooks

For repositories used with Claude Code on the web, put the steps above in a SessionStart hook or setup script so every fresh container gets a working toolchain without manual intervention. See https://code.claude.com/docs/en/claude-code-on-the-web for environment configuration.
