# Stage 9 日誌：主程式 (main.cpp)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 用 `std::make_unique<FileRomLoader>` 建立 loader（展示 `unique_ptr` 多型）✅
- 呼叫 `loader->load(info)` 載入 ROM ✅
- 用 `std::move(info)` 將 RomInfo 移入 `Famicom` ✅
- loader 離開作用域自動釋放（RAII）✅
- 印出 ROM 資訊，確認輸出正確 ✅

## 最終程式碼（main.cpp，24行）
```cpp
#include <iostream>
#include <memory>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"
#include "nes/famicom.h"

int main() {
    nes::RomInfo info;
    std::cout << "Step0 C++ - NES ROM Loader" << std::endl;

    auto loader = std::make_unique<nes::FileRomLoader>("nestest.nes");
    auto result = loader->load(info);

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

## 討論重點

### 1. `std::unique_ptr` — 獨佔所有權的智慧指標
- `auto loader = std::make_unique<FileRomLoader>(...)` 建立 unique_ptr
- unique_ptr = 只有一個人能擁有這個物件，不能複製，只能 move
- 離開作用域自動 `delete`，不用手動釋放（RAII）
- 對照原版 C：`malloc` + 手動 `free`，容易忘記 free 或 free 兩次

### 2. 為什麼用 `std::make_unique` 而不是 `new`
- `std::make_unique<T>(args...)` 比 `std::unique_ptr<T>(new T(args...))` 更安全簡潔
- make_unique 是 C++14 引入，一次配置物件 + control block（雖然 unique_ptr 沒有 control block，但慣例一致）
- 不直接寫 `new`：讓記憶體管理集中在智慧指標裡

### 3. `unique_ptr` 展示多型
- `auto loader = std::make_unique<nes::FileRomLoader>(...)`
- loader 的型別是 `unique_ptr<FileRomLoader>`，指向 FileRomLoader
- 之後可以改成 `unique_ptr<RomLoader>` 指向不同子類，展示多型
- `loader->load(info)` 用 `->` 因為 loader 是指標（智慧指標）

### 4. `.` vs `->`
- 之前 Stage 8：`loader.load(info)` — loader 是物件，用 `.`
- Stage 9：`loader->load(info)` — loader 是 unique_ptr（指標），用 `->`
- `->` 是 `(*loader).load()` 的語法糖

### 5. 完整流程回顧
```
main()
  ├── RomInfo info;                          // 空 RomInfo
  ├── make_unique<FileRomLoader>("...")      // 建 loader（unique_ptr）
  ├── loader->load(info)                     // 讀 ROM 進 info
  ├── Famicom famicom(std::move(info))       // move info 進 Famicom
  ├── famicom.get_rom_info()                 // 透過 getter 取資料
  ├── cout << ROM 資訊                       // 印出
  └── return 0                               // loader/famicom 自動釋放（RAII）
```

### 6. 對照原版 C 的 main.c
| 原版 C | 本版 C++ | 差異 |
|--------|---------|------|
| `malloc` + `fopen` | `make_unique` + `ifstream` | RAII 自動釋放 |
| 函數指標 `load_rom` | 虛擬函數 `load()` | 型別安全多型 |
| 手動 `free_rom` | 不需要 | vector RAII |
| `init`/`uninit` | 建構子/解構子 | 物件生命週期自動管理 |

### 7. `= 0` 純虛擬函數
- `virtual ErrorCode load(RomInfo& out) = 0;` 的 `= 0` 是「純虛擬函數」
- `= 0` 不是數學的零，是 C++ 特殊語法，意思是「不寫實作，強制子類覆寫」
- 有 `= 0` 的類別叫抽象類別，不能建立物件
- 對比三種寫法：
  - `virtual void foo();` — 只宣告，實作寫在 .cpp，呼叫時才連結
  - `virtual void foo() {}` — 基類提供預設實作，子類可覆寫可不覆寫
  - `virtual void foo() = 0;` — 不寫實作，強制子類覆寫，基類不能建立物件
- RomLoader 用 `= 0` 因為父類不知道怎麼讀檔，交給子類決定

### 8. 完整多型 vs 目前寫法的差異
- 目前寫法：`auto loader = std::make_unique<FileRomLoader>(...)`
  - `auto` 推導出 `unique_ptr<FileRomLoader>`（具體型別）
- 完整多型：`std::unique_ptr<RomLoader> loader = std::make_unique<FileRomLoader>(...)`
  - loader 型別是父類 `unique_ptr<RomLoader>`，指向子類物件
- 功能上現在一樣，差異在「替換」和「擴充」時：

```cpp
// 完整多型版 — 主邏輯不用改
std::unique_ptr<nes::RomLoader> loader;
if (from_file) {
    loader = std::make_unique<nes::FileRomLoader>("nestest.nes");
} else {
    loader = std::make_unique<nes::MemoryRomLoader>(data);
}
loader->load(info);  // 不管哪種都這樣呼叫

// 目前寫法 — 要改兩段，loader->load 無法搬到 if 外
if (from_file) {
    auto loader = std::make_unique<nes::FileRomLoader>(...);
    loader->load(info);  // 只能在 if 裡呼叫
} else {
    auto loader = std::make_unique<nes::MemoryRomLoader>(...);
    loader->load(info);  // 只能在 if 裡呼叫
}
```

### 9. 為什麼目前寫法不能把 `loader->load` 搬到 if 外
- 兩個問題：
  1. **作用域**：`auto loader` 宣告在 `{}` 裡，離開 `{}` 就銷毀，外面看不到
  2. **型別不同**：`auto` 推導出 `unique_ptr<FileRomLoader>` 和 `unique_ptr<MemoryRomLoader>`，兩個型別不同無法統一
- 完整多型版能解決：一開始宣告 `unique_ptr<RomLoader> loader`，型別確定，所有子類都能塞進去，`loader->load(info)` 可以搬到 if 外
- 根本原因：`auto` 推導出具體型別，具體型別之間不能互換；父類型別可以裝任何子類

## 遇到的問題

### 問題 1：拼字錯誤 `std::make_uniqure`
- 多了一個 `r`，寫成 `make_uniqure`
- 修正：改為 `std::make_unique`

## Review 建議
- 程式碼正確，完整展示 unique_ptr 多型
- 整個流程清晰：load → move → Famicom → getter → 印資訊
- RAII 貫穿全程：loader 和 famicom 離開作用域自動釋放
- 到此為止，Stage 0-9 的主程式邏輯全部完成

## 學習心得
Stage 9 的核心是用 `unique_ptr` 組裝所有元件。`make_unique` 取代了原版 C 的 `malloc`，`->` 取代了 `.`（因為是指標）。unique_ptr 展示了 C++ 多型的正確用法：父類指標指向子類物件，透過虛擬函數動態分派。到這個階段，整個 Step0 的主程式流程已經完整：建立 loader → load ROM → move 進 Famicom → 透過 getter 印出 ROM 資訊。接下來 Stage 10-12 是測試，確保程式碼正確性。
