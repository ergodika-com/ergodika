# Ergodika 高级全栈开发者指南

![语言](https://img.shields.io/badge/语言-中文-9333ea?style=flat-square)

[English](ERGODIKA_FULLSTACK.en.md) | [Italiano](ERGODIKA_FULLSTACK.it.md) | [Русский](ERGODIKA_FULLSTACK.ru.md) | [中文](ERGODIKA_FULLSTACK.zh.md) | [한국어](ERGODIKA_FULLSTACK.ko.md)

[← 返回文档索引](../ERGODIKA_FULLSTACK.md)

> 公开架构说明。帮助高级开发者理解 **Ergodika 的工程实现方式**（完整源码在私有仓库），无需先阅读全部文件。

## 适用读者

需要掌握架构、不变量与模块边界的 **高级全栈工程师** — 非安装教程。

**一句话：** 桌面端市场驾驶舱，高密度可视化、Binance 实时数据，以及将订单流压力 **声音化** 的桌面音频 MVP（CVD → 声像，sentiment → 音高），对交易所保持只读。

---

## 目录

| 章节 | 内容 |
|------|------|
| [系统总览](#系统总览) | 运行时与数据流 |
| [设计不变量](#设计不变量不可违背) | 架构规则 |
| [仓库地图](#仓库地图逻辑所在位置) | 私有代码树路径 |
| [IPC 契约](#ipc-契约rust--typescript) | 前后端对齐 |
| [行情接入](#行情接入桌面-vs-浏览器) | Binance、空状态、订单流 |
| [UI 架构](#ui-架构svelte) | Quartet、V-Matrix、图表 |
| [音频引擎](#音频引擎桌面-mvp) | 线程与映射 |
| [安全模型](#安全模型) | Vault、只读账户 |
| [测试与 DX](#测试与-dx) | 质量保障 |
| [工程亮点](#工程亮点) | 可扩展性 |
| [路线图](#技术路线图简述) | 后续计划 |

---

## 系统总览

**桌面优先**（Tauri + SvelteKit + Rust），并提供 **静音浏览器演示**。一套 UI 代码；通过 `isTauri()` 与 `feedController` 切换两种运行时。

```
┌──────────────────────────────────────────────────────────────────┐
│  SvelteKit UI（桌面原生 WebView）                                  │
│  CockpitQuartet · V-Matrix · lightweight-charts · 诊断            │
│  feedController: idle | ipc | web                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ Tauri 事件 + invoke（仅桌面）
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  Rust 后端                                                        │
│  live_feed/     → ErgodikaState 快照（约 20 Hz 发往 UI）          │
│  credentials/   → AES-256-GCM 保险库、Binance 只读流              │
│  audio_engine/  → cpal 回调、4 路混音（桌面 MVP）                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │ WSS + REST（Binance 官方端点）
                             ▼
                      Binance 现货
```

**吞吐策略：** Rust 端按 WebSocket 全速率计算；发往 WebView 的 IPC **按签名去重** 并 **时间批处理**（约 20 Hz），不伪造中间价格。

### 代码组织方式

1. **Rust 后端** = 行情、订单流、保险库、音频的单一事实来源。
2. **Svelte 前端** = 只消费快照，主路径不伪造价格。
3. **版本化 IPC 契约** 连接两层。
4. **双运行时**：完整桌面 vs 静音 Web 演示。

---

## 设计不变量（不可违背）

1. **主路径禁止假行情。** 首个真实 tick 前：`price = 0`，明确空状态文案。
2. **对交易所只读。** 仅监控与声音化；API Key 应为 Binance 现货只读权限。
3. **IPC 以 schema 为先。** `ipc_contract/contract.json` 为唯一命令/事件定义。
4. **视听一致（已实现部分）。** CVD/OBI/VWAP 在 Rust 计算，TypeScript 模块保持 Web 演示 parity。
5. **音频回调安全。** cpal 回调仅 pull 样本；市场/渲染线程经 **无锁 SPSC** 队列。
6. **仅 QUARTET 产品 UI。** 2×2 四品种；已移除 SOLO；OMNI 目前仅为 manifest 标签。

---

## 仓库地图（逻辑所在位置）

私有树 `ergodika_app/` 典型结构：

| 区域 | 路径（示意） | 职责 |
|------|-------------|------|
| UI 外壳 | `CockpitQuartet.svelte` 等 | 布局与槽位 |
| V-Matrix | `Vmatrix*.svelte`, `marketSim.ts` | 由真实快照计算 HUD |
| Web 行情 | `webBinanceHub.ts` | 非 IPC 时 Binance WS |
| IPC 行情 | `ipcState.ts`, `quartet.ts` | listen + invoke |
| 类型 | `ergodikaState.ts` | 与 Rust 对齐 |
| IPC 契约 | `ipc_contract/contract.json` | 命令/事件单一来源 |
| Rust feed | `live_feed/` | WS、订单流、K 线 |
| 载荷 | `ergodika_payload.rs` | 去重签名 |
| 账户 | `credentials_vault.rs` | 加密与只读流 |
| 音频 | `audio_engine/` | cpal、混音 |
| Manifest | `app_manifest.json` | QUARTET 配置 |

**跟踪交易对：** `btcusdt`, `ethusdt`, `solusdt`, `xrpusdt`。

---

## IPC 契约（Rust ↔ TypeScript）

- 版本化 `contract.json`。
- 事件：quartet 快照 + 可选 `MarketEvent`。
- Invoke：JSON camelCase → Rust snake_case。
- 前端仅使用 `IPC_INVOKE`，禁止重复硬编码字符串。
- Rust 侧 `ipc_contract_alignment_tests` 做 CI 校验。

**Emit 策略：** 内存缓存始终更新；WebView 仅在 `ipc_quartet_signature` 变化时 emit（约 50 ms 间隔）+ 1 s 诊断心跳。

---

## 行情接入（桌面 vs 浏览器）

| 模式 | 数据源 |
|------|--------|
| `ipc` | Rust `live_feed` |
| `web` | `webBinanceHub` + 开发代理 `__binance` |
| `idle` | 空状态，无合成填充 |

**订单流（已交付）：** CVD、OBI、VWAP Anchor、Kinetic Impact、节拍网格 / Beat Bang。

---

## UI 架构（Svelte）

SvelteKit 2、Svelte 5、Tailwind 4。Store 管理高频快照；`lightweight-charts` v5；`vmatrixSmooth.ts` 仅视觉平滑。界面文案为 **英文**。

---

## 音频引擎（桌面 MVP）

**浏览器演示无声音。** 市场线程更新 pan/pitch；SPSC 队列；渲染线程 lerp；cpal 仅 pull。音高 ← sentiment；声像 ← CVD（Blumlein）。

---

## 安全模型

前端不存明文密钥；AES-256-GCM + 系统钥匙串；Binance WebSocket API v3 签名订阅；PnL 来自钱包与 FIFO `myTrades`（数据一致时）。

---

## 测试与 DX

Vitest、`cargo test --lib ipc_contract`、`npm run start` / `start:web`。

私有仓库文档：`CONTINUITA.md`、`AUDIO_ENGINE.md`、`USER_GUIDE.md` 等。

---

## 工程亮点

- 行情接入、UI、DSP 分层隔离，便于独立演进。
- 无锁音频路径，适应行情突发。
- 主路径拒绝假 tick，保护用户信任。
- 类型化 IPC + 对齐测试，降低 TS/Rust 协作成本。

---

## 技术路线图简述

- 多 lane DSP 与空间声像强化。
- 基于 replay 的视/听回归测试。
- UI 模块化以复用于官网展示。

---

*2026 年 5 月。用户文档：[`lang/README.zh.md`](../../lang/README.zh.md)。*
