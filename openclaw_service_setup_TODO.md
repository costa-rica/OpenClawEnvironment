# OpenClaw Service Setup — TODO

See `openclaw_service_setup.md` for full plan and rationale.

**Convention:** Each task shows what will happen. Below grouped tasks, a **"Run as nick"** block
contains the exact command(s) to paste into the terminal.

---

## Setup

- [x] Create `openclaw_svc` user with home directory and bash shell
- [x] Configure npm user-local prefix for `openclaw_svc`
- [x] Install openclaw as `openclaw_svc`
- [x] Verify binary is available

**Run as nick:**
first:

```
sudo useradd -m -s /bin/bash -G nick openclaw_svc
```

Then

```bash
sudo -u openclaw_svc bash -c '
npm config set prefix ~/.npm-global
echo "=== npm prefix ===" && npm config get prefix
export PATH="$HOME/.npm-global/bin:$PATH"
echo "=== Installing openclaw ==="
npm install -g openclaw@latest
echo "=== Verifying binary ==="
~/.npm-global/bin/openclaw --version
'
```

---

## Runtime Footprint Mapping

- [x] List all installed files under the npm global prefix
- [x] Check for config files created
- [x] Check for local data / state files created
- [x] Check for any other dotfiles in home dir
- [x] Run openclaw once to trigger first-time setup (if any) and re-check all above paths — `.openclaw` dir created during install/verify run

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c '
  echo "=== npm-global files ==="
  find ~/.npm-global -type f | sort
  echo "=== config files ==="
  find ~/.config -type f 2>/dev/null | sort
  echo "=== local data files ==="
  find ~/.local -type f 2>/dev/null | sort
  echo "=== home dotfiles ==="
  ls -la ~/
'
```

- [x] Inspect `~/.openclaw/` — created automatically on first run; contains openclaw config/state. Understand contents before lock down.
  - Only file: `agents/main/agent/auth.json` (2 bytes, likely `{}` — empty/unset auth)
  - All dirs are `700`, file is `600` — strictly owner-only. The `g+rwX` lock down step will open these to the `nick` group.

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c '
  echo "=== .openclaw contents ==="
  find ~/.openclaw -type f | sort
  echo "=== .openclaw permissions ==="
  ls -laR ~/.openclaw
'
```

- [x] Document any env vars openclaw requires
  - `HOME=/home/openclaw_svc` — required, no home dir by default in sudo context
  - `XDG_RUNTIME_DIR=/run/user/$(id -u)` — required for systemd user service commands
  - `PATH` — must include `~/.npm-global/bin`
- [x] Identify any paths accessed outside `/home/openclaw_svc/` (tmp, /var, sockets, etc.)
  - `/tmp/openclaw/` — gateway writes its log files here (e.g. `openclaw-2026-02-19.log`)

---

## Onboarding & Gateway Startup

- [x] Run onboard wizard as `openclaw_svc` (quick setup; systemd daemon install skipped — no XDG session)

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c 'export HOME=/home/openclaw_svc && export PATH="$HOME/.npm-global/bin:$PATH" && openclaw onboard'
```

- [x] Enable linger so openclaw_svc has a persistent systemd user session at boot

**Run as nick:**

```bash
sudo loginctl enable-linger openclaw_svc
```

- [x] Install gateway as a systemd user service

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export XDG_RUNTIME_DIR=/run/user/$(id -u)
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw gateway install
'
```

- [x] Set gateway bind to loopback (secure — LAN access handled via SSH tunnel, not direct exposure)

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw config set gateway.bind loopback
'
```

- [x] Start the gateway service

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export XDG_RUNTIME_DIR=/run/user/$(id -u)
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw gateway start
'
```

- [x] Confirm gateway is listening on loopback only

**Run as nick:**

```bash
ss -tlnp | grep 18789
```

Expected: `127.0.0.1:18789` only — no `0.0.0.0`

- [x] Confirm web UI is accessible from Mac via SSH tunnel

**Run as nick (from Mac):**

```bash
ssh -f -N -L 18789:127.0.0.1:18789 nick@10.0.0.11
```

Then open: `http://localhost:18789/#token=<token>`

---

## Nick's Shell Convenience Function

- [x] Add `oclaw` function to `~/.zshrc` so gateway commands can be run without the full `sudo -u openclaw_svc bash -c '...'` boilerplate

**Add to `~/.zshrc` (after the gateway is confirmed working):**

```bash
# Openclaw alias to run with openclaw_svc user
unalias oclaw 2>/dev/null
oclaw() {
  sudo -u openclaw_svc bash -c '
    export HOME=/home/openclaw_svc
    export XDG_RUNTIME_DIR=/run/user/$(id -u)
    export PATH="$HOME/.npm-global/bin:$PATH"
    openclaw "$@"
  ' -- "$@"
}
```

Then reload: `source ~/.zshrc`

Usage: `oclaw gateway start|stop|restart`

**Note:** `unalias oclaw 2>/dev/null` is required to prevent a zsh parse error if the function is re-sourced while an alias of the same name is still live in the session.

---

## Pre-Lockdown Testing

Run openclaw interactively as `openclaw_svc` to explore behavior and establish a baseline before restrictions are applied. Repeat the same tests post-lockdown to verify results are identical (lockdown should not affect openclaw's runtime capabilities — only login access and group ownership change).

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c 'export HOME=/home/openclaw_svc && export PATH="$HOME/.npm-global/bin:$PATH" && openclaw'
```

While in the openclaw session, run these test prompts in order and note what succeeds or fails:

**Write tests — verify what openclaw can and can't write to:**
- `write a file called test_pre.txt to /home/openclaw_svc/ containing the word "success"` — should succeed (own home)
- `write a file called test_pre.txt to /tmp/ containing the word "success"` — should succeed (world-writable)
- `write a file called test_pre.txt to /home/nick/ containing the word "success"` — may succeed (nick group access)
- `write a file called test_pre.txt to /root/ containing the word "success"` — should fail (no access)
- `write a file called test_pre.txt to /etc/ containing the word "success"` — should fail (no access)

**Read tests — verify what openclaw can read:**
- `read the file /etc/passwd and tell me the first 3 lines` — world-readable, should succeed
- `read the file /etc/shadow` — should fail (root-only)

**After the session, snapshot what new files were created:**

```bash
sudo -u openclaw_svc bash -c '
  echo "=== new files in home ==="
  find /home/openclaw_svc -type f -newer /home/openclaw_svc/.npmrc | sort
  echo "=== files written to /tmp ==="
  find /tmp -user openclaw_svc 2>/dev/null | sort
'
```

- [x] Record which write/read tests passed and failed
  - `/home/openclaw_svc/` write: 1 (pass)
  - `/tmp/` write: 1 (pass)
  - `/root/` write: 0 (fail — expected)
  - `/etc/` write: 0 (fail — expected)
  - `/etc/passwd` read: 1 (pass)
  - `/etc/shadow` read: 0 (fail — expected)
  - results written to `/home/openclaw_svc/openclaw_lockdown_tests.csv` by Yoda

---

## Lock Down

- [x] Remove login shell from `openclaw_svc`
- [x] Transfer group ownership of home dir to `nick` group
- [x] Verify nick can read/write files under `/home/openclaw_svc/`
- [x] Extend permissions to any paths outside `/home/nick/` as discovered above
  - `/tmp/` — world-writable, no action needed
  - `/home/nick/` — openclaw_svc removed from `nick` group; no longer has read access (intended)

**Run as nick:**

```bash
sudo usermod -s /usr/sbin/nologin openclaw_svc
sudo chown -R openclaw_svc:nick /home/openclaw_svc
sudo chmod -R g+rwX /home/openclaw_svc
ls -la /home/openclaw_svc
```

---

## Post-Lockdown Testing

Repeat the same openclaw session and test prompts from Pre-Lockdown Testing. Results should be identical — lockdown restricts login access, not runtime filesystem permissions.

**Run as nick:**

```bash
sudo -u openclaw_svc bash -c 'export HOME=/home/openclaw_svc && export PATH="$HOME/.npm-global/bin:$PATH" && openclaw'
```

- [ ] Repeat all write/read test prompts and record results
  - `/home/openclaw_svc/` write:
  - `/tmp/` write:
  - `/home/nick/` write:
  - `/root/` write:
  - `/etc/` write:
  - `/etc/passwd` read:
  - `/etc/shadow` read:

- [ ] Verify nick can read/write files created by openclaw_svc
  ```bash
  ls -la /home/openclaw_svc/
  cat /home/openclaw_svc/test_pre.txt
  touch /home/openclaw_svc/nick_test.txt && echo "nick write access confirmed"
  ```

- [ ] Confirm `openclaw_svc` can no longer log in interactively
  ```bash
  su - openclaw_svc
  getent passwd openclaw_svc
  ```
  `su` expected: `su: Authentication failure` (no password set — auth fails before shell is reached)
  `getent` expected: line ending in `/usr/sbin/nologin` (second layer — shell is nologin)
  Note: `sudo -u openclaw_svc -s` bypasses nologin by using `$SHELL` from the environment — do not use it as a lockdown test.

---

## Decision & Next Steps

- [ ] Decide: run openclaw as `openclaw_svc` directly, or hand paths to `app_runner`
- [ ] Set up systemd service unit for whichever user will run openclaw
- [ ] Test openclaw runs correctly under the chosen restricted user
