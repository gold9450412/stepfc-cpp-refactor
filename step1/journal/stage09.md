---
layout: default
title: Stage 9：main.cpp — 讀中斷向量
---

# Stage 9 日誌：main.cpp — 讀中斷向量

## 日期
2026-07-12

## 狀態
✅ 完成

## 完成事項
- main.cpp 加 `#include "nes/cpu_vectors.h"` ✅
- 標題改為 `Step1 C++ - CPU Memory & Mapper` ✅
- 讀三個中斷向量（NMI / RESET / IRQ）✅
- 印出三個向量位址 ✅
- Build 通過，輸出正確 ✅

## 最終程式碼

### main.cpp（32行）
```cpp
#include <iostream>
#include <memory>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"
#include "nes/famicom.h"
#include "nes/cpu_vectors.h"

int main() {
    nes::RomInfo info;
    std::cout << "Step1 C++ - CPU Memory & Mapper" << std::endl;
    
    std::unique_ptr<nes::RomLoader> loader = std::make_unique<nes::FileRomLoader>("nestest.nes");
    auto result = loader->load(info);
    
    if (result == nes::ErrorCode::Ok) {
        nes::Famicom famicom(std::move(info));
        const auto& rom = famicom.get_rom_info();
        std::cout << "ROM: PRG-ROM: " << static_cast<int>(rom.header().prg_rom_count) << " x 16kb" << "    CHR-ROM: " << static_cast<int>(rom.header().chr_rom_count) << " x 8kb" << "    Mapper: " << static_cast<int>(rom.mapper_number()) << std::endl;
        std::cout << "Mirroring: " << (rom.mirroring() ? "Vertical" : "Horizontal") << "    Save RAM: " << (rom.has_save_ram() ? "Yes" : "No") << "    Four Screen: " << (rom.four_screen() ? "Yes" : "No") << std::endl;

        uint16_t nmi = famicom.cpu_read(nes::kNmiVector) | (famicom.cpu_read(nes::kNmiVector + 1) << 8);
        uint16_t reset = famicom.cpu_read(nes::kResetVector) | (famicom.cpu_read(nes::kResetVector + 1) << 8);
        uint16_t irq = famicom.cpu_read(nes::kIrqVector) | (famicom.cpu_read(nes::kIrqVector + 1) << 8);
        std::cout << "NMI Vector:   $" << std::hex << std::uppercase << nmi << std::endl;
        std::cout << "RESET Vector: $" << reset << std::endl;
        std::cout << "IRQ Vector:   $" << irq << std::endl;
    } else {
        std::cout << "Load failed" << std::endl;
    }

    return 0;
}
```

### 實際輸出（nestest.nes）
```
Step1 C++ - CPU Memory & Mapper
ROM: PRG-ROM: 1 x 16kb    CHR-ROM: 1 x 8kb    Mapper: 0
Mirroring: Horizontal    Save RAM: No    Four Screen: No
NMI Vector:   $C5AF
RESET Vector: $C004
IRQ Vector:   $C5F4
```

### 驗證
用 `xxd -s $((16 + 16384 - 6)) -l 6 nestest.nes` 讀 PRG-ROM 最後 6 bytes：
```
af c5 04 c0 f4 c5
```
- NMI ($FFFA-B): `af c5` → $C5AF ✓
- RESET ($FFFC-D): `04 c0` → $C004 ✓
- IRQ ($FFFE-F): `f4 c5` → $C5F4 ✓

### SPEC.md 預期輸出勘誤
SPEC.md 原本寫三個向量都是 $C000，這是錯的。nestest.nes 的實際向量是 $C5AF/$C004/$C5F4。已修正 SPEC.md。

## 討論重點

### 1. 為什麼要讀「兩個」byte
- 中斷向量是一個 **16 位元位址**（CPU 要跳去哪裡執行）
- 但 CPU 記憶體每個位置只存 **8 位元（1 byte）**
- 一個 16 位元位址要拆成兩半存：
  - $FFFA 存「低 8 位元」
  - $FFFB 存「高 8 位元」
- 兩個 byte 拼起來才是完整位址，讀的時候也要讀兩次

### 2. Little-endian（小端序）— 低 byte 在前、高 byte 在後
- 6502 CPU 的規矩：**小的放前面，大的放後面**
- 舉例，位址 $C5AF：
  ```
  位址 $FFFA: 存 AF  ← 低 byte（小的）
  位址 $FFFB: 存 C5  ← 高 byte（大的）
  ```
- 記憶體裡排成 `AF C5`，讀出來要拼回 $C5AF

### 3. `<< 8` 和 `|` 的作用
- `cpu_read()` 每次只回傳 1 byte（uint8_t，範圍 0-255）
- 要拼成 16 位元，得把高 byte 推到上面去：
  ```
  低 byte:  0xAF  →  0000 0000 1010 1111
  高 byte:  0xC5  →  0000 0000 1100 0101

  高 byte << 8（往左推 8 位）→  1100 0101 0000 0000
  低 byte 不動                →  0000 0000 1010 1111

  兩者用 | 合併              →  1100 0101 1010 1111
                             = 0xC5AF
  ```
- `<< 8` = 把高 byte 推到上半部
- `|` = 把低 byte 填進下半部

### 4. 為什麼用 `|` 不用 `+`
- `|` 和 `+` 在這裡結果一樣（因為兩邊 bit 不重疊）
- 但業界用 `|`：
  - `|` 是**位元合併**，語意精確：把兩塊 bit 拼起來
  - `+` 是數值相加，讀者要想一下為什麼不會進位
  - 硬體/底層程式慣例就是 `|`

### 5. 為什麼外層有括號
- `<<` 的優先順序比 `|` 高，其實不寫括號也對
- 但寫括號是**給人看的安全習慣**，確保先移位再合併，不靠背誦優先順序表

### 6. nesdev wiki 有沒有說「小端」和「讀 2 byte」— 有
wiki 的 CPU interrupts 頁面用 **PCL / PCH** 術語表達：

tick-by-tick breakdown：
```
 6   A   R  fetch PCL (A = FFFE for IRQ, A = FFFA for NMI), set I flag
 7   A   R  fetch PCH (A = FFFF for IRQ, A = FFFB for NMI)
```
- **PCL** = Program Counter **Low**（低 byte），讀自低位址 $FFFE / $FFFA
- **PCH** = Program Counter **High**（高 byte），讀自高位址 $FFFF / $FFFB

兩個 tick 各讀一個 byte = 讀 2 byte ✅
低 byte 在低位址、高 byte 在高位址 = little-endian ✅

wiki 沒有寫出「little-endian」這個英文字，但用 PCL/PCH + 位址順序表達了同一件事。

還有一句更直接的：
> "read the new program counter from $FFFE and $FFFF, as if `JMP ($FFFE)`"

`JMP ($FFFE)` 是 6502 的間接跳轉指令，行為就是讀 $FFFE 拿低 byte、讀 $FFFF 拿高 byte，組成 16 位元位址跳過去。

### 7. std::hex + std::uppercase 的 sticky 行為
- `std::hex`：讓 cout 以十六進位輸出
- `std::uppercase`：大寫（$C5AF 不是 $c5af）
- 兩者都是 **sticky**（黏著的），設定一次後續都生效
- 所以只有第一行寫 `std::hex << std::uppercase`，後面兩行不用重複

### 8. 為什麼用 Famicom::cpu_read 包一層，不直接用 bus_.read()
- **封裝**：外面不該知道 Famicom 內部有 CpuBus，這是實作細節
- **未來擴展**：Step2+ 加 PPU 後 Famicom 會有 cpu_read / ppu_read，由 Famicom 負責分派
- 比喻：Famicom 是櫃台，CpuBus 是倉庫，客人跟櫃台說「我要讀 $FFFC」，櫃台自己去倉庫拿

### 9. 封裝藏的是「用」不是「看」
- 打開 code 當然看得到 `private: CpuBus bus_;`，但看到了用不到
- `famicom.bus_.reset()` → 編譯錯誤（private）
- 封裝不是防「看」，是防「用」：**編譯器幫你擋下不該呼叫的東西**
- 對比：
  - **不封裝**：`bus().read()` 把整個 bus 物件交出去，bus 的所有 public 方法外面都能亂呼叫
  - **封裝**：`cpu_read()` 只開放 read/write，bus 的其他方法外面碰不到
- 封裝的真正價值是**畫界線**：
  - **public** = 我答應你永遠會有的介面（可以放心用）
  - **private** = 內部實作，我隨時可能改（你別依賴它）
- 你改 private 不影響別人，改 public 一定影響別人

### 10. 為什麼 PRG-ROM 用 std::vector 不用 std::array
```cpp
std::vector<uint8_t> prg_rom(header.prg_rom_count * 16384);
```
- `std::array<uint8_t, N>` 的 N 必須是**編譯期常數**（如 2048、8192）
- `std::vector` 的大小可以**執行期決定**
- `header.prg_rom_count` 是讀檔頭才知道的值，編譯期未知
- 如果用 array：`std::array<uint8_t, header.prg_rom_count * 16384>` → 編譯錯誤
- 對照 CpuBus 的 `ram_` 用 `std::array<uint8_t, 2048>`，因為 RAM 固定 2KB 是寫死的常數

### 11. make_mapper 只會有 Mapper000 嗎
- 目前 Step1 只實作 Mapper000（NROM），因為 nestest.nes 用的就是 mapper 0
- 原版 StepFC 之後的 step 會加更多 mapper：
  - **Mapper 1**（MMC1）：bank 切換，支援大容量 ROM（如 The Legend of Zelda）
  - **Mapper 4**（MMC3）：更複雜的 bank 切換（如 Super Mario Bros. 3）
- 到時候加 `case 1:` `case 4:` 就好，結構不用改
- 這就是 Factory pattern 的好處——加新 Mapper 只加 case，不改呼叫端

### 12. make_unique 能加變數名字嗎
- 已經有變數名字了，在等號左邊：
  ```cpp
  std::unique_ptr<nes::RomLoader> loader = std::make_unique<nes::FileRomLoader>("nestest.nes");
  //                              ^^^^^^
  //                              這就是變數名字
  ```
- stack 寫法：`型別 變數名(參數)` → `Famicom famicom(std::move(info))`
- heap 寫法：`型別 變數名 = make_unique<型別>(參數)`
- 可以用 `auto` 簡化：`auto loader = std::make_unique<...>(...)`

### 13. 不用指標直接 FileRomLoader loader("...") 可以嗎
- 可以，`FileRomLoader loader("nestest.nes")` 也能跑
- 差別在**多型**：
  - 不用指標：具體型別，沒有多型
  - 用父類指標：`unique_ptr<RomLoader>` 指向子類，多型
- 什麼時候需要多型：之後加 MemoryRomLoader、NetworkRomLoader 時，多型版換 loader 不改主邏輯
- 目前只有一種 loader，不用指標也行；用指標是為了教學多型概念 + 未來擴展
- 取捨：簡單優先 → 不用指標；擴展優先 → 用父類指標

### 14. prg_data_ 為什麼不用 std::move
```cpp
Mapper000(const RomInfo& rom)
    : prg_data_(rom.prg_rom().data())
```
- `prg_data_` 是 **raw pointer**（`const uint8_t*`），不是擁有資源的智慧指標
- `std::move` 是用來**轉移所有權**的，對指標沒意義——指標只是抄地址，沒有東西可以搬
- 而且 `rom` 是 `const&`，不能 move const 的東西（move 會改動原物件，const 不准改）
- 比喻：raw pointer 是地址紙條，你抄了地址，房子還是別人的
- 資料流向：RomInfo（擁有 vector）→ Mapper000（借用 const uint8_t*，指著 vector 裡的資料）
- 跟 Stage 5 的 `CpuBus::mapper_` 同一個模式——**unique_ptr 擁有，raw pointer 借用**

### 15. ram_ 目前是空的，什麼時候會被載入東西
- 目前 `ram_` 全是 0（`std::array<uint8_t, 2048> ram_{}` 的 `{}` 初始化清零）
- **RAM 不會被「載入」東西**——它是 CPU 的工作記憶體，不是從 ROM 載入的
- 內容是**程式執行時動態產生**的：
  ```
  開機 → RAM 全 0
  CPU 讀 RESET 向量 → $C004
  CPU 跳到 $C004 開始執行程式
    → 執行 LDA #$05 → CPU 寫 $05 到 RAM 某位址
    → 執行 STA $00  → RAM $0000 = 0x05
    ...程式不斷讀寫 RAM
  ```
- 三種記憶體的差異：

| 記憶體 | 開機時 | 誰寫入 |
|--------|--------|--------|
| RAM ($0000-$07FF) | 全 0 | **CPU 執行程式時寫入**（變數、堆疊） |
| SRAM ($6000-$7FFF) | 全 0 | **CPU 執行時寫入**（存檔資料） |
| PRG-ROM ($8000-$FFFF) | 載入 ROM 時就有 | **唯讀**，從 .nes 檔讀進來 |

- ROM 是程式碼（指令），RAM 是程式執行時用的草稿紙
- CPU 跑起來之後（Step2+ 實作 CPU），自然會往 RAM 寫東西
- 現在 Step1 只驗證讀取路徑，CPU 還沒開始跑，所以 RAM 是空的

### 16. Stage 9 做的事 — 驗證整條路通了
1. **載入 ROM**：跟 Step0 一樣，讀 .nes 檔
2. **建 Famicom**：Famicom 內部建 Mapper000 + CpuBus，把 PRG-ROM 接到 $8000-$FFFF
3. **讀中斷向量**：CPU 開機時不知道程式從哪跑，要去 $FFFC-$FFFD 讀 RESET 向量取得入口位址
4. **印出結果**：
   - NMI → $C5AF（發生 NMI 時 CPU 跳去 $C5AF）
   - RESET → $C004（開機時 CPU 跳去 $C004 開始執行）
   - IRQ → $C5F4（發生 IRQ 時 CPU 跳去 $C5F4）

**為什麼要讀這三個**：之後 Step2+ 實作 CPU 時，CPU 開機第一件事就是讀 RESET 向量取得程式入口。現在先把「讀得到」這件事驗證好，確認 Mapper + CpuBus 的讀取路徑是通的。

## 學習心得
Stage 9 是 Step1 的整合階段，把前面 Stage 0-8 做的所有元件串起來：ROM 載入 → Mapper 映射 → CpuBus 讀取 → 中斷向量。核心知識是 little-endian 組裝：16 位元位址拆成兩個 byte 存在記憶體，讀出來要用 `<< 8` 把高 byte 推上去再用 `|` 合併。nesdev wiki 用 PCL/PCH 術語表達小端序，沒有寫出「little-endian」這個字但意思一樣。`std::hex` + `std::uppercase` 是 sticky 的，設定一次後續生效。Famicom 的 `cpu_read` 包一層是封裝，藏的不是「用哪個 bus」而是「bus 這個物件的存在本身」，讓外面只能用你允許的操作，改 private 不波及別人。PRG-ROM 用 vector 不用 array 因為大小執行期才知道。make_mapper 目前只有 Mapper000，之後加 case 就好，這是 Factory pattern 的好處。make_unique 的變數名字在等號左邊，可以用 auto 簡化。不用指標直接建構也可以，差別在多型——目前只有一種 loader 不用指標也行。Mapper000 的 `prg_data_` 不用 move 因為 raw pointer 沒有所有權，只是借用 RomInfo vector 裡的資料。RAM 不會被「載入」，是 CPU 執行程式時動態寫入的，目前 CPU 還沒跑所以是空的。這個 Stage 驗證了整條讀取路徑通了，為 Step2+ 實作 CPU 執行打好基礎。
