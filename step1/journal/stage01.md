---
layout: default
title: Stage 1：中斷向量常數 (cpu_vectors.h)
---

# Stage 1 日誌：中斷向量常數 (cpu_vectors.h)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/cpu_vectors.h` ✅
- 定義 `constexpr uint16_t` 常數：`kNmiVector`, `kResetVector`, `kIrqVector` ✅
- 放在 `namespace nes` 裡 ✅

## 最終程式碼
```cpp
#pragma once

#include <cstdint>

namespace nes {

constexpr uint16_t kNmiVector    = 0xFFFA;
constexpr uint16_t kResetVector  = 0xFFFC;
constexpr uint16_t kIrqVector    = 0xFFFE;

} // namespace nes
```

## 討論重點

### 1. 為什麼需要中斷向量
- CPU 開機時不知道程式從哪開始跑，要去 $FFFC-$FFFD 讀 RESET 向量取得程式入口位址
- 三個向量：NMI ($FFFA)、RESET ($FFFC)、IRQ/BRK ($FFFE)
- PRG-ROM 最後 6 bytes 落在 $FFFA-$FFFF 就是中斷向量
- Step1 最終目標：載入 ROM → 把 PRG-ROM 映射到 CPU 位址空間 → 讀三個中斷向量 → 印出來

### 2. nesdev wiki CPU interrupts 頁面重點
- 6502 有三種中斷：IRQ（可遮蔽）、NMI（不可遮蔽）、RESET（啟動）
- NMI 是邊緣觸發（偵測高→低變化），IRQ 是準位觸發（偵測低電位狀態）
- I flag 保護 ISR 不被打斷，CLI/SEI/PLP 有一條指令延遲
- NMI 不能被遮蔽 — I flag 對 NMI 無效
- 中斷劫持：NMI 優先 > IRQ > BRK
- 精確時序細節是 Step2+ 實作 CPU 時才需要

### 3. `constexpr` 編譯期常數
- `constexpr` = 編譯期常數（compile-time constant）
- vs `#define`：無型別無 scope 的文字替換
- vs `const`：可能被編譯器當變數，不一定能在編譯期算
- 編譯期處理的好處：(1) 零執行期成本 (2) 編譯期就能抓錯 (3) 能用在需要常數的地方（陣列大小、switch case）

### 4. `k` 前綴命名慣例
- Google C++ Style Guide：`constexpr`/`const` 常數用 `k` 前綴 + PascalCase
- `k` 一眼看出是常數不是變數
- 不用全大寫 — C++ 有 namespace 和 constexpr 不需要靠大寫避衝突
- 對照專案其他常數：`kMagic`、`Flags6::Mirroring` 同風格

### 5. `uint16_t` 的選擇
- 中斷向量是 16 位元位址（$FFFA 等），用 `uint16_t` 精確對應
- 不用 `int`（浪費空間，且大小不固定）
- 不用 `uint8_t`（位址範圍 $0000-$FFFF 需要 16 位元）

### 6. 原版 vs 新版
- 原版 C：直接用 magic number `0xFFFA` 散落在程式碼中
- 新版 C++：用 `kNmiVector` 等具名常數，語意清楚，改一處就行

### 7. virtual ~RomLoader() = default 補充討論
- 之前 step0 的繼承觀念複習：
- RomLoader 需要 virtual 解構子因為會被繼承（`RomLoader* ptr = new FileRomLoader(...)` 再 `delete ptr`）
- 沒有 virtual 解構子，delete 只呼叫父類解構子，子類被跳過 → 洩漏
- `= default` = 編譯器生成預設實作，RomLoader 沒有需要手動清理的資源
- FileRomLoader 不寫解構子（Rule of Zero），成員 std::string 自己管記憶體

## Review 建議
- 程式碼正確，11 行精簡乾淨
- 命名風格與專案一致（k 前綴 + PascalCase）
- header-only，不需要改 CMakeLists.txt

## 學習心得
Stage 1 雖然只有 11 行，但背後的中斷向量概念很重要。CPU 開機時不知道從哪跑，靠讀 RESET 向量 ($FFFC) 取得入口位址。NMI 和 IRQ 各有不同觸發方式（邊緣 vs 準位），精確時序是之後的事。用 `constexpr` 定義硬體位址常數，語意清楚又零執行期成本，比原版散落的 magic number 好維護。
