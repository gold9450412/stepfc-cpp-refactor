---
layout: default
title: Stage 1：錯誤碼定義 (error.h)
---

# Stage 1 日誌：錯誤碼定義 (error.h)

## 日期
2026-06-27

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/error.h` ✅
- 定義 `enum class ErrorCode`，包含四個值 ✅
- 放在 `namespace nes` 裡 ✅
- 每個值加註解說明用途 ✅

## 最終程式碼
```cpp
#pragma once

namespace nes {

enum class ErrorCode {
    Ok = 0,       // 成功
    FileNotFound, // 找不到檔案
    IllegalFile,  // 非法檔案格式
    OutOfMemory   // 記憶體不足
};

} //namespace nes
```

## 討論重點

### 1. enum class vs enum（C → C++ 的改進）
- 原版 C 用 `enum sfc_error_code`，值會洩漏到外面，可能跟其他 enum 撞名
- C++ 用 `enum class`，值不會洩漏，一定要寫 `ErrorCode::Ok` 不能只寫 `Ok`
- `enum class` 不會隱式轉成 `int`，型別安全。不能寫 `if (result == 0)`，必須寫 `if (result == ErrorCode::Ok)`

### 2. namespace 的作用與重複定義
- `namespace` 可以重複定義，編譯器會合併所有同名的 namespace 內容
- 每個檔案（error.h、nes_header.h、rom_info.h...）都用 `namespace nes`，全部合併
- namespace 防的是「跟別的 namespace 撞名」，不防同一個 namespace 內部撞名
- enum class 防的是「同一個 enum 裡的值跟別的 enum 的值撞名」
- 兩個搭在一起有兩層保護：namespace 防外部，enum class 防內部

### 3. 命名風格：全大寫 vs PascalCase
- 全大寫 + 底線（`FILE_NOT_FOUND`）是 C 語言習慣，因為 C 沒有 namespace 和 enum class，只能靠命名前綴避衝突
- PascalCase（`FileNotFound`）是業界 C++ 慣例，Google C++ Style Guide 和 LLVM 都這樣建議
- C++ 有了 enum class 和 namespace 保護，不需要靠命名前綴避衝突
- `ErrorCode::FileNotFound` 比 `ErrorCode::FILE_NOT_FOUND` 更易讀

### 4. namespace 內不縮排
- 業界慣例：namespace 內容不縮排，減少縮排層級
- Google C++ Style Guide 和 LLVM Coding Standards 都建議這樣
- 縮排會讓大型檔案的程式碼被推到第 4、5 層，不易閱讀

### 5. #pragma once 的作用
- 防止標頭檔被重複引入
- 如果 a.h 和 b.h 都 include error.h，main.cpp 同時 include a.h 和 b.h，error.h 會被引入兩次
- 沒有保護會導致 enum class 重複定義 → 編譯錯誤
- `#pragma once` 讓編譯器記住「這檔案引入過了」，第二次自動跳過
- 原版 C 用 `#ifndef` guard（三行），`#pragma once` 更簡潔（一行）
- 雖然非 C++ 標準，但 g++、clang++、MSVC 全都支援，業界 C++ 幾乎都用

### 6. CMakeLists.txt 是否需要更新
- error.h 是標頭檔（header-only），不需要加到 `add_executable`
- 但之後 main.cpp 要 `#include "nes/error.h"` 時，需要加 `target_include_directories(step0_cpp PRIVATE src)`
- 這行告訴編譯器 `src/` 是 include 根目錄，才能找到 `src/nes/error.h`
- 可以先加也可以之後加

## 遇到的問題

### 問題 1：初版使用了 C 風格前綴命名
- 初版寫成 `SFC_ERROR_OK`、`SFC_ERROR_FAILED` 等，沿用了原版 C 的命名
- 修正：改為 PascalCase（`Ok`、`FileNotFound`），因為 C++ 有 namespace 和 enum class 保護，不需要前綴

### 問題 2：初版缺少註解
- 初版沒有為每個值加註解
- 修正：每個值後面加上一行中文註解說明用途

### 問題 3：命名選擇 IllegalFile vs IllegalFormat
- SPEC 建議 `IllegalFormat`，學員選擇 `IllegalFile`
- 語意上可以接受（原版 C 也是 `ILLEGAL_FILE`），保留學員的選擇

## Review 建議
- 程式碼正確，符合業界 C++ 慣例
- 小細節：`} //namespace nes` 建議改成 `} // namespace nes`（// 後加空格），但不影響功能

## 學習心得
Stage 1 的核心是理解 C++ 的型別安全機制。`enum class` + `namespace` 兩層保護取代了 C 語言靠命名前綴的作法，讓程式碼更乾淨也更安全。`#pragma once` 則是現代 C++ 標頭檔保護的標準做法，比傳統的 `#ifndef` guard 更簡潔。命名風格從全大寫改為 PascalCase，反映了 C++ 不需要靠命名避衝突的設計哲學。
