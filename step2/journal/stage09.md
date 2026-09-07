---
layout: default
title: "Stage 09 — main.cpp：反組譯三個向量"
---

# Stage 09 — main.cpp：反組譯三個向量

> 日期：2026-09-06
> 對應 SPEC Stage 9：載入 ROM → 讀三向量 → 反組譯第一條指令並輸出

## 完成事項

- `main.cpp`（32 → 33 行）：第 25-27 行三行輸出各加 `": " << famicom.disassemble(向量)`，一行搞定一個向量的反組譯輸出
- 建置零警告、17/17 測試通過
- 實際輸出（nestest.nes）：
  ```
  NMI Vector:   $C5AF: PHA
  RESET Vector: $C004: SEI
  IRQ Vector:   $C5F4: RTI
  ```

## 討論重點

### 1. `std::hex` 是黏的（sticky formatting）

第 25 行設了 `std::hex << std::uppercase`，26/27 行不用再設——iostream 的格式旗標一旦設定持續生效到被改掉為止。這是雙刃劍：省重複、但「這行為什麼是 hex」的答案可能在很遠的上一行。三行連續輸出場景下剛好順用。

### 2. SPEC 預期輸出 `???` 是錯的——實際是 PHA / RTI

寫 SPEC 時猜「NMI/IRQ 向量指向的地方可能是資料、解不出指令、印 `???`」。實際驗證：

- 查 nestest.nes 的實際 bytes：`$C5AF` = 0x48 = **PHA**、`$C5F4` = 0x40 = **RTI**、`$C004` = 0x78 = **SEI**
- 兩個原因讓 `???` 不可能出現：
  1. 我們的表 **256 格全滿**（含 unofficial），任何 byte 都解得出 mnemonic
  2. 這兩個位置本來就真的是指令——NMI/IRQ handler 的入口

SPEC 的預期輸出已修正為實際值。教訓：預期輸出該用「查證後的資料」寫，不該用「感覺」寫。

### 3. 為什麼 NMI 開場是 PHA、IRQ 開場是 RTI

中斷處理常式（interrupt handler）的標準長相：

- **進場**：PHA / PHX / PHY——把暫存器推進堆疊「保護現場」，因為 handler 執行過程會用到 A/X/Y，不保存就會破壞被打斷的主程式狀態
- **收尾**：PLA / PLX / PLY 還原現場 → RTI（ReTurn from Interrupt）從堆疊彈回 PC 和 P，回到被打斷的地方

nestest 的 NMI handler 第一條 PHA 是教科書開場；IRQ handler 直接 RTI（這個 ROM 的 IRQ 沒做事，立刻返回）。

### 4. 對照原版 `sfc_fc_disassembly()`

原版（`sfc_cpu.c`）：

- 「暴力讀 3 字節」——不管指令多長，op/a1/a2 一次讀齊（多讀的無所謂）
- `memset(buf, ' ', 8)` 手工填縮排 + `sfc_btoh` 手拼 hex 到固定 `char buf[]`
- 呼叫端要自己準備 buffer、自己印

我們的版本：`famicom.disassemble(addr)` 回 `std::string`，讀多少 operand 由 `info.mode` 決定（只讀需要的），buffer 管理全部交給 std::string。三行向量輸出的改動就是三行——原版要動 buffer 生命週期的改動絕不只三行。

## 遇到的問題

無。本 stage 只有三行改動，零 bug。

## Review 觀察

- `famicom` 是非 const 物件，呼叫 const 成員函式 `disassemble()` 合法（const 承諾單向：const 物件只能呼叫 const 函式；非 const 物件兩種都能呼叫）
- 三行輸出格式對稱，`std::endl` 齊一

## 學習心得

- **整合的價值**：Stage 4-8 一路把零件做好（Disassembler → Famicom 便捷方法），最後 main.cpp 的「大結局」只剩三行——好介面讓整合變得無聊，這是好事
- **預期輸出要用查證的**：SPEC 裡的範例輸出寫錯（???），根因是當初用推測寫預期。測試先行觀念的變體：預期值本身要可靠，不然測試比對的是錯的答案
- 讀 ROM bytes 驗證輸出（xxd 看原始資料）是驗證反組譯器最直接的手段——Stage 11 的測試會把這件事自動化

## 下一關預告

Stage 10：指令表測試（`test_instruction_table.cpp`）——表長度、錨點 opcode、256 格全查得到，用迴圈批次驗證而不是手寫 256 個測試。
