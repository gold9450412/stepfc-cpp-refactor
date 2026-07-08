---
layout: default
title: Stage 2：NES 檔頭結構 (nes_header.h / nes_header.cpp)
---

# Stage 2 日誌：NES 檔頭結構 (nes_header.h / nes_header.cpp)

## 日期
2026-06-27

## 狀態
✅ 完成

## 完成事項
- 建立 `src/nes/nes_header.h` ✅
- 定義 `struct NesHeader`，為解析後的值型別 ✅
  - `std::array<uint8_t, 4> magic` — 不用 `uint32_t`，避免 endian 問題
  - `uint8_t prg_rom_count`, `chr_rom_count`, `flags6`, `flags7`
  - `std::array<uint8_t, 8> reserved`
- 定義 `constexpr std::array<uint8_t, 4> kMagic{'N', 'E', 'S', 0x1A}` ✅
- 用 `namespace Flags6` / `namespace Flags7` + `constexpr` 定義位元遮罩常數 ✅
- 建立 `src/nes/nes_header.cpp` ✅
- 實作 `parse_header()` ✅
  - 逐 byte 從 raw 取出各欄位，不依賴 struct memory layout
  - 驗證 magic number，不符回傳 `false`

## 最終程式碼

### nes_header.h
```cpp
#pragma once

#include <array>
#include <cstdint>

namespace nes {

struct NesHeader {
    std::array<uint8_t, 4> magic; // NES檔案的標誌，應為 "NES\x1A"
    uint8_t prg_rom_count;         // PRG ROM大小，以16KB為單位
    uint8_t chr_rom_count;         // CHR ROM大小，以8KB為單位
    uint8_t flags6;               // 控制標誌1
    uint8_t flags7;               // 控制標誌2
    std::array<uint8_t, 8> reserved; // 保留位元，應為 (bytes 8-15)
};

// Magic number: "NES\x1A"
constexpr std::array<uint8_t, 4> kMagic = {'N', 'E', 'S', 0x1A};

// Flags 6 (byte 6) 的位元遮罩
namespace Flags6 {
    constexpr uint8_t Mirroring = 0x01; // 位元0: 命名表鏡像方式
    constexpr uint8_t SaveRam = 0x02; // 位元1: 有電池備份 SRAM
    constexpr uint8_t Trainer = 0x04; // 位元2: 有 512 Byte Trainer
    constexpr uint8_t FourScreen = 0x08; // 位元3: 四畫面模式
    constexpr uint8_t MapperLow = 0xF0; // 位元7-4: Mapper編號的低4位元
}

// Flags 7 (byte 7) 的位元遮罩
namespace Flags7 {
    constexpr uint8_t VsUnisystem = 0x01; // 位元0: VS. Unisystem
    constexpr uint8_t PlayChoice10 = 0x02; // 位元1: PlayChoice-10
    constexpr uint8_t MapperHigh = 0xF0; // 位元7-4: Mapper編號的高4位元
}

bool parse_header(const std::array<uint8_t, 16>& raw, NesHeader& out);

} // namespace nes
```

### nes_header.cpp
```cpp
#include "nes_header.h"

namespace nes {

bool parse_header(const std::array<uint8_t, 16>& raw, NesHeader &out) {
    for (int i = 0; i < 4; ++i) {
        if (raw[i] != kMagic[i]) {
            return false;
        }
    }

    out.magic[0] = raw[0];
    out.magic[1] = raw[1];
    out.magic[2] = raw[2];
    out.magic[3] = raw[3];
    out.prg_rom_count = raw[4];
    out.chr_rom_count = raw[5];
    out.flags6 = raw[6];
    out.flags7 = raw[7];
    for (int i = 0; i < 8; ++i) {
        out.reserved[i] = raw[8 + i];
    }
    return true;
}

} //namespace nes
```

## 討論重點

### 1. 為什麼用 `std::array<uint8_t, 4>` 而不是 `uint32_t` 存 magic
- 原版 C 用 `uint32_t id` 直接讀 4 bytes，有 **endian 問題**：x86 是 little-endian，記憶體裡 `4E 45 53 1A` 讀成 `uint32_t` 是 `0x1A53454E`，不是直覺的 `0x4E45531A`
- 用 `std::array<uint8_t, 4>` 逐 byte 比較，沒有 endian 問題
- 這是 SPEC 規格書的核心改動：**逐欄位安全解析** vs struct 直接映射磁碟格式

### 2. endian 風險與 struct memory layout 風險詳解
- **Endianness**：CPU 讀多位元組整數的順序不同。x86（little-endian）低位址放低位元，big-endian 反之。同一個 `uint32_t` 在不同平台值不同
- **Struct memory layout**：編譯器可能在 struct 成員間插入 padding 做對齊。直接 `fread` 進 struct，檔案資料會跟 padding 錯位。padding 多寡因編譯器、平台而異
- **新版做法**：先讀 16 bytes 到 `std::array<uint8_t, 16> raw`（連續 byte，無 padding），再逐欄位 `out.xxx = raw[N]`。完全不依賴 struct 佈局，跨平台安全

### 3. `cstdint` 的作用
- 提供固定位元寬整數型別：`uint8_t`（8位）、`uint16_t`（16位）、`uint32_t`（32位）
- 保證跨平台一致：`char`/`short`/`int` 的大小在不同平台可能不同，`uint8_t` 永遠是 8 位元
- 寫模擬器必須用，因為硬體暫存器、ROM 格式都有固定位元寬

### 4. `std::array` vs C array
- `std::array` 有 `.size()` 可查大小
- 可複製賦值（C array 不能直接 `=` 賦值）
- 可用 `==` 比較內容（C array 比較的是指標不是內容）
- 不會退化成指標（C array 傳參數時退化成指標，丟失大小資訊）
- `.at()` 有邊界檢查（`[]` 沒有）
- magic 需要跟 `kMagic` 比較，`std::array` 可直接 `==`

### 5. `constexpr` vs `#define` vs `const` vs `enum`
- **`#define`**：C 前處理器巨集，無型別、無 scope、除錯看不到名稱
- **`enum`**：unscoped，值會洩漏到外部，會隱式轉 `int`
- **`const`**：可能被編譯器當變數，無法用在不該改的地方（如陣列大小）
- **`constexpr`**：編譯期常數，有型別有 scope，可用在陣列大小、`switch` 等
- 不一定分配記憶體：只讀值不分配，取位址才分配

### 6. 為什麼 Flags 用 `constexpr uint8_t` 而不是 `enum class`
- Flags 是**位元遮罩**，需要位元運算（`&`、`|`、`>>`）
- `enum class` 不支援隱式位元運算，要自己重載 `operator&` 等
- `constexpr uint8_t` 直接可用 `flags6 & Flags6::Trainer`，不需要重載
- `enum class` 適合「分類列舉值」（如 ErrorCode），`constexpr` 適合「數值常數」（如遮罩）

### 6.5 `enum class` vs `constexpr` 隱式轉換的釐清
**重點：真正擋隱式轉換的是 `enum class`，不是 `constexpr`。**

```cpp
constexpr uint8_t SaveRam = 0x02;
int x = SaveRam;  // ✅ 可以！uint8_t → int 是標準數值提升

enum class ErrorCode { Ok, FileNotFound };
int y = ErrorCode::Ok;  // ❌ 編譯錯誤！enum class 擋隱式轉換
```

| 特性 | `enum class` | `constexpr` + namespace |
|------|-------------|------------------------|
| 擋隱式轉 int | ✅ | ❌ |
| 解決名稱衝突 | ✅ | ✅ |
| 支援位元運算 | ❌（要重載） | ✅ |

- Flags 用 `constexpr` 是為了位元運算方便，**接受**隱式轉換不擋
- Flags 本來就是整數遮罩，轉成 int 沒有危險，跟 ErrorCode 那種「分類列舉」性質不同
- `enum class` 的正確用法：
  ```cpp
  // 跟同類型比（最常用）
  if (result == ErrorCode::Ok) { ... }  // ✅
  // 真的需要 int 時，顯式轉換
  int z = static_cast<int>(ErrorCode::Ok);  // ✅ 明確轉型
  // 不能寫 if (result == 0)，編譯器會擋
  ```

### 7. 命名：`prg_rom_count` vs `prg_rom_size`
- 初版用 `prg_rom_size`，但存的不是 byte 數，是「單位數量」（幾個 16KB）
- `count` 比 `size` 語意更準確
- 原版 C 用 `count_prgrom16kb`，C 風格前綴命名（因為 C 沒有 namespace 和 class 成員提供 context）
- 現代 C++ 靠 `NesHeader::` 提供 context，不需要前綴

### 8. `namespace Flags6{}` 後不需要分號
- `namespace` 定義後不需要 `;`（跟 `struct`/`class` 不同）
- 學員寫了 `};`，多餘但不報錯（編譯器允許空宣告後接 `;`）

### 9. namespace 內不用 `::` 存取同名成員
- 在 `namespace nes { }` 內部，直接寫 `kMagic` 即可，編譯器在當前 namespace 找到
- 在 namespace 外面（如 main.cpp），要寫 `nes::kMagic`
- `#include "nes_header.h"` 用雙引號（找專案內標頭檔），不是 `< >`（找系統標頭檔）

### 10. namespace 重複與撞名
- namespace **可以重複**，編譯器會合併所有同名 namespace 的內容
- 但合併不是隔離：兩個 `namespace nes { }` 裡有同名 `kMagic` 會 redefinition 編譯錯誤
- 防撞名靠兩層：namespace 防「不同 namespace 之間」撞名，enum class / class 成員防「同一個 namespace 內」撞名

### 11. 原版 C vs 新版 C++ 在 Stage 2 的具體差異

#### (1) Magic 驗證方式
- **原版**：`fread` 進 `sfc_nes_header_t`，`id` 是 `uint32_t`，用 union hack 繞 endian 問題（`sfc_famicom.c:72-81`）
- **新版**：先讀 `std::array<uint8_t, 16> raw`，逐 byte `raw[i] != kMagic[i]`，無 endian 問題

#### (2) 讀檔頭方式
- **原版**：`fread(&nes_header, sizeof(nes_header), 1, file)` — 直接 fread 進 struct，依賴 struct memory layout（padding 風險）
- **新版**：先讀 16 bytes 到 `std::array<uint8_t, 16> raw`，再逐欄位 `out.xxx = raw[N]` — 不依賴 struct 佈局

#### (3) 常數定義方式
- **原版**：`enum { SFC_NES_VMIRROR = 0x01, ... }` — unscoped enum，值洩漏到全域，需前綴 `SFC_NES_` 避撞名
- **新版**：`namespace Flags6 { constexpr uint8_t Mirroring = 0x01; }` — namespace 隔離，有型別，`constexpr` 編譯期常數

#### (4) Struct 用途
- **原版**：`sfc_nes_header_t` 同時接 `fread`（磁碟映射）又當運算用 → 一個 struct 兩種用途
- **新版**：`NesHeader` 只當解析後的值型別，磁碟讀取用 `std::array<uint8_t, 16>` → 職責分離

#### (5) 命名風格
| 原版 | 新版 | 理由 |
|------|------|------|
| `sfc_nes_header_t` | `NesHeader` | C 靠 `_t` 後綴，C++ 靠 namespace + class |
| `count_prgrom16kb` | `prg_rom_count` | C 靠前綴提供 context，C++ 靠 `NesHeader::` |
| `SFC_NES_VMIRROR` | `Flags6::Mirroring` | C 靠全大寫前綴避撞名，C++ 靠 namespace |

### 12. 職責分離（Separation of Concerns）
- **原版問題**：`sfc_nes_header_t` 同時做兩件事：(a) 對應磁碟格式（fread 直接灌進去）(b) 程式運算用。磁碟格式改了，struct 要改，運算邏輯也跟著改 → 牽一髮動全身
- **新版做法**：三個東西各司其職
  - `raw`（`std::array<uint8_t, 16>`）：只負責「忠實接住檔案的 16 bytes」，不管欄位語意
  - `NesHeader`：只負責「程式裡有意義的欄位」，不管磁碟長什麼樣
  - `parse_header`：負責「轉換」，把 raw bytes 翻譯成有意義的欄位
- **好處**：磁碟格式變了只改 `parse_header`，運算需求變了只改 `NesHeader`，改一邊不會弄壞另一邊
- **比喻**：就像翻譯 — 原版是一個人同時聽外語又用外語思考；新版是先聽寫原文（raw），再翻譯（parse_header），最後用母語思考（NesHeader）

### 13. `constexpr` vs `#define` 詳細對比

| | `#define` | `constexpr` |
|---|---|---|
| 本質 | 前處理器文字替換 | 真正的 C++ 變數/函式 |
| 型別 | 無 | 有 |
| Scope | 從定義點到檔案結尾，不受 namespace 限制 | 受 namespace/class 限制 |
| 除錯 | 看不到名稱，只看到替換後的值 | 看得到變數名稱 |
| 安全性 | 文字替換易出錯（`DOUBLE(3+4)` → `3+4*2=11`） | 正常 C++ 語法，正確 `14` |
| 取位址 | 不行 | 可以 |

- `#define MAX_SIZE 16` 前處理後變成 `16`，編譯器看不到 `MAX_SIZE` 這名字
- `constexpr int kMaxSize = 16` 有型別 `int`，編譯器會檢查，受 scope 限制
- **結論**：現代 C++ 一律用 `constexpr`，`#define` 只留給 `#include` 防護或條件編譯

### 14. `enum class` 的型別安全 — 不會隱式轉 `int`

#### C 的 `enum`（不安全）
```c
enum ErrorCode { Ok, FileNotFound };
ErrorCode result = FileNotFound;
if (result == 1)    // OK，result 隱式轉成 int 來比
int x = result;     // OK，直接轉成 int
result = 42;        // OK，42 不是合法值但編譯器不擋
```

#### C++ 的 `enum class`（安全）
```cpp
enum class ErrorCode { Ok, FileNotFound };
ErrorCode result = ErrorCode::FileNotFound;
if (result == 1)              // ❌ 編譯錯誤！不能拿 ErrorCode 跟 int 比
if (result == ErrorCode::Ok)  // ✅ 只能這樣寫
int x = result;               // ❌ 編譯錯誤！不能隱式轉 int
result = 42;                  // ❌ 編譯錯誤！42 不是 ErrorCode
```

#### 「隱式轉換」是什麼
- 編譯器自動幫你轉型，不問你。`enum` 的值會自動變成 `int`，所以可以 `result == 1`、`int x = result`
- `enum class` 把這條自動轉換的路堵死，強迫你寫 `ErrorCode::Ok` 這種明確名稱

#### 「型別安全」是什麼
- 編譯器幫你擋住不合理的操作。`enum class` 不讓你拿 enum 跟 int 比、不讓你塞不合法的數字進去、不讓你隱式轉型
- 「不安全」= 編譯器放行但執行時可能出錯；「安全」= 編譯器直接擋下

## 遇到的問題

### 問題 1：初版用 `prg_rom_size` 命名
- 初版欄位名為 `prg_rom_size` / `chr_rom_size`
- 修正：改為 `prg_rom_count` / `chr_rom_count`，因為存的是單位數量不是 byte 數

### 問題 2：`parse_header` 實作位置
- 討論是否放在 `.h`（inline）或另開 `.cpp`
- 決定：另開 `nes_header.cpp`，header/implementation 分離，符合業界慣例

## Review 建議
- 程式碼正確，邏輯清晰
- `parse_header` 用迴圈驗證 magic + 迴圈填 reserved，簡潔易讀
- 之後 CMakeLists.txt 需加 `nes_header.cpp` 到 build target 和 `target_include_directories`

## 學習心得
Stage 2 的核心是理解「安全解析」的思維。原版 C 直接 `fread` 進 struct，看似簡潔但有 endian 風險和 struct memory layout 依賴。現代 C++ 的做法是先讀 raw bytes 到 `std::array`，再逐欄位解析到 struct，雖然多寫幾行但更安全也更可攜。

更深層的學習是「職責分離」：原版一個 struct 身兼「接磁碟格式」和「程式運算」兩種用途，新版拆成 raw bytes（接磁碟）+ NesHeader（運算用）+ parse_header（轉換），各司其職，改一邊不會弄壞另一邊。

另外也搞懂了幾個 C++ 基礎觀念：`constexpr` 比 `#define` 多了型別、scope 和除錯能力；`enum class` 比 `enum` 多了型別安全，堵死隱式轉 int 的路，強迫寫出明確的名稱。這些都是現代 C++ 「讓編譯器幫你擋錯」的設計哲學。
