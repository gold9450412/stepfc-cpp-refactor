---
layout: default
title: Stage 6：CpuBus read() 實作 (cpu_bus.cpp)
---

# Stage 6 日誌：CpuBus read() 實作 (cpu_bus.cpp)

## 日期
2026-07-11

## 狀態
✅ 完成

## 完成事項
- 實作 `read(uint16_t addr) const`，6 個位址區間分派 ✅
- Build 通過 ✅

## 最終程式碼（cpu_bus.cpp，27行）
```cpp
#include "cpu_bus.h"

namespace nes {

CpuBus::CpuBus(Mapper* mapper)
    : mapper_(mapper) {
}

uint8_t CpuBus::read(uint16_t addr) const {
    if (addr <= 0x1FFF) {
        return ram_[addr & 0x07FF];
    }
    if (addr <= 0x3FFF) {
        return 0;
    }
    if (addr <= 0x401F) {
        return 0;  // APU 暫存器 stub
    }
    if (addr <= 0x5FFF) {
        return 0;  // Expansion ROM stub
    }
    if (addr <= 0x7FFF) {
        return sram_[addr - 0x6000];
    }
    return mapper_->read_prg(addr - 0x8000);
}

} // namespace nes
```

## 討論重點

### 1. `addr & 0x07FF` — RAM 鏡像原理
- RAM 只有 2KB（$0800），但 CPU 配給它 8KB 位址空間（$0000-$1FFF）
- 2KB 要填滿 8KB，重複 4 次
- `& 0x07FF` 遮掉高位元只保留低 11 bit（0-2047）
- 硬體原因：CPU 16 條位址線，RAM 只有 11 條，A12-A11 沒接，自然鏡像
- 不是故意設計鏡像，是省掉解碼電路的成本
- 真實 NES 程式正常只用 $0000-$07FF，不碰鏡像區
- 模擬器要處理鏡像：忠實還原硬體行為

### 2. `addr - 0x6000` — SRAM 索引平移
- SRAM 在 $6000-$7FFF，`sram_` 陣列從 0 開始
- `addr - 0x6000` 把位址平移到陣列索引（$6000 → 0, $7FFF → 8191）
- Wiki 只列位址範圍不教實作，「減 $6000」是模擬器作者的設計

### 3. `addr - 0x8000` — PRG-ROM 偏移量
- PRG-ROM 在 $8000-$FFFF，Mapper 的 `read_prg` 接收的是偏移量（0-32767）
- `addr - 0x8000` 把 CPU 位址轉成 Mapper 內部偏移量
- Mapper 內部再決定怎麼映射（16KB 鏡像或 32KB 直接）

### 4. if 鏈不用 else
- 每個 `if` 裡都有 `return`，不會 fall-through 到下一個
- 不需要寫 `else if`，省一層縮排
- 業界 C++ 慣例：early return 減少嵌套

### 5. PPU/APU/Expansion stub 回傳 0
- Step1 只做 CPU 記憶體空間，PPU 和 APU 還沒實作
- stub 回傳 0 是「安全值」，不會崩潰也不會誤導
- 對照原版 C：`assert(!"NOT IMPL")` → release mode 失效（assert 在 NDEBUG 下消失）
- 回傳 0 更安全：永遠有效，不依賴 build mode

### 6. `sram_` 為什麼是 8KB
- nesdev wiki CPU memory map：$6000-$7FFF = $2000 (8KB) = SRAM/PRG-RAM
- $2000 = 8192 bytes = 8KB
- 這 8KB 是 CPU 位址空間配給 SRAM 的窗口大小，不是所有卡帶都有 8KB SRAM
- 只有 `flags6 & SaveRam` 有設定時才有電池備份 SRAM

### 7. `explicit` 只用在建構子和轉換運算子
- explicit 防的是「隱式轉換」，隱式轉換只發生在建構子和轉換運算子
- 不能用在一般函式
- `explicit operator bool()` 是 C++11 新增，最常見用途是智慧指標和串流

## Review 建議
- 程式碼正確，位址解碼邏輯清晰
- if 鏈 early return 風格乾淨
- PPU/APU stub 回傳 0 比 assert 更安全
- `& 0x07FF` 鏡像和 `- 0x6000` 平移都是業界標準做法

## 學習心得
Stage 6 的核心是位址解碼。CPU 64KB 位址空間切成 6 個區間，每個區間有不同的處理方式。RAM 鏡像用 `& 0x07FF` 是硬體設計的自然結果 — CPU 位址線比 RAM 多，沒接的線被忽略。SRAM 用 `- 0x6000` 平移到陣列索引。PRG-ROM 透過 Mapper 讀取，`- 0x8000` 轉成偏移量。PPU/APU stub 回傳 0 比 assert 更安全，不依賴 build mode。
