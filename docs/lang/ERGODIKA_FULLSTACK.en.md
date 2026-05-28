# Ergodika for Senior Full-Stack Developers

![Language](https://img.shields.io/badge/Language-English-2563eb?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← Back to docs index](../ERGODIKA_FULLSTACK.md)

> Public architecture brief. Explains **how Ergodika was built** so a senior developer can reason about design before reading every file in the private repository.

## Who this is for

Senior **full-stack** engineers who need architecture, invariants, and module boundaries — not an install tutorial.

**Elevator pitch:** Ergodika is a desktop market cockpit with high-density visual analytics, live Binance ingestion, and a desktop audio MVP that **sonifies** order-flow pressure (CVD → pan, sentiment → pitch) while staying read-only toward exchanges.

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
| [Audio engine](#audio-engine-desktop-mvp) | Threads, mapping, realtime constraints |
| [Security model](#security-model) | Vault, read-only account channel |
| [Testing and DX](#testing-and-dx) | How quality is enforced |
| [Impact highlights](#impact-oriented-highlights) | Why the architecture scales |
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
                      Binance Spot (public + optional user stream)
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
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts` | HUD from real snapshots (`computeVMatrixSnapshot`) |
| Feed (web) | `webBinanceHub.ts`, `feedController.ts` | Binance WS when not on IPC |
| Feed (IPC) | `ipcState.ts`, `quartet.ts` | listen + invoke, rAF batching |
| Types | `ergodikaState.ts` | `ErgodikaState` mirror of Rust |
| IPC contract | `ipc_contract/contract.json` | Invokes/events single source of truth |
| Rust feed | `src-tauri/src/live_feed/` | WS, order flow, klines, emit |
| Payload | `ergodika_payload.rs`, `market_event.rs` | Snapshot, dedup signature |
| Account | `binance_user_stream.rs`, `credentials_vault.rs` | Vault, user stream, FIFO PnL |
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
| `ipc` | Rust `live_feed` | Full quartet, account, klines invoke |
| `web` | `webBinanceHub` | Binance WS + REST via dev proxy `__binance` |
| `idle` | — | Empty state, no synthetic fill-in |

**Order flow (shipped):** CVD, OBI, VWAP Anchor, Kinetic Impact, tape activity / Beat Bang.

Rust modules: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `kinetic_impact_sensor`, `tape_activity`.  
Web parity: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, `kineticImpactSensor.ts`, `tapeActivitySensor.ts`.

---

## UI architecture (Svelte)

- SvelteKit 2, Svelte 5, Tailwind 4, ultra-dark glassmorphism.
- Stores: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- Charts: `lightweight-charts` v5; OHLC bootstrap cap ~10k bars.
- Smoothing: `vmatrixSmooth.ts` — visual only.
- Diagnostics: `RuntimeDiagnostics.svelte` (Last frame, Snapshot age, Live ingest).
- UI copy in **English**; docs may be multilingual.
- **Slot model:** `quartetChartSlots` from manifest — avoid duplicate slot wiring in leaf components.

---

## Audio engine (desktop MVP)

**Browser demo is silent.**

| Piece | Role |
|-------|------|
| Market thread | Updates atomic pan/pitch targets |
| `rtrb` SPSC | ~112 ms cushion; never blocks callback |
| Render thread | `PitchHzLerp` / `PanGainLerp` (~320 ms), `sin` |
| cpal callback | Pull-only, soft limiter |
| Mixer | 4 voices ÷ 4, mute fade ~18 ms |

**Shipped mapping:** pitch ← sentiment + fundamental; pan ← CVD (Blumlein). Whale/OBI/reverb lanes in `audio_mapping.json` = backlog.

---

## Security model

- Secrets never stored in clear text in the frontend.
- AES-256-GCM + OS keyring (file fallback on WSL).
- Binance user stream: WebSocket API v3 signed subscribe.
- PnL from wallet + `myTrades` FIFO when coherent.

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

- **Scalability:** isolated ingestion, UI, DSP layers.
- **Realtime:** lock-free audio path under bursty markets.
- **Product integrity:** no fake ticks on the main path.
- **DX:** typed IPC + alignment tests.
- **Operability:** runtime diagnostics for triage.

---

## Short technical roadmap

- DSP lane hardening and spatialization.
- Replay-based visual/audio regression suite.
- UI modularization for institutional web reuse.

---

*May 2026. End-user docs: [`lang/README.en.md`](../../lang/README.en.md).*
