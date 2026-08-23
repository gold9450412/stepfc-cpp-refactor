---
layout: default
title: Stage 3：Mapper000 NROM (mapper000.h / mapper000.cpp)
---

# Stage 3 日誌：Mapper000 NROM (mapper000.h / mapper000.cpp)

## 日期
2026-06-29

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/mapper000.h` ✅
- 建立 `src/nes/mapper000.cpp` ✅
- `class Mapper000 : public Mapper` ✅
- 建構子接收 `const RomInfo&`（取 PRG-ROM 資料指標和大小）✅
- 實作 `read_prg(addr)`：16KB 用 `addr % 16384`，32KB 直接映射 ✅
- 實作 `write_prg(addr, data)`：ROM 唯讀，空實作 ✅
- private 成員：`const uint8_t* prg_data_` + `uint32_t prg_size_` ✅
- 更新 CMakeLists.txt（加 `mapper000.cpp`）✅
- Build 通過 ✅

## 最終程式碼

### mapper000.h（21行）
```cpp
#pragma once

#include <cstdint>

#include "mapper.h"
#include "rom_info.h"

namespace nes {
class Mapper000 : public Mapper {
public:
    explicit Mapper000(const RomInfo& rom);

    uint8_t read_prg(uint16_t addr) const override;
    void write_prg(uint16_t addr, uint8_t data) override;   

private:
    const uint8_t* prg_data_;
    uint32_t prg_size_;
};

} // namespace nes
```

### mapper000.cpp（25行）
```cpp
#include "mapper000.h"

namespace nes {
Mapper000::Mapper000(const RomInfo& rom)
    : prg_data_(rom.prg_rom().data())
    , prg_size_(static_cast<uint32_t>(rom.prg_rom().size())) 
    {
    }

uint8_t Mapper000::read_prg(uint16_t addr) const {
    if (prg_size_ <= 16384) {
        // 16KB: 鏡像
        return prg_data_[addr % 16384];
    }
    // 32KB: 直接映射
    return prg_data_[addr];
}

void Mapper000::write_prg(uint16_t addr, uint8_t data) {
    // ROM 唯讀，忽略寫入
    (void)addr;
    (void)data;
}

} // namespace nes
```

### Build 結果
- 編譯成功 ✅
- 無 warning（用 `(void)` 消除 unused parameter warning）

## 討論重點

### 1. NROM = Nintendo ROM，Mapper number 0
- NES 卡帶最簡單的電路板設計，ROM 直接焊上去，沒有映射晶片
- NROM-128: 16KB PRG（鏡像填滿 32KB），NROM-256: 32KB PRG（直接映射）
- 例子：Super Mario Bros. (NROM-128), Donkey Kong
- write_prg() 空實作 = NROM 沒有 bank register 可寫

### 2. Bank = 切換視窗
- NROM（沒 bank）：窗戶固定，ROM 直接看
- MMC1（有 bank）：窗戶可滑動，256KB ROM 切成 16 個 16KB bank
- 原版 C 的 `prg_banks[8]` = 8 個 8KB 窗口指標，NROM 固定不改

### 3. 鏡像填滿
- CPU 位址空間 $8000-$FFFF 有 32KB，ROM 只有 16KB
- 16KB ROM 重複讀一遍：$8000-$BFFF=前半，$C000-$FFFF=鏡像（重複前半）
- `addr % 16384` 讓 $4000-$7FFF 繞回 $0000-$3FFF
- 為什麼要填滿：中斷向量在 $FFFA-$FFFF，ROM 必須填到 $FFFF CPU 才讀得到

### 4. PRG-ROM 從 $8000 開始，不是 $6000
- $6000-$7FFF 是 SRAM（PRG-RAM），只有 Family BASIC 等少數卡帶有
- $8000-$BFFF 才是 PRG-ROM 前 16KB
- nesdev wiki NROM 頁面：`CPU $8000-$BFFF: First 16 KiB of PRG-ROM`
- Mapper000 的 read_prg 處理 $8000-$FFFF 的 PRG-ROM，不碰 $6000-$7FFF 的 SRAM

### 5. `explicit` 防隱式轉換
- 沒 explicit：`use_mapper(info)` 偷偷建臨時 Mapper000(info)
- 有 explicit：必須明確寫 `use_mapper(Mapper000(info))`

### 6. `prg_size_` 為什麼用 `uint32_t`
- iNES header count 是 uint8_t (0-255)，PRG 最大 255×16KB≈4MB
- uint16_t 也夠，uint32_t 為了擴展性和業界慣例

### 7. `prg_data_` 用 raw pointer 不用 vector
- Mapper000 不擁有 PRG-ROM 資料，只是「看」它
- 資料生命週期由 RomInfo 管理（vector），Mapper000 拿指標來讀
- 業界原則：擁有權用 unique_ptr/shared_ptr，非擁有的存取用 raw pointer 或 reference

### 8. `read_prg` 是 const，`write_prg` 不是 const
- `read_prg` 只讀不改物件 → const 方法
- `write_prg` 雖然 NROM 是空實作，但其他 Mapper（MMC1 等）寫入會改 bank register → 非 const

### 9. `(void)` vs `[[maybe_unused]]` — 消除 unused parameter warning
- `write_prg` 的參數刻意不用（ROM 唯讀），但 `-Wall -Wextra` 會警告
- 兩種消除方法：

#### 方法 1：`(void)addr; (void)data;`（C 慣用寫法）
- `(void)addr;` 是在「用」這個參數 — 把它轉型成 void 然後丟掉
- 編譯器看到「用了」就不警告
- 為什麼能 build 過：`(void)addr;` 是合法的表達式語句，評估 addr → 轉 void → 丟掉，執行期零成本
- C 語言流傳下來的慣用寫法，更通用（C/C++ 都能用）

#### 方法 2：`[[maybe_unused]]`（C++17 屬性）
- 在 .h 宣告加屬性：`void write_prg([[maybe_unused]] uint16_t addr, [[maybe_unused]] uint8_t data) override;`
- 原理不同：不是「用」參數，而是直接告訴編譯器「我知道可能不用，別警告」
- C++17 引入，語意更明確

#### 兩者比較
| | `(void)` | `[[maybe_unused]]` |
|---|---|---|
| 原理 | 假裝在用參數 | 明確宣告可能不用 |
| 位置 | .cpp 實作裡 | .h 宣告裡 |
| 語言 | C/C++ 通用 | C++17 專有 |
| 語意 | 間接（騙編譯器） | 直接（明說意圖） |

- 學員最終選擇用 `(void)` 方法

### 10. 原版 C vs 新版 C++ 對比
| 原版 C | 新版 C++ | 差異 |
|--------|---------|------|
| `prg_banks[8]` 裸指標陣列 | `prg_data_` + `prg_size_` | 簡化：NROM 不需要 8 個 bank |
| `sfc_load_prgrom_8k` 設定 bank 指標 | 建構子直接存指標+大小 | 一步到位 |
| `id2 = count_prgrom16kb & 2` 判斷鏡像 | `prg_size_ <= 16384` 判斷 | 語意更清楚 |
| `assert(!"NOT IMPL")` write | 空實作 + `(void)` | 不會在 release 失效 |

## Review 建議
- 程式碼正確，邏輯清晰
- `read_prg` 的 16KB/32KB 分支判斷正確
- `write_prg` 空實作符合 NROM 規格（沒有 bank register）
- `(void)` 消除 warning 的做法正確
- raw pointer 使用正確：不擁有資料，只讀取

## 學習心得
Stage 3 的核心是實作第一個具體 Mapper。NROM 是最簡單的 Mapper，沒有 bank 切換邏輯，只有 16KB 鏡像和 32KB 直接映射兩種模式。`addr % 16384` 的鏡像技巧讓 16KB ROM 填滿 32KB 位址空間，確保中斷向量 $FFFA-$FFFF 讀得到。`(void)` 消除 warning 的技巧也很實用 — ROM 唯讀的 `write_prg` 刻意不用參數，用 `(void)` 告訴編譯器「我故意的」。raw pointer + 不擁有的設計原則也在這階段實踐：Mapper000 拿 RomInfo 的指標來讀，不負責釋放。
