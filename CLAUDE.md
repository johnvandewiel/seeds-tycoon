# Seeds Tycoon: instructions for Claude Code

Read `HANDOFF.md` first: it has the product decisions, game rules, data model and build order.

## Context
- Owner: John. Stack: Medusa.js v2 backend, Next.js storefronts, PostgreSQL (RLS tenant isolation), Caddy, Ubuntu with systemd services, self-hosted.
- The seed platform already has an affiliate module with referral codes and commission tracking. Extend it; don't build a parallel one. Ask where that code lives if it isn't in this repo.
- `prototype/index.html` is the approved Sunny Garden look and interaction. Match it when building the real dashboard.
- `design/*.dc.html` are design-canvas source files, kept for reference only. Don't build on that format.

## Rules
- Every number shown in the game comes from real data (clicks, orders, commissions). No fake progress in production.
- Server-side validation for harvest, quest claims and payouts. XP is an append-only, idempotent event log.
- Money in integer cents, EUR. Commission rate is snapshotted per commission.
- First-party cookies only. Hash IP/UA. Keep the public influencer shop adult-looking with an 18+ gate.
- Build in the order of HANDOFF.md section 8 and confirm open decisions (section 9) with John before hard-coding them.
