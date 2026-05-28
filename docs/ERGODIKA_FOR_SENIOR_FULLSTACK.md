# Ergodika for Senior Full-Stack Developers

> **Public architecture brief.** This repository documents the product and onboarding narrative. The **application source** (SvelteKit + Tauri + Rust) lives in the private codebase; this guide explains **how Ergodika is engineered** so a senior developer can reason about design without reading every file first.
>
> **i18n:** [Italiano](#it-italiano) · [English](#en-english) · [Русский](#ru-русский) · [中文](#zh-中文) · [한국어](#ko-한국어)

[![IT](https://img.shields.io/badge/lang-IT-16a34a?style=flat-square)](#it-italiano)
[![EN](https://img.shields.io/badge/lang-EN-2563eb?style=flat-square)](#en-english)
[![RU](https://img.shields.io/badge/lang-RU-f59e0b?style=flat-square)](#ru-русский)
[![ZH](https://img.shields.io/badge/lang-ZH-9333ea?style=flat-square)](#zh-中文)
[![KO](https://img.shields.io/badge/lang-KO-ec4899?style=flat-square)](#ko-한국어)

---

## Technical navigation (language-agnostic)

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

---

## Design invariants (non-negotiable)

These constraints explain many “unusual” choices a new contributor will see:

1. **No fake market data on the main path.** Until the first real tick: `price = 0`, explicit empty copy. Errors and stale feeds → diagnostics and degradation messages, **not** synthetic OHLC or drifted prices.
2. **Read-only toward exchanges.** Monitoring and sonification only; no order placement in the product surface. API keys are intended for **read-only** Binance Spot scopes.
3. **Schema-first IPC.** Command names and event channels are defined once in `ipc_contract/contract.json`, code-generated / tested on both sides — no duplicated magic strings.
4. **Visual–sonic parity (where shipped).** Order-flow metrics (CVD, OBI, VWAP session) are computed in Rust with **browser parity** in TypeScript math modules for the web demo.
5. **Audio callback safety.** The cpal callback only pulls mixed samples; heavy work stays on market and render threads; **lock-free SPSC** queue between them.
6. **QUARTET-only product UI.** Four concurrent symbols in a 2×2 grid; legacy SOLO layout removed; OMNI is manifest labeling only today.

---

## Repository map (where logic lives)

Typical private tree (`ergodika_app/`):

| Area | Path (indicative) | Responsibility |
|------|-------------------|----------------|
| UI shell | `src/lib/components/CockpitQuartet.svelte`, `QuartetSlotCard.svelte`, `CockpitLeftRail.svelte` | Layout, slot wiring, rail controls |
| V-Matrix | `src/lib/components/Vmatrix*.svelte`, `src/lib/mocks/marketSim.ts` | HUD lanes from **real** snapshots (`computeVMatrixSnapshot` — no tick simulator) |
| Feed (web) | `src/lib/live/webBinanceHub.ts`, `src/lib/stores/feedController.ts` | Binance combined WS when not on IPC |
| Feed (IPC) | `src/lib/tauri/ipcState.ts`, `src/lib/stores/quartet.ts` | listen + invoke, ingress heartbeat, rAF batching |
| Types | `src/lib/types/ergodikaState.ts` | `ErgodikaState` mirror of Rust payload |
| IPC contract | `src/lib/config/ipc_contract/contract.json` | Single source of truth for invokes/events |
| Rust feed | `src-tauri/src/live_feed/` | WS adapters, order flow, klines, runtime emit |
| Payload | `src-tauri/src/ergodika_payload.rs`, `market_event.rs` | Snapshot shape, dedup signature, quartet symbols |
| Account | `src-tauri/src/binance_user_stream.rs`, `credentials_vault.rs` | Encrypted credentials, user stream, FIFO PnL helpers |
| Audio | `src-tauri/src/audio_engine/` | cpal, mixer, mapping, lerp, instruments |
| Manifest | `static/app_manifest.json` | QUARTET labels, layout hints |

**Tracked symbols (desktop IPC quartet):** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt` — Rust and UI must stay aligned (`cockpitSymbols.ts`, manifest).

---

## IPC contract (Rust ↔ TypeScript)

- **File:** `src/lib/config/ipc_contract/contract.json` (versioned, e.g. `version: 3`).
- **Events:** aliases for quartet snapshot and optional `MarketEvent` lane (names must match `emit` in `live_feed/runtime.rs`).
- **Invokes:** camelCase keys in JSON → snake_case `#[tauri::command]` in `lib.rs` (e.g. `fetchQuartetKlinesBootstrap` → `fetch_quartet_klines_bootstrap`).
- **FE usage:** import `IPC_INVOKE` from `ipc_contract/index.ts`; **never** duplicate command strings in components.
- **CI guard:** `ipc_contract_alignment_tests` in Rust compares JSON to registered commands.

**Emit policy (why the UI feels “smooth”):** backend always updates in-memory latest cache for `get_latest_quartet`; WebView receives emits only when `ipc_quartet_signature` changes (quantized floats + key text fields) and at a minimum interval (~50 ms), plus a 1 s heartbeat for diagnostics.

---

## Market ingestion (desktop vs browser)

| Mode | Trigger | Source | Notes |
|------|---------|--------|-------|
| `ipc` | Tauri desktop | Rust `live_feed` | Full quartet, account stream, klines invoke |
| `web` | Browser / Vite dev | `webBinanceHub` | Binance-only WS + REST via dev proxy `__binance` |
| `idle` | No subscription | — | Empty market state, no synthetic fill-in |

**Streams (Binance spot, indicative):** `@trade`, `@ticker`, `@bookTicker`, `@depth5@100ms` — deserialized in Rust; combined URL builder in TS for web.

**Order flow (shipped):**

- **CVD** — session cumulative volume delta, pressure bar for HUD.
- **OBI** — depth imbalance on top levels.
- **VWAP Anchor** — session VWAP vs last price, normalized bar.
- **Kinetic Impact** — microstructure collision score (whale size + CVD acceleration).
- **Tape activity / rhythm** — BPM master, 4/4 grid, Beat Bang visual metronome per slot.

Rust: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `depth_imbalance_aggregator`, `kinetic_impact_sensor`, `tape_activity`.  
Web parity: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, `kineticImpactSensor.ts`, `tapeActivitySensor.ts`.

---

## UI architecture (Svelte)

- **Framework:** SvelteKit 2, Svelte 5 runes where adopted, Tailwind 4, glassmorphism ultra-dark theme.
- **State:** Svelte stores (`quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`) — high-frequency snapshots copied shallowly (`snapshotErgodikaState`) before rAF.
- **Charts:** `lightweight-charts` v5 via shared setup (`lightweightChartsSetup.ts`); Time + Interval bars; bootstrap OHLC capped (~10k bars) with merge rules.
- **Smoothing:** `vmatrixSmooth.ts` — visual lerp only; does not replace feed truth.
- **Diagnostics:** `RuntimeDiagnostics.svelte` — Market Link, Last frame, Snapshot age, Live ingest heartbeat (not NLP “radar”).
- **Language:** UI copy in **English**; documentation for agents/users may be multilingual.

**Slot model:** `quartetChartSlots` derives chart/rail/mixer wiring from `app_manifest.json` + cockpit symbol picks — avoid duplicating slot indices inside leaf components.

---

## Audio engine (desktop MVP)

**Not available in browser demo.** Desktop-only path:

| Piece | Role |
|-------|------|
| Market thread | Reads quartet snapshots, updates atomic pan/pitch targets |
| `rtrb` SPSC queue | ~112 ms cushion; producer never blocks cpal callback |
| Render thread | Pop frames, apply `PitchHzLerp` / `PanGainLerp` (~320 ms), `sin` oscillator |
| cpal callback | Pull-only, soft limiter on master bus |
| Mixer | 4 voices ÷ 4, per-slot mute with ~18 ms fade, master gain |

**Mapping (shipped MVP):**

- **Pitch** ← sentiment + per-slot fundamental (Sound page).
- **Pan** ← CVD via Blumlein (`blumlein_pan.rs`).
- **Not in mix yet:** whale/OBI/reverb lanes described in `audio_mapping.json` (design backlog).

FE controls: `CockpitSoundToggle`, `CockpitAudioMaster`, `/sound` route, IPC `set_audio_enabled`, `set_quartet_slot_mutes`, etc.

---

## Security model

- API secrets **never** sent to the frontend process for storage in clear text.
- **AES-256-GCM** payload encryption; MEK in OS keyring with file fallback on headless/WSL.
- Binance user stream via **WebSocket API v3** signed subscribe (post–listenKey deprecation).
- **PnL / position** from spot wallet + `myTrades` FIFO when coherent; otherwise conservative UI labeling.

---

## Testing and DX

| Layer | Tooling |
|-------|---------|
| Frontend unit | Vitest (`npm run test`, `validate`) |
| IPC contract | `cargo test --lib ipc_contract` |
| Dev desktop | `npm run start` (Vite + Tauri, port 5173 strict) |
| Dev web | `npm run start:web` |
| Rust check | `npm run cargo:check` (target dir on disk for WSL) |

**Contributor docs (private repo):** `docs/CONTINUITA.md`, `docs/AGENT_ONBOARDING.md`, `docs/ARCHITECTTURA_FEED_UI.md`, `docs/AUDIO_ENGINE.md`, `docs/USER_GUIDE.md` (end-user, English).

---

## Impact-oriented highlights

- **Scalability:** ingestion, UI, and DSP are isolated so each layer can evolve without cascading rewrites.
- **Realtime:** lock-free audio queue + pull-only callback keeps latency stable under bursty market updates.
- **Product integrity:** explicit empty/degraded states protect trust — no “chart theater” with fake ticks.
- **DX:** typed IPC contract and alignment tests make TS/Rust changes predictable.
- **Operability:** runtime diagnostics (Last frame, Snapshot age, feed source) speed up incident triage.

---

## it-Italiano

### Per chi e questo documento

Per sviluppatori **full-stack senior** che devono capire **come e stato programmato Ergodika**: non e un tutorial di installazione, ma una mappa mentale di architettura, invarianti e moduli.

### Elevator pitch

Ergodika e un cockpit desktop per analisi di mercato in tempo reale: visualizzazione ad alta densita, pipeline dati live Binance, e un motore audio MVP che **sonifica** pressione di order flow (CVD → pan, sentiment → pitch) senza sostituire il giudizio del trader.

### Come ho strutturato il codice

1. **Backend Rust = fonte di verita** per tick, order flow, vault, audio.
2. **Frontend Svelte = osservatore reattivo** che non inventa prezzi; consuma snapshot e comanda il motore.
3. **Contratto IPC versionato** come API interna tra i due mondi.
4. **Due runtime** con lo stesso bundle: desktop completo vs demo web muta.

### Sfide affrontate (da senior a senior)

- Separare **frequenza WS** da **frequenza UI** senza perdere fedelta percepita.
- Mantenere **parita metrica** tra HUD e suono su CVD/OBI/VWAP.
- Proteggere il thread audio da lock e allocazioni nel callback.
- Gestire WSL/desktop (ALSA, keyring, cargo target su disco) senza rompere il loop dev.

### Stato attuale (maggio 2026)

- QUARTET 2×2, quattro coppie spot, V-Matrix estesa, Beat Bang, master tempo.
- Account Binance read-only opzionale con vault locale.
- Audio MVP 4 voci; roadmap fundsp / lane whale-OBI-reverb.

### Roadmap tecnica sintetica

- Hardening DSP multi-lane e spatializzazione.
- Replay harness per regression visive/audio.
- Modularizzazione componenti per sito istituzionale.

---

## en-English

### Who this is for

Senior **full-stack** engineers who need to understand **how Ergodika was built** — architecture, invariants, and module boundaries — before diving into the private repository.

### Elevator pitch

Ergodika is a desktop market cockpit: high-density visual analytics, live Binance ingestion, and a desktop audio MVP that **sonifies** order-flow pressure (CVD → pan, sentiment → pitch) while remaining read-only toward exchanges.

### How the codebase is organized

1. **Rust backend = source of truth** for ticks, order flow, vault, and audio.
2. **Svelte frontend = reactive observer** — no fabricated prices; snapshots + controls only.
3. **Versioned IPC contract** as the internal API between layers.
4. **Dual runtime** from one bundle: full desktop vs silent web demo.

### Problems solved intentionally

- Decouple **WS rate** from **UI emit rate** without lying to the user.
- Keep **metric parity** between HUD and sonification on CVD/OBI/VWAP.
- Keep the audio callback allocation-free and lock-free.
- Support WSL/desktop dev constraints (ALSA, keyring, on-disk Cargo target).

### Current state (May 2026)

- QUARTET 2×2, four spot pairs, extended V-Matrix, Beat Bang, master tempo.
- Optional Binance read-only account with local encrypted vault.
- 4-voice audio MVP; roadmap for fundsp and additional sonic lanes.

### Short technical roadmap

- DSP lane hardening and spatialization.
- Replay-based visual/audio regression suite.
- UI modularization for institutional web reuse.

---

## ru-Русский

### Для кого этот документ

Для **senior full-stack** разработчиков, которым нужно понять **как спроектирован и реализован Ergodika**: архитектура, инварианты и границы модулей до чтения всего репозитория.

### Суть проекта

Ergodika — desktop cockpit для анализа рынка в реальном времени: плотная визуальная аналитика, live Binance, MVP аудио-движка, который **озвучивает** давление order flow (CVD → pan, sentiment → pitch), оставаясь read-only к бирже.

### Организация кода

1. **Rust backend** — источник истины для тиков, order flow, vault и audio.
2. **Svelte frontend** — реактивный наблюдатель без поддельных цен.
3. **Версионированный IPC-контракт** между слоями.
4. **Два runtime** из одного бандла: desktop и беззвучная web demo.

### Инженерные решения

- Разделение частоты **WS** и **emit в UI** без искажения данных.
- Паритет метрик HUD и звука (CVD/OBI/VWAP).
- Безопасный audio callback (SPSC, только pull).
- Поддержка WSL/desktop dev.

### Текущее состояние (май 2026)

- QUARTET 2×2, V-Matrix, Beat Bang, master tempo.
- Опциональный read-only аккаунт Binance с локальным vault.
- MVP 4 голоса; roadmap fundsp и дополнительных sonic lane.

---

## zh-中文

### 文档对象

面向需要理解 **Ergodika 如何实现** 的高级全栈工程师：架构、不可变约束与模块边界（完整源码在私有仓库）。

### 核心思路

- **Rust 后端** 负责真实行情、订单流、凭证与桌面音频。
- **Svelte 前端** 只消费快照，主路径不伪造价格。
- **版本化 IPC 契约** 连接两层。
- **双运行时**：Tauri 桌面完整功能 vs 浏览器静音演示。

### 当前能力（2026 年 5 月）

QUARTET 四格驾驶舱、CVD/OBI/VWAP、Kinetic Impact、可选 Binance 只读账户、本地加密 vault、四声部音频 MVP。

---

## ko-한국어

### 문서 대상

**Ergodika 구현 방식**을 이해해야 하는 시니어 풀스택 개발자용 아키텍처 브리핑입니다(애플리케이션 소스는 비공개 저장소).

### 설계 요약

- **Rust 백엔드**: 틱, 오더플로우, vault, 데스크톱 오디오의 단일 진실 공급원.
- **Svelte 프론트**: 스냅샷 소비만; 메인 경로에서 가짜 시세 없음.
- **버전 IPC 계약**으로 TS/Rust 정렬.
- **이중 런타임**: Tauri 데스크톱 vs 무음 웹 데모.

### 현재 상태 (2026년 5월)

QUARTET 2×2, CVD/OBI/VWAP, Kinetic Impact, Binance read-only 계정(선택), 로컬 암호화 vault, 4성부 오디오 MVP.

---

*Last aligned with product state: May 2026. For end-user documentation see `lang/README.*.md`.*
