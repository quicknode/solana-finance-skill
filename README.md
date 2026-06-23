# Solana Finance Plugin & Skill

![Quicknode Solana Finance Claude Skill](assets/banner.png)

A Claude Code plugin for creating and editing Solana projects, specifically Rust projects in Anchor and Quasar, with a focus on:

- Accuracy, consistency and security for financial software
- Maintainability, readability, and minimal code
- Current best practices including Rust/LiteSVM for testing

This skill has the **most stars of any ecosystem Solana Claude skill** and is based on production-code used by some of the largest programs on Solana. If you're new to Solana: Solana programs are called 'smart contracts' older blockchains, that's what this skill builds.

> [!TIP]
> If you find this plugin useful, please add a GitHub star above! 🙏

## What is this?

**Solana Finance** is a Claude Code skill that bundles the `solana` skill — a reusable instruction set Claude automatically applies when working on Solana code, or that you can invoke manually. Skills are triggered automatically based on context (e.g. opening an Anchor program or a Solana Kit client).

## Installation

### As a plugin (recommended)

Once the plugin is published, install it from the marketplace:

```
/plugin install solana-finance
```

### As a standalone skill

You can also install just the skill directly:

```bash
npx skills add https://github.com/quicknode/solana-finance-skill
```

This installs the skill to your Claude Code skills directory (`~/.claude/skills/`).

## Usage

Once installed, the skill automatically applies when Claude Code works on Solana/Anchor/Quasar projects. You can also invoke it manually:

```
/solana-finance:solana
```

(When installed as a standalone skill rather than via the plugin, invoke it with `/solana`.)

## Repository layout

```
.claude-plugin/plugin.json   # Plugin manifest (name, author, version)
skills/solana/               # The skill
  SKILL.md                   # Skill entry point + general guidelines
  ANCHOR.md  RUST.md  QUASAR.md  TYPESCRIPT.md   # Language/framework references
```
