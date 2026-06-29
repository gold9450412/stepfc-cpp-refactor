---
layout: default
title: Stage 5：檔案讀取 — 開檔與驗證 (file_rom_loader.h / .cpp 前半)
---

# Stage 5 日誌：檔案讀取 — 開檔與驗證 (file_rom_loader.h / .cpp 前半)

## 日期
2026-06-28

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/file_rom_loader.h` ✅
- 建立 `src/nes/file_rom_loader.cpp` ✅
- `class FileRomLoader : public RomLoader` ✅
- 建構子接收 `std::string` 檔案路徑 ✅
- `std::ifstream` 二進位開檔 ✅
- `load()` 開檔失敗回傳 `ErrorCode::FileNotFound` ✅
- 讀取 16 byte 到 `std::array<uint8_t, 16> raw` ✅
- 呼叫 `parse_header(raw, header)` 驗證 magic，失敗回傳 `ErrorCode::IllegalFile` ✅
- 更新 CMakeLists.txt（加 `file_rom_loader.cpp`）✅
- 更新 main.cpp 測試編譯 ✅
- Build 通過，輸出正確 ✅

## 最終程式碼

### file_rom_loader.h（17行）
```cpp
#pragma once

#include "rom_loader.h"
#include <string>

namespace nes {

class FileRomLoader : public RomLoader {
public:
    explicit FileRomLoader(std::string filepath);
    ErrorCode load(RomInfo& out) override;

private:
    std::string filepath_;
};

} // namespace nes
```

### file_rom_loader.cpp（34行）
```cpp
#include "file_rom_loader.h"
#include <fstream>
#include <array>
#include "nes_header.h"

namespace nes {

FileRomLoader::FileRomLoader(std::string filepath) 
    : filepath_(std::move(filepath)) {
}

ErrorCode FileRomLoader::load(RomInfo& out) {
    std::ifstream file(filepath_, std::ios::binary);
    if (!file) {
        return ErrorCode::FileNotFound;
    }

    std::array<uint8_t, 16> raw;
    file.read(reinterpret_cast<char*>(raw.data()), 16);
    if (file.gcount() != 16) {
        return ErrorCode::IllegalFile;
    }

    NesHeader header;
    if (!parse_header(raw, header)) {
        return ErrorCode::IllegalFile;
    }

    return ErrorCode::Ok;
}

} // namespace nes
```

### main.cpp（20行）
```cpp
#include <iostream>
#include "nes/file_rom_loader.h"
#include "nes/rom_info.h"

int main() {
    nes::RomInfo info;
    std::cout << "Step0 C++ - NES ROM Loader" << std::endl;
    std::cout << "Loaded: " << info.is_loaded() << std::endl;

    nes::FileRomLoader loader("nestest.nes");
    auto result = loader.load(info);
    if (result == nes::ErrorCode::Ok) {
        std::cout << "Load OK" << std::endl;
    } else {
        std::cout << "Load failed" << std::endl;
    }
    return 0;
}
```

### Build 結果
```
Step0 C++ - NES ROM Loader
Loaded: 0
Load OK
```
- 編譯成功，一個 warning: `unused parameter 'out'`（正常，Stage 6/7 才會用 out）

## 討論重點

### 1. `: public RomLoader` — 公開繼承
- `public` 繼承表示 FileRomLoader「是一個」RomLoader
- 父類的 public 成員在子類仍是 public，外面能用
- 還有 `private` 繼承（父類成員全變 private）和 `protected` 繼承，很少用
- 業界幾乎都用 public 繼承

### 2. `explicit` 防止隱式轉換
- 沒 `explicit`：`run("abc.nes")` 偷偷建臨時 FileRomLoader（char* → string → FileRomLoader），可能有副作用
- 有 `explicit`：`run("abc.nes")` 編譯錯誤，必須寫 `run(FileRomLoader("abc.nes"))`
- `explicit` 擋的是「建構子自作主張」，不擋你設計接字串的介面
- 單參數建構子業界習慣都加 `explicit`，除非真的想允許隱式轉換

### 3. `override` 關鍵字
- `ErrorCode load(RomInfo& out) override;`
- 明確告訴編譯器在覆寫父類虛擬函數
- 打錯名字或簽名不對會報錯（沒有 override 編譯器可能只給 warning）
- C++11 引入，業界幾乎都會加

### 4. `std::ifstream` — C++ 讀檔串流
- RAII：建構子開檔，解構子自動關檔（不用像 C 的 `fclose`）
- `std::ios::binary` = 二進位模式，不轉換換行
  - Windows 上文字模式會把 CRLF → LF，破壞二進位資料
  - Linux 上沒差，但寫上去跨平台安全
- `if (!file)` = 開檔失敗（檔案不存在、權限不足等）
- 對照原版 C：`fopen(..., "rb")` + 手動 `fclose`

### 5. `std::ios` 是什麼
- C++ 串流基礎類別，定義開檔模式常數
- `binary`（二進位）、`in`（讀）、`out`（寫）、`app`（接尾）、`trunc`（清空）
- 可用 `|` 組合，如 `std::ios::binary | std::ios::in`

### 6. private 成員為什麼 load() 可以用
- `filepath_` 是 private，但 `load()` 是同類別的方法，自己人可以存取自己的 private
- `private` 防外面，不防自己（門鎖防外人不防有鑰匙的自己人）
- 外面 `loader.filepath_` → 編譯錯誤；`loader.load(info)` → OK

### 7. sink parameter idiom（by value + std::move）
- 建構子 `FileRomLoader(std::string filepath)` 接 by value
- 初始化列表 `filepath_(std::move(filepath))` 把參數搬進成員
- 呼叫端可以 move（不要了就搬過去，零拷貝），也可以 copy（還需要就留著）
- 比 `const&` 更有彈性（const& 永遠拷貝）

### 8. 繼承與多型
- `FileRomLoader loader("nestest.nes")` = 直接用具體型別，沒多型
- `RomLoader* loader = new FileRomLoader(...)` = 父類指標指向子類，多型上場
- 多型不是必須：if/else 分支也能做到
- 多型的價值：(1) 主邏輯只寫一次 (2) 加新類型不用改主程式 (3) 一個函數接父類參考全部通用
- if/else 決定走哪段程式碼（每段要自己寫），多型決定用哪個物件（主邏輯只寫一次）
- StepFC 之後 Mapper 有十幾種，不用多型會寫到死

### 9. `std::array` vs `std::vector`
- array：固定大小編譯期決定，內嵌 stack，無額外開銷
- vector：動態大小，heap allocation，有指標+大小+容量開銷
- 規則：大小不變用 array，執行期會變用 vector
- iNES 檔頭固定 16 byte，用 array 剛好

### 10. `data()` 方法
- `std::array` 的方法，回傳指向內部陣列的原始指標（`uint8_t*`）
- C 函數（如 `ifstream::read`）要裸指標，`data()` 就是給那個指標

### 11. `reinterpret_cast`
- 重新解讀記憶體型別，`uint8_t*` → `char*`
- `ifstream::read` 要求 `char*`，是 C++ 標準庫歷史包袱
- 四種轉型裡最危險的，但這裡 `uint8_t` 跟 `char` 底層相同，安全

### 12. `file.read()` 做什麼
- 從檔案目前位置讀 byte，copy 到指標指向的記憶體，讀完位置自動往後移

### 13. `file.gcount()` 驗證讀取長度
- `read()` 不保證讀滿指定 byte（檔案太短就讀不到）
- `gcount()` 回傳上次 `read()` 實際讀到的 byte 數
- `if (file.gcount() != 16)` 確保真的讀到 16 byte，不夠就是非法檔案

### 14. `loaded_` 為什麼帶參數建構子是 true
- 預設建構子 `RomInfo()`：`loaded_ = false`（空的，沒載入）
- 帶參數建構子 `RomInfo(header, prg, chr)`：`loaded_ = true`（資料都有了）
- 成員宣告 `bool loaded_ = false` 是預設值，初始化列表 `loaded_(true)` 覆蓋它

### 15. 直接用了什麼就 include 什麼
- main.cpp 直接用 `FileRomLoader` 和 `RomInfo`，就 include `file_rom_loader.h` 和 `rom_info.h`
- 不靠「A include B，B include C，所以 A 也能用 C」的鏈式 include
- 業界最佳實踐：依賴要明確，不靠副作用

## 遇到的問題

### 問題 1：初版漏了 `#include "rom_loader.h"`
- file_rom_loader.h 繼承 RomLoader，必須 include 父類標頭檔
- 修正：補上 `#include "rom_loader.h"`

### 問題 2：初版錯字 `ste::string`
- 建構子參數寫成 `ste::string filepath`
- 修正：改為 `std::string filepath`

### 問題 3：初版漏了 `#include <array>` 和 `#include "nes_header.h"`
- load() 用了 `std::array` 和 `NesHeader` + `parse_header`
- 修正：補上 `#include <array>` 和 `#include "nes_header.h"`

## Review 建議
- 程式碼正確，結構清晰
- `explicit` 和 `override` 都正確使用，符合業界慣例
- sink parameter idiom 運用正確
- 錯誤路徑完整：FileNotFound → IllegalFile（gcount 不足）→ IllegalFile（magic 不符）
- `reinterpret_cast` 使用正確，是跟 C 標準庫互動的必要手段

## 學習心得
Stage 5 的核心是實作繼承介面的具體類別。`explicit` 防止建構子隱式轉換是重要的安全措施，`override` 確保正確覆寫父類虛擬函數。`std::ifstream` 的 RAII 特性讓我們不用手動 `fclose`，比原版 C 的 `fopen`/`fclose` 更安全。`file.read()` + `gcount()` 的雙重驗證確保讀取正確，`reinterpret_cast` 是跟 C 標準庫互動的必要手段。sink parameter idiom（by value + move）比傳統 const& 更有彈性，是現代 C++ 傳遞容器參數的慣用手法。多型 vs if/else 的討論讓我理解了為什麼要用介面：不是為了現在，而是為了將來加新類型時不用改主程式。
