# EVM Memory 與 Stack

除了 [[storage]] 和 [[calldata]] 外，EVM 執行時還有兩個至關重要的區域：**Memory (內存)** 和 **Stack (堆棧)**。

---

## 1. Memory (內存) 詳細解析

Memory 是一個**線性**、**可變**且**字節定址 (Byte-addressed)** 的空間。

### 1.1 操作指令
*   **MSTORE (p, v):** 將 32-byte 的值 `v` 寫入內存地址 `p` 開始的位置。
*   **MLOAD (p):** 從內存地址 `p` 開始讀取 32 bytes 的數據。
*   **MSTORE8 (p, v):** 僅寫入 1 byte (`v` 的最低有效字節) 到地址 `p`。

### 1.2 Solidity 的內存管理 (Free Memory Pointer)
Solidity 如何知道下一個變數該放在哪？它依賴於**空閒內存指針 (Free Memory Pointer)**。
*   **位置:** 固定保留在內存地址 **`0x40`**。
*   **機制:**
    1.  EVM 啟動時，會先保留前 128 bytes (0x00 - 0x7F) 給特殊用途。
    2.  讀取 `0x40` 的值（初始通常指向 `0x80`）。
    3.  當你 `new` 一個陣列或結構時，它會從 `0x40` 指向的位置開始寫入。
    4.  寫入完畢後，更新 `0x40` 指向下一個可用的空位。

### 1.3 Gas 成本：為什麼是二次方？
Memory 的設計目的是為了防止單筆交易佔用過多節點內存。
公式如下：
$$ Cost = 3 \times a + \frac{a^2}{512} $$
*(其中 `a` 是擴展後的 32-byte 字詞數量)*

*   **小量使用:** 當內存用量很少時，$\frac{a^2}{512}$ 幾乎為 0，成本線性增長。
*   **大量使用:** 當 `a` 變大，平方項會迅速超越線性項。
    *   擴展到 1KB: ~96 Gas
    *   擴展到 1MB: ~800,000 Gas
    *   擴展到 1GB: **天文數字** (超過區塊 Gas Limit)

這意味著你不能在 Memory 中創建一個超巨大的陣列來進行排序或複雜運算。

---

## 2. Stack (堆棧)

EVM 是一個基於堆棧的虛擬機 (Stack-based VM)。所有的運算（加減乘除、邏輯判斷）都是在 Stack 上完成的。

### 核心特性
*   **LIFO (Last-In-First-Out):** 後進先出。
*   **字長固定:** 每個元素（Word）固定為 **32 Bytes (256 bits)**。
*   **深度限制:** Stack 的最大深度為 **1024**。如果超過這個深度，交易會報錯 (Stack Overflow)。
*   **訪問限制:** 你只能輕鬆操作 Stack 頂部的數據。雖然有 `DUP` 和 `SWAP` 指令，但最深只能訪問到第 16 個元素。

### "Stack Too Deep" 錯誤
這是 Solidity 開發者最常遇到的惡夢。當你在一個函式中定義了太多的區域變數，導致編譯器無法在不超過 Stack 第 16 個元素限制的情況下處理這些變數時，就會觸發此錯誤。
*   **解決方案:** 使用 `struct` 封裝變數、減少區域變數、或使用大括號 `{}` 縮小變數作用域。

---

## 3. 數據區域大對比 (The Big Picture)

| 特性 | **Stack** | **Memory** | **[[storage|Storage]]** | **[[calldata|Calldata]]** |
| :--- | :--- | :--- | :--- | :--- |
| **持久性** | 暫時 | 暫時 | **永久** | 暫時 (歷史記錄除外) |
| **可變性** | 可讀寫 | 可讀寫 | 可讀寫 | **唯讀** |
| **Gas 成本** | 極低 | 中等 (二次方增長) | **極高** | 低 |
| **容量限制** | 1024 深度 | 虛擬無限 (受 Gas 限制) | $2^{256}$ Slots | 交易大小限制 |
| **定址方式** | LIFO | 字節偏移 (Offset) | 插槽索引 (Slot) | 字節偏移 (Offset) |

---

## 4. Solidity 範例

```solidity
function complexCalculation(uint256 x) public pure returns (uint256) {
    // x 是從 Stack 傳入的
    
    // 分配到 Memory (使用 memory 關鍵字)
    uint256[] memory tempArray = new uint256[](3);
    tempArray[0] = x * 2;
    
    // y 是區域變數，存在 Stack
    uint256 y = tempArray[0] + 10;
    
    return y;
}
```

## 5. 什麼時候需要關鍵字？(Data Location Rules)

在 Solidity 中，並非所有變數都需要標註位置。這取決於數據的類型：

### 5.1 值類型 (Value Types) -> 存在 Stack
這些類型大小固定且極小，直接存在 Stack 中，**不需要**關鍵字：
*   `uint`, `int`, `bool`, `address`, `bytes1` ~ `bytes32`, `enum`

### 5.2 引用類型 (Reference Types) -> 必須指定位置
這些類型大小不固定，**必須**使用 `memory`, `storage` 或 `calldata`：
*   `array`, `struct`, `string`, `bytes`

### 5.3 函式參數的選擇
對於外部函式 (`external`) 的引用類型參數：
*   **優先使用 `calldata`:** 它是唯讀的，且避免了內存複製，最節省 Gas。
*   **使用 `memory`:** 只有當你需要在函式內部修改該參數內容時才使用。

### 5.4 實戰範例 (Code Example)

```solidity
contract DataLocation {
    // 狀態變數 (預設就是 storage)
    uint256 public globalNum;
    
    // 1. 值類型 (Value Type) -> 不需要關鍵字
    // x, y 存在 Stack
    function add(uint256 x, uint256 y) public pure returns (uint256) {
        bool result = x > y; // bool 也是值類型，不用加
        return x + y;
    }

    // 2. 引用類型 (Reference Type) -> 必須加關鍵字
    // text 必須指定 calldata 或 memory
    function processString(string calldata text) external {
        
        // ❌ 錯誤寫法: string s = "hello"; (必須指定位置)
        
        // ✅ 正確寫法:
        string memory s = "hello"; // 指定存在 memory
        
        // 修改 memory 變數是允許的 (雖然 string 操作在 Solidity 裡很麻煩)
        // 但不能修改 calldata 變數 text
    }
}
```

---

## 相關知識
*   **MSTORE / MLOAD:** 操作內存的指令。
*   **PUSH / POP:** 操作堆棧的指令。
