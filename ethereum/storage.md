# Ethereum Storage (存儲)

Storage 是 Ethereum 中唯一**永久性 (Persistent)** 且**可變 (Mutable)** 的數據存儲區域。每個智慧合約都有自己獨立的 Storage 空間。

## 1. 核心特性
*   **合約獨立性 (Contract-specific):** 每個智慧合約都有自己專屬、獨立的 Storage 空間。合約 A 預設無法直接訪問合約 B 的 Storage。
*   **永久保存:** 寫入 Storage 的數據會永久保存在區塊鏈狀態 (State) 中，直到被顯式刪除或合約自毀。
*   **鍵值對結構 (Key-Value):** 是一個巨大的映射表。
*   **容量與定址 (Addressing):**
    *   共有 **$2^{256}$ 個插槽 (Slots)**。
    *   Slot 的索引 (Key) 是一個 `uint256`。這比 `int64` 或是任何傳統計算機的內存定址範圍都要大得多。
*   **稀疏儲存 (Sparse Storage):** 雖然理論空間極大，但只有非零值 (Non-zero) 會實際佔用實體數據庫的空間。初始狀態下所有 Slot 皆為 `0`。
*   **昂貴:** 這是 Ethereum 中最貴的存儲方式，因為所有全節點都必須維護這份狀態。

## 2. 底層實作 (Implementation)
*   **Merkle Patricia Trie (MPT):** 所有合約的 Storage 內容被組織成一棵 MPT 樹 (Storage Trie)。
*   **State Root:** 這棵樹的根哈希 (Storage Root) 被儲存在該合約的帳戶狀態 (Account State) 中。
*   **LevelDB/Pebble:** 節點軟體 (如 Geth) 底層使用 Key-Value 資料庫來實際儲存這些 MPT 節點。

## 3. Gas 成本 (SSTORE Operations)
Storage 的寫入操作 (`SSTORE`) 費用非常複雜，經過多次 EIP 調整 (主要是 [EIP-2200](https://eips.ethereum.org/EIPS/eip-2200) 和 [[eip-2929]])。

主要概念分為 **冷讀取 (Cold Access)** 和 **暖讀取 (Warm Access)**，以及值的變化情況。

### 3.1 寫入成本 (Write Costs)
| 情境 (Scenario) | 說明 | Gas Cost |
| :--- | :--- | :--- |
| **0 $	o$ Non-Zero** | **初始化 (Init):** 將空插槽設為非零值。最貴的操作，因為增加了狀態樹的大小。 | **22,100** (20k + 2.1k Cold) |
| **Non-Zero $	o$ Non-Zero** | **修改 (Modify):** 修改已存在的非零值。 | **5,000** (Cold) / **2,900** (Warm) |
| **Non-Zero $	o$ 0** | **刪除 (Delete):** 清除插槽。會獲得 **Gas 退款 (Refund)**。 | **5,000** (Cold) + Refund |
| **Dirty Slot Write** | **重複寫入:** 在同一筆交易中多次修改同一個插槽。 | **100** (Warm) |

*註: EIP-2929 引入了首次訪問 (Cold) 需額外支付 2,100 Gas 的規則。之後同交易內的訪問 (Warm) 則便宜很多。*

### 3.2 讀取成本 (SLOAD)
*   **Cold SLOAD:** 首次讀取某個 Slot。 **2,100 Gas**
*   **Warm SLOAD:** 同一交易中再次讀取。 **100 Gas**

## 4. Storage Layout 與 Slot Packing (深入解析)
EVM 的 Storage 是一個巨大的陣列，每個位置稱為一個 **Slot**，大小固定為 **32 Bytes (256 bits)**。

Solidity 編譯器會按照定義順序，將狀態變數塞進這些 Slot 中。了解這個規則對於節省 Gas 至關重要。

### 4.1 基本規則
1.  **順序分配:** 變數從 Slot 0 開始依序放置。
2.  **緊密打包 (Tight Packing):** 如果多個相鄰變數的大小加起來不超過 32 Bytes，它們會被「擠」在同一個 Slot 裡。
3.  **新起一行:** 如果當前 Slot 剩下的空間放不下下一個變數，該變數會直接從下一個完整的 Slot 開始放。

### 4.2 實戰範例：打包 vs 不打包

#### ❌ 沒打包好 (Gas 較貴)
每個變數都佔用了一個全新的 Slot，即使它們很小。
```solidity
contract BadPacking {
    uint128 public a; // Slot 0 (佔用 16 bytes, 剩 16 bytes)
    uint256 public b; // Slot 1 (因為 Slot 0 剩的不夠放 32 bytes, 所以 b 獨佔 Slot 1)
    uint128 public c; // Slot 2 (a 和 c 中間被 b 隔開了, c 只能去 Slot 2)
}
```
*讀取這 3 個變數需要 3 次 SLOAD。*

#### ✅ 打包完美 (Gas 較省)
通過調整變數順序，讓 `a` 和 `c` 擠在一起。
```solidity
contract GoodPacking {
    uint128 public a; // Slot 0 (佔用 16 bytes)
    uint128 public c; // Slot 0 (剛好塞進剩下 16 bytes) -> a, c 共用 Slot 0
    uint256 public b; // Slot 1
}
```
*讀取 `a` 和 `c` 只需要 1 次 SLOAD (讀 Slot 0 就全拿到了)。寫入時也可能節省 SSTORE。*

### 4.3 複雜型別的存儲定位 (Advanced)
對於大小不固定的型別，Solidity 使用 `keccak256` 哈希運算來尋找存儲位置，以避免覆蓋到順序排列的變數。

#### 1. 動態陣列 (Dynamic Array)
假設一個動態陣列 `uint256[] public arr` 定義在 **Slot $p$**：
*   **Slot $p$:** 存儲陣列的 **長度 (Length)**。
*   **數據位置:** 第一個元素 `arr[0]` 存放在 `keccak256(p)`。
*   **第 $n$ 個元素:** 存放在 `keccak256(p) + n`。

#### 2. 映射 (Mapping)
假設一個映射 `mapping(uint256 => uint256) public map` 定義在 **Slot $p$**：
*   **Slot $p$:** 什麼都不存（只是個佔位符，確保 Slot $p$ 不被其他變數使用）。
*   **數據位置:** 對於鍵 `k`，其值存放在 `keccak256(k . p)`。
    *   *(注意：`.` 代表連接符。如果 `k` 是類型如 `address`，會補齊至 32 bytes 再與 $p$ 連接後做哈希)*

#### 3. 弦與位元組 (string & bytes)
*   **短數據 (< 31 bytes):** 直接存在 Slot $p$ 裡。最後一個 byte 存儲 `length * 2`。
*   **長數據:** Slot $p$ 存儲 `length * 2 + 1`，實際數據存在 `keccak256(p)`。

#### 4.4 實戰範例：陣列存儲

```solidity
contract ArrayStorage {
    uint256 public a = 11;       // Slot 0
    uint256[2] public fixedArr;  // Slot 1, 2 (連續排列)
    uint256[] public dynArr;     // Slot 3 (只存長度)
    uint256 public c = 55;       // Slot 4
}
```

*   **fixedArr[0]** 就在 **Slot 1**。
*   **dynArr[0]** 的位置計算：
    1. 找到 `dynArr` 的定義槽位：**Slot 3**。
    2. 計算數據起點：`p = keccak256(3)`。
    3. **dynArr[0]** 就在 `p`；**dynArr[1]** 在 `p + 1`。

#### 4.5 陣列內的打包 (Packing in Arrays)
不僅狀態變數會打包，陣列內部的元素如果小於 32 bytes，也會進行打包以節省空間。

*   **`uint256[]`:** 每個元素獨佔一個 Slot。
*   **`uint128[]`:** 每個 Slot 可以容納 **2 個** 元素。
*   **`uint8[]`:** 每個 Slot 可以容納 **32 個** 元素。

這意味著對於小型數據，使用適當的型別可以極大地降低 Storage 的佔用成本。

## 5. Solidity 中的使用
Solidity 會自動將狀態變數映射到 Storage Slots。

```solidity
contract StorageExample {
    // Slot 0
    uint256 public count; 
    
    // Slot 1 (Packed: address 20 bytes + bool 1 byte < 32 bytes)
    address public owner;
    bool public isActive;

    function update(uint256 newCount) external {
        // SSTORE: Non-Zero -> Non-Zero (5000 or 2900 gas)
        count = newCount;
    }
}
```

## 5. Storage vs Memory vs Calldata

| 特性 | Storage | [[memory_stack|Memory]] | [[calldata|Calldata]] |
| :--- | :--- | :--- | :--- |
| **持久性** | **永久** (區塊鏈狀態) | **暫時** (僅限交易執行期間) | **暫時** (交易輸入數據) |
| **可變性** | 可讀寫 | 可讀寫 | **唯讀** |
| **作用域** | 全合約共享 | 函式內部 | 函式參數 |
| **Gas 成本** | **極高** (SSTORE) | 低 (線性增長) | 低 (EIP-2028) |
| **使用時機** | 狀態變數、用戶餘額 | 臨時運算、字串處理 | 外部傳入的大量數據 |

## 相關 EIP
*   **EIP-2200:** 結構化定義了 Net Gas Metering (淨 Gas 計量)。
*   **EIP-2929:** 提高了狀態訪問成本 (Cold Access)，以防禦 DoS 攻擊。