---
layout: default
title: Stage 8：Famicom 擴展 (famicom.h / famicom.cpp)
---

# Stage 8 日誌：Famicom 擴展 (famicom.h / famicom.cpp)

## 日期
2026-07-11

## 狀態
✅ 完成

## 完成事項
- famicom.h 加 4 個 include ✅
- famicom.h 新增 3 個方法宣告 + 2 個成員變數 ✅
- famicom.cpp 建構子改寫 ✅
- famicom.cpp 三個方法實作（cpu_read, cpu_write, mapper）✅
- Build 通過 ✅

## 最終程式碼

### famicom.h（25行）
```cpp
#pragma once

#include "rom_info.h"
#include <memory>
#include "mapper.h"
#include "cpu_bus.h"
#include "mapper_factory.h"

namespace nes {

class Famicom {
public:
    explicit Famicom(RomInfo rom_info);
    const RomInfo& get_rom_info() const;
    uint8_t cpu_read(uint16_t addr) const;
    void cpu_write(uint16_t addr, uint8_t data);
    const Mapper& mapper() const;

private:
    RomInfo rom_info_;
    std::unique_ptr<Mapper> mapper_;
    CpuBus bus_;
};

} // namespace nes
```

### famicom.cpp（28行）
```cpp
#include "famicom.h"
#include <utility>

namespace nes {

Famicom::Famicom(RomInfo rom_info)
    : rom_info_(std::move(rom_info))
    , mapper_(make_mapper(rom_info_))
    , bus_(mapper_.get()) {
}

const RomInfo& Famicom::get_rom_info() const {
    return rom_info_;
}

uint8_t Famicom::cpu_read(uint16_t addr) const {
    return bus_.read(addr);
}

void Famicom::cpu_write(uint16_t addr, uint8_t data) {
    bus_.write(addr, data);
}

const Mapper& Famicom::mapper() const {
    return *mapper_;
}

} // namespace nes
```

## 討論重點

### 1. 指標 `*` vs 參考 `&`
- `*`（指標）：可以是空的（nullptr），要用 `->` 存取
- `&`（參考）：保證有效（不能是 nullptr），用 `.` 存取
```cpp
void foo(Mapper* mapper) {
    if (mapper == nullptr) return;  // 要檢查
    mapper->read_prg(0);
}
void foo(Mapper& mapper) {
    mapper.read_prg(0);  // 不用檢查，保證有東西
}
```
- 規則：可能是空的用 `*`，保證有東西用 `&`
- 套回專案：CpuBus 建構子用 `Mapper*`（可能空），getter `mapper()` 回傳 `const Mapper&`（保證有）

### 2. `unique_ptr` 的 `.get()` 是什麼
- `mapper_` 是 `unique_ptr<Mapper>`（智慧指標），不是普通指標
- `mapper_` = unique_ptr 物件本身（不能直接當指標用）
- `mapper_.get()` = 取出裡面的裸指標 `Mapper*`（借用，不轉移所有權）
- `*mapper_` = 取出指向的 Mapper 物件本身（dereference）
- 比喻：unique_ptr 是保險箱，`.get()` 拿出地址（裸指標），`*` 拿出東西本身
- CpuBus 建構子要的是 `Mapper*`，所以用 `.get()` 取出裸指標交給它

### 3. `make_unique<Mapper000>(rom)` 等同什麼
- 等同 `std::unique_ptr<Mapper000>(new Mapper000(rom))`
- `make_unique` 是 `new` + 包進 `unique_ptr` 的語法糖
- 為什麼用 make_unique：少打字、更安全（例外安全）

### 4. 為什麼 `unique_ptr<Mapper000>(new Mapper000(rom))` 型別出現兩次
```cpp
std::unique_ptr<Mapper000>(new Mapper000(rom))
//    ^^^^^^^^^^              ^^^^^^^^^^
//    第一次：告訴 unique_ptr 裝什麼  第二次：真正建物件
```
- 第一次 = 範本參數（指定保險箱規格）
- 第二次 = new 物件（把東西做出來放進去）
- `make_unique<Mapper000>(rom)` 省掉重複：只說一次型別

### 5. `Famicom famicom(std::move(info))` 為什麼型別只出現一次
```cpp
Famicom    famicom   (std::move(info));
//  型別    變數名      建構子參數
```
- 直接建在 stack 上，不需要 `new`，所以型別只寫一次
- 對照 `unique_ptr` 是包裝層，要寫「裝什麼」+「建什麼」，型別出現兩次

### 6. `make_unique` vs 直接建構的差異
| 寫法 | 放哪 | 誰負責釋放 |
|------|------|-----------|
| `Famicom famicom(...)` | stack | 離開作用域自動解構 |
| `make_unique<Famicom>(...)` | heap | unique_ptr 離開作用域自動 delete |
| `make_unique<Mapper000>(rom)` | heap | unique_ptr 離開作用域自動 delete |

- main 裡用直接建構：Famicom 是區域變數，不需要 heap
- Mapper 用 make_unique：要多型（父類指標指向子類），必須放 heap

### 7. 成員初始化順序
- C++ 成員初始化順序是按**宣告順序**（不是初始化列表順序）
- `rom_info_` → `mapper_` → `bus_`：依序完成
- 到 `bus_` 時 `mapper_` 已經建好，`.get()` 才能拿到有效指標
- 如果宣告順序寫反（`bus_` 在 `mapper_` 前面），`bus_` 建構時 `mapper_` 還沒初始化 → 崩潰

### 8. 成員函式 vs 自由函式 — 為什麼 `Famicom::cpu_read` 要加前綴，`make_mapper` 不用
- `Famicom::cpu_read` 是**成員函式**（class 的方法），宣告在 class { } 裡面
  - 在 .cpp 實作時要加 `Famicom::` 告訴編譯器「這是 Famicom 的方法」
  - 不加的話編譯器會以為你在定義一個獨立函式，跟 class 無關
- `make_mapper` 是**自由函式**（free function），不屬於任何 class
  - 宣告在 mapper_factory.h 的 namespace 裡，class 外面
  - 不需要 class 前綴
- 對照表：

| | 成員函式 | 自由函式 |
|---|---|---|
| 宣告位置 | class { } 裡面 | namespace { } 裡面，class 外面 |
| 實作寫法 | `Famicom::cpu_read(...)` | `make_mapper(...)` |
| 需要前綴 | 要 `ClassName::` | 不用 |
| 例子 | `cpu_read`, `cpu_write`, `mapper` | `make_mapper`, `parse_header` |

### 9. `return *mapper_` 為什麼 `*` 能接 `&` 回傳型別
- `*mapper_` 的 `*` 是**解引用運算子**，不是指標型別宣告
- `mapper_` 是 unique_ptr（智慧指標），`*mapper_` 是取出 Mapper 物件本體
- unique_ptr 的 `operator*()` 回傳 `Mapper&`（參考），跟原生指標 `*p` 一樣
- 回傳型別 `const Mapper&` 接住 `Mapper&`，加上 const 保護不讓外部改

```cpp
mapper_        → unique_ptr<Mapper>（智慧指標）
*mapper_       → Mapper&（解引用，取出物件本身）
return *mapper_ → 用 const Mapper& 接住（加 const 保護）
```

- 日常生活比喻：`mapper_` = 你家地址（一張紙條），`*mapper_` = 你家這棟房子，`const Mapper&` = 借別人看房子但不准改裝潢

### 10. `&` 配 `*` vs `*` 配 `*` — getter 回傳參考還是指標
兩種寫法都合法，重點是型別要對得上：

| 回傳型別 | 裡面用什麼 | 配對 | 外部怎麼用 |
|---------|-----------|------|-----------|
| `const Mapper&`（參考） | `*mapper_`（解引用拿物件） | `&` 配 `*` | `mapper.read_prg(...)` |
| `const Mapper*`（指標） | `mapper_.get()`（拿裸指標） | `*` 配 `*` | `mapper->read_prg(...)` |

```cpp
// & 配 *（現在的寫法，業界慣例）
const Mapper& mapper() const {
    return *mapper_;
}

// * 配 *（指標寫法）
const Mapper* mapper() const {
    return mapper_.get();
}
```

- 拿到物件本體 → 用 `&` 接
- 拿到指標 → 用 `*` 接
- 業界慣例：getter 優先用 `&`（參考），因為 mapper_ 一定有值，參考保證「一定有東西」，呼叫端不用寫 null 檢查
- 如果可能 null（例如某些卡帶沒有 mapper），才用指標回傳 null

## 學習心得
Stage 8 的核心是物件組合與生命週期管理。Famicom 用 `unique_ptr` 擁有 Mapper，CpuBus 用 raw pointer 借用 Mapper，所有權模型清晰。建構子的初始化順序（rom_info_ → mapper_ → bus_）體現了依賴關係：bus_ 依賴 mapper_，mapper_ 依賴 rom_info_。`.get()` 從 unique_ptr 取出裸指標是常見的借用模式。`make_unique` vs 直接建構的差異在於 stack vs heap，多型需要 heap 所以用 make_unique。`return *mapper_` 的 `*` 是解引用運算子（拿物件），不是指標宣告，用 `const Mapper&` 接住加上 const 保護。getter 回傳參考而非指標是業界慣例，保證「一定有東西」不用 null 檢查。
