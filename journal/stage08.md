---
layout: default
title: Stage 8：模擬器主體 (famicom.h / famicom.cpp)
---

# Stage 8 日誌：模擬器主體 (famicom.h / famicom.cpp)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/famicom.h` ✅
- 建立 `src/nes/famicom.cpp` ✅
- `class Famicom` 持有 `RomInfo`（直接擁有，非指標）✅
- 建構子接收 `RomInfo`（by move，轉移所有權）✅
- `const RomInfo& get_rom_info() const` — 唯讀存取 ✅
- 解構子無需手動清理 — RAII 自動處理 ✅
- 更新 CMakeLists.txt（加 `famicom.cpp`）✅
- 更新 main.cpp，印出完整 ROM 資訊 ✅
- Build 通過，輸出符合 SPEC 預期 ✅

## 最終程式碼

### famicom.h（16行）
```cpp
#pragma once

#include "rom_info.h"

namespace nes {

class Famicom {
public:
    explicit Famicom(RomInfo rom_info);
    const RomInfo& get_rom_info() const;

private:
    RomInfo rom_info_;
};

} // namespace nes
```

### famicom.cpp（14行）
```cpp
#include "famicom.h"
#include <utility>

namespace nes {

Famicom::Famicom(RomInfo rom_info)
    : rom_info_(std::move(rom_info)) {
}

const RomInfo& Famicom::get_rom_info() const {
    return rom_info_;
}

} // namespace nes
```

### main.cpp（23行）
```cpp
#include <iostream>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"
#include "nes/famicom.h"

int main() {
    nes::RomInfo info;
    std::cout << "Step0 C++ - NES ROM Loader" << std::endl;

    nes::FileRomLoader loader("nestest.nes");
    auto result = loader.load(info);
    if (result == nes::ErrorCode::Ok) {
        nes::Famicom famicom(std::move(info));
        const auto& rom = famicom.get_rom_info();
        std::cout << "ROM: PRG-ROM: " << static_cast<int>(rom.header().prg_rom_count)
                  << " x 16kb    CHR-ROM: " << static_cast<int>(rom.header().chr_rom_count)
                  << " x 8kb    Mapper: " << static_cast<int>(rom.mapper_number()) << std::endl;
        std::cout << "Mirroring: " << (rom.mirroring() ? "Vertical" : "Horizontal")
                  << "    Save RAM: " << (rom.has_save_ram() ? "Yes" : "No")
                  << "    Four Screen: " << (rom.four_screen() ? "Yes" : "No") << std::endl;
    } else {
        std::cout << "Load failed" << std::endl;
    }
    return 0;
}
```

### Build 結果
```
Step0 C++ - NES ROM Loader
ROM: PRG-ROM: 1 x 16kb    CHR-ROM: 1 x 8kb    Mapper: 0
Mirroring: Horizontal    Save RAM: No    Four Screen: No
```
- 對照 SPEC.md 預期輸出全部符合 ✅

## 討論重點

### 1. Famicom 為什麼要包一層 RomInfo
- Famicom 是模擬器主體，RomInfo 只是其中一部分
- 之後 step1+ 會加 CPU、PPU、APU、Memory、Mapper 等成員
- main.cpp 不該直接碰 RomInfo，應透過 Famicom 存取
- 最小責任原則：Famicom 負責模擬器運作，RomInfo 負責存卡帶資料

### 2. 為什麼要 getter 不用 public 成員
- public 成員 = 外面讀寫都行，你管不了
- private + getter = 外面只能讀（回傳 const&），讀法你決定
- 封裝核心：不是「能不能改值」，是「你控制別人怎麼存取」
- 之後改實作（如 lazy loading）只改 getter 一行，外面不用改
- public 成員改實作要改所有呼叫端
- `const` 成員會殺死 assignment（copy/move assignment 被刪除）
- private 擋的是「寫」和「直接摸」，不是擋「讀」
- getter 讓你開放「讀」但保留控制權

### 3. 建構子用 sink parameter idiom（by value + move）
- `explicit Famicom(RomInfo rom_info)` 接 by value
- `rom_info_(std::move(rom_info))` 搬進成員
- 呼叫端 `Famicom famicom(std::move(info))` 把 info 搬進去，零拷貝
- 搬完 `info` 空了，main 裡不能再使用 `info`

### 4. RAII — 不需要解構子
- `RomInfo` 內部用 `std::vector`，vector 自己管記憶體
- `Famicom` 死掉時，成員 `rom_info_` 的解構子自動執行，vector 釋放
- 不需要手動 `free`，不需要寫解構子
- 對照原版 C：`sfc_famicom_t` 有 `init`/`uninit`，手動管理生命週期

### 5. 架構決策 — Famicom 只擁有 RomInfo，不擁有 RomLoader
- RomLoader 是短命物件（讀完即丟），不該被模擬器長期持有
- main.cpp 裡 loader 讀完 ROM 就離開作用域自動釋放
- Famicom 只持有結果（RomInfo），不持有工具（RomLoader）
- 對照原版 C：`sfc_famicom_t` 同時持有 interface + rom_info，職責不清晰

### 6. `static_cast<int>()` 在 cout 中的用途
- `rom.header().prg_rom_count` 是 `uint8_t`，cout 會當 char 印出（亂碼）
- `static_cast<int>()` 轉成 int，cout 印出數字
- C++ 串流對 `uint8_t` 的處理是歷史包袱

## 遇到的問題

### 問題 1：拼字錯誤 `RPG-ROM`
- 初版寫成 `RPG-ROM`（角色扮演遊戲）
- 修正：改為 `PRG-ROM`（PRG = Program）

### 問題 2：成員名稱錯誤 `rpg_rom_count`
- 初版寫成 `rpg_rom_count`
- 修正：改為 `prg_rom_count`

## Review 建議
- 程式碼正確，設計清晰
- 架構決策優秀：Famicom 只擁有 RomInfo 不擁有 RomLoader，職責分離
- sink parameter idiom + std::move 零拷貝組裝
- 輸出完全符合 SPEC 預期
- main.cpp 清楚展示完整流程：load → move into Famicom → 透過 getter 印資訊

## 學習心得
Stage 8 的核心是物件生命週期與所有權。Famicom 直接擁有 RomInfo（非指標），用 `std::move` 零拷貝轉移所有權。RAII 讓我們不需要寫解構子，成員自動釋放。封裝的意義不只是「能不能改」，而是「你控制別人怎麼存取」— private + const getter 讓你保留改實作的彈性。架構決策 Famicom 不擁有 RomLoader 體現了最小責任原則：模擬器只擁有結果，不擁有工具。到這個階段，主程式完整流程已經走通：建立 loader → load ROM → move 進 Famicom → 透過 getter 印出 ROM 資訊。
