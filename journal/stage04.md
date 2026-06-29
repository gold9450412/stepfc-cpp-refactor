---
layout: default
title: Stage 4：讀取介面 — 抽象類別 (rom_loader.h)
---

# Stage 4 日誌：讀取介面 — 抽象類別 (rom_loader.h)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/rom_loader.h` ✅
- 定義抽象類別 `class RomLoader` ✅
- 純虛擬函數 `virtual ErrorCode load(RomInfo& out) = 0;` ✅
- 虛擬解構子 `virtual ~RomLoader() = default;` ✅
- 不需要 `free()` — RAII 自動處理 ✅

## 最終程式碼
```cpp
#pragma once

#include "error.h"
#include "rom_info.h"

namespace nes {

class RomLoader {
public:
    virtual ~RomLoader() = default;
    virtual ErrorCode load(RomInfo& out) = 0;
};

} // namespace nes
```

## 討論重點

### 1. 為什麼 RomInfo 不用解構子
- 成員 `std::vector` 自動管記憶體（RAII），RomInfo 死掉時 vector 自動釋放
- 基本型別（bool, NesHeader）不用清理
- 編譯器自動生成解構子，依序呼叫每個成員的解構子
- 原版 C 用 malloc 必須手動 free，需要 `free_rom`；C++ 用 vector 不需要

### 2. 為什麼 RomLoader 不用建構子
- RomLoader 是抽象類別（有 `= 0`），根本不能建立物件
- 真正被建立的是子類 FileRomLoader（Stage 5），建構子寫在子類

### 3. virtual 是什麼
- 沒有 virtual：`Dad* p = new Son(); p->say();` 呼叫 `Dad::say()`（編譯期決定）
- 有 virtual：執行期看 p 實際指向什麼，指向 Son 就呼叫 `Son::say()`（動態分派）
- 套到解構子：`delete loader` 時，有 `virtual ~` 會先呼叫子類解構子再呼叫父類
- 建構子：先父後子；解構子：先子後父（兒子可能用到老爸的東西）

### 4. = 0 不是數學的零
- `= 0` 是 C++ 特殊語法，意思是「純虛擬函數，不寫實作」
- 跟 `int a = 0` 完全無關，只是長得像
- C++ 之父選了 `= 0` 而非 `pure`/`abstract` 關鍵字（不想加新關鍵字）

### 5. virtual vs = 0 的區分

| 寫法 | 基類要寫實作？ | 子類要覆寫？ | 能建立物件？ |
|------|--------------|------------|------------|
| `virtual void foo() {}` | ✅ 要 | 可覆寫可不覆寫 | ✅ 能 |
| `virtual void foo() = 0` | ❌ 不寫 | **必須**覆寫 | ❌ 不能 |

- 一個類別只要有任何一個 `= 0`，就是抽象類別，不能建立物件
- 跟有幾個函數無關，只看有沒有 `= 0`
- 建構子可以留，但叫不到（因為不能建立物件）

### 6. 為什麼要做成介面
- 之後可能有不同子類：FileRomLoader、MemoryRomLoader、NetworkRomLoader
- 依賴反轉：主程式依賴介面不依賴實作，換實作不用改主程式
- StepFC 後面的 Mapper（MMC1、MMC3...）也用繼承+虛擬函數
- 原版 C 用函數指標 `sfc_interface_t` 達到同樣目的（C 的多型）

### 7. 原版 C 的函數指標 vs C++ 的虛擬函數
- 原版 `sfc_interface_t` 用函數指標 `load_rom` + `free_rom` 達到多型
- C++ 用 `virtual` 函數，編譯器自動建 vtable（虛擬函數表），比手動函數指標更安全
- 原版需要 `free_rom` 因為 malloc 的記憶體不會自動釋放；C++ 用 vector + RAII 不需要

### 8. 什麼時候需要自訂解構子
- **需要**：類別持有需要手動釋放的資源（raw pointer `new`、C API handle 如 `FILE*`/`SDL_Texture*`）
- **不需要**（Rule of Zero）：成員都是 RAII 型別（`std::vector`、`std::array`、`std::string`、`std::ifstream`）或基本型別（`bool`、`uint8_t`）
- 核心原則：**有 `new` 就要有 `delete`**。能用 RAII 型別取代手動管理，就不用寫解構子

範例 — 需要解構子（raw pointer）：
```cpp
class Foo {
public:
    Foo() { data_ = new bool[500]; }  // 動態配置
    ~Foo() { delete[] data_; }        // 必須手動釋放，否則記憶體洩漏
private:
    bool* data_;
};
```

範例 — 不需要解構子（RAII）：
```cpp
class Foo {
private:
    std::vector<bool> data_{500};  // vector 自己 new/delete，不用寫解構子
};
```

| 型別 | 動態配置？ | 需要解構子？ |
|------|----------|------------|
| `bool` | 否 | 否 |
| `std::array<bool, 8>` | 否（固定大小，內嵌在物件裡） | 否 |
| `std::vector<bool>` | 是（內部 new） | vector 自己寫了，你不用寫 |
| `bool*`（你 new 的） | 是 | **你要自己寫 delete** |

### 9. getter 回傳參考（`&`）不用處理釋放
- 參考（`&`）不擁有記憶體，只是别名，不需要 free
- `const NesHeader& header() const` 只是回傳 `header_` 成員的參考，不配置任何記憶體
- 記憶體歸屬：`header_` 成員變數 → RomInfo 擁有它；`header()` 回傳的 `&` → 只是看它的窗口，不擁有
- RomInfo 死了 → `header_` 跟著死 → 參考也失效（dangling reference）
- 解構子管的是**成員變數**，不是方法。方法只是函數，呼叫完就沒了

## Review 建議
- 程式碼正確，14行精簡乾淨
- 介面設計良好：只有一個 `load()` 方法，職責單一
- 虛擬解構子 `= default` 是業界標準寫法

## 學習心得
Stage 4 的核心是理解 C++ 的多型機制。原版 C 用函數指標 `sfc_interface_t` 達到多型，C++ 用 `virtual` 函數更安全更優雅。`virtual ~RomLoader() = default` 確保子類解構時正確呼叫子類解構子。`load() = 0` 純虛擬函數讓 RomLoader 成為介面，強制子類必須實作。RAII 取代了原版的 `free_rom`，展現了 C++ 記憶體管理的優勢。這是 StepFC 後面 Mapper 繼承體系的基礎。
