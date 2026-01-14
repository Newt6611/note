# 如何在伺服器上設定 Bitwarden 生成的 SSH 金鑰

這份文件說明如何將 Bitwarden 生成的 SSH 公鑰 (Public Key) 配置到 Linux 伺服器上，以便使用 SSH 金鑰進行身份驗證。

## 步驟 1：從 Bitwarden 取得公鑰

1. 打開 Bitwarden 應用程式或瀏覽器擴充功能。
2. 找到您生成的 SSH 金鑰項目。
3. 複製 **公鑰 (Public Key)**。
   - 格式通常以 `ssh-rsa`, `ssh-ed25519` 等開頭。

## 步驟 2：登入伺服器

使用現有的方式（密碼或其他金鑰）登入您的目標伺服器：

```bash
ssh username@your-server-ip
```

## 步驟 3：設定 `authorized_keys`

1. 確保使用者家目錄下有 `.ssh` 資料夾：
   ```bash
   mkdir -p ~/.ssh
   ```

2. 編輯或建立 `authorized_keys` 檔案：
   ```bash
   vim ~/.ssh/authorized_keys
   ```

3. 將 **步驟 1** 複製的公鑰貼上到檔案的新一行中（在 vim 中按 `i` 進入插入模式，貼上後按 `Esc`，輸入 `:wq` 並按 `Enter` 存檔離開）。

## 步驟 4：設定正確的權限

SSH 對檔案權限非常敏感，必須設定正確才能運作：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## 步驟 5：測試連線

1. 登出伺服器。
2. 確保您的 SSH 客戶端（例如 Bitwarden SSH Agent）已載入該私鑰。
3. 嘗試再次登入：
   ```bash
   ssh username@your-server-ip
   ```

如果設定成功，您應該不需要輸入密碼即可登入（或是輸入 Bitwarden 金鑰的 passphrase）。

## 步驟 6：設定 SSH 別名 (Alias)

為了方便登入（例如直接輸入 `ssh bot`），您可以在 **本地電腦** 上設定 SSH 設定檔。

1. 在您的本地電腦，編輯 `~/.ssh/config` 檔案：
   ```bash
   vim ~/.ssh/config
   ```

2. 加入以下內容：
   ```ssh
   Host bot
       HostName your-server-ip
       User username
       # 如果有特定的私鑰路徑可指定，若使用 Bitwarden Agent 則可省略
       # IdentityFile ~/.ssh/id_ed25519
   ```

3. 儲存後，您就可以直接使用別名登入：
   ```bash
   ssh bot
   ```

## 疑難排解

如果無法連線，請檢查：

1. **權限問題**：確認 `~/.ssh` 為 700，`authorized_keys` 為 600。
2. **SSHD 設定**：檢查 `/etc/ssh/sshd_config` 確保已啟用金鑰登入：
   ```
   PubkeyAuthentication yes
   AuthorizedKeysFile .ssh/authorized_keys
   ```
   (修改設定後需重啟 ssh 服務: `sudo systemctl restart sshd`)
