# atrium

Home-level dotfiles managed with GNU Stow. Use this repo for configuration folders and files that live directly under `~` (for example: `~/.notionmanager`, `~/.oauthmanager`, `~/.incept`).

> For app configs under `~/.config` (XDG_CONFIG_HOME), use the separate repo **bindu**: https://github.com/suhailphotos/bindu.git

## Why a separate repo?
- **bindu** handles everything inside `~/.config` (XDG).
- **atrium** handles **non-XDG** items that sit at the top of your home directory.
Keeping them separate keeps your mental model clean and avoids accidental overlap.

## Recommended layout

Each package directory contains the actual hidden path it owns. Example:

```
atrium/
├── notionmanager/
│   └── .notionmanager/
│       └── config.toml
├── oauthmanager/
│   └── .oauthmanager/
│       └── settings.json
├── incept/
│   └── .incept/
│       └── config.yaml
└── README.md
```

This layout lets Stow link a package into `~` cleanly.

## Quick start

Install Stow:
- macOS: `brew install stow`
- Ubuntu/Debian: `sudo apt-get install stow`

Clone and link:
```bash
git clone https://github.com/yourname/atrium.git
cd atrium

# Link select packages into $HOME
stow -t "$HOME" notionmanager oauthmanager incept
```

Update after edits:
```bash
# Restow re-evaluates links for the package(s)
stow -R -t "$HOME" notionmanager
```

Remove links:
```bash
# Unstow removes the symlinks from $HOME
stow -D -t "$HOME" oauthmanager
```

Adopt existing files:
```bash
# Moves matching files from $HOME into the repo, then links them back
# Review the diff after this step
stow --adopt -t "$HOME" notionmanager
```

Dry run first (recommended):
```bash
stow -n -v -t "$HOME" notionmanager
# -n no-action, -v verbose: shows what would be linked
```

## Conventions

- One **package per tool**. The package name is plain (e.g., `notionmanager`), while the contents include the actual hidden folder (e.g., `.notionmanager/`).
- Target is always `$HOME` with `-t "$HOME"`.
- Keep unrelated files out of stow’s view or ignore them with `.stow-local-ignore`.

Example `.stow-local-ignore` to place at repo root:
```
^README\.md$
^LICENSE$
^\.git(ignore)?$
^\.stow-local-ignore$
```

## Notes

- Avoid overlapping ownership with **bindu**. If something belongs in `~/.config`, keep it in **bindu**.
- Commit after `--adopt` to capture the move. Test with a dry run when unsure.
- Works on macOS and Linux.
