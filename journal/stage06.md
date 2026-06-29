---
layout: default
title: Stage 6：檔案讀取 — 讀取 PRG/CHR 資料 (file_rom_loader.cpp 中半)
---

# Stage 6 日誌：檔案讀取 — 讀取 PRG/CHR 資料 (file_rom_loader.cpp 中半)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 根據 header 算出 PRG/CHR 大小（`count * 16KB` / `count * 8KB`）✅
- 跳過 Trainer（若 `flags6 & Flags6::Trainer` 有設定，跳 512 bytes）✅
- `std::vector<uint8_t>` 配置並用 `ifstream::read()` 讀取資料 ✅
- Build 通過，輸出正確 ✅

## 最終程式碼（file_rom_loader.cpp，51行）
```cpp
#include "file_rom_loader.h"
#include <fstream>
#include <vector>
#include <array>
#include "nes_header.h"

namespace nes {

FileRomLoader::FileRomLoader(std::string filepath) 
    : filepath_(std::move(filepath)) {
}

ErrorCode FileRomLoader::load(RomInfo& out) {
    std::ifstream file(filepath_, std::ios::binary);
    if (!file) {
        return ErrorCode::FileNotFound;
    }

    std::array<uint8_t, 16> raw;
    file.read(reinterpret_cast<char*>(raw.data()), 16);
    if (file.gcount() != 16) {
        return ErrorCode::IllegalFile;
    }

    NesHeader header;
    if (!parse_header(raw, header)) {
        return ErrorCode::IllegalFile;
    }

    // 跳過 Trainer（若存在）
    if (header.flags6 & Flags6::Trainer) {
        file.seekg(512, std::ios::cur);
    }

    // 讀取 PRG-ROM
    std::vector<uint8_t> prg_rom(header.prg_rom_count * 16384);
    file.read(reinterpret_cast<char*>(prg_rom.data()), prg_rom.size());
    if (file.gcount() != static_cast<std::streamsize>(prg_rom.size())) {
        return ErrorCode::IllegalFile;
    }

    // 讀取 CHR-ROM
    std::vector<uint8_t> chr_rom(header.chr_rom_count * 8192);
    file.read(reinterpret_cast<char*>(chr_rom.data()), chr_rom.size());
    if (file.gcount() != static_cast<std::streamsize>(chr_rom.size())) {
        return ErrorCode::IllegalFile;
    }

    return ErrorCode::Ok;
}

} // namespace nes
```

## 討論重點

### 1. 跳過 Trainer
- `header.flags6 & Flags6::Trainer` 用位元 AND 檢查 bit 2 是否設定
- `file.seekg(512, std::ios::cur)` 從目前位置往前跳 512 bytes
- Trainer 是可選的 512 byte 區塊，存在於 header 之後、PRG-ROM 之前
- 如果沒有 Trainer（bit 2 = 0），就不跳，直接讀 PRG-ROM

### 2. `std::vector` 建構子直接指定大小
- `std::vector<uint8_t> prg_rom(header.prg_rom_count * 16384)`
- 建構時指定大小，vector 自動配置記憶體並初始化
- 對照原版 C：`malloc(count * 16384)` — vector 更安全，不用手動 free

### 3. `static_cast` vs `reinterpret_cast`

| 轉型 | 用途 | 例子 |
|------|------|------|
| `static_cast` | 數值型別之間轉換 | `size_t` → `streamsize`，`int` → `float` |
| `reinterpret_cast` | 指標/參考重新解讀記憶體 | `uint8_t*` → `char*` |

- `prg_rom.size()` 是數值不是指標，用 `static_cast`
- `reinterpret_cast<streamsize>(prg_rom.size())` 會報錯 — 不支援數值轉換，只認指標
- `gcount()` 回傳 `std::streamsize`（有號），`prg_rom.size()` 回傳 `size_t`（無號）
- 型別不同直接比會觸發 `-Wsign-compare` warning，用 `static_cast` 對齊型別

### 4. 資料流向：Stage 6 只負責讀，Stage 7 組裝
- `prg_rom` 和 `chr_rom` 是 `load()` 的區域變數
- Stage 6 只負責讀到 vector 裡
- Stage 7 會在 `return Ok` 之前組裝進 `out`：
  ```cpp
  out = RomInfo(header, std::move(prg_rom), std::move(chr_rom));
  ```
- `std::move` 把區域變數資料搬進 `out`，不是拷貝
- 搬完區域變數空了，解構不釋放任何東西（零拷貝）

### 5. 邊界情況：CHR-RAM（chr_rom_count = 0）
- 有些卡帶用 CHR-RAM 代替 CHR-ROM，`chr_rom_count = 0`
- `std::vector<uint8_t> chr_rom(0)` → 空 vector，size 0
- `file.read(..., 0)` 讀 0 byte，`gcount()` 回傳 0
- `0 != 0` 不成立，不會誤判為 IllegalFile
- 邏輯安全，不需要特殊處理

## 遇到的問題

### 問題 1：初版錯字 `header.flag6`
- 漏了 `s`，寫成 `header.flag6`
- 修正：改為 `header.flags6`

## Review 建議
- 程式碼正確，錯誤路徑完整
- `static_cast` 和 `reinterpret_cast` 使用正確，各司其職
- 邊界情況（chr_rom_count = 0）自然處理，不需特殊判斷
- 資料流向設計良好：讀取（Stage 6）與組裝（Stage 7）分離

## 學習心得
Stage 6 的核心是讀取二進位資料。`std::vector` 取代了原版 C 的 `malloc`+`fread`，RAII 自動管理記憶體。`static_cast` vs `reinterpret_cast` 的區分是重要觀念：數值轉數值用 static_cast，指標轉指標用 reinterpret_cast。邊界情況的討論讓我理解了 chr_rom_count=0（CHR-RAM）為什麼不需要特殊處理 — vector 和 read 的空操作天然安全。
