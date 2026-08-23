---
layout: default
title: Stage 10：CpuBus 測試 (tests/test_cpu_bus.cpp)
---

# Stage 10 日誌：CpuBus 測試 (tests/test_cpu_bus.cpp)

## 日期
2026-07-13

## 狀態
✅ 完成

## 完成事項
- 建立 tests/test_cpu_bus.cpp ✅
- 定義 FakeMapper（測試用假 Mapper，回傳固定值 0x42）✅
- 5 個 TEST case 全部通過 ✅
- 更新 tests/CMakeLists.txt ✅
- 12 個測試全部通過（舊 7 + 新 5）✅

## 最終程式碼

### test_cpu_bus.cpp（63行）
```cpp
#include <gtest/gtest.h>
#include "nes/cpu_bus.h"
#include "nes/mapper.h"

namespace {
class FakeMapper : public nes::Mapper {
public:
    uint8_t read_prg(uint16_t addr) const override {
        (void)addr;
        return 0x42;
    }
    void write_prg(uint16_t addr, uint8_t data) override {
        (void)addr;
        (void)data;
    };
} // namespace

TEST(CpuBusTest, RamMirroring) {
    FakeMapper mapper;
    nes::CpuBus bus(&mapper);
    bus.write(0x0000, 0xAB);
    EXPECT_EQ(bus.read(0x0800), 0xAB);
    EXPECT_EQ(bus.read(0x1000), 0xAB);
    EXPECT_EQ(bus.read(0x1800), 0xAB);
}

TEST(CpuBusTest, RamBoundary) {
    FakeMapper mapper;
    nes::CpuBus bus(&mapper);
    bus.write(0x07FF, 0xCD);
    EXPECT_EQ(bus.read(0x07FF), 0xCD);
    EXPECT_EQ(bus.read(0x0FFF), 0xCD);
    EXPECT_EQ(bus.read(0x17FF), 0xCD);
    EXPECT_EQ(bus.read(0x1FFF), 0xCD);
}

TEST(CpuBusTest, SramReadWrite) {
    FakeMapper mapper;
    nes::CpuBus bus(&mapper);
    bus.write(0x6000, 0x12);
    EXPECT_EQ(bus.read(0x6000), 0x12);
    bus.write(0x7FFF, 0x34);
    EXPECT_EQ(bus.read(0x7FFF), 0x34);
}

TEST(CpuBusTest, PpuApuStubReturnsZero) {
    FakeMapper mapper;
    nes::CpuBus bus(&mapper);
    EXPECT_EQ(bus.read(0x2000), 0);
    EXPECT_EQ(bus.read(0x3FFF), 0);
    EXPECT_EQ(bus.read(0x4000), 0);
    EXPECT_EQ(bus.read(0x401F), 0);
}

TEST(CpuBusTest, PrgRomThroughMapper) {
    FakeMapper mapper;
    nes::CpuBus bus(&mapper);
    EXPECT_EQ(bus.read(0x8000), 0x42);
    EXPECT_EQ(bus.read(0xBFFF), 0x42);
    EXPECT_EQ(bus.read(0xC000), 0x42);
    EXPECT_EQ(bus.read(0xFFFF), 0x42);
}
```

### tests/CMakeLists.txt 更新（28行）
- 新增 test_cpu_bus target（第 20-21 行）
- 新增 gtest_discover_tests(test_cpu_bus)（第 28 行）

### ctest 結果
```
100% tests passed, 0 tests failed out of 12
  1. SmokeTest.BasicAssertion              ✅
  2. RomInfoTest.DefaultConstructorIsNotLoaded  ✅
  3. RomInfoTest.GettersReturnCorrectValues      ✅
  4. RomInfoTest.ConstInterfaceWorks        ✅
  5. FileRomLoaderTest.LoadValidNesFile     ✅
  6. FileRomLoaderTest.LoadNonexistentFile  ✅
  7. FileRomLoaderTest.LoadIllegalFile      ✅
  8. CpuBusTest.RamMirroring                ✅
  9. CpuBusTest.RamBoundary                 ✅
 10. CpuBusTest.SramReadWrite              ✅
 11. CpuBusTest.PpuApuStubReturnsZero      ✅
 12. CpuBusTest.PrgRomThroughMapper        ✅
```

## 討論重點

### 1. 匿名 namespace vs 命名 namespace
- `namespace nes { }` = 有名字的 namespace，東西放進 `nes` namespace
- `namespace { }` = 匿名 namespace，東西只在這個檔案可見
- FakeMapper 是測試專用的假物件，不該跟正式的 `nes::Mapper000` 混在一起，所以用匿名 namespace 把它關在這個檔案裡
- 註解要對應：匿名 namespace 的註解寫 `} // namespace`（不帶名字）

### 2. #include 放在 namespace 裡面的問題
- `#include` 是前處理器指令，應該放在檔案最前面
- 放在 namespace 裡面會把引入的標頭檔內容包進匿名 namespace，可能造成問題
- 規則：`#include` 一律放在檔案頂部，namespace 裡只放 class/function 定義

### 3. FakeMapper 為什麼回傳固定值 0x42
- 一眼認出：看到 0x42 就知道是 FakeMapper 回傳的
- 不是 0x00：如果回傳 0，測試分不清「Mapper 回傳 0」還是「CpuBus 走到 stub 回傳 0」
- 不是 0xFF：0xFF 是常見初始值或錯誤值，容易混淆
- 業界慣例：0x42 是程式圈常見測試值（出自 The Hitchhiker's Guide to the Galaxy，42 是「生命、宇宙及一切的終極答案」）

### 4. 測試裡不用 unique_ptr，Famicom 要用
| | 測試 | Famicom |
|---|---|---|
| 知道具體型別 | ✅ 知道是 FakeMapper | ❌ 不知道（執行期決定） |
| 需要多型 | ❌ 不需要 | ✅ 需要 |
| 放哪 | stack 就好 | 必須 heap（new 子類） |
| 用什麼 | 直接建構 | unique_ptr |

- 測試裡明確知道用 FakeMapper，直接 stack 建構最簡單
- Famicom 要執行期才決定用哪個 Mapper，必須多型 → 必須 heap → 必須 unique_ptr
- Mapper 是抽象類別不能直接建立物件，要用多型就必須 new 子類用父類指標接

### 5. RAM 鏡像邊界 — $07FF 和 $0800 不是同一個 byte
- `$07FF & 0x07FF = $07FF`（RAM 第一份最後一個 byte）
- `$0800 & 0x07FF = $0000`（RAM 第二份第一個 byte，映射到 $0000）
- 所以 $07FF 和 $0800 **不是同一個 byte**
- $07FF 的正確鏡像位址：$0FFF、$17FF、$1FFF（全部 & 0x07FF = $07FF）
- SPEC.md 原本寫「$07FF 和 $0800 是同一個 byte」是錯的，需要修正

### 6. 5 個 TEST case 驗證什麼

| TEST | 驗證內容 |
|------|---------|
| RamMirroring | 寫 $0000 讀鏡像 $0800/$1000/$1800 應該一樣 |
| RamBoundary | 寫 $07FF 讀鏡像 $0FFF/$17FF/$1FFF 應該一樣 |
| SramReadWrite | SRAM $6000/$7FFF 讀寫正確 |
| PpuApuStubReturnsZero | PPU $2000-$3FFF + APU $4000-$401F stub 回傳 0 |
| PrgRomThroughMapper | PRG-ROM $8000-$FFFF 走 Mapper 回傳 0x42 |

## 學習心得
Stage 10 是 CpuBus 的單元測試。核心技巧是用 FakeMapper（mock/stub 模式）隔離 CpuBus 的測試——假 Mapper 回傳固定值 0x42，讓測試專注驗證 CpuBus 的位址分派邏輯。匿名 namespace 把 FakeMapper 關在測試檔裡不外露。測試裡不用 unique_ptr 因為明確知道用 FakeMapper 不需要多型，Famicom 要用因為執行期才決定用哪個 Mapper。RAM 鏡像邊界測試要注意 $07FF 和 $0800 不是同一個 byte（& 0x07FF 結果不同），正確鏡像是 $0FFF/$17FF/$1FFF。
