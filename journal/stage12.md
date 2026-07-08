# Stage 12 日誌：檔案讀取測試 (test_file_rom_loader.cpp)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 測試讀取 `nestest.nes` — 驗證 PRG/CHR count、mapper、mirroring ✅
- 測試讀取不存在的檔案 — 驗證回傳 `ErrorCode::FileNotFound` ✅
- 測試讀取非法檔案（magic 不符）— 驗證回傳 `ErrorCode::IllegalFile` ✅
- 至少 3 個 TEST case ✅
- Build + ctest 全部通過（7 個測試 100%）✅

## 最終程式碼

### test_file_rom_loader.cpp（45行）
```cpp
#include <gtest/gtest.h>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"
#include <fstream>
#include <cstdio>

TEST(FileRomLoaderTest, LoadValidNesFile) {
    nes::FileRomLoader loader("nestest.nes");

    nes::RomInfo info;
    auto result = loader.load(info);

    EXPECT_EQ(result, nes::ErrorCode::Ok);
    EXPECT_TRUE(info.is_loaded());
    EXPECT_EQ(info.header().prg_rom_count, 1);
    EXPECT_EQ(info.header().chr_rom_count, 1);
    EXPECT_EQ(info.mapper_number(), 0);
    EXPECT_FALSE(info.mirroring());
}

TEST(FileRomLoaderTest, LoadNonexistentFile) {
    nes::FileRomLoader loader("nonexistent.nes");
    nes::RomInfo info;
    auto result = loader.load(info);
    EXPECT_EQ(result, nes::ErrorCode::FileNotFound);
    EXPECT_FALSE(info.is_loaded());
}

TEST(FileRomLoaderTest, LoadIllegalFile) {
    // 建一個 magic 不符的假檔案
    const char* filename = "/tmp/bad_magic.nes";
    {
        std::ofstream file(filename, std::ios::binary);
        const char bad_data[16] = {'B', 'A', 'D', 0x00, 1, 1, 0, 0,
                                     0, 0, 0, 0, 0, 0, 0, 0};
        file.write(bad_data, 16);
    }
    nes::FileRomLoader loader(filename);
    nes::RomInfo info;
    auto result = loader.load(info);
    EXPECT_EQ(result, nes::ErrorCode::IllegalFile);
    EXPECT_FALSE(info.is_loaded());
    std::remove(filename);
}
```

### tests/CMakeLists.txt（24行）
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

add_executable(test_file_rom_loader test_file_rom_loader.cpp)
target_link_libraries(test_file_rom_loader PRIVATE nes_lib gtest_main)

include(GoogleTest)
gtest_discover_tests(test_step0)
gtest_discover_tests(test_rom_info)
gtest_discover_tests(test_file_rom_loader
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
```

### ctest 結果
```
100% tests passed, 0 tests failed out of 7
  1. SmokeTest.BasicAssertion              ✅
  2. RomInfoTest.DefaultConstructorIsNotLoaded  ✅
  3. RomInfoTest.GettersReturnCorrectValues      ✅
  4. RomInfoTest.ConstInterfaceWorks        ✅
  5. FileRomLoaderTest.LoadValidNesFile     ✅
  6. FileRomLoaderTest.LoadNonexistentFile  ✅
  7. FileRomLoaderTest.LoadIllegalFile      ✅
```

## 討論重點

### 1. 三種測試路徑覆蓋
- **正常路徑**（LoadValidNesFile）：讀真實的 nestest.nes，驗證 Ok + 各欄位正確
- **錯誤路徑 1**（LoadNonexistentFile）：讀不存在的檔案，驗證 FileNotFound
- **錯誤路徑 2**（LoadIllegalFile）：讀 magic 不符的假檔案，驗證 IllegalFile
- 對應 `load()` 的三個回傳值：Ok、FileNotFound、IllegalFile

### 2. 動態建臨時檔測非法檔案
- TEST 3 用 `std::ofstream` 寫 16 byte 假檔頭到 `/tmp/bad_magic.nes`
- magic 設為 `'B', 'A', 'D', 0x00`（不是 "NES\x1A"）
- 測完用 `std::remove(filename)` 清理臨時檔
- 比「預先準備一個 .nes 檔」更好：測試自足，不依賴外部檔案
- 大括號 `{}` 限制 `ofstream` 作用域，離開自動關檔（RAII）

### 3. `WORKING_DIRECTORY` 的用途
- `gtest_discover_tests(test_file_rom_loader WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})`
- 讓測試程式在 `step0/cpp/` 目錄下執行，才能找到 `nestest.nes`
- 沒設的話測試會在 `build/tests/` 跑，`nestest.nes` 找不到

### 4. CMakeLists.txt 踩坑
- 初版用 `set_tests_properties(test_file_rom_loader PROPERTIES WORKING_DIRECTORY ...)`
- 報錯：找不到 test（gtest_discover_tests 註冊的測試名是個別 case 名，不是 target 名）
- 修正：直接把 `WORKING_DIRECTORY` 當參數傳給 `gtest_discover_tests`

### 5. `nestest.nes` 的檔頭驗證
- 用 `xxd -l 16 nestest.nes` 確認：`4e45 531a 0101 0000...`
- `4e 45 53 1a` = "NES\x1A" magic ✅
- `01` = PRG-ROM count = 1（1 × 16KB = 16KB）
- `01` = CHR-ROM count = 1（1 × 8KB = 8KB）
- `00 00` = flags6/flags7 = 0 → mapper = 0, mirroring = Horizontal

### 6. `--test-dir build` vs `WORKING_DIRECTORY` 不衝突
- `ctest --test-dir build`：控制 ctest 從哪跑（build/ 目錄）
- `WORKING_DIRECTORY`：控制測試程式從哪跑（step0/cpp/）
- 兩個完全不同層面，不衝突

### 7. 只 build 測試的指令
- `cmake --build build --target test_step0 test_rom_info test_file_rom_loader`
- `--target` 指定只編這 3 個 target，CMake 自動先編 nes_lib 依賴
- 不會編 step0_cpp 主程式

## 遇到的問題

### 問題 1：拼字錯誤 `LoadVaildNesFile`
- 初版寫成 `LoadVaildNesFile`（Vaild → Valid）
- 修正：改為 `LoadValidNesFile`

### 問題 2：CMakeLists.txt WORKING_DIRECTORY 設法
- 初版用 `set_tests_properties` → 報錯
- 原因：gtest_discover_tests 註冊的是個別 case 名，不是 target 名
- 修正：直接傳給 `gtest_discover_tests` 的參數

## Review 建議
- 程式碼正確，三種路徑完整覆蓋
- 動態建臨時檔的設計優秀：測試自足、自動清理
- `WORKING_DIRECTORY` 設定正確，確保 nestest.nes 找得到
- 7 個測試全部通過，Step0 C++ 重構測試全部完成

## 學習心得
Stage 12 的核心是整合測試與錯誤路徑測試。三個 TEST case 覆蓋了 `load()` 的三個回傳路徑（Ok、FileNotFound、IllegalFile）。動態建臨時檔測非法 magic 是重要的測試技巧：不依賴外部檔案、測試自足、自動清理。CMake 的 `WORKING_DIRECTORY` 踩坑學到了 `gtest_discover_tests` 的註冊機制 — 測試名是個別 case 不是 target。到此為止，Step0 C++ 重構 13 個 Stage 全部完成，7 個測試 100% 通過。
