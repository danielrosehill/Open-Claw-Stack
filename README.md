# Open Claw Stack

![Agentic AI MCP Model v3](diagrams/060426/processed/agentic-ai-mcp-model-v3.png)

This repository version controls documentation for a unified deployment stack bundling [OpenClaw](https://github.com/openclaw/openclaw) and [MetaMCP](https://github.com/metatool-ai/metamcp).

## Latest Iteration: Agentic AI MCP Model v3

The v3 model expands the stack into a multi-layered, multi-gateway topology designed to validate MCP discovery and tool routing across nested aggregation points.

### Clients

- **Desktop Client** — connects directly to the LAN VM when on-network.
- **Remote Client (Telegram)** — reaches the stack from outside the LAN via a Cloudflare Tunnel + Tailscale path.

### LAN VM (primary host)

The LAN VM hosts the core of the stack:

- **OpenClaw GW** — the user-facing gateway / chat surface.
- **Meta MCP 1** — the primary MCP aggregator that OpenClaw talks to. It fans out to both **local** (LAN) and **remote** (cloud) MCP backends.
- **Docker Network** — internal Docker bridge containing supporting services, including a **Postgres** instance used for conversation records.

### LAN Aggregator / GW (experimental)

A second-tier aggregator running on the LAN, reached locally from Meta MCP 1. It groups together purely on-LAN MCP sources:

- **Home Assistant**
- **NAS**
- **OpnSense**
- **SBC Aggregator** (experimental) — a further nested aggregator that itself fronts small single-board computers (**RPI 1**, **RPI 2**) exposing their own MCP endpoints.

Both the **LAN Aggregator GW** and the **SBC Aggregator** are experimental. The point of this layout is to validate that MCP tool discovery, naming, and invocation still work cleanly when traffic traverses **multiple gateway layers** (OpenClaw → Meta MCP 1 → LAN Aggregator → SBC Aggregator → device).

### Remote / Cloud MCP services

Meta MCP 1 also routes outbound to cloud MCP services over a Cloudflare path:

- **Google (Work)** and **Google (Personal)**
- **Replicate**
- **Pinecone** (RAG)
- **Meno** (memory)

### Meta MCP 2 (localhost)

A second MetaMCP instance pinned to **localhost** on the desktop, intended for tools that are cumbersome to run inside the VM or are better suited to on-device workloads — currently fronting **Blender MCP** and **Revit**.

### Aggregation layers (v3)

![v3 MCP Aggregation Layers](diagrams/060426/processed/aggregation-layers-v3.png)


In the v3 model an individual MCP server can sit behind **up to three nested gateway layers**:

| Layer | Gateway | Role |
|-------|---------|------|
| **L1** | **Meta MCP 1** (on the LAN VM) | First-level aggregator that OpenClaw GW talks to. Fans out to local and remote MCP backends. |
| **L2** | **LAN Aggregator / GW** (experimental) | Second-level aggregator reached locally from L1. Groups LAN-only sources (HA, NAS, OpnSense) and the SBC branch. |
| **L3** | **SBC Aggregator** (experimental) | Third-level aggregator nested under L2, fronting the small single-board computers (RPI 1, RPI 2). |

So in the worst case, a tool call from the desktop client traverses: **OpenClaw GW → Meta MCP 1 (L1) → LAN Aggregator (L2) → SBC Aggregator (L3) → device MCP**. The point of the experimental L2/L3 branch is to validate that MCP discovery, naming, and tool invocation remain coherent across this many hops.

### Suggested MCP servers

A non-exhaustive list of MCP servers that fit naturally into the v3 layout:

| Target | MCP server | Notes |
|--------|------------|-------|
| OpnSense | [Pixelworlds/opnsense-mcp-server](https://github.com/Pixelworlds/opnsense-mcp-server) | Sits behind the L2 LAN Aggregator. |
| Home Assistant | [tevonsb/homeassistant-mcp](https://github.com/tevonsb/homeassistant-mcp) | LAN-side, behind L2. |
| Blender | [ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp) | Pinned to Meta MCP 2 (localhost). |
| Pinecone (RAG) | [pinecone-io/pinecone-mcp](https://github.com/pinecone-io/pinecone-mcp) | Cloud, via remote branch. |
| Google Workspace | [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) | Cloud, via remote branch. |

### Design intent

- Keep private/LAN-only services strictly on the LAN side, behind nested aggregators.
- Keep cloud services proxied through a single hardened egress path.
- Keep host-bound creative tooling (Blender / Revit) on a separate localhost MetaMCP rather than pushing it through the VM.
- Use the experimental nested-aggregator branch to stress-test multi-hop MCP discovery before promoting it out of experimental status.

## Components

| Component | Purpose | Port |
|-----------|---------|------|
| **OpenClaw** | Personal AI assistant (gateway + CLI) | 18789 (gateway), 18790 (bridge) |
| **MetaMCP** | MCP aggregator/orchestrator/gateway | 12008 |
| **PostgreSQL** | Database backend for MetaMCP | 5432 |
| **Watchtower** | Automatic container image updates | — |
| **Cloudflare Tunnel** | Secure external access without port forwarding | — |

## Access Patterns

### On-LAN access

![On-LAN access](diagrams/060426/processed/access-onlan-v3.png)

When the desktop is on the home network it talks directly to the LAN VM (LAN IP / anti-hairpin DNS), hits OpenClaw GW, which fans out through Meta MCP 1 to LAN MCP servers and outbound to external/cloud MCP services.

### Remote access via Cloudflare

![Remote access via Cloudflare](diagrams/060426/processed/access-remote-cloudflare-v3.png)

From off-network, the client (e.g. Telegram) reaches the LAN VM via a Cloudflare Tunnel — no ports opened on the home network. From there the request lands on OpenClaw GW and follows the same internal aggregation chain.

---

Both projects use **Docker Compose** as their official deployment method. This stack combines them into a single `docker-compose.yml` so they can be deployed together on an Ubuntu server.

- **OpenClaw** runs as a gateway service exposing a web UI and bridge for channel integrations
- **MetaMCP** provides a unified MCP endpoint, aggregating multiple MCP servers behind a single proxy with a management UI
- **PostgreSQL** is required by MetaMCP for configuration and state storage

OpenClaw can be configured to use MetaMCP as its MCP endpoint, giving the assistant access to all MCP servers managed through the MetaMCP dashboard.

**Watchtower** monitors all running containers and automatically pulls updated images, keeping the stack current without manual intervention.

**Cloudflare Tunnel** (`cloudflared`) provides secure ingress to the services without opening ports on the server. Configure ingress rules in the Cloudflare Zero Trust dashboard to route your domain(s) to the OpenClaw and MetaMCP services.

## Deployment

Both components follow their upstream Docker deployment methods:

- **OpenClaw**: uses the official `openclaw` Docker image with volume mounts for config and workspace
- **MetaMCP**: uses the official `ghcr.io/metatool-ai/metamcp` image with a PostgreSQL dependency

### Setup

1. Clone this repo and create your `.env`:
   ```bash
   cp example.env .env
   ```
2. Set `CLOUDFLARE_TUNNEL_TOKEN` (create a tunnel in [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) > Networks > Tunnels)
3. Configure tunnel ingress rules to point your domains at `openclaw-gateway:18789` and `metamcp:12008`
4. Deploy:
   ```bash
   docker compose up -d
   ```

See the `docker-compose.yml` for the combined stack configuration.

## References

- [OpenClaw Docker docs](https://docs.openclaw.ai/install/docker)
- [MetaMCP Docker docs](https://docs.metamcp.com)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [MetaMCP GitHub](https://github.com/metatool-ai/metamcp)
