# Ergodika 풀스택 개발자 가이드

![언어](https://img.shields.io/badge/언어-한국어-ec4899?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← 문서 인덱스로](../ERGODIKA_FULLSTACK.md)

> 공개 아키텍처 브리핑. **Ergodika가 어떻게 구현되었는지** 개발자가 전체 파일을 읽기 전에 이해할 수 있도록 정리했습니다(애플리케이션 소스는 비공개 저장소).

## 대상 독자

아키텍처, 불변 조건, 모듈 경계가 필요한 **풀스택** 엔지니어 — 설치 튜토리얼이 아닙니다.

**한 줄 요약:** 데스크톱 마켓 콕핏, 고밀도 시각 분석, Binance 라이브 수집, 오더플로우 압력을 **소리로 표현**하는 데스크톱 오디오 MVP(CVD → pan, sentiment → pitch), 거래소에는 read-only.

---

## 목차

| 섹션 | 내용 |
|------|------|
| [시스템 개요](#시스템-개요) | 런타임과 데이터 흐름 |
| [설계 불변 조건](#설계-불변-조건) | 아키텍처 규칙 |
| [저장소 맵](#저장소-맵-로직-위치) | 비공개 트리 경로 |
| [IPC 계약](#ipc-계약rust--typescript) | FE/백엔드 정렬 |
| [마켓 수집](#마켓-수집-데스크톱-vs-브라우저) | Binance, 빈 상태 |
| [UI 아키텍처](#ui-아키텍처svelte) | Quartet, V-Matrix |
| [오디오 엔진](#오디오-엔진-데스크톱-mvp) | 스레드, 매핑 |
| [보안 모델](#보안-모델) | Vault, read-only |
| [테스트와 DX](#테스트와-dx) | 품질 도구 |
| [엔지니어링 하이라이트](#엔지니어링-하이라이트) | 확장성 |
| [로드맵](#기술-로드맵-요약) | 다음 단계 |

---

## 시스템 개요

**데스크톱 우선**(Tauri + SvelteKit + Rust) + **무음 브라우저 데모**. UI 코드는 하나; `isTauri()`와 `feedController`로 런타임 분기.

```
┌──────────────────────────────────────────────────────────────────┐
│  SvelteKit UI (데스크톱 네이티브 WebView)                           │
│  CockpitQuartet · V-Matrix · lightweight-charts · 진단           │
│  feedController: idle | ipc | web                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Tauri 이벤트 + invoke (데스크만)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Rust 백엔드                                                      │
│  live_feed/     → ErgodikaState 스냅샷 (~20 Hz UI 전송)          │
│  credentials/   → AES-256-GCM vault, Binance read-only         │
│  audio_engine/  → cpal 콜백, 4성부 믹서 (데스크톱 MVP)           │
└────────────────────────────┬─────────────────────────────────────┘
                             │ WSS + REST (Binance 공식 엔드포인트)
                             ▼
                      Binance Spot
```

**처리량 원칙:** Rust에서 WS 전체 속도로 계산; WebView IPC는 **서명 기반 dedup** + **시간 배치**(~20 Hz), 중간 가격을 만들지 않음.

### 코드 구성

1. **Rust 백엔드** = 틱, 오더플로우, vault, 오디오의 단일 진실 공급원.
2. **Svelte 프론트** = 스냅샷 소비만, 메인 경로에서 가짜 시세 없음.
3. **버전 IPC 계약**으로 레이어 연결.
4. **이중 런타임**: 전체 데스크톱 vs 무음 웹 데모.

---

## 설계 불변 조건

1. **메인 경로에 가짜 시세 금지.** 첫 실 tick 전 `price = 0`.
2. **거래소 read-only.** 모니터링·소리화만.
3. **schema-first IPC** (`ipc_contract/contract.json`).
4. **시각·청각 parity**(구현된 범위): CVD/OBI/VWAP.
5. **오디오 콜백 안전:** pull only, **lock-free SPSC**.
6. **QUARTET UI만** (2×2, 4심볼).

---

## 저장소 맵 (로직 위치)

비공개 `ergodika_app/`:

| 영역 | 경로 | 역할 |
|------|------|------|
| UI | `CockpitQuartet.svelte` 등 | 레이아웃 |
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts` | 실제 스냅샷 HUD |
| Web feed | `webBinanceHub.ts` | IPC 없을 때 WS |
| IPC feed | `ipcState.ts`, `quartet.ts` | listen + invoke |
| 타입 | `ergodikaState.ts` | Rust 미러 |
| IPC | `ipc_contract/contract.json` | 단일 계약 |
| Rust | `live_feed/` | WS, order flow |
| Audio | `audio_engine/` | cpal, mixer |

**추적 심볼:** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt`.

---

## IPC 계약 (Rust ↔ TypeScript)

- 버전 관리 `contract.json`.
- 이벤트: quartet + 선택 `MarketEvent`.
- Invoke: camelCase → snake_case.
- FE는 `IPC_INVOKE`만 사용.
- `ipc_contract_alignment_tests`로 CI 검증.

**Emit:** `ipc_quartet_signature` 변경 시(~50 ms) + 1 s heartbeat.

---

## 마켓 수집 (데스크톱 vs 브라우저)

| 모드 | 소스 |
|------|------|
| `ipc` | Rust `live_feed` |
| `web` | `webBinanceHub` |
| `idle` | 빈 상태 |

**오더플로우:** CVD, OBI, VWAP Anchor, Kinetic Impact, Beat Bang.

---

## UI 아키텍처 (Svelte)

SvelteKit 2, Svelte 5, Tailwind 4. `lightweight-charts` v5. UI 문구는 **영어**.

---

## 오디오 엔진 (데스크톱 MVP)

**브라우저 데모는 무음.** pitch ← sentiment; pan ← CVD(Blumlein). SPSC + render lerp + cpal pull.

---

## 보안 모델

AES-256-GCM, OS keyring, Binance WS API v3, FIFO PnL(데이터 일치 시).

---

## 테스트와 DX

Vitest, `cargo test --lib ipc_contract`, `npm run start` / `start:web`.

비공개 문서: `CONTINUITA.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## 엔지니어링 하이라이트

- ingestion/UI/DSP 레이어 분리.
- 버스트 환경에서 lock-free 오디오.
- 가짜 tick 없는 메인 경로.
- 타입 IPC + 정렬 테스트.

---

## 기술 로드맵 요약

- DSP lane 강화·공간화.
- replay 기반 회귀 테스트.
- 기관용 웹 재사용을 위한 UI 모듈화.

---

*2026년 5월. 사용자 문서: [`lang/README.ko.md`](../../lang/README.ko.md).*
