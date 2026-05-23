# opencode-mobile

A single OpenCode skill for running OpenCode from a phone with Clawdex Mobile, without requiring the phone and computer to be on the same Wi-Fi.

Start here:

- `skills/opencode-mobile/SKILL.md`

Highlights:

- no same-Wi-Fi requirement
- works on office/campus/hotel Wi-Fi and cellular networks through Cloudflare quick tunnels
- phone scans a QR instead of copying a changing tunnel URL
- bridge binds to `127.0.0.1` in remote mode
- query-token auth is disabled for safer public-tunnel use

This repository intentionally contains only reusable instructions and scripts. It does not contain bridge tokens, local `.env.secure` files, logs, or machine-specific secrets.
