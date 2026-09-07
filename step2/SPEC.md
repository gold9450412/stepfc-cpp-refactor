---
layout: default
title: Step2 C++ 重構規格書
---

# Step2 C++ 重構規格書

## 目的

將 StepFC 的 Step2（6502 反組譯）用現代 C++ 重新實作。目標不是「跑 CPU 指令」，而是：

1. 建立 6502 指令表（256 個 opcode → 指令 + 定址模式 + 長度）
2. 建立反組譯器（Disassembler）
3. 輸出 NMI / RESET / IRQ 三個向量入口處的第一條組合語言指令

**由你親自撰寫程式碼，我負責引導與 review。**

## 與原版的差異（C → 現代 C++）

| 原版（業餘） | 重構版（商業級） | 為什麼 |
|--------------|------------------|--------|
| `union sfc_6502_code_t` 把 op/a1/a2/ctrl 疊在一起 | `InstructionTable` 直接查表取 `InstructionInfo` | union 型別不安全、容易踩 endian 地雷 |
| `char name[3]` 手拼指令名稱 | `enum class Instruction` + `to_string()` | 強型別、不怕拼錯、可 switch |
| C enum 寫定址模式 | `enum class AddressingMode` | 型別安全，不隱式轉 int |
| `sfc_opname_data[256]` 裸陣列 | `constexpr std::array<InstructionInfo, 256>` | 編譯期常數、有 `.size()`、可查 |
| 指令長度塞在 `ctrl` 欄位一起編碼 | `InstructionInfo.length` 明確成員 | 資料與控制分離 |
| 反組譯寫死 `char buf[SFC_DISASSEMBLY_BUF_LEN]` | 回傳 `std::string` | C++ 風格，不用管理 buffer 大小 |
| 自由函式 `sfc_6502_disassembly()` | `Disassembler` 類別，Constructor 注入 `const CpuBus&` | 可測試、依賴反轉 |
| 全部塞 sfc_6502.c 一個檔（400+ 行） | 拆 `instruction.h` / `instruction_table.h/.cpp` / `disassembler.h/.cpp` | 單一職責 |
| 無測試 | Google Test 涵蓋指令表、反組譯格式、端到端 | 業界標準 |

## C++ 版本

C++17

## 需要安裝的工具與函式庫

與 Step1 完全相同，**不需要新安裝任何東西**。

## 專案結構

```
step2/cpp/
├── CMakeLists.txt
├── SPEC.md                        ← 本規格書
├── nestest.nes                    ← 測試用 ROM（已從 step1 複製）
├── src/
│   ├── main.cpp                   ← 主程式（反組譯三個向量）
│   └── nes/
│       ├── error.h                ← 從 step1 複製
│       ├── nes_header.h/.cpp      ← 從 step1 複製
│       ├── rom_info.h/.cpp        ← 從 step1 複製
│       ├── rom_loader.h           ← 從 step1 複製
│       ├── file_rom_loader.h/.cpp ← 從 step1 複製
│       ├── cpu_vectors.h          ← 從 step1 複製
│       ├── mapper.h               ← 從 step1 複製
│       ├── mapper000.h/.cpp       ← 從 step1 複製
│       ├── mapper_factory.h/.cpp  ← 從 step1 複製
│       ├── cpu_bus.h/.cpp         ← 從 step1 複製
│       ├── famicom.h/.cpp         ← 從 step1 複製（本 step 擴展）
│       ├── instruction.h          ← 新增：Instruction + AddressingMode enum class
│       ├── instruction_table.h    ← 新增：InstructionInfo + 查表介面
│       ├── instruction_table.cpp  ← 新增：256 條 opcode 表資料
│       ├── cpu.h/.cpp            ← 新增：CPU 暫存器 + reset（狀態容器，不執行指令）
│       ├── disassembler.h        ← 新增：反組譯器宣告
│       └── disassembler.cpp       ← 新增：反組譯器實作
├── tests/
│   ├── CMakeLists.txt
│   ├── test_smoke.cpp             ← 從 step1 複製
│   ├── test_rom_info.cpp          ← 從 step1 複製
│   ├── test_file_rom_loader.cpp   ← 從 step1 複製
│   ├── test_cpu_bus.cpp           ← 從 step1 複製
│   ├── test_mapper000.cpp         ← 從 step1 複製
│   ├── test_integration.cpp       ← 從 step1 複製
│   ├── test_instruction_table.cpp ← 新增：指令表測試
│   └── test_disassembler.cpp      ← 新增：反組譯測試
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
  ├── RomInfo (from step0/1)
  ├── unique_ptr<Mapper> (抽象類別)
  │     └── Mapper000 (NROM)
  └── CpuBus (CPU 64KB 位址空間)
        ├── RAM (2KB)
        ├── SRAM (8KB)
        └── Mapper*

Disassembler
  └── 依賴 const CpuBus&（唯讀記憶體）
      └── 查表 InstructionTable（256 entries，constexpr）
```

### 6502 指令表核心結構

每個 opcode（0x00-0xFF）對應：

```cpp
struct InstructionInfo {
    Instruction instruction;    // 例如 Instruction::LDA
    AddressingMode mode;        // 例如 AddressingMode::Immediate
    uint8_t length;            // 指令長度（1~3 bytes）
};
```

### 定址模式一覽（13 種）

| 模式 | 長度 | 組語寫法範例 |
|------|------|-------------|
| Accumulator  | 1 | `ASL A` |
| Implied      | 1 | `SEI` |
| Immediate    | 2 | `LDA #$AB` |
| ZeroPage     | 2 | `LDA $AB` |
| ZeroPage,X   | 2 | `LDA $AB,X` |
| ZeroPage,Y   | 2 | `LDA $AB,Y` |
| Absolute     | 3 | `LDA $ABCD` |
| Absolute,X   | 3 | `LDA $ABCD,X` |
| Absolute,Y   | 3 | `LDA $ABCD,Y` |
| Indirect     | 3 | `JMP ($ABCD)` |
| Indirect,X   | 2 | `LDA ($AB,X)` |
| Indirect,Y   | 2 | `LDA ($AB),Y` |
| Relative     | 2 | `BPL $ABCD`（顯示跳躍目標位址）|

### 反組譯器介面

```cpp
class Disassembler {
public:
    explicit Disassembler(const CpuBus& bus);

    // 反組譯指定地址的一條指令
    // addr: 指令位址（PC 位置）
    // next_addr: 輸出參數，下一條指令的位址
    // 回傳格式化字串，例如 "$C004: SEI"
    std::string disassemble(uint16_t addr, uint16_t& next_addr) const;

    // 簡易版，不需 next_addr
    std::string disassemble(uint16_t addr) const;

private:
    const CpuBus& bus_;
};
```

### Relative 定址的目標計算

Relative 定址的 operand 是 **int8_t 相對偏移**：
```
target = addr + 2 + (int8_t)offset
```

### Unofficial Opcodes

全部 256 個 opcode 都放進表，包含 unofficial/illegal（`SLO`, `RLA`, `DCP`, `ISC`, `LAX`, `SAX`, `ALR`, `ANC`, `ARR`, `AXS`, `LAS`, `XAA`, `AHX`, `TAS`, `SHX`, `SHY`, `NOP` 變種, `STP` 等）。

原因：nesdev wiki 明確寫「An accurate NES emulator must implement all instructions, not just the official ones.」反組譯階段全收，執行階段（Step3+）再處理語意。

## Stages

### Stage 0: 環境設定 + 複製 step1 程式碼
- [x] 確認 step1 程式碼複製到 `step2/cpp/`
- [x] CMakeLists.txt 專案名改 `step2_cpp`
- [x] `cmake -B build && cmake --build build && ctest --test-dir build` 通過（17 個測試全過）
- [x] `./build/step2_cpp` 執行正常，輸出同 step1

**需安裝**：無新工具

**學習重點**：專案重用、分支式開發起點

**對照原版**：無（基礎建設）

---

### Stage 1: 指令與定址模式 enum (`instruction.h`)
- [x] 定義 `enum class AddressingMode`（13 種值）
- [x] 定義 `enum class Instruction`（全部指令含 unofficial）
- [x] 宣告 `const char* to_string(Instruction)` 和 `const char* to_string(AddressingMode)`
- [x] 放在 `namespace nes`

**需安裝**：無

**學習重點**：enum class 設計、switch 覆蓋的完整性

**對照原版**：`enum sfc_6502_instruction` + `enum sfc_6502_addressing_mode`

---

### Stage 2: 指令表 (`instruction_table.h` / `instruction_table.cpp`)
- [x] 定義 `struct InstructionInfo { Instruction instruction; AddressingMode mode; uint8_t length; }`
- [x] 建立 `constexpr std::array<InstructionInfo, 256> kInstructionTable`（塞滿 256 條）
- [x] 提供 `const InstructionInfo& lookup(uint8_t opcode)` 查表
- [x] unofficial opcode 也要放（表裡註解分區）

**需安裝**：無

**學習重點**：constexpr std::array、查表法（table lookup）取代 switch

**對照原版**：`s_opname_data[256]`（裸 C 陣列）

---

### Stage 3: CPU 暫存器 (`cpu.h` / `cpu.cpp`)
- [x] 定義 `struct CpuRegisters`：A, X, Y, SP, PC, P（全部 uint8_t 或 uint16_t 對應硬體尺寸）
- [x] 定義 `class Cpu`：持有 `CpuRegisters` + `CpuBus&`
- [x] 提供 `reset()`：從 `kResetVector` 讀位址設到 PC
- [x] 提供 getter
- [x] 注意：本 step 不做指令執行，只做狀態容器

**需安裝**：無

**學習重點**：暫存器建模、資料封裝

**對照原版**：原版 Step2 沒有 CPU 類別（Step3 才有）

---

### Stage 4: Disassembler 骨架 (`disassembler.h` / `disassembler.cpp`)
- [x] 定義 `class Disassembler`，建構子注入 `const CpuBus&`
- [x] 實作 `disassemble(uint16_t addr, uint16_t& next_addr)`：
  - 讀 opcode
  - 查表取得 InstructionInfo
  - 根據 mode 讀 operand bytes
  - 計算 next_addr = addr + info.length
- [x] 格式：先只輸出 mnemonic，operand 下一 Stage 加

**需安裝**：無

**學習重點**：依賴注入、查表驅動（table-driven）設計

**對照原版**：`sfc_fc_disassembly()`（自由函式 + char buf）

---

### Stage 5: 反組譯輸出 — 無 operand 模式
- [x] Accumulator：`ASL A`
- [x] Implied：`SEI`
- [x] Immediate：`LDA #$AB`（operand 是 1 byte hex）
- [x] Relative：`BPL $ABCD`（計算跳躍目標）

**需安裝**：無

**學習重點**：定址模式字串格式化、`int8_t` 相對跳躍

**對照原版**：`sfc_6502_disassembly()` 的 `SFC_AM_IMP/ACC/IMM/REL` 分支

---

### Stage 6: 反組譯輸出 — 1 byte 位址模式
- [x] ZeroPage：`LDA $AB`
- [x] ZeroPage,X：`LDA $AB,X`
- [x] ZeroPage,Y：`LDA $AB,Y`
- [x] Indirect,X：`LDA ($AB,X)`
- [x] Indirect,Y：`LDA ($AB),Y`

**需安裝**：無

**學習重點**：零頁格式（2 位 hex）

**對照原版**：同上

---

### Stage 7: 反組譯輸出 — 2 byte 位址模式
- [x] Absolute：`LDA $ABCD`
- [x] Absolute,X：`LDA $ABCD,X`
- [x] Absolute,Y：`LDA $ABCD,Y`
- [x] Indirect：`JMP ($ABCD)`

**需安裝**：無

**學習重點**：絕對位址格式（4 位 hex）、little-endian 組裝

**對照原版**：同上

---

### Stage 8: Famicom 擴展
- [x] 在 `class Famicom` 新增 `Cpu cpu_;` 成員
- [x] Famicom 建構子初始化 cpu_（連接 bus_）
- [x] 提供 `const Cpu& cpu() const` 與 `Cpu& cpu()`
- [x] 提供 `Disassembler disassembler()` 或直接 `disassemble(uint16_t addr)` 便捷方法

**需安裝**：無

**學習重點**：物件組合的擴展

**對照原版**：原版 Step2 沒有 CPU 實體，直接 `sfc_fc_disassembly()` 走全域

---

### Stage 9: main.cpp — 反組譯三個向量
- [x] 載入 ROM → 建 Famicom
- [x] 讀 NMI/RESET/IRQ 三個向量
- [x] 對每個向量反組譯第一條指令並輸出
- [x] 輸出格式範例如下

**需安裝**：無

**學習重點**：整合應用、字串格式化

**對照原版**：`main.c` 的 `sfc_fc_disassembly(v0, &famicom, b0)` 三次呼叫

**預期輸出**（nestest.nes，已查證實際 bytes：$C5AF=0x48 PHA、$C004=0x78 SEI、$C5F4=0x40 RTI）：
```
Step2 C++ - CPU Memory & Mapper
ROM: PRG-ROM: 1 x 16kb    CHR-ROM: 1 x 8kb    Mapper: 0
Mirroring: Horizontal    Save RAM: No    Four Screen: No
NMI Vector:   $C5AF: PHA
RESET Vector: $C004: SEI
IRQ Vector:   $C5F4: RTI
```
（原草稿猜測 NMI/IRQ 會印 `???` 是錯的：表 256 格全滿且兩個 handler 入口本來就是真指令，NMI 開場 PHA=中斷保護現場、IRQ 開場 RTI=直接返回）

---

### Stage 10: 指令表測試 (`test_instruction_table.cpp`)
- [x] 測試所有 256 個 opcode 查表都回傳有效 InstructionInfo（迴圈批次驗證 length 1-3）
- [x] 測試特定 opcode 對應正確（0x78 → SEI/Implied，0xA9 → LDA/Immediate，共 8 錨點）
- [x] 測試表長度 = 256（由 `lookup()` 隱含保證——測 public 介面而非伸手進 .cpp）
- [x] 測試官方指令 length 正確（錨點涵蓋 1/2/3 三種長度）

**需安裝**：無

**學習重點**：資料表驗證

---

### Stage 11: Disassembler 測試 (`test_disassembler.cpp`)
- [x] 用 FakeMapper（測試替身：`class FakeMapper : public nes::Mapper`，內建可 set 的 PRG bytes）塞自訂 bytes
- [x] 逐種定址模式驗證輸出字串（13 種全測，Relative 測前進+後退兩方向）
- [x] 驗證 Relative 跳躍目標計算正確（含負 offset 的符號擴充）
- [x] 驗證 next_addr 正確（1/2/3-byte 指令各批都有涵蓋）

**需安裝**：無

**學習重點**：Mock + 全模式覆蓋

---

### Stage 12: 整合測試 + 收尾
- [x] 對 `nestest.nes` 三個向量反組譯，驗證 RESET = `SEI`（`DisassembleNestestResetVector`）
- [x] 確認所有測試通過（34/34）
- [x] 確認 SPEC.md 所有 checkbox 打勾
- [x] 確認所有日誌寫完（stage01 ~ stage12）

**需安裝**：無

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
./build/step2_cpp

# 測試
ctest --test-dir build --output-on-failure

# 只跑測試
cmake --build build --target test_smoke test_rom_info test_file_rom_loader test_cpu_bus test_mapper000 test_integration test_instruction_table test_disassembler && ctest --test-dir build --output-on-failure
```
