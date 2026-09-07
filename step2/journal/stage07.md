---
layout: default
title: "Stage 07 — 反組譯輸出：2 byte 位址模式"
---

# Stage 07 — 反組譯輸出：2 byte 位址模式

> 日期：2026-09-06
> 對應 SPEC Stage 7：Absolute / Absolute,X / Absolute,Y / Indirect

## 完成事項

- `disassembler.cpp` 新增四個 case（74 行 → 84 行）：
  - `Absolute`：`result += " $" + to_hex(bus_.read(addr + 2)) + to_hex(bus_.read(addr + 1));`
  - `AbsoluteX`：同 Absolute + `",X"`
  - `AbsoluteY`：同 Absolute + `",Y"`
  - `Indirect`：同 Absolute 外面加括號 `" ($" ... + ")"`
- **刪掉 `default: break;`**——13 個 AddressingMode 全數到齊，讓 `-Wswitch` 接手守門
- 建置零警告、17/17 測試通過、主程式輸出正常

## 討論重點

### 1. Absolute 與 Indirect 的差別：繞幾層

- **Absolute `LDA $ABCD`**：operand 就是目的地，讀完 2 bytes 直接去。像「門牌號直接寫在紙條上」。
- **Indirect `JMP ($ABCD)`**：operand 是「放指標的格子」，要先去 `$ABCD`/`$ABCD+1` 讀 2 bytes 拼出真目標再跳。像「紙條上寫的是另一張紙條的位置」。

程式碼對照：

```cpp
// Absolute：operand 本身就是目的地
uint16_t addr = (read(pc + 1) << 8) | read(pc + 2);
value = read(addr);

// Indirect：operand 是「放位址的位址」
uint16_t ptr  = (read(pc + 1) << 8) | read(pc + 2);   // 指標倉庫
uint16_t dest = (read(ptr) << 8) | read(ptr + 1);     // 倉庫裡的指標
pc = dest;
```

跟 `reset()` 讀 $FFFC/$FFFD 是同一招——向量位置本身不是程式起點，是「放起點的格子」。

### 2. IndirectX/Y vs Indirect：倉庫門牌大小，不是指標大小

| | operand | 倉庫（指標存放處） | 倉庫裡的指標 |
|---|---|---|---|
| `(d),Y` / `(d,X)` | 1 byte | 只能零頁 $0000-$00FF | 16-bit |
| `(a)` | 2 bytes | 任意 64K | 16-bit |

**指標永遠 16-bit**。差別只在「倉庫開哪、門牌多大」：零頁版程式碼短 1 byte 但倉庫困在零頁。跟 Stage 6 的零頁指標是同一族概念，只是 Indirect 把倉庫放大到全位址空間。

### 3. 大端小端 vs 印刷順序——兩件事別疊在一起

記憶體**存**是 little-endian（低 byte 在 `addr+1`、高 byte 在 `addr+2`）；螢幕**印**是人類習慣高位在左（`$1234` 不會印 `$3412`）。所以程式碼先讀 `+2` 再讀 `+1`——**不是因為大端**，是印刷順序跟閱讀習慣走，跟儲存序無關。

> 一句話：記憶體存小端（低在前）；螢幕印高位在左。兩件事剛好相反，所以看起來像「反著讀」。

### 4. 為什麼刪 default（本 stage 最有味道的一步）

- 之前 `default: break;` 是**施工鷹架**：case 沒寫齊時，沒 default 的 switch 在 `-Wswitch` 下編不過
- 13 個 case 全到齊 → 鷹架拆除 → `-Wswitch` 從「絆腳石」變「守門員」：未來加第 14 種定址模式卻忘了寫 case，編譯期直接報錯，而不是靜靜印出沒 operand 的爛輸出
- 跟 Stage 1 to_string 故意不寫 default 是同一套哲學：**讓編譯器記住你會忘記的事**

## 遇到的問題

- 無 bug。四個 case 全部一次到位（畢竟是 Stage 5/6 pattern 的複製變化，手感已經養成）。

## Review 觀察

- Absolute 三兄弟共用「讀 +2 高、+1 低」的 pattern，Indirect 只是加括號——沒有抽取 helper 的必要：四個 case 各自展開反而一眼看懂，過早抽象會傷可讀性
- `to_hex` 接受 uint8_t，直接分開印兩個 byte 就好，不需要先組 uint16_t 再拆——省掉 `static_cast` 雜訊（跟 Relative case 組完再拆是兩種寫法並存，Relative 必須組因為 target 要先做加法）

## 學習心得

- 13 種定址模式的**輸出格式全部完成**——反組譯器的格式化主體到此收工
- 「儲存序」與「印刷序」的分離是這關最大觀念：endian 講的是記憶體怎麼擺，人怎麼讀是另一回事
- 間接定址一以貫之：`(d),y` 零頁倉庫、`(a)` 全域倉庫、reset 向量 $FFFC 也是倉庫——「位址的位址」這個概念在 6502 無所不在
- 鷹架理論再 +1：`default` 是鷹架、`(void)addr;` 是鷹架、aggregate init 補零也是鷹架——工地完成，該拆就拆

## 下一關預告

Stage 8：Famicom 擴展——把 Cpu 和 Disassembler 組裝進 Famicom 主機類別。
