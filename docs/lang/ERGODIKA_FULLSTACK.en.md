# Ergodika for Full-Stack Developers

![Language](https://img.shields.io/badge/Language-English-2563eb?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← Back to docs index](../ERGODIKA_FULLSTACK.md)

> Public architecture brief. Explains **how Ergodika was built** so a developer can reason about design before reading every file in the private repository.

## Who this is for

**Full-stack** engineers who need architecture, invariants, and module boundaries — not an install tutorial.

**Elevator pitch:** Ergodika is a desktop market cockpit with high-density visual analytics, live Binance ingestion, and a desktop audio MVP that **sonifies** order-flow pressure (OBI → pan, 4/4 metric grid → pitch) while staying read-only toward exchanges.

## Tech stack

| Layer | Technologies |
|-------|----------------|
| Desktop shell | **Tauri 2** (native WebView) |
| Frontend | **SvelteKit 2**, **Svelte 5**, **TypeScript**, **Tailwind 4** |
| Backend | **Rust** — `live_feed/`, `credentials/`, `audio_engine/` |
| Charts | **lightweight-charts** v5 |
| Audio | **cpal**, lock-free **SPSC** (`rtrb`), realtime DSP in Rust |
| Market data | **Binance Spot** (WSS + REST) |
| Security | **AES-256-GCM** vault, OS keyring |
| Quality | **Vitest**, versioned **IPC contract** + Rust alignment tests |

**Document scope:** architecture and engineering decisions in this public docs repository. Application source ships in the private `ergodika_app` tree; this brief lets reviewers assess system design without proprietary access.

---

## Table of contents

| Section | What you learn |
|--------|----------------|
| [System in one picture](#system-in-one-picture) | Runtime layers and data flow |
| [Design invariants](#design-invariants-non-negotiable) | Rules that shaped every module |
| [Repository map](#repository-map-where-logic-lives) | Where to look first in the private tree |
| [IPC contract](#ipc-contract-rust--typescript) | How FE and backend stay aligned |
| [Market ingestion](#market-ingestion-desktop-vs-browser) | Binance paths, empty state, order flow |
| [UI architecture](#ui-architecture-svelte) | Quartet cockpit, V-Matrix, charts |
| [Audio engine](#audio-engine-desktop-mvp) | Threads, DSP pipeline, mapping, realtime |
| [Security model](#security-model) | Vault, read-only posture |
| [Testing and DX](#testing-and-dx) | How quality is enforced |
| [Impact highlights](#impact-oriented-highlights) | Why the architecture scales |
| [Engineering challenges](#engineering-challenges-solved) | Hard problems and trade-offs |
| [Skills demonstrated](#skills-demonstrated) | What this project evidences in review |
| [Roadmap](#short-technical-roadmap) | Next engineering steps |

---

## System in one picture

Ergodika is a **desktop-first market cockpit** (Tauri + SvelteKit + Rust) with a **silent browser demo**. One UI codebase; two runtimes gated by `isTauri()` and `feedController`.

```
┌──────────────────────────────────────────────────────────────────┐
│  SvelteKit UI (native WebView on desktop)                         │
│  CockpitQuartet · V-Matrix · lightweight-charts · diagnostics     │
│  feedController: idle | ipc | web                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Tauri events + invoke (desktop only)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Rust backend                                                     │
│  live_feed/     → ErgodikaState snapshots (~20 Hz emit to UI)     │
│  credentials/   → AES-256-GCM vault, Binance read-only stream     │
│  audio_engine/  → cpal callback, 4-voice mixer (desktop MVP)      │
└────────────────────────────┬─────────────────────────────────────┘
                             │ WSS + REST (official Binance endpoints)
                             ▼
                      Binance Spot (public market data)
```

**Throughput discipline:** market math runs at full WS rate in Rust; IPC to the WebView is **signature-deduped** and **time-batched** (~20 Hz) so the UI stays responsive without inventing intermediate prices.

### How the codebase is organized

1. **Rust backend = source of truth** for ticks, order flow, vault, and audio.
2. **Svelte frontend = reactive observer** — no fabricated prices; snapshots + controls only.
3. **Versioned IPC contract** as the internal API between layers.
4. **Dual runtime** from one bundle: full desktop vs silent web demo.

---

## Design invariants (non-negotiable)

1. **No fake market data on the main path.** Until the first real tick: `price = 0`, explicit empty copy. Errors and stale feeds → diagnostics, **not** synthetic OHLC.
2. **Read-only toward exchanges.** Monitoring and sonification only; API keys intended for **read-only** Binance Spot scopes.
3. **Schema-first IPC.** Commands and events defined once in `ipc_contract/contract.json`, tested on both sides.
4. **Visual–sonic parity (where shipped).** CVD, OBI, VWAP computed in Rust with **browser parity** in TypeScript modules.
5. **Audio callback safety.** cpal callback only pulls samples; **lock-free SPSC** queue from market/render threads.
6. **QUARTET-only product UI.** Four symbols in 2×2; SOLO removed; OMNI is manifest labeling only today.

---

## Repository map (where logic lives)

Typical private tree (`ergodika_app/`):

| Area | Path (indicative) | Responsibility |
|------|-------------------|----------------|
| UI shell | `CockpitQuartet.svelte`, `QuartetSlotCard.svelte`, `CockpitLeftRail.svelte` | Layout, slot wiring |
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts`, `scalpingScorecard.ts` | HUD from real snapshots; UI-only scalping scorecard |
| Feed (web) | `webBinanceHub.ts`, `feedController.ts` | Binance WS when not on IPC |
| Feed (IPC) | `ipcState.ts`, `quartet.ts` | listen + invoke, rAF batching |
| Types | `ergodikaState.ts` | `ErgodikaState` mirror of Rust |
| IPC contract | `ipc_contract/contract.json` | Invokes/events single source of truth |
| Rust feed | `src-tauri/src/live_feed/` | WS, order flow, klines, emit |
| Payload | `ergodika_payload.rs`, `market_event.rs` | Snapshot, dedup signature |
| Credentials | `credentials_vault.rs` | AES-256-GCM vault, OS keyring (BYOK when enabled) |
| Audio | `src-tauri/src/audio_engine/` | cpal, mixer, mapping |
| Manifest | `static/app_manifest.json` | QUARTET labels |

**Tracked symbols:** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt`.

---

## IPC contract (Rust ↔ TypeScript)

- **File:** `ipc_contract/contract.json` (versioned).
- **Events:** quartet snapshot + optional `MarketEvent` (must match `live_feed/runtime.rs` emit).
- **Invokes:** camelCase in JSON → snake_case `#[tauri::command]` in `lib.rs`.
- **FE:** use `IPC_INVOKE` only — no duplicate string literals.
- **CI:** `ipc_contract_alignment_tests` in Rust.

**Emit policy:** in-memory cache always updated; WebView emits on `ipc_quartet_signature` change (~50 ms min interval) + 1 s diagnostic heartbeat.

---

## Market ingestion (desktop vs browser)

| Mode | Source | Notes |
|------|--------|-------|
| `ipc` | Rust `live_feed` | Full quartet, klines invoke |
| `web` | `webBinanceHub` | Binance WS + REST via dev proxy `__binance` |
| `idle` | — | Empty state, no synthetic fill-in |

**Order flow (shipped):** CVD, OBI, VWAP Anchor, Kinetic Impact, tape activity / Beat Bang.

Rust modules: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `kinetic_impact_sensor`, `tape_activity`.  
Web parity: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, `kineticImpactSensor.ts`, `tapeActivitySensor.ts`.

---

## UI architecture (Svelte)

- SvelteKit 2, Svelte 5, Tailwind 4, ultra-dark glassmorphism.
- **QUARTET cockpit:** symmetric 2×2 grid monitoring four symbols; rail controls for timeframe, interval, and Sound.
- **V-Matrix HUD:** normalized order-flow metrics (position in range, whale flow, spread, CVD, OBI, VWAP anchor, kinetic impact) plus **Flow Direction** labels and **Scalping score** from **live snapshots only** via `computeVMatrixSnapshot` — no simulated tick loop on the main path.
- Stores: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- Charts: `lightweight-charts` v5; OHLC bootstrap cap ~10k bars; Binance-aligned interval presets.
- Smoothing: `vmatrixSmooth.ts` — visual polish only, never a substitute for real feed data.
- Diagnostics: `RuntimeDiagnostics.svelte` (Last frame, Snapshot age, Live ingest).
- UI copy in **English**; docs may be multilingual.
- **Slot model:** `quartetChartSlots` from manifest — avoid duplicate slot wiring in leaf components.

### Scalping score

UI-only **pre-entry scorecard** from smoothed `VMatrixSnapshot` via `computeScalpingScorecard` (`scalpingScorecard.ts`). **Not** in the Rust IPC payload; **not** a trading signal or order trigger.

| Piece | Detail |
|-------|--------|
| Total | **0–100** from fixed weights on existing lanes |
| Weights | Spread tightness **20**, impact stability **10**, OBI strength **15**, CVD strength + OBI/CVD coherence **20**, whale flow **10**, tape activity **25** |
| Bands | **A** (≥80), **B** (65–79), **C** (50–64), **NO_TRADE** (<50 or gate blocked) |
| Conservative gate | Forces **NO_TRADE** when spread is too wide (`Vs ≥ 0.82`), tape activity is too low (`activity01 ≤ 0.22`), or OBI and CVD point opposite ways |
| UI | `VmatrixSlotColumn.svelte` — band chip, score bar, gate tooltip (`Spread too wide`, `Low tape activity`, `OBI/CVD divergence`) |
| vs Flow Direction | **Independent** — score judges execution quality; Flow Direction labels describe aggressive buy/sell pressure consensus |

---

## Audio engine (desktop MVP)

**Browser demo is silent.** All real-time **audio DSP** runs in Rust (`src-tauri/src/audio_engine/`). The Svelte app is **audio-agnostic**: IPC controls only (`audioEngine.ts`, Sound tab); `src/lib/audio/bridge.ts` is contract parity / DEV logging — no synthesis in JavaScript.

| Piece | Role |
|-------|------|
| Market thread | Updates atomic pan/pitch targets from order-flow metrics |
| `rtrb` SPSC | ~112 ms cushion; never blocks callback |
| Render thread | `PitchHzLerp` / `PanGainLerp` (~320 ms), `sin` oscillators |
| cpal callback | Pull-only, soft limiter |
| Mixer | 4 voices ÷ 4, mute fade ~18 ms |

### Audio DSP pipeline

```
live_feed / order-flow (Rust)
    → atomic pan & pitch targets
    → lock-free SPSC (~112 ms)
    → render: pitch/pan lerp + sine voices
    → 4-voice mixer + soft limiter
    → cpal pull callback (no locks, no heap alloc)
```

| DSP layer | Where | Notes |
|-----------|--------|------|
| Mapping contract | `audio_mapping.json`, `docs/AUDIO_ENGINE.md` | Lane definitions; product-owned before implementation |
| Shipped sonification | OBI → **pan** (Blumlein); 4/4 metric grid + slot fundamental → **pitch**; kinetic impact → strike overlay | Same Rust metrics as HUD where parity is required |
| Backlog lanes | whale flow, OBI, reverb | Listed in mapping JSON; not in active MVP mix |
| UI controls | `stores/audioEngine.ts`, master volume | Desktop IPC only |

**Realtime rules:** treat the cpal callback as a hard RT path — pull only; never block market/UI on audio; keep **visual–sonic parity** on CVD, OBI, VWAP.

**Private docs:** `AUDIO_ENGINE.md`, `ARCHITECTTURA_FEED_UI.md`. Design references for spatial/lane work: Curtis Roads, Julius O. Smith III.

---

## Security model

- Secrets never stored in clear text in the frontend.
- AES-256-GCM + OS keyring (file fallback on WSL) when BYOK credentials are enabled.

---

## Testing and DX

| Layer | Tooling |
|-------|---------|
| Frontend | Vitest (`npm run test`, `validate`) |
| IPC | `cargo test --lib ipc_contract` |
| Desktop dev | `npm run start` |
| Web dev | `npm run start:web` |
| Rust | `npm run cargo:check` |

Private repo docs: `CONTINUITA.md`, `AGENT_ONBOARDING.md`, `ARCHITECTTURA_FEED_UI.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## Impact-oriented highlights

- **Scalability:** isolated ingestion, UI, and DSP layers evolve independently.
- **Realtime:** lock-free audio path under bursty WebSocket load; UI fed at ~20 Hz without fabricating prices.
- **Product integrity:** honest empty state and diagnostics instead of synthetic OHLC on errors.
- **Cross-language DX:** schema-first IPC with automated alignment tests between Rust and TypeScript.
- **Operability:** runtime diagnostics (frame age, ingest heartbeat) for production triage.
- **Security posture:** encrypted credential vault; exchange interaction limited to read-only monitoring scopes.

---

## Engineering challenges solved

| Challenge | Approach |
|-----------|----------|
| WS throughput vs UI frame budget | Full-rate market math in Rust; **signature-deduped**, **time-batched** IPC (~20 Hz) to the WebView |
| Cross-runtime parity | One Svelte bundle; `feedController` switches `ipc` / `web` / `idle` without duplicate product logic |
| Visual–sonic consistency | Order-flow metrics (CVD, OBI, VWAP) computed in Rust; TS parity modules for browser demo; audio driven from the same sources where shipped |
| Hard-realtime audio | Market thread writes atomics → **SPSC** queue → render lerp → **pull-only** cpal callback (no locks, no heap alloc in callback) |
| Trust under missing data | `price = 0` and explicit copy until first real tick; stale/error paths surface diagnostics, not mock candles |
| Desktop portability | Tauri native shell; WSL/file keyring fallbacks; ALSA/desktop audio path documented in private ops guides |

---

## Skills demonstrated

- **Full-stack systems design:** dual-runtime desktop + web from a single UI codebase with clear backend boundaries.
- **Rust systems programming:** concurrent market ingestion, cryptographic vault, realtime DSP and mixer.
- **TypeScript/Svelte product engineering:** reactive stores, chart integration, high-information-density UX.
- **API integration:** Binance Spot WSS/REST, kline bootstrap and interval governance.
- **Contract-driven development:** versioned IPC schema, CI alignment tests, camelCase/snake_case discipline.
- **Domain modeling:** order-flow analytics (CVD, OBI, VWAP Anchor, kinetic/tape signals) with testable modules.
- **Audio DSP (applied):** sonification mapping, spatial pan law, thread-safe buffer handoff under load.

---

## Short technical roadmap

- DSP lane hardening and spatialization.
- Replay-based visual/audio regression suite.
- UI modularization for institutional web reuse.

---

*May 2026. End-user docs: [`lang/README.en.md`](../../lang/README.en.md).*
