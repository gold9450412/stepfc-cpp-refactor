---
layout: default
title: Stage 2：抽象 Mapper 類別 (mapper.h)
---

# Stage 2 日誌：抽象 Mapper 類別 (mapper.h)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/mapper.h` ✅
- 定義 `class Mapper`，抽象類別 ✅
- 純虛擬函數 `virtual uint8_t read_prg(uint16_t addr) const = 0;` ✅
- 純虛擬函數 `virtual void write_prg(uint16_t addr, uint8_t data) = 0;` ✅
- 虛擬解構子 `virtual ~Mapper() = default;` ✅

## 最終程式碼
```cpp
#pragma once

#include <cstdint>

namespace nes {

class Mapper {
public:
    virtual ~Mapper() = default;
    virtual uint8_t read_prg(uint16_t addr) const = 0;
    virtual void write_prg(uint16_t addr, uint8_t data) = 0;
};

} // namespace nes
```

## 討論重點

### 1. Mapper 介面設計
- Mapper 負責 PRG-ROM 的位址映射（bank switching）
- `read_prg(addr)` — CPU 讀 PRG-ROM 時呼叫，mapper 決定實際讀哪個 byte
- `write_prg(addr, data)` — CPU 寫 PRG-ROM 時呼叫（ROM 通常忽略，但有些 mapper 有 PRG-RAM）
- 之後 StepFC 會加更多 mapper：MMC1、MMC3 等，都用繼承 + virtual

### 2. read_prg 是 const，write_prg 不是 const
- `read_prg` 是 `const` 方法 — 讀不改變物件狀態
- `write_prg` 不是 `const` — 寫可能改變 mapper 內部狀態（如 bank register）
- 即使 Mapper000（NROM）的 write_prg 是空實作，介面仍要允許寫

### 3. 與 step0 RomLoader 的比較
| | RomLoader | Mapper |
|---|---|---|
| 純虛擬函數 | `load()` | `read_prg()` + `write_prg()` |
| 虛擬解構子 | `= default` | `= default` |
| 子類 | FileRomLoader | Mapper000, MMC1, MMC3... |
| 用途 | 讀檔（短命） | 位址映射（長期） |

### 4. 原版 C vs 新版 C++
- 原版 `sfc_mapper_t` 用函數指標 struct 達到多型
- 新版用 `virtual` 函數，編譯器自動建 vtable，型別安全
- 加新 mapper 只需新增子類 + factory 裡加一個 case，不用改主程式

### 5. class 定義結尾要分號
- `class Mapper { ... };` — 結尾要有 `;`
- 初版漏了分號，修正後正確
- `namespace { }` 結尾不需要分號（跟 class 不同）

## 遇到的問題

### 問題 1：class 定義結尾漏分號
- 初版第 12 行寫 `}` 沒有 `;`
- 修正：改為 `};`

## Review 建議
- 程式碼正確，14 行精簡
- 介面設計良好：read const / write 非 const，職責清晰
- header-only，不需要改 CMakeLists.txt
- 與 RomLoader 結構一致，複習了抽象類別的寫法

## 學習心得
Stage 2 跟 step0 的 RomLoader 幾乎一樣的模式 — 抽象類別 + 純虛擬 + 虛擬解構子。差別在 Mapper 有兩個方法（read + write）且 read 是 const。Mapper 是 NES 模擬器的核心擴展點：不同卡帶用不同 mapper，透過 virtual 達到多型，主程式不用改。原版 C 用函數指標 struct 達到同樣目的，C++ 的 virtual 更安全更優雅。
