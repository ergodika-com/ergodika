# Ergodika для Full-Stack разработчиков

![Язык](https://img.shields.io/badge/Язык-Русский-f59e0b?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← К оглавлению](../ERGODIKA_FULLSTACK.md)

> Публичный архитектурный brief. Объясняет **как спроектирован и реализован Ergodika**, чтобы разработчик мог понять систему до чтения всего приватного репозитория.

## Для кого

**Full-stack** инженеры: архитектура, инварианты, границы модулей — не установочный туториал.

**Суть:** desktop cockpit с плотной визуальной аналитикой, live Binance и MVP аудио, который **озвучивает** давление order flow (CVD → pan, sentiment → pitch) в режиме read-only к бирже.

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
| [Audio](#audio-engine-mvp-desktop) | Потоки, mapping |
| [Безопасность](#модель-безопасности) | Vault, read-only |
| [Тесты и DX](#тестирование-и-dx) | Инструменты |
| [Highlights](#ключевые-инженерные-плюсы) | Масштабируемость |
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
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts` | HUD из реальных snapshot |
| Feed web | `webBinanceHub.ts`, `feedController.ts` | WS без IPC |
| Feed IPC | `ipcState.ts`, `quartet.ts` | listen + invoke |
| Типы | `ergodikaState.ts` | Зеркало Rust payload |
| IPC | `ipc_contract/contract.json` | Единый контракт |
| Rust feed | `src-tauri/src/live_feed/` | WS, order flow |
| Payload | `ergodika_payload.rs` | Подпись dedup |
| Account | `credentials_vault.rs` | Шифрование |
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

**Order flow:** CVD, OBI, VWAP Anchor, Kinetic Impact, Beat Bang.

---

## Архитектура UI (Svelte)

SvelteKit 2, Svelte 5, Tailwind 4. Store + `lightweight-charts` v5. Smoothing только визуальный. Диагностика: Last frame, Snapshot age. UI на **английском**.

---

## Audio engine (MVP desktop)

Браузер **без звука**. Потоки market/render, очередь SPSC, callback только pull. Pitch ← sentiment; pan ← CVD (Blumlein).

---

## Модель безопасности

AES-256-GCM, keyring OS, WebSocket API v3 Binance, FIFO PnL когда данные согласованы.

---

## Тестирование и DX

Vitest, `cargo test --lib ipc_contract`, `npm run start` / `start:web`.

Приватная документация: `CONTINUITA.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## Ключевые инженерные плюсы

- Изолированные слои ingestion / UI / DSP.
- Lock-free audio под burst.
- Честное пустое состояние.
- Типизированный IPC.

---

## Краткий roadmap

- Усиление DSP и spatialization.
- Replay/regression тесты.
- Модульность UI для web.

---

*Май 2026. Пользовательская документация: [`lang/README.ru.md`](../../lang/README.ru.md).*
