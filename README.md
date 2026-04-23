[Hackaday.io Project Page.md](https://github.com/user-attachments/files/27013557/Hackaday.io.Project.Page.md)
# FPGA_Spectrum_Engine# FPGAスペクトルエンジンの展望
## FPGA Spectrum Engine — Vision & Architecture

> **10,240 independent oscillators. 1-sample latency. 0.001 Hz resolution.**  
> A universal spectrum engine — FM synthesis, fractal audio, and beyond.  
>
> **10,240本の独立オシレータ。1サンプル遅延。0.001Hz分解能。**  
> FM音源・スペクトルフラクタル合成・音楽シーン直接合成への汎用基盤。

---

## ▶ Interactive Project Page / インタラクティブプロジェクトページ

**[→ Open Full Project Page](https://YOUR-GITHUB-PAGES-URL-HERE)**  
*(Spectrum waterfall animation, full bilingual documentation)*

---

## What is this? / これは何か？

This project is a **real-time additive synthesis engine** implemented on FPGA (Terasic DE10-nano, Cyclone V SoC), capable of running **10,240 independent sinusoidal oscillators simultaneously** with:

- **1-sample output latency** (20 µs at 48 kHz)
- **0.001 Hz frequency resolution** per bin
- **Constant compute load** — silence and full musical scenes require identical processing

このプロジェクトは、FPGA（Terasic DE10-nano、Cyclone V SoC）上に実装されたリアルタイム加算合成エンジンです。**10,240本の独立した正弦波オシレータ**を同時に動作させ、以下の性能を実現します：

- **1サンプル出力遅延**（48kHzで20µs）
- **ビンあたり0.001Hz周波数分解能**
- **常時一定の演算負荷** — 無音でも音楽シーン合成中でも、秒間500億回の演算は変わらない

The key insight: **by controlling only the bin parameters (frequency, amplitude, phase) of an iDFT engine, every known synthesis paradigm becomes a special case** — FM synthesis, additive synthesis, polygonal synthesis, spectral fractal synthesis, and more.

核心的な洞察：**iDFTエンジンのビンパラメータ（周波数・振幅・位相）の与え方だけで、あらゆる既知の合成パラダイムを包含できる。** FM合成・加算合成・ポリゴナル合成・スペクトルフラクタル合成がすべて同一ハードウェアの特殊ケースとなる。

---

## Hardware / ハードウェア

| Item | Spec |
|------|------|
| **Board** | Terasic DE10-nano (Cyclone V SoC 5CSEBA6U23I7) |
| **FPGA Fabric** | 110k LEs, 112 DSP blocks |
| **ARM HPS** | Dual Cortex-A9 @ 800 MHz |
| **Sample Rate** | 48 kHz |
| **Bin Count** | 10,240 (2,048 bins × 5 parallel modules) |
| **Clock** | 100 MHz per module |
| **Output** | 24-bit DAC via I²S |
| **Control I/F** | Gigabit Ethernet (UDP), AXI bridge ARM↔FPGA |

### Prototype (2020) / 原型（2020年）

The physical layer was first implemented on a **Terasic C5G board** in autumn 2020 as an 80-voice polyphonic additive synthesizer with 128 bins, controlled via MIDI-CC from MAX8. This prototype **demonstrated the core compute architecture** and the fundamental viability of the approach.

物理層は2020年秋、**Terasic C5Gボード**上に、MAX8からMIDI-CCで制御する80音ポリフォニック128ビン加算合成シンセサイザーとして実装・完成済みです。これは本アーキテクチャの計算基盤の実現可能性を実証しています。

---

## Core Implementation: Why 11th-Order Maclaurin? / なぜ11次マクローリン展開か？

Most FPGA sine generators use ROM lookup tables or CORDIC. This engine uses **direct 11th-order Maclaurin expansion via Horner's method**:

大多数のFPGA正弦波生成器はROMテーブルまたはCORDICを使用します。本エンジンは**Horner法による11次マクローリン展開を直接計算**します：

```
sin(x) ≈ x − x³/3! + x⁵/5! − x⁷/7! + x⁹/9! − x¹¹/11!
Error ≈ |x|¹³/13! ≈ 1.3×10⁻⁷  (≈ 23-bit precision)
```

**Why this choice / この選択の理由：**

1. **Verifiability** — Coefficients are 1/n!, a mathematical necessity. Any engineer can verify, understand, and port the implementation without black boxes.  
   **検証可能性** — 係数は1/n!という数学的必然。誰でも検証・理解・移植できる。

2. **Pipeline-natural** — Horner's method maps perfectly to a fixed-latency DSP pipeline, one result per clock cycle.  
   **パイプラインとの親和性** — Horner法は固定レイテンシDSPパイプラインに完璧に対応し、毎クロック1演算が得られる。

3. **No memory contention** — 2,048 bins sharing a single pipeline avoids the BRAM port conflicts inherent in ROM-based approaches.  
   **メモリ競合なし** — 2,048ビンが単一パイプラインを時分割し、ROMベースのBRAMポート競合を完全回避。

4. **Phase precision preserved** — Input phase is fed directly to the polynomial; no quantization loss from table indexing.  
   **位相精度の保持** — 位相を多項式に直接入力するためテーブルインデックス量子化損失がない。

### Module Architecture / モジュールアーキテクチャ

```
100 MHz pipeline × 5 parallel modules = 500 MHz equivalent throughput

Each module:
  [Phase Acc] → [Horner Pipeline, 11 stages] → [sin/cos out]
      ↓                                               ↓
  [NCO freq reg]                              [Amplitude × bin]
                                                      ↓
                                           [Adder tree, 2048→1]

5 modules → [Final adder] → [48 kHz DAC output]
```

**Constant compute / 常時演算：**
```
100 MHz × 5 modules × ~20 ops/bin = ~10,000,000,000 ops/sec
This number does not change whether the output is silence or a full orchestral scene.
この数値は、出力が無音でも壮大な音楽シーンでも変わらない。
```

---

## 3-Layer Architecture / 3層アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│  LAYER 01: PC (Abstraction / 抽象層)             │
│  - High-level sound design language              │
│  - Open Prompt: LLM-collaborative bin generation │
│  - Musical scene description, sequencer, UI      │
└────────────────────┬────────────────────────────┘
                     │ GbE · UDP stream
┌────────────────────▼────────────────────────────┐
│  LAYER 02: ARM HPS (Middle / ミドル層)           │
│  - Abstract → 10,240 bin expansion              │
│  - Note events, polyphony management             │
│  - Per-bin envelope / LFO @ 1 kHz               │
│  - AXI bridge DMA to FPGA                        │
└────────────────────┬────────────────────────────┘
                     │ AXI bridge · GB/s on-chip
┌────────────────────▼────────────────────────────┐
│  LAYER 03: FPGA (Physical / 物理層)              │
│  - 5 × 2,048-bin Maclaurin pipeline modules     │
│  - 32-bit+ phase accumulators (NCO)              │
│  - 10,240-bin adder tree → 48 kHz/24-bit DAC    │
└─────────────────────────────────────────────────┘
```

---

## Synthesis Paradigms as Special Cases / 合成パラダイムの統一

Because bin parameters are fully programmable, every synthesis method becomes a **parameter assignment problem**:

ビンパラメータが完全に自由に設定できるため、あらゆる合成手法が**パラメータ配置問題**に還元される：

| Synthesis Method | Bin Frequency | Bin Amplitude |
|-----------------|---------------|---------------|
| **FM Synthesis** | ωc ± nωm | Jₙ(β) |
| **Polygonal + Bessel** | (kN+1)ω₀ − nωm | cₖ(N)·Jₙ(kNβ) |
| **Geometric / Fractal** | f₀·rᵏ | r^(−αk) |
| **Cantor Spectrum** | Cantor set positions | Cantor measure |
| **1/fᵅ Noise** | Log-spaced | fₖ^(−α/2) |
| **Shepard Tone** | f₀·2ᵏ | Bell envelope |
| **Physical Model** | Mode frequencies | Modal amplitudes |

### The FM Connection / FMとの接続

The classic Chowning FM synthesis equation:

```
sin(ωct + β·sin ωmt) = Σ Jₙ(β)·sin[(ωc + nωm)t]
```

means **FM sideband amplitudes are Bessel function values Jₙ(β)**. In this engine, those bin amplitudes are set directly — FM becomes one special case of a far more general spectral control system.

古典的なChowning FM合成方程式は、**FM側波帯振幅がベッセル関数値Jₙ(β)であること**を意味します。本エンジンではその振幅を直接設定できるため、FMはより汎用的なスペクトル制御系の特殊ケースとなります。

---

## iDFT / SDFT Duality — Toward Robotic Sensing / iDFT・SDFT二重性とロボティクス応用

An iDFT bin and an SDFT (Sliding DFT) bin share nearly identical hardware resources:

iDFTビンとSDFT（スライディングDFT）ビンは、ほぼ同一のハードウェアリソースを共有します：

```
iDFT:  yk[n] = Ak · e^j(ωkn + φk)           (synthesis output)
SDFT:  Xk[n] = e^j2πk/N · (Xk[n-1] + x[n] − x[n-N])  (analysis input)
```

A **mixed iDFT/SDFT bin pool** enables:
- Active sensing: emit a selective-frequency impulse via iDFT, analyze the piezo response via SDFT
- Vibration analysis at 0.001 Hz resolution for material recognition
- Potential application as **high-resolution robotic tactile sensors** (fingertip / sole)

**iDFT/SDFT混在ビン構成**により：
- 能動センシング：iDFTで選択周波数インパルスを出力し、SDFTでピエゾ素子の応答を解析
- 0.001Hz分解能の振動解析による素材認識
- **高分解能ロボット触覚センサ**（指先・足裏）への応用可能性

This creates a direct connection between this audio synthesis project and **state-of-the-art robotics research** — a natural collaboration bridge for STEAM education contexts.

これはこの音響合成プロジェクトと**最先端ロボット研究**の直接的な接続点となり、STEAM教育でのコース間コラボレーションの架け橋になります。

---

## Open Prompt — Beyond Open Source / オープンプロンプト：オープンソースを超えて

This project introduces the concept of **Open Prompt** as a new paradigm for knowledge sharing in the LLM era:

本プロジェクトは、LLM時代の知識共有の新しいパラダイムとして**オープンプロンプト**という概念を提唱します：

| | Open Source | Open Prompt |
|--|-------------|-------------|
| **What is distributed** | Source code | Architecture + mathematics |
| **Reproduction** | Fork and modify | Regenerate with LLM |
| **Ownership** | Derivative of original | Each implementer owns a genuine original |
| **Commercial use** | License-governed | Structurally free — no obligations |
| **配布物** | ソースコード | アーキテクチャ＋数理記述 |
| **再構成方法** | フォークして改変 | LLMと共に再生成 |
| **所有権** | 元コードの派生物 | 各実装者が真のオリジナルを所有 |
| **商用利用** | ライセンスに従う | 構造的に自由—気兼ね不要 |

The 11th-order Maclaurin coefficients are `1/n!` — a mathematical necessity, not a creative choice. Anyone who understands the architecture can regenerate their own implementation with any LLM. **Code is ephemeral; the knowledge architecture is the commons.**

11次マクローリンの係数は`1/n!`——創造的選択ではなく数学的必然。アーキテクチャを理解した者は誰でもLLMで自分の実装を再生成できる。**コードは一時的生成物、知識こそが共有財産。**

**Pioneer advantage / 先駆者の利益：** The more the community grows, the greater the demand for systematic FPGA education, books, and workshops that only the original architect can provide. Opening the design *creates* the market for teaching it.  
コミュニティが育つほど、体系化されたFPGA教育・書籍・ワークショップへの需要が生まれる。設計を開放することが、それを教える市場を創出する。

---

## Synthesis Paradigm Lineage / 合成パラダイムの系譜

```
1973  John Chowning (Stanford CCRMA) → FM Synthesis → Yamaha DX7 (1983)
         Sidebands at ωc ± nωm, amplitudes = Jₙ(β)

2011  ERRORSMITH (Erik Wiegand) → Razor (Native Instruments Reaktor)
         320 directly-controlled partials, spectral-domain FM computation

2020  This project (C5G prototype)
         2,048 bins × 5 modules = 10,240 bins, hardware-verified
         FM and Razor are now special cases

NEXT  Spectral Fractal Engine (DE10-nano, 3-layer architecture)
         Geometric, Cantor, 1/fα, Shepard — by bin parameter assignment alone
```

---

## STEAM Education Potential / STEAM教育としての可能性

Unlike typical robotics-kit STEAM programs, this project offers:

典型的なロボットキットSTEAMプログラムと異なり、本プロジェクトは以下を提供します：

- **No black boxes** — "Why does sound come out?" leads directly to trigonometry and Fourier analysis  
  **ブラックボックスなし** — 「なぜ音が出るか」は三角関数とフーリエ解析に直結する

- **No library memorization** — Understanding replaces memorization  
  **暗記不要** — ライブラリ暗記ではなく原理理解で進む

- **Continuum with SOTA** — What a student builds shares the same mathematical principles as Razor and this engine  
  **最先端との連続性** — 学習者が作るものは最先端と同じ数学的原理を持つ

- **Immediate aesthetic feedback** — "Does it sound good?" is answered in one second  
  **即時美的フィードバック** — 「良い音か」は1秒で判断できる

- **Cross-disciplinary** — Mathematics, physics, electronics, music, art in one project  
  **学際性** — 数学・物理・電子工学・音楽・芸術が一つのプロジェクトで交差する

---

## Roadmap / ロードマップ

- [x] **Phase 1 (2020)** — C5G prototype: 2,048-bin × 5 modules, 80-voice poly, MIDI-CC control, public demo 2020.11.18
- [ ] **Phase 2 (In Progress)** — DE10-nano port, ARM HPS integration, 10,240-bin scale verification
- [ ] **Phase 3** — GbE UDP protocol, PC abstraction layer, fractal bin generators
- [ ] **Phase 4** — Open Prompt release on Hackaday.io, architecture documentation, STEAM program design, NIME/ICMC paper submission

---

## Target Communities / 対象コミュニティ

- **Hackaday.io** — Primary venue / 主拠点
- **lines (llllllll.co)** — Experimental electronic music (Monome community)
- **Mod Wiggler** — Hardware synth DIY, FPGA subforum
- **NIME / ICMC** — Academic conference paper submission
- **GitHub** — Open Prompt repository (architecture docs, Verilog skeletons)

---

## Proof of Concept: 2020 Waveform Demo / 実証：2020年波形デモ

On 2020.11.18, a test was posted to Twitter (now X): **2,048 sinusoidal oscillators detuned from 996 Hz to 1004 Hz in 1/256 Hz steps**, all at equal amplitude (97% max level). The resulting waveform shows the characteristic Dirichlet-kernel envelope of a phase-coherent oscillator bank — physical proof that the architecture functions as described.

2020年11月18日、Twitterに投稿したテスト：**2,048本のオシレータを996Hzから1004Hzの間で1/256Hz刻みでデチューン**し、同一レベル（最大の97%）で鳴らした録音。波形はDirichletカーネル型の包絡を示し、位相コヒーレントなオシレータバンクの数学的証明となっている。

The 2,048-bin test uses exactly one module at full capacity — confirming that five such modules yield the full 10,240-bin architecture.

2,048ビンのテストはちょうど1モジュールのフル動作確認であり、5基並列で10,240ビンアーキテクチャが実現できることを裏付けている。

---

## License / ライセンス

**Open Prompt** — This project is distributed as architectural knowledge, not as source code.  
The mathematical descriptions, design rationale, and architectural documentation are in the public domain.  
Individual implementations regenerated from this architecture are the sole property of each implementer.

**オープンプロンプト** — 本プロジェクトはソースコードではなく、アーキテクチャ知識として配布されます。  
数理的記述・設計根拠・アーキテクチャ文書はパブリックドメインです。  
本アーキテクチャから再生成された各実装は、それぞれの実装者の単独所有物となります。

---

*Built with FPGA, mathematics, and the conviction that knowledge shared is knowledge multiplied.*  
*FPGAと数学と、「共有された知識は増殖する」という確信のもとに。*
