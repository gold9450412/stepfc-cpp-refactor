---
layout: default
title: "Stage 10 — 指令表測試（test_instruction_table.cpp）"
---

# Stage 10 — 指令表測試（test_instruction_table.cpp）

> 日期：2026-09-07
> 對應 SPEC Stage 10：256 格全查得到 + 錨點 opcode 驗證 + 表長度隱含保證 + length 正確

## 完成事項

- `tests/test_instruction_table.cpp`（52 行，2 個 TEST）：
  1. `AllOpcodesLookupSuccessfully`：迴圈 0x00-0xFF 全部 `lookup()`，每格驗 `length` 介於 1-3
  2. `AnchorOpcodesAreCorrect`：8 根錨點釘住 instruction/mode/length 三欄
- `tests/CMakeLists.txt` 註冊 `test_instruction_table`（add_executable + gtest_discover_tests）
- 建置零警告、**19/19 測試通過**（17 + 2）

## 討論重點

### 1. 測試戳不到表——資訊隱藏的必然結果

第一版測試寫 `kInstructionTable.size()`，compile 直接爆：**header 裡根本沒有這個名字**。Stage 2 的設計（表是 .cpp 私有財、只露 `lookup()`）在測試上浮出代價——但結論是「測 public 介面」而不是「為了測試破壞封裝」：表長度 256 已被 `lookup()` 隱含保證（`uint8_t` 任何值都是合法索引，少一格 `lookup(那格)` 就炸）。

### 2. 迴圈測「合法」，錨點測「正確」

- 迴圈測試：256 格全查得到、length 1-3——抓**施工鷹架漏填**（aggregate init 補零 → length=0）和**手殘爆表**
- 侷限：若全表 256 格都填 `length=1`，迴圈照樣綠——它不驗「值對不對」
- 錨點補上「正確性」的抽樣驗證

### 3. 錨點怎麼選（本次最大的學習討論，拆三塊講解）

**第一塊——先想表會怎麼錯**：表是手貼四批（0x00-0x3F…0xC0-0xFF），最可能的錯不是單格手滑，是**整批貼錯位**（複製貼上 64 行，錯就整批一起錯，一錯一大片）。

**第二塊——大面積的錯，插一根釘子就炸**：某批整批歪 64 格，只要在該範圍插一個錨點，查到的值就不對，測試立刻紅。抓「整批錯」不需要 256 根釘子。

**第三塊——釘子插哪**：每批至少一根 + 合起來涵蓋多種模式與長度 + 挑「容易混的鄰居」（$A9 Immediate vs $A5 ZeroPage 只差一個 bit，一對釘子釘住模式欄位沒歪）。

### 4. 為什麼不 256 格全測

要測 256 格就得先寫出 256 格的期望值——等於**再手抄一份表**，抄錯兩邊一起錯（測試淪為「我的表 == 我抄的表」）。全量正確性交給第三層：Stage 12 拿 nestest 權威 log 對答案（真實資料踩過每一格）。**測試是在成本與涵蓋之間取捨，沒有免費的無敵**。

### 5. 錨點抓不到的（使用者自己想通的）

- 單格手滑（$96 LDX 打成 LDY，釘子不在那格）
- 沒被釘到的模式填錯
- 同批內部小位移（釘子在歪掉那半之前）

所以整個體系是三層：迴圈（合法）→ 錨點（整批錯）→ nestest 全 log（單格手滑、零星錯全部現形）。

## 遇到的問題

1. **測試檔貼入說明文字**：把 AI 訊息裡「tests/CMakeLists.txt 在 test_integration 之後加兩行」的說明段落整段貼進 .cpp（第 11-14 行非程式碼），review 抓到刪除
2. **AI 自己講錯錨點分配被使用者抓包**：AI 稱「每批至少一根釘子」並把 $4C JMP 歸到 0x00-0x3F 批——**$4C=十進位 76 在 0x40-0x7F**，0x00-0x3F 批實際零釘子。使用者一句「4c不是比3f大嗎」抓出錯誤。修正：補 `$20` JSR/Absolute/3（填 0x00-0x3F 空窗）+ `$A5` LDA/ZeroPage/2（補 ZeroPage 模式、與 $A9 成對）
3. **AI 埋的引導問題按預期爆炸**：第一版 `kInstructionTable.size()` compile 失敗，正是用來帶出「測 public 介面」的討論

## Review 觀察

- `static_cast<uint8_t>(opcode)` 處理 int 迴圈變數餵 uint8_t 參數，顯式轉型正確
- 期望值用 `1`/`2`/`3`（int）與 `uint8_t length` 比較——gtest 的 EXPECT_EQ 對整數這樣比安全
- 錨點選擇事後修正後的分配：四批全覆蓋（0x00-0x3F×1、0x40-0x7F×2、0x80-0xBF×2、0xC0-0xFF×2）、五種模式（Implied/Immediate/Absolute/Relative/ZeroPage）、三種長度全有

## 學習心得

- **測試策略是分層的**：合法性（迴圈）→ 抽樣正確性（錨點）→ 全量驗證（權威 log），每層抓的錯誤類型不同，互相不取代
- **錨點選擇的思路比錨點本身重要**：先想「這份資料怎麼產生的、會怎麼錯」（手貼四批 → 整批錯位風險），再設計抽樣點——這是測試設計的通用心法
- **封裝與可測性的張力**：資訊隱藏後測試只能走 public 介面，看似損失，其實逼你測「行為」而非「內部資料形狀」——反而更穩（內部重構時測試不用跟著改）
- 使用者抓到 AI 講錯分配（$4C 位置）——「講得漂亮」不等於「做得正確」，表格類資料的宣稱要逐格可驗證

## 下一關預告

Stage 11：Disassembler 測試（`test_disassembler.cpp`）——FakeMapper 塞自訂 bytes，13 種定址模式逐一驗證輸出字串、Relative 跳躍目標、next_addr。step2 最大的一關。
