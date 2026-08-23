---
layout: default
title: Stage 0：環境設定 + 複製 step0 程式碼
---

# Stage 0 日誌：環境設定 + 複製 step0 程式碼

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 確認 step0 程式碼已複製到 `step1/cpp/`（10 個檔案：error.h, nes_header.h/.cpp, rom_info.h/.cpp, rom_loader.h, file_rom_loader.h/.cpp, famicom.h/.cpp, main.cpp）✅
- 確認 3 個舊測試已複製到 `step1/cpp/tests/`（test_smoke.cpp, test_rom_info.cpp, test_file_rom_loader.cpp）✅
- 確認 `nestest.nes` 已複製 ✅
- 建立 `CMakeLists.txt`（專案名 `step1_cpp`，nes_lib 含 step0 的 .cpp）✅
- 建立 `tests/CMakeLists.txt`（FetchContent gtest + 3 個舊 test target）✅
- `cmake -B build && cmake --build build && ctest --test-dir build` 通過（7 個舊測試全過）✅

## 與 step0 CMakeLists.txt 的差異
- 專案名：`step0_cpp` → `step1_cpp`
- 結構相同：nes_lib 靜態函式庫 + 執行檔 + Google Test
- nes_lib 包含相同的 4 個 .cpp（nes_header, rom_info, file_rom_loader, famicom）
- 測試沿用 step0 的 3 個 test target（smoke, rom_info, file_rom_loader）

## 複製的檔案清單
### src/nes/（10 個檔案）
| 檔案 | 來源 | 功能 |
|------|------|------|
| error.h | step0 | ErrorCode enum class |
| nes_header.h | step0 | iNES 檔頭結構 + parse_header |
| nes_header.cpp | step0 | parse_header 實作 |
| rom_info.h | step0 | ROM 資訊類別 + getter |
| rom_info.cpp | step0 | RomInfo 實作 |
| rom_loader.h | step0 | 抽象讀取介面 |
| file_rom_loader.h | step0 | 檔案讀取類別 |
| file_rom_loader.cpp | step0 | load() 實作 |
| famicom.h | step0 | 模擬器主體（step1 會擴展） |
| famicom.cpp | step0 | Famicom 實作（step1 會擴展） |

### tests/（3 個測試 + CMakeLists.txt）
| 檔案 | 測試數 | 功能 |
|------|--------|------|
| test_smoke.cpp | 1 | 最小煙霧測試 |
| test_rom_info.cpp | 3 | RomInfo 初始狀態 + getter + const |
| test_file_rom_loader.cpp | 3 | 檔案讀取：正常/不存在/非法 |

## Build 結果
```
100% tests passed, 0 tests failed out of 7
  1. SmokeTest.BasicAssertion
  2. RomInfoTest.DefaultConstructorIsNotLoaded
  3. RomInfoTest.GettersReturnCorrectValues
  4. RomInfoTest.ConstInterfaceWorks
  5. FileRomLoaderTest.LoadValidNesFile
  6. FileRomLoaderTest.LoadNonexistentFile
  7. FileRomLoaderTest.LoadIllegalFile
```

## 學習心得
Stage 0 的核心是專案重用。Step1 建立在 Step0 的基礎上 — ROM 載入、檔頭解析、Famicom 主體都已經完成，直接複製過來就能用。CMake 的多目標建置讓我們能輕鬆重用 nes_lib 函式庫，測試也能沿用。接下來 Stage 1 開始才是 Step1 的新功能：中斷向量常數、Mapper、CpuBus。
