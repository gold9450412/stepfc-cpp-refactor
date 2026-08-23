---
layout: default
title: Stage 11：Mapper000 測試 (tests/test_mapper000.cpp)
---

# Stage 11 日誌：Mapper000 測試 (tests/test_mapper000.cpp)

## 日期
2026-08-23

## 狀態
✅ 完成

## 完成事項
- 建立 `tests/test_mapper000.cpp` ✅
- TEST 1：16KB PRG 鏡像驗證 ✅
- TEST 2：32KB PRG 直接映射驗證 ✅
- TEST 3：write_prg 唯讀驗證 ✅
- 更新 `tests/CMakeLists.txt` ✅
- 15 個測試全部通過（舊 12 + 新 3）✅

## 最終程式碼

### tests/test_mapper000.cpp（77行）
```cpp
#include <gtest/gtest.h>
#include "nes/mapper000.h"
#include "nes/rom_info.h"

#include <vector>
#include <cstdint>

TEST(Mapper000Test, Read16KBMirror) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 1;
    header.chr_rom_count = 1;
    header.flags6 = 0;
    header.flags7 = 0;
    header.reserved = {};

    std::vector<uint8_t> prg(16384);
    prg[0] = 0xAB;
    prg[1] = 0xCD;
    std::vector<uint8_t> chr(8192);

    nes::RomInfo info(header, std::move(prg), std::move(chr));
    nes::Mapper000 mapper(info);

    EXPECT_EQ(mapper.read_prg(0x0000), 0xAB);
    EXPECT_EQ(mapper.read_prg(0x0000 + 1), 0xCD);
    EXPECT_EQ(mapper.read_prg(0x4000), 0xAB);
    EXPECT_EQ(mapper.read_prg(0x4000 + 1), 0xCD);
}

TEST(Mapper000Test, Read32KBDirect) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 2;
    header.chr_rom_count = 1;
    header.flags6 = 0;
    header.flags7 = 0;
    header.reserved = {};

    std::vector<uint8_t> prg(32768);
    prg[0] = 0xAB;
    prg[1] = 0xCD;
    prg[16384] = 0x12;
    prg[16385] = 0x34;
    std::vector<uint8_t> chr(8192);

    nes::RomInfo info(header, std::move(prg), std::move(chr));
    nes::Mapper000 mapper(info);

    EXPECT_EQ(mapper.read_prg(0x0000), 0xAB);
    EXPECT_EQ(mapper.read_prg(0x0000 + 1), 0xCD);
    EXPECT_EQ(mapper.read_prg(0x4000), 0x12);
    EXPECT_EQ(mapper.read_prg(0x4000 + 1), 0x34);
}

TEST(Mapper000Test, WritePrgIgnored) {
    nes::NesHeader header;
    header.magic = {'N', 'E', 'S', 0x1A};
    header.prg_rom_count = 1;
    header.chr_rom_count = 1;
    header.flags6 = 0;
    header.flags7 = 0;
    header.reserved = {};

    std::vector<uint8_t> prg(16384);
    prg[0] = 0xAB;
    std::vector<uint8_t> chr(8192);

    nes::RomInfo info(header, std::move(prg), std::move(chr));
    nes::Mapper000 mapper(info);

    mapper.write_prg(0x0000, 0xFF);
    EXPECT_EQ(mapper.read_prg(0x0000), 0xAB);

    mapper.write_prg(0x4000, 0xFF);
    EXPECT_EQ(mapper.read_prg(0x4000), 0xAB);
}
```

### tests/CMakeLists.txt（32行）
新增兩段：
```cmake
add_executable(test_mapper000 test_mapper000.cpp)
target_link_libraries(test_mapper000 PRIVATE nes_lib gtest_main)

gtest_discover_tests(test_mapper000)
```

### ctest 結果
```
100% tests passed, 0 tests failed out of 15
  13. Mapper000Test.Read16KBMirror   ✅
  14. Mapper000Test.Read32KBDirect   ✅
  15. Mapper000Test.WritePrgIgnored  ✅
```

---

## 討論重點

### 1. 建測試架構：直接建 RomInfo + Mapper000，不走檔案

跟 Stage 10（CpuBus 測試）的模式一樣：

- **不走檔案**：不用 FileRomLoader 讀真正的 .nes，直接在記憶體裡建 RomInfo
- **自製測試資料**：`std::vector<uint8_t> prg(16384)`，在關鍵位置放辨識值（0xAB、0xCD、0x12、0x34）
- **直接建 Mapper000**：不經過 `make_mapper()`，因為要測的是 Mapper000 本身

這樣測試完全獨立於外部檔案，跑得快、結果可控。

### 2. 測試裡的 mapper number 是 0，但其實不重要

TEST 裡寫了：

```cpp
header.flags6 = 0;
header.flags7 = 0;
```

算出來 `mapper_number = (flags6 >> 4) | (flags7 & 0xF0) = 0`。

**但這個數字在測試裡根本沒被用到**，因為：

```cpp
nes::Mapper000 mapper(info);   // 直接建構
```

不是這樣：

```cpp
auto mapper = nes::make_mapper(info);   // 依 mapper_number 選 Mapper
```

`Mapper000` 建構子只看 `rom.prg_rom()` 的資料和大小，從頭到尾不碰 `mapper_number`。所以測試裡 flags6/flags7 寫 0 只是為了讓 header 完整，不影響測試行為。

**區分兩種測試目標**：
- Stage 11 測試「Mapper000 的行為」→ 直接建構
- 如果要測「mapper number 0 會建立 Mapper000」→ 才需要走 `make_mapper()`（這是 Stage 12 的事）

### 3. 16KB 鏡像規則複習

NROM-128（16KB PRG）要填滿 CPU 的 32KB 位址視窗（$8000-$FFFF），做法是**後半鏡像前半**：

| CPU 位址 | Offset（減 $8000）| 對應 ROM 索引 |
|---|---|---|
| $8000 | $0000 | 0 |
| $BFFF | $3FFF | 16383（ROM 最後一個 byte）|
| $C000 | $4000 | **0（鏡像回開頭）**|
| $FFFF | $7FFF | **16383（鏡像）**|

數學式：`rom_index = addr % 16384`

所以測試驗證：
```cpp
EXPECT_EQ(mapper.read_prg(0x0000), 0xAB);  // 前半
EXPECT_EQ(mapper.read_prg(0x4000), 0xAB);  // 後半鏡像，同一個 byte
```

### 4. 32KB 直接映射規則複習

NROM-256（32KB PRG）剛好填滿 32KB，不用鏡像：

| CPU 位址 | Offset | 對應 ROM 索引 |
|---|---|---|
| $8000 | $0000 | 0（前半）|
| $BFFF | $3FFF | 16383 |
| $C000 | $4000 | **16384（後半真正的資料）**|
| $FFFF | $7FFF | 32767 |

測試在前半和後半放不同值，確保兩邊讀到不同資料：

```cpp
prg[0] = 0xAB;       // 前半
prg[16384] = 0x12;   // 後半

EXPECT_EQ(mapper.read_prg(0x0000), 0xAB);
EXPECT_EQ(mapper.read_prg(0x4000), 0x12);  // 跟 16KB 模式的關鍵差異
```

### 5. 測試怎麼指定 16KB 還是 32KB？看 vector 大小，不是 header

TEST 1 用了兩個地方寫 16KB：

```cpp
header.prg_rom_count = 1;          // metadata 說 1 × 16KB
std::vector<uint8_t> prg(16384);  // 實際資料 16384 bytes
```

**重點：Mapper000 只看 vector 大小**：

```cpp
// mapper000.cpp 建構子
Mapper000::Mapper000(const RomInfo& rom)
    : prg_data_(rom.prg_rom().data())
    , prg_size_(static_cast<uint32_t>(rom.prg_rom().size()))  // ← 這裡
```

分支判斷：
```cpp
if (prg_size_ <= 16384) {
    return prg_data_[addr % 16384];  // 16KB 鏡像
}
return prg_data_[addr];              // 32KB 直接映射
```

所以就算 header 寫 `prg_rom_count = 2` 但 vector 只有 16384 bytes，還是走鏡像分支。**資料本身才是事實**。

### 6. 唯讀行為是誰負責的？兩層保護

TEST 3 驗證「寫入被忽略」，背後有兩層設計：

**第一層：write_prg 空實作（被測試的對象）**

```cpp
// mapper000.cpp
void Mapper000::write_prg(uint16_t addr, uint8_t data) {
    (void)addr;   // 假裝有用，消 unused warning
    (void)data;
    // 什麼都不做，寫入直接被忽略
}
```

**第二層：prg_data_ 是 const 指標（編譯器保證）**

```cpp
// mapper000.h
const uint8_t* prg_data_;
```

就算有人在 write_prg 裡寫 `prg_data_[addr] = data;`，編譯器直接報錯（不能寫 const 資料）。

**測試驗證第一層**：呼叫 `write_prg(0x0000, 0xFF)` 後再讀，值還是 `0xAB`，證明寫入真的被忽略。

### 7. 測試值的選擇（為什麼用 0xAB / 0xCD / 0x12 / 0x34 / 0xFF）

- **不用 0x00**：vector 預設值是 0，分不出「資料原本就是 0」還是「測試值沒寫進去」
- **不用 0xFF 做讀取驗證**：0xFF 容易跟未初始化/錯誤值搞混（TEST 3 用 0xFF 當「嘗試寫入的值」，正因為它顯眼，寫前有放 0xAB 對照）
- **每個關鍵位置不同值**：前半 0xAB/0xCD、後半 0x12/0x34，組合起來可驗證有沒有讀錯位置
- **連讀兩個 byte**：+1 位址也測，防止「只驗一個 byte 剛好矇對」

### 8. 踩到的坑：cmake 跑錯目錄

在 `step1/` 跑 `cmake -B build` 會報錯：

```
CMake Error: The source directory "/home/tony/stepfc/StepFC/step1" does not appear to contain CMakeLists.txt.
```

因為 CMakeLists.txt 在 `step1/cpp/` 裡。正確做法：

```bash
cd step1/cpp
cmake -B build && cmake --build build && ctest --test-dir build --output-on-failure
```

## 遇到的問題
- 無程式碼問題，3 個 TEST 一次寫對
- 只有 cmake 執行目錄錯誤（環境操作問題）

## Review 建議
- 3 個 TEST 覆蓋了 NROM 的三個核心行為：16KB 鏡像、32KB 直接映射、唯讀
- 測試資料設計良好（不同位置不同值）
- 每個 TEST 都是獨立場景，無相依性

## 學習心得
Stage 11 的核心學到：

1. **直接建構 vs Factory**：測單一類別行為用直接建構；測 type dispatch 才用 make_mapper
2. **鏡像的測試方法**：前後半放相同/不同值，讀 offset 0x0000 vs 0x4000 比對
3. **資料來源的真相**：Mapper000 的行為由 `prg_rom().size()` 決定，不是 header 的欄位
4. **唯讀的雙重保障**：行為上（write_prg 空）+ 型別上（const 指標）都擋了
5. **測試值的挑選**：避開 0x00 和 0xFF 這類「跟預設/錯誤狀態撞衫」的值
