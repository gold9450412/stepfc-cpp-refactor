---
layout: default
title: 首頁
---

# StepFC C++ 重構日誌

將 StepFC 的 Step0（讀取 NES ROM）用現代 C++ 重新實作的完整開發記錄。

## 規格書

- [Step0 C++ 重構規格書](SPEC.html)

## 開發日誌

| Stage | 標題 | 連結 |
|-------|------|------|
| 0 | 建置環境 | [查看](journal/stage00.html) |
| 1 | 錯誤碼定義 (error.h) | [查看](journal/stage01.html) |
| 2 | NES 檔頭結構 (nes_header.h / nes_header.cpp) | [查看](journal/stage02.html) |
| 3 | ROM 資訊類別 (rom_info.h / rom_info.cpp) | [查看](journal/stage03.html) |
| 4 | 讀取介面 — 抽象類別 (rom_loader.h) | [查看](journal/stage04.html) |
| 5 | 檔案讀取 — 開檔與驗證 (file_rom_loader.h / .cpp 前半) | [查看](journal/stage05.html) |
| 6 | 檔案讀取 — 讀取 PRG/CHR 資料 (file_rom_loader.cpp 中半) | [查看](journal/stage06.html) |
| 7 | 解析 Flags 填入 ROM 資訊 (file_rom_loader.cpp 後半) | [查看](journal/stage07.html) |
| 8 | 模擬器主體 (famicom.h / famicom.cpp) | [查看](journal/stage08.html) |
| 9 | 主程式 (main.cpp) | [查看](journal/stage09.html) |
| 10 | 測試環境建置 (Google Test) | [查看](journal/stage10.html) |
| 11 | ROM 資訊測試 (test_rom_info.cpp) | [查看](journal/stage11.html) |
| 12 | 檔案讀取測試 (test_file_rom_loader.cpp) | [查看](journal/stage12.html) |
