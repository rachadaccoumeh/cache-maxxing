# Contributing to cache-maxxing

Thanks for helping. This guide covers setup, the rules every change keeps, and how a change gets in.

## Before you start

- **Bugs:** open an issue with the bug template. Include the surface (desktop app, VS Code, terminal), your Claude Code version (`claude --version`), the plugin version and a screenshot of the bar.
- **Features:** open an issue first, so we can agree on the behaviour before you write code.
- **Security problems:** don't open an issue; see [SECURITY.md](SECURITY.md).

## Setup

You need Claude Code 2.1.325 or later, `git`, and [`bun`](https://bun.sh) (only to regenerate `prices.json`).

```sh
git clone https://github.com/5m0k3/cache-maxxing.git
cd cache-maxxing
git checkout dev
./scripts/dev-sync.sh          # builds the dev copy, cache-maxxing-dev, into .claude/skills/
```

Open a Claude Code chat in this folder. It loads the **dev copy**, so your edits show up there without touching an installed release. If you also have cache-maxxing installed, disable it for this folder (`.claude/settings.json`: `"enabledPlugins": { "cache-maxxing@cache-maxxing": false }`). After each edit, run `./scripts/dev-sync.sh` again; open chats pick it up after `/reload-plugins`.

## Layout

| Path | What it is |
|---|---|
| `hooks/logic.ts` | Pure logic: prices, the rules, every text the bar shows, stats and update checks. No `$` calls, so it's tested directly. |
| `hooks/register.tsx` | Wiring: events, state, storage, drawing, the four commands. |
| `hooks/handoff.ts`, `hooks/handoff-skill.md` | `/handoff`: what is extracted, and what Haiku is told. |
| `types/index.d.ts` | The `$.state` contract. |
| `tests/` | `logic.test.ts` (exact texts), `handoff.test.ts`, `bar.test.tsx` (the real hooks, drawn on every surface). |
| `prices.json` | Generated from the bundled price table; what `/cache-update` downloads. |
| `scripts/` | `dev-sync.sh`, `release.sh`, `prices-json.ts`. |

## Rules every change keeps

1. **No network and no model calls**, except `/cache-update` (GETs of this repository's raw files, nothing sent) and `/handoff` (one `$.model.complete` to Haiku). `$.http` and `$.process` appear only in the update command, and `$.model` only in the handoff. Never `$.model.fork`.
2. **Never block a message.** Warn with a pop-up; never drop or rewrite what the user sends.
3. **Fail open.** A hook that errors must never get in Claude Code's way; storage is best-effort and the bar runs from memory.
4. **Identical text on every surface.** `tests/bar.test.tsx` draws the bar on desktop, vscode and terminal and requires the same text. A surface-specific change adds its own test and never edits the shared expectations.
5. **Suggestions, not commands.** The bar's tips are phrased as options ("Try /handoff …").
6. **Measure, don't guess.** If a number in the code comes from a measurement, say which in a comment.

## Making a change

1. Branch from `dev`: `git checkout -b fix/short-name dev`.
2. Write a failing test first where you can, then the change.
3. Run the checks:
   ```sh
   ./scripts/dev-sync.sh
   claude plugin validate .     # must show no TypeScript errors
   claude plugin test .         # every test passes
   ```
4. Commit with [Conventional Commits](https://www.conventionalcommits.org/): `fix: …`, `feat: …`, `docs: …`, `test: …`, `chore: …`. One logical change per commit.
5. Open a pull request **against `dev`** and fill in the template. Keep it small; include a screenshot for anything visible.

`main` holds released versions only. Users install `main`, so nothing lands there except through a release.

## Updating prices

Prices live in `PRICES` in `hooks/logic.ts`, as `[input, cache read]` per million tokens, from the [official pricing page](https://platform.claude.com/docs/en/about-claude/pricing).

1. Edit the row (or add one, keyed `family-major-minor`, e.g. `opus-5-5`).
2. Set `PRICES_DATE` to today: a downloaded table only overrides the built-in one if it is newer.
3. Run `bun scripts/prices-json.ts` and commit `prices.json` with the change.

Once on `main`, everyone who runs `/cache-update` gets the new prices, without a new release.

## Releasing (maintainers)

1. Merge `dev` into `main`, and bump `"version"` in `.claude-plugin/plugin.json`.
2. Add the version to `CHANGELOG.md`.
3. Run `./scripts/release.sh <version>`. It refuses unless it is on `main` with a clean tree, a fresh `prices.json`, a new `PRICES_DATE` for changed prices and passing tests. Then it tags, updates the local live copy, and pushes `main` and the tag.

## License

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE).
