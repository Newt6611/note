Original GitHub Link: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md

# EIP-20 學習筆記

## 摘要 (Summary)
EIP-20 定義了同質化代幣 (Fungible Tokens) 的標準介面。這套標準 API 讓代幣能在不同的應用程式（如錢包、去中心化交易所和鏈上應用）之間無縫互操作。

## 動機 (Motivation)
在 EIP-20 出現之前，每個代幣合約可能都有自己獨特的 API 介面。這導致開發者在整合不同代幣時需要編寫大量的客製化程式碼。建立此標準的目的是為了提供一個通用的介面，讓代幣易於在以太坊生態系統中被重複使用與整合。

## 規範細節 (Specification Details)

### 必須實作的方法 (Methods)
*   `name()` (選填): 回傳代幣全名。資料類型：`string`（例如 "MyToken"）。
*   `symbol()` (選填): 回傳代幣符號。資料類型：`string`（例如 "MYT"）。
*   `decimals()` (選填): 回傳代幣使用的小數點位數。資料類型：`uint8`（預設建議為 18，以維持與以太幣的一致性）。
*   `totalSupply()`: 回傳目前的代幣總發行量。
*   `balanceOf(address _owner)`: 回傳指定帳戶的代幣餘額。
*   `transfer(address _to, uint256 _value)`: 從呼叫者帳戶轉移指定數量的代幣到目標地址。成功時必須觸發 `Transfer` 事件。
*   `transferFrom(address _from, address _to, uint256 _value)`: 從 `_from` 地址轉移代幣到 `_to` 地址。通常搭配 `approve` 使用。
*   `approve(address _spender, uint256 _value)`: 授權 `_spender` 從呼叫者帳戶提取最多 `_value` 數量的代幣。成功時必須觸發 `Approval` 事件。
*   `allowance(address _owner, address _spender)`: 回傳 `_spender` 目前被授權從 `_owner` 提取的剩餘金額。

### 必須實作的事件 (Events)
*   `Transfer(address indexed _from, address indexed _to, uint256 _value)`: 當代幣被轉移時觸發（包含轉移數量為 0 的情況）。
*   `Approval(address indexed _owner, address indexed _spender, uint256 _value)`: 當 `approve` 方法被成功呼叫時觸發。

## Go-Ethereum 實作程式碼 (Implementation)
EIP-20 是智慧合約層級的標準，而非 Go-Ethereum (Geth) 核心共識協議的一部分。不過，Geth 的 `accounts/abi/bind` 模組常被用來與符合 EIP-20 標準的合約進行互動。

在 `go-ethereum` 的測試程式碼中，常會看到模擬 ERC20 行為的範例，或是在 `params/config.go` 中定義與某些特定代幣（如預編譯合約或核心合約）相關的邏輯。

```go
// Geth 中與合約互動的範例（使用 abigen 產生的綁定）
// 相關程式碼通常位於 accounts/abi 相關目錄下。
// 連結：https://github.com/ethereum/go-ethereum/tree/master/accounts/abi
```

## 思考與疑問 (Thoughts & Questions)
<!-- 學習過程中的思考 -->
1. ERC-20 的 `approve` 方法存在 Race Condition（競爭條件）問題，後續的 EIP-717 或 ERC-2612 (Permit) 是如何優化或解決這個問題的？
2. 為什麼 `decimals` 是選填的？大多數代幣為什麼都選擇 18 位？