You are an assistant helping me manage my Ethereum EIP study notes in Obsidian.

I want to start studying a new EIP. Please follow these steps to set up the file for me.

**Input:**
- EIP Number: [EIP_NUMBER] (e.g., 1559)

**Task:**
1.  **Fetch Content:**
    *   Find the original markdown content of the EIP from the official Ethereum EIPs GitHub repository (`https://github.com/ethereum/EIPs`).
    *   Get the raw content (but we will not save the full content, just the link).
2.  **Create EIP Note File:**
    *   Create a file named `eips/eip-[EIP_NUMBER].md` (e.g., `eips/eip-1559.md`).
    *   **Crucial:** At the very top of this file, add a link to the original GitHub page: `Original GitHub Link: [URL_TO_GITHUB_BLOB]`
    *   Use the following Traditional Chinese (Taiwan) template for the content:

```markdown
Original GitHub Link: [URL_TO_GITHUB_BLOB]

# EIP-[EIP_NUMBER] 學習筆記

## 摘要 (Summary)
<!-- 這裡寫下 EIP 的主要內容摘要 -->

## 動機 (Motivation)
<!-- 為什麼需要這個 EIP？解決了什麼問題？ -->

## 規範細節 (Specification Details)
<!-- 核心的技術規範 -->

## Go-Ethereum 實作程式碼 (Implementation)
<!-- 這裡的程式碼連結或片段 -->
相關程式碼通常位於 `core/types` 或 `consensus` 目錄下。

```go
// 請搜尋 go-ethereum 倉庫中的相關實作程式碼並粘貼在這裡，或者提供連結
```

## 思考與疑問 (Thoughts & Questions)
<!-- 學習過程中的思考 -->
```

**Instructions for the Agent:**
- If you have the capability to run shell commands, please execute the `write_file` commands directly.
- If you need to search for the EIP content, please do so.
- Confirm when the setup is complete.
