---
layout: default
title: "Stage 12 — 整合測試 + 收尾（step2 完結）"
---

# Stage 12 — 整合測試 + 收尾（step2 完結）

> 日期：2026-09-07
> 對應 SPEC Stage 12：nestest 三向量反組譯驗證 RESET=`SEI`、全部測試通過、SPEC 全勾、日誌補齊

## 完成事項

- `tests/test_integration.cpp` 新增第 3 個 TEST `DisassembleNestestResetVector`（56 行）：
  - 開頭沿用 `ReadNestestVectors` 的前置（載 nestest.nes → `ASSERT_EQ` Ok → 建 Famicom）
  - 讀 RESET 向量（$FFFC/$FFFD 組 16-bit）→ `EXPECT_EQ(famicom.disassemble(reset), "SEI")`
- 建置零警告、**34/34 測試通過**（33 + 1）
- step2 全部 12 個 Stage 完結

## 討論重點

### 1. 端到端測試（end-to-end）——一個斷言驗整條生產線

這個 TEST 只有一行斷言 `EXPECT_EQ(..., "SEI")`，但它踩過的卻是**整條鏈**：FileRomLoader（讀檔/iNES 解析）→ RomInfo → Famicom（組裝）→ CpuBus（分診 $8000-$FFFF）→ Mapper000（bank 映射）→ Disassembler（lookup + 格式化）。任何一環壞掉，這行都會紅。這是它與 Stage 11 單元測試的根本差異：

- **單元測試（test_disassembler.cpp）**：FakeMapper 控制輸入，驗 Disassembler 本身的邏輯
- **整合測試（test_integration.cpp）**：真檔案真零件，驗「組裝起來的整機」

分類標準不是「放在哪個檔案」，是「**測的是誰**」。

### 2. 為什麼只驗 RESET 一個向量

SPEC 的驗收就只要 RESET=`SEI`。NMI=`PHA`、IRQ=`RTI` 不是不重要——是**已經被涵蓋**：Stage 9 在 main.cpp 輸出驗過（且當時查證過 ROM 實際 bytes $C5AF=0x48、$C5F4=0x40）。測試不必重複斷言別處已驗證的事實，但 RESET 是「模擬器開機後第一條會執行的指令」，有獨立的符號意義，值得在測試裡釘一根樁。

### 3. 權威 log 對答案的誠實範圍（第三層驗證的真相）

Stage 10 留下待辦「nestest 權威 log 全量對答案」。收尾時面對現實：**nestest.log 是執行流（follows execution flow）**——每一行是 CPU 實際執行到的指令，跟著 branch/jump 跳著走。要走出同樣的路徑，前提是 CPU 會執行、會跳轉——那是 **step3** 的事。step2 只有反組譯器，沒有執行引擎，無法重演 log 的路徑。

所以 step2 的「對答案」等價物是**向量級抽查**：$C004 的 byte 0x78（Stage 9 查證過）→ `SEI`，與 nestest.log 自動模式從 $C000 起、開頭就是 SEI 一致。全量 256 格 opcode 的正確性，在 step2 由「迴圈（合法）+ 8 錨點（整批錯）」兩層把關，第三層（全 log 比對）**正式移交給 step3**——屆時 CPU 逐條執行 nestest 並逐行比對 log，才是那個測試真正的家。

> 教訓：寫計畫時把「對權威 log」排進 step2 是過度承諾——log 驗證的前提（執行流）在 step3 才成立。待辦要排在使用它的能力就位**之後**。

### 4. `static_cast<int8_t>` 在產品碼，不在測試——測試是照妖鏡

複習 RelativeBneBackward 時曾找不到 `static_cast<int8_t>` 在哪：它在 `disassembler.cpp:37`（**被測的產品碼**），測試裡看不到。分工：

```cpp
// disassembler.cpp:37 —— 產品碼做符號解讀
const int8_t offset = static_cast<int8_t>(bus_.read(addr + 1));
```

測試餵 0xFB、斷言 `$7FFD`，就是在**驗證**這行 cast 有沒有做對。沒有它的話 `0x8002 + 0xFB = 0x80FD`（往 +251 跳），測試立刻爆炸。$7FFD 的手算過程：

- 0xFB = 1111_1011（二補數）→ 最高 bit 1 → 負數 → 251 - 256 = **-5**
- target = next_addr + offset = 0x8002 - 5 = **0x7FFD**（往回跳 5 格：$8001→$8000→$7FFF→$7FFE→$7FFD）

### 5. 新測試放在 test_integration.cpp 的理由

一開始使用者問「哪個 cpp」——答案是放 `test_integration.cpp` 而非 `test_disassembler.cpp`。理由見討論重點 1：這個測試的主體是「真檔案 + 全零件」，不是「控制輸入驗單元」。放對分類，未來找測試才會直覺。

## 遇到的問題

- 無程式碼 bug（貼上一次到位、編譯零警告、34/34 綠）。
- 小插曲：使用者一度不確定要貼到哪個檔案——提醒：以「測的是誰」分類，整合測試進 test_integration.cpp。

## Review 觀察

- `DisassembleNestestResetVector` 與 `ReadNestestVectors` 有約 8 行重複前置（載檔、建 Famicom）。若未來第三個、第四個向量級測試出現，可抽 TEST_F fixture（`NestestFamicomTest` 提供 `make_famicom()` helper）。目前兩個測試，重複一次可接受——rule of three，等第三個再抽。
- 測試名 `DisassembleNestestResetVector` 名稱即規格（測什麼、測誰一目了然），慣例正確。

## 學習心得

step2（6502 反組譯器）完結。回望 12 個 Stage 的產出：

- **資料層**：Instruction/InstructionInfo enums、256 格 constexpr 指令表（含 19 個 unofficial opcode）
- **格式層**：13 種定址模式的輸出（`to_hex` 查表法、little-endian 儲存序 vs 人類閱讀序、Relative 跳躍目標計算）
- **狀態層**：CpuRegisters + reset()（$FD/$0x34 的硬體指紋）
- **組裝層**：Famicom facade（零件進櫃子、工具用完即丟）
- **測試層**：三層防護（迴圈合法 → 錨點正確 → 權威資料抽查）+ FakeMapper 測試替身 + 端到端整合測試，34 個測試

測試數軌跡：17（step1 帶入）→ 19（+指令表 2）→ 20（+FakeMapper 接線 1）→ 33（+定址模式 13 批次）→ **34**（+向量驗證 1）。

下一個是 **step3：CPU 指令執行**——模擬器最大的一關（256 條指令真的動起來），nestest.log 全量對答案的承諾也在那裡兌現。反組譯器（step2）在那之前先上工：step3 debug 時，反組譯 output 就是「CPU 現在在跑什麼」的顯微鏡。
