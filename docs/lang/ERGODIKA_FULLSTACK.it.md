# Ergodika per sviluppatori Full-Stack

![Lingua](https://img.shields.io/badge/Lingua-Italiano-16a34a?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← Torna all'indice](../ERGODIKA_FULLSTACK.md)

> Brief architetturale pubblico. Spiega **come e stato programmato Ergodika** cosi uno sviluppatore puo capire il design prima di leggere ogni file del repository privato.

## A chi serve

Sviluppatori **full-stack** che cercano architettura, invarianti e confini dei moduli — non un tutorial di installazione.

**Elevator pitch:** Ergodika e un cockpit desktop con analisi visiva ad alta densita, ingestione live Binance e audio MVP che **sonifica** la pressione di order flow (CVD → pan, sentiment → pitch), in modalita read-only verso l'exchange.

## Stack tecnologico

| Layer | Tecnologie |
|-------|------------|
| Shell desktop | **Tauri 2** (WebView nativa) |
| Frontend | **SvelteKit 2**, **Svelte 5**, **TypeScript**, **Tailwind 4** |
| Backend | **Rust** — `live_feed/`, `credentials/`, `audio_engine/` |
| Chart | **lightweight-charts** v5 |
| Audio | **cpal**, code **SPSC** lock-free (`rtrb`), DSP realtime in Rust |
| Mercato | **Binance Spot** (WSS + REST; user stream read-only opzionale) |
| Sicurezza | Vault **AES-256-GCM**, keyring OS |
| Qualita | **Vitest**, contratto **IPC** versionato + test di allineamento Rust |

**Ambito del documento:** decisioni architetturali in questo repository pubblico. Il codice applicativo vive nel tree privato `ergodika_app`; il brief permette di valutare il design senza accesso al codice proprietario.

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
| [Motore audio](#motore-audio-mvp-desktop) | Thread, pipeline DSP, mapping, realtime |
| [Sicurezza](#modello-di-sicurezza) | Vault, canale account read-only |
| [Test e DX](#test-e-dx) | Qualita e tooling |
| [Highlight](#highlight-orientati-allimpatto) | Perche scala |
| [Sfide affrontate](#sfide-affrontate) | Problemi difficili e trade-off |
| [Competenze evidenziate](#competenze-evidenziate) | Cosa dimostra il progetto in review |
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

### Organizzazione del codice

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
- **Cockpit QUARTET:** griglia 2×2 su quattro simboli; rail per timeframe, intervallo e Sound.
- **HUD V-Matrix:** sei sensi di mercato normalizzati (prezzo, volume, spread, PnL% non realizzato, sentiment, impact) da **snapshot live** con `computeVMatrixSnapshot` — nessun loop di tick simulato sul path principale.
- Store: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- Chart: `lightweight-charts` v5; bootstrap OHLC ~10k barre; preset intervallo allineati a Binance.
- Smoothing: `vmatrixSmooth.ts` — solo polish visivo, mai sostituto del feed reale.
- Diagnostica: `RuntimeDiagnostics.svelte` (Last frame, eta snapshot, Live ingest).
- Copy UI in **inglese**; documentazione multilingua ammessa.
- **Slot:** derivare wiring da `quartetChartSlots` + manifest.

---

## Motore audio (MVP desktop)

**La demo browser e muta.** Tutto il **DSP audio** realtime vive in Rust (`src-tauri/src/audio_engine/`). Il frontend e **audio-agnostic**: comandi via IPC (`audioEngine.ts`, tab Sound); `src/lib/audio/bridge.ts` e solo parita contratto / log DEV — nessuna sintesi in JavaScript.

| Pezzo | Ruolo |
|-------|--------|
| Thread mercato | Target atomici pan/pitch da metriche order flow |
| Coda `rtrb` SPSC | ~112 ms; non blocca il callback |
| Thread render | Lerp pitch/pan ~320 ms, oscillatori `sin` |
| Callback cpal | Solo pull, limiter morbido |
| Mixer | 4 voci ÷ 4, mute ~18 ms |

### Pipeline DSP audio

```
live_feed / order-flow (Rust)
    → target atomici pan e pitch
    → SPSC lock-free (~112 ms)
    → render: lerp pitch/pan + onde sinusoidali
    → mixer 4 voci + limiter morbido
    → callback cpal pull (no lock, no alloc heap)
```

| Layer DSP | Dove | Note |
|-----------|------|------|
| Contratto mapping | `audio_mapping.json`, `docs/AUDIO_ENGINE.md` | Lane definite dal prodotto prima dell'implementazione |
| Sonificazione spedita | sentiment + fondamentale → **pitch**; CVD → **pan** (Blumlein) | Stesse metriche Rust dell'HUD dove serve parita |
| Lane in backlog | whale flow, OBI, reverb | In mapping JSON; non nel mix MVP attivo |
| Controlli UI | `stores/audioEngine.ts`, volume master | Solo IPC desktop |

**Regole realtime:** il callback cpal e un percorso RT rigido — solo pull; non bloccare mercato/UI sull'audio; **parita visivo-sonica** su CVD, OBI, VWAP.

**Doc privata:** `AUDIO_ENGINE.md`, `ARCHITECTTURA_FEED_UI.md`. Riferimenti di design per lane e spazializzazione: Curtis Roads, Julius O. Smith III.

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

- **Scalabilita:** layer ingestion, UI e DSP evolvono in modo indipendente.
- **Realtime:** audio lock-free sotto burst WebSocket; UI a ~20 Hz senza prezzi inventati.
- **Integrita prodotto:** stato vuoto onesto e diagnostica al posto di OHLC sintetici in errore.
- **DX cross-language:** IPC schema-first con test automatici Rust ↔ TypeScript.
- **Operativita:** diagnostica runtime (eta frame, heartbeat ingest) per triage in produzione.
- **Sicurezza:** vault cifrato; permessi exchange limitati al read-only di monitoraggio.

---

## Sfide affrontate

| Sfida | Approccio |
|-------|-----------|
| Throughput WS vs budget UI | Calcolo mercato a frequenza piena in Rust; IPC **deduplicato per firma** e **batchato** (~20 Hz) verso la WebView |
| Parita tra runtime | Un bundle Svelte; `feedController` commuta `ipc` / `web` / `idle` senza duplicare la logica prodotto |
| Coerenza visivo-sonica | Metriche order flow in Rust; moduli TS di parita per la demo browser; audio dalle stesse fonti dove spedito |
| Audio hard-realtime | Thread mercato → atomici → **SPSC** → lerp render → callback cpal **solo pull** (no lock/alloc nel callback) |
| Fiducia senza dati | `price = 0` fino al primo tick; path stale/errore → diagnostica, non candele mock |
| Portabilita desktop | Shell Tauri; fallback keyring su WSL; path audio ALSA documentato nel repo privato |

---

## Competenze evidenziate

- **System design full-stack:** desktop + web da un solo codice UI con confini backend chiari.
- **Rust di sistema:** ingestione concorrente, vault crittografico, DSP e mixer realtime.
- **Prodotto TypeScript/Svelte:** store reattivi, integrazione chart, UX ad alta densita informativa.
- **Integrazione API:** Binance Spot WSS/REST, user stream firmato opzionale, bootstrap kline.
- **Sviluppo guidato da contratto:** schema IPC versionato, test CI, disciplina camelCase/snake_case.
- **Domain modeling:** analytics order flow (CVD, OBI, VWAP Anchor, segnali cinetici/tape) con moduli testabili.
- **DSP audio applicato:** mapping di sonificazione, legge di pan spaziale, handoff buffer thread-safe sotto carico.

---

## Roadmap tecnica sintetica

- Hardening DSP multi-lane e spatializzazione.
- Harness replay per regression visive/audio.
- Modularizzazione UI per sito istituzionale.

---

*Maggio 2026. Documentazione utente: [`lang/README.it.md`](../../lang/README.it.md).*
