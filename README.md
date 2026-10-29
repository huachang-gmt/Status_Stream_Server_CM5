# Status Stream Server CM5

## 1. 專案簡介

`Status_Stream_Server_CM5` 是執行於 Raspberry Pi CM5 的 C++ Status Stream 接收程式。

本專案目前的主要工作，是接收 STM32H755 CM7 經由 USART6 傳送的 Status Stream 封包，並進行封包標頭、Payload 格式及 CRC16 檢查。

本專案採用 C++17 與 CMake 建置。

目前已完成的功能包括：

* 開啟 Raspberry Pi CM5 的 UART 裝置。
* 設定 UART 為 115200 baud、8 data bits、No parity、1 stop bit。
* 使用 `poll()` 與 `read()` 接收串列資料。
* 處理 UART 分段接收的資料。
* 尋找 Status Stream 封包標頭。
* 驗證封包標頭及 Payload 長度。
* 驗證 Payload 的格式標記及結尾。
* 計算並驗證 CRC16。
* 檢查封包序號。
* 顯示有效封包資訊及 Payload 預覽。
* 統計接收位元組數與封包錯誤數。

本階段已經完成 STM32H755 CM7 至 Raspberry Pi CM5 的實機 UART 傳輸驗證。

**注意：** 目前版本已完成 UART 接收與封包驗證，但尚未完成將有效封包透過 TCP 轉送至既有 Qt Status Stream Client 的整合與實機驗證。

---

## 2. 系統架構

### 2.1 目前已驗證的傳輸路徑

```text
┌────────────────────────────────────┐
│          STM32H755 CM7             │
│                                    │
│  Status Stream Mock Samples        │
│              │                     │
│              ▼                     │
│  Status Stream Packet Builder      │
│              │                     │
│              ▼                     │
│  151-byte Packet + CRC16           │
│              │                     │
│              ▼                     │
│  USART6 TX / PC6                   │
└──────────────┬─────────────────────┘
               │
               │ UART
               │ 115200 baud
               │ 8N1
               ▼
┌────────────────────────────────────┐
│       Raspberry Pi CM5             │
│                                    │
│  GPIO15 / RXD0 / Physical Pin 10   │
│              │                     │
│              ▼                     │
│        /dev/ttyAMA0                │
│              │                     │
│              ▼                     │
│     poll() / read()                │
│              │                     │
│              ▼                     │
│      Packet Parser                 │
│              │                     │
│              ▼                     │
│  Header / Payload / CRC16 Check    │
│              │                     │
│              ▼                     │
│       Valid Packet Output          │
└────────────────────────────────────┘
```

### 2.2 完整系統的目標架構

後續完成 TCP 轉送功能後，整體 Status Stream 資料路徑預計如下：

```text
┌───────────────────────────────────┐
│          STM32H755 CM7            │
│                                   │
│  Status Data                      │
│       │                           │
│       ▼                           │
│  Packet Builder                   │
│       │                           │
│       ▼                           │
│  USART6 TX / PC6                  │
└──────────────┬────────────────────┘
               │ UART 115200 8N1
               ▼
┌───────────────────────────────────┐
│       Raspberry Pi CM5            │
│                                   │
│  /dev/ttyAMA0                     │
│       │                           │
│       ▼                           │
│  UART Receiver                    │
│       │                           │
│       ▼                           │
│  Packet Parser                    │
│       │                           │
│       ▼                           │
│  Header / Payload / CRC16 Check   │
│       │                           │
│       ▼                           │
│  TCP Server                       │
└──────────────┬────────────────────┘
               │ Ethernet / TCP
               ▼
┌───────────────────────────────────┐
│      Qt Status Stream Client      │
│                                   │
│  TCP Receive                      │
│       │                           │
│       ▼                           │
│  Packet Parser                    │
│       │                           │
│       ▼                           │
│  Status Data Decode               │
│       │                           │
│       ▼                           │
│  GUI Display                      │
└───────────────────────────────────┘
```

以上第二張圖描述的是完整系統的目標架構，不代表 CM5 的 TCP Server 已經包含在目前版本內。

既有 Qt Status Stream Client 已經具有 Status Stream 封包解析與 GUI 顯示功能。後續整合時，應盡可能保留原有 Qt Client 的封包解析及顯示邏輯，避免重複修改已驗證的功能。

---

## 3. 硬體連接

### 3.1 STM32H755 至 Raspberry Pi CM5

| STM32H755       | Raspberry Pi CM5                | 說明         |
| --------------- | ------------------------------- | ---------- |
| USART6 TX / PC6 | GPIO15 / RXD0 / Physical Pin 10 | UART 資料傳送  |
| GND             | GND                             | 共地         |
| USART6 RX       | 目前接收測試不需要                       | 本階段為單向資料傳輸 |

UART 設定：

| 項目           | 設定             |
| ------------ | -------------- |
| Baud rate    | 115200         |
| Data bits    | 8              |
| Parity       | None           |
| Stop bits    | 1              |
| Flow control | None           |
| Linux device | `/dev/ttyAMA0` |

**電氣注意事項：** 確認 STM32 與 Raspberry Pi 的 UART 電壓規格相容。Raspberry Pi GPIO 使用 3.3 V 邏輯，不可將不相容的電壓直接接入 GPIO。

---

## 4. Status Stream 封包格式

每個 Status Stream 封包固定為 151 bytes。

|  Offset |        長度 | 欄位             | 說明                          |
| ------: | --------: | -------------- | --------------------------- |
|       0 |    1 byte | Magic 0        | `0x47`，ASCII `G`            |
|       1 |    1 byte | Magic 1        | `0x53`，ASCII `S`            |
|       2 |    1 byte | Version        | `0x01`                      |
|       3 |    1 byte | Type           | `0x01`，Status Stream        |
|     4–7 |   4 bytes | Sequence       | uint32，Little-Endian        |
|     8–9 |   2 bytes | Payload Length | uint16，Little-Endian，固定 139 |
|  10–148 | 139 bytes | Payload        | ASCII HEX 格式資料              |
| 149–150 |   2 bytes | CRC16          | uint16，Little-Endian        |

總長度：

```text
10-byte Header
+ 139-byte Payload
+ 2-byte CRC16
= 151 bytes
```

### 4.1 Payload 格式

Payload 長度固定為 139 bytes，內容如下：

```text
> + 136 ASCII HEX characters + CR LF
```

| Payload 相對 Offset |        長度 | 內容                                   |
| ----------------: | --------: | ------------------------------------ |
|                 0 |    1 byte | `>`                                  |
|             1–136 | 136 bytes | 68-byte 原始資料轉換而成的 136 個 ASCII HEX 字元 |
|               137 |    1 byte | `\r`                                 |
|               138 |    1 byte | `\n`                                 |

原始資料為 68 bytes。

每個原始 byte 轉換為兩個大寫 ASCII HEX 字元，因此：

```text
68 bytes × 2 = 136 ASCII HEX characters
```

加上開頭的 `>` 與結尾的 CRLF：

```text
1 + 136 + 2 = 139 bytes
```

### 4.2 CRC16 規格

CRC16 採用以下設定：

| 項目                   | 設定                  |
| -------------------- | ------------------- |
| Initial value        | `0xFFFF`            |
| Reflected polynomial | `0xA001`            |
| CRC coverage         | Packet offset 0–148 |
| CRC length           | 2 bytes             |
| CRC storage          | Little-Endian       |

CRC 計算範圍包含：

* Magic
* Version
* Type
* Sequence
* Payload Length
* 完整 Payload

不包含最後兩個 CRC bytes。

CM5 重新計算 CRC16，並與封包內的 CRC 值比較。兩者相同才視為 CRC 驗證通過。

---

## 5. CM5 接收程式的工作流程

目前程式位於：

```text
src/main.cpp
```

主要處理流程如下。

### Step 1：開啟 UART

程式開啟：

```text
/dev/ttyAMA0
```

並設定：

```text
115200 baud, 8N1
```

使用非阻塞方式開啟裝置，避免 UART 暫時沒有資料時卡住整個程式。

### Step 2：等待接收事件

程式使用 Linux `poll()` 等待 UART 接收事件。

當 UART 有資料可讀時，再使用 `read()` 讀取資料。

這樣可以避免不斷執行無意義的讀取迴圈。

### Step 3：處理分段資料

UART 的一次 `read()` 不保證恰好取得一個完整封包。

可能遇到：

* 一次只收到部分封包。
* 一次收到一個完整封包。
* 一次收到多個封包的資料。

因此，程式將收到的 bytes 放入接收緩衝區，再依照封包規格解析。

### Step 4：尋找 Magic

程式尋找以下兩個連續 bytes：

```text
0x47 0x53
```

也就是 ASCII：

```text
GS
```

如果前面存在無法辨識的資料，程式會跳過這些資料，繼續尋找下一個封包標頭。

### Step 5：等待完整封包

找到 Magic 後，程式必須取得完整的 151 bytes 才能進行完整驗證。

如果資料尚未收齊，就保留在緩衝區，等待下一次 UART 接收。

### Step 6：驗證封包標頭

檢查：

* Magic 是否為 `GS`。
* Version 是否為 `0x01`。
* Type 是否為 `0x01`。
* Payload Length 是否為 139。

### Step 7：驗證 Payload 格式

檢查：

* Payload 開頭是否為 `>`。
* Payload 結尾是否為 CRLF。

### Step 8：驗證 CRC16

重新計算封包 offset 0–148 的 CRC16。

如果計算結果與 offset 149–150 儲存的 CRC 不一致，該封包即判定為 CRC 錯誤。

### Step 9：顯示有效封包

通過驗證後，程式會輸出：

* 有效封包編號。
* Sequence。
* Payload 長度。
* CRC16。
* 累計接收位元組數。
* Payload 前段預覽。

### Step 10：移除已處理封包

成功解析封包後，程式移除已處理的 151 bytes，繼續解析緩衝區中剩餘的資料。

因此，即使一次讀取包含多個封包，也能逐一處理。

---

## 6. 編譯環境

本專案使用：

* Raspberry Pi CM5
* Linux
* C++17
* CMake 3.16 或更新版本
* GNU C++ 編譯器

專案結構：

```text
Status_Stream_Server_CM5/
├── .gitignore
├── CMakeLists.txt
├── README.md
└── src/
    └── main.cpp
```

`build/` 是建置產生的目錄，不應作為原始碼的一部分提交至 Git。

---

## 7. 編譯方式

進入專案目錄：

```bash
cd /home/herman/Status_Stream_Server_CM5
```

建立建置目錄：

```bash
cmake -S . -B build
```

編譯：

```bash
cmake --build build
```

編譯成功後，執行檔位於：

```text
build/Status_Stream_Server_CM5
```

---

## 8. 執行方式

確認 STM32H755 CM7 已上電並執行 Status Stream 傳送程式，而且 UART 線路已正確連接。

啟動接收程式：

```bash
cd /home/herman/Status_Stream_Server_CM5
./build/Status_Stream_Server_CM5
```

正常啟動時，應看到類似：

```text
UART device : /dev/ttyAMA0
UART format : 115200 baud, 8N1
Packet size : 151 bytes
Waiting for Status Stream packets... Press Ctrl+C to exit.
```

有效封包會以類似以下格式顯示：

```text
VALID packet #1: seq=0, payload=139, CRC=0xEE15, total_bytes=1347
  Payload preview: >0000000340000000...
```

其中 CRC、Sequence 及 Payload 預覽會依照實際接收資料而改變。

結束程式時按下：

```text
Ctrl+C
```

程式會顯示接收統計。

---

## 9. 實機測試結果

### 9.1 測試項目

本階段已完成以下測試：

| 測試項目                   | 結果   |
| ---------------------- | ---- |
| STM32H755 CM7 Build    | PASS |
| CM5 CMake Build        | PASS |
| USART6 至 CM5 UART 硬體連接 | PASS |
| 115200 baud、8N1 接收     | PASS |
| 151-byte 封包解析          | PASS |
| Magic、Version、Type 驗證  | PASS |
| Payload Length 驗證      | PASS |
| Payload 開頭及結尾驗證        | PASS |
| CRC16 驗證               | PASS |
| 連續接收 100 個有效封包         | PASS |

### 9.2 100 個封包的接收統計

實機測試結果：

```text
UART receiver stopped.
Total bytes: 15100
Valid packets: 100
Invalid headers: 0
Payload errors: 0
CRC errors: 0
```

結果說明：

```text
100 packets × 151 bytes = 15100 bytes
```

實際接收位元組數與預期完全一致。

100 個封包全部通過目前程式的封包格式及 CRC16 驗證。

這代表目前已經驗證 STM32H755 CM7 至 Raspberry Pi CM5 的基本 UART 傳輸與封包驗證流程。

### 9.3 尚未完成的驗證

以下項目不應因為上述測試成功，就直接標記為 PASS：

* 五組模擬資料所有欄位的逐一比對。
* 原始 68-byte 資料完整解碼與內容核對。
* Sequence 遺失、重複或跳號統計。
* CM5 TCP Server 轉送。
* 既有 Qt Status Stream Client 的端到端整合測試。
* 後續改用 STM32H755 CM7 的實際狀態資料。

---

## 10. Qt Status Stream Client 整合說明

既有 Qt Status Stream Client 負責接收 TCP 資料、解析 Status Stream 封包、解碼狀態資料並顯示於 GUI。

完整系統的目標資料路徑為：

```text
STM32H755 CM7
    │
    │ USART6 / UART
    ▼
Raspberry Pi CM5
    │
    │ UART Receive
    ▼
Packet Validation
    │
    │ Valid Packet
    ▼
TCP Server
    │
    │ Ethernet / TCP
    ▼
Qt Status Stream Client
    │
    ▼
Status Packet Parser
    │
    ▼
Status Data Decode
    │
    ▼
GUI Display
```

### 整合原則

1. STM32H755 CM7 負責產生 Status Stream 封包。
2. CM5 負責接收 UART 資料並驗證封包。
3. 後續 TCP Server 應將通過驗證的完整封包轉送給 Qt Client。
4. TCP 是串流傳輸，Qt Client 不能假設一次 TCP `read()` 就等於一個完整封包。
5. Qt Client 應繼續依照既有封包規格尋找 Magic、判斷封包長度並驗證資料。
6. 應盡可能保留已經驗證成功的 Qt Client 解析及 GUI 顯示功能。

**目前版本的 CM5 程式尚未實作上述 TCP 轉送流程，因此不能單靠目前的 UART 測試結果宣告 Qt 整合完成。**

---

## 11. 後續開發順序

建議依照以下順序繼續：

### 階段 A：完成 UART 接收驗證

* 確認 151-byte 封包格式。
* 確認 CRC16。
* 確認連續接收穩定性。

目前此階段已通過 100 個封包的實機測試。

### 階段 B：驗證五組模擬資料

* 核對五組資料的循環順序。
* 確認 68-byte 原始資料內容。
* 確認每個欄位的解碼結果。

### 階段 C：整合 CM5 TCP Server

* 接收並驗證 UART 封包。
* 將完整有效封包透過 TCP 傳送。
* 避免改變既有的 151-byte 封包格式。
* 處理 TCP 分段接收與連線狀態。

### 階段 D：整合 Qt Status Stream Client

* 連接 CM5 TCP Server。
* 確認 Qt Client 能持續收到資料。
* 驗證 GUI 顯示與原有解碼結果。
* 測試 TCP 斷線及重新連線。

### 階段 E：改用實際狀態資料

* 以同事提供的實際狀態資料取代模擬資料。
* 維持既有封包格式。
* 重新驗證 STM32、CM5 及 Qt Client 的完整資料路徑。

---

## 12. Git 版本管理

本專案應追蹤：

* `CMakeLists.txt`
* `src/main.cpp`
* `README.md`
* `.gitignore`

不應將一般建置產物及本機備份納入正式版本。

上傳前建議先檢查：

```bash
cd /home/herman/Status_Stream_Server_CM5
git status --short
```

檢查忽略規則：

```bash
git check-ignore build/
```

檢查專案檔案：

```bash
git ls-files
```

請在確認檔案清單正確後，再自行執行 Git add、commit 與 push。

---

## 13. 目前專案狀態摘要

目前已經完成：

* STM32H755 CM7 Status Stream 封包產生。
* USART6 UART 傳送。
* CM5 UART 接收。
* 151-byte 封包解析。
* CRC16 驗證。
* 100 個有效封包的實機驗證。

目前仍待完成：

* 五組模擬資料完整內容核對。
* CM5 TCP Server。
* Qt Status Stream Client 端到端驗證。
* 實際狀態資料整合。

本專案以逐步驗證、維持既有封包規格及避免不必要修改為開發原則。
