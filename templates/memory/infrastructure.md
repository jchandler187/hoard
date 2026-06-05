# memory/infrastructure.md  <!-- src:agent ts:2026-06-05 ttl:90d -->

## Node-1 (192.168.1.230)
- Hosts the OpenClaw gateway and main agent
- Loopback only — not exposed to LAN or internet
- Tailscale: node1.tail507e53.ts.net

## Node-2 (192.168.1.86)
- Intel N95, 16GB RAM, 466GB disk
- Runs: Docker, Netdata, n8n, development workspaces
- SSH key-only auth (`ssh node2`)

## Services
- OpenClaw gateway: 127.0.0.1:18789
- Pi-hole: DNS on port 53
- Ollama: local model inference
- Tailscale funnel routes: / (8089), /calendar (1880), /bills (1881)