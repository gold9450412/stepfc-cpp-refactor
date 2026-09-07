---
layout: default
title: "Stage 1 日誌：指令與定址模式 enum (src/nes/instruction.h / instruction.cpp)"
---

# Stage 1 日誌：指令與定址模式 enum (src/nes/instruction.h / instruction.cpp)

## 日期
2026-09-05

## 狀態
✅ 完成

## 完成事項
- 定義 `enum class AddressingMode`（13 種定址模式）✅
- 定義 `enum class Instruction`（56 官方 + 18 unofficial = 74 個指令）✅
- 宣告並實作 `const char* to_string(Instruction)` 與 `const char* to_string(AddressingMode)` ✅
- `instruction.cpp` 加入 CMakeLists.txt 的 `nes_lib` ✅
- 編譯零警告、17 個測試全過 ✅

## 最終程式碼

### src/nes/instruction.h（107 行）
```cpp
#pragma once

#include <cstdint>

namespace nes {

enum class AddressingMode {
    Accumulator,  // ASL A
    Implied,      // SEI
    Immediate,    // LDA #$AB
    ZeroPage,     // LDA $AB
    ZeroPageX,    // LDA $AB,X
    ZeroPageY,    // LDA $AB,Y
    Absolute,     // LDA $ABCD
    AbsoluteX,    // LDA $ABCD,X
    AbsoluteY,    // LDA $ABCD,Y
    Indirect,     // JMP ($ABCD)
    IndirectX,    // LDA ($AB,X)
    IndirectY,    // LDA ($AB),Y
    Relative,     // BPL $ABCD
};

enum class Instruction {
    ADC,
    AND,
    ASL,
    BCC,
    BCS,
    BEQ,
    BIT,
    BMI,
    BNE,
    BPL,
    BRK,
    BVC,
    BVS,
    CLC,
    CLD,
    CLI,
    CLV,
    CMP,
    CPX,
    CPY,
    DEC,
    DEX,
    DEY,
    EOR,
    INC,
    INX,
    INY,
    JMP,
    JSR,
    LDA,
    LDX,
    LDY,
    LSR,
    NOP,
    ORA,
    PHA,
    PHP,
    PLA,
    PLP,
    ROL,
    ROR,
    RTI,
    RTS,
    SBC,
    SEC,
    SED,
    SEI,
    STA,
    STX,
    STY,
    TAX,
    TAY,
    TSX,
    TXA,
    TXS,
    TYA,

    // --- Unofficial opcodes ---
    ALR,
    ANC,
    ARR,
    AHX,
    DCP,
    ISC,
    KIL,
    LAX,
    LAS,
    RLA,
    RRA,
    SAX,
    SHX,
    SHY,
    SLO,
    SRE,
    TAS,
    XAA,
};

const char* to_string(Instruction instruction);
const char* to_string(AddressingMode mode);

} // namespace nes
```

### src/nes/instruction.cpp（104 行）
```cpp
#include "instruction.h"

namespace nes {

const char* to_string(AddressingMode mode) {
    switch (mode) {
        case AddressingMode::Accumulator: return "A";
        case AddressingMode::Implied:      return "IMP";
        case AddressingMode::Immediate:    return "IMM";
        case AddressingMode::ZeroPage:     return "ZPG";
        case AddressingMode::ZeroPageX:    return "ZPX";
        case AddressingMode::ZeroPageY:    return "ZPY";
        case AddressingMode::Absolute:     return "ABS";
        case AddressingMode::AbsoluteX:    return "ABX";
        case AddressingMode::AbsoluteY:    return "ABY";
        case AddressingMode::Indirect:     return "IND";
        case AddressingMode::IndirectX:    return "IZX";
        case AddressingMode::IndirectY:    return "IZY";
        case AddressingMode::Relative:     return "REL";
    }
    return "???";
}

const char* to_string(Instruction instruction) {
    switch (instruction) {
        case Instruction::ADC: return "ADC";
        case Instruction::AND: return "AND";
        case Instruction::ASL: return "ASL";
        case Instruction::BCC: return "BCC";
        case Instruction::BCS: return "BCS";
        case Instruction::BEQ: return "BEQ";
        case Instruction::BIT: return "BIT";
        case Instruction::BMI: return "BMI";
        case Instruction::BNE: return "BNE";
        case Instruction::BPL: return "BPL";
        case Instruction::BRK: return "BRK";
        case Instruction::BVC: return "BVC";
        case Instruction::BVS: return "BVS";
        case Instruction::CLC: return "CLC";
        case Instruction::CLD: return "CLD";
        case Instruction::CLI: return "CLI";
        case Instruction::CLV: return "CLV";
        case Instruction::CMP: return "CMP";
        case Instruction::CPX: return "CPX";
        case Instruction::CPY: return "CPY";
        case Instruction::DEC: return "DEC";
        case Instruction::DEX: return "DEX";
        case Instruction::DEY: return "DEY";
        case Instruction::EOR: return "EOR";
        case Instruction::INC: return "INC";
        case Instruction::INX: return "INX";
        case Instruction::INY: return "INY";
        case Instruction::JMP: return "JMP";
        case Instruction::JSR: return "JSR";
        case Instruction::LDA: return "LDA";
        case Instruction::LDX: return "LDX";
        case Instruction::LDY: return "LDY";
        case Instruction::LSR: return "LSR";
        case Instruction::NOP: return "NOP";
        case Instruction::ORA: return "ORA";
        case Instruction::PHA: return "PHA";
        case Instruction::PHP: return "PHP";
        case Instruction::PLA: return "PLA";
        case Instruction::PLP: return "PLP";
        case Instruction::ROL: return "ROL";
        case Instruction::ROR: return "ROR";
        case Instruction::RTI: return "RTI";
        case Instruction::RTS: return "RTS";
        case Instruction::SBC: return "SBC";
        case Instruction::SEC: return "SEC";
        case Instruction::SED: return "SED";
        case Instruction::SEI: return "SEI";
        case Instruction::STA: return "STA";
        case Instruction::STX: return "STX";
        case Instruction::STY: return "STY";
        case Instruction::TAX: return "TAX";
        case Instruction::TAY: return "TAY";
        case Instruction::TSX: return "TSX";
        case Instruction::TXA: return "TXA";
        case Instruction::TXS: return "TXS";
        case Instruction::TYA: return "TYA";
        case Instruction::ALR: return "ALR";
        case Instruction::ANC: return "ANC";
        case Instruction::ARR: return "ARR";
        case Instruction::AHX: return "AHX";
        case Instruction::DCP: return "DCP";
        case Instruction::ISC: return "ISC";
        case Instruction::KIL: return "KIL";
        case Instruction::LAX: return "LAX";
        case Instruction::LAS: return "LAS";
        case Instruction::RLA: return "RLA";
        case Instruction::RRA: return "RRA";
        case Instruction::SAX: return "SAX";
        case Instruction::SHX: return "SHX";
        case Instruction::SHY: return "SHY";
        case Instruction::SLO: return "SLO";
        case Instruction::SRE: return "SRE";
        case Instruction::TAS: return "TAS";
        case Instruction::XAA: return "XAA";
    }
    return "???";
}

} // namespace nes
```

---

## 討論重點（學習紀錄）

### 1. 13 種定址模式是怎麼從 wiki 數出來的

來源：https://www.nesdev.org/wiki/CPU_addressing_modes

頁面結構是**兩張表**，沒有直接寫「13」這個數字：

| 表 | 張數 | 內容 |
|---|---|---|
| `## Indexed addressing` | 6 種 | `d,x` / `d,y` / `a,x` / `a,y` / `(d,x)` / `(d),y` — 用 X/Y 暫存器幫忙算位址的模式 |
| `## Other addressing` | 7 種 | Implicit / A / `#v` / `d` / `a` / label / `(a)` |

6 + 7 = 13。表的 `Abbr` 欄是該模式**組語寫法的長相**（例：`(d),y` → `LDA ($AB),Y`）。

### 2. `#` 的有無 = 值 vs 位址

```
LDA #$05   ; Immediate → A = 5（operand 本身就是值）
LDA $05    ; ZeroPage  → A = 記憶體[$0005] 的內容（operand 是位址）
```

一個 `#`，意思完全不同。Immediate 的 wiki 原文："Uses the 8-bit operand itself as the value... rather than fetching a value from a memory address."

### 3. 三個基本詞

| 詞 | 意思 |
|---|---|
| ZeroPage | 1 byte 指定位址 → 只指得到 $0000~$00FF（NES 上 = 內建 RAM 前 256 格）|
| Absolute | 2 bytes 指定位址 → 指得到全部 64K |
| Indirect | operand 是「存位址的格子的位址」（位址的位址，轉一手）|

ZeroPage 存在的價值：指令短 1 byte、執行快、ROM 省 — 6502 的招牌設計。

X/Y 是**位址偏移量**：`LDA $05,X` = 讀 `($05 + X)` 那格。跑迴圈改暫存器就能掃一排資料。

### 4. 定址模式存取的不是 PRG ROM，是整個 CPU 64K 位址空間

`CpuBus::read()` 那個 switch 就是答案：

| 位址 | 內容 |
|---|---|
| $0000-$1FFF | RAM（2KB 鏡像）|
| $2000-$3FFF | PPU 暫存器 |
| $4000-$401F | APU / 手柄 |
| $6000-$7FFF | SRAM |
| $8000-$FFFF | PRG ROM |

`LDA $0543` 讀的是 RAM 不是 ROM。6502 本身不知道什麼是 ROM — 它只發 16-bit 位址，接線的是匯流排（CpuBus + Mapper）。ROM 只能讀不能寫。

### 5. 助憶符（mnemonic）與 wiki 對照頁

opcode 是 byte（機器語言），助憶符是給人記的縮寫：ADC = ADd with Carry、LDA = LoaD Accumulator、DEX = DEcrement X。規律：`B??` = Branch、`CL?`/`SE?` = 清/設旗標。

wiki 頁面（nesdev.org 擋機器人，要瀏覽器開）：
- 完整 256 opcode 矩陣：https://www.nesdev.org/wiki/CPU_unofficial_opcodes（雖叫 unofficial，實為官方+非官方全表）
- 每指令詳細頁：http://www.obelisk.me.uk/6502/reference.html

### 6. opcode 矩陣怎麼讀、為什麼那麼多 ADC

16×16 表 = 座標地圖：**列 = opcode 高 4 bit，欄 = 低 4 bit**。第 6 列第 9 行 = `$69`。

> **opcode ≠ 指令。一個指令有幾種定址模式，就佔幾個 opcode。**

ADC 有 8 種定址模式 → `$69/$65/$75/$6D/$7D/$79/$61/$71` 共 8 格。「加法」只有一件，但「去哪拿運算元」有 8 種拿法，CPU 必須先知道是哪一種。

這正是 Stage 2 `kInstructionTable[256]` 的由來：讀到 byte → 查表 → InstructionInfo{instruction, mode, length}。

### 7. `$60` = RTI（reviewer 記錯被修正）

reviewer 一度說 `$60` 是 KIL — **錯誤**。`$60` 是 **RTI**（ReTurn from Interrupt，中斷返回），官方指令。KIL/JAM 是 `$02/$12/$22/...` 那排。教訓：opcode 對照要以 wiki 為準，不要信記憶。

### 8. `enum class` 沒辦法 cout — C++ 沒有反射

```cpp
std::cout << Instruction::ADC;  // 編譯錯誤！enum class 不准隱式轉 int
```

`"ADC"` 這三個字只存在於原始碼，編譯後只剩數字 0。C++ 沒有「runtime 還記得名字」的功能，所以 enum → 字串必須自己寫 `to_string()` 對照表。

### 9. 為什麼回傳 `const char*` 而不是 `std::string`

字串字面值（`"LDA"`）編譯時就躺在執行檔唯讀區：

- 回傳 `const char*` = 抄門牌地址給你（0 配置、0 複製）
- 回傳 `std::string` = 照著蓋新房子再給你鑰匙（配置 + 複製字元）

「查名字」函式回傳 `const char*`（或 C++17 的 `std::string_view`）是業界慣例。

### 10. switch 故意不寫 `default` — 讓編譯器抓 bug

`-Wall -Wextra` 開著時，switch 沒有 default 且漏列任何 enum 值，編譯器會警告。這比 default 更安全：新增 enum 值時忘了加 case，**編譯期**就會被提醒，不是 runtime 印出 `???` 才發現。函式結尾的 `return "???"` 只是安撫編譯器「所有路徑都有回傳值」。

## 遇到的問題
- 無編譯錯誤、無測試失敗。本 stage 順利。
- 小瑕疵：instruction.h 曾有多餘空行（已清理）。

## Review 建議
- `enum class` + `to_string` + switch 全覆蓋的組合正確，是業界標準做法
- unofficial 指令用註解分區（`// --- Unofficial opcodes ---`）清楚
- 縮寫（IMP/IMM/ZPG…）跟 nesdev / 原版 sfc_6502.c 一致，好維護

## 學習心得

1. **定址模式 = operand 的解釋方式**。值（Immediate）還是位址（其他 12 種）
2. **13 = 6 + 7**：wiki 用「索引」與「其他」兩張表分類，數格子就知道
3. **opcode ≠ 指令**：一個指令 N 種模式就 N 個 opcode，256 格是 byte 的全排列
4. **C++ 沒有反射**：enum 的名字編譯後消失，to_string 要人工寫
5. **`const char*` 回傳字面值零成本**：不複製現成的東西
6. **switch 無 default + 全警告**：把 bug 攔在編譯期
