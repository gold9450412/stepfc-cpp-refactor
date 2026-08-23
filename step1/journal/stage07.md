---
layout: default
title: Stage 7：CpuBus write() 實作 (cpu_bus.cpp)
---

# Stage 7 日誌：CpuBus write() 實作 (cpu_bus.cpp)

## 日期
2026-07-11

## 狀態
✅ 完成

## 完成事項
- 實作 `write(uint16_t addr, uint8_t data)`，6 個位址區間分派 ✅
- Build 通過，零 warning ✅

## 最終程式碼（cpu_bus.cpp，49行）
```cpp
#include "cpu_bus.h"

namespace nes {

CpuBus::CpuBus(Mapper* mapper)
    : mapper_(mapper) {
}

uint8_t CpuBus::read(uint16_t addr) const {
    if (addr <= 0x1FFF) return ram_[addr & 0x07FF];
    if (addr <= 0x3FFF) return 0;
    if (addr <= 0x401F) return 0;  // APU 暫存器 stub
    if (addr <= 0x5FFF) return 0;  // Expansion ROM stub
    if (addr <= 0x7FFF) return sram_[addr - 0x6000];
    return mapper_->read_prg(addr - 0x8000);
}

void CpuBus::write(uint16_t addr, uint8_t data) {
    if (addr <= 0x1FFF) { ram_[addr & 0x07FF] = data; return; }
    if (addr <= 0x3FFF) { return; }
    if (addr <= 0x401F) { return; }
    if (addr <= 0x5FFF) { return; }
    if (addr <= 0x7FFF) { sram_[addr - 0x6000] = data; return; }
    mapper_->write_prg(addr - 0x8000, data);
}

} // namespace nes
```

## 討論重點

### 1. `void` 方法需要顯式 `return;`
- `read()` 每個 if 有回傳值，自然跳出，不需要寫 `return`
- `write()` 是 `void`，沒有回傳值，寫完一個區間後不跳出會繼續往下跑
- 必須在每個 if 區間後加 `return;` 顯式跳出
- 這是 read（有回傳值）和 write（void）的關鍵差異

### 2. 鏡像用 `&` 不用 `%`
- `addr % 2048` 跟 `addr & 0x07FF` 結果完全一樣（2048 = 2^11）
- 但 `&` 是單一 bit 運算（1 個時脈），`%` 是除法運算（可能 20+ 個時脈）
- CpuBus 的 `read()`/`write()` 是 CPU 每個指令都會呼叫的熱路徑，用 `&` 更好
- `&` 更貼近硬體真實行為：RAM 只有 11 條位址線，A12-A15 沒接被遮掉
- Mapper000 的 `addr % 16384` 也可以改成 `addr & 0x3FFF`（16384 = 2^14）

### 3. CPU memory map 中「ROM」就等於 PRG-ROM
- 在 CPU memory map 上下文裡，不需要加 "PRG" 前綴區分
- CPU 位址空間裡只有一種 ROM
- "PRG-ROM" 這個詞主要出現在 iNES 檔案格式上下文（.nes 裡有 PRG + CHR 兩包資料）
- CPU 永遠碰不到 CHR-ROM，CHR-ROM 在 PPU 位址空間裡
- 兩個獨立位址空間：CPU 64KB（RAM/SRAM/PRG-ROM/暫存器）、PPU 16KB（CHR-ROM/Name Table）

### 4. PPU/APU/Expansion stub 忽略寫入
- 跟 read() 一樣，PPU 和 APU 還沒實作
- write() 的 stub 用 `return;` 忽略寫入（不做任何事）
- 對照原版 C：`assert(!"NOT IMPL")` → release mode 失效
- 忽略寫入更安全：不會崩潰，不依賴 build mode

### 5. `explicit` 只用在建構子和轉換運算子
- 隱式轉換只發生在建構子和轉換運算子（conversion operator）
- 建構子 `Foo(int)` 會被編譯器拿來偷偷轉型
- 一般函式不會創造「型別 A → 型別 B」的轉換路徑
- `explicit operator bool()` 是 C++11 新增，最常見用途是智慧指標和串流

## 遇到的問題

### 問題 1：初版沒加 `return;`
- AI 引導的初版 write() 沒有在每個 if 區間後加 return
- 寫完 RAM 後不跳出，會繼續往下跑到 PRG-ROM，寫兩份資料
- 原因：read() 有回傳值自然跳出，write() 是 void 需要顯式寫 return
- 修正：每個 if 區間後面加 `return;`

### 問題 2：回傳型別寫錯
- 第 28 行回傳型別寫 `uint8_t`，應為 `void`（簽名不符 .h 宣告）
- 修正：改為 `void`

### 問題 3：SRAM 區間寫錯陣列
- 第 43 行 SRAM 區間寫 `ram_`，應為 `sram_`
- `ram_` 只有 2KB，`addr - 0x6000` 最大 8191 會越界
- 修正：改為 `sram_`

## Review 建議
- 程式碼正確，與 read() 對稱
- 每個 if 區間都有 `return;`，不會 fall-through
- `& 0x07FF` 鏡像是熱路徑最佳實踐
- PPU/APU stub 忽略寫入，安全且不依賴 build mode

## 學習心得
Stage 7 的核心是 write() 實作。最大的教訓是 `void` 方法需要顯式 `return;` — read() 有回傳值自然跳出，write() 不寫 return 會 fall-through 到下一個區間。這是 read/write 對稱設計中容易忽略的差異。鏡像用 `&` 不用 `%` 是熱路徑優化，更貼近硬體行為。兩個獨立位址空間（CPU vs PPU）的觀念也在此釐清：CPU 永遠碰不到 CHR-ROM。
