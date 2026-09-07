---
layout: default
title: "Stage 11 — Disassembler 測試（test_disassembler.cpp）"
---

# Stage 11 — Disassembler 測試（test_disassembler.cpp）

> 日期：2026-09-07
> 對應 SPEC Stage 11：FakeMapper 塞自訂 bytes + 13 種定址模式全測 + Relative 跳躍目標 + next_addr

## 完成事項

- `tests/test_disassembler.cpp`（118 行，14 個 TEST）：
  - **FakeMapper 測試替身**：`class FakeMapper : public nes::Mapper`，內部 `std::array<uint8_t, 32*1024> prg_`，加上測試專用 `set_prg(uint16_t addr, uint8_t data)`（吃 CPU 座標，內部減 $8000）
  - **14 個測試**全數沿用同一 pattern（FakeMapper → CpuBus → Disassembler → 斷言字串 + next_addr）：
    1. `ImpliedSei`（$78 → `"SEI"`）
    2. `AccumulatorAsl`（$0A → `"ASL A"`）
    3. `ImmediateLda`（$A9 0xAB → `"LDA #$AB"`）
    4. `ZeroPageLda`（$A5 → `"LDA $AB"`）
    5. `ZeroPageXLda`（$B5 → `"LDA $AB,X"`）
    6. `ZeroPageYLdx`（$B6 → `"LDX $AB,Y"`）
    7. `IndirectXLda`（$A1 → `"LDA ($AB,X)"`）
    8. `IndirectYLda`（$B1 → `"LDA ($AB),Y"`）
    9. `AbsoluteLda`（$AD 0xCD 0xAB → `"LDA $ABCD"`）
    10. `AbsoluteXLda`（$BD → `"LDA $ABCD,X"`）
    11. `AbsoluteYLda`（$B9 → `"LDA $ABCD,Y"`）
    12. `IndirectJmp`（$6C → `"JMP ($ABCD)"`）
    13. `RelativeBneForward`（$D0 0x05 → `"BNE $8007"`，0x8002+5）
    14. `RelativeBneBackward`（$D0 0xFB → `"BNE $7FFD"`，0x8002-5）
- `tests/CMakeLists.txt` 註冊 `test_disassembler`
- 建置零警告、**33/33 測試通過**（19 + 14）

## 討論重點

### 1. FakeMapper——測試替身（test double）

Disassembler 測試需要「餵已知 bytes 進 bus 再斷言輸出」，但真 ROM 無法精準控制內容。解法：造一個假的 Mapper（只服務測試），接線 FakeMapper → **真品 CpuBus** → 被測對象 Disassembler。中間層用真品很重要——bus 的分診邏輯（RAM/SRAM/ROM）也順便被測到了；假的只有最外圈那層「卡帶」。

### 2. 座標系轉換的分工（本 stage 最核心的觀念）

`cpu_bus.cpp` 在轉手給 mapper 前已先 `addr - 0x8000`，**mapper 收到的是 0-$7FFF 的 PRG 偏移**。分工：
- bus：CPU 位址 → PRG 偏移的轉換（$8000-$FFFF 範圍判斷與平移）
- mapper：只管 bank 切換/鏡像（Mapper000 同慣例）

FakeMapper 的兩個函式因此活在**不同座標系**：
- `read_prg(addr)`：caller 是 bus，進來的已是轉換後座標——直接 `prg_[addr]`
- `set_prg(addr, data)`：caller 是測試，進來的是 CPU 座標——要自己減 $8000

口訣：**進來的是 bus 座標直接用；進來的是 CPU 座標減 $8000**。`set_prg` 設計成吃 CPU 座標是刻意的——測試端 `set_prg(0x8000, …)` 與 `bus.read(0x8000)` 同一座標系，測試更好寫；轉換只做一處。

### 3. 分批施工的 pattern 複用

14 個測試全是同一個模板的複製變化（換 opcode、換期望字串、換 next_addr），分五批給（接線 1 個 → Acc/Imm 2 個 → 零頁 3 個 → 間接 2 個 → 絕對 4 個 → Relative 2 個）。寫到絕對定址批時 pattern 已內化，3-byte 指令（塞三格、next_addr=0x8003、little-endian 存放序）一次到位零 bug。

### 4. ZeroPageY 用 LDX 不用 LDA

6502 沒有 `LDA $AB,Y` 這個組合（Stage 2 填表遇過：ZeroPageY 只出現在 STX/LDX/LAX）——測試選 opcode 要挑「真的存在」的組合，不然表裡查出來的就不是想測的模式。

### 5. Relative 測前進也要測後退

`target = next_addr + (int8_t)offset` 的精髓在 `static_cast<int8_t>`：
- 前進（offset=0x05）：正數直接加，測不出轉型的存在
- 後退（offset=0xFB=-5）：**沒轉有號就露餡**——uint8_t 直接加會變 `0x8002 + 0xFB = 0x80FD`，測試直接紅

一對測試釘住「符號擴充」這個最容易寫錯的點。

### 6. IndirectX / IndirectY 的括號位置

`($AB,X)` X 在括號內（取指標前先加）vs `($AB),Y` Y 在括號外（拿到指標後才加）——Stage 6 教的語意直接化成兩行斷言字串，貼反立刻現形。

## 遇到的問題（AI 四連錯 + stack smashing，全紀錄）

1. **CMake 沒註冊 → 假綠燈**：第一個測試貼完跑出 19/19 全綠，但新測試**從沒被編譯過**——AI 檢查時誤稱「註冊齊全」其實看漏 `tests/CMakeLists.txt`。教訓：**加新測試後數字沒 +1 就是沒跑到**，全綠可能是假的
2. **include 路徑錯**：AI 骨架寫 `"src/nes/cpu_bus.h"`，但根 CMakeLists 是 `target_include_directories(nes_lib PUBLIC src)`，include root 是 `src/`——正確是 `"nes/cpu_bus.h"`（對照 test_instruction_table.cpp 慣例可免錯）
3. **漏 `nes::` 前綴**：所有類別都在 namespace nes，`class FakeMapper : public nes::Mapper`、`nes::CpuBus`、`nes::Disassembler` 要寫滿（對照 test_cpu_bus.cpp 慣例）
4. **位址雙重減（AI 引導錯方向）**：AI 第一時間讓使用者在 `read_prg` 也減 $8000——但 bus 已經減過了，mapper 再減就是雙重減
5. **stack smashing detected**：使用者把減號改錯邊——`set_prg` 用 `prg_[addr]`（索引 0x8000=32768 界外寫 32KB 陣列）、`read_prg` 留減號（uint16_t 溢位繞回 0x8000 界外讀）。stack smash = 偵測到堆疊緩衝區越界寫——FakeMapper 的 32KB `prg_` 是 TEST 區域變數，放在堆疊上，越界寫就是砸堆疊。症狀是測試印出 `BRK`（讀到界外垃圾 0x00）而不是直接崩潰
6. 小 review 抓過：漏 `#include <array>`（誰用誰 include）、`set_prg` 定義漏掛 `FakeMapper::` 前綴（變成自由函式摸不到 private member，編譯爆）、使用者貼到半途狀態

## Review 觀察

- 測試本體用 `const std::string text = ...` 接回傳值，C++17 保證 copy elision，零成本
- 可選改進：測試檔直接用了 `std::string`，照「誰用誰 include」可以自己補 `#include <string>`（目前靠 `disassembler.h` 間接拿到，能用但不符合自家慣例的字面要求）
- 14 個測試重複 FakeMapper/CpuBus/Disassembler 三行接線——GoogleTest 有 fixture（`TEST_F`）可以消掉重複，但 14 個測試的樣板還很淺，先不抽（rule of three：重複三次以上且會持續生長才抽）

## 學習心得

- **全綠要看得起來才可信**：「測試數有沒有 +1」比「有沒有紅燈」更基本——沒跑到的測試永遠是綠的
- **座標系轉換只做一處、做在分界線上**：bus/mapper 的分界線就是「CPU 座標」和「PRG 偏移」的國界，跨界處轉一次，兩邊各自活在簡單的世界裡。忘記「對方已經轉過了」就是雙重減的地雷
- **讀越界不會立刻崩，會安靜地錯**：stack smash 的症狀是「讀到 0 解出 BRK」這種看似正常的錯誤輸出——記憶體錯誤常見的模樣不是爆炸，是裝沒事
- **負數測試是轉型程式碼的照妖鏡**：`static_cast<int8_t>` 這行只有遇到負 offset 才有意義，只測正數等於沒測到它的存在
- 本 stage AI 出錯四次全部被逐一抓出（假綠燈、include 路徑、namespace、位址轉換）——照著慣例檔案對照、追進實際程式碼讀 bus 的減法，是抓出這些錯的手段

## 下一關預告

Stage 12：整合測試 + 收尾——對 nestest.nes 三向量反組譯驗證 RESET=`SEI`、確認 33 測試全綠、SPEC 全打勾、日誌補齊。
