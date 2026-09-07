---
layout: default
title: "Stage 6 日誌：反組譯輸出 — 1 byte 位址模式"
---

# Stage 6 日誌：反組譯輸出 — 1 byte 位址模式

日期：2026-09-06

## 完成事項

- `disassembler.cpp` 新增五個 case，處理 operand 為 1 byte 的定址模式：
  - `ZeroPage` → `LDA $AB`
  - `ZeroPageX` → `LDA $AB,X`
  - `ZeroPageY` → `LDA $AB,Y`
  - `IndirectX` → `LDA ($AB,X)`
  - `IndirectY` → `LDA ($AB),Y`
- 五種模式共用同一套 pattern：讀 `bus_.read(addr + 1)` 得 operand，`to_hex()` 轉兩位 hex，差別只在前綴/後綴
- 建置零警告、17/17 測試全過

## 討論重點

### 1. ZeroPage 家族三兄弟

- **ZeroPage** `LDA $AB`：不加索引，直接去 $AB 拿。位址寫死在指令裡（1 byte operand）
- **ZeroPageX** `LDA $AB,X`：去 $AB+X 拿（位址先加 X 再去拿）
- **ZeroPageY** `LDA $AB,Y`：去 $AB+Y 拿

`$AB` 是編譯進指令的常數，X/Y 是執行當下暫存器裡的值。「沒 X 沒 Y」= 不用暫存器加偏移。

用途：敵人資料排在零頁 $40 起跳每隻 1 格，`LDA $40,X` 一行就能「X 是幾號就讀幾號」。

### 2. IndirectX vs IndirectY：差在繞幾層

- **ZeroPageY** `LDA $AB,Y`：一層。位址直接算出來 `$AB + Y`，去那格拿資料，結束
- **IndirectY** `LDA ($AB),Y`：兩層。先去 $AB/$AC（共 2 bytes）拿 16-bit 位址，再把這位址 + Y 去拿資料

具體例子（Y=3，記憶體 $00AB 內容 `10 80`）：
- `LDA $AB,Y` → 去 $00AB+3 = **$00AE** 拿（永遠困在零頁）
- `LDA ($AB),Y` → 先讀出 $8010，再 +3 → 去 **$8013** 拿（可飛到 64K 任何地方）

括號 `($AB)` 的意思：「這格存的不是資料，是通往某處的地址」。

### 3. 為什麼指標是 2 bytes：little-endian

間接定址拿的指標是 16-bit，一個位址要 2 bytes 才裝得下：

```
$00AB: 10   ← 低 byte
$00AC: 80   ← 高 byte
```

6502 慣例：低 byte 在前、高 byte 在後（little-endian，小頭序）。`($AB)` = 從 $AB 開始連讀兩格拼成 16-bit 位址。

跟 `reset()` 讀 $FFFC/$FFFD 拼 PC 是同一個手法。

### 4. 括號位置是語意不是排版

- `($AB,X)`：X 加在取指標**之前**——決定去零頁**哪一格**讀指標（挑指標）
- `($AB),Y`：Y 加在取指標**之後**——指標目標**附近**再偏移

nesdev wiki CPU_addressing_modes 頁的 Formula 欄把順序寫得很明確：

```
(d,x): val = PEEK( PEEK((arg + X) % 256) + PEEK((arg + X + 1) % 256) * 256 )
(d),y: val = PEEK( PEEK(arg) + PEEK((arg + 1) % 256) * 256 + Y )
```

`arg + X` 在最內層、`+ Y` 在最外層。所以括號位置是照公式結構印的，不是隨便的排版。

wiki 也註明 "(d),y mode is used far more often than (d,x)"，(d,x) 適合零頁放一排指標的表（音樂引擎用 X=0,4,8,12 切換 APU 聲道）。

### 5. 為什麼要有 IndirectY

零頁那 256 格可以當「指標倉庫」：遊戲把指標寫進零頁（例如「敵人陣列在哪」），程式碼不用改就能透過改指標切換資料來源。這是 6502 沒有真正間接定址暫存器時的替代方案，射擊遊戲的子彈清單幾乎都靠它。

### 6. 為什麼 IndirectY 比較熱門

IndirectY 的指標可以直接指到零頁外、再靠 Y 撈整排資料；IndirectX 少用得多。

## 遇到的問題

### 1. ZeroPage case 貼上時變數名寫錯 + 漏 break

第一次貼 ZeroPage case 時寫成 `text +=`（函式內變數其實叫 `result`），且漏了 `break;`（會 fall-through 到 default）。Review 抓到後自行修正。教訓重申：每個 case 結尾必 break（Stage 5 Immediate 才教訓過）。

### 2. IndirectY 括號位置寫反

寫成 `",Y)"` 而非 `"),Y"`。剛講完「括號位置是語意」就貼反，證明眼到心到需要練習。修正後正確。

## Review 建議

- 縮排跑掉（case 內 4 格應為 8 格）與多餘空行：自行手動調回，純格式無傷大雅

## 學習心得

- Stage 6 的五個 case 全是 Stage 5 Immediate 模式的複製變化，工作量確實如事前評估的「小」
- 教學順序（ZeroPage → 索引版 → 間接版）由淺入深，括號語意是最後也最重要的一塊
- nesdev 的 Formula 欄用 PEEK 巢狀結構表達定址順序，讀懂它就等於讀懂硬體行為
- 反組譯器只需要「印對字」，執行期的實際定址計算是 step3 CPU 的事；但現在弄懂語意，step3 就不用重學

## 下一步

Stage 7：2 byte 位址模式（Absolute `LDA $ABCD`、AbsoluteX `LDA $ABCD,X`、AbsoluteY `LDA $ABCD,Y`、Indirect `JMP ($ABCD)`），16-bit 位址拆兩次 to_hex 的 pattern 重複使用。
