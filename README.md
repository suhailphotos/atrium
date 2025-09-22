# atrium

Home-level dotfiles managed with **GNU Stow**. This repo is for configuration folders and files that live directly under `~` (e.g. `~/.notionmanager`, `~/.oauthmanager`, `~/.incept`, `~/.parivaha`).

> For app configs under `~/.config` (XDG_CONFIG_HOME), use the separate repo **bindu**: https://github.com/suhailphotos/bindu.git

---

## Why a separate repo?

- **bindu** handles everything inside `~/.config` (XDG).
- **atrium** handles **non-XDG** items that sit at the top of your home directory.

Keeping them separate keeps the mental model clean and avoids accidental overlap.

---

## Recommended layout

Each *package* directory contains the actual hidden path it owns. Example:

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

This layout lets Stow link a package into `~` cleanly: `~/.notionmanager -> atrium/notionmanager/.notionmanager`.

---

## Manual usage (GNU Stow)

### Install Stow
- macOS: `brew install stow`
- Ubuntu/Debian: `sudo apt-get install stow`

### Clone and link
```bash
git clone https://github.com/yourname/atrium.git
cd atrium

# Link select packages into $HOME
stow -t "$HOME" notionmanager oauthmanager incept parivaha
```

### Update after edits
```bash
# Restow re-evaluates links for the package(s)
stow -R -t "$HOME" notionmanager
```

### Remove links
```bash
# Unstow removes the symlinks from $HOME (files stay in the repo)
stow -D -t "$HOME" oauthmanager
```

### Adopt existing files (pull from $HOME into the repo)
```bash
# Moves matching files from $HOME into the repo, then links them back
# Dry-run first, then do it for real
stow -n -v --adopt -t "$HOME" notionmanager
stow -v --adopt -t "$HOME" notionmanager
```

### Add a new package

Say you want to manage `~/.parivaha`:

- **If `~/.parivaha` already exists** (adopt it):
  ```bash
  mkdir -p parivaha
  stow -n -v --adopt -t "$HOME" parivaha   # dry-run
  stow -v --adopt -t "$HOME" parivaha      # real
  # Result: ~/.parivaha -> atrium/parivaha/.parivaha
  ```

- **If starting fresh** (no existing folder in `~`):
  ```bash
  mkdir -p parivaha/.parivaha
  # Put files in parivaha/.parivaha/…
  stow -t "$HOME" parivaha
  ```

### Rename a package or its target folder

There are two names involved:
- The **package** (e.g., `notionmanager/`)
- The **hidden folder it owns** (e.g., `.notionmanager/`)

**Rename only the package (keep same target):**
```bash
stow -D -t "$HOME" notionmanager
git mv notionmanager notion
stow -t "$HOME" notion
# Still links ~/.notionmanager -> atrium/notion/.notionmanager
```

**Rename the *target* hidden folder** (e.g., `.notionmanager` → `.notion`):
```bash
stow -D -t "$HOME" notionmanager
git mv notionmanager/.notionmanager notionmanager/.notion
git mv notionmanager notion     # optional
stow -t "$HOME" notion
# Now ~/.notion -> atrium/notion/.notion
# If ~/.notion already exists as a real dir, use --adopt
stow -v --adopt -t "$HOME" notion
```

### Restow on a fresh machine

If `$HOME` already has real folders (created before cloning this repo):
```bash
# Dry-run
stow -n -v --adopt -t "$HOME" incept notionmanager oauthmanager parivaha
# Do it
stow -v --adopt -t "$HOME" incept notionmanager oauthmanager parivaha
```

If they’re already **symlinks** pointing into `atrium`, a simple restow works:
```bash
stow -R -t "$HOME" incept notionmanager oauthmanager parivaha
```

### Ignore junk & macOS Finder files

Create a `.stow-global-ignore` at the repo root so Stow never tries to link these:
```
(^|/)\.DS_Store$
^\.stow-(local|global)-ignore$
```

Also add `.DS_Store` to `.gitignore` to keep the repo clean.

---

## Automation with Ansible

A dedicated role `atrium_stow` stows packages from `{{ matrix_root }}/atrium` into `$HOME` on macOS and Linux.

- **Graceful skip**: if `{{ matrix_root }}/atrium` doesn’t exist (e.g., Dropbox not installed/synced yet), the role prints a one-line skip and does nothing.
- **Idempotent**: if links are already correct, subsequent runs do nothing.
- **Noise-proof**: role creates `.stow-global-ignore` and cleans stray `\.DS_Store` so Finder files never cause conflicts.

### Role variables (defaults)

```yaml
# roles/atrium_stow/defaults/main.yml
matrix_root: "{{ lookup('env','MATRIX') | default((ansible_system == 'Darwin') | ternary(ansible_env.HOME + '/Library/CloudStorage/Dropbox/matrix', '/home/' + ansible_user + '/Dropbox/matrix'), true) }}"
atrium_root: "{{ matrix_root }}/atrium"

# list of package folder names under {{ atrium_root }}
# e.g. ['incept','notionmanager','oauthmanager','parivaha']
atrium_stow_packages: []

# first-run on a fresh box? set true to pull any existing ~ files into repo
atrium_use_adopt: false

# pass -R to restow (re-evaluate links)
atrium_restow: false

# present => stow, absent => unstow
atrium_state: "present"
```

### Configure which packages to stow

`group_vars/macos.yml`:
```yaml
atrium_stow_packages:
  - incept
  - notionmanager
  - oauthmanager
  - parivaha

# For a brand-new box with real dotfolders already in ~, enable adopt once:
atrium_use_adopt: true     # turn back to false after first run
```

`group_vars/linux.yml` (similar):
```yaml
atrium_stow_packages:
  - incept
  - notionmanager
  - oauthmanager
  - parivaha
atrium_use_adopt: true
```

### Add the role to your playbooks

`playbooks/macos_local.yml`:
```yaml
roles:
  - { role: atrium_stow, tags: ['atrium','stow'] }
  # …the rest
```

`playbooks/linux_remote.yml`:
```yaml
roles:
  - { role: linux_bootstrap,   tags: ['linux_bootstrap','bootstrap'] }
  - { role: bindu_config_repo, tags: ['bindu','config'] }
  - { role: atrium_stow,       tags: ['atrium','stow'] }
  # …the rest
```

### How to run (Ansible)

- **macOS (local):**
  ```bash
  ansible-playbook playbooks/macos_local.yml -l quasar --tags atrium
  ```

- **Linux (remote):**
  ```bash
  ansible-playbook playbooks/linux_remote.yml -l nimbus --tags atrium
  ```

- **First-run adopt** (if real `~/.incept`, `~/.notionmanager`, etc. already exist):
  ```bash
  ansible-playbook playbooks/macos_local.yml -l quasar --tags atrium -e atrium_use_adopt=true
  # or on Linux
  ansible-playbook playbooks/linux_remote.yml -l nimbus --tags atrium -e atrium_use_adopt=true
  ```

- **Force restow** (re-evaluate links after reorganizing repo):
  ```bash
  ansible-playbook playbooks/macos_local.yml -l quasar --tags atrium -e atrium_restow=true
  ```

- **Unstow** (remove symlinks from `$HOME`, keep files in repo):
  ```bash
  ansible-playbook playbooks/macos_local.yml -l quasar --tags atrium -e atrium_state=absent
  ```

### Brand-new Mac bootstrap flow (safe defaults)

1) Run your bootstrap (fonts, base tools, etc.). The stow role will **skip** if `atrium` isn’t present yet:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/suhailphotos/helix/refs/heads/main/scripts/install_ansible_local.sh)" -- --sf_fonts
```

2) After Dropbox has synced `atrium`, stow:
```bash
# Just link packages (no adopt)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/suhailphotos/helix/refs/heads/main/scripts/install_ansible_local.sh)" -- --tags atrium
```

3) If the machine created real dotfolders in `~` already, do a one-time **adopt**:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/suhailphotos/helix/refs/heads/main/scripts/install_ansible_local.sh)" -- --tags atrium -e atrium_use_adopt=true
```

After that, you can run the stow tag any time without extra vars.

---

## Conventions

- One **package per tool**. The package name is plain (e.g., `notionmanager`), while the contents include the actual hidden folder (e.g., `.notionmanager/`).
- Target is always `$HOME` with `-t "$HOME"` (or via the Ansible role using `-t {{ ansible_env.HOME }}`).
- Keep unrelated files out of Stow’s view or ignore them with `.stow-local-ignore` / `.stow-global-ignore`.

Example `.stow-local-ignore` to place at the repo root:
```
^README\.md$
^LICENSE$
^\.git(ignore)?$
^\.stow-(local|global)-ignore$
```

---

## Troubleshooting

- **Conflict: “cannot stow … over existing target … and --adopt not specified”**  
  Use `--adopt` (manual) or `-e atrium_use_adopt=true` (Ansible), or remove/rename the conflicting file in `~`.

- **Nested directories (e.g., `.incept/.incept`)**  
  Ensure the package contains exactly one hidden folder (e.g., `incept/.incept/…`) and that the link in `~` points directly to it. If you see an extra nested level, move contents up one level in the repo, remove the extra folder, then restow.

- **macOS `.DS_Store` noise**  
  Keep `.stow-global-ignore` with a `\.DS_Store` rule at the repo root. The role also deletes stray `.DS_Store` and passes `--ignore` to Stow to avoid future conflicts.

---

## Notes

- Avoid overlapping ownership with **bindu**. If something belongs in `~/.config`, keep it in **bindu**.
- After a `--adopt` run, commit the changes to capture file moves. Always dry-run when unsure.
- Works on macOS and Linux.
