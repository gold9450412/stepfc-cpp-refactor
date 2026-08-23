---
layout: default
title: Stage 4：Mapper Factory 函式 (mapper_factory.h / mapper_factory.cpp)
---

# Stage 4 日誌：Mapper Factory 函式 (mapper_factory.h / mapper_factory.cpp)

## 日期
2026-06-29

## 狀態
✅ 完成

## 完成事項
- `error.h` 加 `ErrorCode::MapperNotFound` ✅
- 建立 `src/nes/mapper_factory.h` ✅
- 建立 `src/nes/mapper_factory.cpp` ✅
- `make_mapper` 宣告回傳 `std::unique_ptr<Mapper>` ✅
- 實作 switch：case 0 → `make_unique<Mapper000>(rom)`，其他 → `nullptr` ✅
- 更新 CMakeLists.txt（加 `mapper_factory.cpp`）✅
- Build 通過，零 warning ✅

## 最終程式碼

### error.h 變更
```cpp
enum class ErrorCode {
    Ok = 0,
    FileNotFound,
    IllegalFile,
    OutOfMemory,
    MapperNotFound  // ← 新增
};
```

### mapper_factory.h（12行）
```cpp
#pragma once

#include <memory>

#include "mapper.h"
#include "rom_info.h"

namespace nes {

std::unique_ptr<Mapper> make_mapper(const RomInfo& rom);

} // namespace nes
```

### mapper_factory.cpp（15行）
```cpp
#include "mapper_factory.h"
#include "mapper000.h"

namespace nes {

std::unique_ptr<Mapper> make_mapper(const RomInfo& rom) {
    switch (rom.mapper_number()) {
        case 0:
            return std::make_unique<Mapper000>(rom);
        default:
            return nullptr;    
    }
}

} // namespace nes
```

### Build 結果
- 編譯成功 ✅，零 warning

## 討論重點

### 1. Factory 函式的意義
- 把「選哪個 Mapper」的邏輯集中在 `make_mapper` 裡
- 用的人只要呼叫 `make_mapper(rom)` 就拿到對的 Mapper，不用管內部怎麼分派
- 對照原版 C 的 `sfc_load_mapper`（switch/case + 函數指標賦值）
- 比喻：make_mapper 像工廠的接單台，你告訴它 mapper number，它決定出哪條生產線

### 2. `unique_ptr<Mapper>` 回傳父類指標 — 多型
- 回傳型別是父類 `Mapper`，實際物件是子類 `Mapper000`
- 之後加更多 Mapper（MMC1 mapper 1、MMC3 mapper 4 等），用的人只認 `Mapper` 基類
- `unique_ptr` 表示獨佔擁有，離開作用域自動 `delete`
- Step0 用過同樣模式：`unique_ptr<RomLoader> loader = make_unique<FileRomLoader>(...)`

### 3. 為什麼用 `unique_ptr` 不直接回傳物件
- Mapper 是抽象類別（有 `= 0`），不能直接建立物件，必須用指標
- C 風格 `new` + 手動 `delete` 的問題：忘了=洩漏、delete 兩次=崩潰、exception 跳過 delete=洩漏
- C++ `make_unique` + 離開作用域自動釋放：不用記、不會錯、exception 也不洩漏
- 日常比喻：沒有 unique_ptr = 租倉庫要自己記得退租；有 unique_ptr = 離開自動退租

### 4. `default: return nullptr` — 未知 Mapper 的處理
- 目前只有 case 0（Mapper000），其他 mapper number 回傳 `nullptr`
- 之後加新 Mapper 只要在 switch 裡加 case，不用改呼叫端
- 呼叫端收到 `nullptr` 可以回傳 `ErrorCode::MapperNotFound`

### 5. 函式不在類別裡 — 自由函式
- `make_mapper` 是自由函式（free function），不在任何 class 裡
- 不需要物件就能呼叫：`auto mapper = make_mapper(rom);`
- Factory 不需要狀態（不存資料），用自由函式最簡單
- 如果 Factory 需要狀態（如快取、註冊表），才用 class

## 遇到的問題

### 問題 1：初版 include 少了結尾引號
- `#include "mapper000.h` → 編譯錯誤
- 修正：`#include "mapper000.h"`

## Review 建議
- 程式碼正確，結構清晰
- Factory pattern 實作正確：switch + make_unique + nullptr fallback
- 自由函式選擇正確：Factory 不需要狀態
- `MapperNotFound` 錯誤碼已加，之後 Famicom 可以檢查 `nullptr` 回傳錯誤

## 學習心得
Stage 4 的核心是 Factory pattern。`make_mapper` 把「選哪個 Mapper」的邏輯集中起來，用的人不用管內部 switch。回傳 `unique_ptr<Mapper>`（父類指標）展現多型：不管將來加多少 Mapper，呼叫端只認基類。`unique_ptr` 取代手動 `new`/`delete`，RAII 自動管理生命週期。抽象類別不能直接回傳物件，必須用指標 — 這也是為什麼 Factory 回傳 `unique_ptr` 而不是 `Mapper`。
