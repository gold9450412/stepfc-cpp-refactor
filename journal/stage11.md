# Stage 11 日誌：ROM 資訊測試 (test_rom_info.cpp)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `tests/test_rom_info.cpp` ✅
- TEST 1：測試初始狀態（`is_loaded() == false`）✅
- TEST 2：測試 getter 回傳正確值（建構後填入資料再驗證）✅
- TEST 3：測試 const 介面（const reference 上只能呼叫 const 方法）✅
- 至少 3 個 TEST case ✅
- 更新 `tests/CMakeLists.txt`（新增 test_rom_info target）✅
- Build + ctest 全部通過（4 tests, 100%）✅

## 最終程式碼

### test_rom_info.cpp（56行）
```cpp
#include <gtest/gtest.h>
#include "nes/rom_info.h"
#include <vector>
#include <cstdint>
#include <utility>

TEST(RomInfoTest, DefaultConstructorIsNotLoaded) {
    nes::RomInfo info;
    EXPECT_FALSE(info.is_loaded());
}

TEST(RomInfoTest, GettersReturnCorrectValues) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 2;
    header.chr_rom_count = 1;
    header.flags6 = 0x30;
    header.flags7 = 0x40;
    header.reserved = {};

    std::vector<uint8_t> prg(2*16384, 0xAA);
    std::vector<uint8_t> chr(1*8192, 0xBB);

    nes::RomInfo info(header, std::move(prg), std::move(chr));

    EXPECT_TRUE(info.is_loaded());
    EXPECT_EQ(info.header().prg_rom_count, 2);
    EXPECT_EQ(info.header().chr_rom_count, 1);
    EXPECT_EQ(info.prg_rom().size(), 2u * 16384u);
    EXPECT_EQ(info.chr_rom().size(), 1u * 8192u);
    EXPECT_EQ(info.mapper_number(), 0x43);
}

TEST(RomInfoTest, ConstInterfaceWorks) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 1;
    header.chr_rom_count = 1;
    header.flags6 = 0x20;
    header.flags7 = 0x00;
    header.reserved = {};

    std::vector<uint8_t> prg(16384, 0xCC);
    std::vector<uint8_t> chr(8192, 0xDD);

    const nes::RomInfo info(header, std::move(prg), std::move(chr));

    EXPECT_TRUE(info.is_loaded());
    EXPECT_EQ(info.header().prg_rom_count, 1);
    EXPECT_EQ(info.prg_rom().size(), 16384u);
    EXPECT_EQ(info.chr_rom().size(), 8192u);
    EXPECT_EQ(info.mapper_number(), 0x02);
    EXPECT_FALSE(info.mirroring());
    EXPECT_FALSE(info.has_save_ram());
    EXPECT_FALSE(info.four_screen());
}
```

### tests/CMakeLists.txt（19行）— 新增 test_rom_info target
```cmake
include(FetchContent)

FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG release-1.12.1
)

FetchContent_MakeAvailable(googletest)

add_executable(test_step0 test_smoke.cpp)
target_link_libraries(test_step0 PRIVATE nes_lib gtest_main)

add_executable(test_rom_info test_rom_info.cpp)
target_link_libraries(test_rom_info PRIVATE nes_lib gtest_main)

include(GoogleTest)
gtest_discover_tests(test_step0)
gtest_discover_tests(test_rom_info)
```

### ctest 結果
```
100% tests passed, 0 tests failed out of 4
  - SmokeTest.BasicAssertion
  - RomInfoTest.DefaultConstructorIsNotLoaded
  - RomInfoTest.GettersReturnCorrectValues
  - RomInfoTest.ConstInterfaceWorks
```

## 討論重點

### 1. 三個 TEST case 的職責分離
- **TEST 1**（DefaultConstructorIsNotLoaded）：驗證預設建構子 `is_loaded() == false`
- **TEST 2**（GettersReturnCorrectValues）：驗證帶參數建構子 + getter 回傳正確值（非 const 物件）
- **TEST 3**（ConstInterfaceWorks）：驗證 const 物件上能呼叫所有 getter（const 介面編譯檢查）
- 三個測試各司其職，不重複

### 2. 為什麼 TEST 2 不用 const
- TEST 2 目的：驗證 getter 回傳**正確值**（非 const 物件）
- TEST 3 目的：驗證 getter 都是 **const 方法**（const 物件）
- 如果 TEST 2 也用 const，跟 TEST 3 職責重複，TEST 3 沒獨立價值
- TEST 2 用非 const 跟 TEST 3 區分職責

### 3. const 介面測試能抓到什麼
- 假設有人不小心把 getter 的 const 拿掉：`const std::vector<uint8_t>& prg_rom();`（忘記 const）
- TEST 2（非 const 物件）：通過（非 const 物件可以呼叫非 const 方法）
- TEST 3（const 物件）：**編譯錯誤**（const 物件不能呼叫非 const 方法）
- TEST 3 是「const 編譯檢查器」，確保所有 getter 都是 const 方法

### 4. `EXPECT_FALSE` / `EXPECT_TRUE` / `EXPECT_EQ`
- `EXPECT_FALSE(x)` — 驗證 x 為 false
- `EXPECT_TRUE(x)` — 驗證 x 為 true
- `EXPECT_EQ(預期, 實際)` — 驗證相等
- `EXPECT_*` 失敗繼續跑（non-fatal）；`ASSERT_*` 失敗立刻停（fatal）
- 測試用 EXPECT 多，ASSERT 用在「後面依賴這個條件」時（如指標非空才能解址）

### 5. `2u * 16384u` 為什麼加 `u`
- `info.prg_rom().size()` 回傳 `size_t`（無號）
- `2 * 16384` 是 `int`（有號），直接比會觸發 `-Wsign-compare` warning
- 加 `u` 變成無號，型別對齊
- `0x43` 是 `int`，`mapper_number()` 回傳 `uint8_t`，gtest 的 EXPECT_EQ 會正確處理

### 6. vector 建構子 `(size, value)`
- `std::vector<uint8_t> prg(2 * 16384, 0xAA)` — 大小 32768，每個元素填 0xAA
- 0xAA = 10101010（二進位），測試填充值，一眼看出資料有沒有正確傳遞
- 不用 0x00 因為全零容易遮蔽 bug（忘記填也是 0）

### 7. 為什麼 vector 不用 array
- 原因 1：RomInfo 建構子參數就是 vector
- 原因 2：array 大小必須編譯期固定，PRG-ROM 大小取決於 `header.prg_rom_count`（執行期才知道）
- Stage 5 檔頭用 array 因為固定 16 byte，PRG/CHR 大小不固定用 vector

### 8. 獨立 target vs 合併
- 選擇獨立 target：保留 test_step0（smoke test）+ 新增 test_rom_info
- 好處：各測試隔離，一個壞不影響另一個
- ctest 分開跑，報告清楚

## Review 建議
- 測試設計優秀：三個 TEST case 職責分離
- TEST 3 的 const 編譯檢查是重要設計，能防止未來不小心移除 const
- `2u * 16384u` 正確處理 sign-compare
- 4 個測試全部通過

## 學習心得
Stage 11 的核心是單元測試設計。三個 TEST case 各司其職：初始狀態、getter 正確值、const 介面。最重要的觀念是 TEST 3 的 const 編譯檢查 — 它不只是跑測試，更是編譯期的防護網，確保所有 getter 都是 const 方法。如果有人不小心移除 const，TEST 3 會直接編譯錯誤。`EXPECT_*` vs `ASSERT_*` 的選擇也是重要觀念：EXPECT 失敗繼續跑收集更多資訊，ASSERT 失敗立刻停避免後續崩潰。
