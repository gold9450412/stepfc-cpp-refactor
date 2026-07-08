---
layout: default
title: Step0 C++ 重構規格書
---

# Step0 C++ 重構規格書

## 目的

將 StepFC 的 Step0（讀取 NES ROM）用現代 C++ 重新實作，學習業界級別的 C++ 開發實踐與模擬器架構設計。由你親自撰寫程式碼，我負責引導與 review。

## 與原版的差異（C → 現代 C++）

| 原版 (C) | 重構版 (C++) | 為什麼 |
|----------|-------------|--------|
| `malloc`/`free` | `std::vector<uint8_t>` | RAII，離開作用域自動釋放 |
| `FILE*` + `fread` | `std::ifstream` | RAII，建構子開檔、解構子自動關檔 |
| 函數指標 | 純虛擬類別 (abstract class) | 物件導向多型，型別安全 |
| `enum` (unscoped) | `enum class` / `constexpr` | 不會隱式轉成 int，不會名稱衝突 |
| `struct` (全部 public) | `class` (封裝 + getter) | 保護內部資料 |
| `init`/`uninit` 函數 | 建構子/解構子 | RAII，物件存在就是可用狀態 |
| 全域函數 | `namespace` 組織 | 避免名稱衝突 |
| Makefile | CMake | 業界標準建置系統 |
| 無測試 | Google Test | 業界標準單元測試框架 |
| 無紀錄 | 開發日誌 | 每階段記錄討論、問題、決策 |
| struct 直接映射磁碟格式 | 逐欄位安全解析 | 避免 endian 問題、不依賴 struct memory layout |

## C++ 版本

C++17

## 需要安裝的工具與函式庫

### 系統工具（需預先安裝）

| 工具 | 最低版本 | 用途 | 安裝方式 (Ubuntu) |
|------|---------|------|-------------------|
| **g++** 或 **clang++** | >= 7 | C++17 編譯器 | `sudo apt install g++` |
| **CMake** | >= 3.14 | 建置系統（FetchContent 需要 3.14+） | `sudo apt install cmake` |
| **Git** | 任意 | CMake FetchContent 抓取第三方函式庫 | `sudo apt install git` |
| **make** | 任意 | CMake 預設 generator | `sudo apt install make` |

### 第三方函式庫（透過 CMake FetchContent 自動抓取）

| 函式庫 | 用途 | 抓取方式 |
|--------|------|---------|
| **Google Test** (gtest) | 單元測試框架 | CMake `FetchContent` 從 GitHub 自動下載 |

## 專案結構

```
step0/cpp/
├── CMakeLists.txt
├── SPEC.md                       ← 本規格書
├── nestest.nes                   ← 測試用 ROM（從 step0/ 複製）
├── src/
│   ├── main.cpp
│   └── nes/
│       ├── error.h               — 錯誤碼 (enum class)
│       ├── nes_header.h          — NES 檔頭結構
│       ├── rom_info.h            — ROM 資訊類別
│       ├── rom_info.cpp
│       ├── rom_loader.h          — 讀取介面（抽象類別）
│       ├── file_rom_loader.h     — 檔案讀取實作
│       ├── file_rom_loader.cpp
│       ├── famicom.h             — 模擬器主體
│       └── famicom.cpp
├── tests/
│   ├── CMakeLists.txt
│   ├── test_rom_info.cpp
│   └── test_file_rom_loader.cpp
└── docs/
    └── journal/                  ← 開發日誌
        ├── stage00.md            — Stage 0 完成後的日誌
        ├── stage01.md            — Stage 1 完成後的日誌
        ├── ...
        └── stage12.md            — Stage 12 完成後的日誌
```

## 開發日誌規則

每個 Stage 完成後，你切到 build 模式，我會在 `docs/journal/stageXX.md` 寫入該階段的日誌，內容包含：

- **完成事項** — 該 stage 做了什麼
- **討論重點** — 我們討論了哪些設計決策
- **遇到的問題** — 實作中碰到的困難、bug、觀念釐清
- **Review 建議** — 我對程式碼的改進建議
- **學習心得** — 該 stage 的核心學習重點回顧

日誌完成後，你切回 plan 模式繼續下一個 stage。

## Stages

### Stage 0: 建置環境
- [x] 確認系統已安裝 g++、cmake、git、make
- [x] 建立 `CMakeLists.txt`（C++17、專案名 `step0_cpp`、啟用警告 `-Wall -Wextra`）
- [x] 建立最小 `src/main.cpp`（印出 `"Step0 C++ - NES ROM Loader"`）
- [x] 確認 `cmake -B build && cmake --build build` 能成功編譯執行

**需安裝**：g++ >= 7、CMake >= 3.14、git、make

**學習重點**：CMake 基礎、C++17 編譯設定、編譯器警告

---

### Stage 1: 錯誤碼定義 (`error.h`)
- [x] 定義 `enum class ErrorCode`，包含：`Ok, FileNotFound, IllegalFile, OutOfMemory`
- [x] 放在 `namespace nes` 裡
- [x] 每個值加一行註解說明用途

**需安裝**：無

**學習重點**：`enum class` vs `enum`、`namespace`

**對照原版**：`sfc_code.h`

---

### Stage 2: NES 檔頭結構 (`nes_header.h`)
- [x] 定義 `struct NesHeader`，為**解析後的值型別**（非磁碟格式的直接映射）
  - `std::array<uint8_t, 4> magic` — 不用 `uint32_t`，避免 endian 問題
  - `uint8_t prg_rom_count`, `chr_rom_count`, `flags6`, `flags7`
  - `std::array<uint8_t, 8> reserved`
- [x] 定義 `constexpr std::array<uint8_t, 4> kMagic{'N', 'E', 'S', 0x1A}`
- [x] 用 `namespace Flags6` / `namespace Flags7` + `constexpr` 定義位元遮罩常數
- [x] 實作 `bool parse_header(const std::array<uint8_t, 16>& raw, NesHeader& out)`
  - 逐 byte 從 raw 取出各欄位，不依賴 struct memory layout
  - 驗證 magic number，不符回傳 `false`

**需安裝**：無（`<array>`, `<cstdint>`）

**學習重點**：`struct` vs `class`、`constexpr`、`std::array`、endian safety、安全解析 vs 暴力映射

**對照原版**：`sfc_nes_header_t` + `SFC_NES_*`（原版直接 fread struct，有 endian 風險）

---

### Stage 3: ROM 資訊類別 (`rom_info.h` / `rom_info.cpp`)
- [x] 定義 `class RomInfo`，私有成員 + const getter
- [x] 持有 `NesHeader header_` — 解析後的檔頭
- [x] 持有 `std::vector<uint8_t> prg_rom_`, `chr_rom_` — ROM 資料本體
- [x] `bool is_loaded() const` — 資料是否已載入
- [x] 只提供 getter，不提供 setter（唯讀介面）

**需安裝**：無（`<vector>`）

**學習重點**：封裝、`const` 方法、`std::vector`、物件擁有資料

**對照原版**：`sfc_rom_info_t`（原版用 raw pointer + malloc，本版用 vector 持有資料）

---

### Stage 4: 讀取介面 — 抽象類別 (`rom_loader.h`)
- [x] 定義抽象類別 `class RomLoader`
- [x] 純虛擬函數 `virtual ErrorCode load(RomInfo& out) = 0;`
- [x] 虛擬解構子 `virtual ~RomLoader() = default;`
- [x] 不需要 `free()` — RAII 自動處理（vector 離開作用域自動釋放）

**需安裝**：無（`<string>`）

**學習重點**：純虛擬函數、抽象類別、虛擬解構子、介面設計

**對照原版**：`sfc_interface_t`（原版有 `load_rom` + `free_rom`，本版用 RAII 不需要 free）

---

### Stage 5: 檔案讀取 — 開檔與驗證 (`file_rom_loader.h` / `.cpp` 前半)
- [x] `class FileRomLoader : public RomLoader`
- [x] 建構子接收檔案路徑 `std::string`
- [x] `std::ifstream` 二進位開檔
- [x] 讀取 16 byte 到 `std::array<uint8_t, 16>`
- [x] 呼叫 `parse_header()` 驗證 magic number

**需安裝**：無（`<fstream>`）

**學習重點**：`std::ifstream`、二進位讀檔、介面實作

**對照原版**：`sfc_load_default_rom` 前半

---

### Stage 6: 檔案讀取 — 讀取 PRG/CHR 資料 (`file_rom_loader.cpp` 中半)
- [x] 根據 header 算出 PRG/CHR 大小（`count * 16KB` / `count * 8KB`）
- [x] 跳過 Trainer（若 `flags6 & Flags6::Trainer` 有設定，跳 512 bytes）
- [x] `std::vector<uint8_t>` 配置並用 `ifstream::read()` 讀取資料

**需安裝**：無

**學習重點**：`std::vector`、`seekg()` / `read()`、動態配置

**對照原版**：`malloc` + `fread`

---

### Stage 7: 檔案讀取 — 解析 Flags 填入 ROM 資訊 (`file_rom_loader.cpp` 後半)
- [x] 拆解 `flags6` / `flags7` 取得 mirroring, trainer, save_ram, four_screen, mapper
- [x] mapper number = `(flags6 >> 4) | (flags7 & 0xF0)`
- [x] 將 header + prg_data + chr_data 組裝進 `RomInfo`
- [x] 完成 `load()` 回傳 `ErrorCode::Ok`
- [x] 錯誤路徑回傳對應 `ErrorCode`（檔案不存在、magic 不符等）

**需安裝**：無

**學習重點**：位元運算、錯誤處理流程、完成完整類別

**對照原版**：填寫 `info->mapper_number` 等（原版另有 `free_rom`，本版用 RAII 不需要）

---

### Stage 8: 模擬器主體 (`famicom.h` / `famicom.cpp`)
- [x] `class Famicom`，持有 `RomInfo`（直接擁有，非指標）
- [x] 建構子接收 `RomInfo`（by move，轉移所有權）
- [x] `const RomInfo& get_rom_info() const` — 唯讀存取
- [x] 解構子無需手動清理 — RAII 自動處理

**需安裝**：無（`<memory>` for `std::move`）

**學習重點**：RAII、move 語意、物件生命週期、最小責任原則

**架構決策**：Famicom 只擁有 RomInfo，不擁有 RomLoader。
RomLoader 是短命物件（讀完即丟），不該被模擬器長期持有。

**對照原版**：`sfc_famicom_t` + init/uninit（原版同時持有 interface + rom_info）

---

### Stage 9: 主程式 (`main.cpp`)
- [x] 用 `std::make_unique<FileRomLoader>` 建立 loader（展示 `unique_ptr` 多型）
- [x] 呼叫 `loader->load(info)` 載入 ROM
- [x] 用 `std::move(info)` 將 RomInfo 移入 `Famicom`
- [x] loader 離開作用域自動釋放（RAII）
- [x] 印出 ROM 資訊，確認輸出正確

**需安裝**：無

**學習重點**：`std::unique_ptr`、`std::make_unique`、`std::move`、組裝所有元件

**對照原版**：`main.c`（原版用 malloc + 函數指標，本版用 unique_ptr + move）

---

### Stage 10: 測試環境建置 (`tests/CMakeLists.txt`)
- [x] CMake `FetchContent` 抓取 Google Test
- [x] 最小 smoke test 確認能跑
- [x] `ctest` 通過

**需安裝**：Google Test（CMake 自動抓取，**不用手動安裝**）

**學習重點**：CMake FetchContent、Google Test 基礎、`ctest`

---

### Stage 11: ROM 資訊測試 (`tests/test_rom_info.cpp`)
- [x] 測試初始狀態（`is_loaded() == false`）
- [x] 測試 getter 回傳正確值（建構後填入資料再驗證）
- [x] 測試 const 介面（const reference 上只能呼叫 const 方法）
- [x] 至少 3 個 TEST case

**需安裝**：Google Test（Stage 10 已設定）

**學習重點**：`TEST` 巨集、`ASSERT_EQ` / `EXPECT_EQ`、測試封裝

---

### Stage 12: 檔案讀取測試 (`tests/test_file_rom_loader.cpp`)
- [x] 測試讀取 `nestest.nes` — 驗證 PRG/CHR count、mapper、mirroring
- [x] 測試讀取不存在的檔案 — 驗證回傳 `ErrorCode::FileNotFound`
- [x] 測試讀取非法檔案（magic 不符）— 驗證回傳 `ErrorCode::IllegalFile`
- [x] 至少 3 個 TEST case

**需安裝**：Google Test（Stage 10 已設定）

**學習重點**：整合測試、fixture、錯誤路徑測試

---

## 工作流程

```
你寫程式碼 (plan 模式討論)
       ↓
完成一個 Stage
       ↓
切到 build 模式
       ↓
我寫日誌到 docs/journal/stageXX.md
       ↓
切回 plan 模式
       ↓
繼續下一個 Stage
```

## 預期最終輸出

主程式：
```
Step0 C++ - NES ROM Loader
ROM: PRG-ROM: 1 x 16kb   CHR-ROM: 1 x 8kb   Mapper: 000
Mirroring: Horizontal   FourScreen: No   SaveRAM: No
```

測試：
```
[==========] Running N tests from M test suites.
[  PASSED  ] N tests.
```

## 原則

1. **每個 Stage 完成後跟我回報**，我會 review 並給建議
2. **不急著往前衝** — 每個 stage 確實搞懂再往下
3. **有問題隨時問** — 不懂的地方比寫錯更值得討論
4. **程式碼由你寫** — 我只引導和 review，不直接幫你寫
5. **每個 Stage 完成後寫日誌** — 記錄學習軌跡
