---
layout: default
title: "Stage 05 — 反組譯輸出：無 operand 模式"
---

# Stage 05 — 反組譯輸出：無 operand 模式

日期：2026-09-06

## 完成事項

- `disassembler.cpp` 新增 `to_hex(uint8_t)` helper（查表法）：
  - `static constexpr char kHexDigits[] = "0123456789ABCDEF"`，函式內 static 藏在最靠近使用處
  - 高 4 bit（`value >> 4`）與低 4 bit（`value & 0x0F`）各索引一個字元
- `disassemble()` 主體改為 **switch on `info.mode`**，新增四個 case：
  - `Immediate`：`" #$" + to_hex(bus_.read(addr + 1))` → `LDA #$AB`
  - `Implied`：什麼都不加 → `SEI`（case 空、break 在）
  - `Accumulator`：`" A"` → `ASL A`
  - `Relative`：`{ }` scope 內算 `target = next_addr + static_cast<int8_t>(operand)`，16bit 拆高低 byte 兩次 `to_hex` → `BNE $C007`
- `default: break;` 保留——其餘九種模式（Stage 6/7）還沒做，先維持 Stage 4 行為（只印 mnemonic）
- 驗證：零警告，17/17 測試全過

## 討論重點

### 1. hex 格式化的三條路（為什麼選查表法）

| 路線 | 評價 |
|------|------|
| `snprintf(buf, "%02X")` | C 風格、要管 buffer，違反本專案 SPEC 原則 |
| `<sstream>` + `std::hex`/`setw`/`setfill` | 可行但有**大坑**：`uint8_t` 直接 `<<` 會被當 `char` 印出亂碼，必須轉 int；一行很長；要多兩個 header |
| `"0123456789ABCDEF"` 查表 | byte 切兩半各索引一個字元，短、零格式化開銷 |

`kHexDigits[11]` = `'B'`：數字→字元就是陣列索引。Stage 6/7（`$AB`、`$ABCD`）與 Stage 9 全靠這顆磚頭。

### 2. `static constexpr` 的空間配置（使用者主動問）

- `static`：**只配置一次、活到程式結束**，之後每次呼叫直接用同一份
- 只寫 `constexpr` 或只寫 `static` 都能跑，但意義不同：
  - `const` = 「不許改」（行為承諾）
  - `constexpr` = 「編譯期定案」（本質宣告）
- 「優化」的白話：把工作從**執行期**挪到**編譯期**。超簡單案例：`constexpr int k = 4 * 5;` 編譯時直接寫死 20 進機器碼；`int arr[kTableSize]` 這種陣列大小**只有 constexpr 進得去**
- 業界慣例：能用 constexpr 就用，把事實講清楚讓編譯器有最大依據

### 3. `char` 不能取代 `to_hex`（關鍵誤解釐清）

使用者問「`CpuBus::read` 改回 `char` 是不是就不用 to_hex」——戳破兩層：

1. **記憶體裡存的是數字，不是 hex**。`1010 1011`、`171`、`AB` 是同一個 byte 的三種寫法，hex 只是給人看的顯示格式
2. `to_hex` 做的是「數字 → 兩個 ASCII 字元」的轉換，`char` 也逃不掉這一步（直接印 char 會印出該 ASCII 碼的怪字元）

而且 `char` 更危險：x86 上預設 **signed**（`char c = 0xAB` 是 -85），位址計算會炸；語意上 ROM 裡是數據不是文字。`uint8_t` 是對的選擇。

### 4. `#$` 符號系統

- `$` = 6502 組語的十六進位記號（同 C 的 `0x`）
- `#` = 「值本身」（Immediate）
- `LDA #$AB`（拿數字 171）vs `LDA $AB`（去位址 $00AB 拿內容）——`#` 是分界線

### 5. Relative 定址的本質

- 指令 2 bytes = opcode + offset（**就這一個 operand**），「讀 2 個」= opcode 一個 + offset 一個
- **target 不存在記憶體裡，是算出來的**：`target = next_addr + offset`。反組譯器模仿 CPU 把它算好印出，比 `BNE +5` 好讀
- offset 是 1 byte **有號數**（`static_cast<int8_t>`），範圍 -128 ~ +127，所以 branch 只能跳附近；更遠要「相反條件 branch + JMP」接力——6502 用小空間換靈活的取捨
- `uint16_t + int8_t` 自動升級 int 運算，結果存回 uint16_t

### 6. `BNE $C007` 在幹嘛

BNE = Branch if Not Equal，看 **Z 旗標**（Z=0 就跳）。配 CMP/CPX 構成 6502 的 if + goto，迴圈必備。反組譯階段只負責印對字，跳不跳是 Step3 執行期的事。

### 7. case 內宣告變數要 `{ }`

`case Relative:` 需要宣告 `offset`/`target`，switch 的 case 共用一層 scope，不括起來宣告會打架。這是 switch 少數需要大括號的場合。

### 8. 16bit 位址拆兩次 `to_hex`

`to_hex` 一次只吃一個 byte（吐 2 個字元），`$C007` 要 `target >> 8` 與 `target & 0xFF` 各印一次拼成 4 位 hex。之後 Stage 7 的 Absolute 家族會重複這個 pattern。

## 遇到的問題

- **Immediate case 漏寫 `break;`**（fall-through 到空的 default，結果碰巧一樣但 `-Wimplicit-fallthrough` 會警告）：教訓 = **每個 case 結尾必 break**，行為碰巧對不代表合法。使用者自己補上修正
- Relative case 的 `{ }` 造成視覺縮排多一層（8 格 vs 其他 case 的 4 格）：純格式 nit，使用者手動調回。switch 本身不算一層縮排

## Review 建議

- 程式碼正確，無需修改
- `to_hex` 放 anonymous namespace 或 static 的選擇：目前直接放 `namespace nes` 是檔案內可見，之後若 main.cpp 也想用再升級到標頭。本 stage 保持現狀即可
- Stage 6/7 就是照樣加 case（1 byte / 2 byte 位址模式），模式已經建立

## 學習心得

1. **型別選擇是語意**：`uint8_t`（數據 byte）vs `char`（文字字元）vs `int8_t`（有號偏移）——同一個 byte 三種身份，選對型別 = 把意圖寫進簽名
2. **`static constexpr` 不是玄學優化**：就是「這常數編譯期定案、只配一份」，跟 Stage 2 的 `kInstructionTable` 同一個精神
3. **顯示格式 ≠ 資料本質**：hex/十進位/二進位都是同一個數的皮，轉換函式（`to_hex`）永遠需要
4. **`#` 與 `$` 是 6502 組語的閱讀鑰匙**：有 `#` 拿值、沒 `#` 拿位址；看到 `$` 就是大寫 hex
5. Relative 的「算出來的 target」呼應了 CPU 的真實行為——反組譯器是 CPU 的模仿者，不是文件的抄寫員
