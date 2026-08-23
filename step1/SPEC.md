---
layout: default
title: Step1 C++ 重構規格書
---

# Step1 C++ 重構規格書

## 目的

將 StepFC 的 Step1（CPU 記憶體位址空間 + Mapper000 + 中斷向量）用現代 C++ 重新實作，學習業界級別的 C++ 開發實踐與模擬器架構設計。由你親自撰寫程式碼，我負責引導與 review。

## 與原版的差異（C → 現代 C++）

| 原版 (C) | 重構版 (C++) | 為什麼 |
|----------|-------------|--------|
| `prg_banks[8]` 裸指標陣列 | Mapper 類別封裝 bank 邏輯 | 外部無法直接修改指標 |
| `main_memory[2KB]` 裸陣列 | `std::array<uint8_t, 2048>` | 有邊界檢查（`.at()`），不退化成指標 |
| `save_memory[8KB]` 裸陣列 | `std::array<uint8_t, 8192>` | 同上 |
| `sfc_read_cpu_address` 自由函數 | `CpuBus` 類別封裝讀寫 | 讀寫邏輯集中，不散落 |
| `sfc_mapper_t` 函數指標 struct | 抽象 `Mapper` 類別 + `virtual` | 型別安全的多型 |
| `sfc_load_mapper` switch/case | Factory function 回傳 `unique_ptr<Mapper>` | 易擴展，不用改主程式 |
| `assert(!"NOT IMPL")` | Stub 方法回傳 0 | release mode 也不會炸 |
| `memset` 初始化 | 成員初始式 `{}` | 不忘記初始化 |
| `sfc_famicom_t` 全 public | `Famicom` 封裝 + getter | 保護內部資料 |
| Makefile | CMake | 業界標準建置系統 |
| 無測試 | Google Test | 業界標準單元測試框架 |

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
step1/cpp/
├── CMakeLists.txt
├── SPEC.md                        ← 本規格書
├── nestest.nes                    ← 測試用 ROM（從 step0 複製）
├── src/
│   ├── main.cpp                   ← 主程式（讀 ROM + 印中斷向量）
│   └── nes/
│       ├── error.h                — 錯誤碼 (enum class) ← 從 step0 複製 + 加 MapperNotFound
│       ├── nes_header.h           — NES 檔頭結構 ← 從 step0 複製
│       ├── nes_header.cpp         — 檔頭解析 ← 從 step0 複製
│       ├── rom_info.h             — ROM 資訊類別 ← 從 step0 複製
│       ├── rom_info.cpp
│       ├── rom_loader.h           — 讀取介面 ← 從 step0 複製
│       ├── file_rom_loader.h      — 檔案讀取 ← 從 step0 複製
│       ├── file_rom_loader.cpp
│       ├── cpu_vectors.h          — 中斷向量位址常數
│       ├── mapper.h               — 抽象 Mapper 類別
│       ├── mapper000.h            — NROM mapper
│       ├── mapper000.cpp
│       ├── cpu_bus.h              — CPU 位址空間 (64KB)
│       ├── cpu_bus.cpp
│       ├── famicom.h              — 模擬器主體（擁有 CpuBus + Mapper）
│       └── famicom.cpp
├── tests/
│   ├── CMakeLists.txt
│   ├── test_smoke.cpp             ← 從 step0 複製
│   ├── test_rom_info.cpp          ← 從 step0 複製
│   ├── test_file_rom_loader.cpp   ← 從 step0 複製
│   ├── test_cpu_bus.cpp            — CPU 匯流排測試
│   └── test_mapper000.cpp          — NROM mapper 測試
└── docs/
    └── journal/                   ← 開發日誌
        ├── stage00.md
        ├── stage01.md
        ├── ...
        └── stage12.md
```

## 開發日誌規則

每個 Stage 完成後，你切到 build 模式，我會在 `docs/journal/stageXX.md` 寫入該階段的日誌，內容包含：

- **完成事項** — 該 stage 做了什麼
- **討論重點** — 我們討論了哪些設計決策
- **遇到的問題** — 實作中碰到的困難、bug、觀念釐清
- **Review 建議** — 我對程式碼的改進建議
- **學習心得** — 該 stage 的核心學習重點回顧

日誌完成後，你切回 plan 模式繼續下一個 stage。

## 架構設計

```
Famicom
  ├── RomInfo (from step0)
  ├── unique_ptr<Mapper> (抽象類別)
  │     └── Mapper000 (NROM)
  └── CpuBus (CPU 64KB 位址空間)
        ├── RAM (2KB, std::array<uint8_t, 2048>)
        ├── SRAM (8KB, std::array<uint8_t, 8192>)
        └── Mapper* (指向 Famicom 的 mapper_)
```

### CPU 記憶體位址空間佈局

| 位址 | 大小 | 標記 | 描述 |
|------|------|------|------|
| $0000 | $800 | | RAM |
| $0800 | $800 | M | RAM (鏡像) |
| $1000 | $800 | M | RAM (鏡像) |
| $1800 | $800 | M | RAM (鏡像) |
| $2000 | 8 | | PPU 暫存器 |
| $2008 | $1FF8 | R | PPU 暫存器 (8 byte 步進鏡像) |
| $4000 | $20 | | APU 暫存器 |
| $4020 | $1FDF | | Expansion ROM |
| $6000 | $2000 | | SRAM |
| $8000 | $4000 | | PRG-ROM (Mapper 控制) |
| $C000 | $4000 | | PRG-ROM (Mapper 控制) |

M = 主記憶體 2KB 鏡像，R = PPU 暫存器 8 byte 步進鏡像

### 中斷向量

| 位址 | 向量 | 說明 |
|------|------|------|
| $FFFA-$FFFB | NMI | 不可遮蔽中斷 |
| $FFFC-$FFFD | RESET | 開機/重置進入點 |
| $FFFE-$FFFF | IRQ/BRK | 可遮蔽中斷/軟體中斷 |

低 byte 在低位址，高 byte 在高位址（little-endian）。

### Mapper000 (NROM)

- 16KB PRG-ROM：$8000-$BFFF = 前 16KB，$C000-$FFFF = 鏡像
- 32KB PRG-ROM：$8000-$FFFF = 直接映射
- bank = 8KB 單位，PRG-ROM 被切成 4 個 bank（32KB）或 2 個 bank + 2 個鏡像（16KB）

## Stages

### Stage 0: 環境設定 + 複製 step0 程式碼
- [x] 確認 step0 程式碼已複製到 `step1/cpp/`（src/nes/*.h/.cpp, tests/, nestest.nes）
- [x] 建立 `CMakeLists.txt`（專案名 `step1_cpp`，nes_lib 含 step0 的 .cpp）
- [x] 建立 `tests/CMakeLists.txt`（FetchContent gtest + 3 個舊 test target）
- [x] 確認 `cmake -B build && cmake --build build && ctest --test-dir build` 通過（7 個舊測試全過）

**需安裝**：g++ >= 7、CMake >= 3.14、git、make

**學習重點**：專案重用、CMake 多目標建置

**對照原版**：無（基礎建設）

---

### Stage 1: 中斷向量常數 (`cpu_vectors.h`)
- [x] 定義 `constexpr uint16_t` 常數：`kNmiVector`, `kResetVector`, `kIrqVector`
  - `kNmiVector = 0xFFFA`
  - `kResetVector = 0xFFFC`
  - `kIrqVector = 0xFFFE`
- [x] 放在 `namespace nes` 裡

**需安裝**：無（`<cstdint>`）

**學習重點**：`constexpr` 硬體位址常數、`uint16_t` 命名

**對照原版**：原版直接用 magic number `0xFFFA` 等

---

### Stage 2: 抽象 Mapper 類別 (`mapper.h`)
- [x] 定義 `class Mapper`，抽象類別
- [x] 純虛擬函數 `virtual uint8_t read_prg(uint16_t addr) const = 0;`
- [x] 純虛擬函數 `virtual void write_prg(uint16_t addr, uint8_t data) = 0;`（ROM 通常忽略寫入）
- [x] 虛擬解構子 `virtual ~Mapper() = default;`

**需安裝**：無（`<cstdint>`）

**學習重點**：抽象類別複習、介面設計、const 方法 vs 非 const 方法

**對照原版**：`sfc_mapper_t`（函數指標 struct）

---

### Stage 3: Mapper000 NROM (`mapper000.h` / `mapper000.cpp`)
- [x] 定義 `class Mapper000 : public Mapper`
- [x] 建構子接收 `const RomInfo&`（從 RomInfo 取 PRG-ROM 資料指標和大小）
- [x] 實作 `read_prg(addr)`：
  - 16KB PRG：`addr % 16384` 映射到 PRG-ROM 資料
  - 32KB PRG：`addr` 直接映射（$8000-$FFFF）
- [x] 實作 `write_prg(addr, data)`：ROM 唯讀，忽略寫入（空實作）
- [x] private 成員：PRG-ROM 資料指標（`const uint8_t*`）+ PRG-ROM 大小

**需安裝**：無

**學習重點**：具體實作、bank 映射邏輯、ROM 唯讀概念、指標 vs 參考

**對照原版**：`sfc_mapper000_reset`（bank 指標賦值）

---

### Stage 4: Mapper Factory 函式
- [x] 在 `mapper.h` 或 `mapper_factory.h` 定義 `std::unique_ptr<Mapper> make_mapper(const RomInfo& rom)`
- [x] 實作在 `mapper_factory.cpp`（或 `mapper.cpp`）
- [x] 根據 `rom.mapper_number()` switch：
  - `0` → 回傳 `make_unique<Mapper000>(rom)`
  - 其他 → 回傳 `nullptr`（之後加更多 mapper）
- [x] 在 `error.h` 加 `ErrorCode::MapperNotFound`

**需安裝**：無（`<memory>`）

**學習重點**：Factory pattern、`unique_ptr` 多型、switch + factory

**對照原版**：`sfc_load_mapper`（switch/case + 函數指標賦值）

---

### Stage 5: CpuBus 骨架 (`cpu_bus.h` / `cpu_bus.cpp`)
- [x] 定義 `class CpuBus`
- [x] private 成員：
  - `std::array<uint8_t, 2048> ram_{};` — 2KB RAM
  - `std::array<uint8_t, 8192> sram_{};` — 8KB SRAM
  - `Mapper* mapper_;` — 指向 Famicom 擁有的 mapper（非擁有）
- [x] 建構子 `explicit CpuBus(Mapper* mapper)` — 接收 mapper 指標
- [x] 宣告 `uint8_t read(uint16_t addr) const;` 和 `void write(uint16_t addr, uint8_t data);`

**需安裝**：無（`<array>`, `<cstdint>`）

**學習重點**：類別組合、成員設計、非擁有指標 vs 擁有指標

**對照原版**：`sfc_famicom_t` 裡的 `main_memory` + `save_memory` + `prg_banks`

---

### Stage 6: CpuBus read() 實作 (`cpu_bus.cpp`)
- [x] 實作 `read(uint16_t addr)`：
  - `$0000-$1FFF`：RAM，`addr & 0x07FF`（2KB 鏡像）
  - `$2000-$3FFF`：PPU 暫存器 stub，回傳 0
  - `$4000-$401F`：APU 暫存器 stub，回傳 0
  - `$4020-$5FFF`：Expansion ROM stub，回傳 0
  - `$6000-$7FFF`：SRAM，直接索引
  - `$8000-$FFFF`：PRG-ROM，透過 `mapper_->read_prg(addr - 0x8000)`

**需安裝**：無

**學習重點**：位址解碼、bit masking（`& 0x07FF`）、鏡像原理

**對照原版**：`sfc_read_cpu_address`（switch + 指標讀取）

---

### Stage 7: CpuBus write() 實作 (`cpu_bus.cpp`)
- [x] 實作 `write(uint16_t addr, uint8_t data)`：
  - `$0000-$1FFF`：RAM，`addr & 0x07FF`
  - `$2000-$3FFF`：PPU 暫存器 stub，忽略寫入
  - `$4000-$401F`：APU 暫存器 stub，忽略寫入
  - `$4020-$5FFF`：Expansion ROM stub，忽略寫入
  - `$6000-$7FFF`：SRAM，直接寫入
  - `$8000-$FFFF`：PRG-ROM，透過 `mapper_->write_prg()`（ROM 通常忽略）

**需安裝**：無

**學習重點**：寫入分派、ROM 寫入保護、const vs 非 const 方法

**對照原版**：`sfc_write_cpu_address`（switch + 指標寫入）

---

### Stage 8: Famicom 擴展 (`famicom.h` / `famicom.cpp`)
- [x] 擴展 `class Famicom`，新增成員：
  - `std::unique_ptr<Mapper> mapper_;`
  - `CpuBus bus_;`
- [x] 建構子：接收 `RomInfo`（by move），呼叫 `make_mapper` 建立 mapper，用 mapper 指標建構 bus
- [x] `uint8_t cpu_read(uint16_t addr) const;` — 透過 bus 讀
- [x] `void cpu_write(uint16_t addr, uint8_t data);` — 透過 bus 寫
- [x] `const Mapper& mapper() const;` — 唯讀存取 mapper

**需安裝**：無（`<memory>`）

**學習重點**：物件組合、生命週期管理、初始化順序

**對照原版**：`sfc_famicom_t` init（同時建記憶體 + mapper）

---

### Stage 9: main.cpp — 讀中斷向量
- [x] 載入 ROM → 建 Famicom
- [x] 印出 ROM 資訊（沿用 step0）
- [x] 讀三個中斷向量：
  - `nmi = cpu_read(kNmiVector) | (cpu_read(kNmiVector + 1) << 8)`
  - `reset = cpu_read(kResetVector) | (cpu_read(kResetVector + 1) << 8)`
  - `irq = cpu_read(kIrqVector) | (cpu_read(kIrqVector + 1) << 8)`
- [x] 印出三個向量位址

**需安裝**：無

**學習重點**：little-endian 組裝、整合所有元件

**對照原版**：`main.c`（讀中斷向量 + 印出）

**預期輸出**（nestest.nes）：
```
Step1 C++ - CPU Memory & Mapper
ROM: PRG-ROM: 1 x 16kb    CHR-ROM: 1 x 8kb    Mapper: 0
Mirroring: Horizontal    Save RAM: No    Four Screen: No
NMI Vector:   $C5AF
RESET Vector: $C004
IRQ Vector:   $C5F4
```
（nestest.nes 的 PRG-ROM 最後 6 bytes = `af c5 04 c0 f4 c5`，分別對應 NMI/RESET/IRQ 向量）

---

### Stage 10: CpuBus 測試 (`tests/test_cpu_bus.cpp`)
- [x] 測試 RAM 鏡像：寫 $0000 讀 $0800 應該一樣
- [x] 測試 RAM 邊界：$07FF 和 $0FFF/$17FF/$1FFF 是同一個 byte
- [x] 測試 SRAM 讀寫：$6000-$7FFF
- [x] 測試 PPU/APU stub：讀 $2000 和 $4000 回傳 0
- [x] 測試 PRG-ROM 讀取：$8000-$FFFF 透過 mapper
- [x] 至少 4 個 TEST case（共 5 個）

**需安裝**：Google Test（Stage 10 已設定）

**學習重點**：測試鏡像、位址分派、mock/stub 測試

---

### Stage 11: Mapper000 測試 (`tests/test_mapper000.cpp`)
- [x] 測試 16KB PRG：$8000 讀到第一 byte，$C000 讀到鏡像（同 $8000）
- [x] 測試 32KB PRG：$8000 和 $C000 讀到不同資料
- [x] 測試 write_prg：寫入不影響 read（ROM 唯讀）
- [x] 至少 3 個 TEST case

**需安裝**：Google Test

**學習重點**：測試 NROM bank 映射、唯讀驗證

---

### Stage 12: 整合測試 + 收尾
- [x] 測試讀取 nestest.nes 的中斷向量（NMI=$C5AF, RESET=$C004, IRQ=$C5F4）
- [x] 測試 `make_mapper` 對未知 mapper 回傳 nullptr
- [x] 確認所有測試通過（舊 7 + 新測試）
- [x] 確認 SPEC.md 所有 checkbox 打勾
- [x] 確認所有日誌寫完

**需安裝**：Google Test

**學習重點**：端到端測試、回歸測試

---

## 工作流程

1. **plan 模式**：討論設計、寫程式碼、review
2. **build 模式**：寫日誌、更新 SPEC.md checkbox
3. 每個 Stage 完成 → 切 build 模式寫日誌 → 切回 plan 模式繼續下一個

## 建置指令

```bash
# 首次建置
cmake -B build && cmake --build build

# 執行
./build/step1_cpp

# 測試
ctest --test-dir build --output-on-failure

# 只跑測試
cmake --build build --target test_smoke test_rom_info test_file_rom_loader test_cpu_bus test_mapper000 && ctest --test-dir build --output-on-failure
```
