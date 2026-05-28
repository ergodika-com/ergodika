# Ergodika per sviluppatori Full-Stack

![Lingua](https://img.shields.io/badge/Lingua-Italiano-16a34a?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← Torna all'indice](../ERGODIKA_FULLSTACK.md)

> Brief architetturale pubblico. Spiega **come e stato programmato Ergodika** cosi uno sviluppatore puo capire il design prima di leggere ogni file del repository privato.

## A chi serve

Sviluppatori **full-stack** che cercano architettura, invarianti e confini dei moduli — non un tutorial di installazione.

**Elevator pitch:** Ergodika e un cockpit desktop con analisi visiva ad alta densita, ingestione live Binance e audio MVP che **sonifica** la pressione di order flow (CVD → pan, sentiment → pitch), in modalita read-only verso l'exchange.

---

## Indice

| Sezione | Contenuto |
|--------|-----------|
| [Sistema in sintesi](#sistema-in-sintesi) | Layer runtime e flusso dati |
| [Invarianti di design](#invarianti-di-design-non-negoziabili) | Regole che guidano ogni modulo |
| [Mappa repository](#mappa-repository-dove-vive-la-logica) | Dove cercare nel tree privato |
| [Contratto IPC](#contratto-ipc-rust--typescript) | Allineamento FE e backend |
| [Ingestione mercato](#ingestione-mercato-desktop-vs-browser) | Binance, stato vuoto, order flow |
| [Architettura UI](#architettura-ui-svelte) | Cockpit Quartet, V-Matrix, chart |
| [Motore audio](#motore-audio-mvp-desktop) | Thread, mapping, vincoli realtime |
| [Sicurezza](#modello-di-sicurezza) | Vault, canale account read-only |
| [Test e DX](#test-e-dx) | Qualita e tooling |
| [Highlight](#highlight-orientati-allimpatto) | Perche scala |
| [Roadmap](#roadmap-tecnica-sintetica) | Prossimi passi |

---

## Sistema in sintesi

Ergodika e un **cockpit desktop-first** (Tauri + SvelteKit + Rust) con **demo browser muta**. Un solo codice UI; due runtime tramite `isTauri()` e `feedController`.

```
┌──────────────────────────────────────────────────────────────────┐
│  UI SvelteKit (WebView nativa su desktop)                         │
│  CockpitQuartet · V-Matrix · lightweight-charts · diagnostica     │
│  feedController: idle | ipc | web                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ eventi + invoke Tauri (solo desktop)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Backend Rust                                                     │
│  live_feed/     → snapshot ErgodikaState (~20 Hz verso UI)        │
│  credentials/   → vault AES-256-GCM, stream Binance read-only   │
│  audio_engine/  → callback cpal, mixer 4 voci (MVP desktop)       │
└────────────────────────────┬─────────────────────────────────────┘
                             │ WSS + REST (endpoint ufficiali Binance)
                             ▼
                      Binance Spot (pubblico + user stream opzionale)
```

**Disciplina di throughput:** il calcolo mercato gira a frequenza WS piena in Rust; l'IPC verso la WebView e **deduplicato per firma** e **batchato nel tempo** (~20 Hz) senza inventare prezzi intermedi.

### Come ho strutturato il codice

1. **Backend Rust = fonte di verita** per tick, order flow, vault, audio.
2. **Frontend Svelte = osservatore reattivo** — niente prezzi inventati; solo snapshot e comandi.
3. **Contratto IPC versionato** come API interna tra i layer.
4. **Due runtime** dallo stesso bundle: desktop completo vs demo web muta.

---

## Invarianti di design (non negoziabili)

1. **Niente dati di mercato finti sul path principale.** Fino al primo tick reale: `price = 0`, copy esplicita. Errori/stale → diagnostica, **non** OHLC sintetici.
2. **Read-only verso gli exchange.** Solo monitoraggio e sonificazione; API key con permessi **sola lettura** Binance Spot.
3. **IPC schema-first.** Comandi ed eventi definiti una volta in `ipc_contract/contract.json`, testati su entrambi i lati.
4. **Parita visivo-sonora (dove spedito).** CVD, OBI, VWAP in Rust con **parita browser** in moduli TypeScript.
5. **Sicurezza callback audio.** Il callback cpal fa solo pull; coda **SPSC lock-free** dai thread mercato/render.
6. **Solo UI QUARTET.** Griglia 2×2 a quattro simboli; layout SOLO rimosso; OMNI oggi e solo etichetta manifest.

---

## Mappa repository (dove vive la logica)

Tree tipico privato (`ergodika_app/`):

| Area | Percorso (indicativo) | Ruolo |
|------|----------------------|--------|
| UI shell | `CockpitQuartet.svelte`, `QuartetSlotCard.svelte`, `CockpitLeftRail.svelte` | Layout, slot |
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts` | HUD da snapshot reali |
| Feed web | `webBinanceHub.ts`, `feedController.ts` | WS Binance senza IPC |
| Feed IPC | `ipcState.ts`, `quartet.ts` | listen + invoke, batch rAF |
| Tipi | `ergodikaState.ts` | Mirror payload Rust |
| Contratto IPC | `ipc_contract/contract.json` | Fonte unica invoke/eventi |
| Feed Rust | `src-tauri/src/live_feed/` | WS, order flow, klines |
| Payload | `ergodika_payload.rs`, `market_event.rs` | Snapshot, firma dedup |
| Account | `binance_user_stream.rs`, `credentials_vault.rs` | Vault, stream utente |
| Audio | `src-tauri/src/audio_engine/` | cpal, mixer, mapping |
| Manifest | `static/app_manifest.json` | Etichette QUARTET |

**Simboli tracciati:** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt`.

---

## Contratto IPC (Rust ↔ TypeScript)

- **File:** `ipc_contract/contract.json` (versionato).
- **Eventi:** snapshot quartet + opzionale `MarketEvent` (allineati a `emit` in `runtime.rs`).
- **Invoke:** camelCase nel JSON → snake_case in `lib.rs`.
- **FE:** solo `IPC_INVOKE` — niente stringhe duplicate nei componenti.
- **CI:** test `ipc_contract_alignment_tests`.

**Policy emit:** cache sempre aggiornata; emit verso WebView solo su cambio `ipc_quartet_signature` (~50 ms min) + heartbeat 1 s.

---

## Ingestione mercato (desktop vs browser)

| Modalita | Sorgente | Note |
|----------|----------|------|
| `ipc` | `live_feed` Rust | Quartet completo, account, klines |
| `web` | `webBinanceHub` | WS Binance + REST via proxy `__binance` |
| `idle` | — | Stato vuoto, niente fill-in sintetico |

**Order flow (spedito):** CVD, OBI, VWAP Anchor, Kinetic Impact, tape activity / Beat Bang.

Moduli Rust: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `kinetic_impact_sensor`, `tape_activity`.  
Parita web: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, ecc.

---

## Architettura UI (Svelte)

- SvelteKit 2, Svelte 5, Tailwind 4, tema ultra-dark.
- Store: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- Chart: `lightweight-charts` v5; bootstrap OHLC ~10k barre max.
- Smoothing: `vmatrixSmooth.ts` — solo polish visivo.
- Diagnostica: `RuntimeDiagnostics.svelte`.
- Copy UI in **inglese**; documentazione multilingua ammessa.
- **Slot:** derivare wiring da `quartetChartSlots` + manifest.

---

## Motore audio (MVP desktop)

**La demo browser e muta.**

| Pezzo | Ruolo |
|-------|--------|
| Thread mercato | Target atomici pan/pitch |
| Coda `rtrb` SPSC | ~112 ms; non blocca il callback |
| Thread render | Lerp pitch/pan ~320 ms, oscillatore `sin` |
| Callback cpal | Solo pull, limiter morbido |
| Mixer | 4 voci ÷ 4, mute ~18 ms |

**Mapping spedito:** pitch ← sentiment + fondamentale; pan ← CVD (Blumlein). Lane whale/OBI/reverb in backlog (`audio_mapping.json`).

---

## Modello di sicurezza

- Secret mai in chiaro nel frontend.
- AES-256-GCM + keyring OS (fallback file su WSL).
- User stream Binance: WebSocket API v3 con subscribe firmato.
- PnL da wallet + FIFO `myTrades` quando coerente.

---

## Test e DX

| Layer | Tool |
|-------|------|
| Frontend | Vitest |
| IPC | `cargo test --lib ipc_contract` |
| Desktop | `npm run start` |
| Web | `npm run start:web` |
| Rust | `npm run cargo:check` |

Documentazione repo privato: `CONTINUITA.md`, `AGENT_ONBOARDING.md`, `ARCHITECTTURA_FEED_UI.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## Highlight orientati all'impatto

- **Scalabilita:** layer ingestion, UI, DSP separati.
- **Realtime:** audio lock-free sotto burst di mercato.
- **Integrita prodotto:** niente tick finti sul path principale.
- **DX:** IPC tipizzato + test di allineamento.
- **Operativita:** diagnostica runtime per triage rapido.

---

## Roadmap tecnica sintetica

- Hardening DSP multi-lane e spatializzazione.
- Harness replay per regression visive/audio.
- Modularizzazione UI per sito istituzionale.

---

## Sfide affrontate

- Separare **frequenza WS** da **frequenza UI** senza perdere fedelta percepita.
- **Parita metrica** HUD ↔ suono su CVD/OBI/VWAP.
- Proteggere il callback audio da lock e allocazioni.
- Supportare WSL/desktop (ALSA, keyring, target Cargo su disco).

---

*Maggio 2026. Documentazione utente: [`lang/README.it.md`](../../lang/README.it.md).*
