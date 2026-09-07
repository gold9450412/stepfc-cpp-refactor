---
layout: default
title: Step2 重構日誌
---

# Step2 C++ 重構日誌

將 StepFC 的 Step2（6502 反組譯器：指令表 + 13 種定址模式格式化 + CPU 暫存器 + Disassembler）用現代 C++17 重新實作的完整開發記錄。

- [回首頁](../index.html)

## 規格書

- [Step2 C++ 重構規格書](SPEC.html)

## 開發日誌

| Stage | 標題 | 連結 |
|-------|------|------|
| 1 | 指令與定址模式 enum (instruction.h / instruction.cpp) | [查看](journal/stage01.html) |
| 2 | 指令表 (instruction_table.h / instruction_table.cpp) | [查看](journal/stage02.html) |
| 3 | CPU 暫存器 (cpu.h / cpu.cpp) | [查看](journal/stage03.html) |
| 4 | Disassembler 骨架 (disassembler.h / disassembler.cpp) | [查看](journal/stage04.html) |
| 5 | 反組譯輸出 — 無 operand 模式 | [查看](journal/stage05.html) |
| 6 | 反組譯輸出 — 1 byte 位址模式 | [查看](journal/stage06.html) |
| 7 | 反組譯輸出 — 2 byte 位址模式 | [查看](journal/stage07.html) |
| 8 | Famicom 擴展：組裝 CPU 與 Disassembler | [查看](journal/stage08.html) |
| 9 | main.cpp — 反組譯三個向量 | [查看](journal/stage09.html) |
| 10 | 指令表測試 (tests/test_instruction_table.cpp) | [查看](journal/stage10.html) |
| 11 | Disassembler 測試 (tests/test_disassembler.cpp) | [查看](journal/stage11.html) |
| 12 | 整合測試 + 收尾（step2 完結） | [查看](journal/stage12.html) |

> Stage 0（環境設定 + 複製 step1 程式碼）未另寫日誌，從 Stage 1 開始記錄。
