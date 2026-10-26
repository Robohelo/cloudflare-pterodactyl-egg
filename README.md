# Cloudflare Tunnel (Token) – Pterodactyl Egg

A Pterodactyl egg that runs [`cloudflared`](https://github.com/cloudflare/cloudflared) with a **remotely-managed tunnel token** from the Cloudflare Zero Trust dashboard.

There is no interactive login, no `cert.pem`, no `config.yml` and no tunnel ID to copy around. You paste the token, start the server, and manage hostnames and target services in the Cloudflare dashboard.

- **Egg file:** `egg-cloudflare-tunnel-token-V1.0.json`
- **Egg version:** 1.0
- **Format:** PTDL_v2 (Pterodactyl Panel 1.x)

---

## Features

- Token-based setup, the method Cloudflare currently recommends for dashboard-created tunnels
- Token is passed via the `TUNNEL_TOKEN` environment variable, so it does not appear on the command line or in the process list
- Installs the standard `linux-amd64` or `linux-arm64` build of cloudflared (chosen automatically)
- Optional version pinning for reproducible installs
- Selectable transport protocol (`auto`, `quic`, `http2`)
- Console detects a successful start by waiting for the "Registered tunnel connection" log line
- Outbound connections only, so no inbound port needs to be opened

---

## Requirements

- Pterodactyl Panel with Wings (egg format `PTDL_v2`)
- A Cloudflare account with Zero Trust enabled (the free plan works)
- A **remotely-managed tunnel** created in the Cloudflare dashboard
- The node must be able to reach `github.com` (install only) and Cloudflare's edge (runtime)

---

## Installation

### 1. Create a tunnel in Cloudflare

1. Open the Cloudflare Zero Trust dashboard.
2. Go to **Networks > Tunnels** and create a new tunnel (type: Cloudflared).
3. Copy the tunnel **token**. It is a long string starting with `eyJ`.

### 2. Import the egg

1. In the admin area of the panel, open **Nests**.
2. Pick a nest (or create a new one, e.g. "Networking") and click **Import Egg**.
3. Upload `egg-cloudflare-tunnel-token-V1.0.json`.

### 3. Create the server

1. Create a new server using the imported egg.
2. Pterodactyl requires one allocation per server, but the tunnel itself does not use it.
3. Small limits are enough. cloudflared is light, so a few hundred MB of RAM and no disk-heavy needs are typical.
4. After the install finishes, open the **Startup** tab and paste your token into **Tunnel Token**.
5. Start the server. When the console shows `Registered tunnel connection`, the tunnel is up.

### 4. Route traffic in Cloudflare

In the tunnel's **Public Hostname** tab, add a hostname and set its service, for example:

| Field     | Example                 |
|-----------|-------------------------|
| Subdomain | `app`                   |
| Domain    | `example.com`           |
| Type      | `HTTP`                  |
| URL       | `172.18.0.1:8080`       |

Cloudflare creates the DNS record automatically.

---

## Variables

| Name | Environment variable | Default | Description |
|------|----------------------|---------|-------------|
| Tunnel Token | `TUNNEL_TOKEN` | *(empty, required)* | Token of the remotely-managed tunnel. Keep it secret. |
| Transport Protocol | `TUNNEL_TRANSPORT_PROTOCOL` | `auto` | `auto`, `quic` or `http2`. Use `http2` if UDP/QUIC is blocked. |
| cloudflared Version | `CFT_VERSION` | `latest` | Release tag to install, e.g. `2025.10.0`, or `latest`. Applied on install or reinstall only. |

---

## How it works

- **Install script** (runs in `ghcr.io/parkervcp/installers:debian`): detects the CPU architecture, downloads the matching `cloudflared` binary from the official `cloudflare/cloudflared` GitHub releases, makes it executable and prints its version.
- **Startup command:** `./cloudflared tunnel --no-autoupdate run`. cloudflared reads the token from `TUNNEL_TOKEN`.
- **Stop signal:** `^C` (SIGINT), which lets cloudflared close its connections cleanly.
- **Runtime image:** `ghcr.io/parkervcp/yolks:debian`.

---

## Updating cloudflared

Auto-update is disabled inside the container. To update:

1. (Optional) set **cloudflared Version** on the Startup tab to the release tag you want, or leave it at `latest`.
2. Open the server's **Settings** tab and choose **Reinstall Server**. This re-downloads the binary and replaces `cloudflared`. Your token variable is kept.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| Install fails at the download step | The node cannot reach GitHub, or the version tag does not exist. Check connectivity, or set **cloudflared Version** to `latest` or a valid release tag. |
| Server stays in "Starting" | The "Registered tunnel connection" line never appeared. Check the console for token errors. |
| Authentication or "token" errors | The token is wrong, truncated, or was rotated. Copy it again from the Cloudflare dashboard and make sure there are no extra spaces. |
| Tunnel connects but the site shows 502 | Cloudflare cannot reach your target service. From inside this container, `localhost` is the container itself. Use the node's Docker gateway (usually `172.18.0.1`) or the node IP and the correct port. |
| Connection errors or timeouts at startup | UDP/QUIC may be blocked. Set **Transport Protocol** to `http2`. |
| Install fails with an image error | If a `parkervcp` image cannot be pulled, edit the egg and use `ghcr.io/pterodactyl/yolks:debian` as the runtime image. |

---

## Notes and limitations

- Designed for HTTP(S) services. Other protocols can be tunneled through Cloudflare, but visitors would need `cloudflared` on their side, so this is not suited for public game servers.
- Do not point a public hostname at the Pterodactyl panel, Wings, or any admin interface without protecting it, for example with Cloudflare Access.
- If you previously used the old login-based egg, tunnels created with `cloudflared tunnel create` are *locally* managed and are not compatible with this token method. Create a new remotely-managed tunnel instead.

---

## Security

- Anyone who has the token can run your tunnel. Users who can open the Startup tab can read it, so only give access to people you trust.
- If the token leaks, rotate it in the Cloudflare dashboard and update the variable.
- Review third-party eggs before importing them, since an egg's install script runs with write access to the server volume.

---

## Status

This egg has been checked for valid JSON and valid install-script syntax. It has **not** been tested on a live Pterodactyl panel by its author, so please test it on a non-critical server first and report anything that breaks.

---

## Credits

- **Written by:** Claude (Claude Sonnet 5.5, an AI model by [Anthropic](https://www.anthropic.com)), at the request of the repository owner
- **Date:** 2026-10-26
- **Based on:** the official [`cloudflared`](https://github.com/cloudflare/cloudflared) documentation for remotely-managed tunnels. Not affiliated with or endorsed by Cloudflare or Pterodactyl.
- **Author field in the egg:** `github@robohelo.xyz`
