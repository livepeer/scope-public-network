---
name: deploying-scope-public-orchestrator
description: Use when an agent (Claude, Codex) deploys a Scope live-runner orchestrator ("public O") on the Livepeer public network with this Docker Compose bundle, or when a deployment looks up but sessions produce no video, or when validating that an orchestrator is genuinely online and healthy rather than just reporting green.
---

# Deploying a Scope Public Orchestrator

## Overview

This repo's Compose bundle runs Scope as a **live runner** behind a go-livepeer
orchestrator, with Caddy for public TLS. An agent can deploy and validate it
unattended.

**Core principle: a Scope O can register, advertise capacity, accept a session,
and even bill the client — while producing zero frames.** `/discovery` returning
`200` with `capacity_available: 1` is **necessary, not sufficient**. The
deployment is healthy only when a real session produces output frames. Treat
"registered" and "discoverable" as *false-green* until proven by frames.

## When to Use

- Standing up a new public Scope orchestrator from this bundle.
- A deployment that "came up" but every session drops / shows no video.
- Asked to confirm an orchestrator is "online and healthy" — go past the
  surface checks in `README.md` (registration + TLS reachability), which a
  non-functional runner still passes.

## Deploy

Follow `README.md` Setup. The agent-relevant essentials:

1. `cp env.example .env`; set `DOMAIN`, `PUBLIC_SERVICE_ADDR` (`https://${DOMAIN}`),
   `ARBITRUM_RPC_URL` (a **working** Arbitrum RPC — see traps), `ETH_ACCT_ADDR`,
   `ORCH_SECRET`. Keep `LIVE_RUNNER_ADDR=http://go-livepeer:8935`.
2. Place the eth keystore + `.eth_secret` in the `livepeer-data` volume
   (`/root/.lpData/...`). The stack will not create them.
3. DNS for `DOMAIN` → host; ports `80`/`443` public. **No Cloudflare Tunnel**
   (buffering breaks live latency).
4. `docker compose config` then `docker compose up -d`.
5. The runner prefetches model artifacts into `scope-shared-data`
   (`/workspace/shared/models`) **before** registering. First boot is slow.

## Validate (run in order — stop at the first failure)

```bash
# 1. All services up; healthcheck must be 'healthy', NOT 'restarting'
docker compose ps

# 2. Orchestrator answers its own healthcheck
docker compose exec go-livepeer-healthcheck curl -fsS http://go-livepeer:8935/healthz

# 3. MODELS ACTUALLY DOWNLOADED (the #1 silent failure)
#    Must list real model dirs (e.g. Wan2.1-T2V-1.3B), not just 'lora'.
docker compose exec scope-live-runner ls -la /workspace/shared/models
docker compose logs scope-live-runner | grep -Ei "No artifacts defined|Prefetching|Failed to load pipeline"

# 4. GPU/build sanity — no CUDA kernel mismatch for this host GPU
docker compose logs scope-live-runner | grep -Ei "no kernel image is available|sageattention|CUDA error"

# 5. On-chain RPC reachable — log must NOT be flooded with 401s
docker compose logs go-livepeer | grep -Ei "401 Unauthorized|does not have access to this network"

# 6. Runner registered AND advertised (necessary, not sufficient)
curl -fsS "https://${DOMAIN}/discovery"   # expect a runner: app live-video-to-video/scope,
                                          # capacity_available >= 1, non-zero price_info

# 7. DECISIVE: a real session produces frames.
#    Drive a session through PUBLIC_SERVICE_ADDR (Scope client in Cloud mode, or
#    a gateway start_scope flow) and confirm output frames > 0 within ~60s.
#    Watch the runner during the session:
docker compose logs -f scope-live-runner   # expect pipeline load + frames, NOT
                                           # "in=none, out=none" / "stream disappeared"
```

Only steps 1–7 **all** passing — especially **3** and **7** — means healthy.
Steps 1, 2, 6 alone are the false green.

## Common Failures (observed)

| Symptom | Root cause | Fix |
|---|---|---|
| `/discovery` 200, `capacity_available:1`, but every session black-holes | Prefetch downloaded nothing — `/workspace/shared/models` holds only `lora/` | Step 3: look for `No artifacts defined`; use a `SCOPE_IMAGE` whose `download_models` populates the model registry; re-pull weights |
| Runner log: `no kernel image is available for execution on the device` | Worker image CUDA kernels not built for the host GPU arch | Use a `SCOPE_IMAGE` built for this GPU (e.g. sm_89 for 4090/L4) |
| go-livepeer log flooded `401 Unauthorized … does not have access to this network` | Bad/over-limited `ARBITRUM_RPC_URL` | Set a working Arbitrum RPC; on-chain ticket redemption needs it |
| Session reaped, media `in=none, out=none` / "stream disappeared" | Runner can't reach its trickle channels | Keep `LIVE_RUNNER_ADDR=http://go-livepeer:8935`; all services on one compose network |
| `insufficient balance, releasing session` after ~N s with no video | Billing is time×pixels, **not** output — a black-holing runner still drains balance | Fix frames (steps 3–4); do not mistake billing activity for health |
| Sessions laggy/drop behind a tunnel | Cloudflare Tunnel buffering | Use Caddy direct TLS (README) |

## Red Flags — do NOT report "healthy" if

- You only checked `docker compose ps`, registration logs, or `/discovery`.
- `/workspace/shared/models` contains only `lora/`.
- `No artifacts defined` appears in prefetch logs.
- No session has ever produced a frame (step 7 unproven).

Registration and discovery are claims the runner makes about itself. Frames are
the only proof.
