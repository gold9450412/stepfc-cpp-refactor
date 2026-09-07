---
layout: default
title: "Stage 2 日誌：指令表 (src/nes/instruction_table.h / instruction_table.cpp)"
---

# Stage 2 日誌：指令表 (src/nes/instruction_table.h / instruction_table.cpp)

## 日期
2026-09-05

## 狀態
✅ 完成

## 完成事項
- 定義 `struct InstructionInfo { Instruction, AddressingMode, uint8_t length }` ✅
- `constexpr std::array<InstructionInfo, 256> kInstructionTable` 填滿 256 條（official + unofficial，unofficial 行尾標 `*`）✅
- 實作 `const InstructionInfo& lookup(uint8_t opcode)` 查表 ✅
- 表以 4 批（0x00-0x3F / 0x40-0x7F / 0x80-0xBF / 0xC0-0xFF）分次填入，邊填邊對照 nesdev 矩陣 ✅
- `enum Instruction` 補上漏掉的 `AXS`（第 19 個 unofficial）✅
- `instruction_table.cpp` 加入 CMakeLists.txt 的 `nes_lib` ✅
- 編譯零警告、17 個測試全過 ✅

## 最終程式碼

### src/nes/instruction_table.h（19 行）
```cpp
#pragma once

#include <cstdint>

#include "instruction.h"

namespace nes {

// 一個 opcode 的完整資訊：指令 + 定址模式 + 指令長度
struct InstructionInfo {
    Instruction instruction;
    AddressingMode mode;
    uint8_t length;
};

// 查表：opcode → InstructionInfo
const InstructionInfo& lookup(uint8_t opcode);

} // namespace nes
```

### src/nes/instruction_table.cpp（304 行，節錄）
```cpp
{% raw %}
#include <array>

#include "instruction_table.h"

namespace nes {

// 256 格 opcode 查表：索引 = opcode byte 本身
// 行尾標 * 的為 unofficial opcode
constexpr std::array<InstructionInfo, 256> kInstructionTable = {{   // ← 注意雙層 brace！
    // 0x00 - 0x0F
    {Instruction::BRK, AddressingMode::Implied,    1}, // 00
    {Instruction::ORA, AddressingMode::IndirectX,  2}, // 01
    {Instruction::KIL, AddressingMode::Implied,    1}, // 02 *
    {Instruction::SLO, AddressingMode::IndirectX, 2}, // 03 *
    {Instruction::NOP, AddressingMode::ZeroPage,   2}, // 04 *
    {Instruction::ORA, AddressingMode::ZeroPage,   2}, // 05
    {Instruction::ASL, AddressingMode::ZeroPage,   2}, // 06
    {Instruction::SLO, AddressingMode::ZeroPage,   2}, // 07 *
    {Instruction::PHP, AddressingMode::Implied,    1}, // 08
    {Instruction::ORA, AddressingMode::Immediate, 2}, // 09
    {Instruction::ASL, AddressingMode::Accumulator, 1}, // 0A
    {Instruction::ANC, AddressingMode::Immediate, 2}, // 0B *
    {Instruction::NOP, AddressingMode::Absolute,   3}, // 0C *
    {Instruction::ORA, AddressingMode::Absolute,   3}, // 0D
    {Instruction::ASL, AddressingMode::Absolute,   3}, // 0E
    {Instruction::SLO, AddressingMode::Absolute,   3}, // 0F *
    // ...（中間 224 格見原檔，結構完全同構）...
    {Instruction::SBC, AddressingMode::AbsoluteX,  3}, // FD
    {Instruction::INC, AddressingMode::AbsoluteX,  3}, // FE
    {Instruction::ISC, AddressingMode::AbsoluteX, 3}, // FF *
}};

// 查表：opcode → InstructionInfo
const InstructionInfo& lookup(uint8_t opcode) {
    return kInstructionTable[opcode];
}

} // namespace nes
{% endraw %}
```

---

## 討論重點（學習紀錄）

### 1. 為什麼表要 256 格

opcode 是 1 byte = 2⁸ = **256 種可能**。CPU 讀到什麼 byte 都必須查得到——包括非法 opcode（真實遊戲如 Micro Machines、Aladdin (E) 依賴非法 opcode，nesdev 明說「An accurate NES emulator must implement all instructions」）。陣列索引直接對應 opcode 值，O(1) 且**永遠查得到**，不存在「查不到」這個狀態。

### 2. 為什麼存 length（而不是從 mode 推導）

指令長度不固定：Implied/Accumulator 1 byte、Immediate/ZeroPage/… 2 bytes、Absolute/… 3 bytes。反組譯器要算 `next_addr = addr + length` 才能往下走。原版 C 把長度塞在 `ctrl` 欄位編碼，本專案選擇明確成員，理由：**資料與控制分離**、**明確勝於隱含**、也不浪費空間（struct padding 剛好放下）。

### 3. 分批填表 + 自動補零（aggregate initialization）

`std::array<T, 256>` 只給前 64 個初始值時，**剩下的格子由編譯器自動補零**（語言內建行為，不是我們寫的程式碼）。所以填表流程設計成：每批內容插在 `};` 之前，檔案隨時保持完整語法，補零格子被後續批次逐批蓋掉，第四批填滿 256 格。

「補零」的副作用：未填格子的三個成員都是 0 → `instruction=0` 恰好是 `Instruction::ADC`（enum 第一個值）、`mode=0` 恰好是 `Accumulator`、`length=0`（非法值，真實指令沒有 0 byte）。看起來像 ADC 只是「三個欄位剛好都是 0」，是垃圾佔位資料，不是真的指令。

### 4. 指令長度的記憶體長相

```
$8000: 48          PHA        length=1 → 只有自己的 opcode
$8000: A9 05       LDA #$05   length=2 → opcode + 1 byte 運算元
$8000: AD 34 12    LDA $1234  length=3 → opcode + 2 byte 位址（little-endian）
```

Absolute 之所以 3 bytes：要表達 0~65535 的任意位址，數學上就需要 16 bit = 2 byte，省不掉。ZeroPage 是「只去頭 256 格」的省 byte 技巧。

反組譯器靠這個吃飯：讀 1 byte → 查表 → length → 跳到 addr+length 繼續。表查錯 length，後面全部拆歪。

### 5. nesdev 矩陣的記法（本 stage 的一大烏龍）

AI 一開始說表上用 `imp` / `imm` 縮寫——**錯的**，那是別的網站（obelisk）的慣例。nesdev 那頁的真正記法是「指令名 + 運算元寫法」：

| 表上樣子 | 意思 | 長度 |
|---|---|---|
| `NOP`（後面空的） | Implied / Accumulator | 1 |
| `ORA #i` / `NOP d` / `BPL *+d` / `ORA (d,x)` | 1-byte 運算元 | 2 |
| `NOP a` / `ORA a,y` / `JMP (a)` | 16-bit 位址 | 3 |

口訣：**名字後面空的 → 1；有 `d` 或 `#i` → 2；有 `a` → 3**。

另外該頁把 KIL 叫 **STP**（$02/$12/$22/…），填表時名稱要對照好。教訓：記法引用錯來源時，對方查不到是正常的，要當場驗證。

### 6. `a,x` = AbsoluteX；「怎麼算位址」和「存什麼」是兩件獨立的事

nesdev 寫 `SHY a,x`，`a,x` 就是「位址 = a + X」= `AddressingMode::AbsoluteX`。可能疑惑：STA a,x（存 A）和 SHY a,x（存 Y）定址模式要不要分兩種？**不用**——定址模式只描述位址怎麼來，跟資料從哪個暫存器來無關（那是 Instruction 的差別）。一個 `AbsoluteX` 同時服務 $9D 和 $9C。

### 7. 6502 暫存器全家族，為什麼定址只用 X、Y

| 暫存器 | 寬度 | 工作 |
|---|---|---|
| A | 8-bit | 累加器——算術的心臟（接 ALU） |
| X | 8-bit | 索引——專門加到位址上 |
| Y | 8-bit | 索引——同 X |
| SP | 8-bit | 堆疊指標，綁死 $0100-$01FF |
| PC | 16-bit | 程式計數器（現在執行到哪） |

X/Y 的硬體線路天生接到位址計算電路（所以才叫索引暫存器）；A 接 ALU；SP/PC 各有專職。指令集的形狀（`LDA a,x` 存在、沒有 `LDA a,a`）就是暫存器分工的形狀。還有 P（旗標）暫存器，執行器階段再講。

### 8. 表的區塊規律（填表時的驗證技巧）

- `x2` 那行（02/12/22/…）全是 KIL；`x3` 那行全是 RMW 合成指令（SLO/RLA/SRE/RRA/DCP/ISC）
- `0x0x` 列 = ORA 系、`0x2x` = AND 系、`0x4x` = EOR 系、`0x6x` = ADC 系、`0xCx` = CMP 系、`0xEx` = SBC 系
- ADC 有 8 種定址 → 8 個 opcode（$69/$65/$75/$6D/$7D/$79/$61/$71），全在 0x60-0x7F——「一個指令 N 種模式就 N 個 opcode」的實體
- `0x6C` JMP (a) 是全表唯一 official Indirect
- `0x80` 區是 store 區：罕見的 ZeroPageY 只出現在 STX/LDX/LAX（$96/$B6/$B7，用 Y 當索引）
- `$9B~$9F` 是「不穩定家族」（TAS/SHX/SHY/AHX），行為不可預測，模擬器界公認最難搞
- `$EB` 是 unofficial 的 SBC #imm——$E9 的完全重複品（史稱 USBC），別手滑填成 NOP

### 9. `lookup()` 不用檢查邊界——讓型別保證正確

參數是 `uint8_t`（0-255），陣列剛好 256 格，任何值都是合法索引。與其在 runtime 檢查範圍，**讓型別系統一開始就排除非法狀態**——這個思路之後會一直出現。

---

## 遇到的問題

### 問題 1：`Instruction::AXS` 編譯錯誤
```
error: 'AXS' is not a member of 'nes::Instruction'
```
**原因**：Stage 1 建 enum 時漏列了 AXS（$CB，nesdev 也叫 AXS，有些文件叫 SBX）。AI 給表內容時直接用了它，沒先檢查 enum。**教訓：下游資料用到上游 enum 時，給碼前要先驗證上游完整。**
**修法**：`instruction.h` unofficial 區補 `AXS,`（字母序 AHX 之後）+ `instruction.cpp` to_string 補 `case Instruction::AXS: return "AXS";`。switch 沒寫 default 的設計（-Wswitch 抓漏 case）在這裡發揮了作用。

### 問題 2：`too many initializers for 'std::array<InstructionInfo, 256>'`
表內容反覆驗證都是整整齊齊的 256 條（256 個 entry、256 個不重複 opcode 註解），錯誤卻一直都在。用二分法隔離（1 條 OK、2 條 OK、3 條爆、跟 enum 無關、連純 `int` struct 都爆），最後確認是 **GCC 11 的 brace elision 解譯問題**：

```cpp
{% raw %}
std::array<S, 256> t = { {...}, {...}, {...} };   // 單層 → GCC 11 解譯錯誤
std::array<S, 256> t = {{ {...}, {...}, {...} }}; // 雙層 → 正確
{% endraw %}
```

**原理**：`std::array` 內部是 `struct { T __elems[N]; }`——包著一條 C 陣列的 struct。嚴格的初始化要**兩層 brace**（外層給 struct、內層給 C 陣列），單層寫法靠「括號省略」規則通融，但 GCC 11 在巢狀情境會把第一個元素吃錯位置，後面全算成多的。**結論：`std::array` 初始化永遠寫雙層 brace 最穩。**

（排查過程中一度以為是重複的 `};` 或多算 entry——都不是。錯誤訊息指著收尾 `};`，真正原因在開頭 `= {` 少一層，跟訊息位置直覺相反。）

### 問題 3：小瑕疵
- `instruction_table.cpp` 一開始忘了 `#include "instruction_table.h"`
- header 一開始留著 `constexpr std::array<...> kInstructionTable;` 宣告——constexpr 變數**一定要有初始值**，不能先宣告後定義；且表是實作細節，藏進 .cpp、只曝光 `lookup()` 是資訊隱藏的正確做法
- header 的 `<array>` 移到 .cpp 後，記得「誰用誰 include」：.cpp 自己補上 `#include <array>`
- 貼 lookup 時殘留一個多餘的 `};`（已刪）

## Review 建議
- 每行行尾的 `// 0x` opcode 註解非常好：對照矩陣除錯時一眼定位，是資料表的良好實務
- unofficial 標 `*` + `// ---` 分區註解清楚
- 分 4 批填 + 每批人工對照 nesdev，流程穩，只有 2 個錯且都不是表資料本身
- 雙層 brace 的 `{% raw %}= {{ ... }}{% endraw %}` 之後維持這個寫法

## 學習心得

1. **256 = byte 的全排列**：表必須填滿，非法 opcode 也是「合法的查詢對象」
2. **明確成員勝於隱含編碼**：length 直接存，不從 mode 推
3. **aggregate initialization 自動補零**：少給的初始值補 0——拿來當「分批施工」的鷹架很方便
4. **nesdev 記法**：空 → 1B、`d`/`#i` → 2B、`a` → 3B；KIL 在該頁叫 STP
5. **定址模式與資料來源是正交的**：一個 AbsoluteX 服務所有「位址 = a + X」的指令
6. **暫存器分工 = 指令集形狀**：X/Y 接位址線路、A 接 ALU
7. **`std::array` 永遠雙層 brace**：`{% raw %}{{ ... }}{% endraw %}`，單層靠 elision 在 GCC 11 會踩雷
8. **讓型別排除非法狀態**（uint8_t 索引 256 格免檢查）比 runtime 防禦好
