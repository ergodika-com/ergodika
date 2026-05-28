# Ergodika для Full-Stack разработчиков

![Язык](https://img.shields.io/badge/Язык-Русский-f59e0b?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← К оглавлению](../ERGODIKA_FULLSTACK.md)

> Публичный архитектурный brief. Объясняет **как спроектирован и реализован Ergodika**, чтобы разработчик мог понять систему до чтения всего приватного репозитория.

## Для кого

**Full-stack** инженеры: архитектура, инварианты, границы модулей — не установочный туториал.

**Суть:** desktop cockpit с плотной визуальной аналитикой, live Binance и MVP аудио, который **озвучивает** давление order flow (OBI → pan, сетка 4/4 → pitch) в режиме read-only к бирже.

## Технологический стек

| Слой | Технологии |
|------|------------|
| Desktop | **Tauri 2** (нативная WebView) |
| Frontend | **SvelteKit 2**, **Svelte 5**, **TypeScript**, **Tailwind 4** |
| Backend | **Rust** — `live_feed/`, `credentials/`, `audio_engine/` |
| Графики | **lightweight-charts** v5 |
| Аудио | **cpal**, lock-free **SPSC** (`rtrb`), realtime DSP в Rust |
| Рынок | **Binance Spot** (WSS + REST) |
| Безопасность | Vault **AES-256-GCM**, OS keyring |
| Качество | **Vitest**, версионированный **IPC** + alignment-тесты |

**Область документа:** архитектурные решения в публичном репозитории документации. Исходники приложения — в приватном `ergodika_app`; brief позволяет оценить дизайн без доступа к коду.

---

## Содержание

| Раздел | О чем |
|--------|--------|
| [Система в одной схеме](#система-в-одной-схеме) | Слои и поток данных |
| [Инварианты](#инварианты-дизайна-обязательные) | Правила проектирования |
| [Карта репозитория](#карта-репозитория-где-искать-код) | Пути в приватном tree |
| [IPC-контракт](#ipc-контракт-rust--typescript) | Связь FE и backend |
| [Ingestion](#ingestion-рынка-desktop-vs-browser) | Binance, пустое состояние |
| [UI](#архитектура-ui-svelte) | Quartet, V-Matrix, графики |
| [Audio](#audio-engine-mvp-desktop) | Потоки, DSP, mapping |
| [Безопасность](#модель-безопасности) | Vault, read-only |
| [Тесты и DX](#тестирование-и-dx) | Инструменты |
| [Highlights](#ключевые-инженерные-плюсы) | Масштабируемость |
| [Инженерные задачи](#решенные-инженерные-задачи) | Сложные проблемы |
| [Навыки](#демонстрируемые-навыки) | Что видно при review |
| [Roadmap](#краткий-roadmap) | Дальше |

---

## Система в одной схеме

**Desktop-first** cockpit (Tauri + SvelteKit + Rust) и **беззвучная web demo**. Один UI; два runtime через `isTauri()` и `feedController`.

```
┌──────────────────────────────────────────────────────────────────┐
│  SvelteKit UI (нативная WebView на desktop)                       │
│  CockpitQuartet · V-Matrix · lightweight-charts · диагностика     │
│  feedController: idle | ipc | web                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Tauri events + invoke (только desktop)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Rust backend                                                     │
│  live_feed/     → снимки ErgodikaState (~20 Hz в UI)              │
│  credentials/   → vault AES-256-GCM, Binance read-only stream     │
│  audio_engine/  → cpal, mixer 4 голоса (MVP desktop)              │
└────────────────────────────┬─────────────────────────────────────┘
                             │ WSS + REST (официальные endpoint Binance)
                             ▼
                      Binance Spot
```

**Throughput:** расчеты на полной частоте WS в Rust; IPC в WebView **дедуплицируется по подписи** и **батчится** (~20 Hz) без выдуманных цен.

### Организация кода

1. **Rust backend** — источник истины для тиков, order flow, vault, audio.
2. **Svelte frontend** — наблюдатель без поддельных цен.
3. **Версионированный IPC-контракт** между слоями.
4. **Два runtime** из одного бандла.

---

## Инварианты дизайна (обязательные)

1. **Никаких фейковых данных на main path.** До первого тика: `price = 0`. Ошибки → диагностика, не синтетический OHLC.
2. **Read-only к бирже.** Только мониторинг и сонификация.
3. **IPC schema-first** в `ipc_contract/contract.json`.
4. **Паритет визуал ↔ звук** для CVD/OBI/VWAP (где реализовано).
5. **Безопасный audio callback:** только pull; **SPSC lock-free**.
6. **Только QUARTET UI** (2×2, 4 символа).

---

## Карта репозитория (где искать код)

Приватное дерево `ergodika_app/`:

| Область | Путь | Задача |
|---------|------|--------|
| UI | `CockpitQuartet.svelte`, `QuartetSlotCard.svelte` | Layout |
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts`, `scalpingScorecard.ts` | HUD из snapshot; UI-only scalping scorecard |
| Feed web | `webBinanceHub.ts`, `feedController.ts` | WS без IPC |
| Feed IPC | `ipcState.ts`, `quartet.ts` | listen + invoke |
| Типы | `ergodikaState.ts` | Зеркало Rust payload |
| IPC | `ipc_contract/contract.json` | Единый контракт |
| Rust feed | `src-tauri/src/live_feed/` | WS, order flow |
| Payload | `ergodika_payload.rs` | Подпись dedup |
| Credentials | `credentials_vault.rs` | AES-256-GCM vault, OS keyring (BYOK при включении) |
| Audio | `audio_engine/` | cpal, mixer |
| Manifest | `app_manifest.json` | QUARTET |

**Символы:** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt`.

---

## IPC-контракт (Rust ↔ TypeScript)

- JSON `contract.json` (versioned).
- События quartet + опционально `MarketEvent`.
- Invoke: camelCase → snake_case в `lib.rs`.
- FE: только `IPC_INVOKE`.
- Тесты: `ipc_contract_alignment_tests`.

**Emit:** emit при изменении `ipc_quartet_signature` (~50 ms) + heartbeat 1 с.

---

## Ingestion рынка (desktop vs browser)

| Режим | Источник |
|-------|----------|
| `ipc` | Rust `live_feed` |
| `web` | `webBinanceHub` |
| `idle` | Пустое состояние |

**Order flow (в проде):** CVD, OBI, VWAP Anchor, Kinetic Impact, tape activity / Beat Bang.

Rust: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `kinetic_impact_sensor`, `tape_activity`.  
Паритет web: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, `kineticImpactSensor.ts`, `tapeActivitySensor.ts`.

---

## Архитектура UI (Svelte)

- SvelteKit 2, Svelte 5, Tailwind 4, ultra-dark glassmorphism.
- **QUARTET cockpit:** сетка 2×2 на четыре символа; rail для timeframe, interval, Sound.
- **V-Matrix HUD:** нормализованные метрики order flow (position in range, whale flow, spread, CVD, OBI, VWAP anchor, kinetic impact), метки **Flow Direction** и **Scalping score** из **живых snapshot** (`computeVMatrixSnapshot`) — без симулированного tick-loop на main path.
- Store: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- Графики: `lightweight-charts` v5; bootstrap OHLC до ~10k баров.
- Smoothing: `vmatrixSmooth.ts` — только визуальный polish.
- Диагностика: `RuntimeDiagnostics.svelte`. UI на **английском**.
- **Слоты:** wiring через `quartetChartSlots` + manifest.

### Scalping score

UI-only **pre-entry scorecard** из сглаженного `VMatrixSnapshot` через `computeScalpingScorecard` (`scalpingScorecard.ts`). **Не** в Rust IPC payload; **не** торговый сигнал.

| Элемент | Детали |
|---------|--------|
| Итог | **0–100** с фиксированными весами lane |
| Веса | Spread **20**, impact stability **10**, OBI **15**, CVD + согласованность OBI/CVD **20**, whale flow **10**, tape activity **25** |
| Полосы | **A** (≥80), **B** (65–79), **C** (50–64), **NO_TRADE** (<50 или gate) |
| Gate | **NO_TRADE** при широком spread (`Vs ≥ 0.82`), низкой activity (`activity01 ≤ 0.22`) или расхождении OBI/CVD |
| UI | `VmatrixSlotColumn.svelte` |
| vs Flow Direction | **Независимы** — score = качество исполнения; Flow Direction = консенсус агрессивного потока |

---

## Audio engine (MVP desktop)

Браузер **без звука.** Весь realtime **audio DSP** в Rust (`src-tauri/src/audio_engine/`). Svelte **audio-agnostic**: только IPC (`audioEngine.ts`, вкладка Sound); `src/lib/audio/bridge.ts` — паритет контракта / DEV-лог, без синтеза в JS.

| Узел | Роль |
|------|------|
| Поток market | Атомарные цели pan/pitch из order flow |
| SPSC `rtrb` | ~112 ms; не блокирует callback |
| Поток render | Lerp pitch/pan ~320 ms, осцилляторы `sin` |
| Callback cpal | Только pull, мягкий limiter |
| Mixer | 4 голоса ÷ 4, mute ~18 ms |

### Конвейер audio DSP

```
live_feed / order-flow (Rust)
    → атомарные pan & pitch
    → lock-free SPSC (~112 ms)
    → render: lerp + синусоиды
    → mixer 4 голосов + limiter
    → cpal pull (без lock и heap alloc)
```

| Слой DSP | Где | Заметки |
|----------|-----|---------|
| Контракт mapping | `audio_mapping.json`, `docs/AUDIO_ENGINE.md` | Lane по решению продукта |
| В проде | OBI → **pan** (Blumlein); сетка 4/4 + fundamental слота → **pitch**; kinetic impact → strike overlay | Паритет с HUD на тех же метриках |
| Backlog | whale, OBI, reverb | В JSON mapping; не в активном MVP mix |
| UI | `stores/audioEngine.ts`, master volume | Только desktop IPC |

**Realtime:** callback cpal — жёсткий RT-путь (только pull); не блокировать market/UI; **визуально-звуковой паритет** CVD, OBI, VWAP.

**Приватная документация:** `AUDIO_ENGINE.md`, `ARCHITECTTURA_FEED_UI.md`. Референсы DSP: Curtis Roads, Julius O. Smith III.

---

## Модель безопасности

Секреты не хранятся в открытом виде во frontend; AES-256-GCM + OS keyring (fallback на WSL) при включённом BYOK.

---

## Тестирование и DX

Vitest, `cargo test --lib ipc_contract`, `npm run start` / `start:web`.

Приватная документация: `CONTINUITA.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## Ключевые инженерные плюсы

- **Масштабируемость:** независимая эволюция ingestion, UI и DSP.
- **Realtime:** lock-free audio при burst WS; UI ~20 Hz без выдуманных цен.
- **Целостность продукта:** честное пустое состояние вместо синтетического OHLC при ошибках.
- **DX:** schema-first IPC с автоматическими alignment-тестами Rust ↔ TypeScript.
- **Операции:** runtime-диагностика (возраст кадра, heartbeat ingest).
- **Безопасность:** шифрованный vault; read-only scopes к бирже.

---

## Решенные инженерные задачи

| Задача | Решение |
|--------|---------|
| WS throughput vs UI | Полная частота расчётов в Rust; IPC **dedup по подписи** + **batch** (~20 Hz) |
| Паритет runtime | Один Svelte bundle; `feedController`: `ipc` / `web` / `idle` |
| Визуал ↔ звук | Order-flow метрики в Rust; TS parity для web; audio из тех же источников |
| Hard-realtime audio | Atomics → **SPSC** → lerp → **pull-only** cpal (без lock/alloc в callback) |
| Доверие без данных | `price = 0` до первого тика; stale → диагностика, не mock candles |
| Портативность | Tauri; fallback keyring на WSL; ALSA в приватной ops-документации |

---

## Демонстрируемые навыки

- **Full-stack system design:** desktop + web из одного UI с чёткими границами backend.
- **Rust:** конкурентный ingestion, криптографический vault, realtime DSP/mixer.
- **TypeScript/Svelte:** реактивные store, charts, UX высокой плотности информации.
- **API integration:** Binance Spot WSS/REST, bootstrap kline.
- **Contract-driven dev:** версионированный IPC, CI alignment tests.
- **Domain modeling:** CVD, OBI, VWAP Anchor, kinetic/tape signals.
- **Applied audio DSP:** sonification mapping, spatial pan, thread-safe handoff.

---

## Краткий roadmap

- Усиление DSP и spatialization.
- Replay/regression тесты.
- Модульность UI для web.

---

*Май 2026. Пользовательская документация: [`lang/README.ru.md`](../../lang/README.ru.md).*
