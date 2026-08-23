---
layout: default
title: Step1 重構日誌
---

# Step1 C++ 重構日誌

將 StepFC 的 Step1（CPU 記憶體位址空間 + Mapper000 NROM + 中斷向量）用現代 C++17 重新實作的完整開發記錄。

- [回 Step0 首頁](../index.html)

## 規格書

- [Step1 C++ 重構規格書](SPEC.html)

## 開發日誌

| Stage | 標題 | 連結 |
|-------|------|------|
| 0 | 環境設定 + 複製 step0 程式碼 | [查看](journal/stage00.html) |
| 1 | 中斷向量常數 (cpu_vectors.h) | [查看](journal/stage01.html) |
| 2 | 抽象 Mapper 類別 (mapper.h) | [查看](journal/stage02.html) |
| 3 | Mapper000 NROM (mapper000.h / mapper000.cpp) | [查看](journal/stage03.html) |
| 4 | Mapper Factory 函式 (mapper_factory.h / mapper_factory.cpp) | [查看](journal/stage04.html) |
| 5 | CpuBus 骨架 (cpu_bus.h / cpu_bus.cpp) | [查看](journal/stage05.html) |
| 6 | CpuBus read() 實作 (cpu_bus.cpp) | [查看](journal/stage06.html) |
| 7 | CpuBus write() 實作 (cpu_bus.cpp) | [查看](journal/stage07.html) |
| 8 | Famicom 擴展 (famicom.h / famicom.cpp) | [查看](journal/stage08.html) |
| 9 | main.cpp — 讀中斷向量 | [查看](journal/stage09.html) |
| 10 | CpuBus 測試 (tests/test_cpu_bus.cpp) | [查看](journal/stage10.html) |
| 11 | Mapper000 測試 (tests/test_mapper000.cpp) | [查看](journal/stage11.html) |
| 12 | 整合測試 + 收尾 (tests/test_integration.cpp) | [查看](journal/stage12.html) |
