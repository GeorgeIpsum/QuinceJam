# QuinceJam: product scope and milestone plan

Date: 2026-10-31. Status: approved in conversation; this document records the product-level decisions that every milestone spec builds on.

## 1. What QuinceJam is

A FigJam replacement: an infinite collaborative whiteboard for diagramming, sticky-note brainstorming, personal thinking, and live workshops. Open source under AGPL-3.0, hosted at quincejam.com as a free-with-limits product, and self-hostable from one Docker image.

Audience, in priority order:

1. The owner and small teams (dogfood first).
2. The public, as a free product with limits at quincejam.com.
3. The owner as a learner: the architecture should exercise new technology (CRDTs, Rust to wasm, Durable Objects, novel database tooling) wherever that does not hurt the product.

Core uses in the first months: diagramming (flowcharts, architecture, sequence, mind maps), brainstorm and sticky walls, personal thinking space with text, images, links and ink, live workshops with others, and importing existing content from FigJam (via the Figma REST API) and from Mermaid.

## 2. Standing decisions

| Decision | Choice | Why |
|---|---|---|
| Canvas engine | Build our own scene graph, renderer, hit-testing and sync. Borrow small libraries (ink outlining, layout, CRDT). | Maximum learning; AGPL compatibility rules out source-available SDKs such as tldraw. |
| Document model | CRDT. Loro (Rust core, movable tree, movable list, LWW map, rich text) behind a sync-engine seam; Yjs is the fallback. | Offline merge, end-to-end-encrypted rooms, peer-hosted rooms and git-like history are only possible without server-side ordering. The server still validates, limits and compacts. |
| License | AGPL-3.0 | Deters wholesale resale by forcing resellers to publish changes. It does not prohibit resale; a trademark on the name is the complementary lever. Dependencies must be AGPL-compatible. |
| Deployment | Cloudflare Workers, Durable Objects, R2, Queues, Hyperdrive to Postgres. Workers Paid plan. | Owner's platform choice. |
| Self-hosting | One codebase. Production bundles run unchanged inside Miniflare (Cloudflare's workerd plus binding simulators) in Docker, with Postgres and S3-compatible storage (MinIO) as required companions. | Byte-identical bundles, no second implementation of any binding. |
| Monorepo | Turborepo 2.11 with experimental Cargo and Go workspaces, pnpm. | Owner's choice; TypeScript, Rust and Go are the languages. |
| Billing | None at launch. Plans, features, subscriptions, overrides and a usage ledger are modeled from day one with a null provider. | Adding Stripe or Polar later is a webhook handler, not a rewrite. |
| Programmatic control | First-class: in-app console, REST and WebSocket API (OpenAPI-first), MCP server, generated SDKs (TypeScript, Go, Rust, Python), CLI, webhooks, board-as-text. All share the UI's command core. | Parity between UI and automation by construction. |

## 3. FigJam parity: keep, defer, drop

The complete inventory (17 areas, 580 rows, with free/paid gating and build implications) is in `docs/research/2026-10-31-figjam-inventory-and-tech-landscape.md`. This section records the decisions.

**Canvas objects.** Keep: stickies, shapes (basic, flowchart, and the engineering set plus AWS/Azure/GCP/Kubernetes stencils, all free), straight/elbow/curved connectors with magnets, labels and ERD caps, rich text, stamps, emotes, marker, highlighter, images, GIFs, tables, sections, code blocks, link unfurls, oEmbed embeds, mind maps, groups, per-type color palettes and transforms. Defer: video upload, washi tape, sticker packs. Drop: Figma design-file paste and component libraries (foreign vector content becomes an opaque SVG or image). Widgets are a reserved node type whose runtime is the console sandbox.

**Interaction and navigation.** Keep all of it. Ship confirmed FigJam gaps from the start: minimap, lasso, select-similar by type, color and author, truly infinite canvas, optional snap-to-grid, Tab-to-create, align and distribute shortcuts. Defer: pages, cross-device clipboard, keyboard layouts beyond US plus two or three.

**Diagramming.** Keep shape libraries, quick-create chains, keyboard-driven diagramming, Mermaid import. Add Mermaid export and native semantic objects: swimlanes with pinned headers, kanban columns that auto-stack, sequence diagrams, Gantt and timeline. Org charts reuse the mind-map tree layout.

**Collaboration.** Keep presence, cursor chat, follow and spotlight, timer, voting, comments with mentions, link sharing with viewer/commenter/editor roles, password links, open sessions for anonymous guests, embeds, notifications. Voting, timer and deterministic sticky sorts are free. Add: facilitator role, anonymous mode with facilitator-only reveal, kick and ban. Defer: audio huddles (Cloudflare Realtime SFU). Drop: live captions, music, and the Slack, Teams, Zoom and Meet apps (webhooks and oEmbed cover those surfaces).

**Templates, widgets, plugins.** Keep built-in and section templates, a template picker, custom team templates (free). The extension model is the console: a sandboxed script is a plugin, a persisted scripted object is a widget. Poll and alignment scale are native objects. No community marketplace in v1, which keeps DMCA and moderation obligations out of v1.

**AI.** Keep: sort and summarize stickies, rewrite, translate and tone, text-to-diagram and mind map with auto-layout, board from prompt, an agent driving the board over MCP. Bring-your-own-key for free users; credit ledger from day one. Defer: image generation, voice-to-stickies, make-it-real. Drop: Jambot-style chat widgets, board-to-deck.

**Import, export, integrations.** Keep PNG, PDF and CSV export; add SVG, Markdown and JSON. Keep image, CSV and PDF import. Add FigJam import via the Figma REST API, Mermaid both ways, JSON Canvas and Excalidraw formats, a public oEmbed endpoint, a write REST API with fine-grained webhooks. Drop: Chrome extension, Jira and Asana widgets, Google Classroom, Zapier.

**Workspace and admin.** Keep: personal workspace per user, teams, nested folders, drafts, starred, recents, search, trash with restore, named versions with branching, ownership transfer, invites, plan and entitlement tables with a null billing provider. Seat types collapse into per-workspace entitlements. Defer: SSO/SAML, SCIM, domain capture, audit-log UI (the table exists), billing groups, connected folders. Self-host has a single-tenant mode.

**Account.** Email plus password, email OTP, magic links, passkeys, Google and GitHub, anonymous guests with link-on-sign-up, personal access tokens and OAuth apps, session list with log-out-everywhere, export-my-data and delete account. English only at launch. Drop: multi-account switching, TOTP (passkeys cover second factor).

**Mobile, accessibility, appearance.** Tablet PWA including Android and pen input; phones view-and-comment only. Keyboard navigation and an accessibility tree from the start. Reduced-motion support. Dark board, board background color, dot-grid toggle, hide UI. No native or desktop apps.

**Trust and operations.** Terms, privacy, acceptable use, abuse-report endpoint, security.txt, a retention table, status page, changelog, shortcuts panel, command palette, sample board on sign-up. Image moderation hook at upload before anything is publicly servable.

## 4. Additions beyond FigJam

Owner-requested: in-app console, REST and WebSocket API, MCP server, SDKs, CLI, self-hosting.

Accepted from brainstorming: board-as-text (a diffable text format that round-trips with Mermaid and D2), webhooks and an event stream, headless Rust export renderer, git-like version history with branching, local-first offline editing, public embeds and read-only share links.

Accepted from competitor research: local-first sync with optional end-to-end-encrypted rooms, facilitation suite with private mode, frames and scenes with presentation mode and per-frame export, docs and board as one document model, hand-drawn style toggle and themes, catch-up summaries from the op log, ink with pressure and partial erase, and later Talktrack-style recordings and runnable AI flows on canvas.

Room modes (added 2026-10-31): `hosted` (default, Durable Object persists), `ephemeral` (Durable Object relays in memory only and clears when the host leaves), `peer` (host's browser is the authority, guests connect over WebRTC with signaling and TURN from us, room dies when the host leaves), `e2e` (hosted relay of ciphertext, key in the URL fragment). Foundation ships the enum, the client transport seam and the signaling skeleton; the sync milestone implements the modes.

## 5. Milestones

Each milestone gets its own spec under `docs/superpowers/specs/`, then an implementation plan, then a build cycle.

1. **Foundation.** Monorepo and tooling, Cloudflare topology, metadata database and schema, auth, entitlements and rate limiting, BoardRoom Durable Object skeleton with the sync-engine seam, room modes and signaling skeleton, self-host image, codegen, CI/CD, observability, dev loop. Spec: `2026-10-31-foundation-design.md`.
2. **Canvas engine and sync.** Loro document model, two-tier renderer (Canvas2D content, DOM editing overlay) with a Rust/wasm core for geometry and hit-testing, stickies, shapes, connectors, text, sections, ink, images, presence, undo, offline cache, room modes.
3. **Command core and programmatic surfaces.** One command layer the UI uses; console, REST and WebSocket API, MCP server, generated SDKs, CLI, webhooks, board-as-text.
4. **Collaboration and facilitation.** Comments, voting, timer, follow, open sessions, facilitator role, anonymous mode, share links, embeds.
5. **Diagramming depth and imports.** Mermaid both ways, auto-layout, FigJam importer, tables, code blocks, mind maps, semantic objects.
6. **AI.** Sort, summarize, rewrite, text-to-diagram, board from prompt, agent over MCP, credit ledger.
7. **Exports, history and self-host polish.** Rust headless renderer, named versions and branching, catch-up, frames and presentation mode, self-host hardening.

## 6. References

- Research brief: `docs/research/2026-10-31-figjam-inventory-and-tech-landscape.md` (also published at https://claude.ai/artifact/7K5GrCE8cJMFuLSgiGDPdg).
- FigJam keymap inventory: `docs/research/figjam-keymap.json`.
