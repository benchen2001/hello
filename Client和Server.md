是的，Windows 11 內建的「Windows Time」服務 (w32time) 可以同時提供 **Client (客戶端)** 和 **Server (伺服器)** 的功能。這表示您的 Windows 11 電腦既可以從外部 NTP 伺服器同步時間，也可以作為 NTP 伺服器向網路中的其他設備提供時間。

-----

### 如何設定 Windows 11 作為 NTP Client

預設情況下，Windows 11 會自動作為客戶端，從 Microsoft 的時間伺服器 (`time.windows.com`) 或網域環境中的 PDC Emulator 進行時間同步。如果您想手動設定其他 NTP 伺服器，可以使用以下步驟：

1.  **開啟管理員模式的命令提示字元或 PowerShell：**

      * 在開始菜單搜尋 "cmd" 或 "PowerShell"，然後右鍵點選並選擇「**以系統管理員身分執行**」。

2.  **設定手動同步的 NTP 伺服器：**

      * 輸入以下指令，將 `<NTP_SERVER_ADDRESS>` 替換為您想要使用的 NTP 伺服器地址 (例如 `pool.ntp.org` 或您的內部 NTP 伺服器 IP)：

    <!-- end list -->

    ```cmd
    w32tm /config /syncfromflags:manual /manualpeerlist:"<NTP_SERVER_ADDRESS>" /update
    ```

      * 如果您有多個 NTP 伺服器，可以用空格分隔它們並用引號括起來，例如：
        ```cmd
        w32tm /config /syncfromflags:manual /manualpeerlist:"ntpserver1.example.com ntpserver2.example.com" /update
        ```

3.  **重新啟動 Windows Time 服務：**

      * 輸入以下指令以套用變更：

    <!-- end list -->

    ```cmd
    net stop w32time
    net start w32time
    ```

4.  **強制同步時間 (可選)：**

      * 您可以立即強制同步時間：

    <!-- end list -->

    ```cmd
    w32tm /resync
    ```

5.  **檢查時間同步狀態：**

      * 輸入以下指令查看目前的時間來源和同步狀態：

    <!-- end list -->

    ```cmd
    w32tm /query /status
    w32tm /query /source
    ```

-----

### 如何設定 Windows 11 作為 NTP Server

將 Windows 11 設定為 NTP 伺服器主要透過修改登錄檔來完成。請**謹慎操作登錄檔**，因為錯誤的修改可能會導致系統問題。建議在修改前備份登錄檔。

1.  **開啟管理員模式的命令提示字元或 PowerShell：**

      * 在開始菜單搜尋 "cmd" 或 "PowerShell"，然後右鍵點選並選擇「**以系統管理員身分執行**」。

2.  **停止 Windows Time 服務：**

      * 在進行登錄檔修改前，請先停止服務：

    <!-- end list -->

    ```cmd
    net stop w32time
    ```

3.  **修改登錄檔設定：**

      * 您可以透過指令直接修改，或開啟 `regedit` 手動修改。以下是透過指令修改的方法：

      * **啟用 NTP 伺服器功能：**

        ```cmd
        reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer" /v Enabled /t REG_DWORD /d 1 /f
        ```

      * **設定 AnnounceFlags：**
        這個設定告訴其他設備該電腦是一個可靠的時間伺服器。將 `AnnounceFlags` 設定為 `5` (十進制) 通常表示該電腦是一個始終可靠的時間伺服器。

        ```cmd
        reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config" /v AnnounceFlags /t REG_DWORD /d 5 /f
        ```

      * **設定 Type 為 NTP (確保您的電腦從外部 NTP 伺服器同步)：**
        如果您的電腦需要從外部 NTP 伺服器同步時間，而不是使用網域階層 (NT5DS)，請設定 `Type` 為 `NTP`。

        ```cmd
        reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" /v Type /t REG_SZ /d "NTP" /f
        ```

      * **設定 LocalNTP (可選，通常在伺服器上設定為 1)：**
        某些指南會建議設定 `LocalNTP` 為 `1`，但主要功能由 `NtpServer\Enabled` 和 `AnnounceFlags` 控制。

        ```cmd
        reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" /v LocalNTP /t REG_DWORD /d 1 /f
        ```

      * **調整 ClockDispersion (可選)：**
        在某些情況下，如果您的客戶端（特別是非 Windows 設備）無法從您的 Windows NTP 伺服器獲取時間，可能是因為 Windows 的預設 `ClockDispersion` 值較高。您可以嘗試將其設定為較低的值，例如 `2`。

        ```cmd
        reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config" /v ClockDispersion /t REG_DWORD /d 2 /f
        ```

4.  **設定 Windows Time 服務為自動啟動：**

    ```cmd
    sc config w32time start= auto
    ```

5.  **啟動 Windows Time 服務：**

    ```cmd
    net start w32time
    ```

6.  **檢查服務狀態：**

      * 確保 Windows Time 服務正在運行：

    <!-- end list -->

    ```cmd
    sc query w32time
    ```

-----

### 注意事項

  * **防火牆：** 確保您的 Windows 11 防火牆允許 UDP 埠 123 的傳入和傳出流量，這是 NTP 服務使用的埠。
  * **可靠性：** Windows 11 作為 NTP 伺服器在小型環境或測試環境中可能夠用，但在生產環境中，建議使用專門的 NTP 伺服器軟體或 Windows Server 上的 PDC Emulator，因為它們通常提供更精確和穩定的時間服務。
  * **網路環境：** 如果您的 Windows 11 電腦有多個網路介面卡，w32time 無法根據特定的介面卡啟用。它會監聽所有可用的網路介面卡上的 UDP 埠 123。
  * **同步來源：** 當您將 Windows 11 設定為 NTP 伺服器時，請確保它本身從一個可靠的外部時間源同步，這樣它才能為其他客戶端提供準確的時間。

希望這些設定對您有幫助！如果您遇到任何問題，可以提供更具體的情況，我可以進一步協助。
