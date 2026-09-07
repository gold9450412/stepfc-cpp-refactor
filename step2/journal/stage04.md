---
layout: default
title: "Stage 04 — Disassembler 骨架（`disassembler.h` / `disassembler.cpp`）"
---

# Stage 04 — Disassembler 骨架（`disassembler.h` / `disassembler.cpp`）

日期：2026-09-06

## 完成事項

- 建立 `src/nes/disassembler.h`（23 行）：
  - `class Disassembler`，建構子 `explicit Disassembler(const CpuBus& bus)`
  - 兩個 `disassemble` 多載：兩參數版（`next_addr` 輸出用）+ 簡易版
  - 成員 `const CpuBus& bus_`（唯讀語意）
  - `CpuBus` 前向宣告，標頭只 include `<cstdint>` + `<string>`
- 建立 `src/nes/disassembler.cpp`（24 行）：
  - 兩參數版：讀 `bus_.read(addr)` 得 opcode → `lookup(opcode)` 得 `InstructionInfo` → `next_addr = addr + info.length` → 回傳 `to_string(info.instruction)`
  - 簡易版：宣告 `uint16_t ignored` 後委派給兩參數版，邏輯只有一份
- `CMakeLists.txt` 的 `nes_lib` 加入 `src/nes/disassembler.cpp`
- 驗證：乾淨重編**零警告**，17/17 測試全過，`step2_cpp` 執行正常

本 stage 只輸出 mnemonic（`SEI`），operand 格式化（`#$AB`、`$ABCD,X`…）是 Stage 5~7。

## 討論重點

### 1. 為什麼參數是 `const CpuBus&`

兩個決定各有理由：
- **`&` 引用**：CpuBus 內含 RAM 2KB + SRAM 8KB，傳值會複製一整坨，借看而已太浪費
- **`const`**：反組譯器的本質是「看不是改」，加 const 後 `bus_.write()` 直接編譯失敗

對照 `Cpu(CpuBus& bus)` **沒有 const**——CPU 真的會寫 RAM（STA 等指令）。**有沒有 const 是在描述類別的本性**。`const T&` 是業界「唯讀大物件參數」的標準句型。

### 2. 尾端 const（`() const`）vs 前綴 const（`const CpuBus&`）的差別

使用者主動問了這題，結論：
- `const CpuBus&`：const 修飾**資料**——綁上去的物件唯讀
- `() const`：const 修飾**這個成員函式**——承諾不改 `*this` 的任何成員。只能加在成員函式上（自由函式沒有成員可改）
- 附帶好處：`() const` 讓 `const Disassembler&` 的持有者也能呼叫，否則 const 物件連反組譯都不能做

### 3. 兩個 `disassemble` 多載與 `ignored` 的用途

兩參數版要求呼叫者提供「有名字、有位址」的變數接收 `next_addr`（`uint16_t&` 引用輸出參數不能塞字面值）。簡易版的呼叫者不在乎下一步，於是自己生 `uint16_t ignored = 0;` 當丟棄容器，讓邏輯**只存在一份**（複製邏輯 = 將來改兩處遲早漏）。

比喻：便當店每份都附筷子（函式內部一定算 next_addr），簡易版客人拿到便當把筷子扔了（呼叫者不用這個輸出）。

### 4. `next_addr` 的兩種使用者

- **反組譯迴圈**：指令長度 1~3 bytes 不固定，連續印一整段程式必須知道「這步多寬」才能走下一步。之後的 debugger 就是這樣一路走
- **Relative 定址**：`BNE` 的跳躍目標 = 指令位址 + 2 + 偏移，那個 2 就是 `next_addr - addr`（Stage 5 實作）

### 5. 「每條指令都要算 next_addr」vs「呼叫者要不要收」

使用者追問「什麼指令不用知道下一步？應該都要吧」——釐清：**內部**每次都算（`info.length` 永遠參與），差別在**外部呼叫者**要不要這個輸出。多載的選擇權在呼叫端。

### 6. `(void)addr;` 鷹架

`-Wextra` 對「宣告了卻沒用」的參數發警告。空殼階段用 `(void)x;` 表達「我故意不用，不是忘了」，是業界標準暗號。填肉後就刪掉。

### 7. `const InstructionInfo& info = lookup(opcode);`

`lookup` 回傳表裡那格的**引用**（不是複製），用 const 引用接，一路零複製。表是 `constexpr` 放 .rodata，runtime 查表就是一次陣列索引。

## 遇到的問題

- `--clean-first` 清掉 nes_lib 後，依賴它的 `step2_cpp` 執行檔暫時消失，重編 main target 後恢復。建置驗證流程的小插曲，無實害
- 本 stage 冒出兩個「使用者主動問 const」的好問題（尾端 const 的意義、參數 const 的理由），都是 C++ 新手到業界水準的分水嶺觀念

## Review 建議

- 程式碼乾淨，無需修改
- 簡易版委派兩參數版的寫法是教科書級的多載重用，保持
- Stage 5 開始會碰字串格式化，屆時會介紹 `<sstream>` 或手拼 hex 的選擇

## 學習心得

1. **依賴注入**：Disassembler 不自己開檔讀 ROM，誰有 bus 誰塞進來——測試時塞假的 bus 就能測（Stage 11 的 FakeMapper 會用到這個設計）
2. **const 是語意文件**：`const CpuBus&` vs `CpuBus&` 的差別，讓呼叫者光看簽名就知道這個類別會不會改他的東西
3. **單一邏輯來源**：兩個多載委派而非複製，是「DRY 原則」的具體實踐
4. **table-driven 的甜頭**：disassemble 主體只有四行——讀、查、算、印。256 格表在 Stage 2 填好後，這裡幾乎不用做事
