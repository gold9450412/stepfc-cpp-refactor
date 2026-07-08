# Stage 10 日誌：測試環境建置 (tests/CMakeLists.txt)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 改主 CMakeLists.txt — 拆出 `nes_lib` 靜態函式庫 ✅
- 建立 `tests/CMakeLists.txt` — FetchContent 抓 Google Test ✅
- 建立 `tests/test_smoke.cpp` — 最小 smoke test ✅
- Build + ctest 通過 ✅

## 最終程式碼

### 主 CMakeLists.txt（22行）
```cmake
cmake_minimum_required(VERSION 3.14)
project(step0_cpp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_library(nes_lib STATIC
    src/nes/nes_header.cpp
    src/nes/rom_info.cpp
    src/nes/file_rom_loader.cpp
    src/nes/famicom.cpp)

target_compile_options(nes_lib PRIVATE -Wall -Wextra)
target_include_directories(nes_lib PUBLIC src)

add_executable(step0_cpp src/main.cpp)
target_link_libraries(step0_cpp PRIVATE nes_lib)

enable_testing()
add_subdirectory(tests)
```

### tests/CMakeLists.txt
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

include(GoogleTest)
gtest_discover_tests(test_step0)
```

### tests/test_smoke.cpp
```cpp
#include <gtest/gtest.h>

TEST(SmokeTest, BasicAssertion) {
    EXPECT_EQ(1 + 1, 2);
}
```

### Build + ctest 結果
```
100% tests passed, 0 tests failed out of 1
```

## 討論重點

### 1. 為什麼要拆出 nes_lib 函式庫
- 之前主程式和 NES 程式碼都編進 `step0_cpp` 一個執行檔
- 加測試後，測試也需要連結 NES 程式碼
- 拆出 `nes_lib` 靜態函式庫：主程式和測試都 link 它，不重複編譯 .cpp
- 對照原版：只有一個執行檔，沒有測試，不需要拆

### 2. 三種函式庫 STATIC / SHARED / MODULE
| 類型 | 產物 | 連結時機 |
|------|------|---------|
| STATIC | `libnes_lib.a` | 編譯時 copy 進執行檔 |
| SHARED | `libnes_lib.so` | 執行時載入（動態連結） |
| MODULE | `.so`/`.dylib` | `dlopen()` 動態載入 |

- 小專案、測試環境用 STATIC，簡單不惹麻煩

### 3. `target_include_directories` 的 PUBLIC vs PRIVATE
- `nes_lib PUBLIC src` — nes_lib 用 `src` 當 include 根，**下游也看得到**
- PRIVATE = 只自己用，下游看不到
- nes_lib 有下游（step0_cpp 和測試都 link 它），下游需要找到 `src/nes/*.h`
- PUBLIC = 設定會傳遞給所有連結 nes_lib 的人

### 4. `target_link_libraries(step0_cpp PRIVATE nes_lib)` 為什麼 PRIVATE
- 執行檔沒有下游（沒人 link 執行檔），用 PRIVATE 就夠
- 執行檔連結函式庫一律用 PRIVATE

### 5. 為什麼用 `src` 當 include 根，不是 `src/nes`
- 用 `src` 當根 + `nes/` 前綴：`#include "nes/rom_info.h"`
- 等於自帶 namespace 防撞名，是業界慣例
- 如果用 `src/nes` 當根，所有 include 要改成 `#include "rom_info.h"`，失去前綴保護

### 6. FetchContent — CMake 下載第三方函式庫
- `FetchContent_Declare` 宣告來源（GitHub repo + tag）
- `FetchContent_MakeAvailable` 下載並編譯
- 第一次 build 花 1-2 分鐘下載編譯 gtest，之後有 cache 不用重抓
- 對照手動安裝：不用 `apt install`，不用 `find_package`，CMake 全自動

### 7. Google Test 基礎
- `TEST(套件名, 案例名)` 定義一個測試案例
- `EXPECT_EQ(預期, 實際)` — 相等通過，不相等繼續跑（適合多個檢查）
- `ASSERT_EQ(預期, 實際)` — 失敗立刻停止（適合前置條件）
- `gtest_main` 提供 main 進入點，不用自己寫 `int main()`
- `gtest_discover_tests` 掃描所有 TEST() 自動註冊到 ctest

### 8. `enable_testing()` + `add_subdirectory(tests)`
- `enable_testing()` 在主 CMakeLists 啟用 ctest
- `add_subdirectory(tests)` 把 tests/ 目錄加入建置
- 執行測試：`ctest --test-dir build --output-on-failure`

## 遇到的問題
- 無重大問題，第一次 build 較慢（下載編譯 gtest）

## Review 建議
- CMake 結構正確，拆出 nes_lib 是測試的必要前置
- FetchContent 用法正確，pin 在 release-1.12.1 確保可重現
- smoke test 確認環境正常，Stage 11-12 可以開始寫實際測試

## 學習心得
Stage 10 的核心是建置測試環境。拆出 `nes_lib` 靜態函式庫是關鍵 — 讓主程式和測試共用同一份編譯結果，不重複編譯。CMake FetchContent 是現代 C++ 專案管理第三方依賴的標準做法，比手動安裝或 git submodule 更乾淨。PUBLIC vs PRIVATE 的區分：函式庫要給下游用的用 PUBLIC，執行檔沒有下游用 PRIVATE。Google Test 的 `TEST()` + `EXPECT_EQ` 模式簡單直覺，`gtest_discover_tests` 自動註冊測試到 ctest，不用手動列舉。
