# OpenClaw Gateway Setup

Picks up after onboarding is complete (`openclaw onboard`).
All commands run as nick unless noted.

---

## 1. Verify gateway bind is loopback

Onboarding may set bind to loopback already. Confirm and enforce it:

```bash
sudo cat /home/openclaw_svc/.openclaw/openclaw.json | grep bind
```

Expected: `"bind": "loopback"`. If not, set it:

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw config set gateway.bind loopback
'
```

With loopback, the gateway only accepts connections from the server itself — not from the LAN
directly. Access from other machines is handled via SSH tunnel (see step 5).

If you later put nginx + TLS in front of the gateway, nginx will proxy from loopback,
so this setting stays as loopback regardless.

---

## 2. Enable linger for openclaw_svc

`openclaw gateway install` uses systemd user services, which require a persistent user session.
`loginctl enable-linger` makes that session available at boot without an interactive login:

```bash
sudo loginctl enable-linger openclaw_svc
```

This also creates `/run/user/1001` (the XDG runtime dir). Confirm it exists:

```bash
ls /run/user/$(id -u openclaw_svc)
```

Expected: permission denied (means it exists — nick can't read it, which is correct).

---

## 3. Install the gateway as a systemd user service

The `XDG_RUNTIME_DIR` env var must be set explicitly when running as openclaw_svc via sudo:

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export XDG_RUNTIME_DIR=/run/user/$(id -u)
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw gateway install
'
```

Expected: `Installed systemd service: /home/openclaw_svc/.config/systemd/user/openclaw-gateway.service`

---

## 4. Start the gateway and verify

```bash
sudo -u openclaw_svc bash -c '
  export HOME=/home/openclaw_svc
  export XDG_RUNTIME_DIR=/run/user/$(id -u)
  export PATH="$HOME/.npm-global/bin:$PATH"
  openclaw gateway start
'
```

Gateway takes ~8 seconds to come up after a start or restart. Confirm it is listening:

```bash
ss -tlnp | grep 18789
```

Expected: `127.0.0.1:18789` and `[::1]:18789` — no `0.0.0.0` entry. Then check the journal:

```bash
sudo -u openclaw_svc bash -c '
  export XDG_RUNTIME_DIR=/run/user/$(id -u)
  journalctl --user -u openclaw-gateway.service -n 20 --no-pager
'
```

Look for: `[gateway] listening on ws://127.0.0.1:18789`

---

## 5. Access the control UI from your Mac

The gateway only listens on loopback, so connect via SSH tunnel from your Mac:

```bash
ssh -f -N -L 18789:127.0.0.1:18789 nick@10.0.0.11
```

Then open in your browser (token is in `/home/openclaw_svc/.openclaw/openclaw.json` under `gateway.auth.token`):

```
http://localhost:18789/#token=<your-token>
```

The `-f` flag backgrounds the tunnel. If the tunnel shows "Connection refused" after a gateway
restart, wait ~10 seconds for the gateway to finish starting, then retry.

---

## Useful commands

Stop the gateway:
```bash
sudo -u openclaw_svc bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u) && export PATH="/home/openclaw_svc/.npm-global/bin:$PATH" && openclaw gateway stop'
```

Restart the gateway:
```bash
sudo -u openclaw_svc bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u) && export PATH="/home/openclaw_svc/.npm-global/bin:$PATH" && openclaw gateway restart'
```

Tail live logs:
```bash
sudo -u openclaw_svc bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u) && journalctl --user -u openclaw-gateway.service -f'
```
