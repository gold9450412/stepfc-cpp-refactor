---
layout: default
title: Stage 0：建置環境
---

# Stage 0 日誌：建置環境

## 日期
2026-06-27

## 狀態
✅ 完成

## 完成事項
- 確認系統工具：g++ 11.4.0 ✅、git 2.34.1 ✅、make 4.3 ✅
- 安裝 CMake 3.25.1 ✅
- 建立 `CMakeLists.txt`（C++17、`-Wall -Wextra`、`EXTENSIONS OFF`）✅
- 建立 `src/main.cpp`（印出 `"Step0 C++ - NES ROM Loader"`）✅
- 編譯執行成功，輸出正確 ✅

## 討論重點

### 1. 為什麼用 CMake 而不是直接用 g++ 命令列
- 單檔編譯用 g++ 一行就夠，但專案檔案增多後命令列會變得很長
- CMake 能自動管理多檔編譯、第三方函式庫連結、跨平台
- 業界 C++ 專案幾乎都用 CMake
- CMake 可以生成 Makefile，等於是「Makefile 的產生器」

### 2. CMakeLists.txt 逐行解說
- `cmake_minimum_required(VERSION 3.14)` — 最低版本要求，3.14 穩定支援 FetchContent
- `project(step0_cpp LANGUAGES CXX)` — 專案名稱，CXX 代表 C++
- `set(CMAKE_CXX_STANDARD 17)` — 設定 C++17
- `set(CMAKE_CXX_STANDARD_REQUIRED ON)` — 不准降級
- `set(CMAKE_CXX_EXTENSIONS OFF)` — 關掉編譯器擴充
- `add_executable(step0_cpp src/main.cpp)` — 定義編譯目標
- `target_compile_options(... PRIVATE -Wall -Wextra)` — 開啟警告

### 3. CMAKE_CXX_EXTENSIONS OFF 的意義
- **ON（預設）**：實際編譯 `gnu++17`，允許 GCC 專有語法（如 VLA、typeof）
- **OFF**：實際編譯純 `c++17`，只准用 ISO 標準語法
- 關掉的理由：跨編譯器可攜性，程式碼不被綁死在 GCC

### 4. GNU++17 vs C++17 哪個是業界標準
- **C++17 是業界標準**（ISO 國際標準）
- GNU++17 是 GCC 專有擴充，只有 g++ 支援
- 業界專案（Google、Facebook 等）幾乎都設定 EXTENSIONS OFF
- 目的：同一份程式碼在 Linux（g++/clang++）、macOS（clang++）、Windows（MSVC）都能編譯

### 5. build 指令詳解：`cmake -B build && cmake --build build && ./build/step0_cpp`
- `&&` 串接三個指令，前一個成功才執行下一個（vs `;` 不管成敗都執行）
- **第一段 `cmake -B build`**：讀 CMakeLists.txt，生成 Makefile 放進 build/。CMake 本身不編譯，只生成建置檔案。這叫 out-of-source build，產物全在 build/ 裡，不弄髒原始碼目錄
- **第二段 `cmake --build build`**：實際執行編譯。等同 `cd build && make`，但跨平台（Windows 自動用 MSBuild）
- **第三段 `./build/step0_cpp`**：執行編譯出來的程式。`./` 代表當前目錄
- 之後只改程式碼沒改 CMakeLists.txt，只需跑後兩段：`cmake --build build && ./build/step0_cpp`

## 遇到的問題

### 問題 1：CMake 未安裝
- 一開始檢查發現 cmake 未安裝
- 解決：`sudo apt install cmake`

### 問題 2：檔案副檔名錯誤 (.c vs .cpp)
- 建立了 `src/main.c` 而非 `src/main.cpp`
- CMake 找不到 `src/main.cpp`，報錯：`Cannot find source file`
- 原因：CMake 認檔案副檔名，`.c` 是 C 語言，`.cpp` 是 C++ 語言
- 解決：將檔名改為 `main.cpp`

## Review 建議
- CMakeLists.txt 寫得完全正確，符合業界慣例
- 建議之後在 `.gitignore` 加入 `build/` 目錄，避免 build 產物進版控

## 學習心得
Stage 0 的核心是建置環境。CMake 一開始看似比直接用 g++ 麻煩，但這是投資 — 專案變大後（多檔、第三方函式庫）CMake 的優勢會越來越明顯。`CMAKE_CXX_EXTENSIONS OFF` 是一個小細節，但代表了業界重視可攜性的思維：從第一天就強迫自己只寫標準 C++。另外，`&&` 串接指令的習慣也很重要 — 讓錯誤及早停止，不白跑後面的步驟。
