---
layout: default
title: Stage 12：整合測試 + 收尾 (tests/test_integration.cpp)
---

# Stage 12 日誌：整合測試 + 收尾 (tests/test_integration.cpp)

## 日期
2026-08-23

## 狀態
✅ 完成

## 完成事項
- 建立 `tests/test_integration.cpp` ✅
- TEST 1：讀取 nestest.nes 真實檔案，驗證三個中斷向量 ✅
- TEST 2：make_mapper 對未知 mapper 回傳 nullptr ✅
- 更新 `tests/CMakeLists.txt` ✅
- 修正 SEGFAULT（WORKING_DIRECTORY 缺失）✅
- 17 個測試全部通過 ✅

## 最終程式碼

### tests/test_integration.cpp（43行）
```cpp
#include <gtest/gtest.h>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"
#include "nes/famicom.h"
#include "nes/cpu_vectors.h"
#include "nes/mapper_factory.h"

TEST(IntegrationTest, ReadNestestVectors) {
    nes::RomInfo info;
    nes::FileRomLoader loader("nestest.nes");
    ASSERT_EQ(loader.load(info), nes::ErrorCode::Ok);

    nes::Famicom famicom(std::move(info));

    uint16_t nmi = famicom.cpu_read(nes::kNmiVector)
                 | (famicom.cpu_read(nes::kNmiVector + 1) << 8);
    uint16_t reset = famicom.cpu_read(nes::kResetVector)
                   | (famicom.cpu_read(nes::kResetVector + 1) << 8);
    uint16_t irq = famicom.cpu_read(nes::kIrqVector)
                 | (famicom.cpu_read(nes::kIrqVector + 1) << 8);

    EXPECT_EQ(nmi, 0xC5AF);
    EXPECT_EQ(reset, 0xC004);
    EXPECT_EQ(irq, 0xC5F4);
}

TEST(IntegrationTest, MakeMapperUnknownReturnsNull) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 1;
    header.chr_rom_count = 1;
    header.flags6 = 0x10;  // mapper number = (0x10 >> 4) = 1
    header.flags7 = 0;
    header.reserved = {};

    std::vector<uint8_t> prg(16384);
    std::vector<uint8_t> chr(8192);

    nes::RomInfo info(header, std::move(prg), std::move(chr));
    auto mapper = nes::make_mapper(info);

    EXPECT_EQ(mapper, nullptr);
}
```

### tests/CMakeLists.txt（36行）最終版
新增兩段：
```cmake
add_executable(test_integration test_integration.cpp)
target_link_libraries(test_integration PRIVATE nes_lib gtest_main)

gtest_discover_tests(test_integration
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
```

### ctest 結果
```
100% tests passed, 0 tests failed out of 17
  16. IntegrationTest.ReadNestestVectors          ✅
  17. IntegrationTest.MakeMapperUnknownReturnsNull ✅
```

---

## 討論重點

### 1. 整合測試跟單元測試的差異

| | 單元測試（Stage 10/11）| 整合測試（Stage 12）|
|---|---|---|
| 測試對象 | 單一類別（CpuBus、Mapper000）| 多類別串起來的完整流程 |
| 測試資料 | 自製假資料（vector + 特定值）| 真實檔案（nestest.nes）|
| 依賴 | FakeMapper 隔離 | 真實 FileRomLoader + Mapper000 |
| 目的 | 驗證局部邏輯 | 驗證整條鏈路能跑通 |

整合測試的價值：**單元測試全過不代表串起來能用**。中間的接縫（Loader → RomInfo → Famicom → CpuBus → Mapper）只有整合測試會走到。

### 2. integral test 放哪的決策

考慮過兩個選項：
- 選項 A：新建 `test_integration.cpp`（採用）
- 選項 B：加到 `test_file_rom_loader.cpp`

選 A 的理由：職責分離。`test_file_rom_loader.cpp` 只測「讀檔案」；`test_integration.cpp` 測「整條鏈路」。以後要加更多端到端測試（例如 CPU 執行第一條指令）時有地方放。

### 3. ASSERT_EQ vs EXPECT_EQ — 什麼時候要用 ASSERT

原本 TEST 1 用：
```cpp
EXPECT_EQ(loader.load(info), nes::ErrorCode::Ok);
```

結果 SEGFAULT。改成：
```cpp
ASSERT_EQ(loader.load(info), nes::ErrorCode::Ok);
```

**差異**：

| | EXPECT_EQ | ASSERT_EQ |
|---|---|---|
| 失敗行為 | 記下失敗，**繼續往下跑** | 失敗立刻**停止這個 TEST** |
| 適用情境 | 後面檢查不依賴前面結果 | 後面依賴前面，失敗繼續跑會 crash |

這裡 load 失敗 → info 是空的 → Famicom 建好→ cpu_read 崩潰。所以必須用 ASSERT_EQ 擋下來。

**記法**：後面步驟「沒有前面就不能跑」→ 用 ASSERT；只是「記錄多一個失敗點」→ 用 EXPECT。

### 4. 重大 Bug：SEGFAULT 除錯過程

**症狀**：
```
16/17 Test #16: IntegrationTest.ReadNestestVectors .....   SEGFAULT
```

**除錯思路，一步步推**：

1. SEGFAULT = 存取了不該存的記憶體（通常是 nullptr dereference）
2. 測試裡可能空指標的地方：`make_mapper` 回傳 nullptr？看起來 nestest.nes 是 mapper 0，應該有 Mapper000
3. 那是哪裡炸？`famicom.cpu_read(...)` → `bus_.read(addr)` → `mapper_->read_prg(addr - 0x8000)` → `prg_data_[...]`
4. 如果 `prg_data_` 是 nullptr → 就 SEGFAULT
5. `prg_data_` 哪來的？`rom.prg_rom().data()`。如果 `prg_rom()` 是空 vector，`data()` 回傳 nullptr
6. 為什麼 RomInfo 是空的？因為 `loader.load(info)` 失敗了
7. 為什麼失敗？因為 ctest 預設在 `build/` 目錄執行，找不到相對路徑的 `"nestest.nes"`

**根本原因**：

`tests/CMakeLists.txt` 裡 `test_file_rom_loader` 有 `WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}`，但新增的 `test_integration` **忘了加**。

**修法**：
```cmake
gtest_discover_tests(test_integration
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
```

**學到的**：
- ctest 的每個測試可有自己的工作目錄，預設是 build 目錄
- 需要讀相對路徑檔案的測試都必須設 `WORKING_DIRECTORY`
- Stage 12 的 defect 其實是「照抄 Stage 10/11 但忘了 Stage 10 有 WORKING_DIRECTORY 的教訓」

### 5. 「CPU 位址的載入」實作到哪了

#### 回顧 `CpuBus::read()` 的現況

| 位址範圍 | 狀態 | 來源 |
|---|---|---|
| $0000-$1FFF RAM | ✅ 完整實作 | `ram_[addr & 0x07FF]`，含鏡像 |
| $2000-$3FFF PPU | ⚠️ stub 回傳 0 | PPU 要等 step2+ |
| $4000-$401F APU | ⚠️ stub 回傳 0 | APU 要等 step2+ |
| $4020-$5FFF Expansion | ⚠️ stub 回傳 0 | 暫不用 |
| $6000-$7FFF SRAM | ✅ 完整實作 | `sram_[addr - 0x6000]` |
| $8000-$FFFF PRG-ROM | ✅ 完整實作 | `mapper_->read_prg(addr - 0x8000)` |

Step1 的範圍本來就是「基本讀寫 + Mapper000 + 中斷向量」，PPU/APU 是後面的事。

### 6. CPU 位址空間跟 .nes header 沒關係

兩個不同層級的東西：

| | iNES header | CPU 位址空間 |
|---|---|---|
| 大小 | 16 bytes | 64 KB |
| 內容 | PRG 大小、mapper number 等 metadata | RAM、PPU、APU、SRAM、PRG-ROM |
| Wiki | https://www.nesdev.org/wiki/INES | https://www.nesdev.org/wiki/CPU_memory_map |

流程：讀 header → 抓 PRG/CHR → **header 丟掉** → 只有 PRG 透過 Mapper 進 CPU 位址空間。CpuBus 永遠不會看到 header 內容。

### 7. ram_ 現在是空的

```cpp
std::array<uint8_t, 2048> ram_{};
```

`{}` 會初始化成全 0，而且沒有 CPU 在跑所以沒人寫入。這不影響中斷向量測試，因為向量在 PRG-ROM 區，不走 RAM。

等以後 step2 開始有 CPU 執行指令，`STA $0200` 之類的寫入才會讓 RAM 有內容。

### 8. 整合測試的向量值是哪來的

不是 Wiki 給的。Wiki 只說向量**位置**（$FFFA-$FFFF），**值**是每個 ROM 自己定的。

`nestest.nes` 的 PRG-ROM 最後 6 bytes：
```
$FFFA: AF → NMI 低 byte（合起 $C5AF）
$FFFB: C5 → NMI 高 byte
$FFFC: 04 → RESET 低 byte（合起 $C004）
$FFFD: C0 → RESET 高 byte
$FFFE: F4 → IRQ 低 byte（合起 $C5F4）
$FFFF: C5 → IRQ 高 byte
```

等等：為什麼 CPU 位址 $FFFA 對應 PRG-ROM 的最後幾個 byte？

因為 nestest.nes 的 PRG 是 16KB，NROM-128 模式鏡像填滿 $8000-$FFFF。CPU $FFFA = offset $7FFA（addr - 0x8000），Mapper000 做 `% 16384` 後 = 16378，就是 PRG-ROM 的倒數第 6 byte。

這就是為什麼之前在 Stage 9 能用 `xxd` 直接驗證 PRG-ROM 的尾端。

### 9. make_mapper 對未知 mapper 回傳 nullptr

TEST 2 用 `flags6 = 0x10`：

```
mapper_number = (flags6 >> 4) | (flags7 & 0xF0)
              = (0x10 >> 4)   | (0x00 & 0xF0)
              = 1 | 0
              = 1
```

Mapper 1 = MMC1，我們還沒實作。`make_mapper` 的 switch 只有 `case 0`，所以走到 `default: return nullptr;`。

這個測試保護未來：新增 Mapper 時要記得加 case；如果加了 case 應該改測別的 mapper number，如果忘了加新 case 測試會抓到。

## 遇到的問題

### SEGFAULT 崩潰（已解決）

**症狀**：`IntegrationTest.ReadNestestVectors` 直接當機，無測試輸出。
**原因**：test_integration 沒設 `WORKING_DIRECTORY`，找不到 nestest.nes → load 失敗 → Famicom 用空 RomInfo → read_prg 解引用 nullptr
**修法**：CMakeLists.txt 加 `WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}` + test 改用 `ASSERT_EQ`
**教訓**：複製其他測試的設定時，要注意 WORKING_DIRECTORY 這種「不是每個 target 都需要但少了會炸」的屬性

## Review 建議
- 整合測試獨立成一個檔案是正確的，結構清晰
- ASSERT_EQ 的使用時機正確
- 兩個 TEST 一個測「成功路徑」（向量讀取），一個測「失敗路徑」（nullptr），覆蓋合理

## 學習心得

Stage 12 的核心學到：

1. **整合測試的價值**：單元測試全過不代表串起來能用。整合測試驗證接縫
2. **ASSERT vs EXPECT**：後面依賴前面 → ASSERT。這個差序不是在測什麼，是在「失敗後能不能繼續跑」
3. **ctest 工作目錄**：預設在 build/，需要讀相對路徑檔案的測試要設 WORKING_DIRECTORY
4. **除錯 SEGFAULT 的思路**：从「哪一行程式碰到 nullptr」往回推資料流
5. **Header vs 記憶體空間是兩回事**：檔案格式（header）只是告訴模擬器怎麼拆分資料，CPU 位址空間只接受拆分後的 PRG-ROM
6. **養成習慣**：下次新增讀檔測試，先想到 WORKING_DIRECTORY

## 結語

Step1 全部 13 個 Stage 完成。從 Step0 的「讀 .nes 檔案」走到 Step1 的「CPU 能看到整個 64KB 空間 + 讀到中斷向量」。下一個 Step 開始就要實作 6502 CPU 執行指令了。
