# 什麼是 Calldata？

在 Ethereum 中，**Calldata** 是交易（Transaction）或外部函式調用（External Function Call）中攜帶的唯讀、不可變的暫時性數據。它主要用於存放傳遞給智慧合約的參數。

## 核心特性
1. **唯讀 (Read-only):** 在合約執行期間，`calldata` 的內容不能被修改。
2. **非持久性 (Non-persistent in State):**
    *   **EVM 執行層面:** 數據僅在交易執行期間存在於虛擬機中。如果合約沒有特意將其寫入 [[storage]]，執行結束後這份數據就不會留在合約的「狀態」裡。
    *   **區塊鏈歷史層面 (重要):** 雖然不存於合約狀態，但 `calldata` 是**交易的一部分**，因此會被打包進區塊並**永久儲存在區塊鏈歷史**中（所有全節點都會保存）。這也是為什麼 Layer 2 過去使用它來確保數據可用性 (Data Availability) 的原因。
3. **成本較低:** 相比於 [[memory_stack|memory]] 或 [[storage]]，使用 `calldata` 作為函式參數通常更節省 Gas，因為 EVM 不需要將其複製到內存中（除非明確要求）。

## Gas 成本
根據 [[eip-2028]] 的規範，目前的 Calldata Gas 成本計算如下：
- **非零字節 (Non-zero byte):** 每字節 16 Gas。
- **零字節 (Zero byte):** 每字節 4 Gas。

## Calldata 與 Layer 2
Layer 2 Rollups（如 Optimism, Arbitrum）在 [[eip-4844]] 實作之前，主要依賴於將批次數據（Batch Data）存放在 Calldata 中。由於 Calldata 費用仍佔 Rollup 總成本的 90% 以上，因此社群引入了 **[[blob_lifecycle|Blob 數據]]** 作為更廉價的替代方案。

## Go-Ethereum 中的實作
在 `go-ethereum` 的核心代碼中（例如 `core/types/transaction.go`），Calldata 對應交易結構體中的 `Data` 欄位：

```go
type LegacyTx struct {
    Nonce    uint64          
    GasPrice *big.Int        
    Gas      uint64          
    To       *common.Address `rlp:"nil"` 
    Value    *big.Int        
    Data     []byte          // <--- 這就是 Calldata (合約調用的輸入數據)
    V, R, S  *big.Int        
}
```

這也解釋了為什麼它被稱為「交易數據」：它本質上就是發送交易時附帶的字節數組。

## 總結：會存在節點裡嗎？
1. **會存在 Node 裡面嗎？**
    * 會。作為「區塊歷史」的一部分，它會被永久存在所有全節點 (Full Node) 的硬碟裡。
2. **會影響 Smart Contract State 嗎？**
    * 不會。它不佔用合約的 `storage` 空間，也不會出現在狀態樹 (State Trie) 中。這就是為什麼它比 `storage` 便宜得多。
