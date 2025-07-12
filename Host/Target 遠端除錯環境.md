好的，身為專業的 Win10 驅動程式設計師，這套 Host/Target 遠端除錯環境是我們日常工作的基礎。您提出的需求「讓程式跑到 source code 第一行就中斷」是核心目標。

以下我將詳細說明如何從零開始架設這套系統，並回答您的所有問題。

我們將分為三個主要部分：

1.  **架設測試機 (Target Computer)**：使其進入可被除錯的模式。
2.  **架設開發機 (Host Computer)**：設定 WinDbg 以連線到測試機。
3.  **執行除錯與中斷**：如何載入驅動程式並讓它停在 `DriverEntry`。

-----

### 名詞定義

  * **開發機 (Host)**：您用來編寫程式、執行 Visual Studio 與 WinDbg Client 的電腦。
  * **測試機 (Target)**：您用來安裝並執行待測驅動程式的電腦。這是被除錯的對象。

-----

### 第 1 部分：架設測試機 (Target)

這是整個設定中最關鍵的一步。我們的目標是開啟測試機的「核心除錯模式 (Kernel Debugging)」，並設定連線方式。現今最推薦、速度也最快的方式是透過網路 (KDNET)。

#### 步驟 1：在測試機上安裝必要工具

您需要在測試機上安裝 Windows SDK 或 WDK，主要是為了取得 `kdnet.exe` 這個工具。

  * 從 [Microsoft 官網下載 Windows Driver Kit (WDK)](https://learn.microsoft.com/zh-tw/windows-hardware/drivers/download-the-wdk)。
  * 安裝 WDK。

#### 步驟 2：設定網路核心除錯 (KDNET)

1.  在測試機上，以**系統管理員身分**開啟「命令提示字元 (Command Prompt)」。

2.  找到您測試機的有線網路卡。執行 `ipconfig` 查看其 IP 位址，假設為 `192.168.1.100`。

3.  執行 `kdnet.exe` 指令來自動設定。這個指令會掃描相容的網路卡並提供設定指令。

    ```shell
    C:\> kdnet.exe
    ```

4.  `kdnet.exe` 的輸出會類似這樣 (這是一個範例)：

    ```
    Network debugging is supported on the following NICs:
    busparams=1.0.0, [Some NIC Description], Intel(R) Ethernet Connection I219-LM

    To configure network debugging for the NIC referenced by busparams=1.0.0,
    run the following command in an elevated Command Prompt.

    "bcdedit /dbgsettings net hostip:w.x.y.z port:n"

    where w.x.y.z is the IP address of the debugger host computer,
    and n is a port number from 49152 to 65535.

    For example:

    bcdedit /dbgsettings net hostip:192.168.1.10 port:50000

    Then, on the debugger host computer, run "windbg -k net:port=50000,key=2.2.2.2.2.2.2.2"
    A key will be automatically generated and displayed after the dbgsettings
    command is entered.
    ```

5.  根據 `kdnet.exe` 的建議，執行 `bcdedit` 指令。您需要提供**開發機 (Host)** 的 IP 位址。假設您的開發機 IP 是 `192.168.1.10`，您可以選擇一個 50000-50039 之間的 Port，例如 `50005`。

    ```shell
    C:\> bcdedit /dbgsettings net hostip:192.168.1.10 port:50005
    ```

6.  執行完上述指令後，系統會自動生成一把金鑰 (Key)。**請務必把這串金鑰抄寫或複製下來**，開發機會需要它。

    ```
    Key=1.2.3.4.1.2.3.4  <-- 這串金鑰非常重要！
    ```

7.  現在，正式啟用除錯模式：

    ```shell
    C:\> bcdedit /debug on
    ```

8.  **重新啟動測試機**。這一步是必須的，重啟後核心才會以除錯模式載入。

#### **如何確定測試機在監聽模式？**

這是一個很好的問題，也是新手容易混淆的地方。
**測試機上並沒有一個叫 `WinDbg.exe` 的程式在「監聽」**。

所謂的「監聽模式」，其實是 Windows 核心 (Kernel) 本身在開機後進入了一種\*\*「可被除錯的狀態」\*\*。它會根據您用 `bcdedit` 設定的參數 (例如網路)，等待來自開發機的除錯器連線。

您可以透過以下方式來**確認設定是否成功**：

1.  **檢查 BCD 設定**：在測試機重啟後，再次以系統管理員開啟 CMD，輸入以下指令：
    ```shell
    C:\> bcdedit /dbgsettings
    ```
    您應該會看到類似以下的輸出，確認 `debug` 項目為 `Yes`，且 `hostip`、`port`、`key` 等資訊都正確無誤。
    ```
    debug           Yes
    dbgsettings     net hostip:192.168.1.10 port:50005
    key             1.2.3.4.1.2.3.4
    ```
2.  **連線測試**：最直接的確認方法，就是直接從開發機的 WinDbg 嘗試連線。如果能成功連上並中斷，就代表測試機已經處於等待連線的狀態。

-----

### 第 2 部分：架設開發機 (Host)

開發機是您操作的主力。

#### 步驟 1：安裝工具

確保您的開發機已安裝：

  * Visual Studio (含 C++ 開發環境)
  * Windows Driver Kit (WDK) (這會整合 WinDbg Preview)

#### 步驟 2：設定 WinDbg 連線

1.  在開發機上，開啟 **WinDbg Preview**。
2.  點擊 `File` -\> `Start debugging` -\> `Attach to kernel`。
3.  切換到 `Net` 標籤頁。
4.  填入您在測試機上得到的資訊：
      * **Port Number**：`50005` (您設定的埠號)
      * **Key**：`1.2.3.4.1.2.3.4` (您抄下來的金鑰)
5.  點擊 `OK`。

此時，WinDbg 會開始嘗試連線。如果測試機已經重啟並在等待狀態，WinDbg 會成功連線並顯示 `[Kernel Connected]`，然後會立刻中斷在測試機的某個核心初始位置，命令提示符會變成 `kd>`。

#### 步驟 3：設定符號檔 (Symbol) 與原始碼路徑

這是讓除錯器能夠對應到您原始碼的關鍵。

1.  **設定符號路徑 (Symbol Path)**：在 WinDbg 的命令列輸入：

    ```kd
    .sympath+ SRV*c:\symbols*http://msdl.microsoft.com/download/symbols
    .sympath+ C:\path\to\your\driver\project\x64\Debug
    .reload /f
    ```

      * 第一行是設定 Microsoft 的公用符號伺服器，並在本機 `c:\symbols` 目錄下快取。這是必要的，否則您只能看到自己 driver 的 call stack。
      * 第二行請換成您自己驅動程式專案編譯後 `.pdb` 檔案所在的目錄。
      * `.reload /f` 是強制重新載入符號。

2.  **設定原始碼路徑 (Source Path)**：

    ```kd
    .srcpath+ C:\path\to\your\driver\project\
    ```

      * 將路徑指向您驅動程式原始碼的根目錄。

-----

### 第 3 部分：載入 Driver 並中斷在第一行

現在萬事俱備，只差載入您的驅動程式並觸發中斷。

#### 步驟 1：設定初始中斷點

您的驅動程式此時還沒有被載入測試機的記憶體中，所以標準的中斷點 `bp` 會失敗。我們需要使用「未解析的中斷點」(Unresolved Breakpoint) `bu`。

驅動程式的入口點函式通常是 `DriverEntry`。假設您的驅動程式模組名稱是 `MyDriver.sys`。

在 WinDbg (`kd>`) 中輸入以下指令：

```kd
bu MyDriver!DriverEntry
```

  * `bu` 的意思是，當中斷器未來載入一個名為 `MyDriver.sys` 的模組時，請在 `DriverEntry` 這個函式上設定中斷點。

#### 步驟 2：讓測試機繼續執行

設定好 `bu` 之後，測試機目前仍處於中斷狀態。輸入 `g` (Go) 並按 Enter，讓測試機繼續執行。

```kd
kd> g
```

測試機現在會繼續開機或回到桌面，等待您安裝驅動程式。

#### 步驟 3：在測試機上安裝並啟動驅動程式

1.  將您編譯好的 `MyDriver.sys` (以及 `MyDriver.inf` 等相關檔案) 複製到測試機上。
      * **專業提示**：您可以在 Visual Studio 的專案屬性中設定「部署 (Deployment)」，讓 VS 在編譯後自動將驅動程式檔案複製到測試機並安裝，這能大幅提高效率。
2.  在測試機上，使用 `sc` 指令或 `devcon.exe` 工具來安裝和啟動您的驅動程式。
      * **使用 `sc` (Service Control)**：
        ```shell
        sc create MyDriver type= kernel binPath= C:\path\on\target\MyDriver.sys
        sc start MyDriver
        ```

#### 步驟 4：觸發中斷！

當您在測試機上執行 `sc start MyDriver` 的那一刻，Windows 的 I/O Manager 會載入 `MyDriver.sys` 並呼叫其 `DriverEntry` 函式。

因為您先前已經設定了 `bu MyDriver!DriverEntry`，開發機上的 WinDbg 會立刻收到通知，**執行將會精準地中斷在您 `DriverEntry` 函式的第一行原始碼上！**

此時，您就可以開始您的除錯工作了：檢查變數、設定其他中斷點、單步執行 (F10/F11) 等。

### 總結流程

1.  **測試機**：`kdnet.exe` -\> `bcdedit /dbgsettings` -\> `bcdedit /debug on` -\> Reboot。
2.  **開發機**：開 WinDbg -\> Attach to Kernel (輸入 IP/Port/Key)。
3.  **WinDbg**：設定 `.sympath` 和 `.srcpath`。
4.  **WinDbg**：設定 `bu MyDriver!DriverEntry`。
5.  **WinDbg**：輸入 `g` 讓測試機跑起來。
6.  **測試機**：複製驅動程式檔案，用 `sc start` 啟動服務。
7.  **開發機**：WinDbg 成功中斷在 `DriverEntry`。
