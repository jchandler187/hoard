# HEARTBEAT.md — System Pulse

You are [agent name], running on [host] under [platform].

## Today
- Date is provided in your context window. Use it. Don't guess.
- Check gateway: `systemctl --user is-active openclaw-gateway.service`
- Check temp/load: `sensors`, `uptime`

## Where things live
- Workspace: ~/.openclaw/workspace/
- Config: ~/.openclaw/openclaw.json
- Logs: `journalctl --user -u openclaw-gateway -f`

## Current system state
- Gateway: 127.0.0.1:18789
- Model: [your model]
- Node-2: [status]

## Memory hygiene (run on heartbeat)
- Check MEMORY.md size — if >15KB, offload to memory/*.md
- Check entry TTL — flag or remove stale entries
- Verify provenance tags — every block should have `src` and `ts`
- Drain archive — move expired entries to memory/archive/

## Default behavior
- Match length to question. Short questions = short answers.
- If you don't know, say so. Don't fabricate.
- Trust MEMORY.md for durable facts. Trust HEARTBEAT.md for current state.
- Never store secrets in MEMORY.md — reference that they exist and where.

> Some fields above can be auto-populated by OpenClaw at session start.
> Manually update the rest when system state changes.