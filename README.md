# OpenClaw

This README.md notes are for humans. Intended to be short.

- other reference files are for both humans and agents and could be longer.

## Objective

The files in this repo document the process of running openclaw under a dedicated,
restricted service account — `openclaw_svc` — rather than as an admin user. The goal
is to limit the agent's access to only what it needs to operate, following the principle
of least privilege.

- ran on Ubuntu 24.04 LTS virtual machine

### References

- `openclaw_service_setup.md` — full plan and rationale for the openclaw_svc user approach
- `openclaw_service_setup_TODO.md` — step-by-step checklist tracking setup progress
- `openclaw_GATEWAY_setup.md` — how to configure and run the gateway as openclaw_svc
- `openclaw_lockdown_tests.csv` — pre/post lockdown test results for filesystem access
- `openclaw_lockdown_tests_blank.csv` — reusable blank template for lockdown testing

## Run OpenClaw

### Stop

```
sudo -u openclaw_svc bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u) && export
PATH="/home/openclaw_svc/.npm-global/bin:$PATH" && openclaw gateway stop'
```

### Start:

```
sudo -u openclaw_svc bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u) && export
PATH="/home/openclaw_svc/.npm-global/bin:$PATH" && openclaw gateway start'
```

## Access the GUI

1. tunnel: `ssh -N -L 18789:127.0.0.1:18789 nick@10.0.0.11`
2. open browser to: `http://localhost:18789/chat?session=agent%3Amain%3Amain`
