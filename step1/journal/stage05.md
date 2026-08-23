---
layout: default
title: Stage 5：CpuBus 骨架 (cpu_bus.h / cpu_bus.cpp)
---

# Stage 5 日誌：CpuBus 骨架 (cpu_bus.h / cpu_bus.cpp)

## 日期
2026-07-11

## 狀態
✅ 完成

## 完成事項
- 定義 `class CpuBus` ✅
- private 成員：`ram_`（2KB std::array）、`sram_`（8KB std::array）、`mapper_`（raw pointer）✅
- 建構子 `explicit CpuBus(Mapper* mapper)` ✅
- 宣告 `read(uint16_t) const` + `write(uint16_t, uint8_t)` ✅
- 更新 CMakeLists.txt（加 `cpu_bus.cpp`）✅
- Build 通過 ✅

## 最終程式碼

### cpu_bus.h（23行）
```cpp
#pragma once

#include <array>
#include <cstdint>

#include "mapper.h"

namespace nes {

class CpuBus {
public:
    explicit CpuBus(Mapper* mapper);
    uint8_t read(uint16_t addr) const;
    void write(uint16_t addr, uint8_t data);

private:
    std::array<uint8_t, 2048> ram_{};
    std::array<uint8_t, 8192> sram_{};
    Mapper* mapper_;
};

} // namespace nes
```

### cpu_bus.cpp（9行）
```cpp
#include "cpu_bus.h"

namespace nes {

CpuBus::CpuBus(Mapper* mapper)
    : mapper_(mapper) {
}

} // namespace nes
```

## 討論重點

### 1. `std::array` 的 `{}` 初始化
- `ram_{}` = 值初始化（value-initialization），整個陣列清零
- 不寫 `{}` = 垃圾值（記憶體殘留），寫了 = 全部 0
- `{}` 可用在所有型別：`int x{}` = 0、`bool b{}` = false
- 原版 C 用 `memset` 做同樣的事

### 2. 為什麼 CpuBus 的 `mapper_` 用 raw pointer 不用 unique_ptr
- **誰擁有 Mapper**：Famicom 用 `unique_ptr` 擁有，CpuBus 只是借用
- 如果兩個 unique_ptr 指向同一個 Mapper → delete 兩次 → 崩潰
- unique_ptr 規則：一個物件只能被一個 unique_ptr 擁有
- 業界慣例：unique_ptr 表達擁有權，raw pointer 表達非擁有的存取

### 3. 為什麼 make_mapper 的 unique_ptr 不會衝突
- make_mapper 是「創造者」，創造完就交出去（move 轉移所有權）
- 資料流向：make_mapper（創造 unique_ptr）→ Famicom（接住成為唯一擁有者）→ CpuBus（拿到 raw pointer 借用）
- 從頭到尾只有一個 unique_ptr 擁有 Mapper

### 4. 為什麼 Famicom 要用 unique_ptr 不用 raw pointer
- raw pointer 要自己寫 `delete` → 忘記=洩漏、例外=洩漏、copy/move 要手動管理
- unique_ptr = RAII 自動釋放，不用寫解構子，例外安全，Rule of Zero
- 總要有人負責 delete，用 unique_ptr 讓編譯器幫你做

### 5. CpuBus 用 raw pointer 為什麼不會洩漏
- 規則：誰 `new` 的，誰負責 `delete`
- CpuBus 沒有 `new` Mapper，只是借用，解構時什麼都不用做
- 比喻：Famicom = 租車公司（擁有車負責報廢），CpuBus = 借車客人（用車不負責報廢）

### 6. 所有權模型總覽
```
make_mapper()  ──創造──→  unique_ptr<Mapper>  ──move──→  Famicom（擁有者）
                                                          │
                                                          └──raw ptr──→ CpuBus.mapper_（借用者）
```
- 創造者（make_mapper）：創造完交出去，不持有
- 擁有者（Famicom）：unique_ptr，負責生命週期
- 借用者（CpuBus）：raw pointer，只讀不擁有

## Review 建議
- 程式碼正確，結構清晰
- raw pointer vs unique_ptr 的所有權設計是業界最佳實踐
- `ram_{}` 和 `sram_{}` 的 `{}` 初始化確保記憶體清零
- `explicit` 防止隱式轉換，跟 step0 一致

## 學習心得
Stage 5 的核心是理解「所有權」vs「借用」的區別。CpuBus 持有 `mapper_` 但不擁有它 — 用 raw pointer 表達「我借用，不負責銷毀」。Famicom 用 unique_ptr 表達「我擁有，我負責銷毀」。這是業界 C++ 處理物件生命週期的標準模式：unique_ptr 表達擁有權，raw pointer/reference 表達非擁有的存取。`std::array` 的 `{}` 初始化則是 C++11 之後清零記憶體的簡潔寫法，取代了原版 C 的 `memset`。
