# OpenClaw Service User Setup

## Goal

Install and run `openclaw` as a dedicated, restricted service user on Ubuntu — not as `nick` (the admin user). The intent is to follow the principle of least privilege for production use.

---

## Background

An initial `app_runner` system user was created with:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin -G nick app_runner
```

This user has **no home directory**, which causes problems for `npm install -g`:
- npm needs a home directory to store its config (`~/.npmrc`), cache (`~/.npm`), and the global package prefix
- Without a home dir, npm either fails or falls back to system paths that require root

Rather than fighting npm, the approach below creates a proper service user, installs openclaw naturally, maps exactly what it needs at runtime, then locks the user down.

---

## Chosen Approach: Dedicated Service User with Home Directory

### 1. Create the `openclaw_svc` user

```bash
sudo useradd -m -s /bin/bash -G nick openclaw_svc
```

- `-m` — creates `/home/openclaw_svc`, giving npm a real working environment
- `-s /bin/bash` — login shell enabled **during setup only**, removed afterward
- `-G nick` — added to the `nick` group for cross-access to `/home/nick/` resources

---

### 2. Switch to the user and configure npm

```bash
sudo su - openclaw_svc
```

Set a user-local npm prefix so global installs don't require sudo:

```bash
npm config set prefix '~/.npm-global'
export PATH="$HOME/.npm-global/bin:$PATH"
```

---

### 3. Install openclaw

```bash
npm install -g openclaw@latest
```

This installs into `/home/openclaw_svc/.npm-global/` as intended, with no permission workarounds.

---

### 4. Map the runtime footprint

After install, run these to understand exactly what openclaw reads/writes at runtime:

```bash
# Installed binaries and package files
find ~/.npm-global -type f | sort

# Config files
find ~/.config -type f 2>/dev/null | sort

# Local data / state
find ~/.local -type f 2>/dev/null | sort

# Any other dotfiles created
ls -la ~/
```

This inventory tells us:
- What paths must be readable/writable at runtime
- Whether openclaw respects `XDG_CONFIG_HOME` / `XDG_DATA_HOME` (important since `app_runner` has no home)
- What env vars may need to be set when running as a restricted user

---

### 5. Lock down the user post-install

Once the runtime footprint is understood:

```bash
# Remove login shell
sudo usermod -s /usr/sbin/nologin openclaw_svc

# Transfer group ownership to nick group so nick can read/write everything
sudo chown -R openclaw_svc:nick /home/openclaw_svc
sudo chmod -R g+rwX /home/openclaw_svc
```

Nick can now access all of `openclaw_svc`'s files via the `nick` group, and `openclaw_svc` can no longer log in interactively.

---

## Permissions Beyond `/home/nick/`

openclaw may need access to paths outside `/home/nick/` (e.g. system tmp dirs, `/var/`, or socket paths). These will be identified during the runtime footprint mapping step. Extend permissions on a case-by-case basis as discovered.

---

## Decision Point After Mapping

Once the runtime footprint is understood, decide:

- **Option A:** Run openclaw directly as `openclaw_svc` (shell removed, systemd service)
- **Option B:** Hand the relevant paths and env vars to the original `app_runner` user and run it there instead

The mapping step informs which option is practical.
