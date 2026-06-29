---
layout: default
title: Stage 3：ROM 資訊類別 (rom_info.h / rom_info.cpp)
---

# Stage 3 日誌：ROM 資訊類別 (rom_info.h / rom_info.cpp)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 定義 `class RomInfo`，私有成員 + const getter ✅
- 持有 `NesHeader header_` — 解析後的檔頭 ✅
- 持有 `std::vector<uint8_t> prg_rom_`, `chr_rom_` — ROM 資料本體 ✅
- `bool is_loaded() const` — 資料是否已載入 ✅
- 只提供 getter，不提供 setter（唯讀介面）✅
- 預設建構子 `RomInfo() = default` ✅
- 帶資料的建構子（sink parameter idiom）✅
- 更新 CMakeLists.txt（加 .cpp 檔、target_include_directories）✅
- 更新 main.cpp 測試編譯通過 ✅

## 最終程式碼

### rom_info.h
```cpp
#pragma once

#include <cstdint>
#include <vector>

#include "nes_header.h"

namespace nes {

class RomInfo {
public:
    RomInfo() = default;

    RomInfo(const NesHeader& header,
           std::vector<uint8_t> prg,
           std::vector<uint8_t> chr);

    // getter: 唯讀存取
    bool is_loaded() const;
    const NesHeader& header() const;
    const std::vector<uint8_t>& prg_rom() const;
    const std::vector<uint8_t>& chr_rom() const;

private:
    bool loaded_ = false;
    NesHeader header_{};
    std::vector<uint8_t> prg_rom_;
    std::vector<uint8_t> chr_rom_;
};

} // namespace nes
```

### rom_info.cpp
```cpp
#include "rom_info.h"
#include <utility>

namespace nes {

RomInfo::RomInfo(const NesHeader& header,
                 std::vector<uint8_t> prg,
                 std::vector<uint8_t> chr)
    : loaded_(true)
    , header_(header)
    , prg_rom_(std::move(prg))
    , chr_rom_(std::move(chr)) {
    
}

// getter
bool RomInfo::is_loaded() const {
    return loaded_;
}

const NesHeader& RomInfo::header() const {
    return header_;
}

const std::vector<uint8_t>& RomInfo::prg_rom() const {
    return prg_rom_;
}

const std::vector<uint8_t>& RomInfo::chr_rom() const {
    return chr_rom_;
}

} // namespace nes
```

### CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.14)

project(step0_cpp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(step0_cpp 
    src/main.cpp
    src/nes/nes_header.cpp
    src/nes/rom_info.cpp)

target_compile_options(step0_cpp PRIVATE -Wall -Wextra)
target_include_directories(step0_cpp PRIVATE src)
```

### main.cpp
```cpp
#include <iostream>
#include "nes/rom_info.h"

int main() {
    nes::RomInfo info;
    std::cout << "Step0 C++ - NES ROM Loader" << std::endl;
    std::cout << "Loaded: " << info.is_loaded() << std::endl;
    return 0;
}
```

## 討論重點

### 1. 宣告 vs 定義 (.h vs .cpp)
- .h = 宣告（declaration）：告訴編譯器函數長什麼樣，不寫內容
- .cpp = 定義（definition）：真正寫函數做什麼
- 比喻：.h 是目錄，.cpp 是內頁
- 為什麼分開：寫在 .h 被多重 include 會重複定義錯誤；.h 宣告可重複，.cpp 定義只能一份
- 不能在兩個 .cpp 都實作同一函數 → multiple definition 連結錯誤

### 2. #pragma once
- 問題：a.h 和 b.h 都 include nes_header.h，main.cpp include 兩者 → nes_header.h 被引入兩次 → struct 重複定義
- #pragma once：編譯器記住檔案，第二次自動跳過
- 對照 C 的 #ifndef guard（3行），#pragma once 更簡潔（1行）
- 非標準但 g++/clang++/MSVC 全支援，業界 C++ 幾乎都用

### 3. std::vector<uint8_t> 存資料不是存地址
- 原版 C：`uint8_t* data_prgrom` = 指標，指向 malloc 出來的記憶體
- 新版：`std::vector<uint8_t> prg_rom_` = 直接擁有整包資料，vector 自己管記憶體（RAII）
- vector 裝的是很多個 uint8_t 排在一起（比喻：停車場，每個元素 1 byte，16384 個 = 16KB）
- vector 怎麼知道要多少空間：建構時指定大小、之後 resize 或 push_back。我們 Stage 6 讀檔時用 resize

### 4. PRG-ROM vs CHR-ROM
- PRG = Program ROM，CPU 執行的機器碼（遊戲邏輯），16KB 單位
- CHR = Character ROM，PPU 讀的圖案 tile（8x8 像素），8KB 單位
- 比喻：PRG 是大腦（邏輯），CHR 是皮膚（畫面）
- 兩者獨立，CPU 只碰 PRG，PPU 只碰 CHR

### 5. NesHeader header_{} 的大括號
- `{}` = 全部清零（value-initialization）
- 沒有 `{}` 可能是亂數（記憶體殘留垃圾值）
- C++11 寫法，可統一用在所有型別

### 6. sink parameter idiom（by value + move）
- `const&` 永遠拷貝，沒得選
- by value：呼叫端可以 move（不要了就搬過去，零拷貝），也可以 copy（還需要就留著）
- 建構子裡再 std::move 進成員
- 比 const& 更有彈性

### 7. C 指標 vs C++ 參考
- C 只有指標 `*`：`void foo(int* p) { *p = 42; }` 呼叫 `foo(&x)`
- C++ 多了參考 `&`：`void foo(int& r) { r = 42; }` 呼叫 `foo(x)`
- 參考不能是空的（指標可以 NULL），參考不用 * 直接用
- C++ 完全支援 C 指標寫法，但業界優先用參考（更安全）
- 指標用於：可空（nullptr）、指標運算、跟 C API 互通

### 8. 雙 const 解釋
```cpp
const std::vector<uint8_t>& prg_rom() const;
```
- 第一個 const（左邊）：回傳的東西不能被改（外面不能 push_back）
- 第二個 const（右邊）：這個方法不會改物件內部（保證沒副作用）
- 兩個是不同層面的保證，都要寫
- `&`：不拷貝，直接看原份

### 9. prg_rom() 是方法不是成員變數
- `prg_rom_`（有底線）= 成員變數（private）
- `prg_rom()`（有括號）= 方法（public getter）
- 名字可一樣，編譯器靠 `()` 區分

### 10. const 方法裡改區域變數可以
- const 只管成員變數，不管區域變數
- `int a = 0; a++;` OK，`loaded_ = true;` 編譯錯誤

### 11. 為什麼需要預設建構子 RomInfo() = default
- C++ 規則：寫了任何建構子，編譯器不自動生成預設建構子
- 沒有 `= default` 就不能寫 `RomInfo info;`（空的）
- SPEC Stage 4 的 `load(RomInfo& out)` 需要先有空 RomInfo
- load 內部用帶參數建構子建臨時的，再 move assign 給 out

### 12. loaded_ 的用途
- 預設建構子：loaded_ = false（成員初始式）
- 帶參數建構子：loaded_ = true（初始化列表覆蓋）
- is_loaded() 回傳 loaded_，讓外面判斷有沒有載入
- 顯式旗標優於用 `!prg_rom_.empty()` 推斷（顯式優於隱式）

### 13. is_loaded() 回傳 bool 不用 &，其他 getter 回傳 const&
- bool 是基本型別（1 byte），拷貝跟傳參考一樣快，直接回傳值更簡單
- vector/NesHeader 很大（幾十 KB），回傳 & 避免拷貝
- 規則：基本型別（bool/int/char 等）回傳值，大物件回傳 const&

### 14. ++i vs i++
- `i++`（後置）：先用舊值再加 1
- `++i`（前置）：先加 1 再用新值
- for 迴圈裡對 int 結果一樣
- 業界偏好 `++i`：對複雜型別（iterator）後置多一次拷貝，前置不拷貝

### 15. target_include_directories 的 PRIVATE/PUBLIC/INTERFACE
- `PRIVATE`：只編譯自己時找得到標頭檔（下游看不到）
- `PUBLIC`：自己用，下游也看得到
- `INTERFACE`：自己不用，只給下游用（header-only 函式庫）
- 我們是執行檔（沒有下游），用 PRIVATE 就夠了
- 「下游」= 依賴你的 target（透過 target_link_libraries）
- 只有拆成多個 target（函式庫 + 執行檔 + 測試）時才需要區分

### 16. add_executable 為什麼要加 .cpp
- CMake 需要知道哪些 .cpp 要編譯
- .h 不用列出（靠 #include 引入），只有 .cpp 要列（每個 .cpp 是獨立編譯單元）
- 不列出不會被編譯 → 連結時找不到函式定義 → undefined reference

### 17. 為什麼要拆函式庫（CMake target）
- 小專案不需要拆，全部編進一個執行檔就好
- 拆的好處：(1) 測試 target 共用同一份 .cpp 不重複列 (2) 編譯速度（只重編必要的）(3) 重複使用 (4) 封裝
- 我們 Stage 10 加測試時才會遇到
- 只 build 特定 target：`cmake --build build --target test_step0`
- CMake 自動追蹤依賴，只重編必要的部分

## 遇到的問題

### 問題 1：main.cpp include 錯檔案
- 初版寫 `#include "nes/nes_header.h"`，但 `RomInfo` 定義在 `rom_info.h` 不是 `nes_header.h`
- 編譯錯誤：`'RomInfo' is not a member of 'nes'`
- 修正：改成 `#include "nes/rom_info.h"`

## Review 建議
- 程式碼正確，符合業界 C++ 慣例
- 封裝、const-correctness、sink parameter idiom 都應用了
- Rule of Zero：不宣告 copy/move/destruct，讓編譯器自動生成正確版本

## 學習心得
Stage 3 的核心是「封裝」和「擁有權」。原版 C 的 `sfc_rom_info_t` 用 raw pointer + malloc，使用者可以亂改內部資料，忘記 free 就漏記憶體。新版 `RomInfo` 用 `std::vector` 直接擁有資料（RAII），私有成員 + const getter 保護內部狀態，物件建構好就不能被改（immutable after construction）。sink parameter idiom（by value + move）讓呼叫端有選擇：要保留就 copy，不要了就 move，比傳統 const& 更有彈性。`bool loaded_` 旗標顯式表達狀態，比用 `!prg_rom_.empty()` 推斷更清楚。CMakeLists.txt 的 `target_include_directories` 和 `add_executable` 多檔編譯是邁向多檔專案的基礎。
