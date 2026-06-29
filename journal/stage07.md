---
layout: default
title: Stage 7：解析 Flags 填入 ROM 資訊 (file_rom_loader.cpp 後半)
---

# Stage 7 日誌：解析 Flags 填入 ROM 資訊 (file_rom_loader.cpp 後半)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 組裝 RomInfo：`out = RomInfo(header, std::move(prg_rom), std::move(chr_rom))` ✅
- 完成 `load()` 回傳 `ErrorCode::Ok` ✅
- 錯誤路徑回傳對應 `ErrorCode`（FileNotFound、IllegalFile）✅
- RomInfo 新增 4 個 getter：`mapper_number()`、`mirroring()`、`has_save_ram()`、`four_screen()` ✅
- mapper number = `(flags6 >> 4) | (flags7 & 0xF0)` ✅
- Build 通過，`unused parameter` warning 消失 ✅

## 最終程式碼

### file_rom_loader.cpp（53行）— 新增組裝行
```cpp
#include "file_rom_loader.h"
#include <fstream>
#include <vector>
#include <array>
#include <utility>
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

    if (header.flags6 & Flags6::Trainer) {
        file.seekg(512, std::ios::cur);
    }

    std::vector<uint8_t> prg_rom(header.prg_rom_count * 16384);
    file.read(reinterpret_cast<char*>(prg_rom.data()), prg_rom.size());
    if (file.gcount() != static_cast<std::streamsize>(prg_rom.size())) {
        return ErrorCode::IllegalFile;
    }

    std::vector<uint8_t> chr_rom(header.chr_rom_count * 8192);
    file.read(reinterpret_cast<char*>(chr_rom.data()), chr_rom.size());
    if (file.gcount() != static_cast<std::streamsize>(chr_rom.size())) {
        return ErrorCode::IllegalFile;
    }

    out = RomInfo(header, std::move(prg_rom), std::move(chr_rom));
    return ErrorCode::Ok;
}

} // namespace nes
```

### rom_info.h（35行）— 新增 4 個 getter 宣告
```cpp
uint8_t mapper_number() const;
bool mirroring() const;
bool has_save_ram() const;
bool four_screen() const;
```

### rom_info.cpp（48行）— 新增 4 個 getter 實作
```cpp
uint8_t RomInfo::mapper_number() const {
    return (header_.flags6 >> 4) | (header_.flags7 & 0xF0);
}

bool RomInfo::mirroring() const {
    return (header_.flags6 & Flags6::Mirroring);
}

bool RomInfo::has_save_ram() const {
    return (header_.flags6 & Flags6::SaveRam);
}

bool RomInfo::four_screen() const {
    return (header_.flags6 & Flags6::FourScreen);
}
```

## 討論重點

### 1. 設計決策 — getter 即時計算，不多存成員變數
- 討論兩種做法：A) 在 load() 裡算好存進 RomInfo B) RomInfo 提供 getter 即時計算
- 選擇 B（資深做法）：RomInfo 提供 getter，從 `header_` 即時計算
- 理由：
  1. 封裝 — 位元運算邏輯只寫一次，不散落各處
  2. 位元運算極快（1~2 個時脈），不需快取
  3. load() 不用改，職責分離
  4. Tell Don't Ask — RomInfo 自己知道怎麼算，不用外面算好塞進來

### 2. 為什麼這 4 個 getter，不包含 trainer
- `mapper_number()`：最常被查，決定用哪個 Mapper
- `mirroring()`：PPU 每次渲染要問
- `has_save_ram()`：記憶體管理要問
- `four_screen()`：PPU 命名表配置要問
- trainer：只在讀檔時跳過用一次（Stage 6 的 `seekg(512)`），執行期不需要，不提供 getter

### 3. mapper_number 的位元運算
- `flags6 >> 4`：取 flags6 高 4 位（mapper 低 4 位），移到低位
- `flags7 & 0xF0`：取 flags7 高 4 位（mapper 高 4 位），留在高位
- `|` 合併：低 4 位 + 高 4 位 = 8 bit mapper number
- 對照原版 C：`info->mapper_number = (control1 >> 4) | (control2 & 0xF0)`

### 4. `std::move` 組裝 RomInfo
- `out = RomInfo(header, std::move(prg_rom), std::move(chr_rom))`
- `std::move` 把區域變數資料搬進 out，零拷貝
- 搬完區域變數空了，解構不釋放任何東西
- `#include <utility>` 需要 for `std::move`

### 5. 為什麼 `mapper_number()` 回傳 `uint8_t` 不用 `const uint8_t`
- by value 回傳的是拷貝（副本），外部改副本不影響內部
- `const uint8_t` 等於管別人的副本，毫無意義
- 規則：by value 不用前面 const，by reference（`&`）才要

### 6. `const NesHeader& header() const` 雙 const 缺一不可
- 第一個 const（左）：保護回傳的東西不能被改
- 第二個 const（右）：保護方法不改物件
- `NesHeader& header() const` → 編譯錯誤：const 方法不能回傳非 const 參考
- `NesHeader& header()`（沒後 const）→ 能編譯但封裝破洞，外部能改內部
- `const NesHeader& header() const` → 業界標準 getter

### 7. 不用改建構子
- 4 個新 getter 都是即時計算，從 `header_` 現算
- 建構子已經把 `header_` 存好了，不需要額外初始化
- 不多存成員變數 = 不用改建構子

### 8. Trainer 如果要存怎麼辦
- 目前只跳過不存（`file.seekg(512)`）
- 如果要存：rom_info.h 加 `trainer_` 成員 + getter，建構子多接參數，load() 改讀取
- 但不這樣做：(1) 原版也沒存 (2) Trainer 極罕見 (3) YAGNI (4) 之後真要用再加
- 什麼時候需要：真正執行模擬時（step1+），CPU 啟動前寫進 $7000-$71FF

## 遇到的問題
- 無重大問題，Stage 6 的基礎打得好，Stage 7 只需加一行組裝 + 4 個 getter

## Review 建議
- 程式碼正確，設計決策優秀
- getter 即時計算是資深做法，封裝好、職責分離
- `load()` 完整錯誤路徑：FileNotFound → IllegalFile（gcount 不足）→ IllegalFile（magic 不符）→ Ok
- `std::move` 零拷貝組裝是現代 C++ 最佳實踐
- `unused parameter 'out'` warning 消失，因為 out 現在有被使用

## 學習心得
Stage 7 的核心是完成完整的 `load()` 流程。設計決策「getter 即時計算」是重要學習：不要多存成員變數，位元運算極快不需要快取，封裝邏輯只寫一次。`std::move` 零拷貝組裝展現了現代 C++ 的效能優勢 — 區域變數的資料直接搬進 out，不複製幾十 KB 的 ROM 資料。雙 const 的意義（保護回傳值 + 保護物件）也在這階段徹底理解。到目前為止，FileRomLoader 的 `load()` 已經完整：開檔 → 讀檔頭 → 驗證 magic → 跳 Trainer → 讀 PRG → 讀 CHR → 組裝 RomInfo → 回傳 Ok。
