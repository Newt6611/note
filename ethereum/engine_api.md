# 以太坊架構筆記：EL 與 CL 的交互機制 (Engine API)

**最後更新時間:** 2026-01-31
**主題:** The Merge 後架構、Engine API 流程、分叉與重組機制

---

## 1. 核心角色定義

在 The Merge 之後，以太坊節點分為兩個獨立運行但緊密合作的層級：

* **CL (Consensus Layer, 共識層)**
    * **代號:** "The Brain" (大腦 / 指揮官)
    * **軟體範例:** Prysm, Lighthouse, Teku
    * **職責:** 負責 PoS 投票、執行 LMD-GHOST 分叉選擇算法、決定誰是主鏈、管理驗證者名單。
    * **特點:** **不執行交易**，它只關心區塊的「共識狀態」，不知道帳戶餘額。

* **EL (Execution Layer, 執行層)**
    * **代號:** "The Muscle" (肌肉 / 工兵)
    * **軟體範例:** Geth, Reth, Nethermind
    * **職責:** 執行 EVM、管理交易池 (Mempool)、維護世界狀態 (StateDB)、實際打包區塊內容。
    * **特點:** **聽命行事**，擁有數據但不知道要在哪條鏈上工作，必須等待 CL 指令。

**溝通管道:** Engine API (Port 8551, HTTP JSON-RPC + JWT Secret)。

---

## 2. 三大核心 Engine API (關鍵區分)

這三個 API 是理解交互的關鍵，**務必區分「驗貨」與「下單」的不同**。

| API 名稱 | 中文俗稱 | 功能描述 | 關鍵參數 |
| :--- | :--- | :--- | :--- |
| **`engine_newPayload`** | **驗貨** | CL 丟一個現成的區塊給 EL，請 EL 檢查內容是否合法 (Gas, Root, Tx)。**注意：這不會觸發挖礦，也不會更新鏈頭。** | `ExecutionPayload` (完整的區塊數據) |
| **`engine_forkchoiceUpdated`** | **上架 / 下單** | 此函數有雙重功能：<br>1. **上架 (無參數):** 告訴 EL 更新最新的鏈頭 (Head)。<br>2. **下單 (有參數):** 告訴 EL 基於新 Head 開始打包新區塊。 | 1. `forkchoiceState` (指針更新)<br>2. **`payloadAttributes`** (挖礦開關) |
| **`engine_getPayload`** | **取貨** | CL 拿著單據向 EL 取回打包好的區塊。 | `payloadID` |

---

## 3. 交互情境流程詳解

### 情境 A：驗收別人的區塊 (Validation / Syncing)
> **場景:** 你的節點從 P2P 網絡收到了別人的區塊，需要驗證並同步。

1.  **收貨:** CL 從 P2P 網絡收到區塊。
2.  **驗貨:** CL 呼叫 `engine_newPayload(Block)`。
    * EL 動作: 解析交易、驗證 Gas、檢查 State Root。
    * EL 回覆: `VALID` (合法) 或 `INVALID` (有毒)。
3.  **上架:** CL 運行 LMD-GHOST 算法，確認這是目前最重的鏈，呼叫 `engine_forkchoiceUpdated(Head=Block, Attributes=null)`。
    * EL 動作: 將 DB 指針移到該區塊，正式更新本地的世界狀態。

### 情境 B：自己要出塊 (Proposing / Mining)
> **場景:** 輪到你的節點當 Proposer (每 6.4 分鐘約一次)。

1.  **下單:** CL 呼叫 `engine_forkchoiceUpdated`。
    * **關鍵差異:** 這次帶上了 **`payloadAttributes`** (包含收款地址 `feeRecipient`, `timestamp`, `random`)。
    * EL 動作: 識別到 Attributes 不為空 -> 啟動 Builder -> 從 Mempool 撈交易 -> 產生 `payloadID`。
2.  **等待:** CL 等待幾秒鐘 (讓 EL 有時間塞滿區塊)。
3.  **取貨:** CL 呼叫 `engine_getPayload(payloadID)`。
    * EL 動作: 停止打包，交出封裝好的 `ExecutionPayload`。
4.  **廣播:** CL 加上共識層數據 (Signature, RANDAO)，廣播到 P2P 網絡。

---

## 4. 分叉與重組 (Forks & Reorgs)

為什麼會有同高度的區塊？如何處理？

### 案例：Alice (Slot 100) vs. Bob (Slot 101)
**前情提要:** 目前最新區塊是 99。Alice 負責 100，Bob 負責 101。

#### 狀況分析
1.  **正常情況:** Alice 出塊 100 -> Bob 看到 100 -> Bob 接在 100 後面出 101。
2.  **分叉發生 (Bob 落後):**
    * Alice 出了 100，但 Bob 因為網路延遲**沒看到**。
    * Bob 以為 Alice 曠工，於是基於 **99** 出了區塊 101 (Block Number 也是 100)。
3.  **共識決戰:**
    * 網路上的其他驗證者大多看到了 Alice 的 100。
    * 根據 LMD-GHOST，Alice 的鏈更重。
    * Bob 的區塊 101 被視為「孤塊 (Orphan)」。

#### 關鍵問題修正 (Misconceptions)

* **Q: Bob 發現錯了，可以在 Alice 後面「重挖」一個新的 101 嗎？**
    * **A: 絕對不行 (Forbidden)。**
    * **協議面:** 同一個 Slot 簽署兩個不同的區塊頭 = **Equivocation (模稜兩可)**。這是 PoS 的死罪，會導致 Bob 被 **Slash (罰沒)** 並踢出驗證者隊列。
    * **時間面:** Slot 只有 12 秒。等 Bob 發現錯誤時，投票窗口已關閉，下一棒 Carol 已經準備接在 Alice 後面了。
    * **結果:** Bob 只能認賠，損失本次出塊獎勵，但保住本金。

* **Q: 這是 Alice 害了 Bob 嗎？**
    * **A: 不是。** 這是 Bob 自己的網路連接問題。區塊鏈是多數決，誰能讓區塊被最多人看到，誰就是贏家。

* **Q: Bob 區塊裡的交易去哪了？**
    * **A:** 當 EL 執行 Reorg (重組) 丟棄 Bob 的區塊時，會將裡面的交易**重新放回交易池 (Mempool)**。下一個出塊者 (Carol) 會重新打包這些交易。

---

## 5. 附錄：共識引擎類型 (Geth 內部)

* **Ethash:** PoW 共識 (Legacy)。
* **Beacon:** PoS 共識 (Current Mainnet)。
* **Clique:** PoA 共識 (Testnet/Private Chain，指定 Signer)。

## 6. 流程圖 (Mermaid)

你可以將以下代碼複製到支援 Mermaid 的編輯器中查看。

```mermaid
sequenceDiagram
    participant CL as CL (Prysm)
    participant EL as EL (Geth/Reth)

    Note over CL, EL: 🟢 情境一：驗收別人的區塊 (Syncing)
    CL->>EL: engine_newPayload(Block) <br> "這塊肉有毒嗎？"
    EL-->>CL: VALID <br> "沒毒，合法的"
    CL->>EL: engine_forkchoiceUpdated(Head=Block, null) <br> "好，設為最新高度 (不打包)"
    EL-->>CL: OK

    Note over CL, EL: 🔴 情境二：自己要出塊 (Proposing)
    CL->>EL: engine_forkchoiceUpdated(Head=Old, Attributes={收錢地址}) <br> "設為最新，並開始幹活！"
    EL-->>CL: payloadID: 12345 <br> "收到，這是領貨單號"
    Note right of CL: ...等待幾秒...
    CL->>EL: engine_getPayload(12345) <br> "做好了沒？給我"
    EL-->>CL: ExecutionPayload (完整區塊) <br> "拿去吧"
