# Running Scope on the Livepeer public network

This Docker Compose bundle runs Scope as a Live Runner behind an orchestrator,
with Caddy handling public TLS and Watchtower keeping container images current.

This differs from earlier orchestrator configurations in that the orchestrator
now requires a functioning SSL endpoint for live runner traffic. This example
includes Caddy which can acquire certificates automatically and handle renewals
without a separate TLS service.

Cloudflare Tunnels are not recommended for this setup due to the buffering
(latency) they introduce.

Scope instances register with the orchestrator at runtime. The orchestrator
advertises the runner’s capabilities and routes sessions to it.


```mermaid
sequenceDiagram
    participant Client
    participant Orchestrator as go-livepeer orchestrator
    participant Scope as Scope live runner

    Scope->>Orchestrator: Register live runner at startup
    Client->>Orchestrator: Request live video-to-video session
    Orchestrator->>Scope: Start live runner session
    Scope-->>Orchestrator: Processed media/events
    Orchestrator-->>Client: Live session output
```

## Files

- `docker-compose.yml` runs Scope, go-livepeer, Caddy, and Watchtower.
- `env.example` lists required runtime configuration.
- `go-livepeer.conf.template` is the single source of truth for the
  orchestrator config. It is rendered into `/tmp/go-livepeer.conf` inside the
  go-livepeer container at startup, and then `livepeer -config` runs that
  rendered file.
- `Caddyfile` terminates TLS for `DOMAIN` and proxies to go-livepeer.

## Setup

1. Copy the example environment and edit the values:

   ```bash
   cp env.example .env
   ```

2. Point DNS for `DOMAIN` at the Docker host and make sure ports `80` and `443`
   are reachable from the public internet.

3. Put the Livepeer Ethereum keystore and password file in the persistent
   `livepeer-data` volume. The defaults expect:

   ```text
   /root/.lpData/arbitrum-one-mainnet/keystore
   /root/.lpData/.eth_secret
   ```

## Secrets And Persistent Data

- The Compose file does not create the Livepeer keystore or `.eth_secret` for
  you. Those must already exist before `go-livepeer` starts.
- By default, `go-livepeer` reads both from the persistent `livepeer-data`
  volume mounted at `/root/.lpData`.
- If you already manage these files on the host, you can replace the named
  volume with bind mounts and point `ETH_KEYSTORE_PATH` / `ETH_PASSWORD_FILE`
  at those mounted paths instead.
- If you keep the named volume, populate it before first startup, for example
  by copying the keystore directory and `.eth_secret` into the volume with a
  one-off container or temporary mount.
- Keep the password file readable by the `go-livepeer` process inside the
  container and avoid storing either secret in the repo or other ephemeral
  container filesystems.

4. Validate the Compose file:

   ```bash
   docker compose config
   ```

5. Start the stack:

   ```bash
   docker compose up -d
   ```

6. Make sure Docker itself starts on boot on the host. On Linux hosts this
   usually means enabling the Docker service with:

   ```bash
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

   On Docker Desktop, enable the setting to start Docker when the host logs in.
   The Compose services already use `restart: unless-stopped`, so once the
   Docker daemon comes back after a reboot, the stack will come back
   automatically.

## Pricing

Both the Scope live runner and go-livepeer are configured with:

```env
TICKET_EV=800000000000
PRICE_PER_UNIT=5
PIXELS_PER_UNIT=995328000000
```

This corresponds to `$0.50/hour` which is what the gateways will pay.

## Public Routing

Caddy exposes `https://${DOMAIN}` and forwards traffic to `go-livepeer:8935`.
Scope registers its live runner internally at `http://scope-live-runner:8989`,
so go-livepeer can call it over the Docker network without exposing the runner
directly.

## Updates

Watchtower uses `nickfedor/watchtower` and runs with label filtering enabled.
Only containers with this label are eligible for automatic updates:

```yaml
com.centurylinklabs.watchtower.enable: "true"
```

The Watchtower container itself is explicitly labeled `false`.

## Staying Up

- All services in `docker-compose.yml` use `restart: unless-stopped`, so they
  restart automatically after container crashes and host reboots.
- Keep Docker configured to start automatically with the host. On Linux, verify
  this with `systemctl is-enabled docker` and `systemctl status docker`. On
  Docker Desktop, verify the startup setting is enabled. If Docker does not
  start at boot, the Compose restart policy cannot bring the services back.
- Do not store the Livepeer keystore or password file inside ephemeral
  container filesystems. Keep them in the persistent `livepeer-data` volume so
  go-livepeer can recover cleanly after restarts.
- The Caddy TLS state is stored in the persistent `caddy-data` and
  `caddy-config` volumes. Preserve those volumes so certificates and account
  state survive restarts.
- After planned maintenance or host reboot, confirm recovery with:

  ```bash
  docker compose ps
  docker compose logs --since=10m go-livepeer scope-live-runner caddy watchtower
  ```

- If you use external monitoring, alert on failed container restarts, `443`
  reachability, and inability to connect to the advertised `PUBLIC_SERVICE_ADDR`.

## Verification

After startup, check:

```bash
docker compose logs go-livepeer
docker compose logs scope-live-runner
docker compose logs caddy
docker compose logs watchtower
```

Expected signs of health:

- go-livepeer starts with `orchestrator true`, `useLiveRunners true`, and
  `network arbitrum-one-mainnet`.
- Scope logs show live-runner registration against `http://go-livepeer:8935`.
- `https://${DOMAIN}` reaches the go-livepeer service through Caddy.
- Watchtower logs show only labeled services are monitored.

## Testing

Test the operator-facing flow with a real Scope client:

1. Find the public service URI being advertised by the orchestrator. This
   should match `PUBLIC_SERVICE_ADDR` and be reachable over HTTPS.
2. Start Scope separately with the orchestrator URL pointed at that service:

   ```bash
   LIVEPEER_ORCH_URL=<service-uri> uv run daydream-scope
   ```

3. Open the Scope UI and choose Cloud mode.
4. Confirm the UI connects successfully and a request can be routed through the
   orchestrator-backed cloud path.

If this test fails, re-check the `go-livepeer`, `scope-live-runner`, and Caddy
logs, and confirm that DNS, TLS, and the advertised service URI all match.

On-chain registration, bonding, funding, and service URI transactions are still
operator-managed steps outside this Compose bundle.
