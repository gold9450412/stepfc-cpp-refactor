---
layout: default
title: "Stage 03 — CPU 暫存器（`cpu.h` / `cpu.cpp`）"
---

# Stage 03 — CPU 暫存器（`cpu.h` / `cpu.cpp`）

日期：2026-09-06

## 完成事項

- 建立 `src/nes/cpu.h`（31 行）：
  - `struct CpuRegisters`：`a/x/y/sp/p`（uint8_t）+ `pc`（uint16_t），尺寸完全對應硬體
  - `class Cpu`：持有 `CpuRegisters reg_` + `CpuBus& bus_`
  - 介面：`explicit Cpu(CpuBus&)`、`void reset()`、`const CpuRegisters& registers() const`
  - `CpuBus` 用前向宣告，不 include
- 建立 `src/nes/cpu.cpp`（23 行）：
  - 建構子初始化 `bus_(bus)`
  - `reset()`：`reg_ = {}` → 覆寫 `sp = 0xFD`、`p = 0x34` → 從 `kResetVector`（$FFFC/$FFFD）讀兩 byte 組 PC
  - PC 組裝：`static_cast<uint16_t>(high) << 8 | low`（先轉型再位移，避免 int promotion 陷阱——使用者自己寫對的）
- `CMakeLists.txt` 的 `nes_lib` 加入 `src/nes/cpu.cpp`
- 驗證：乾淨重編**零警告**（-Wall -Wextra），17/17 測試全過

## 討論重點

### 1. 暫存器尺寸的權威依據

nesdev `CPU_registers` 頁（https://www.nesdev.org/wiki/CPU_registers）：

- 只有 **PC 是 2-byte**（16-bit），A/X/Y/SP/P 全部 **byte-wide**（8-bit）
- nesdev 把 stack pointer 叫 **S**，我們程式叫 `sp`，同一個東西
- P 實際只用 6 bit，但佔 1 byte

### 2. `explicit` 建構子

單參數建構子會被拿去做隱式轉換（`print(42.0)` 偷偷變 `print(Meters(42.0))`）。加 `explicit` 後傳錯型別編譯期直接抓到。**業界慣例：單參數建構子一律加**。

### 3. 引用成員 `bus_`

- CPU 綁一條 bus 終身不換、絕不為 null → 引用比指標更精確表達語意
- 引用鐵則：綁定後**不能重綁**（`r = b` 是把值寫進去，不是改綁對象）
- **修正一個講太快的地方**：有引用成員的 class，copy constructor **可以用**（新引用綁同一顆物件），被刪掉的是 copy assignment（`operator=`）——因為 operator= 要改綁，違反引用鐵則

### 4. 成員預設初始化 `reg_ = {}`

未初始化的成員是垃圾值，讀它是 UB。宣告處寫 `reg_ = {}` 一勞永逸，建構路徑再怎麼多都不會漏。

### 5. 前向宣告 vs #include

- 只需要「名字」（宣告引用/指標參數）→ 前向宣告就夠
- 要用「內容」（呼叫 `bus_.read()`）→ 必須 include 完整定義
- 原則：**誰用誰 include**（cpu.cpp 用到 bus_.read()，所以 cpu.cpp include cpu_bus.h）
- 好處：改 `cpu_bus.h` 不會連鎖重編所有 include 到 `cpu.h` 的檔（縮小爆炸半徑）。編譯是一個 TU 一個 TU 各自進行的

### 6. `reset()` 的值從哪來

nesdev `CPU_power_up_state` 頁（https://www.nesdev.org/wiki/CPU_power_up_state）的實機量測表格：

- **A/X/Y = 0**：硬體未定義，沒有遊戲依賴它
- **SP = $FD**：開機時 S 從 $00 開始，reset 扣 3 次假 push = $FD（見下面 SP 教學）
- **P = $0x34**：I=1（禁中斷）+ bit5=1（沒接線）+ bit4=1（B flag 佔位）
- **PC = ($FFFC)**：從 reset vector 讀

原版 `step3/sfc_famicom.c` 也是設 $FD/$034，另外有一段 `#if 1` 強制 PC=$C000 的 nestest hack——之後再討論怎麼處理（傾向不抄，模擬器忠實走 vector，測試需要的話在測試端處理）。

### 7. SP（Stack Pointer）深入教學

- **stack 是什麼**：CPU 遇到中斷/副程式跳躍時，暫存器「暫放」的地方。疊盤子模型：後放先拿（LIFO）
- **在哪**：硬體規定死在 **$0100~$01FF**（記憶體第 2 頁，256 bytes），不能搬家
- **為什麼 SP 只要 8-bit**：真正位址 = **$0100 + SP**。SP 只存頁內偏移，高位固定 $01，256 格記「第幾格」就夠
- **方向**：從天花板往下長。空房間時 SP 在頂樓；放越多 SP 越小，$0100 是地板（滿了）
- **push/pop 的嚴格順序**：push = **先寫入，再 SP--**；pop = **先 SP++，再讀出**。因為 SP 指向「下一個空位」（AI 第一次講反了順序，被使用者抓包後修正）
- **$FD 的由來**：`CPU_power_up_state` 頁表格原文 `S: $00 - 3 = $FD`。頁尾註解 [1]：RESET 走的是與 NMI/IRQ/BRK **共用的中斷電路**，那套邏輯會 push **PC（2 bytes）+ P（1 byte）共 3 次**，但 2A03 在 reset 期間**禁止真的寫入**，所以「空轉」3 次：SP 照扣，資料沒寫。模擬器不演這齣戲，直接寫死最終結果 $FD
- **`reg_ = {}` 與 $FD 的關係**：`{}` 只是把全部清 0（A/X/Y 留在 0），sp/p 再由 reset() 覆寫成硬體規定值——兩段式，不是一次到位

### 8. P（Status Register）深入教學

- **概念**：A/X/Y 存「一個數值」，P 存 **8 個獨立的開關（flag）**——ALU 每次運算的副作用備忘錄，條件分支（BEQ/BNE/BCS…）讀它決定跳不跳
- **8 格排法**（bit 7 在左）：`N V - B D I Z C`
- **開機值 $34 = 0b0011_0100 逐位對照**：bit5=1（沒接線）、bit4=1（B）、bit2=1（I），其餘 0
- **bit 5**：硬體缺陷，那條線沒接到任何電路，讀永遠得 1。整個生態系直接接受這個怪癖，模擬器忠實填 1
- **bit 4（B flag）的真相**：P 裡其實**沒有 B 這個 bit**，它只在 PHP 把 P 推進記憶體時以 1 出現在資料裡。規格書佔位說明，不是真實開關（step3 講 BRK 時回來）
- **bit 2（I = Interrupt disable）**：開機值裡唯一有功能的 1。「請勿打擾」牌子：I=1 → IRQ 被無視。開機設 1 的理由：系統還沒初始化好，等遊戲準備好了自己 CLI 拆牌。注意：**是「開機後的預設狀態禁中斷」，不是「reset 過程禁中斷」**（reset 那幾個 cycle 本來就不可能有中斷插進來）
- **I 擋不了 NMI**：N = Non-Maskable，不可遮蔽。NMI 是 PPU 每幀用的，遊戲關不掉
- **其他五個**：Z（結果為 0）、N（最高 bit 1 = 負數，補數約定）、C（無符號爆 255 / 借位）、V（有號爆 -128~+127，跟 C 是不同層的爆）、D（十進位模式，**NES 上沒接線，永遠 0**）
- 記憶口訣：Z/N 是「結果的長相」，C/V 是「加減法有沒有出事」，D 是裝飾品

### 9. CpuBus 複習（使用者忘記，重新走一遍）

CPU 眼中的 64K 記憶體世界，`read(addr)` 是分診櫃檯：$0000-$1FFF → RAM（2KB，鏡像）、$6000-$7FFF → SRAM、$8000-$FFFF → mapper 轉手給卡帶。bus 是線路不是倉庫：PRG ROM 資料在 Mapper000 裡。RAM 開機全 0，遊戲程式跑起來自己寫入；SRAM 有電池存檔不消失。

## 遇到的問題

1. **AI 講錯 push 順序**：第一版把 push 講成「先 SP-- 再寫入」，使用者追問「起點不是 $0100 嗎」抓到矛盾，修正為 6502 的正確順序「先寫再減」+ 起點在頂樓 $01FF（開機狀態另計，實際 $00-3=$FD）。教訓：模擬器開發中「方向」和「順序」這種細節要查證再講
2. **`reg_ = {}` 與 SP 初值的混淆**：使用者發現 `{}` 清零後 sp=0 不是 $FF，釐清了「`{}` 是基線歸零、reset() 再覆寫硬體值」的兩段式設計
3. **nesdev 術語對照**：nesdev 的 S 就是我們的 sp，看到不同名字要知道是同一個東西

## Review 建議

- 這個 stage 的程式碼很乾淨，沒有需要改的地方
- `static_cast<uint16_t>(high) << 8 | low` 的寫法（先轉再移）比 AI 示範的更精確，保留
- 唯一提醒：之後 step3 做 interrupt 時，`0xFD` 這種 magic number 可以考慮抽常數或註解來源（CPU_power_up_state），現在先不動

## 學習心得

1. **結構設計**：Cpu 是「狀態容器 + bus 引用」，本 step 不執行指令——職責單一，之後擴充不動舊碼
2. **引用 vs 指標的選擇**：語意優先。「綁定後不換、絕不為 null」用引用，編譯器幫你守契約
3. **初始化順序**：`= {}` 基線 + reset() 覆寫，分層表達「通用歸零」和「硬體規定值」
4. **查證習慣**：nesdev 的實機量測頁（CPU_power_up_state）是 reset 值的權威來源，S=$FD 這種「看起來隨便」的數字背後都有電路原因（3 次假 push）
5. **旗標暫存器的世界觀**：P 不是數值是 8 個開關，這是組合語言條件分支的基礎
