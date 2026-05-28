# Ergodika 풀스택 개발자 가이드

![언어](https://img.shields.io/badge/언어-한국어-ec4899?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← 문서 인덱스로](../ERGODIKA_FULLSTACK.md)

> 공개 아키텍처 브리핑. **Ergodika가 어떻게 구현되었는지** 개발자가 전체 파일을 읽기 전에 이해할 수 있도록 정리했습니다(애플리케이션 소스는 비공개 저장소).

## 대상 독자

아키텍처, 불변 조건, 모듈 경계가 필요한 **풀스택** 엔지니어 — 설치 튜토리얼이 아닙니다.

**한 줄 요약:** 데스크톱 마켓 콕핏, 고밀도 시각 분석, Binance 라이브 수집, 오더플로우 압력을 **소리로 표현**하는 데스크톱 오디오 MVP(OBI → pan, 4/4 메트릭 그리드 → pitch), 거래소에는 read-only.

## 기술 스택

| 계층 | 기술 |
|------|------|
| 데스크톱 | **Tauri 2**(네이티브 WebView) |
| 프론트 | **SvelteKit 2**, **Svelte 5**, **TypeScript**, **Tailwind 4** |
| 백엔드 | **Rust** — `live_feed/`, `credentials/`, `audio_engine/` |
| 차트 | **lightweight-charts** v5 |
| 오디오 | **cpal**, lock-free **SPSC**(`rtrb`), Rust 실시간 DSP |
| 시장 데이터 | **Binance Spot**(WSS+REST) |
| 보안 | **AES-256-GCM** vault, OS keyring |
| 품질 | **Vitest**, 버전 IPC 계약 + Rust 정렬 테스트 |

**문서 범위:** 공개 문서 저장소의 아키텍처·엔지니어링 결정. 앱 소스는 비공개 `ergodika_app`에 있으며, 본 brief는 코드 없이 시스템 설계를 평가할 수 있게 합니다.

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
| [오디오 엔진](#오디오-엔진-데스크톱-mvp) | 스레드, DSP 파이프라인, 매핑 |
| [보안 모델](#보안-모델) | Vault, read-only |
| [테스트와 DX](#테스트와-dx) | 품질 도구 |
| [엔지니어링 하이라이트](#엔지니어링-하이라이트) | 확장성 |
| [엔지니어링 과제](#해결한-엔지니어링-과제) | 난제와 트레이드오프 |
| [역량](#프로젝트가-보여주는-역량) | 리뷰·면접 관점 |
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
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts`, `scalpingScorecard.ts` | 실제 스냅샷 HUD; UI 전용 scalping scorecard |
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

**오더플로우(출시):** CVD, OBI, VWAP Anchor, Kinetic Impact, tape activity / Beat Bang.

Rust: `order_flow_delta_analyzer`, `cvd_session_scale`, `vwap_session`, `kinetic_impact_sensor`, `tape_activity`.  
Web parity: `orderFlowMath.ts`, `cvdSessionScale.ts`, `vwapSession.ts`, `kineticImpactSensor.ts`, `tapeActivitySensor.ts`.

---

## UI 아키텍처 (Svelte)

- SvelteKit 2, Svelte 5, Tailwind 4, ultra-dark glassmorphism.
- **QUARTET 콕핏:** 2×2 네 심볼; timeframe·interval·Sound 레일.
- **V-Matrix HUD:** position in range, whale flow, spread, CVD, OBI, VWAP anchor, kinetic impact 등 정규화 order-flow 메트릭, **Flow Direction** 라벨, **Scalping score** — **라이브 스냅샷**만으로 `computeVMatrixSnapshot`, 메인 경로에 시뮬 tick 없음.
- Store: `quartet.ts`, `feedController`, `audioEngine.ts`, `masterTempo.ts`.
- 차트: `lightweight-charts` v5; OHLC 부트스트랩 ~10k.
- `vmatrixSmooth.ts`는 시각 폴리시만.
- 진단: `RuntimeDiagnostics.svelte`. UI 문구 **영어**.
- 슬롯: `quartetChartSlots` + manifest.

### Scalping score

UI 전용 **pre-entry scorecard**: 스무딩된 `VMatrixSnapshot`을 `computeScalpingScorecard`(`scalpingScorecard.ts`)로 계산. Rust IPC payload **미포함**; **거래 신호 아님**.

| 항목 | 내용 |
|------|------|
| 총점 | **0–100**, lane 고정 가중치 |
| 가중치 | spread **20**, impact stability **10**, OBI **15**, CVD + OBI/CVD 일치 **20**, whale flow **10**, tape activity **25** |
| 밴드 | **A**(≥80), **B**(65–79), **C**(50–64), **NO_TRADE**(<50 또는 gate) |
| 보수 gate | spread 과도(`Vs ≥ 0.82`), activity 저하(`activity01 ≤ 0.22`), OBI/CVD 역방향 시 **NO_TRADE** |
| UI | `VmatrixSlotColumn.svelte` |
| vs Flow Direction | **독립** — score는 실행 품질; Flow Direction은 공격적 매수/매도 압력 합의 |

---

## 오디오 엔진 (데스크톱 MVP)

**브라우저 데모는 무음.** 실시간 **오디오 DSP**는 Rust(`src-tauri/src/audio_engine/`)에서만 동작합니다. Svelte는 **오디오 비의존**: IPC 제어만(`audioEngine.ts`, Sound 탭); `src/lib/audio/bridge.ts`는 계약 정렬 / DEV 로그용이며 JS 합성은 없습니다.

| 구성 | 역할 |
|------|------|
| 마켓 스레드 | 오더플로우 지표로 원자 pan/pitch 타깃 갱신 |
| SPSC `rtrb` | ~112 ms 버퍼; 콜백 비차단 |
| 렌더 스레드 | pitch/pan lerp ~320 ms, `sin` 오실레이터 |
| cpal 콜백 | pull 전용, 소프트 리미터 |
| 믹서 | 4성부 ÷4, 뮤트 램프 ~18 ms |

### 오디오 DSP 파이프라인

```
live_feed / order-flow (Rust)
    → 원자 pan & pitch 타깃
    → lock-free SPSC (~112 ms)
    → 렌더: lerp + 사인 보이스
    → 4성부 믹서 + 소프트 리미터
    → cpal pull (락·힙 할당 없음)
```

| DSP 계층 | 위치 | 비고 |
|----------|------|------|
| 매핑 계약 | `audio_mapping.json`, `docs/AUDIO_ENGINE.md` | lane은 제품 결정 후 구현 |
| 출시 sonification | OBI → **pan**(Blumlein); 4/4 메트릭 그리드 + 슬롯 fundamental → **pitch**; kinetic impact → strike overlay | HUD와 동일 Rust 지표(필요 시) |
| 백로그 lane | whale, OBI, reverb | mapping JSON에만; 현재 MVP 믹스 미포함 |
| UI 제어 | `stores/audioEngine.ts`, 마스터 볼륨 | 데스크톱 IPC만 |

**실시간 규칙:** cpal 콜백은 하드 RT 경로(pull만); 마켓/UI를 오디오에 막지 않음; CVD·OBI·VWAP **시청각 일치**.

**비공개 문서:** `AUDIO_ENGINE.md`, `ARCHITECTTURA_FEED_UI.md`. 공간화·lane 설계 참고: Curtis Roads, Julius O. Smith III.

---

## 보안 모델

프론트엔드에 평문 시크릿 없음; BYOK 활성화 시 AES-256-GCM + OS keyring(WSL 파일 fallback).

---

## 테스트와 DX

Vitest, `cargo test --lib ipc_contract`, `npm run start` / `start:web`.

비공개 문서: `CONTINUITA.md`, `AUDIO_ENGINE.md`, `USER_GUIDE.md`.

---

## 엔지니어링 하이라이트

- **확장성:** ingestion·UI·DSP 레이어 독립 진화.
- **실시간:** WS 버스트에서 lock-free 오디오; UI ~20 Hz, 가격 조작 없음.
- **제품 무결성:** 정직한 빈 상태·진단, 오류 시 합성 OHLC 없음.
- **DX:** schema-first IPC, Rust↔TS 자동 정렬 테스트.
- **운영:** 런타임 진단(프레임 age, ingest heartbeat).
- **보안:** 암호화 vault; 거래소 read-only 모니터링 scope.

---

## 해결한 엔지니어링 과제

| 과제 | 접근 |
|------|------|
| WS 처리량 vs UI | Rust 전속도 계산; IPC **서명 dedup** + **배치**(~20 Hz) |
| 런타임 parity | 단일 Svelte 번들; `feedController` `ipc`/`web`/`idle` |
| 시청각 일치 | Rust order-flow; web TS parity; 출시 범위에서 동일 소스 |
| 하드 RT 오디오 | atomics → **SPSC** → lerp → cpal **pull only**(콜백 lock/alloc 없음) |
| 데이터 부재 신뢰 | 첫 tick 전 `price = 0`; stale는 진단, mock 캔들 아님 |
| 이식성 | Tauri; WSL keyring fallback; ALSA는 비공개 ops 문서 |

---

## 프로젝트가 보여주는 역량

- **풀스택 시스템 설계:** 단일 UI로 데스크톱+무음 웹, 백엔드 경계 명확.
- **Rust:** 동시 ingestion, 암호 vault, 실시간 DSP/믹서.
- **TypeScript/Svelte:** 반응형 store, 차트, 고밀도 UX.
- **API 통합:** Binance Spot WSS/REST, K-line bootstrap.
- **계약 주도 개발:** 버전 IPC, CI alignment tests.
- **도메인 모델링:** CVD, OBI, VWAP Anchor, kinetic/tape 신호.
- **응용 오디오 DSP:** sonification mapping, spatial pan, thread-safe handoff.

---

## 기술 로드맵 요약

- DSP lane 강화·공간화.
- replay 기반 회귀 테스트.
- 기관용 웹 재사용을 위한 UI 모듈화.

---

*2026년 5월. 사용자 문서: [`lang/README.ko.md`](../../lang/README.ko.md).*
