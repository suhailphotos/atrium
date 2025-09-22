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

---

## How to add a new package

Say you want to manage `~/.parivaha`:

**If `~/.parivaha` already exists in your home:** (adopt it)
```bash
mkdir -p parivaha
# Dry-run first
stow -n -v --adopt -t "$HOME" parivaha
# Do it for real
stow -v --adopt -t "$HOME" parivaha
```
This moves `~/.parivaha` into `atrium/parivaha/.parivaha` and creates a symlink back in `~`.

**If starting fresh (no existing folder in `~`):**
```bash
mkdir -p parivaha/.parivaha
# Put files in parivaha/.parivaha/…
stow -t "$HOME" parivaha
```

> Tip: keep non-stow junk (like `.DS_Store`) out of Stow’s view using a `.stow-global-ignore` file at the repo root. Example patterns:
> ```
> (^)|/)\.DS_Store$
> ^\.stow-(local|global)-ignore$
> ```

---

## How to delete (unstow) a package

To remove symlinks from `~` while keeping files in the repo:
```bash
stow -D -t "$HOME" parivaha
```

To remove **and** also delete the files from the repo (be careful):
```bash
stow -D -t "$HOME" parivaha
git rm -r parivaha
```

---

## How to rename a package or target folder

There are two names involved:
- The **package name** (e.g., `notionmanager/`)
- The **hidden folder it owns** inside the package (e.g., `.notionmanager/`)

### Rename just the package directory (keep target the same)
```bash
# 1) Unstow the old package
stow -D -t "$HOME" notionmanager

# 2) Rename the package dir
git mv notionmanager notion

# 3) Restow under the new package name
stow -t "$HOME" notion
```
The symlink in `~` still points to `~/.notionmanager` because the inner folder name didn’t change.

### Rename the *target* hidden folder (e.g., `.notionmanager` → `.notion`)
```bash
# 1) Unstow the existing package so ~/.notionmanager is no longer linked
stow -D -t "$HOME" notionmanager

# 2) Rename the inner directory
git mv notionmanager/.notionmanager notionmanager/.notion

# (Optional) Rename the package directory too
git mv notionmanager notion

# 3) Restow the (possibly renamed) package
stow -t "$HOME" notion
```
You’ll now have `~/.notion` linked to `atrium/notion/.notion`.

> If `~/.notion` already exists as a real directory on disk, use `--adopt`:
> ```bash
> stow -v --adopt -t "$HOME" notion
> ```

---

## Restow on a fresh machine (source already exists)

Scenario: you cloned `atrium` and your `$HOME` already has real folders like `~/.incept` because you created them before cloning.

Use `--adopt` so Stow pulls them into the repo and replaces them with symlinks:

```bash
# Dry-run first to confirm
stow -n -v --adopt -t "$HOME" incept notionmanager oauthmanager parivaha

# Do it for real
stow -v --adopt -t "$HOME" incept notionmanager oauthmanager parivaha
```

If they’re already **symlinks** pointing into `atrium`, a simple restow works:
```bash
stow -R -t "$HOME" incept notionmanager oauthmanager parivaha
```

---

## Conventions

- One **package per tool**. The package name is plain (e.g., `notionmanager`), while the contents include the actual hidden folder (e.g., `.notionmanager/`).
- Target is always `$HOME` with `-t "$HOME"`.
- Keep unrelated files out of Stow’s view or ignore them with `.stow-local-ignore` / `.stow-global-ignore`.

Example `.stow-local-ignore` to place at repo root:
```
^README\.md$
^LICENSE$
^\.git(ignore)?$
^\.stow-(local|global)-ignore$
```

---

## Troubleshooting

- **Conflicts like “cannot stow … over existing target … and --adopt not specified”**
  Use `--adopt` *or* remove/rename the conflicting file in `~`.

- **Nested directories (e.g., `.incept/.incept`)**
  Ensure the package contains exactly one hidden folder (e.g., `incept/.incept/…`) and that the link in `~` points directly to it. If you see an extra nested level, move its contents up one level in the repo, remove the extra folder, then restow.

- **macOS `.DS_Store` noise**
  Create a `.stow-global-ignore` file with a `\.DS_Store` pattern, and add `.DS_Store` to `.gitignore`.

---

## Notes

- Avoid overlapping ownership with **bindu**. If something belongs in `~/.config`, keep it in **bindu**.
- Commit after `--adopt` to capture the move. Test with a dry run when unsure.
- Works on macOS and Linux.

