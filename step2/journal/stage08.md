---
layout: default
title: "Stage 08 — Famicom 擴展：組裝 CPU 與 Disassembler"
---

# Stage 08 — Famicom 擴展：組裝 CPU 與 Disassembler

> 日期：2026-09-06
> 對應 SPEC Stage 8：Famicom 加入 `Cpu cpu_` 成員 + `disassemble()` 便捷方法

## 完成事項

- `famicom.h`（28 → 30 行）：
  - `#include "cpu.h"` + `#include <string>`
  - getter `const Cpu& cpu() const;`
  - 便捷方法 `std::string disassemble(uint16_t addr) const;`
  - 成員 `Cpu cpu_;`（宣告在 `bus_` 之後）
- `famicom.cpp`（33 → 39 行）：
  - 建構子初始化清單加 `cpu_(bus_)`（`bus_(mapper_.get())` 結尾 `{` 改 `,`）
  - `cpu()` getter 實作
  - `disassemble()` 實作：內造 `Disassembler d(bus_); return d.disassemble(addr);`
- 建置零警告、17/17 測試通過、主程式輸出正常

## 討論重點

### 1. 成員初始化順序 = 宣告順序（不是初始化清單的書寫順序）

`Cpu` 的建構子是 `explicit Cpu(CpuBus& bus)`，成員 `cpu_` 要綁 `bus_`。C++ 規則：**成員依宣告順序初始化**，初始化清單怎麼排不影響。若 `cpu_` 宣告在 `bus_` 之前，`cpu_(bus_)` 執行時 `bus_` 還沒建構——綁到不存在的物件，UB。`-Wreorder` 會抓初始化清單書寫序跟宣告序不一致的寫法，但「引用綁到未建構成員」要靠自己守紀律：**被依賴的成員宣告在前面**。

Famicom 的依賴鏈：`rom_info_` → `mapper_`（用 rom_info_）→ `bus_`（用 mapper_）→ `cpu_`（用 bus_）——一條線排下來。

### 2. Disassembler 進 Famicom 的兩種設計（A vs B）

- **A. 存成成員** `Disassembler disassembler_;`——放大鏡常掛在機器上，呼叫端 `fc.disassembler().disassemble(addr)`
- **B. 便捷方法，用時現造**——`fc.disassemble(addr)` 一個動詞，內部造即丟

**業界答案：B**。理由：

1. Famicom 的成員櫃該放**零件**（bus/cpu/mapper——機器運轉必須一直在的東西）；Disassembler 是**外部工具**（機器不靠它運轉，是人拿來看機器的放大鏡）——身分不同，不該佔零件櫃
2. Disassembler 零狀態（只有一個 `const CpuBus&`），現造成本趨近零；成員的價值在保存昂貴或需延續的狀態，這裡沒有
3. 呼叫端 API 更順

口訣：**零件進櫃子，工具用完即丟**。A 什麼時候才對：工具變貴（有快取、有狀態、建構重）的時候。

### 3. 便捷方法為什麼只開一參數版（不開 next_addr 版）

Facades 不該複製底層 API 的每一個進口：

- 現在唯一呼叫者 main.cpp 只印三個向量各一條，用不到 next_addr——開沒人用的多載 = YAGNI
- 真需要連續反組譯的呼叫端可以直接用 `Disassembler` 本體
- 每個便捷方法都是 API 面積，之後改簽名都背相容性包袱——便捷層越薄越好

未來 main.cpp 真要跑反組譯迴圈時，再於 famicom.h/.cpp 加 `disassemble(uint16_t addr, uint16_t& next_addr)` 多載（內部同樣轉發），介面對稱、實作薄轉發——facade 標準長相。**需求出現前不先蓋**。

### 4. 宣告裡的參數名是文件

`std::string disassemble(uint16_t) const;` 能編譯，但 `uint16_t addr` 才是給讀 header 的人看的——不用翻 .cpp 就知道「這是指令位址」。宣告與實作的參數名保持一致。

## 遇到的問題

- 功能面零 bug。Review 抓到兩個風格小瑕疵（使用者自修後 build 模式複查確認）：
  1. `const{` 少空格（應為 `const {`，與檔內其他函式一致）
  2. header 參數名 `addr` 被省略

## Review 觀察

- `disassemble()` 的 `Disassembler d(bus_)` 是 local——每次呼叫建構/解構，零狀態工具的標準用法，編譯器連inline 都不用考慮就會最佳化掉
- famicom.cpp 的 include 順序（自家 header → `<utility>` → `disassembler.h`）符合專案慣例

## 學習心得

- **組合（composition）的施工順序有感**：加一個會綁引用的成員，宣告位置、初始化清單、順序紀律三件事連動——這是 C++ 物件組裝跟其他語言最不一樣的地方
- **Facade 的厚度是一個設計決策**：開什麼門、不開什麼門，考量的是「現在的呼叫者要什麼」而不是「底層有什麼」
- 工具 vs 零件的心智模型：有狀態、貴、機器運轉需要 → 成員；無狀態、便宜、人用的 → 便捷方法現造

## 下一關預告

Stage 9：main.cpp 反組譯三個向量——NMI/RESET/IRQ 入口處的第一條指令，step2 的最終輸出終於登場。
