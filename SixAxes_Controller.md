這份文件非常重要，因為它描述的是 **SixAxes 六軸控制器如何將 EtherCAT 系統的即時狀態，透過 TCP 傳送給上位機監控軟體**。

我會採用「原文翻譯 → 技術原理 → 實際資料解析 → 網路補充 → 與 STM32H755 / CM5 / Qt 專案的關聯」的方式說明。

不會只翻譯英文，而是希望讓你真正理解這份通訊協定，包括為什麼這樣設計、資料如何排列，以及接收端應該如何解析。

# 第一部分：Routine Status Packet 是什麼？

## 1. 原文翻譯

### SixAxes Controller EtherCAT Command Manual

### 2. Routine Status Packet Format Description

Routine Packet（例行狀態封包）提供系統即時狀態摘要，包含：

* 控制器狀態（Controller Status）
* I/O 訊號
* 各軸位置（Axis Positions）
* 夾爪狀態（Gripper States）
* 相機座標（Camera Coordinates）

系統每隔 200ms，會將這個封包以 HEX String（十六進位字串）的格式傳送到：

`localhost:8888`

讓上位機（Host）或監控軟體接收、解析並即時顯示系統狀態。

## 2. 技術解釋

這段話包含幾個重要觀念。

### A. Routine Status Packet

Routine 可以理解為例行性、週期性的工作。

因此，Routine Status Packet 就是：

**控制器定期主動回報目前系統狀態的資料封包。**

它不是由上位機下達命令後才回傳，而是控制器按照固定週期持續傳送。

例如：

| 時間     | 控制器傳送的資料 |
| ------ | -------- |
| 0 ms   | 第一次狀態封包  |
| 200 ms | 第二次狀態封包  |
| 400 ms | 第三次狀態封包  |
| 600 ms | 第四次狀態封包  |
| 800 ms | 第五次狀態封包  |

因此：

200ms = 0.2 秒

每秒傳送：

$$
\frac{1000}{200}=5
$$

也就是每秒 5 次，稱為 **5Hz 更新頻率**。

這裡要注意：

200ms 是控制器傳送狀態封包的週期，不代表 EtherCAT 本身的通訊週期也是 200ms。

EtherCAT 底層可能以更高頻率交換資料，再由上層程式每 200ms 擷取一次資料並傳送。

### B. localhost:8888

這裡使用的是 TCP/IP 網路通訊的概念。

`localhost` 代表本機（Loopback Address），通常對應：

`127.0.0.1`

`8888` 則是 TCP 或 UDP 通訊所使用的 Port Number（連接埠號碼）。

從文件描述來看，控制器會將資料傳送到本機的 8888 Port。

例如：

```text
SixAxes Controller
        |
        | Routine Status Packet
        | 每 200ms 傳送一次
        |
        v
127.0.0.1:8888
        |
        v
Host / Monitoring Software
        |
        v
解析資料並顯示狀態
```

這裡有一個值得注意的地方：

`localhost` 只代表控制器所在的本機，不代表 Raspberry Pi CM5，也不代表其他電腦。

如果控制器和 Qt Client 位於不同電腦，就需要確認文件是否還有描述 TCP Server、TCP Client，以及實際 IP 的設定方式。

不能直接假設其他電腦也能使用 `localhost:8888` 連線。

---

# 第二部分：Routine Status Packet 的封包結構

## 1. 原文翻譯

### 2.1 Packet Overview

Routine Packet 是固定長度的 HEX String，總長度為：

* 68 Bytes
* 136 個 HEX Characters

資料按照固定順序排列。

### 2.2 Routine Status Packet Format Description

Routine Status Packet 是控制器回傳的固定長度狀態字串，用來表示：

* 系統狀態
* 感測器數值
* 各軸位置

封包具有以下特性：

1. 開頭一定是 `>` 字元。
2. 結尾一定是 Carriage Return + Line Feed，也就是 `\r\n`。
3. 所有欄位都使用連續的 HEX String 編碼。
4. 各欄位按照 Byte 邊界排列。
5. 資料順序固定，可以讓接收端使用固定的解析方式。

## 2. 先理解 HEX String 與 Binary Data 的差異

這是整份文件非常重要的觀念。

假設控制器要傳送一個 Byte：

```text
0x41
```

如果使用 Binary Data 傳送：

實際只需要傳送 1 Byte：

```text
01000001
```

如果使用 HEX String 傳送：

則會將它轉換成兩個 ASCII 字元：

```text
41
```

實際傳送的是：

| 字元  | ASCII HEX |
| --- | --------- |
| '4' | 0x34      |
| '1' | 0x31      |

因此：

**1 Byte Binary Data = 2 個 HEX Characters。**

所以文件中：

68 Bytes × 2 = 136 HEX Characters。

這也是為什麼文件同時提到：

* 68 Bytes
* 136 Characters

兩者並不矛盾。

### 實際封包格式

假設傳送的資料如下：

```text
>0102030405060708090A...最後一個Byte\r\n
```

這個封包包含：

| 組成        |             長度 |
| --------- | -------------: |
| 開頭 `>`    |         1 Byte |
| HEX 資料    | 136 Characters |
| 結尾 `\r\n` |        2 Bytes |
| 合計        |      139 Bytes |

因此，這裡要區分兩種長度：

* 資料本體：68 Bytes。
* 實際 ASCII 傳輸封包：139 Bytes。

這個差異在撰寫 C++ TCP 接收程式時非常重要。

不能把 68 Bytes 當成實際 TCP 接收長度。

---

# 第三部分：68 Bytes 的資料排列方式

這是整份文件最核心的部分。

原文 Table 1：

| Section               | Data Type | Total Bytes | Description    |
| --------------------- | --------- | ----------: | -------------- |
| Controller Status     | UINT32    |           4 | 32-bit 控制器狀態   |
| Analog Input (AI)     | UINT16    |          16 | 一個 AI 裝置，8 個通道 |
| Main Stages Positions | DOUBLE    |          48 | 一個六軸平台的六個軸位置   |
| Total                 |           |          68 | 總長度            |

我們逐一分析。

## 1. Controller Status：4 Bytes

資料型態：

`UINT32`

意思是：

Unsigned Integer 32-bit，32 位元無號整數。

一個 UINT32：

$$
32 \div 8 = 4\ Bytes
$$

因此，控制器狀態使用 4 Bytes 表示。

例如：

```text
00000003
```

這是 8 個 HEX Characters，代表 4 Bytes。

它並不是一個單純的數字，而是由 32 個 Bit 組成的狀態旗標（Status Flags）。

每個 Bit 可以表示不同的控制器狀態。

稍後第三部分會詳細說明 Bit 的意義。

## 2. Analog Input (AI)：16 Bytes

資料型態：

`UINT16`

代表 16-bit 無號整數。

每個 AI Channel 使用：

$$
16 \div 8 = 2\ Bytes
$$

文件指出：

一個 AI 裝置有 8 個 Channels。

所以：

$$
8 \times 2 = 16\ Bytes
$$

資料排列如下：

| AI Channel   | Data Type | Bytes |
| ------------ | --------- | ----: |
| AI Channel 0 | UINT16    |     2 |
| AI Channel 1 | UINT16    |     2 |
| AI Channel 2 | UINT16    |     2 |
| AI Channel 3 | UINT16    |     2 |
| AI Channel 4 | UINT16    |     2 |
| AI Channel 5 | UINT16    |     2 |
| AI Channel 6 | UINT16    |     2 |
| AI Channel 7 | UINT16    |     2 |
| Total        |           |    16 |

這代表控制器可以一次回報 8 個類比輸入通道的數值。

例如：

```text
0A00 1400 1E00 2800
3200 3C00 4600 5000
```

這裡只是示範資料排列，還不能直接判定實際電壓或感測器數值。

為什麼？

因為目前文件沒有提供：

* ADC 解析度。
* 感測器量測範圍。
* 電壓對應關係。
* 是否經過比例換算。
* 正負值或偏移量處理方式。

例如，假設 AI 數值為 1000，它可能代表：

* ADC 原始計數值 1000。
* 1.000V。
* 經過比例換算後的感測器數值。

這些必須由 AI 的硬體規格或控制器程式確認。

## 3. Main Stages Positions：48 Bytes

這部分與六軸運動控制最有關係。

資料型態：

`DOUBLE`

DOUBLE 通常代表 IEEE 754 的 64-bit 雙精度浮點數。

每個 DOUBLE：

$$
64 \div 8 = 8\ Bytes
$$

文件指出：

一個 Stage 有六個軸：

```text
X
Y
Z
RX
RY
RZ
```

因此：

$$
6 \times 8 = 48\ Bytes
$$

資料排列：

| Axis  | Data Type | Bytes |
| ----- | --------- | ----: |
| X     | DOUBLE    |     8 |
| Y     | DOUBLE    |     8 |
| Z     | DOUBLE    |     8 |
| RX    | DOUBLE    |     8 |
| RY    | DOUBLE    |     8 |
| RZ    | DOUBLE    |     8 |
| Total |           |    48 |

這裡需要理解六軸的意義。

前三個：

* X：X 軸直線位置。
* Y：Y 軸直線位置。
* Z：Z 軸直線位置。

後三個：

* RX：繞 X 軸旋轉。
* RY：繞 Y 軸旋轉。
* RZ：繞 Z 軸旋轉。

因此，這不是只有 XYZ 三軸的直線運動平台，而是包含三個旋轉自由度的六自由度平台。

例如：

```text
X  = 100.25
Y  = 200.50
Z  = 300.75

RX = 0.10
RY = 0.20
RZ = 0.30
```

以上只是示意數值。

實際單位可能是：

* XYZ：mm。
* RX、RY、RZ：degree 或 radian。

目前這份文件沒有說明單位，必須另外確認。

### 網路補充：IEEE 754 DOUBLE

這裡補充一個非常重要的技術知識。

DOUBLE 並不是把數字直接轉成 HEX 字串。

例如：

```text
100.25
```

它必須先按照 IEEE 754 Binary64 格式轉換成 8 Bytes 的二進位資料，再把這 8 Bytes 轉成 16 個 HEX Characters。

IEEE 754 Binary64 的結構如下：

| 欄位             | Bit 數量 |
| -------------- | -----: |
| Sign（正負號）      |      1 |
| Exponent（指數）   |     11 |
| Fraction（有效數字） |     52 |
| Total          |     64 |

這是電腦表示浮點數的標準格式。

參考資料：

* [IEEE 754 浮點數格式相關標準說明](https://standards.ieee.org/ieee/754/6210/?utm_source=chatgpt.com)
* [Binary64 格式與 Little Endian 說明](https://www.iteh.us/catalog/standards/iso/9bcb6a06-74ff-46ef-bf9e-40a27697a26a/iso-25178-71-2026?utm_source=chatgpt.com)

**特別注意：目前文件沒有說明 DOUBLE 的 Byte Order（位元組順序）。**

例如，同一個 DOUBLE，在 Little Endian 與 Big Endian 的傳輸表示可能不同。

所以，接收端不能只看到 16 個 HEX Characters，就直接假設其排列方式。

必須向控制器程式設計者確認。

---

# 第四部分：完整 68 Bytes 封包配置

現在把剛才的三個區塊合在一起。

| Offset    | Field             | Data Type |  Bytes | HEX Characters |
| --------- | ----------------- | --------- | -----: | -------------: |
| 0～3       | Controller Status | UINT32    |      4 |              8 |
| 4～5       | AI Channel 0      | UINT16    |      2 |              4 |
| 6～7       | AI Channel 1      | UINT16    |      2 |              4 |
| 8～9       | AI Channel 2      | UINT16    |      2 |              4 |
| 10～11     | AI Channel 3      | UINT16    |      2 |              4 |
| 12～13     | AI Channel 4      | UINT16    |      2 |              4 |
| 14～15     | AI Channel 5      | UINT16    |      2 |              4 |
| 16～17     | AI Channel 6      | UINT16    |      2 |              4 |
| 18～19     | AI Channel 7      | UINT16    |      2 |              4 |
| 20～27     | X                 | DOUBLE    |      8 |             16 |
| 28～35     | Y                 | DOUBLE    |      8 |             16 |
| 36～43     | Z                 | DOUBLE    |      8 |             16 |
| 44～51     | RX                | DOUBLE    |      8 |             16 |
| 52～59     | RY                | DOUBLE    |      8 |             16 |
| 60～67     | RZ                | DOUBLE    |      8 |             16 |
| **Total** |                   |           | **68** |        **136** |

這張表非常值得保留。

因為未來無論使用：

* STM32。
* Raspberry Pi CM5。
* Qt/C++。
* Python。
* 其他監控軟體。

只要遵守這份資料排列規則，就可以解析相同的封包。

---

# 第五部分：Controller Status Encoding（控制器狀態編碼）

這是文件的 2.3 節。

## 1. 原文翻譯

控制器狀態使用 32-bit 整數表示，每個 Bit 代表一種特定狀態。

Table 2：

| Bit | Name      | Description                |
| --: | --------- | -------------------------- |
|   0 | CONNECT   | 連線狀態，1 表示已連線               |
|   1 | VOLTAGEON | Servo 電源狀態，1 表示已供電         |
|   3 | ISMOVING  | 運動狀態，1 表示軸正在移動             |
|   4 | ISFA      | Fast Alignment 狀態，1 表示正在執行 |
|   5 | HOMINGEND | Homing 完成狀態，1 表示已完成        |
|   6 | ERROR     | 錯誤狀態，1 表示發生錯誤              |

注意：

Bit 2 沒有定義。

Bit 7～31 目前保留。

## 2. 什麼是 Bit Flag？

這裡要先理解 Bit Flag（位元旗標）。

假設控制器狀態為：

```text
00000003
```

轉換成二進位：

```text
00000000 00000000 00000000 00000011
```

最右邊是 Bit 0。

由右向左：

| Bit | Value | 意義        |
| --: | ----: | --------- |
|   0 |     1 | CONNECT   |
|   1 |     1 | VOLTAGEON |
|   2 |     0 | Reserved  |
|   3 |     0 | ISMOVING  |
|   4 |     0 | ISFA      |
|   5 |     0 | HOMINGEND |
|   6 |     0 | ERROR     |

因此：

`0x00000003`

代表：

* 控制器已連線。
* Servo 電源已開啟。
* 沒有移動。
* 沒有執行 Fast Alignment。
* Homing 尚未完成。
* 沒有錯誤。

這就是 Bit Flag 的用途。

一個 UINT32 可以同時表示多個不同的狀態，不需要為每一個狀態另外配置一個 Byte。

## 3. 如何使用 C++ 判斷狀態？

這裡補充實際程式設計的方式。

假設：

```cpp
uint32_t status = 0x00000003;
```

可以使用 Bitwise AND（位元 AND 運算）檢查某個狀態。

例如：

```cpp
bool connected = (status & (1U << 0)) != 0;
bool voltageOn = (status & (1U << 1)) != 0;
bool isMoving  = (status & (1U << 3)) != 0;
bool isFA      = (status & (1U << 4)) != 0;
bool homingEnd = (status & (1U << 5)) != 0;
bool error     = (status & (1U << 6)) != 0;
```

為什麼使用：

```cpp
1U << 3
```

因為它代表將數字 1 左移 3 個 Bit：

```text
00000001
```

變成：

```text
00001000
```

也就是：

```text
0x08
```

再與 status 執行 AND 運算，就可以知道 Bit 3 是否為 1。

這種方式稱為 Bit Mask（位元遮罩）。

在 STM32、EtherCAT、嵌入式 C/C++ 開發中非常常見。

---

# 第六部分：Table 3 的 Bit Mapping 如何理解？

原文 Table 3：

```text
Bit No. 31 30 29 28 ... 16
Function Reserved

Bit No. 15 14 13 12 11 10 9 8 7 6 5 4 3 2 1 0
Function Reserved e he fa m vo c
```

這個表格的排版可能在文件轉換過程中失去了部分對齊。

不過，根據 Table 2，可以整理成：

|  Bit | Symbol   | 完整名稱      |
| ---: | -------- | --------- |
|    0 | c        | CONNECT   |
|    1 | vo       | VOLTAGEON |
|    2 | Reserved | 未定義       |
|    3 | m        | ISMOVING  |
|    4 | fa       | ISFA      |
|    5 | he       | HOMINGEND |
|    6 | e        | ERROR     |
| 7～31 | Reserved | 保留        |

其中：

* c = Connect
* vo = Voltage On
* m = Moving
* fa = Fast Alignment
* he = Homing End
* e = Error

這樣就能看懂 Table 3 的縮寫。

---

# 第七部分：補充研究——EtherCAT 的 Status Word 與這份文件有什麼關係？

這部分我特別搜尋了 EtherCAT 相關技術資料。

因為文件標題是：

SixAxes Controller EtherCAT Command Manual

而且使用：

* UINT32
* Status Flags
* Servo Voltage
* Motion Status
* Homing
* Error

這些都是運動控制系統經常使用的術語。

但這裡有一個重要觀念：

**這份文件中的 Controller Status，不應直接當成標準 EtherCAT CiA 402 Statusword。**

為什麼？

因為標準 CiA 402 Drive Profile 使用的 Statusword，通常是 16-bit，也就是 UINT16。

其中部分 Bit 的定義如下：

| Bit | CiA 402 標準定義       |
| --: | ------------------ |
|   0 | Ready to Switch On |
|   1 | Switched On        |
|   2 | Operation Enabled  |
|   3 | Fault              |
|   4 | Voltage Enabled    |
|   5 | Quick Stop         |
|   6 | Switch On Disabled |
|   7 | Warning            |
|  10 | Target Reached     |

參考資料：

* [CiA 402 Drive Statusword 說明（ROS 2 EtherCAT Driver）](https://icube-robotics.github.io/ethercat_driver_ros2/developer_guide/cia402_drive.html?utm_source=chatgpt.com)
* [Kollmorgen EtherCAT Status Word 說明](https://webhelp.kollmorgen.com/mkd/english/Content/Ethercat/EtherCat_05_04_03_Status%20Machine%20Bits.htm?TocPath=MKD+EtherCAT+Manual%7CEtherCAT+Profile%7CCANopen+over+EtherCAT+%28CoE%29+status+machine%7C_____3&utm_source=chatgpt.com)

比較一下：

| Bit | SixAxes 文件 | CiA 402 標準         |
| --: | ---------- | ------------------ |
|   0 | CONNECT    | Ready to Switch On |
|   1 | VOLTAGEON  | Switched On        |
|   2 | Reserved   | Operation Enabled  |
|   3 | ISMOVING   | Fault              |
|   4 | ISFA       | Voltage Enabled    |
|   5 | HOMINGEND  | Quick Stop         |
|   6 | ERROR      | Switch On Disabled |

兩者定義明顯不同。

因此，我的判斷是：

這份文件描述的是 SixAxes 控制器自行定義的 32-bit 系統狀態字，而不是直接將 EtherCAT 標準 Statusword 傳出來。

這並不代表它與 EtherCAT 無關。

控制器完全可以讀取 EtherCAT 各個 Slave 的狀態，再整合成自己定義的 32-bit Controller Status。

這種設計在工業控制系統中很常見。

---

# 第八部分：這份文件目前還缺少哪些重要資訊？

雖然目前已經知道 68 Bytes 的結構，但如果要獨立開發接收端，還有幾個關鍵問題需要釐清。

我幫你整理如下：

| 項目                  | 目前文件是否說明 | 需要確認的內容                    |
| ------------------- | -------- | -------------------------- |
| Packet Length       | 已說明      | 68 Bytes                   |
| HEX Encoding        | 已說明      | 136 個 HEX Characters       |
| Start Character     | 已說明      | `>`                        |
| End Character       | 已說明      | `\r\n`                     |
| Update Interval     | 已說明      | 200ms                      |
| TCP Port            | 已說明      | 8888                       |
| Controller Status   | 已說明      | UINT32                     |
| AI Channels         | 部分說明     | 8 個 UINT16                 |
| AI Unit             | 未說明      | ADC 原始值或實際物理量              |
| Axis Position       | 部分說明     | 6 個 DOUBLE                 |
| Axis Unit           | 未說明      | mm、degree 或 radian         |
| Byte Order          | 未說明      | Little Endian 或 Big Endian |
| DOUBLE Encoding     | 未完整說明    | IEEE 754 Binary64          |
| Axis Order          | 部分說明     | XYZ + RXRYRZ               |
| TCP Connection Role | 未說明      | Server 或 Client            |
| Error Definition    | 部分說明     | ERROR Bit 6 的具體錯誤條件        |

其中，我認為最需要優先確認的是：

**1. UINT32、UINT16、DOUBLE 的 Byte Order 是否全部採用 Little Endian？**

**2. 六軸 DOUBLE 的單位分別是什麼？**

**3. AI 八個 Channel 分別對應哪些實際感測器？**

**4. ERROR Bit 6 是不是只代表總體錯誤？有沒有其他錯誤碼可以查詢？**

**5. localhost:8888 是由控制器建立 TCP Server，還是由監控軟體建立 TCP Server？**

這些資訊會直接影響接收端的實作。

---

# 第九部分：與 STM32H755、Raspberry Pi CM5、Qt 專案的關聯

最後，我把這份文件與你目前正在進行的系統整合連結起來。

你目前的系統架構包含：

* STM32H755：負責控制器端資料處理。
* EtherCAT：取得六軸運動資料。
* SPI2：傳輸軌跡資料。
* Raspberry Pi CM5：接收 STM32 傳來的資料。
* Qt Client：顯示監控資料。

這份 Routine Status Packet 描述的是另一種資料：

**控制器即時狀態監控資料。**

它與你先前處理的軌跡資料並不相同。

可以這樣理解：

| 項目   | Routine Status Packet      | Trajectory Packet |
| ---- | -------------------------- | ----------------- |
| 主要用途 | 即時狀態監控                     | 軌跡資料傳輸            |
| 更新方式 | 每 200ms                    | 依照軌跡取樣週期          |
| 資料內容 | 狀態、AI、六軸位置                 | 軌跡位置及相關資料         |
| 資料長度 | 68 Bytes                   | 依目前設計為 72 Bytes   |
| 編碼方式 | HEX ASCII String           | Binary Data       |
| 傳輸通道 | TCP                        | SPI2              |
| 接收端  | Host / Monitoring Software | Raspberry Pi CM5  |
| 顯示用途 | Qt 即時監控                    | 軌跡記錄與分析           |

這兩種資料的目的不同。

例如：

假設六軸平台正在執行一段軌跡。

Routine Status Packet 可以每 200ms 回報一次目前的位置及控制器狀態。

Trajectory Packet 則可以持續傳輸更高頻率的軌跡點資料。

因此：

* Routine Status Packet 比較適合即時監控。
* Trajectory Packet 比較適合軌跡記錄與後續分析。

兩者可以同時存在於同一套控制系統中。

---

# 本篇最重要的觀念整理

我希望你先掌握以下 8 個重點：

1. Routine Status Packet 是控制器週期性回報的即時狀態封包。
2. 傳送週期是 200ms，也就是 5Hz。
3. 資料本體固定為 68 Bytes。
4. 因為使用 HEX ASCII 編碼，所以資料本體會變成 136 個字元。
5. 封包開頭是 `>`，結尾是 `\r\n`，所以實際傳輸長度為 139 Bytes。
6. 68 Bytes 由 Controller Status（4 Bytes）、AI（16 Bytes）、六軸位置（48 Bytes）組成。
7. 六軸位置使用 DOUBLE，也就是每軸 8 Bytes 的 IEEE 754 雙精度浮點數。
8. Controller Status 是 SixAxes 自行定義的 32-bit 狀態旗標，不應直接套用 CiA 402 標準 Bit 定義。

**我的建議：**

接下來閱讀文件時，我們應該特別注意每個資料欄位的實際意義、資料型態、Byte Order，以及控制器內部如何取得這些數值。

尤其是六軸位置資料，這會直接關係到你目前 STM32H755 EtherCAT 軌跡資料的整合工作。

你可以繼續貼下一段文件，我會沿用這個方式，除了翻譯，也會補充相關的 EtherCAT、STM32、C/C++、TCP/IP 技術知識，並且將各章節之間的關係串聯起來。


---

這一段比上一段更深入，開始描述控制器內部資料的來源，以及如何將原始數值轉換成實際工程單位。

我認為有四個部分特別值得注意：

1. **Analog Input（AI）：** 八個類比輸入通道分別連接什麼設備，以及如何將 ADC 原始數值轉換成電壓。
2. **HEX to DOUBLE：** 如何將 HEX 字串還原成 C++ 的 double 浮點數。
3. **Axis Position Conversion：** EtherCAT 驅動器取得的 Encoder Position，如何轉換成實際的六軸位置。
4. **Digital Input：** 步進馬達及驅動器的數位輸入訊號，如何使用 Bit 來表示。

另外，這一段有兩個非常重要的地方與上一段文件存在技術上的矛盾。我會特別標示，因為它們可能直接影響你未來的 STM32、CM5 和 Qt 程式設計。

---

# 第一部分：2.4 Analog Input Channel Mapping

## 1. 原文翻譯

系統包含兩個 AI 裝置，每個裝置具有四個類比輸入通道，總共八個通道。

功能對應如下：

### Table 4：AI 裝置與通道功能對照表

| Device   | Channel | Function  | 功能        |
| -------- | ------- | --------- | --------- |
| Device 0 | AI00    | Load cell | 荷重元       |
| Device 0 | AI01    | Reserved  | 保留        |
| Device 0 | AI02    | I2C0      | I2C0      |
| Device 0 | AI03    | I2C3      | I2C3      |
| Device 0 | AI04    | PM Ch0    | PM 第 0 通道 |
| Device 0 | AI05    | PM Ch1    | PM 第 1 通道 |
| Device 0 | AI06    | PM Ch2    | PM 第 2 通道 |
| Device 0 | AI07    | PM Ch3    | PM 第 3 通道 |

這裡先提醒一個文件問題：

原文說明有兩個 AI Devices，每個裝置四個通道。

但表格卻將 AI00～AI07 全部列在 Device 0 底下。

合理的排列可能是：

| Device   | Channel | Function  |
| -------- | ------- | --------- |
| Device 0 | AI00    | Load cell |
| Device 0 | AI01    | Reserved  |
| Device 0 | AI02    | I2C0      |
| Device 0 | AI03    | I2C3      |
| Device 1 | AI04    | PM Ch0    |
| Device 1 | AI05    | PM Ch1    |
| Device 1 | AI06    | PM Ch2    |
| Device 1 | AI07    | PM Ch3    |

但是，這只是根據每個 Device 四個通道推測出來的排列，不能直接當成已確認的硬體配置。

需要向同事確認 Device 1 是否才是 AI04～AI07 的正確歸屬。

## 2. 什麼是 AI（Analog Input）？

AI 是 Analog Input 的縮寫，也就是類比輸入。

類比訊號與數位訊號最大的差異：

**類比訊號可以在一定範圍內連續變化，而數位訊號通常只有 0 和 1 兩種邏輯狀態。**

例如：

假設一個感測器輸出電壓：

```text
0.0V
0.1V
0.2V
0.3V
...
9.9V
10.0V
```

這就是類比訊號。

但是，電腦無法直接使用這種連續電壓進行數值運算，因此需要 ADC。

ADC = Analog-to-Digital Converter。

中文稱為類比數位轉換器。

其工作流程：

```text
實際物理量
    |
    v
感測器
    |
    v
類比電壓
    |
    v
ADC
    |
    v
數位原始數值
    |
    v
控制器程式
    |
    v
轉換成實際工程單位
```

例如：

荷重元受到外力，產生電壓變化。

ADC 將電壓轉換成 16-bit 整數。

最後由控制器程式換算成電壓，甚至進一步換算成重量或牛頓。

這就是 AI Channel 的用途。

---

# 第二部分：2.4.1 AI Channel Voltage Scaling

這部分是整份文件中非常重要的數學轉換。

因為八個 AI 通道雖然全部使用 UINT16，但不同感測器的電壓範圍不同，因此不能全部使用同一個轉換公式。

文件將它分成兩類。

## 1. Load Cell Channels（荷重元通道）

原文：

Voltage Range：

-10V ～ +10V

轉換公式：

$$
V=\left(\frac{RawData}{65535}\times20\right)-10
$$

其中：

* V：實際電壓。
* Raw Data：ADC 原始數值。
* 65535：16-bit 無號整數最大值。
* 20V：整個電壓範圍的寬度。
* -10V：電壓起始值。

### 為什麼是 65535？

因為：

$$
2^{16}=65536
$$

16-bit 可以表示 65536 種不同的數值。

範圍是：

$$
0\sim65535
$$

因此文件使用 65535 作為換算公式的分母。

### 實際計算

假設 ADC 回傳：

```cpp
uint16_t rawData = 0;
```

代入：

$$
V=\left(\frac{0}{65535}\times20\right)-10
$$

結果：

$$
V=-10V
$$

如果：

```cpp
uint16_t rawData = 32768;
```

則：

$$
V=\left(\frac{32768}{65535}\times20\right)-10
$$

結果約為：

$$
V=0.000153V
$$

也就是接近 0V。

如果：

```cpp
uint16_t rawData = 65535;
```

結果：

$$
V=+10V
$$

整理如下：

| Raw Data |  換算電壓 |
| -------: | ----: |
|        0 |  -10V |
|    16384 | 約 -5V |
|    32768 |  約 0V |
|    49151 | 約 +5V |
|    65535 |  +10V |

這裡有一個重要的硬體知識。

### 為什麼負電壓可以使用 UINT16 表示？

你可能會疑惑：

UINT16 不是只能表示 0～65535 嗎？

那麼 -10V 到 +10V 的負電壓如何表示？

答案是使用一種稱為：

**Offset Binary（偏移二進位）**

的編碼方式。

概念如下：

| 電壓   |  ADC 數值 |
| ---- | ------: |
| -10V |       0 |
| -5V  | 約 16384 |
| 0V   | 約 32768 |
| +5V  | 約 49151 |
| +10V |   65535 |

也就是利用數值中間點代表 0V。

這與一般使用二補數（Two's Complement）表示負整數的方式不同。

補充參考：

* [Analog Devices：Bipolar 與 Unipolar ADC 編碼說明](https://ez.analog.com/data_converters/precision_adcs/a/documents/c/ad717x-faq/DO14211/faq-what-is-the-data-output-coding-format-used-in-ad717x-family?utm_source=chatgpt.com)
* [Texas Instruments：16-bit ADC Transfer Function](https://www.ti.com/tw/lit/gpn/ads8688w?utm_source=chatgpt.com)

這裡特別提醒：

文件的公式隱含了 Offset Binary 的假設。如果實際 ADC 使用 Two's Complement，便不能直接套用這個公式。

因此要確認控制器回傳的 Raw Data 是否已經轉換成 Offset Binary。

---

## 2. PM Channels（PM Ch0～Ch3）

原文：

Voltage Range：

0V ～ 10V

轉換公式：

$$
V=\frac{RawData}{65535}\times10
$$

這與 Load Cell 最大的不同，就是起始電壓為 0V，而不是 -10V。

### 實際計算

| Raw Data |   換算電壓 |
| -------: | -----: |
|        0 |     0V |
|    16384 | 約 2.5V |
|    32768 |   約 5V |
|    49151 | 約 7.5V |
|    65535 |    10V |

例如：

```cpp
uint16_t rawData = 32768;
```

則：

$$
V=\frac{32768}{65535}\times10
$$

結果：

$$
V\approx5.000076V
$$

### 3. 為什麼要分成兩種公式？

因為不同感測器的電壓範圍不同。

假設：

AI00 是 Load Cell，AI04 是 PM Ch0。

即使兩個 ADC 都回傳：

```text
32768
```

它們的實際意義都是接近 5V 嗎？

不是。

| Channel        | Raw Data | 實際電壓 |
| -------------- | -------: | ---: |
| AI00 Load Cell |    32768 | 約 0V |
| AI04 PM Ch0    |    32768 | 約 5V |

這個差異非常重要。

如果接收端沒有根據 Channel Function 選擇正確的公式，就會產生錯誤的感測器數值。

### 4. C++ 實際轉換方式

假設未來要在 CM5 或 Qt 程式中處理這八個 AI 數值，可以使用：

```cpp
double ConvertLoadCellVoltage(uint16_t raw)
{
    return (static_cast<double>(raw) / 65535.0 * 20.0) - 10.0;
}

double ConvertPMVoltage(uint16_t raw)
{
    return (static_cast<double>(raw) / 65535.0) * 10.0;
}
```

這裡使用 `static_cast<double>`，是為了避免整數除法造成精度損失。

例如：

```cpp
uint16_t raw = 32768;

double loadCellVoltage = ConvertLoadCellVoltage(raw);
double pmVoltage = ConvertPMVoltage(raw);
```

這兩個變數會得到不同結果。

**另外要注意：電壓換算不等於重量換算。**

Load Cell 雖然稱為荷重元，但這份文件目前只提供電壓換算公式。

如果要得到實際重量，例如 kg 或 N，還需要知道：

* Load Cell 的額定容量。
* 感測器靈敏度。
* 放大器增益。
* 零點偏移。
* 校正係數。

這些目前都沒有提供。

---

# 第三部分：2.5 Principle of HEX to double Conversion

這部分非常重要，尤其與你目前 STM32H755 和 Raspberry Pi CM5 的 C++ 程式設計直接相關。

## 1. 原文翻譯

Routine Packet 中的每個 DOUBLE 數值，使用 IEEE 754 標準的 64-bit 浮點數格式編碼。

每個數值由 8 Bytes 組成，也就是 16 個 HEX Characters。

IEEE 754 Double Precision 格式由三個部分組成：

| 欄位       | Bit 數 | 說明        |
| -------- | ----: | --------- |
| Sign     |     1 | 正負號       |
| Exponent |    11 | 指數        |
| Mantissa |    52 | 尾數／有效數字部分 |
| Total    |    64 | 64-bit    |

### Sign

表示正負號：

* 0：正數。
* 1：負數。

### Exponent

指數欄位使用 Bias = 1023 的偏移編碼。

### Mantissa

表示浮點數的小數有效部分。

---

## 2. IEEE 754 到底是什麼？

IEEE 754 是一套國際標準，用來規範電腦如何表示浮點數。

例如：

```cpp
double position = 123.456;
```

在 C++ 中，這個變數通常使用 64-bit 表示。

但是電腦實際儲存的不是：

```text
123.456
```

而是 64 個 Bit 組成的二進位資料。

這 64 個 Bit 再轉換成 HEX，就會得到 16 個 HEX Characters。

例如：

```text
405EDD2F1A9FBE77
```

這是一個示意用的 IEEE 754 Binary64 HEX 表示。

因此：

```text
123.456
    |
    v
IEEE 754 Binary64
    |
    v
8 Bytes
    |
    v
16 HEX Characters
```

接收端必須將這 16 個 HEX Characters 還原成原本的 double。

### 網路補充資料

可以使用 IEEE 754 線上轉換工具，親自輸入十進位數字，觀察其 64-bit 二進位和 HEX 表示：

[IEEE 754 Floating Point Converter](https://www.ieee-754.net/guide?utm_source=chatgpt.com)

這個工具也能選擇 Big Endian 或 Little Endian，對理解這份文件很有幫助。

---

## 3. IEEE 754 的數學公式

文件提供：

$$
Value=(-1)^{sign}\times1.mantissa\times2^{exponent-1023}
$$

我們拆開理解。

### 第一部分：正負號

$$
(-1)^{sign}
$$

如果 Sign = 0：

$$
(-1)^0=+1
$$

如果 Sign = 1：

$$
(-1)^1=-1
$$

### 第二部分：Mantissa

假設 Mantissa 表示：

```text
1.101
```

二進位小數代表：

$$
1+\frac{1}{2}+\frac{0}{4}+\frac{1}{8}
$$

因此：

$$
1.101_2=1.625
$$

### 第三部分：Exponent

Exponent 儲存的不是直接使用的指數。

必須減掉 Bias：

$$
ActualExponent=Exponent-1023
$$

假設：

```text
Exponent = 1023
```

那麼：

$$
1023-1023=0
$$

因此：

$$
2^0=1
$$

三個部分組合起來，就能還原原始浮點數。

**補充：** 文件提供的公式是針對一般 Normalized Number（正規化浮點數）。IEEE 754 還有 Zero、Subnormal、Infinity 和 NaN 等特殊情況，實務上建議交給 C++ 浮點數轉換機制處理，不要自己只用這個公式解析所有情況。

---

## 4. Endianness：這是最需要注意的地方

文件特別提醒：

Most platforms use little-endian representation in memory.

However, transmitted HEX strings are typically in big-endian (most significant byte first).

Always verify the endianness.

中文：

大多數平台在記憶體中使用 Little Endian。

但是傳輸的 HEX 字串通常使用 Big Endian，也就是最高有效位元組先傳送。

解讀二進位資料時，一定要確認 Endianness。

### Big Endian 與 Little Endian

假設一個 32-bit 數值：

```text
0x12345678
```

Big Endian 的 Byte 排列：

```text
12 34 56 78
```

Little Endian 的 Byte 排列：

```text
78 56 34 12
```

兩者代表相同的數值，只是記憶體或傳輸資料的排列順序不同。

這就是 Endianness。

### 與你的 STM32H755 和 CM5 有什麼關係？

這兩個平台通常都是 Little Endian。

但是，不能因此認為 TCP 傳來的 HEX String 也一定是 Little Endian。

例如，控制器傳送：

```text
405EDD2F1A9FBE77
```

如果這 16 個 HEX Characters 是 Big Endian 的 IEEE 754 表示，那麼接收端應該先將它解讀成：

```text
40 5E DD 2F 1A 9F BE 77
```

再轉成 double。

如果你的 C++ 主機使用 Little Endian，直接把這八個 Bytes 複製進 double 的記憶體，結果就可能錯誤。

這是兩個不同層次的事情：

* HEX 字串的排列順序。
* CPU 記憶體中的 Byte 排列順序。

不能混為一談。

---

# 第四部分：2.6 Axis Position Conversion

這一節有一個非常重要的技術矛盾，我先說明原文，再指出問題。

## 1. 原文翻譯

Axis Position（軸位置）來自三個 DRIVE GROUP，每個 DRIVE GROUP 包含四個軸，因此總共有 12 個軸。

每個軸的位置是 32-bit Encoder Position。

控制器會先將 Encoder Position 轉換成實際物理單位（mm 或 degrees），再編碼成 HEX。

轉換方式：

* X/Y/Z 軸：使用 Linear Encoder Ratios（線性編碼器比例）。
* RX/RY/RZ 軸：使用 Angular Ratios（角度比例）。

每個軸佔用 4 Bytes。

12 個軸總共：

$$
12\times4=48\ Bytes
$$

## 2. 什麼是 Encoder Position？

Encoder 中文稱為編碼器。

它的作用是回報馬達或機構目前的位置變化。

例如：

```text
馬達旋轉
    |
    v
Encoder 偵測旋轉量
    |
    v
產生 Pulse / Count
    |
    v
控制器計算位置
    |
    v
轉換成 mm 或 degree
```

假設一個線性軸：

Encoder 每累積 1000 Counts，平台移動 1mm。

那麼：

| Encoder Count | 實際位置 |
| ------------: | ---: |
|          1000 |  1mm |
|          2000 |  2mm |
|          3000 |  3mm |
|          4000 |  4mm |

這就是 Linear Encoder Ratio。

例如：

$$
Position(mm)=\frac{EncoderCount}{1000}
$$

如果是旋轉軸，可能使用：

$$
Position(degree)=\frac{EncoderCount}{CountsPerRevolution}\times360
$$

但實際上還可能包含：

* 減速比。
* 螺桿導程。
* 電子齒輪比。
* 編碼器解析度。
* 方向反轉設定。

所以真正的轉換公式必須依據各個軸的硬體設定。

---

## 3. 非常重要：這裡與上一節的 48 Bytes 發生矛盾

請比較上一段 2.2 和這一段 2.6。

### 上一段 2.2

文件明確說明：

六軸：

```text
X Y Z RX RY RZ
```

每個軸使用 DOUBLE：

$$
6\times8=48\ Bytes
$$

### 這一段 2.6

文件明確說明：

三個 DRIVE GROUP，每組四軸：

$$
3\times4=12\ Axes
$$

每個軸使用 32-bit：

$$
12\times4=48\ Bytes
$$

兩者雖然總長度都是 48 Bytes，但是資料意義完全不同。

| 比較項目   | 2.2      | 2.6                     |
| ------ | -------- | ----------------------- |
| 軸數量    | 6 軸      | 12 軸                    |
| 每軸資料長度 | 8 Bytes  | 4 Bytes                 |
| 資料型態   | DOUBLE   | 32-bit Encoder Position |
| 總長度    | 48 Bytes | 48 Bytes                |
| 資料意義   | 六軸物理位置   | 十二軸位置資料                 |

這不是單純的文字差異。

它會直接影響接收端解析方式。

### 我的技術判斷

可能存在兩種不同的資料概念：

**第一種：六軸平台的物理座標。**

```text
X
Y
Z
RX
RY
RZ
```

每個位置是 DOUBLE。

**第二種：三組 DRIVE GROUP 的原始軸位置。**

```text
Drive Group 0：Axis 0～3
Drive Group 1：Axis 0～3
Drive Group 2：Axis 0～3
```

每個位置是 32-bit 整數。

這兩種資料可能分別存在於控制器的不同內部資料結構。

但目前文件卻沒有清楚說明它們之間的關係。

因此，不能直接把 2.6 的 12 軸資料取代 2.2 的六軸 DOUBLE 解析方式。

**這個問題必須向同事確認。**

---

# 第五部分：2.7 Digital Input Bit of Stepper Motor

這一節介紹步進馬達的數位輸入狀態。

## 1. 原文翻譯

用來表示外部輸入訊號的邏輯狀態。

Table 5：

|   Bit | Symbol | Description                 | 中文        |
| ----: | ------ | --------------------------- | --------- |
|     0 | MEL    | Negative limit input signal | 負向極限輸入    |
|     1 | PEL    | Positive limit input signal | 正向極限輸入    |
|     2 | ORG    | Home input signal           | 原點輸入      |
|  3～15 | -      | Reserved                    | 保留        |
|    16 | EMG    | Emergency Input             | 緊急停止輸入    |
|    17 | ALM    | Alarm signal from driver    | 驅動器警報     |
|    18 | -      | Reserved                    | 保留        |
|    19 | IDX    | EZ signal input             | 編碼器 Z 相訊號 |
|    20 | INP    | In-position input           | 定位完成輸入    |
|    21 | DI     | Digital Input               | 一般數位輸入    |
| 22～31 | -      | Reserved                    | 保留        |

這裡使用的是 32-bit 的 Digital Input Status Word。

每個 Bit 表示一種輸入訊號狀態。

---

## 2. MEL：Negative Limit

MEL = Minus End Limit。

中文可以理解為負向極限訊號。

假設一個平台的 X 軸只能移動：

```text
-100mm ～ +100mm
```

當平台移動到負向極限附近，就可能觸發 MEL。

例如：

```text
X 軸向負方向移動
        |
        v
到達負向極限開關
        |
        v
MEL = 1
        |
        v
控制器偵測到負向極限
```

控制器通常會根據安全設定停止或限制該軸運動。

## 3. PEL：Positive Limit

PEL = Plus End Limit。

與 MEL 相反。

代表正方向的極限輸入。

例如：

```text
X 軸向正方向移動
        |
        v
到達正向極限開關
        |
        v
PEL = 1
```

MEL 和 PEL 通常會形成機械運動範圍的保護機制。

## 4. ORG：Origin

ORG = Origin Signal。

也就是原點訊號。

在運動控制中，常常需要執行 Homing（回原點）。

例如：

```text
機台啟動
    |
    v
執行 Homing
    |
    v
馬達開始尋找原點
    |
    v
觸發 ORG
    |
    v
控制器建立參考位置
```

注意：

ORG 觸發不一定代表整個 Homing 程序已經完成。

實際上還可能需要尋找 Encoder Index（EZ）或執行額外的定位程序。

因此，ORG 與上一段的 HOMINGEND 是不同概念：

* ORG：原點開關的輸入狀態。
* HOMINGEND：控制器回報 Homing 程序是否完成。

## 5. EMG：Emergency Input

EMG = Emergency Input。

中文是緊急停止輸入。

這是一個非常重要的安全訊號。

當緊急停止開關被觸發時，控制器會根據安全迴路設計採取相應動作。

但這裡要特別提醒：

**不能只依靠軟體讀取 EMG Bit 來實現安全停機。**

真正的緊急停止功能，必須配合硬體安全迴路、驅動器安全功能及適用的安全標準。

這個 Bit 比較適合用於狀態監控與診斷。

## 6. ALM：Alarm

ALM = Alarm Signal。

代表驅動器回報的警報訊號。

例如：

* 馬達過電流。
* 驅動器過熱。
* 編碼器異常。
* 電源異常。
* 伺服驅動器內部故障。

實際警報內容要透過驅動器的 Error Code 或 Alarm Code 取得。

ALM 通常只能表示是否發生警報，不能單靠這個 Bit 知道具體故障原因。

## 7. IDX / EZ：Encoder Index

IDX 是文件使用的名稱。

EZ 通常代表 Encoder Z-phase，也就是編碼器 Z 相訊號。

一般增量式編碼器會提供：

* A 相。
* B 相。
* Z 相。

其中 Z 相通常每旋轉一圈產生一次基準脈波。

可以用來協助：

* 精確定位。
* 原點校正。
* 編碼器位置基準確認。

## 8. INP：In-Position

INP = In-Position。

代表驅動器回報馬達已經進入定位完成範圍。

例如：

目標位置：

```text
100.000 mm
```

實際位置：

```text
99.998 mm
```

如果誤差已經進入驅動器設定的定位完成範圍，驅動器可能會輸出 INP。

注意：

INP 不一定代表位置誤差完全等於零，而是表示符合驅動器設定的定位完成條件。

## 9. DI：Digital Input

DI 是一般數位輸入。

例如：

* 外部按鈕。
* 感測器。
* 外部控制訊號。
* 其他機台的觸發訊號。

具體功能取決於實際硬體接線與控制器設定。

### 網路補充：這些縮寫在運動控制系統中很常見

我找到 ADLINK 運動控制器的技術手冊，其中也有相同的 MEL、PEL、ORG、EMG、ALM、EZ、INP 定義。

這表示這些不是 SixAxes 專屬的命名，而是工業運動控制領域常見的訊號名稱。

參考：

[ADLINK PCI-8258 Motion Control I/O 說明](https://www.manualshelf.com/manual/adlink/pci-8258/user-guide-traditional-chinese/page-85.html?utm_source=chatgpt.com)

[ADLINK AMP-204C / AMP-208C Motion Control I/O 說明](https://manualzz.com/doc/63378007/adlink-amp-204c-amp-208c-dsp-based-4-8-axis-advanced-puls...?utm_source=chatgpt.com)

---

# 第六部分：如何使用 C++ 解析 Digital Input？

假設控制器回傳：

```cpp
uint32_t digitalInput = 0x00230007;
```

這個數值包含多個 Bit 狀態。

可以使用：

```cpp
bool MEL = (digitalInput & (1U << 0)) != 0;
bool PEL = (digitalInput & (1U << 1)) != 0;
bool ORG = (digitalInput & (1U << 2)) != 0;

bool EMG = (digitalInput & (1U << 16)) != 0;
bool ALM = (digitalInput & (1U << 17)) != 0;

bool IDX = (digitalInput & (1U << 19)) != 0;
bool INP = (digitalInput & (1U << 20)) != 0;
bool DI  = (digitalInput & (1U << 21)) != 0;
```

例如：

```cpp
if (EMG)
{
    // 緊急停止輸入訊號已觸發
}

if (ALM)
{
    // 驅動器警報訊號已觸發
}

if (INP)
{
    // 驅動器回報定位完成
}
```

這種方式與上一段的 Controller Status 完全相同。

都是使用 Bit Mask 判斷狀態。

但要注意：

目前這段文件沒有說明 Digital Input Status Word 究竟位於 Routine Packet 的哪個位置。

這是後面需要確認的地方。

---

# 第七部分：2.8 Suggested Parsing Strategy

## 1. 原文翻譯

因為所有欄位都具有固定長度及固定排列順序，接收端可以使用以下方式解析 Routine Packet：

1. 擷取前面 8 個 Characters（4 Bytes）作為 Controller Status Word，並解析其中的 Bit。
2. 接著按照各欄位的 Byte 長度，依序擷取後續資料。
3. 根據資料用途，將數值轉換成對應的工程單位，例如 mm、degrees 或原始數值。

## 2. 技術解釋

這裡其實是在說明 Fixed-Length Packet Parsing。

也就是固定長度封包解析。

例如：

假設收到：

```text
>000000030A0014001E002800...
```

接收端不需要尋找每個欄位的分隔符號。

只要知道每個欄位的位置，就能直接擷取。

例如：

| 欄位                | HEX 字元起始位置 | 長度 |
| ----------------- | ---------: | -: |
| Controller Status |          0 |  8 |
| AI00              |          8 |  4 |
| AI01              |         12 |  4 |
| AI02              |         16 |  4 |
| AI03              |         20 |  4 |
| AI04              |         24 |  4 |
| AI05              |         28 |  4 |
| AI06              |         32 |  4 |
| AI07              |         36 |  4 |
| X                 |         40 | 16 |
| Y                 |         56 | 16 |
| Z                 |         72 | 16 |
| RX                |         88 | 16 |
| RY                |        104 | 16 |
| RZ                |        120 | 16 |

總共 136 個 HEX Characters。

這是根據前面 2.2 的 68 Bytes 結構整理出來的。

不過，這個解析表目前有一個前提：

**必須先確認 2.2 的六軸 DOUBLE 格式才是實際 Routine Packet 格式，而不是 2.6 的十二軸 32-bit 格式。**

---

# 第八部分：整份 Routine Status 文件目前發現的問題

我把前後兩段合併檢查，整理出目前需要向同事確認的事項。

這部分非常重要，因為它關係到你未來接收資料時會不會解析錯誤。

| 編號 | 項目                | 文件描述                   | 問題                    |
| -- | ----------------- | ---------------------- | --------------------- |
| 1  | AI Device Mapping | 兩個 Device，各四通道         | 表格卻將八個通道全部列為 Device 0 |
| 2  | Axis Data Format  | 2.2 使用六個 DOUBLE        | 2.6 使用十二個 32-bit 整數   |
| 3  | Axis Position     | XYZ + RXRYRZ           | 沒有提供實際單位與轉換比例         |
| 4  | Endianness        | HEX 通常 Big Endian      | 沒有明確指定實際傳輸順序          |
| 5  | Digital Input     | 定義 32-bit Bit Mapping  | 沒有指出它位於 Packet 的哪個位置  |
| 6  | AI Voltage        | 提供換算公式                 | 沒有說明 ADC 實際輸出編碼方式     |
| 7  | Gripper / Camera  | 第一段提到                  | 後續欄位表格沒有提供對應資料        |
| 8  | I2C0 / I2C3       | 列在 AI Channel Function | 沒有解釋它們實際代表的量測或功能      |

其中第 2 項最嚴重。

因為它涉及封包結構本身，而不是單純的欄位說明不完整。

---

# 第九部分：與你目前 STM32H755 專案的關聯

我特別將這份文件與你目前已經完成的 EtherCAT、SPI2、CM5 軌跡傳輸工作連結起來。

目前你處理的系統可以分成兩種資料。

### A. Routine Status Packet

用途：即時狀態監控。

資料包含：

* Controller Status。
* AI 感測器數值。
* 六軸位置。

特性：

* 每 200ms 傳送。
* TCP/IP。
* HEX ASCII。
* 68 Bytes 原始資料。

### B. Trajectory Packet

用途：軌跡資料記錄。

你目前已經完成的架構：

* STM32H755 CM7 取得 EtherCAT 軌跡資料。
* SPI2 DMA 傳送。
* 72 Bytes Binary Packet。
* Raspberry Pi CM5 接收。
* 儲存軌跡資料。

兩者的差異：

| 項目   | Routine Status | Trajectory |
| ---- | -------------- | ---------- |
| 用途   | 即時狀態監控         | 軌跡記錄       |
| 傳輸頻率 | 5Hz            | 約 1kHz     |
| 資料格式 | HEX ASCII      | Binary     |
| 資料長度 | 68 Bytes       | 72 Bytes   |
| 傳輸介面 | TCP            | SPI2       |
| 接收端  | Host           | CM5        |
| 資料用途 | 顯示狀態           | 儲存軌跡       |

這兩種資料不應混淆。

尤其 Routine Status 裡的六軸物理座標，與 Trajectory Packet 裡的軌跡點資料，雖然可能都包含位置，但它們的來源、更新頻率及用途不同。

---

# 第十部分：我建議你特別記住的技術知識

這一段文件，我希望你掌握以下概念：

| 技術名詞             | 你需要理解的意思         |
| ---------------- | ---------------- |
| Analog Input     | 類比輸入             |
| ADC              | 將電壓轉成數位數值        |
| UINT16           | 16-bit 無號整數      |
| Offset Binary    | 利用中間值表示零電壓的編碼方式  |
| Voltage Scaling  | 原始 ADC 數值轉換成實際電壓 |
| IEEE 754         | 電腦浮點數表示標準        |
| DOUBLE           | 64-bit 雙精度浮點數    |
| Sign             | 浮點數正負號           |
| Exponent         | 浮點數指數            |
| Mantissa         | 浮點數有效數字部分        |
| Endianness       | Byte 排列順序        |
| Encoder Position | 編碼器回報的位置計數       |
| Linear Ratio     | 線性位置換算比例         |
| Angular Ratio    | 角度換算比例           |
| MEL / PEL        | 負向／正向極限          |
| ORG              | 原點訊號             |
| EMG              | 緊急停止輸入           |
| ALM              | 驅動器警報            |
| EZ / IDX         | 編碼器 Z 相訊號        |
| INP              | 定位完成訊號           |
| Bit Mask         | 使用位元遮罩判斷個別狀態     |

## 最後，我認為目前最值得向同事確認的三個問題

讀完這兩段 Routine Status 文件後，我會優先確認以下三件事：

**問題一：**

Routine Status Packet 的 48 Bytes 軸位置資料，究竟是：

* 6 個 DOUBLE？
* 還是 12 個 INT32？

這兩種格式不能同時成立於同一個 48 Bytes 欄位。

**問題二：**

Digital Input 的 32-bit 狀態字，究竟是 Routine Packet 的一部分，還是另一個獨立命令回傳的資料？

因為目前 68 Bytes 的封包配置沒有替它預留空間。

**問題三：**

兩個 AI Devices 的八個通道，是否確實分別為 AI00～AI03 和 AI04～AI07？

這會影響未來 AI 數值的通道對應。

---

**我的建議是：目前先不要急著寫 Routine Status 的接收程式。**

這份文件已經足以理解整體資料概念，但還有幾個協定定義需要釐清。

你可以繼續貼下一個章節。等整份 Manual 閱讀完畢，我們再把所有封包格式、命令、回應、資料型態和傳輸方式整理成一份完整的通訊協定規格，這樣未來整合 STM32H755、CM5 和 Qt 時，就不需要反覆猜測文件中沒有交代的內容。


---

有了這份 Overview，現在可以看出比前兩段更多的系統架構資訊，而且有幾個先前無法判斷的問題，現在已經有了比較合理的解釋。

我重新把三段文件交叉比對後，發現這份 Manual 描述的並不是單純的六軸馬達控制器，而是一套包含：

* 六軸主平台（Main Stage）
* 壓電平台（Piezo Stage）
* EtherCAT Servo Drives
* Analog / Digital I/O
* SMC Gripper
* 感測器介面
* TCP/IP 上位機通訊

的整合式運動控制系統。

我分成八個部分說明，最後會整理目前文件的疑點，以及我對整體系統架構的理解。

# 一、1.1 System Architecture：系統架構

## 1. 原文翻譯

GMT SixAxes Controller 運動控制系統採用分層架構，將上層使用者控制邏輯與底層現場設備操作分離。

上層控制器（Host PC 或 Embedded Controller）透過 TCP/IP 與 GMT SixAxes Controller 通訊，並使用 ASCII 格式傳送運動或 I/O 命令。

GMT SixAxes Controller 在內部扮演兩個角色：

1. Protocol Bridge（通訊協定橋接器）。
2. Real-time Coordinator（即時協調控制器）。

它會將收到的命令轉換成 EtherCAT Fieldbus Transaction，進而控制：

* Servo Drives（伺服驅動器）。
* I/O Modules（輸入輸出模組）。
* Sensor Interfaces（感測器介面）。

系統支援：

**Main Stage（主平台）**

* 每個 Module Side 有六個伺服運動軸。
* 分成 Right Side 和 Left Side。

**Piezo Stage（壓電平台）**

* 每個 Module Side 有三個伺服運動軸。
* 分成 Right Side 和 Left Side。

另外還支援多組 Analog I/O。

系統內部有一個 Central Data Handler（中央資料處理器），負責透過週期性 PDO（Process Data Object）交換，協調所有資料更新。

控制器同時負責：

* 同步控制。
* 狀態監控。
* 即時回饋資料封裝（Routine Packet）。

目的在於確保閉迴路控制（Closed-loop Control）的完整性。

---

## 2. 這段最重要的觀念：分層架構

這裡要理解的是，GMT SixAxes Controller 並不是單純把 TCP 命令直接送給馬達。

它中間還有一層控制邏輯。

整體架構可以理解為：

![Image](https://images.openai.com/static-rsc-4/OId-VXutcC-C-cx1L4rYMtBQAIVB0wQyBcw9N3k-WOEZ_xbxyruruqXjTNNaJLF3oU9k6bZApLMj474kv1atQ77orlQPsspACNqEZ_h-uLX-qw0UOEbKo0XrQcZqxWCr2sExg63DB-HluP_YRrMYKp9LR-Rz8z74bSsARBRoYh7OWCW96WUCGXThdQ2JPUQJ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UXkcedTzL9JW1UiighnyDj5AvlLkmtNAhrm6XGwtWR-pfeCzS_kUl8xGqa4Z1VbuTQfovYHSR86X3hrM8W-0vDapIbRrAfMHAw8kMXXPb1ty3mAWR9V6K28XMjjzdohES-3iAbo4dgEXq5d-wFdFRwmoSaVSYqJabPSytiuRDmqvfBfaDPFe_Rc5clw4Ltnb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/igqwzH9WJMM3huiU8sNfY1FgkIsfEU9L43yWSWrv4ZIqoYjmJeUXAVSMG9ED0iCRbT7T6YElFbCR79OjSl_qjO_jLjFjpm3HKqWXaglBkpLVDFsHEYHnhLUn1fu6NP029VYTKagqS3um8Y3mzV4dRzvIFIhyog1_bpWK2AxUcEpvD8msZ90s2pWfRdbZ10iN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mBONqJn84Axw_4skMWbGvTgdw1uW3Qhsn5eD7EzzhAQ_NXkbZBBb3OxV7kwsr34WdvJ6fUJYWrriepRN3JbPu_yY0ZISd3MUEsvVk5taC0qJ0p6NlFewFziI0bq3orbo9wti926Zf05c0yQ2bEK3W4HT-sen2eMDRDLnUtq81J0Iwk-hjIstAsqzLz9mpvLA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ri5qbpQ2q3zAvKpr-5vIfjQE1unbZsx9BcinKaX7lLhiUVaFJMyyOid6AMQztwcut_WnZNGF0IEPD5OE-sssXiA5gDmVzeYIcrcUdoaTfIVmrCDSsi6e4djHXvEF3wCblPkj-gSb9Vd3vy3hDKazli5HEdmh43RpSbbEBy6xKo6mGGqaS9m8tZR2DGcAqD-o?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1a_Yj6Pk8EJzXevYApNPwfRpMzthem8Q_1Egmowba6RfeiZ7ZFgG65nJjhbhcy0joBUhxBYxZ6IGJcuSjh6vpbhOhvcIW-ilnNiVflx-bwLcfRR9_DbJH45m1lv3PPYNPSmp6zo-Lj-V5Zq_LIc3tYZHNjbUtGqox1RlwSUZTynk7fB3_tFbaehRRKZHuRcj?purpose=fullsize)

```text
          Host PC / Embedded Controller
                    |
                    |
                 TCP/IP
                    |
          +--------------------+
          | GMT SixAxes        |
          | Controller         |
          |                    |
          | Command Parser     |
          |                    |
          | Central Data       |
          | Handler            |
          |                    |
          | Motion Control     |
          +--------------------+
                    |
                    |
                 EtherCAT
                    |
          +---------+----------+
          |         |          |
          v         v          v
       Servo     I/O Module  Sensors
       Drives
          |
          v
       Motors
```

這裡有三個不同的通訊層次。

| 層次                                | 通訊方式     | 主要用途            |
| --------------------------------- | -------- | --------------- |
| 上位機 → GMT Controller              | TCP/IP   | 傳送 ASCII 命令     |
| GMT Controller → EtherCAT Devices | EtherCAT | 控制馬達及讀取設備資料     |
| EtherCAT Devices → Controller     | PDO      | 週期性回傳位置、狀態及 I/O |

這就是所謂的 Layered Architecture（分層架構）。

### 為什麼要這樣設計？

假設上位機想要讓 X 軸移動 100mm。

上位機不需要知道：

* Servo Drive 的 EtherCAT Slave Address。
* PDO Mapping。
* Encoder Resolution。
* 馬達電子齒輪比。
* EtherCAT Sync Manager 設定。
* CiA 402 狀態機。

上位機只需要傳送一個人類容易理解的 ASCII 命令。

例如，假設控制器定義的命令格式是：

```text
MOV X 100
```

這只是示意，並非文件已確認的正式命令。

GMT Controller 收到後，會負責將它轉換成內部運動控制指令，再透過 EtherCAT 控制 Servo Drive。

這樣上位機就不需要直接處理 EtherCAT。

**這也是為什麼你的同事會將 Command Gateway 與底層 SPI 軌跡接收程式分開設計：不同層次負責不同工作。**

---

# 二、這套系統到底有多少個軸？

這是 Overview 帶給我們最重要的新資訊。

原文：

> Axes of main stage servo motion (6 per module side: Right and Left)

意思是：

每一側的 Main Stage 有六個伺服運動軸，分成左右兩側。

因此：

$$
6\times2=12\ Axes
$$

接著：

> Axes of piezo stage servo motion (3 per module side: Right and Left)

每一側的 Piezo Stage 有三個伺服運動軸。

因此：

$$
3\times2=6\ Axes
$$

整理：

| 系統          | Right Side | Left Side |   合計 |
| ----------- | ---------: | --------: | ---: |
| Main Stage  |        6 軸 |       6 軸 | 12 軸 |
| Piezo Stage |        3 軸 |       3 軸 |  6 軸 |
| Total       |        9 軸 |       9 軸 | 18 軸 |

所以，根據 Overview：

**整套系統至少描述了 18 個伺服運動軸。**

這裡有一個容易混淆的地方。

## 6-DOF 不一定等於 6 顆馬達

你之前看到：

```text
X Y Z RX RY RZ
```

這是六自由度（6 Degrees of Freedom）的座標表示。

代表平台可以描述：

* 三個方向的直線運動。
* 三個方向的旋轉運動。

但實際機構可能使用不同數量的馬達共同完成這六個自由度。

例如，某些平台採用 Stewart Platform（史都華平台）機構，會使用六個線性致動器共同控制一個六自由度平台。

![Image](https://images.openai.com/static-rsc-4/6hwkbFBBS7hKwHtRGz9tY4GdTPQIvu3woWJLUu4vTmMaDoJLISRZmJLSCOZKiY8x2KXVPOxCYwmkEoG4dB-iTGirHmumG64t3SVHS-ZQApgvP0vDu9lGO8DHYU4pwBx7TflCZkc1G3gJ3tY7UriyG4fXujtuMieZCknABR_km4oen7xBgYmEgQJVR3MsoXYz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ETD0wAY933Nt7pXTJuIyrkxUgo5fYR2Wlul094V526TgdVmIQNHKIoH85Qyr8S5fTHZNtCMQTpsCBx4hP6x1MEi3FSvaYfv0Wy5pCNCtTc6A_Xtw7jHaIoCulxSVjgImGJqSOxe6sF5ZmodHwnYM8qM5nkkeBzOWMMzD_KiG1YOFF2OwWs0fpKRo-XzIVgf9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8lK3lCOv5KRUFDtR_W2NZldbWURnq1OcFtq8C0qqV9TbDynsLag3fQbnbCcbkkHpwtfrG_P-vEqHF4-iIS_ruCs1KOlyhGUDkVzhQNWH8ccg_R_cmKeGDOiB6C7uC1ghNS66KXb-r9mWu0zuLJnSebTCWRrJ2Zvwi9IeTCSzERyfXTXIKGNqL0KMGmfJTpZw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vTD6Qn7O8uaoVd6BYqZTkoIhflf0HXA1CgL8P7g0wMljaFxBZyjGfPFIxDIGzb8_n3DgYlGWmAbs_2rdQmA5GFg7ajkB_VbcjUNh4q6mm5yTjHZ5rW6XKQPj1I-aprHTeS_f3Oyw-5M4a7z53bvZALvysaJP3OafQMBtBDWOQsiLClbpFiwZL1DJX9YKQO-T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EFCB8fldurVqiOHyi7fKQUO76WMvn2QKSBcskFcKxtvR35z-6cil1s2JYxdB8bxt4dHb121KYofYCyJnuyEoksDVD7jiQCpuYka1TfC1Ivz-9D69wR2Wy3J7rvJAagvMqJEAueK0C1Bu4j44-CIYgGzfH2DEQ4fZbmHZkKGFKXNddEh_aYN_5IYQqvwnfzaw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VcpAFMTck13nMChmQLHaoYc_eaJdXZum4OuydZmKzAH-72x1gKnAB_ChOVRjOtN7cM3b5LCoNtDAS4FXGvl4WA2JA10n6JSHvfYraNQvQ4fbc2BkP8aOC-v1joRP8qhUHNISXiSWzBc6bYEnkdIspkOEaIrFtYEiC1O1gHBwlNPsXQyBVdfs3B8oX-Se69bl?purpose=fullsize)

因此：

| 名詞     | 意義      |
| ------ | ------- |
| 6 Axes | 六個實際驅動軸 |
| 6 DOF  | 六個自由度   |
| XYZ    | 三個直線座標  |
| RXRYRZ | 三個旋轉座標  |

它們相關，但不是完全相同的概念。

這也讓我們有了新的線索，可以重新檢查 2.6 的十二軸資料問題。

---

# 三、重新檢查 2.6：12 個 Encoder Position 的問題

前面我們發現：

2.2 說：

六個軸，每個軸使用 DOUBLE。

$$
6\times8=48\ Bytes
$$

2.6 說：

三個 DRIVE GROUP，每組四個軸。

$$
3\times4=12\ Axes
$$

每個軸使用 32-bit：

$$
12\times4=48\ Bytes
$$

現在加入 Overview：

Main Stage 確實有：

$$
6\times2=12\ Axes
$$

這讓 2.6 的描述變得比較合理。

### 我們可以提出一個新的可能性

假設三個 DRIVE GROUP，每組四個軸：

```text
Drive Group 0：4 Axes
Drive Group 1：4 Axes
Drive Group 2：4 Axes
```

總共：

```text
12 Axes
```

這個數量剛好等於左右兩側 Main Stage 的總驅動軸數。

因此，2.6 的 12 個 Encoder Position，很可能是 Main Stage 左右兩側的實際驅動軸位置。

但是，這仍然只是合理推測。

目前還不知道：

* 每個 Drive Group 對應哪一側。
* 每個 Group 的四個 Axis 分別控制什麼機構。
* Piezo Stage 是否使用其他資料結構。
* 2.2 的六個 DOUBLE 是否代表左右兩側的平台座標，還是單一平台的六自由度座標。

### 一個值得注意的可能性

12 個實際驅動軸的位置，可以透過運動學轉換，得到六個平台座標：

```text
12 個 Encoder Positions
          |
          v
     運動學計算
          |
          v
     6-DOF Platform
          |
          v
    X Y Z RX RY RZ
```

這在多致動器運動平台中是可能的。

但文件並沒有證實 Routine Packet 是否真的同時執行這個轉換。

因此，我們目前不能確定 2.2 和 2.6 是：

1. 同一批資料的不同表示方式。
2. 不同資料來源。
3. 文件版本不一致造成的錯誤。

這是目前整份 Manual 最需要釐清的問題之一。

---

# 四、1.2 Communication Method：通訊方式

這一節提供了非常重要的資訊。

## 1. 原文翻譯

GMT SixAxes Controller 使用 TCP/IP Socket Connection 作為命令通訊介面。

採用 Request-Response Model（請求與回應模式），使用 ASCII 編碼命令。

通訊設定：

| 項目                 | 設定     |
| ------------------ | ------ |
| Protocol           | TCP/IP |
| Command Port       | 9999   |
| Status Stream Port | 8888   |

Command Layer：

* 命令格式：可讀取的 ASCII 字串。
* 結束符號：`\r\n`。
* 結構：使用空格或其他分隔符號區分參數。
* 數值：使用十進位或十六進位。
* Boolean：通常使用 0 或 1。

Timing and Behavior：

**Synchronous Command**

命令採用 Request-Response 模式，執行後立即回覆。

**Asynchronous Routine Status**

Routine Status 每 200ms 非同步傳送一次。

Internal Transport：

* Fieldbus Protocol：EtherCAT。
* Device Profile：支援 CiA 402 伺服驅動器、數位／類比 I/O、SMC Gripper。
* Real-time Performance：透過 PDO 進行具確定性的週期性資料交換及同步。

---

# 五、兩個 TCP Port 的真正用途

這裡可以明確區分兩條通訊路徑。

![Image](https://images.openai.com/static-rsc-4/_uYI94VNHdCXf6wbfigj_LOrZPy6tE63uqRPW7lsUJI_tlUOJtjUzF_Z0OwusMK9GEjVEanPZKAWToKwAD9Hz-k018vjZIrwVgPc4XVdCWUNU6gRGdBVpk9g93wfPmhHc1mbXYRPkEuv1lPtzDscPAKS21CnD895CoF5FpMq0dkIBZoae0SqLwJEBtjyBzao?purpose=fullsize)

## Port 9999：Command Interface

這是命令通訊介面。

用途：

上位機送命令給 GMT Controller。

例如：

```text
Host PC
   |
   | TCP Port 9999
   |
   v
GMT SixAxes Controller
   |
   | 執行命令
   |
   v
回傳執行結果
```

這是雙向通訊：

* Host → Controller：命令。
* Controller → Host：命令回應。

例如：

```text
Host:
MOV ...

Controller:
>Done
```

以上只是示意，正式命令和回應格式仍要以後續 Command Manual 為準。

## Port 8888：Status Stream

這是狀態資料通道。

用途：

控制器定期傳送 Routine Status Packet。

```text
GMT SixAxes Controller
          |
          |
          | 每200ms
          |
          v
      Port 8888
          |
          v
    Monitoring Software
          |
          v
       Qt Client
```

這條通訊不需要上位機每次詢問：

「現在 X 軸位置是多少？」

控制器會按照週期主動提供狀態資料。

這種方式稱為 Asynchronous Status Stream。

### 為什麼要分開？

因為命令和狀態監控有不同需求。

| 比較   | Port 9999        | Port 8888        |
| ---- | ---------------- | ---------------- |
| 用途   | 命令控制             | 狀態監控             |
| 通訊模式 | Request-Response | 非同步資料流           |
| 方向   | 雙向               | 主要由控制器送出         |
| 更新方式 | 收到命令後處理          | 每 200ms          |
| 資料格式 | ASCII Command    | HEX ASCII Packet |
| 目的   | 控制設備             | 監控設備             |

這種分離設計很合理。

例如：

當 Qt Client 正在執行一個耗時的運動命令時，另外一條 Status Stream 仍然可以持續接收位置和狀態。

兩條通道不必互相等待。

---

# 六、特別注意：localhost:8888 的真正意義

現在 Overview 再次提到：

> Broadcast every 200 ms to localhost:8888

這句話值得特別研究。

`localhost` 通常代表：

```text
127.0.0.1
```

也就是 Loopback Address（迴送位址）。

這代表資料送往本機，而不是送往網路上的其他電腦。

這裡有一個重要的 TCP/IP 知識：

**TCP 是連線導向的資料流協定，不是一般意義上的廣播協定。**

因此，文件使用 Broadcast 這個字，可能只是泛指定期推送資料，而不是 TCP 層級的真正 Broadcast。

TCP 通訊通常需要 Client 和 Server：

| 角色              | 工作               |
| --------------- | ---------------- |
| TCP Server      | 建立監聽 Socket，等待連線 |
| TCP Client      | 主動連接 Server      |
| TCP Data Stream | 雙向傳輸資料           |

參考：

* [RFC 9293：Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293?utm_source=chatgpt.com)
* [Linux IP Man Pages：Loopback Address](https://man7.org/linux/man-pages/man7/ip.7.html?utm_source=chatgpt.com)

### 我認為這裡有兩種可能架構

**可能 A：GMT Controller 內部有 Status Stream Client。**

```text
GMT Controller
      |
      | TCP Client
      |
      v
127.0.0.1:8888
      |
      v
Local Status Server
      |
      v
Monitoring Software
```

**可能 B：控制器本身就是 Status Server。**

```text
GMT Controller
      |
      | TCP Server :8888
      |
      v
Host PC / Qt Client
```

但是第二種情況就不應該將目的位址描述為 `localhost:8888`。

因為遠端 Qt Client 必須連接控制器的實際 IP，而不是自己的 localhost。

所以，這裡需要向同事確認：

**GMT SixAxes Controller 的 Port 8888 究竟是 TCP Server，還是控制器內部的 TCP Client？**

這個問題會直接影響未來 Qt Client 的連線方式。

---

# 七、Internal Transport：EtherCAT、PDO、CiA 402

這一段讓我們更清楚理解 GMT Controller 的內部運作方式。

## 1. EtherCAT 是什麼？

![Image](https://images.openai.com/static-rsc-4/UXkcedTzL9JW1UiighnyDj5AvlLkmtNAhrm6XGwtWR-pfeCzS_kUl8xGqa4Z1VbuTQfovYHSR86X3hrM8W-0vDapIbRrAfMHAw8kMXXPb1ty3mAWR9V6K28XMjjzdohES-3iAbo4dgEXq5d-wFdFRwmoSaVSYqJabPSytiuRDmqvfBfaDPFe_Rc5clw4Ltnb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TAufyUJF5WOSO6IwRClbr2eu7XvB8MLkvYSoBPT5xrkw50ykhGb9QaXDYuat1fJ5rMSk9SR7mulWU0i3xODj-wWinbIxsGJ0Un629ZGP7gOwZzNKeTjumKDRKOZHNfdLk0Q6KbBpUBUsK43fcOd6_yK3uoflQ7SMplseT6aDDCzJlVUrlAu1IED9VkIOyi0_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tE0gL4G8jhYZexEwreEAukjjvbo9brt-OfkEMbV30kICZvuZd92uccyr8XmWr-Cvh-pAqbMlyj4kylNbWcCbF_2KLQaBJO6WzgIx23w2mbjLl8v3e2o4Iroo3fD9DmnlYE7F5DN50W6m5-rtbYET8JpOC6shK9Y9CUzLfEciPGFwscZ_DIazY9XppkJvK-ZX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QFHXEa-Bo-hWk8Ul9eAmuYT8yiyuA_zouPw4K9iNBT59BT2THeIPlFkuCXdRTqfbjnZbFy5QCs1TH2HuNO3wiONR4T4pBUSK8sPo12frPggz6wV-zayO1j2pTdlmjQXJv9kE3AtUVQ6V_RzzNaLfDZ5_TH2K7CVkU6KGbCQbZQ8xtLD0VGFK2L64Hovf5sJ7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TWatpqZeKAEOT68y72qifeTV8kMEFbi1RNydEJpqlx3cCaFmqCccrGJdmXtktnVc8Vdhmkl2c6vMCg-rdNJl-AYQTp32W5CapBDSLMUjCaBV6nDIZmV3h-4pNVFMXOHHaquZrcon1Z3cwKLIkVWJBTJaAAW-Sltpb76cH6lnjSa3Mkjiq1ucqkYfAfLXcERA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nKfbBinCMoCIjh6cPW626Jp1JE25RhFCMlnJspuZ-ZxQDsq7LxacqPtxKoKAW_VH-KbaZrEb69V86pWBnNKQdc4ZN1Wi6xl4FE7oO6jEclrJHav7oM3ld8ZuZE-4jezfSkhzMnAz3CyFDHpyz9TWtno6k7iPVkUFWGbZim5_WlbCI10w4D25Dzlp2028yeFo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zJBtsiAppOXsMwZgzJTjKZuRIoj2AUrJ0tT7q4zYQGqCHSd02Qx0LAYb4Y8F3muazWXn_1HIUDAZEcPcfj3nKAPhMhUYf44iM444o_WHqKfqn0ZGrGOkqm9_3d2dzFVMLF0J0zxYldbOFkqj9D9OTUS4Ua4Mk_jfRNCstg7S7OZxjbmVyD4VUyT1u3dCQnSM?purpose=fullsize)

EtherCAT 是工業乙太網路通訊技術。

主要用於：

* 工業自動化。
* 多軸運動控制。
* 伺服馬達控制。
* 即時 I/O 資料交換。

它與一般 TCP/IP 網路不同。

EtherCAT 有 Master 和 Slave 的架構。

例如：

```text
GMT Controller
  EtherCAT Master
        |
        |
        v
   EtherCAT Network
        |
   +----+----+---------+
   |         |         |
   v         v         v
 Servo 1   Servo 2   I/O Module
   |         |
   v         v
 Motor 1   Motor 2
```

GMT Controller 負責管理 EtherCAT 網路，Servo Drive 則作為 Slave。

這裡也要區分：

EtherCAT 不等於 TCP/IP。

雖然兩者都使用乙太網路相關技術，但 EtherCAT 的即時資料交換不是透過一般 TCP Socket 完成的。

## 2. PDO 是什麼？

PDO = Process Data Object。

中文：製程資料物件。

它負責週期性交換即時資料。

例如：

| 資料              | 傳輸方向               |
| --------------- | ------------------ |
| Target Position | Controller → Servo |
| Target Velocity | Controller → Servo |
| Controlword     | Controller → Servo |
| Actual Position | Servo → Controller |
| Actual Velocity | Servo → Controller |
| Statusword      | Servo → Controller |

這裡有兩個重要名稱：

### RxPDO

對 Slave 來說，接收來自 Master 的資料。

例如：

```text
Controller
    |
    | Target Position
    v
Servo Drive
```

### TxPDO

對 Slave 來說，傳送給 Master 的資料。

例如：

```text
Servo Drive
    |
    | Actual Position
    v
Controller
```

這個方向是以 Slave 的角度命名，初學者很容易搞混。

EtherCAT Technology Group 的官方資料也有說明 PDO Mapping 和 PDO Assignment 的運作方式：

[EtherCAT Technology Group：PDO Mapping and Configuration](https://www.ethercat.org/cn/technology/知識庫/pdo-mapping-and-configuration/?utm_source=chatgpt.com)

[Beckhoff：EtherCAT Slave Process Data Settings](https://infosys.beckhoff.com/content/1033/el6731/402899211.html?utm_source=chatgpt.com)

## 3. CiA 402 是什麼？

CiA 402 是工業驅動器的標準設備設定檔。

它規範了伺服驅動器的控制狀態、操作模式，以及常見控制資料。

例如：

| Object | 名稱                    | 用途      |
| ------ | --------------------- | ------- |
| 0x6040 | Controlword           | 控制驅動器狀態 |
| 0x6041 | Statusword            | 回報驅動器狀態 |
| 0x6060 | Modes of operation    | 操作模式    |
| 0x607A | Target position       | 目標位置    |
| 0x6064 | Position actual value | 實際位置    |
| 0x60FF | Target velocity       | 目標速度    |
| 0x606C | Velocity actual value | 實際速度    |

例如，CSP（Cyclic Synchronous Position）就是循環同步位置控制模式。

控制器每個 EtherCAT 週期送出新的目標位置，Servo Drive 持續依照目標位置控制馬達。

這與你之前在 STM32H755 CM7 專案中處理的 EtherCAT 軌跡資料非常相關。

可以參考：

* [QEC：EtherCAT CiA 402 User Manual](https://www.qec.tw/ethercat/api/cia402/?utm_source=chatgpt.com)
* [EtherCAT Technology Group：EtherCAT Technology](https://www.ethercat.org/en/technology.html?trk=public_post_comment-text&utm_source=chatgpt.com)

**但是要注意：GMT 文件只說支援 CiA 402-compliant servo drives，並沒有說明所有 Servo Drive 都使用完全相同的 PDO Mapping。**

實際 PDO 結構仍要看各驅動器的設定。

---

# 八、重新整理整套 GMT 系統與你目前專案的關係

現在有了 Overview、Routine Status 和前面兩段說明，我們可以建立比較完整的系統認知。

![Image](https://images.openai.com/static-rsc-4/dcIfCNlb21so4peR8cV2nYL8YjR3mx8qip9td4if2GaUUaD0nUeie-FnnClTN3iw86MmdxazzOuJ-cXLANuNq0AXDx_QxueXLqzGQmNCXgYAZlHyiU_xA0kky_cI3NZtIvhINz0uZjChgcv_ROrr1bYm4PaPM1Mv8TZPbyeuJ-z1Rzz8AayGoSoh8SQRztTk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GrzhspAPWrXezKhFBrOqdj_u15LHQhlnskeID_kQsEsFWLGhttswOWg6DJdK5iYXSxYGH0Y3IjXkvw0iI6g24EyYHvsgarTIdJKTILsOlc9dx1F6pdSJobY9Vjojkqcl11KYJxJxomiVWj5CksWHeSAqXbaDxc7zSE0KL_dnMjec0IbOA4BaKsyEj3diDrnz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bVKoZic3LPK3ZLXRIDTOm0GQ6G6F5RAxdH0vFzEE6XXkBkficXlfJQmy4zd55CYIJKwAJWGhzKEXMLwATme7UNQ7wA86r4TPrshYwXCHbdwaMp-QWR-3h95r_eGs9xratnmo8OeyAKeqZ51bCr7E5Y_1MVCiOIAECwqKtyTnc4jH3zVQt37ey1qoGRiB5lef?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rovW2Iae5ltSbbT7FTEg6bDH4CXihgqPVvFuFSsMbms7LEXiCDrqcKXHKKQZJtWzQERm89IVi2oIfYACUTar80PDF8x8pADFpHp0Zncjhvwt6KfXBvSjxwg3EvMMfssGrpHqozqg40KOzAHcImtAtprq4s6hfWICnsniFGFMeTyFdaIjUaAq34zYg5wb9CIs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QFHXEa-Bo-hWk8Ul9eAmuYT8yiyuA_zouPw4K9iNBT59BT2THeIPlFkuCXdRTqfbjnZbFy5QCs1TH2HuNO3wiONR4T4pBUSK8sPo12frPggz6wV-zayO1j2pTdlmjQXJv9kE3AtUVQ6V_RzzNaLfDZ5_TH2K7CVkU6KGbCQbZQ8xtLD0VGFK2L64Hovf5sJ7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zJBtsiAppOXsMwZgzJTjKZuRIoj2AUrJ0tT7q4zYQGqCHSd02Qx0LAYb4Y8F3muazWXn_1HIUDAZEcPcfj3nKAPhMhUYf44iM444o_WHqKfqn0ZGrGOkqm9_3d2dzFVMLF0J0zxYldbOFkqj9D9OTUS4Ua4Mk_jfRNCstg7S7OZxjbmVyD4VUyT1u3dCQnSM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UZlxTbS289Qh_9P0u41uHFIrQ3sK0MrNLuQKKF_3A00DxlSyvvsl5gP2R_Hp2MJSBwklb4LTArMjGKo47XC_NBMATp8xJy3FpMa20CYcc4YY9XOnnS8E_L5NyCz5L-E6AwggyXpTmsy4MRJ9lA8D31abPvCcYs3lo9JsIXKA_BrDx3INGFowEoc8s29bsXOO?purpose=fullsize)

我將它整理成以下架構：

```text
                 Host PC / Qt Client
                         |
                 +-------+-------+
                 |               |
                 | TCP 9999      | TCP 8888
                 |               |
                 v               ^
          Command Interface   Status Stream
                 |               |
                 v               |
          +-----------------------------+
          |                             |
          |     GMT SixAxes Controller  |
          |                             |
          |     Command Parser          |
          |     Central Data Handler    |
          |     Motion Coordinator      |
          |     Routine Packet Builder  |
          |                             |
          +--------------+--------------+
                         |
                      EtherCAT
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
       Main Stage    Piezo Stage     I/O Modules
       12 Axes        6 Axes         AI / DI
          |              |              |
          v              v              v
       Servo Drives   Servo Drives   Sensors
          |
          v
      Actual Position
      Encoder Feedback
```

這裡的 12 軸與 6 軸，是依照 Overview 所述的 Main Stage 和 Piezo Stage 數量整理。

但實際硬體分組和 Routine Packet 是否涵蓋全部 18 軸，還沒有得到確認。

## 與你的 STM32H755 / CM5 專案對照

| 項目      | GMT SixAxes | 你目前的整合系統             |
| ------- | ----------- | -------------------- |
| 上位機通訊   | TCP/IP      | Qt Client            |
| 命令介面    | TCP 9999    | GMT_Command_Gateway  |
| 狀態監控    | TCP 8888    | Status Stream Client |
| 底層控制    | EtherCAT    | STM32H755 CM7        |
| 週期性資料交換 | PDO         | EtherCAT PDO         |
| 軌跡資料傳輸  | 文件未完整說明     | SPI2 DMA             |
| 軌跡接收    | 文件未完整說明     | Raspberry Pi CM5     |
| 軌跡儲存    | 文件未完整說明     | CM5 UDS / LOG        |

這裡有一點值得特別強調：

GMT SixAxes 的 TCP Command、Routine Status Stream 和 EtherCAT PDO 是三種不同層次的資料交換。

你目前的 SPI2 軌跡傳輸則是另外一條資料路徑。

不要因為它們都在傳送位置資料，就將它們視為同一種協定。

---

# 九、重新檢查目前三段文件，發現的問題

這次我把 Overview 和 Routine Status 兩段放在一起，整理出更完整的疑點。

| 編號 | 項目               | 文件資訊                     | 我的判斷                     |
| -- | ---------------- | ------------------------ | ------------------------ |
| 1  | Main Stage       | 左右兩側，各六軸                 | 總共 12 個驅動軸               |
| 2  | Piezo Stage      | 左右兩側，各三軸                 | 總共 6 個驅動軸                |
| 3  | Drive Groups     | 三組，每組四軸                  | 剛好 12 軸，可能對應 Main Stage  |
| 4  | Routine Position | 六個 DOUBLE                | 與 12 軸 INT32 資料定義不一致     |
| 5  | AI Devices       | 兩組，每組四通道                 | Table 4 的 Device 編號可能錯誤  |
| 6  | Digital Input    | 定義 32-bit 狀態             | 未列入 68 Bytes 封包          |
| 7  | Gripper          | 支援 SMC Gripper           | Routine Packet 沒有對應欄位    |
| 8  | Camera           | 第一段提及 Camera Coordinates | 尚無資料結構                   |
| 9  | TCP 8888         | Status Stream            | TCP Server / Client 角色不明 |
| 10 | TCP 9999         | Command Interface        | 預期由控制器監聽，但仍需確認           |
| 11 | Axis Unit        | mm / degrees             | 尚未提供每個軸的換算規格             |
| 12 | PDO              | Cyclic PDO Exchange      | 尚未提供完整 PDO Mapping       |

其中，我認為有三個問題最值得優先確認。

### 問題 A：Routine Packet 是否真的包含全部 18 軸？

Overview 描述：

* Main Stage：12 軸。
* Piezo Stage：6 軸。

但 Routine Packet 只描述 48 Bytes 的位置資料。

目前可能性包括：

1. 只回報 Main Stage。
2. 只回報一組六自由度平台座標。
3. 透過某種運動學轉換後，回報部分代表性座標。
4. 文件漏掉其他資料欄位。

目前無法確定。

### 問題 B：為什麼 Routine Packet 提到 Gripper 和 Camera，卻沒有欄位？

Overview 說明支援 SMC Gripper。

Routine Status 第一段也說包含 Gripper States 和 Camera Coordinates。

但目前 68 Bytes 封包表格完全沒有這些欄位。

這可能代表：

* 這些資料透過其他命令取得。
* 文件的 Routine Packet 表格不完整。
* 文件混合了不同版本的規格。

這需要繼續閱讀後面的 Command Manual。

### 問題 C：Digital Input 是不是另一個獨立資料命令？

目前 Table 5 提供 32-bit Digital Input Mapping。

但 68 Bytes 的 Routine Packet 沒有預留 4 Bytes 給 Digital Input。

因此，我傾向先將它視為尚未確認是否屬於 Routine Packet 的獨立狀態資料。

不能直接把它塞進目前已定義的 68 Bytes 結構。

---

# 十、我建議接下來閱讀 Manual 的順序

現在我們已經掌握了系統架構。

接下來我建議依照以下順序閱讀：

| 順序 | 文件內容                  | 閱讀目的          |
| -- | --------------------- | ------------- |
| 1  | Overview              | 已完成，理解系統架構    |
| 2  | Routine Status        | 已完成，理解週期性狀態資料 |
| 3  | Command Description   | 理解所有 ASCII 命令 |
| 4  | Motion Command        | 理解六軸運動控制方式    |
| 5  | Servo / Drive Command | 理解伺服驅動器控制     |
| 6  | I/O Command           | 理解 AI、DI、DO   |
| 7  | Gripper Command       | 理解 SMC 夾爪控制   |
| 8  | Camera Command        | 理解相機座標及資料     |
| 9  | Error / Response      | 理解錯誤碼及命令回應    |

尤其是接下來的 Command Description，我認為非常重要。

因為我們現在已經知道：

* Port 9999 是命令介面。
* 命令採用 ASCII。
* 命令以 `\r\n` 結尾。
* 命令使用空格或參數分隔。
* 命令可能有同步回應。

接下來就要了解每個命令的實際格式。

這也能讓我們判斷目前的 GMT_Command_Gateway 是否已經完整支援同事要求的命令。

---

## 最後，我想先建立一個重要結論

從這三段文件來看，GMT SixAxes Controller 的核心設計理念是：

**上位機負責下達高階命令與監控，GMT Controller 負責運動控制協調，EtherCAT 負責即時驅動器與 I/O 資料交換。**

這三個層次分工明確。

而你目前 STM32H755、Raspberry Pi CM5 和 Qt 的整合工作，正好涉及這三個層次之間的資料傳輸。

另外，我建議先保留以下四個待確認問題，後續閱讀文件時再逐一核對：

1. Main Stage 的 12 軸是否就是 3 個 Drive Groups × 4 Axes？
2. Routine Packet 的 48 Bytes 究竟是 6 個 DOUBLE，還是 12 個 INT32？
3. Port 8888 是控制器 TCP Server，還是控制器內部的 TCP Client？
4. Gripper、Camera 和 Digital Input 是否屬於 Routine Packet？

**你可以繼續貼下一章，我會把這些疑點帶入後續分析，不會每讀一章就重新猜測一次。**


---

# Q & A

```
什麼是 Load Cell 和 PM ？
```

在這份 SixAxes Controller EtherCAT 系統文件中，**Load Cell（荷重元）和 PM（文件中的縮寫）都出現在 Analog Input（類比輸入，AI）通道配置表中**。它們應該是控制器用來取得外部感測資訊的輸入來源。

不過，這兩者的角色並不相同。我先從最基本的概念開始說明。

# 一、先了解 Analog Input（類比輸入）是什麼

STM32 或控制器除了可以接收數位訊號（Digital Input），也可以接收類比訊號（Analog Input）。

兩者最大的差別是：

| 類型                | 訊號形式       | 例子           |
| ----------------- | ---------- | ------------ |
| Digital Input（DI） | 只有 0 或 1   | 開關開啟或關閉、限位訊號 |
| Analog Input（AI）  | 連續變化的電壓或電流 | 壓力、重量、溫度、光強度 |

例如：

假設某個感測器輸出 0～10V：

* 0V：可能代表沒有偵測到負載。
* 2V：代表有一定程度的負載。
* 5V：代表更大的負載。
* 10V：代表感測器量測範圍的上限。

控制器透過 Analog Input 讀取電壓，再依照感測器的特性，將電壓換算成實際的物理量。

例如：

```text
外部感測器
    │
    │ 輸出類比電壓
    ▼
GMT Controller Analog Input
    │
    │ ADC 轉換
    ▼
Raw Data（UINT16）
    │
    │ 電壓換算
    ▼
實際量測數值
```

這就是 Load Cell 和 PM 會出現在 Analog Input 表格中的原因。

---

# 二、Load Cell 是什麼？

![Image](https://images.openai.com/static-rsc-4/GXycd_hSuttwuuzsoQAy0VnxQzjccmVtitA1t0TEsxCKOzTwOPjTycFMlEPJ9Lrc9xoD4wV2Uva9OXVtEU6kS_PZ7S8_6fFE5oyi9cXZcpLFAHRS-UqzIu4owMpfapCB-5GMYp170XGxNEy2eEED-7FJZ8X2kKl1ZPzxeNkXQu1usOuXHXKvXD2hDZWHY3Oa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gusgRTQqE6Qp52dapq24gZAb1ujHpO2dcOptFe-7Rf2MMjNbNwWbhOYT-jP_BkXctBcM8TvSaQgMN8Y7BIIHWgelEZ8xSatRtuwuRBAlFNvu4xlz60m9aW6LKNzWwgMuz2k6SQpyOfYQyJ9_Uux93U3H4KFILW0txrIEgRK18WG4xAuHutuDSlMDOy698XaG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/F238crhFgpL32uS8kpR7iRMOf8JkDB1KByT88SkJBbf_CzGIgARzvZeXkcdwok_AEu6HIyjB1TsT3xnrMkpkOz9dll7dJe0DHjRfG1X0Cb7gX_cjGvY-69aovOSGcjAbIAMyd_k5UCaRaA_0575QoF7_BR9ru6QCeIXex7ABwygd3jr-GAos9LfKNLwKkGu1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W-NM_iMcJutLhpPuFKcpxxTt0I91s_-lTvfU_faPvH2IYfUXJK9ugrcIQwPS9KnE6zzQBQfM8mY0IpwFcNCpVhbLiQHDe0F3dL8f_J90qDRJut_eLfx-udP-QOp2eTSk-PdOD_mK2N3FAW3WcM2koDcwLEhreIRxTyCmQkws_tlej41tLUgJ69er4yszITTC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gnab43XwZYa4XggQwWDjy6-TWnN4pN2nzT3xRZxxkDQjPh2v278r_o4JgLCy871h3W0-5qkZUD48HetwNJhE8iJJGXflXTnu-epd6vPr3cMNSzwecrSDrdYf226biOdhLFFZjLNgXMkpsWajahmuZEqVM-unorTjFanRfMeazkIiuMVZUYRUtCkPe0EQ4BO0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MQrqy9aPHYGXESurd88rkMyfKOwve2LQkyBdUUDgEMSStm5Nm6A0k_DkQZl7xxmjgVH4tHhCeBCa5RBp46PwgdlhqkQ4JUIE1PId5FKE9ncobNjpG8mKIrKANwcFU_1Mp7E5nm23OXyVvXk0v0CUHo-bqqwdK-_aAjRetZoaHVGUqkfTyE_JT9acPGQvTpYb?purpose=fullsize)

**Load Cell 中文稱為荷重元或負載感測器，是用來測量力量或重量的感測器。**

簡單來說，它可以讓機器知道：

* 目前承受多少重量？
* 目前受到多大的力量？
* 機構是否施加了過大的壓力？
* 執行機構接觸物體時，施加了多少力量？

## 2.1 Load Cell 如何運作？

常見的 Load Cell 使用一種叫做 Strain Gauge（應變規）的元件。

它的基本原理如下：

1. Load Cell 受到外力。
2. 內部金屬結構產生非常微小的變形。
3. 應變規隨著金屬結構變形，電阻也跟著改變。
4. 電路將這個微小的電阻變化轉換成電壓訊號。
5. 控制器讀取電壓，計算出受到的力量。

例如：

```text
       外部施加力量
             ↓
      ┌───────────┐
      │ Load Cell │
      └───────────┘
             │
             │ 電壓訊號
             ▼
       Analog Input
             │
             ▼
       ADC Raw Data
             │
             ▼
       電壓換算
             │
             ▼
       力量 / 重量
```

注意：Load Cell 輸出的電壓通常非常微小，實際系統可能需要搭配放大器或訊號調理電路，才能提供控制器可以讀取的電壓。

## 2.2 在 GMT SixAxes 系統中扮演什麼角色？

依照文件的 Analog Input Mapping：

| 通道   | 功能        |
| ---- | --------- |
| AI00 | Load Cell |
| AI01 | Reserved  |
| AI02 | I2C0      |
| AI03 | I2C3      |

這表示 GMT Controller 將 AI00 指定給 Load Cell 使用。

因此，合理的理解是：

**GMT Controller 可以透過 AI00 取得 Load Cell 的量測訊號，讓上層控制程式或監控軟體知道機構目前受到多少力量。**

例如，假設六軸機構正在移動，末端機構接觸到一個物體：

```text
六軸機構移動
      │
      ▼
末端機構接觸物體
      │
      ▼
產生接觸力量
      │
      ▼
Load Cell 偵測力量
      │
      ▼
GMT Controller 讀取 AI00
      │
      ▼
上層軟體取得量測資訊
```

這種資訊可能用於：

* 接觸力量監控。
* 避免機構施加過大的力量。
* 量測機構的負載。
* 力量控制或接觸判斷。

但要特別注意：**文件目前只證明 AI00 對應 Load Cell，並沒有證明系統已經實作力回授控制（Force Feedback Control）。**

有 Load Cell，不代表一定有力量閉迴路控制。這需要看後續控制命令及控制程式。

## 2.3 文件中的 Load Cell 電壓換算

文件提供的公式是：

$$
V=\left(\frac{RawData}{65535}\times20\right)-10
$$

代表：

| Raw Data |  換算電壓 |
| -------: | ----: |
|        0 |  -10V |
|    16384 | 約 -5V |
|    32768 |  約 0V |
|    49151 | 約 +5V |
|    65535 |  +10V |

這裡有一個很重要的觀念：

**這個公式只能將 ADC 數值換算成電壓，不能直接換算成重量或力量。**

例如，假設 Load Cell 讀到 2.5V，並不代表力量就是 2.5N。

如果要換算成牛頓（N），還需要知道：

* Load Cell 的額定量程。
* 感測器的靈敏度。
* 放大器的放大倍率。
* 零點偏移。
* 校正係數。

這些資訊目前沒有出現在我們看到的文件內容中。

---

# 三、PM 是什麼？

![Image](https://images.openai.com/static-rsc-4/Z5CcML1YFUtqCJjljKF3JQUEx5MpmLuecz9ZqKF7Ur2KMDZvB2ZWEOzSxJ_AcoVmxrF0Lu6zNfM4UyK_U1a_I7FDS1qlrRZHNGbCRUQ5nSw83g9EcrYizat0qXadNkCiaORHqAE4Pf3kW3-WmOtaAGyvy5S64TsbIpwVmK2dmUCWHCgfc1YcmOHueYRWfPJ3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LFSd7c7L2UsMT3xGXw7a-nxyRc84XO1Y3kCe69FdNdCQIFhblcnOefthnip6N_LsG87DNQDwmEc_DGMEYXTOIM1F6Xx_-lTYKP5O-bMN_D4LTBpxKQV4Xo7KDz35K7bGFEM9JA1zl-I2j3GDM4qP0yo0nYvZBQmZpGZHoh_bEjWxpDtKuoqlDFlLKMEoCCzu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0YkJMWAO9vPgkBWg8DYHtEaKxCoszBDQTa6xJj0tmHnxEQAm03CMy5Iqc1wNgdIItyOxaOGI0vaMvrw4XM8HbKb0PyMWjtvvMeGaQqy82e5rqhG5E8jnXlm_b-y6DOaqPwT-MHUbKd-xNySIeCUUX9-SMJHJdNhFXvz71u-jjmDBCnD-ToklToJ8l4Nu6AS_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TuGGgPFGDqCimTnTUJRJ4cCDO0goj0WGW264SjRnyDHwUT-qk-3-yhVfhVkUauA8KHfgEZHPJryi9n-EtsOMo52Rtd9nYY8CKPVO39K_BItMYS7HotEcNMpUkJEvMynYcu-xx_gdh3ciOUooyQwxgROz_-UY3lBRbccYJoM0VnT91-v5jGUaWCfdQV5OYu3O?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XCEvwg1d8CG0lGlik4vXnWBMpdjZN4izIh7mSLTDy6qnR-7o4IW-LYjarNN5l1vaETsmOfoOF4QTSAV38cmpHsysGqgfxtcU-LK60Ikzw9Cb2kN8XR6ouDIID3vFy9Faf571gg0yw05iyspnhaPyCl1tl3FMiv-4J0KGLlaa36X4a3J09TrFpVNgTeSX0Y8M?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/s0aNu1D-JwDGdqplvQwdHSucVWr7DSWWqc6Uv92dZAHJkKjyZl9wUbIzwGfKJbCYJOfCQyRddOi0m12rIDUYmhQSvutUUiPr6fSpOwXNVE0x0dpPdApWpY0yQq31P7ZKGpTnGV6OJeFmcZk00iTI6-pNmZ_HW8KVsGHbrkLlDrIELoZeW80KIQUlcvQJQXB6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/raYwTLMi3HGUMZEeG_CRXUCewLlqpVWmAsh14YcW0-mxVu9qMAOs8D7AhW3JA5PlR-zZm0-0IKbgEn5ipkpYtPS9A-TEDurHjLUWDFYybYgZrt-jJCfvH9Yc_hHbQwSUbXv1f1PpFshdJjTKDvZGb2UzDcSyue7rAx0V0Ckc2g8fQ-urmSeCU78hto-cZhRF?purpose=fullsize)

這部分需要特別小心。

**目前文件只寫 PM，沒有提供完整英文名稱，因此不能直接確定 PM 的意思。**

在工業設備及光電系統中，PM 有可能代表 Power Meter（功率計），尤其是搭配文件中的通道配置：

| 通道   | 功能     |
| ---- | ------ |
| AI04 | PM Ch0 |
| AI05 | PM Ch1 |
| AI06 | PM Ch2 |
| AI07 | PM Ch3 |

四個通道都標示 PM，讓我認為它有可能是 Power Meter 的縮寫。

如果這裡的 PM 確實是 Power Meter，那麼它就是用來測量某種設備輸出的功率。

例如，若系統使用雷射或光學設備，Power Meter 就可能用來監測光功率。

## 3.1 如果 PM 是 Power Meter，它如何運作？

![Image](https://images.openai.com/static-rsc-4/tP1CX452Q4AdmaxFwGyZ89hD9rigWr9DM89deBIM4Z2vxvT8LNsYPbU56yAQuAWtXGxXPrm4monW28TQmfJUXsOZybOGD514EqmaFrjV4AYN5NOgMQwiI2U8lh4DxC_2SIWyyFiIcGRvnJ-Glh_SQwJAM2InEHEu8KzXTfFgCvPka3XfON8tLNyPkJ3k1yC_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/eLC7VMzS8IHDnSKSpBm9bAq5geWXhwfZqiXi7_esLiF72ciCDfvb_ODIxkGfJowY634leR5wr_Kpgyd4iuEtiwEqj0tl0S2kAovniAraczxC9J-jVWYzJv8PQKoWnnhY3JMGgpxnX1T47SjEbIYK1nMqdaeRtIZLP6zkNs12P85lweGU8n7FuApkvE0z8COJ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/byK1Iejz4fBVYOxYvpBQig89zOn35EhFzaeG7RBUcKUJqKy9-BfU-vO2pZ6iC-KZtxrTMq0_QjI2HPWVy04BiLn216EwW8kYB3BwutdtUtkdlQLiMEVyEBBOfl9QGyHowQ5UJ4T-OA4O2HJ9iKbHv_jxf_qETI_l7AwH2oVxUD7KvQGxBMUwslUq4JRtwXvI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xI7RaAE9d-cvjx8dw3LcvCfU_03fNAklGwWdJqMO3voFuJEqYnGWuy0i-LCyyiwB-85Y03c2E-Uf-IT7fog3qZwvauOic3SgOMj6sLBz8EZw8ndGwmfT-65IM7M8uspST-d6BO8V7tK5Xe3vE8Les2vYqv-MY_wbFjPEGA3qTR7A9W79kKyRurwXXR8L3NkW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vyGBhYBQFEFeMbj5aRghCTdfjaSW07N122nysy2hcx8Sx5LyB2BF5mAlIOZ2RnOcziP6-AVXgi7mesuJz-a1UZ91em83Ud9hdruyCmQ1XzOfmt9itQZX25D2em0HHCVbrKfRdK6aSLYP-LTtzyPw_jP-0ijVDOAAW6vB9ONNN4u-WqsX9CftHEBhkpxjA0uo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dSuxOaUWRcDS4DQkEOvy1zutiWN2aFhgCGuKtKwV6sr9f7FbEhN5_vWhsQv7be_ROpYHwx2VO7ai75miGXg-r7YXi5LFK2rLrjTY3WLXd2nQWY8Jh7jKtPsuhvK8SBYT8dHZ9gaDkuqmQ1lD4JZhRlcSNOuxf553qUsVFqmUI2OFAjskMFmbUwtA_5v3UKHW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4pwkME1LedKkzzr1_RHtpCANOnRlg0pKoxf3BVZp8IY8D9TJY7f-2XdEa_3riOVt6JPamFJy1q-d_9m4nwYfNG21maTmNeryt584aJCjDJE7_WSVc99b6gBoJEq13fLssMJxXvImQvv6ySSKRj7QPO7qc-xVXauKjpz3tlaSv7T26yXt_a0iKgHmw2VwuwXL?purpose=fullsize)

以光功率計為例：

1. 光源輸出光線。
2. Power Meter 感測光線的能量。
3. 感測器將光能轉換成電訊號。
4. 經過內部電路處理，輸出電壓。
5. GMT Controller 透過 Analog Input 讀取電壓。
6. 軟體依照感測器規格換算成光功率。

例如：

```text
雷射光源
   │
   ▼
Power Meter Sensor
   │
   │ 輸出電壓
   ▼
GMT Controller Analog Input
   │
   ▼
ADC Raw Data
   │
   ▼
電壓換算
   │
   ▼
功率數值（例如 W 或 mW）
```

如果確實是這種用途，那麼它可能用來：

* 監控光源輸出功率。
* 比較不同光源或通道的輸出。
* 監控設備運作是否穩定。
* 提供上層軟體顯示或記錄。

但是，以上只是以 Power Meter 為前提的解釋，**目前不能確認 GMT 系統中的 PM 就是光功率計，也不能確認它的量測對象。**

## 3.2 文件中的 PM 電壓換算

文件提供：

$$
V=\frac{RawData}{65535}\times10
$$

這代表 PM 的 Analog Input 使用 0～10V 的換算範圍。

| Raw Data |   換算電壓 |
| -------: | -----: |
|        0 |     0V |
|    16384 | 約 2.5V |
|    32768 |   約 5V |
|    49151 | 約 7.5V |
|    65535 |    10V |

同樣要注意：

這個公式只負責將 Raw Data 換算成電壓。

如果要進一步換算成 W、mW 或其他物理單位，還需要 PM 感測器的轉換比例及校正資料。

---

# 四、Load Cell 和 PM 在整個系統中的關係

把兩者放在一起看，就比較容易理解了。

| 比較項目          | Load Cell | PM（尚待確認）             |
| ------------- | --------- | -------------------- |
| 可能的中文名稱       | 荷重元       | 若為 Power Meter，則為功率計 |
| 主要用途          | 測量力量或重量   | 若為 Power Meter，則測量功率 |
| 感測對象          | 機構受到的外力   | 可能是光功率或其他功率          |
| Controller 通道 | AI00      | AI04～AI07            |
| 電壓範圍          | -10V～+10V | 0V～+10V              |
| Raw Data      | UINT16    | UINT16               |
| 換算結果          | 電壓        | 電壓                   |
| 是否能直接得到物理量    | 不行，還需要校正  | 不行，還需要校正             |

從系統架構來看，可以理解成：

```text
              GMT SixAxes Controller
                       │
               Analog Input（AI）
                       │
          ┌────────────┴────────────┐
          │                         │
       AI00                     AI04~AI07
          │                         │
          ▼                         ▼
      Load Cell                 PM Sensors
          │                         │
          ▼                         ▼
       力量訊號                 功率或其他訊號
          │                         │
          └────────────┬────────────┘
                       │
                       ▼
                  ADC Raw Data
                       │
                       ▼
                 Routine Status
                       │
                       ▼
                 Host / Qt Client
```

這張圖是依照文件的通道配置及 Analog Input 機制整理的概念圖；目前還不能確認所有 AI 數值都已經包含在 Routine Status Packet 中，也不能確認 Qt Client 是否已經顯示這些物理量。

---

# 五、我認為目前最值得向同事確認的兩件事

尤其是 PM，我不希望因為縮寫常見就直接告訴你錯誤答案。

建議向同事確認：

**問題 1：PM 的完整名稱是什麼？**

是否為 Power Meter？如果是，它測量的是雷射光功率、馬達功率，還是其他設備的功率？

**問題 2：Load Cell 和 PM 的物理量換算在哪裡進行？**

目前文件只提供電壓換算公式。是否還有其他校正參數，可以將電壓換算成 N、kg、W 或 mW？

這會影響你未來在 Qt Client 中顯示資料的方式。

例如，你可能會遇到兩種不同的設計：

* **方式 A：** STM32 / GMT Controller 傳送 Raw Data 或電壓，由 Qt Client 負責換算。
* **方式 B：** STM32 / GMT Controller 已經完成物理量換算，Qt Client 只負責顯示。

兩者的程式架構差異很大。

**我的建議是：先把 Load Cell 理解成「測量機構受力的感測器」，把 PM 暫時視為「可能是功率計的感測通道」，等同事確認縮寫後再定案。** 這樣我們後續分析命令與 Routine Status Packet 時，就不會建立在錯誤假設上。

---

```
Gripper、Camera 是什麼？在這個系統扮演什麼角色？以下網址：https://www.gmtglobalinc.com/ 請你搜尋產品，對應目前我想要了解的所有問題或是剛剛貼給你的文件內容來回答。也就是搜尋此網址內的產品及說明頁面，充分理解後，回答我的問題，還有可以修正剛剛你貼給我的回答。
```

我重新搜尋了你提供的 **GMT GLOBAL 官方網站**，這次也把產品頁面、設備介紹及產品型錄放在一起比對。

這次查到的資料很有幫助，因為 GMT 官方網站展示的設備，確實能讓我們更具體地理解《SixAxes Controller EtherCAT Command Manual》裡的 Gripper、Camera、Load Cell 和 PM 分別在做什麼。

先說結論：

**這套系統不只是控制馬達移動的控制器，它更像是一套精密光學對位與自動化設備的運動控制核心。**

它除了控制平台的位置，也可能需要知道：

* 工件是否被夾住？
* 相機看到的工件位置在哪裡？
* 平台是否施加了適當的力量？
* 光學元件耦合後，光功率是否達到最佳值？
* 各個軸目前的位置及運動狀態如何？

這幾件事就分別涉及 Gripper、Camera、Load Cell、Power Meter。

以下逐一說明。

# 一、先了解 GMT 的產品實際在做什麼

我找到幾個與這份手冊非常相關的官方產品。

## 1. Multi-Device Active Aligner（多器件主動對位耦合機）

![Image](https://images.openai.com/static-rsc-4/TOCtimf321PLYNQ4JMWKHaOdX0K1knYcNab5oZnToN7WgEiBv37bKMH0OXLcbA3TFwqJNS6t74rB7ZQWFrJW8SSpGnc8038Zt4tfAtEBnYyeGfyKCBlnAKTjypWwQrwPWOp4XvRB5aLhsmKkHP2NI_CS1ygm0hcSLyFiXB9LowYJt7ezvQ3w5W08BFhz0K5x?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bWzhYmCFXBromfOPOGxY5jMesZv_G5jhcR5v8-o3SdavBXoNdK-mOG7RYXKhNDX0QtxwFWyNHSeC8BMs6QZ0VsLLHQBq3W-t_kusmevkxxzGP-dCi2hsYzd0ECiwiSoJWTP3-9iTuTKn9OiNihJ-BbCBuTtK1KnK9Qc4iWjHk2TtpO5gQV3bTa2Z193EB0Ic?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J1FaiHHOXeFexRa0LDjA9CQb9m6_fqXgfFhjgsxjLabmGv2Y6gntgLtsAPPBGVh-O3V3-VWgzAraBvwkV0IN19tgTg0WnRDnNCFI8OxyKmTGw3qBoYcdNzn2spm2ojVngaTcQ22uG4DxEoxw02qK2lbIDRiBUgRocFmJxLFE5OrerBj0nOek9QIZxSA6RUq4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mHzSq4GcoXt0xRpcn4DGOJOvTE3g5K30I5KK0VUMXyt822joETpeFiYQm6XlO5DPjLzV5q1da-1al6tYpCXyTDaNEY3XUwm0a40uVfn24NXJs6IandZikkBMFJ3Z4MCA3Z-eX0Si7pPaLLBCEZEW4aDPH_Xi8eEr9NFiwEHqgXniNs7nv5ums_v7UaxbDnhY?purpose=fullsize)

官方產品頁：

[GMT Multi-Device Active Aligner](https://www.gmtglobalinc.com/en/equipment/multi-device-active-aligner?utm_source=chatgpt.com)

這台設備主要用途包括：

* Dual Lens Coupling：雙透鏡耦合。
* Fiber Array Rx Coupling：光纖陣列接收端耦合。
* Fiber Array Tx Coupling：光纖陣列發射端耦合。
* Isolator Mounting：光隔離器安裝。

官方規格還提到：

| 項目      | 規格                |
| ------- | ----------------- |
| 運動平台    | Dual 6-axis       |
| 相機數量    | 3 台               |
| 相機解析度   | 500 萬畫素           |
| 相機解析度精度 | 1.4 μm            |
| 相機移動軸   | XYZ 三軸            |
| 工件固定    | 電動夾具、真空吸附         |
| 力量相關    | Pressure sensing  |
| 雷射功率計   | Santec MPM-220 系列 |

這裡很重要。

我們之前討論的 SixAxes Controller 裡面有：

* Main Stage
* Piezo Stage
* Gripper
* Camera
* Load Cell
* PM

現在從 GMT 的實際設備可以看出，這些不是毫無關聯的獨立功能，而是精密光學對位設備中常見的不同子系統。

不過，這台設備的規格不能直接當成你手上那份 SixAxes Controller 的實際硬體配置。它是用來理解 GMT 應用場景的參考。

---

# 二、Gripper 是什麼？

![Image](https://images.openai.com/static-rsc-4/mloLaHtQUPFME7NvB_FYGPeRhy0XDyAolRY7VdwI7f3yZWIRCw-9jxX8Hw94M6bzj3A3_y_NzIK_MKjWryiyyHCjC9XaacEVre9a-KVaBKcz6khDfkKgCUaFj1PM0l5LiRxFSVVtry8dPhmY41KCbeP1U-Ea7KCNSKkCc3acYiqim2-KTpqZPc0_Fx0Fylod?purpose=fullsize)

Gripper 中文通常稱為：

**夾爪、機械夾爪或夾持器。**

它的工作很簡單，就是夾住工件，避免工件在機器移動或加工時發生位移。

## 2.1 為什麼精密設備需要 Gripper？

假設我們要將一個非常小的光學鏡片安裝到光學模組上。

整個流程可能是：

1. 機器取得一個鏡片。
2. Gripper 夾住鏡片。
3. 六軸平台移動。
4. Camera 辨識鏡片與目標位置。
5. 控制器調整平台位置。
6. 將鏡片精確放置到指定位置。
7. 完成耦合或固定。

如果沒有 Gripper，鏡片可能在移動過程中掉落或偏移。

所以 Gripper 的角色就是：

**負責夾持、固定、釋放工件。**

## 2.2 GMT 的 Gripper 有什麼特色？

我查到 GMT 官方的 Electric Grippers 產品頁：

[GMT Electric Grippers](https://www.gmtglobalinc.com/en/electric-gripper?utm_source=chatgpt.com)

官方特別提到：

* 使用可程式化的驅動器。
* 可以依照預先設定的扭矩限制控制夾持力量。
* 可以精確定位及夾取物件。
* 可以依照物件形狀、尺寸進行簡單的瑕疵篩選。

這代表 Gripper 不一定只是單純的「打開、關閉」。

它還可以控制夾持力量。

例如：

假設我們要夾一個非常脆弱的光學元件：

| 夾持力量 | 可能結果   |
| ---- | ------ |
| 太小   | 工件可能滑落 |
| 適當   | 工件穩定固定 |
| 太大   | 工件可能損壞 |

所以 Gripper 的控制通常還需要考慮夾持力量。

## 2.3 Gripper 與 SixAxes Controller 的關係

你之前提供的手冊在 Overview 提到：

> device profiles support CiA402 servo drives, digital/analog I/O, SMC grippers.

這表示控制器除了控制伺服軸，也考慮了 Gripper 這種外部設備。

這裡要區分兩件事：

* Servo Axis：控制平台移動。
* Gripper：控制夾爪開啟、閉合及夾持。

Gripper 不一定是六軸平台的一部分，也不一定占用六軸運動座標。

它通常是另一個獨立的設備。

另外，手冊提到 SMC Grippers，而 GMT 官方網站也有自己的 Electric Grippers 產品。兩者不能直接認定是同一款硬體。

**在這份手冊中，Gripper 的實際型號、通訊方式和控制訊號仍然需要確認。**

## 2.4 為什麼 Routine Status 會有 Gripper States？

手冊提到 Routine Status Packet 包含 Gripper States。

這就很好理解了。

控制器可能需要讓上位機知道：

* Gripper 是否開啟？
* Gripper 是否閉合？
* Gripper 是否完成夾持？
* Gripper 是否發生錯誤？

例如：

```text
Qt Client
    │
    ▼
GMT Controller
    │
    ▼
Gripper
    │
    ▼
夾持工件
```

Gripper 完成動作後，控制器可以把狀態回報給 Qt Client。

這樣 Qt Client 就能顯示：

```text
Gripper: OPEN
Gripper: CLOSED
```

實際可回報哪些狀態，要以手冊後續的命令及資料定義為準。

---

# 三、Camera 是什麼？

![Image](https://images.openai.com/static-rsc-4/bWzhYmCFXBromfOPOGxY5jMesZv_G5jhcR5v8-o3SdavBXoNdK-mOG7RYXKhNDX0QtxwFWyNHSeC8BMs6QZ0VsLLHQBq3W-t_kusmevkxxzGP-dCi2hsYzd0ECiwiSoJWTP3-9iTuTKn9OiNihJ-BbCBuTtK1KnK9Qc4iWjHk2TtpO5gQV3bTa2Z193EB0Ic?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/N-KZbcBLF1mXmaF-0BOn8PDLpfUTwEcq6J0Z6W-Ye0bmtepSW4DGcUkkLbd3uk_ooLBXrE0KUMWw_rqViJV1gXg1OaPUPzPP3wVk9oG-zzmYvxPVnkNpDegGxFyHJOGklNmy9hF5k5o--Wt8gWo0UbsuxzoKw5ilIWY7K-2DdQ3BdriNslgkpnxTlYRcOCRy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bcvkQuh39lAYZaXdeGWsEP6nR7SpUobkOuWFYd9B29zazKTFMaAV-blmdPVhFOl5aAPzUFMdyX6s41BbHvq9lBRvZ2mBev8Pp__lf1ljE7GtD75_e6Orpx3Un0V-1EZMv1-2Bsf2m6CsKfJziuMg8RUEGfskbKU8U5OeI_dkhwpn7kptyLOnFDg7YYAq4uFN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mj3nuQZoisw2Qh9fANofjOGIufm0r1ZaeRXTZMb6fqJ1PZQzo9ERzh5Ong_wp8BoDZp99vIoPool83bWIf3tcVJMgTM5unhzf3UYU22yCsOrlU6S9H66NrVkl20XViJXgoiOVabA7y5DlxqRQhAhr6nCLm_3g5pW--eCYuwzx329Yq-M3iufBlEhRL_HvJ2U?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/C3eax-EgGCto_YvU3-PhYYDFAlrmqbqlV3iE25-R6mXu-tMlehlG5IR2SIPjXY2KYXfrYZ1XpW9IfqBHMo22J-n6to3dBeCFiR9qGJfWMFMPrDfhJmpf4nIcwX91cycrq5I4XNQoDxRn54gnejLlyN9Yc4Uux1mbIYK7OYBT0s3SWOBRm4ygARojERAdeIqj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4gPYV-WPvIbj_ZqqkM2Xms6IrnG-NcM9lK_Tj08YHtTfiCPsJL3pGQQTl-YoeWn0HDFhVh_S4rVyq0hHYffHwdhEUzEYEefRS6AbRwRd4cOSRQb6EslsP0AhqKdzyaZadKt6xUbRYfngqDeEyoXit4rSiJ8aRbWKSF4GCovunKVQkI-wH_UU7lBiavPdRc90?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_oMg9USZQb4LuxnH7nSKFCBN9vjjF8binAvGTWdQHYcUcXaLpgFtyzP15ZxneyssoSy077FbkA8I2EeEpDE90e3mKCf_ykR1b6YAoglc57Ph5Lh99MKRi0tb2751tE7btN-Fp3pbQQJqYaokiLsckbN8aCqCfYSkCTcpxc4AgUIx4TZzfsYeA25Zt5-Sns0a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TOCtimf321PLYNQ4JMWKHaOdX0K1knYcNab5oZnToN7WgEiBv37bKMH0OXLcbA3TFwqJNS6t74rB7ZQWFrJW8SSpGnc8038Zt4tfAtEBnYyeGfyKCBlnAKTjypWwQrwPWOp4XvRB5aLhsmKkHP2NI_CS1ygm0hcSLyFiXB9LowYJt7ezvQ3w5W08BFhz0K5x?purpose=fullsize)

Camera 就是相機。

但在 GMT 的精密設備中，它不是一般用來拍照的相機，而是：

**Machine Vision（機器視覺）系統的一部分。**

它的主要工作是讓機器知道工件實際的位置與方向。

## 3.1 為什麼六軸平台需要 Camera？

假設我們要把一個光纖陣列對準一個光學元件。

我們知道目標位置應該是：

```text
X = 10.000 mm
Y = 20.000 mm
Z = 5.000 mm
```

但是，工件實際放置的位置可能有誤差。

例如：

```text
預期位置：
X = 10.000 mm
Y = 20.000 mm

實際位置：
X = 10.025 mm
Y = 19.980 mm
```

如果只依照預先設定的座標移動，兩個光學元件可能無法正確對準。

這時候就需要 Camera。

Camera 拍攝工件，透過影像辨識找出：

* 工件中心點。
* 工件邊緣。
* 工件方向。
* 參考標記的位置。
* 工件與目標位置的偏差。

接著把這些資訊交給控制系統。

控制器就能根據誤差調整平台。

這個流程叫做 Vision-based Alignment（視覺對位）。

## 3.2 Camera 和六軸平台如何配合？

可以想像成：

```text
             Camera
                │
                ▼
          拍攝工件影像
                │
                ▼
          影像辨識處理
                │
                ▼
        計算位置及角度誤差
                │
                ▼
          GMT Controller
                │
                ▼
          六軸平台調整
                │
                ▼
          工件精密對位
```

例如：

Camera 偵測到工件偏右 5 μm。

控制器就可以指示平台往左修正 5 μm。

實際系統還會涉及相機座標系與平台座標系之間的轉換、標定及運動學計算，並不只是直接把影像座標當成馬達座標。

## 3.3 GMT 官方產品如何使用 Camera？

在 Multi-Device Active Aligner 中，官方規格明確提到：

* 3 台相機。
* 500 萬畫素 CCD。
* 1.4 μm 解析度。
* 相機具有獨立的 XYZ 三軸移動機構。

([高明鐵 GMT][1])

這裡還有一個值得注意的地方：

**Camera 本身也可能有自己的運動軸。**

也就是說，除了負責移動工件的六軸平台，還可能有另外的 XYZ 軸專門移動相機。

這些相機軸不一定包含在 Main Stage 或 Piezo Stage 的六軸數量中。

## 3.4 為什麼 Routine Status 會包含 Camera Coordinates？

現在回頭看你之前貼的文件。

Routine Status Packet 提到：

> camera coordinates

這很可能是用來回報相機相關的位置資料，例如：

* 相機目前的位置。
* 相機移動軸的位置。
* 影像辨識得到的座標。
* 對位參考座標。

但目前不能確認這個欄位究竟是哪一種。

尤其要區分：

| 名稱                | 意義           |
| ----------------- | ------------ |
| Camera Position   | 相機本身的位置      |
| Image Coordinate  | 影像中的像素座標     |
| Object Coordinate | 工件在機器座標系中的位置 |
| Alignment Error   | 工件與目標位置的偏差   |

這些資料的物理意義完全不同。

後續如果手冊有 Camera Coordinates 的欄位定義，我們需要特別確認它是哪一種。

---

# 四、Load Cell 是什麼？重新結合 GMT 的產品來理解

![Image](https://images.openai.com/static-rsc-4/Izw68HosijnE2pIus2xXvb81bMr6QJf6u2uIbHCdU37gmnZTsURHUAeN3UCVfPwHzz9TlFezzvfFvS0xf-quTIwa93BDabPEM1WFKZwFmmc5c2vOIoDHRhf9OPMkiq0__YRuvR_p6IVpTWi_05CwsyM5ar7RrhIz9pZZ6C1LFwbsd2dli_79-aLpGX0nBvIb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XyP_uZWO5_A0KdVPVcWvXbVMtYBJa4xtpyhP3kGDx5B6Lt0HcUfy-qOquJ2RFHHwoJvDJ_7SpJ3avXQso-3Xw5GpY6b5B0y3MwkJ0RPbXEgozSpxTXonDiSJhDHvv6PjhlMqFCCgILGWZZRyniHbLsO1vLSSR2EnAJiTRzr7o_UsYsnWpNy1-_NyC6KN07U2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ibfGnp71xKnCSx-mNZNw5wAMUEJdgvIZ8l2XP0ZgSywRGKAcd0s2i9Y1h2YcMSKNusulJRMVPuPxMTjh_8tLhCtPbljrD-2YBGpdirNco-tWeuU0-Oujzoj7dXHXVVV7AHvOvAH72hiV0Ct0gFN8YKPqbj2Enzg0p00J5pmgt0dn8pLgMOKBGq77lF4Z1lah?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UtDizMitguteODEMOpqVQQmM7TvsFP4p5fajHYrLkoyPWvb7HLHdau4eLP5VWWxjautXxhwIlefAFsZi1jU25n7HfDGBPfR3PSkQnkaxog2QG-RU1IKFkOuGDD5OJEYamwX2lmgEUj0Mhysh7GrywW--jm8cKF1x-rqWwMariTRk8hKrihxA_bi3py0a-v71?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WyPrrdNxGDg4HgssWSGtxMNpqMCgiTziKExhmbjJ41lK6mMqPVu9VW-s2BGkXZXd4NxkaEBGAMeYamq56VUW8f7jmJ5n6YQ_8GhAbsQNdBXJiK45zgURmiV4wPdfWLTUhP56AInXa2aU2yf7kmlwsYguLzUs5uZiudxe2bOVVC4YnKiO6J9h1Th2mXApP93c?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/q6GDLEnhjI9JxX-vM6Bh8V3YlyIV8jRV_TwuNnrYx67y5_N-YBIlKCU8NFoObDG_irARzoNFdyqP-buiclIYgtCJB1AKqGzHtp7k-PfwRpeF09hlyYApjrOW8m0OfteuygwhSW4o5tFLEWhrjkeK-vZnBmlWlQutUrhQXb1bb8Fr9WMK73N2sUuJW4pAFQ3e?purpose=fullsize)

前面我已經解釋過 Load Cell 是荷重元。

這次搜尋 GMT 官方產品後，可以再補充一個更貼近實際設備的用途：

**Load Cell 不只是測量重量，也可以用來監測精密機構在組裝、接觸或壓合時所施加的力量。**

例如，GMT 的設備規格提到：

* Pressure sensing。
* Automatic level adjustment。
* High adjustment。

而 GMT 的設備型錄也提到 Bonding Force Control（接合力量控制）及 Pressure Sensor Monitoring。([高明鐵 GMT][2])

這些功能與 Load Cell 的用途相當吻合。

## 4.1 舉一個實際例子

假設設備要將一個光學元件壓合到指定位置。

如果沒有力量監測，可能發生：

* 壓力太小：元件沒有完全接觸。
* 壓力太大：元件受損。
* 壓力不均：元件傾斜。

這時候 Load Cell 就能測量機構受到的力量。

```text
         六軸平台
             │
             ▼
        接近工件
             │
             ▼
        接觸工件
             │
             ▼
       Load Cell 感測
             │
             ▼
        量測接觸力量
             │
             ▼
       回報控制系統
             │
             ▼
        判斷是否適當
```

因此，Load Cell 的重要性是讓控制系統除了知道「移動到哪裡」，還可能知道「接觸時施加了多大的力量」。

## 4.2 Load Cell 和 Gripper 有什麼不同？

這兩者很容易混淆。

| 比較     | Gripper | Load Cell |
| ------ | ------- | --------- |
| 中文     | 夾爪      | 荷重元       |
| 類型     | 執行器     | 感測器       |
| 工作     | 夾住或釋放工件 | 測量力量      |
| 是否主動動作 | 是       | 否，主要負責感測  |
| 輸出     | 夾持動作    | 量測訊號      |
| 主要目的   | 固定工件    | 監測受力      |

簡單記憶：

**Gripper 負責施加夾持動作，Load Cell 負責測量力量。**

不過，不代表每個 Gripper 都一定搭配 Load Cell。GMT 的電動夾爪也可以利用驅動器設定的扭矩限制控制夾持力量。

---

# 五、PM 是什麼？這次可以更有把握地確認

這部分是我這次搜尋後，最值得修正前一個回答的地方。

之前我說 PM 可能是 Power Meter。

現在從 GMT 官方產品來看，**PM 很可能就是 Power Meter，而且在 GMT 的光通訊設備中，確實有 Laser Power Meter（雷射光功率計）這種設備。**

## 5.1 GMT 官方產品中的 Laser Power Meter

![Image](https://images.openai.com/static-rsc-4/Q9DP_lMSoocj1KK9iMNeLroYBzp0MNKdM5me5GUkPEByWBk0y-TfbiOwosmFBrcs-sAjNLPF7s4e6GaD8PDJcBt85hcba5h6EniGmnQ5C6ecZ4nAyXKp57QAGThGKiH4ww-Swt4KEXFgU0h62vWfnq2tVhchywYv5IWKAmAiCWjLGCRG6BMthMiAeYa7UVVA?purpose=fullsize)

在 GMT 的 Multi-Device Active Aligner 官方規格中，明確列出：

* Laser Power Meter。
* 型號：Santec MPM-220 系列。
* 感測波長範圍：840～1700 nm。
* 量測範圍：-80～+8 dBm（依選配而定）。

([高明鐵 GMT][1])

這代表 GMT 的光通訊設備確實需要量測光功率。

## 5.2 Power Meter 在光學耦合中扮演什麼角色？

這是整個系統非常重要的部分。

假設我們要把光纖對準光學元件。

即使 Camera 看起來已經把兩者對準，也不代表光學耦合效果已經達到最佳。

因為光纖與光學元件之間的光路可能非常敏感。

位置稍微偏差，就可能導致光功率下降。

所以需要 Power Meter 來確認實際的光學效果。

例如：

| 光纖與光學元件的位置 | Power Meter 讀值 |
| ---------- | -------------: |
| 位置 A       |        -20 dBm |
| 位置 B       |        -15 dBm |
| 位置 C       |        -10 dBm |
| 位置 D       |         -8 dBm |

以上數值只是示意。

在這個例子中，D 的光功率最高，表示這個位置的光學耦合效果可能最好。

控制系統就可以利用 Power Meter 的量測結果，不斷微調平台位置，尋找最佳光功率。

這就是：

**Active Alignment（主動式對位）。**

它不只是透過相機讓兩個物體看起來對準，而是實際量測光學輸出，找出光傳輸效果最佳的位置。

## 5.3 Camera 和 Power Meter 的差別

這兩者在主動對位時會互相配合，但目的不同。

| 項目    | Camera  | Power Meter |
| ----- | ------- | ----------- |
| 中文    | 相機      | 功率計         |
| 量測對象  | 影像      | 光功率         |
| 主要用途  | 辨識位置及角度 | 測量光學輸出      |
| 回答的問題 | 工件在哪裡？  | 光耦合效果好不好？   |
| 主要資料  | 影像、座標   | 光功率數值       |

你可以這樣理解：

* Camera 負責找出工件的幾何位置。
* Power Meter 負責確認光學傳輸效果。
* 六軸平台負責調整位置。
* 控制器負責協調這些設備。

這四者組合起來，就能完成精密光學主動對位。

---

# 六、把四種設備放在一起看

現在我們可以建立一個比較完整的概念。

![Image](https://images.openai.com/static-rsc-4/EqqNlqobdNnr_VXbj0qTSvSI1mwin8Gk4cO2mdpaZreFmLCj0CsIF1HIjpoLmqz0PVRoK0piYkrXKReoHM_LjHsAgyCk88QJYI1PzI19Pq14jn1nwdyNphLgV3HHvbzJA-yYib3d2BXiefqEM63hEREsLxySD4B5SA4EFGjO_6OJSkLWe78sIFlif1TXIpMl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KfLkj73ku7xr2s3EXMq4mayFLg5Rx5Eu_7_-JoqNfwk4RsvfYuP5N8LhngD-2J8WDUQToZCOnsG4JBy8BC7qxIoDwVJvbA4zUXkkvbuTmIqwksJl9vK8qCTs7IwTR8rLrnbyprTaK_iDEuQktzc0bAifOySbQcgcSrWDYj6xWQbvQuyuXOyq6qAlXM9ht9Hp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bsgyV2xLQ5pBx8GkabVefY6WJZoAsABTLcRAjzbB1sdcwIEymaXVlDfv5-HjrRSgwnt6OVMNt927JOLiOYJ_f6vclvDvKFhJ4O4UN09vwz6EcoaF_vHXZwyK3D5U8bBDyjOEviX4uJvPOAnUbzomFyegHdiTmoIU8DjG-aRaW9QQIQ2Xd1hSfXDxyIsHvHTE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hY4Cclooy1bPyhtxREmPIguqaU9996aCxyLk1Y1RGWddN29lHMks6kRkteobFWm1JvfojmhVv3y3wxy7IYUscURF7eOaYIJI8W_VAHvfPiRSQ9hBPU-DILZfBl7gRGOebk2gcpp-9VBrulWJRsZ3xXAnWVJdTDpRtnUcafOb9_Eep5T2FpUlQnd88tw8iuWG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GxZoi2ZIUPbIiKVEe2HRNFpRMwMVS7OixDE0I9IDezvdC2QikLOTJnMpqvsMQ-mzIdpHmDO0A_Yx6gSpsengmXD0ZktKjxJg1llERACGVrkbc7rbXu6ckcurat5EZJA2CniTNynOHxmf9VQOkw_fh7VUXBxW-m_AZGZo5LwLkOMgHYhCBdNZzm3V9ea1yk_y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/DSn35H9G0YGg9ZAGqkbOQBYo3dsbfMDVYVHXCLQTy57n73pFEpXbDdFCHm2ot1SLsukkfV-shpgdeSIjRsEvMG-q1bxBkvQqoY_IiatmYz_m4pXYnKvBqVv1ucenKx1kjR7XOOhwLqiHxpMQzP-ZtpHZGIjnFfYl0sYVx9sh8y5M9lLvjyxIKw39JzJsa4lL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wNQJUM6bwxmf4sS-4TPYm5X0ONVU5imLrg6ZfDQVAlRlb2SlsNKrmRQewfzvX4oiKEp3dc-xLaB4cRZ1StIo_fW2L7gqLed--0DEOX04OQh495IX9MfeJ0ZyGnKGcpasvCqT6_-38O0r_uK46bCF8l83luDZU5_df311VdZUAYqtzS2bliRpdAicAwcwuNRi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AIVJ55LF5F-qqCgbiFxs-mH_y0lyaaeT8GnMOYdAiBIYJ6IUT6KoE79j54g1Mb4MoiJpHnDF_pSbZgcDZLfjakI6ppBfOOsuVwpuv8fmLXyLOOCWJPf0k4Ttt-wLifLD-1cT0DTeljvMgGa2JHVnnf2O0o9UFK331kZb9JQat-SV_7M9Yh2TNJy33c0D8Gx_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Yj35Us0o8cYJMdsIKzE9x4Gf4EDPEPmaWFh4A4vqmGM4M-ChN5725He8HtP3nFUPc7ZB1Rx-Mcx8NH6wYBjn0dSe4JrQBQTEv5Gc5-XzNfTZflPY011vxuXMdpU0OaT4e51qIvyGlqGcNCS0dmSK3_RCzaAj6DZDvwv52XPsHgsXksosDgvh_cTaeBwgh_Sf?purpose=fullsize)

假設 GMT 設備要把光纖陣列精確耦合到光學元件。

整個過程可以想像成：

```text
                 GMT Controller
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Gripper       Camera      六軸平台
          │            │            │
          ▼            ▼            ▼
       固定工件     辨識位置      調整位置
          │            │            │
          └────────────┼────────────┘
                       │
                       ▼
                 光學元件耦合
                       │
                       ▼
                 Power Meter
                       │
                       ▼
                  量測光功率
                       │
                       ▼
                判斷耦合效果
                       │
                       ▼
                進一步微調平台
```

而 Load Cell 則可能在工件接觸、壓合或高度調整時提供力量量測。

所以整體可以整理成：

| 設備                       | 在系統中的角色 | 主要回答的問題       |
| ------------------------ | ------- | ------------- |
| Main Stage / Piezo Stage | 運動平台    | 要移動到哪裡？       |
| Gripper                  | 夾持執行器   | 工件如何固定？       |
| Camera                   | 視覺感測器   | 工件實際位置在哪裡？    |
| Load Cell                | 力量感測器   | 接觸或壓合時施加多少力量？ |
| PM / Power Meter         | 光功率感測器  | 光學耦合效果如何？     |

這就是我認為你需要建立的整體觀念。

---

# 七、修正之前對 Routine Status Packet 的理解

現在有了 GMT 官方產品資訊，我們可以重新檢視你之前貼的手冊。

手冊說 Routine Status Packet 會包含：

* Controller Status
* I/O Signals
* Axis Positions
* Gripper States
* Camera Coordinates

另外，手冊的 Analog Input Mapping 提到：

| AI Channel | 功能        |
| ---------- | --------- |
| AI00       | Load Cell |
| AI01       | Reserved  |
| AI02       | I2C0      |
| AI03       | I2C3      |
| AI04       | PM Ch0    |
| AI05       | PM Ch1    |
| AI06       | PM Ch2    |
| AI07       | PM Ch3    |

現在我們可以做出更有根據的解讀：

### 1. AI00：Load Cell

很可能是力量感測器的類比訊號輸入。

控制器取得電壓後，依照感測器校正資料，才能換算成實際力量。

### 2. AI04～AI07：PM Ch0～Ch3

結合 GMT 官方光通訊設備使用 Laser Power Meter 的資訊，PM 很可能是 Power Meter。

這四個通道可能對應四組功率量測訊號。

但要特別注意：官方產品頁證明 GMT 有使用 Laser Power Meter，並不能證明你手上的控制器一定連接四個 Santec 光功率計。

另外，官方產品規格也分別列出了 Power Meter 和 Laser Power Meter，前者的規格欄位包含電壓、電流，後者則是光功率感測器。因此，手冊的 PM Ch0～Ch3 是否全部指光功率計，仍然需要同事確認。

### 3. Gripper States

這是夾爪的狀態回報，不是力量感測數值。

例如開啟、閉合、夾持完成等狀態。

### 4. Camera Coordinates

這是相機相關的座標資訊。

但目前仍不清楚是相機軸位置、影像辨識座標，還是工件對位座標。

---

# 八、這裡還有一個重要的文件問題：68 Bytes 的封包容量

我想提醒你一件事。

你之前提供的 Routine Status Packet Table 1：

| 欄位                   | 資料型別       |  Bytes |
| -------------------- | ---------- | -----: |
| Controller Status    | UINT32     |      4 |
| Analog Input         | UINT16 × 8 |     16 |
| Main Stage Positions | DOUBLE × 6 |     48 |
| **Total**            |            | **68** |

但手冊其他段落又提到：

* Gripper States
* Camera Coordinates
* Digital Input

問題來了：

**這些資料究竟放在哪裡？**

因為目前 Table 1 已經用完 68 Bytes。

這裡可能有幾種情況：

1. Gripper States 和 Camera Coordinates 是另一種封包。
2. 這些資訊包含在其他命令的回應中。
3. Table 1 只是部分資料欄位，沒有列出完整內容。
4. 文件不同章節使用了不同版本的封包格式。

這個問題仍然沒有解決。

而且我們之前也發現 2.2 的六個 DOUBLE 與 2.6 的十二個 INT32 都剛好占用 48 Bytes，但代表完全不同的資料。

所以目前不能直接認定 68 Bytes 就是所有 Routine Status 資訊的完整內容。

---

# 九、這次搜尋後，我對整份文件的理解更新

我把目前已知的資訊整理成一張表：

| 名稱             | 中文      | 功能             | 目前理解程度        |
| -------------- | ------- | -------------- | ------------- |
| Main Stage     | 主運動平台   | 主要機構位置控制       | 已有基本理解        |
| Piezo Stage    | 壓電平台    | 高精度微小位移        | 已有基本理解        |
| Gripper        | 夾爪      | 夾持及釋放工件        | 已有官方產品佐證      |
| Camera         | 相機      | 影像辨識及視覺對位      | 已有官方產品佐證      |
| Load Cell      | 荷重元     | 接觸力量或負載量測      | 已有技術原理及設備用途佐證 |
| PM             | 很可能是功率計 | 可能是光功率量測       | 有官方光功率計產品佐證   |
| EtherCAT       | 工業即時通訊  | 控制器與設備間的即時資料交換 | 已有基本理解        |
| PDO            | 製程資料物件  | 週期性交換控制及狀態資料   | 已有基本理解        |
| Routine Status | 例行狀態資料  | 定期回報控制器及設備狀態   | 封包內容仍有矛盾      |

## 十、我建議接下來向同事確認的問題

經過這次搜尋，我認為有幾個問題值得確認：

1. **PM Ch0～Ch3 是否代表四組 Laser Power Meter 的類比輸出？**
2. **Load Cell 是否用於平台接觸力量、壓合力量或高度調整的力量監測？**
3. **Camera Coordinates 是相機 XYZ 軸位置，還是影像辨識後的工件座標？**
4. **Gripper States 是否包含在 Routine Status Packet？**
5. **Routine Status Packet 的 68 Bytes 是否為完整封包？**

---

## 最後，我想讓你記住一個核心觀念

從 GMT 的產品來看，這套系統的工作不只是「讓馬達移動」。

它真正要完成的是一個精密自動化流程：

* **Gripper：** 把工件固定好。
* **Camera：** 找出工件的正確位置。
* **Main Stage / Piezo Stage：** 精確調整位置。
* **Load Cell：** 監測接觸或壓合力量。
* **Power Meter：** 確認光學耦合效果。
* **GMT Controller：** 協調所有設備並回報狀態。

我認為這是理解你同事所寫的 SixAxes Controller 程式最重要的基礎。

另外，這次查到的 GMT 官方產品頁也很值得保留：

* [GMT GLOBAL 官方網站](https://www.gmtglobalinc.com/?utm_source=chatgpt.com)
* [GMT Multi-Device Active Aligner](https://www.gmtglobalinc.com/en/equipment/multi-device-active-aligner?utm_source=chatgpt.com)
* [GMT Rx FAU Aligner](https://www.gmtglobalinc.com/en/equipment/rx-fau-aligner?utm_source=chatgpt.com)
* [GMT Tx FAU Aligner](https://www.gmtglobalinc.com/en/equipment/tx-fau-aligner?utm_source=chatgpt.com)
* [GMT Electric Grippers](https://www.gmtglobalinc.com/en/electric-gripper?utm_source=chatgpt.com)
* [GMT Hexapod Platform](https://www.gmtglobalinc.com/upload-files/catalogue/display-information/EN/GMT-Hexapod_Platform_GSS-100-d_-EN_P.pdf?utm_source=chatgpt.com)

後續我們分析命令時，我會把這些設備角色一起考慮，避免只從命令字串或 C++ 程式碼的角度理解，而忽略了命令實際控制的是哪一種機構或感測器。

[1]: https://www.gmtglobalinc.com/en/equipment/multi-device-active-aligner "Multi-Device Active Aligner | GMT GLOBAL"
[2]: https://www.gmtglobalinc.com/upload-files/catalogue/display-information/EN/Equipment_and_Wafer_System-DM-EN_P.pdf?utm_source=chatgpt.com "Equipment、Wafer System-DM-EN_P-20260204"


---

PDO 是 EtherCAT 裡面非常重要的名詞。你在 SixAxes Controller 文件、伺服驅動器規格，以及 STM32 EtherCAT 程式中經常看到它，是因為：

**PDO 是 EtherCAT 主站與從站在正常運轉時，交換即時控制資料的主要方式。**

例如：

* STM32 / GMT Controller 告訴伺服驅動器：「目標位置是多少？」
* 伺服驅動器回報：「我目前的位置是多少？」
* STM32 / GMT Controller 告訴驅動器：「目標速度是多少？」
* 伺服驅動器回報：「目前速度是多少？有沒有發生錯誤？」

這些資料通常透過 PDO 週期性地交換。

我會先解釋 PDO，再把它與其他相關名詞連起來，最後用 GMT 官方的伺服驅動器產品來對照。

# 一、PDO 是什麼？

PDO 的完整英文名稱是：

**Process Data Object（過程資料物件）**

先拆開理解：

* Process：設備目前正在執行的工作或製程。
* Data：設備運作時需要交換的資料。
* Object：按照特定規則組織起來的資料項目。

所以 PDO 可以先理解成：

**把設備運作時需要即時交換的資料，按照預先設定的格式組合起來，透過 EtherCAT 傳送。**

例如，假設 GMT Controller 要控制一個伺服馬達：

```text
GMT Controller（EtherCAT Master）
              │
              │ PDO
              │
              ▼
        Servo Drive（Slave）
              │
              ▼
            Motor
```

Controller 可能需要傳送：

| 資料              | 意義          |
| --------------- | ----------- |
| Controlword     | 控制命令，例如伺服啟動 |
| Target Position | 目標位置        |
| Target Velocity | 目標速度        |
| Target Torque   | 目標扭矩        |

伺服驅動器則可能回傳：

| 資料              | 意義      |
| --------------- | ------- |
| Statusword      | 伺服驅動器狀態 |
| Actual Position | 馬達目前位置  |
| Actual Velocity | 馬達目前速度  |
| Actual Torque   | 馬達目前扭矩  |

這些資料可以組成 PDO，按照設定的週期不斷交換。

例如：

```text
第 1 次交換：
Controller → Target Position = 1000
Drive      → Actual Position = 950

第 2 次交換：
Controller → Target Position = 1000
Drive      → Actual Position = 970

第 3 次交換：
Controller → Target Position = 1000
Drive      → Actual Position = 990

第 4 次交換：
Controller → Target Position = 1000
Drive      → Actual Position = 1000
```

這只是用來說明資料交換的示意，實際數值和回報內容取決於驅動器設定。

你可以看到，Controller 不需要每次都另外發送一個 TCP 命令詢問馬達位置，而是可以透過 PDO 持續取得回報。

這就是 PDO 的重要性。

---

# 二、PDO 有兩種方向：RxPDO 和 TxPDO

這是剛開始學 EtherCAT 最容易混淆的地方。

PDO 分成兩種：

1. RxPDO
2. TxPDO

最重要的觀念是：

**Rx 和 Tx 的方向是以 EtherCAT Slave（從站）的角度來命名，而不是以 Master（主站）的角度。**

## 2.1 RxPDO

RxPDO = Receive PDO。

意思是從站接收的 PDO。

因此：

**Master → Slave**

例如 GMT Controller 要求伺服驅動器移動到 1000：

```text
GMT Controller                 Servo Drive
  (Master)                       (Slave)
     │                              │
     │                              │
     │──── Target Position = 1000 ─>│
     │                              │
     │──── Controlword ────────────>│
     │                              │
     │                              ▼
     │                         執行馬達控制
```

對伺服驅動器來說，這些資料是它接收的資料，所以稱為 RxPDO。

常見的 RxPDO 資料：

| 資料              | 意義   |
| --------------- | ---- |
| Controlword     | 控制字  |
| Target Position | 目標位置 |
| Target Velocity | 目標速度 |
| Target Torque   | 目標扭矩 |

## 2.2 TxPDO

TxPDO = Transmit PDO。

意思是從站傳送出去的 PDO。

因此：

**Slave → Master**

例如伺服驅動器回報目前位置：

```text
GMT Controller                 Servo Drive
  (Master)                       (Slave)
     │                              │
     │                              │
     │<──── Actual Position = 950 ──│
     │                              │
     │<──── Statusword ─────────────│
     │                              │
     ▼                              │
  取得馬達位置                      │
  判斷驅動器狀態                    │
```

對伺服驅動器來說，這些資料是它傳送出去的資料，所以稱為 TxPDO。

常見的 TxPDO 資料：

| 資料              | 意義   |
| --------------- | ---- |
| Statusword      | 狀態字  |
| Actual Position | 實際位置 |
| Actual Velocity | 實際速度 |
| Actual Torque   | 實際扭矩 |

### 記憶方式

只要記住這張表就可以：

| 名稱    | 從站的角度       | 傳輸方向           |
| ----- | ----------- | -------------- |
| RxPDO | Receive，接收  | Master → Slave |
| TxPDO | Transmit，傳送 | Slave → Master |

這個方向觀念非常重要，後面看到 EtherCAT 程式碼時會一直用到。

---

# 三、PDO、SDO、CoE 是什麼關係？

你可能也常常看到 SDO、CoE。

這幾個名詞是 EtherCAT 裡面互相關聯的概念。

先看這張圖：

```text
                  EtherCAT
                      │
                      ▼
                     CoE
       CAN application protocol over EtherCAT
                      │
              ┌───────┴───────┐
              │               │
              ▼               ▼
            PDO             SDO
              │               │
              ▼               ▼
         週期性資料         非週期性資料
         即時控制           參數讀寫
```

這裡的 CoE 是 EtherCAT 的應用層協定之一。

全名是：

**CAN application protocol over EtherCAT**

它讓 EtherCAT 可以使用與 CANopen 相似的物件字典、PDO、SDO 等機制。

[EtherCAT Technology Group（CoE 技術說明）](https://www.ethercat.org/en/technology.html?trk=public_post_comment-text&utm_source=chatgpt.com)

## 3.1 SDO 是什麼？

SDO = Service Data Object。

中文可以理解為服務資料物件。

它主要用於讀取或修改設備的參數。

例如：

假設你有一個伺服驅動器，想知道：

* 馬達最大速度設定是多少？
* 電流限制是多少？
* 加速度設定是多少？
* 目前使用哪一種運動模式？
* PDO 的資料映射設定是什麼？

這些資料就可能透過 SDO 讀取或修改。

例如：

```text
STM32 EtherCAT Master
          │
          │ SDO Read
          ▼
      Servo Drive
          │
          ▼
   回傳參數設定值
```

SDO 比較像是你向設備詢問某個參數，或要求設備修改某個設定。

例如：

```text
讀取馬達最大速度
設定加速度
設定運動模式
讀取錯誤代碼
```

這些通常不是每 1ms 都需要執行的操作。

所以 SDO 通常用於設備初始化、參數設定、診斷及非週期性資料存取。

## 3.2 PDO 和 SDO 最大的差別

| 比較   | PDO                   | SDO                 |
| ---- | --------------------- | ------------------- |
| 全名   | Process Data Object   | Service Data Object |
| 主要用途 | 即時控制及狀態交換             | 參數讀取與設定             |
| 傳輸特性 | 週期性                   | 通常非週期性              |
| 資料內容 | 預先配置的控制與狀態資料          | 透過索引存取物件字典          |
| 常見用途 | 目標位置、實際位置             | 加速度、速度限制、模式設定       |
| 通訊方式 | EtherCAT Process Data | CoE Mailbox         |

舉個實際例子：

你想讓馬達移動到 100 mm。

大致可以分成兩個階段：

**第一階段：設定馬達參數**

例如：

* 設定運動模式。
* 設定最大速度。
* 設定加速度。

這些可以使用 SDO。

**第二階段：開始進行即時運動控制**

例如：

* 每個週期傳送新的目標位置。
* 每個週期接收實際位置。
* 每個週期接收驅動器狀態。

這些通常使用 PDO。

因此，可以簡單記成：

> SDO 負責設定設備，PDO 負責設備運作時的即時資料交換。

這是入門時很好用的理解方式，但不是絕對限制：SDO 也可以在運轉期間使用，而 PDO 也不一定只傳送馬達控制資料。

---

# 四、Object Dictionary（物件字典）又是什麼？

這個名詞也非常重要。

PDO 和 SDO 都會接觸到 Object Dictionary（OD）。

你可以把 Object Dictionary 想像成：

**每一個 EtherCAT 設備內部的一本資料字典，裡面記錄設備有哪些參數、狀態及控制資料。**

例如伺服驅動器可能有：

| Index  | 名稱                    | 意義   |
| ------ | --------------------- | ---- |
| 0x6040 | Controlword           | 控制字  |
| 0x6041 | Statusword            | 狀態字  |
| 0x6060 | Modes of operation    | 運動模式 |
| 0x6064 | Position actual value | 實際位置 |
| 0x607A | Target position       | 目標位置 |
| 0x60FF | Target velocity       | 目標速度 |

這些 Index 就像資料的編號。

例如：

```text
0x6040 = Controlword
0x6041 = Statusword
0x6064 = Position actual value
```

當你看到程式碼：

```cpp
0x607A
```

就應該想到：

這很可能是 CiA 402 伺服驅動器的 Target Position 物件。

注意，這些是常見的 CiA 402 物件，不代表你的 GMT 系統一定全部都使用相同的 PDO 映射。

EtherCAT Technology Group 對 Object Dictionary 的解釋，也有 Index、Subindex、資料型別及存取權限等定義。([prelive.ethercat.org][1])

## 4.1 Index 和 Subindex 是什麼？

Object Dictionary 的資料除了 Index，還可能有 Subindex。

例如：

```text
Index:    0x6040
Subindex: 0x00
```

你可以把它想成：

* Index：資料項目的編號。
* Subindex：該資料項目內的子項目編號。

對於初學者，先記住：

**Index + Subindex 可以定位 EtherCAT 設備中的一個資料項目。**

---

# 五、PDO Mapping（PDO 映射）又是什麼？

這是另一個非常重要的名詞。

前面提到，伺服驅動器的 Object Dictionary 裡面有很多資料。

但是，不是所有資料都需要每個 EtherCAT 週期傳輸。

例如：

```text
Servo Drive Object Dictionary
│
├── 0x6040 Controlword
├── 0x6041 Statusword
├── 0x6060 Operation Mode
├── 0x6064 Actual Position
├── 0x607A Target Position
├── 0x60FF Target Velocity
├── 其他參數
└── 其他設定
```

如果每次都傳送所有資料，會造成不必要的資料量。

所以我們可以挑選真正需要週期性傳輸的資料，組合成 PDO。

這個動作就叫：

**PDO Mapping（PDO 映射）。**

例如：

### RxPDO Mapping

選擇 Master 要傳送給 Slave 的資料：

| 順序 | Index  | 資料              |      長度 |
| -- | ------ | --------------- | ------: |
| 1  | 0x6040 | Controlword     | 16 bits |
| 2  | 0x607A | Target Position | 32 bits |
| 3  | 0x6060 | Operation Mode  |  8 bits |

總長度：

16 + 32 + 8 = 56 bits。

### TxPDO Mapping

選擇 Slave 要回傳給 Master 的資料：

| 順序 | Index  | 資料              |      長度 |
| -- | ------ | --------------- | ------: |
| 1  | 0x6041 | Statusword      | 16 bits |
| 2  | 0x6064 | Actual Position | 32 bits |
| 3  | 0x606C | Actual Velocity | 32 bits |

總長度：

16 + 32 + 32 = 80 bits。

上面的內容只是示意，實際 PDO Mapping 必須依照伺服驅動器支援的資料項目及設定決定。

---

# 六、GMT 官方產品如何使用 PDO？

這部分我特別查了 GMT 官方網站上的伺服驅動器資料。

## 6.1 GMT R-SERVO 伺服驅動器

![Image](https://images.openai.com/static-rsc-4/Y0Kqozc0J4rFfPO88n84i8KhBNiWP52Aw4uZQRxuA_gtUpGAX5Rfne51eoU78pYJ55slZTxwa39-dcsdK9QNKXTU1fSupyMSTfDbzO2AoPFc3ZuLnQQOcUY1AoI_MQIzrJNUuwd_laOww78ACx-17jQRncK_JZkppVv1VIo_fV3BseUZwsB3RSRAPvDmfesN?purpose=fullsize)

官方產品頁：

[GMT R-SERVO 伺服驅動器](https://www.gmtglobalinc.com/%E4%BC%BA%E6%9C%8D%E9%A9%85%E5%8B%95%E5%99%A8/R-SERVO?utm_source=chatgpt.com)

官方資料提到 R-SERVO 支援：

* EtherCAT
* CANopen
* Modbus
* Pulse
* Analog Control

而且支援：

* CiA 402 Drive Profile
* Profile Position Mode
* Profile Velocity Mode
* Cyclic Synchronous Position Mode
* Cyclic Synchronous Velocity Mode
* Homing Mode

其中有一個很值得注意的規格：

**Dynamic PDO Mapping。**

這代表 PDO 的資料映射可以依照驅動器支援的功能進行配置，而不是所有情況都只能使用固定資料組合。

## 6.2 GMT P-SERVO EC 步進驅動器

![Image](https://images.openai.com/static-rsc-4/Y0Kqozc0J4rFfPO88n84i8KhBNiWP52Aw4uZQRxuA_gtUpGAX5Rfne51eoU78pYJ55slZTxwa39-dcsdK9QNKXTU1fSupyMSTfDbzO2AoPFc3ZuLnQQOcUY1AoI_MQIzrJNUuwd_laOww78ACx-17jQRncK_JZkppVv1VIo_fV3BseUZwsB3RSRAPvDmfesN?purpose=fullsize)

官方產品頁：

[GMT P-SERVO EC](https://www.gmtglobalinc.com/%E6%AD%A5%E9%80%B2%E9%A9%85%E5%8B%95%E5%99%A8/p-servo-ec?utm_source=chatgpt.com)

官方資料提到：

* 支援 EtherCAT。
* 支援 CoE。
* 符合 CiA 402 標準。
* 支援位置、速度、同步位置、同步速度及原點復歸模式。
* 最小同步時間間隔 500 μs。

這裡可以看出 PDO 與運動控制的關係。

例如：

假設使用 Cyclic Synchronous Position Mode（CSP，循環同步位置模式）。

Controller 會週期性地透過 PDO 傳送目標位置。

伺服驅動器則週期性回傳實際位置及狀態。

這種方式很適合需要精確同步控制的多軸運動設備。

---

# 七、CSP、CSV、PP 等又是什麼？

這些也是你在 EtherCAT 文件中很可能遇到的名詞。

它們是 CiA 402 定義的伺服運動模式。

先看表格：

| 縮寫     | 英文                          | 中文概念     | 控制方式              |
| ------ | --------------------------- | -------- | ----------------- |
| PP     | Profile Position            | 位置模式     | 設定目標位置，由驅動器執行定位曲線 |
| PV     | Profile Velocity            | 速度模式     | 設定目標速度            |
| CSP    | Cyclic Synchronous Position | 循環同步位置模式 | 週期性傳送目標位置         |
| CSV    | Cyclic Synchronous Velocity | 循環同步速度模式 | 週期性傳送目標速度         |
| Homing | Homing Mode                 | 原點復歸模式   | 尋找機械原點或參考位置       |

## 7.1 PP：Profile Position

假設你要馬達移動到 100 mm。

Controller 設定：

```text
Target Position = 100 mm
Velocity = 10 mm/s
Acceleration = 設定值
```

驅動器依照自己的運動曲線完成定位。

這種模式下，運動軌跡主要由驅動器內部的 Profile 設定決定。

## 7.2 CSP：Cyclic Synchronous Position

這個模式與 PDO 的關係非常密切。

假設 Controller 每 1ms 更新一次目標位置：

| 時間  | Target Position |
| --- | --------------: |
| 0ms |               0 |
| 1ms |             0.1 |
| 2ms |             0.2 |
| 3ms |             0.3 |
| 4ms |             0.4 |
| 5ms |             0.5 |

Controller 每個週期透過 PDO 傳送新的目標位置。

伺服驅動器同步接收並執行。

這樣 Controller 就能更直接地掌握多軸運動軌跡。

這也是精密多軸控制很常使用的模式之一。

---

# 八、EtherCAT 的其他相關名詞

現在把常見名詞整理起來。

| 名詞          | 全名                                         | 用途                        |
| ----------- | ------------------------------------------ | ------------------------- |
| EtherCAT    | Ethernet for Control Automation Technology | 工業即時通訊技術                  |
| CoE         | CAN application protocol over EtherCAT     | EtherCAT 上的 CANopen 應用層協定 |
| PDO         | Process Data Object                        | 週期性即時資料交換                 |
| RxPDO       | Receive PDO                                | 從站接收的資料                   |
| TxPDO       | Transmit PDO                               | 從站傳送的資料                   |
| SDO         | Service Data Object                        | 參數讀取及設定                   |
| OD          | Object Dictionary                          | 設備參數與資料字典                 |
| PDO Mapping | PDO 映射                                     | 決定 PDO 包含哪些資料             |
| ESI         | EtherCAT Slave Information                 | 從站設備描述 XML 檔              |
| SM          | SyncManager                                | 管理從站資料區域及通訊               |
| DC          | Distributed Clocks                         | 分散式時鐘同步                   |
| CiA 402     | Drive Profile                              | 伺服驅動器標準設備規範               |

其中有三個我希望你特別認識。

## 8.1 ESI：EtherCAT Slave Information

ESI 是一個 XML 格式的設備描述檔。

它通常由設備製造商提供。

裡面描述：

* 設備名稱。
* Vendor ID。
* Product Code。
* 支援的 PDO。
* PDO Mapping。
* 支援的運動模式。
* 通訊設定資訊。

例如，你把 GMT 的 EtherCAT 伺服驅動器接到 STM32 Controller，主站必須知道這個設備支援什麼資料及功能。

ESI 就是重要的設備描述來源之一。

可以把它理解成：

**ESI 是告訴 EtherCAT Master：這個 Slave 是什麼設備，以及它支援哪些功能。**

不過，ESI 是設備能力的描述，實際運行時的 PDO 配置仍可能需要由主站設定。

## 8.2 SM：SyncManager

SyncManager 是 EtherCAT 從站內部用來管理資料交換的機制。

你可以先把它理解成：

**負責管理 Master 和 Slave 之間資料區域及資料交換的硬體／協定機制。**

它會管理從站的 Process Data 區域及 Mailbox 通訊區域。

例如：

```text
EtherCAT Slave
│
├── SyncManager
│      │
│      ├── PDO Output Data
│      │      Master → Slave
│      │
│      └── PDO Input Data
│             Slave → Master
│
└── Mailbox
       │
       └── CoE / SDO
```

這是概念示意，實際 SyncManager 通道配置依照從站硬體及 ESI 而定。

## 8.3 DC：Distributed Clocks

DC 是 Distributed Clocks（分散式時鐘）。

它的目的在於讓 EtherCAT 網路上的多個設備能夠有精確、同步的時間基準。

例如：

假設你有六個伺服軸。

如果六個軸各自按照不同時間執行，可能產生同步誤差。

DC 可以讓多個從站按照共同的時間基準同步執行。

```text
             EtherCAT Master
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
       Axis 1     Axis 2     Axis 3
         │          │          │
         ▼          ▼          ▼
       Sync       Sync       Sync
         │          │          │
         └──────────┴──────────┘
                    │
              同步執行
```

這對多軸運動控制非常重要。

特別是使用 CSP、CSV 等週期性同步模式時，DC 可以協助提高多軸控制的時間一致性。

---

# 九、把所有名詞串成一個實際運作流程

假設你有一個 GMT 六軸平台，使用 EtherCAT 控制六個伺服驅動器。

整個系統可以分成以下階段。

### 階段一：設備初始化

```text
STM32 EtherCAT Master
          │
          ▼
      搜尋 Slave
          │
          ▼
      讀取 ESI
          │
          ▼
      設定 PDO Mapping
          │
          ▼
      SDO 設定伺服參數
          │
          ▼
      EtherCAT 進入 OP
```

### 階段二：進入正常運作

假設 EtherCAT 週期為 1ms。

```text
                STM32 Master
                     │
             ┌───────┴───────┐
             │               │
             ▼               ▲
          RxPDO             TxPDO
             │               │
             ▼               │
         Servo Drive  ────────┘
             │
             ▼
           Motor
```

每個週期：

1. STM32 傳送目標位置。
2. STM32 傳送控制字。
3. Servo Drive 接收資料。
4. Servo Drive 控制馬達。
5. Servo Drive 回傳實際位置。
6. Servo Drive 回傳狀態字。
7. STM32 讀取回傳資料。
8. 進入下一個週期。

這就是 PDO 在整個系統中的角色。

---

# 十、回到你正在研究的 SixAxes Controller

現在我們把這些概念與你之前貼的文件連起來。

文件提到：

> Central Data Handler coordinates field updates through cyclic PDO exchange.

這句話很重要。

它表示 GMT Controller 裡面有一個 Central Data Handler，透過週期性 PDO 交換協調各個 Field Device 的資料。

我會這樣理解它：

```text
                  Host PC / Qt
                       │
                       │ TCP/IP
                       ▼
               GMT SixAxes Controller
                       │
                Central Data Handler
                       │
                 EtherCAT Master
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Servo 1      Servo 2       Servo 3
          │            │            │
          ▼            ▼            ▼
       Servo 4      Servo 5       Servo 6
```

Controller 可能透過 PDO：

* 傳送各軸的目標位置。
* 接收各軸的實際位置。
* 接收伺服驅動器狀態。
* 傳送控制命令。
* 交換 I/O 狀態。

這樣就能持續掌握整個運動平台的狀態。

不過，手冊目前沒有揭露 SixAxes Controller 的實際 PDO Mapping，所以我們還不能確定每個 PDO 的資料欄位及長度。

另外，你之前手冊的 Routine Status Packet 是每 200ms 傳送一次，而 EtherCAT PDO 是週期性交換資料。

**這兩個週期不能混為一談。**

例如：

```text
EtherCAT PDO：每 1ms 交換一次（示意）
                    │
                    ▼
             Controller 取得資料
                    │
                    ▼
       Routine Status 每 200ms 回報一次
                    │
                    ▼
                 Qt Client
```

這表示 Qt Client 不一定能看到 EtherCAT 每個 1ms 的資料變化。它可能只收到控制器整理後、每 200ms 更新的狀態摘要。

---

# 十一、這些名詞與你目前 STM32H755 專案的關係

這裡我想特別區分兩件事情。

你目前已經完成的 STM32H755 EtherCAT 軌跡資料整合，使用的是：

```text
EtherCAT
   │
   ▼
CM7
   │
   ▼
SPI2
   │
   ▼
Raspberry Pi CM5
```

CM7 取得 EtherCAT 的軌跡資料，再透過 SPI2 傳送 72 Bytes 封包到 CM5。

這個架構中：

* EtherCAT PDO 是 STM32 與 EtherCAT 從站之間的資料交換方式。
* SPI2 是 STM32 與 CM5 之間的另一種通訊介面。
* 72 Bytes SPI 封包是你們自行定義的資料格式。

所以：

**PDO 不等於 SPI 封包，也不等於你們的 72 Bytes 軌跡資料封包。**

但你可以把它們理解成不同層次的資料傳輸：

| 通訊           | 連接對象                                 | 主要用途         |
| ------------ | ------------------------------------ | ------------ |
| EtherCAT PDO | STM32 / GMT Master ↔ EtherCAT Slaves | 週期性控制及回報設備資料 |
| TCP/IP       | GMT Controller ↔ Host PC             | 命令與狀態通訊      |
| SPI2         | STM32 CM7 ↔ Raspberry Pi CM5         | 傳輸軌跡資料       |
| UART         | STM32 ↔ CM5（依目前 Status Stream 架構）    | 傳輸狀態封包       |

這些通訊各自有不同的封包格式及運作機制，不能混在一起理解。

---

# 十二、建議你先記住這六個名詞

如果你目前是初學者，不需要一次把所有 EtherCAT 細節都記住。

先掌握以下六個就足夠建立基礎：

| 名詞                | 你可以先這樣理解          |
| ----------------- | ----------------- |
| PDO               | 正常運作時，週期性交換的即時資料  |
| RxPDO             | 從站接收 Master 傳來的資料 |
| TxPDO             | 從站傳送資料給 Master    |
| SDO               | 讀取或修改設備參數         |
| Object Dictionary | 設備內部的資料字典         |
| PDO Mapping       | 決定哪些資料要放進 PDO     |

進一步了解時，再學 ESI、SM、DC 和 CiA 402。

## 官方技術資料，建議保留

以下是這次查到、值得你日後閱讀的資料：

1. [EtherCAT Technology Group－PDO Mapping and Configuration](https://prelive.ethercat.org/cn/technology/%E7%9F%A5%E8%AF%86%E5%BA%93/pdo-mapping-and-configuration/?utm_source=chatgpt.com)：直接說明 PDO 如何選擇資料、設定映射，以及啟用哪些 PDO。
2. [EtherCAT Technology Group－SDO](https://prelive.ethercat.org/de/technology/knowledge-base/sdo/?utm_source=chatgpt.com)：說明 SDO 如何讀取及寫入 Object Dictionary。
3. [EtherCAT Technology Group－Object Dictionary](https://prelive.ethercat.org/de/technology/knowledge-base/object/?utm_source=chatgpt.com)：說明 Index、Subindex 與資料項目的結構。
4. [GMT R-SERVO 官方產品頁](https://www.gmtglobalinc.com/%E4%BC%BA%E6%9C%8D%E9%A9%85%E5%8B%95%E5%99%A8/R-SERVO?utm_source=chatgpt.com)：了解 GMT 伺服驅動器支援的 EtherCAT、CiA 402 及運動模式。
5. [GMT R-SERVO 官方手冊](https://www.gmtglobalinc.com/upload-files/catalogue/electrical-controls/R-SERVO_Single-Axis_Series-Servo_Driver-Manual-TW-P.pdf?utm_source=chatgpt.com)：裡面有 CANopen、EtherCAT、Object Dictionary、PDO 和 SDO 的介紹。
6. [GMT P-SERVO EC 官方產品頁](https://www.gmtglobalinc.com/%E6%AD%A5%E9%80%B2%E9%A9%85%E5%8B%95%E5%99%A8/p-servo-ec?utm_source=chatgpt.com)：了解 GMT 步進驅動器的 CoE、CiA 402 及同步運動控制功能。

---

**我建議下一個值得深入理解的主題是 CiA 402。**

因為你已經理解 PDO 和 SDO 之後，下一個問題自然就是：

「PDO 裡面傳送的 Controlword、Statusword、Target Position、Actual Position，到底是按照什麼標準定義的？」

這就會進入 CiA 402。

它是你理解 GMT 六軸伺服控制程式非常重要的一塊拼圖。

[1]: https://prelive.ethercat.org/de/technology/knowledge-base/object/?utm_source=chatgpt.com "Object: ETG"


---

```
CoE 提到的CAN是什麼？PDO Mapping  我還是不懂，可以使用生活中簡單的易懂的例子說明嗎？
```

這次我想換一種教法：先不講太多專業名詞，而是用「汽車、道路、便當盒」這些生活中的例子來理解。

你問的兩個問題其實有關聯：

1. **CAN 是什麼？為什麼 CoE 裡面會出現 CAN？**
2. **PDO Mapping 到底在做什麼？為什麼需要 Mapping？**

先說結論：

* CAN 是一種工業設備常用的通訊技術。
* CANopen 是建立在 CAN 上的一套通訊規則。
* CoE 是把 CANopen 的許多通訊規則帶到 EtherCAT 上使用。
* PDO Mapping 是決定每次交換資料時，要放哪些資料、順序如何、各占多少空間。

我們一步一步來。

# 一、CAN 是什麼？

CAN 的全名是：

**Controller Area Network（控制器區域網路）**

它是一種讓多個電子控制設備互相交換資料的通訊技術。

這種技術最早由 Bosch 開發，最常見的應用之一就是汽車。

## 1.1 用汽車來理解 CAN

想像一台汽車裡面有很多電子控制設備：

* 引擎控制器
* ABS 煞車控制器
* 安全氣囊控制器
* 儀表板
* 車門控制器
* 方向盤控制器

這些設備都需要互相交換資訊。

例如：

ABS 控制器需要知道車輪目前的轉速，引擎控制器可能需要知道車輛目前的運作狀態，儀表板則需要顯示車速。

如果每個設備都必須直接拉一條通訊線給其他每個設備，線路會非常複雜。

因此，CAN 採用共用通訊匯流排的方式。

![Image](https://images.openai.com/static-rsc-4/RfiOlKFiVxnH9AjunrAVp8VsPZP_9OF7JjxX70Dze8OjZpLVXgeMxxB8SnRzdeW5N0qwzBXx8dpxGYhN4RcbVfMnY3akoOTiyN5I7Xu1YcZwR_VGOvdjRxAq9xr-albiV4CNaF4SqdaF9RNp9Uhai21wwlBeHUqyrGt5iCTOfH8vOAQqqhlhDtKz2pEnchhA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xiv5oqh8UAkN21oJb4dG96z4jw3Q7EfwO3weEP5gH35wUqAO9qE4bBiVfmvlWcWHrfmfgAYPVOQdAje-wOtpjLoKjbPYWInxX4mDTIa9nPcrtKzalH3btasVgpqcvKZRXW08VMMe83cDN59IC3T7ak8lb-0PCgdkuJUXGrEYL0yQG-_QIISxObEH3cn78jB2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QEtQfkOwRAd-pczl9fAlk0KJJui4Kaj5JrTGj3JzBG20oKnRmJ8MXYuApVwU7IESz5Y3azxYoi40t7rSuR81lNzOEZmYfTFTAk48mS5BPFDevhwCy-uULFbBcTnrxg3RwLTymM0HZCiu_p2HQGWANuVfEwT8FvFD1T9hB2ZWPgADAv8PPcLg_dFtBdEWMYBP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qTIjyqcdfZ7HCcB0dfHl2M0ix7YKzTOjS9o1ldciiXnfnI0E76TUsyQ6Z4xxFO1vJe2cLotji6Gy7JysujIavBhTPx3xwWka7StTtkr4MbdXCD2OanQHsQ-H6Tc-HpMNyo40vVEDjJPxNqpuffSaKaoQDUDdw-IxjaQn-2b7qsxkVoPVHEAib_puZyD-homI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uVQ_L8iWmU17-2UZ2CroXluwMZC-zfARRQukxvB_vuLt_pHg6uunDnKjfBRoNyhiQhc6Wpk_S0ZPa9hssoQoMuqpv8wwOZenTirOd88z9k2ew7ETxRW60_ekz05oYrJz9W1FU4EnjTg8GXFLzt5_GeDwwpxIQVB-OV0u0Df0RtDcL1OoUZwbUNS4mi2S-2hQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Opv31F0gEgZ5XtK25ougvBz6Flfamn6KBRXvwbC0fPUqk5U82TUXOA2lcofd4wWThtiYWdVxpyG2CZfucIckobMcghvCoTO2ragSrkvMLrcwtHhQYJgHezrOpqRpHRO1wP1-tnn5S3xD_oCpi6Ctmlj6Qfi8OJDsXLQgJj7vqPBgFqvDHnDik9sUR75bNnzp?purpose=fullsize)

概念如下：

```text
             CAN Bus
    ═════════════════════════
       │       │       │
       │       │       │
       ▼       ▼       ▼
    引擎ECU   ABS ECU  儀表板
       │       │       │
       ▼       ▼       ▼
     引擎     煞車    顯示車速
```

每個設備都可以透過共同的 CAN Bus 傳送或接收訊息。

這就是 CAN 的基本概念。

## 1.2 CAN 使用什麼線路？

一般 CAN Bus 使用兩條主要訊號線：

* CAN_H
* CAN_L

它們採用差動訊號傳輸。

你可以先把它理解成：

不是單純依靠某一條線的電壓判斷 0 或 1，而是比較兩條線之間的電壓差。

這種方式有助於抵抗外部電氣雜訊。

例如，在工業設備中，馬達、驅動器和電源都可能產生電氣干擾，差動傳輸有助於提高通訊可靠性。

另外，CAN 還有一個特點：

**多個設備可以共用同一條通訊匯流排，而且具有訊息優先權仲裁機制。**

例如，煞車相關訊息可能需要比一般儀表資訊更高的優先權。

CAN 會依照訊息識別碼（Identifier）進行仲裁。

這裡先不用深入，知道它可以讓多個設備共用通訊線路就好。

參考：[NASA/ESA CAN Bus 技術介紹](https://www.esa.int/Enabling_Support/Space_Engineering_Technology/Onboard_Computers_and_Data_Handling/CAN_-_Controller_Area_Network_Bus?utm_source=chatgpt.com)

---

# 二、CAN 和 CANopen 有什麼不同？

這是另一個很容易混淆的地方。

你可以這樣想：

**CAN 是道路，CANopen 是車輛在這條道路上行駛時遵守的交通規則。**

例如：

| 名稱          | 生活中的比喻  | 真正的用途   |
| ----------- | ------- | ------- |
| CAN         | 道路      | 通訊傳輸技術  |
| CANopen     | 交通規則    | 設備通訊協定  |
| CAN 訊息      | 行駛的車輛   | 傳送的資料訊息 |
| CANopen PDO | 定期配送的貨物 | 週期性控制資料 |
| CANopen SDO | 特別寄送的包裹 | 參數讀取與設定 |

CANopen 規定了很多事情，例如：

* 設備如何表示自己的參數。
* 資料如何編號。
* 如何讀取設備參數。
* 如何傳送控制命令。
* 如何回報設備狀態。
* 多個設備如何管理。

其中就包含我們前面學過的：

* Object Dictionary
* PDO
* SDO
* PDO Mapping

因此，CANopen 不只是定義如何傳送資料，也定義了設備之間如何理解這些資料。

---

# 三、那 CoE 又是什麼？為什麼 EtherCAT 裡面會有 CAN？

CoE 的完整名稱是：

**CANopen over EtherCAT**

中文可以理解為：

**在 EtherCAT 上使用 CANopen 的通訊機制。**

這裡有一個非常重要的觀念：

**CoE 並不是把 CAN 的兩條實體線路接到 EtherCAT 上，也不是把 CAN 封包原封不動塞進 Ethernet。**

它主要是讓 EtherCAT 設備沿用 CANopen 的許多應用層規則。

例如：

* Object Dictionary
* PDO Mapping
* SDO
* CiA 402 伺服驅動器設備規範

這樣設備製造商就不必為 EtherCAT 重新設計一套完全不同的參數與控制資料結構。

EtherCAT Technology Group 官方也明確說明，CoE 沿用 CANopen 的 Object Dictionary、PDO Mapping、SDO 等機制。([ethercat.org][1])

## 3.1 用 GMT 的產品來理解

我查了 GMT 官方的 R-SERVO 伺服驅動器。

官方資料顯示，R-SERVO 支援：

* EtherCAT
* CANopen
* Modbus
* 脈衝控制
* 類比量控制

也就是說，GMT 的驅動器可以提供不同的上位通訊方式。([高明鐵 GMT][2])

這裡特別注意：

**CANopen 和 EtherCAT 是不同的通訊選項。**

例如：

```text
               GMT R-SERVO
                    │
           ┌────────┴────────┐
           │                 │
           ▼                 ▼
        CANopen           EtherCAT
           │                 │
           ▼                 ▼
        CAN Bus         EtherCAT Network
```

如果選擇 CANopen，就使用 CANopen 的通訊方式。

如果選擇 EtherCAT，則可以透過 CoE 使用 CANopen 的應用層機制。

所以你在 EtherCAT 文件中看到 CAN，不代表 STM32 一定還需要接 CAN_H 和 CAN_L。

**如果你的 STM32H755 是透過 EtherCAT 控制 GMT 的 EtherCAT 驅動器，就不需要因為 CoE 這個名稱而另外接一組 CAN Bus。**

這點非常重要。

GMT 官方資料：

* [GMT R-SERVO 官方產品頁](https://www.gmtglobalinc.com/%E4%BC%BA%E6%9C%8D%E9%A9%85%E5%8B%95%E5%99%A8/R-SERVO?utm_source=chatgpt.com)
* [GMT R-SERVO 使用手冊](https://www.gmtglobalinc.com/upload-files/catalogue/electrical-controls/R-SERVO_Single-Axis_Series-Servo_Driver-Manual-TW-P.pdf?utm_source=chatgpt.com)

---

# 四、接下來是重點：PDO Mapping 到底是什麼？

我先不使用 EtherCAT 的專業術語。

我們用「準備便當」來理解。

## 4.1 假設你要準備一個便當盒

你有很多食物：

* 白飯
* 雞腿
* 青菜
* 滷蛋
* 香腸
* 豆腐
* 玉米

但是便當盒的空間有限。

你不可能把所有食物都塞進去。

你必須決定：

1. 這個便當要放哪些食物？
2. 每種食物放多少？
3. 放置順序如何？

例如：

```text
┌─────────────────────────┐
│         便當盒           │
├─────────────┬───────────┤
│             │           │
│    白飯     │   雞腿    │
│             │           │
├─────────────┼───────────┤
│    青菜     │   滷蛋    │
└─────────────┴───────────┘
```

你決定只放白飯、雞腿、青菜、滷蛋。

這個決定就像 PDO Mapping。

**PDO Mapping 就是決定要從設備的資料字典中，挑選哪些資料放進 PDO，以及資料的排列方式。**

---

# 五、把便當盒換成伺服驅動器

假設我們有一台 GMT 伺服驅動器。

它內部有很多資料：

| Index  | 名稱              | 意義   |
| ------ | --------------- | ---- |
| 0x6040 | Controlword     | 控制字  |
| 0x6041 | Statusword      | 狀態字  |
| 0x6060 | Operation Mode  | 運動模式 |
| 0x6064 | Actual Position | 目前位置 |
| 0x607A | Target Position | 目標位置 |
| 0x60FF | Target Velocity | 目標速度 |
| 0x606C | Actual Velocity | 目前速度 |

你可以把這些資料想成剛剛便當店裡的所有食物。

但是我們現在要控制馬達，不需要每次都交換全部資料。

我們只需要挑選真正重要的資料。

例如：

**Controller 要傳給伺服驅動器：**

* Controlword
* Target Position
* Operation Mode

**伺服驅動器要傳回 Controller：**

* Statusword
* Actual Position
* Actual Velocity

這就是選擇 PDO 資料的過程。

---

# 六、實際做一個 PDO Mapping 範例

我們來假設一個簡化的 GMT 伺服控制情境。

注意：以下是教學示例，不是已確認的 GMT R-SERVO 預設 PDO 配置。

## 6.1 第一個便當：RxPDO

還記得 RxPDO 是什麼嗎？

從 Slave（伺服驅動器）的角度來看：

RxPDO = 從站接收 Master 傳來的資料。

所以：

**STM32 → Servo Drive**

我們決定放入三個欄位：

| 順序 | 資料名稱            | Index  |        資料大小 |
| -- | --------------- | ------ | ----------: |
| 1  | Controlword     | 0x6040 |     2 Bytes |
| 2  | Target Position | 0x607A |     4 Bytes |
| 3  | Operation Mode  | 0x6060 |      1 Byte |
|    | **總計**          |        | **7 Bytes** |

這就是 RxPDO Mapping。

資料排列如下：

```text
STM32 傳送給 Servo Drive

┌──────────────┬───────────────────┬───────────────┐
│ Controlword  │ Target Position   │ Operation Mode│
│   2 Bytes    │    4 Bytes        │    1 Byte     │
└──────────────┴───────────────────┴───────────────┘

總長度：7 Bytes
```

假設：

```text
Controlword    = 0x000F
Target Position = 10000
Operation Mode  = 8
```

這代表 Controller 傳送的資料包含：

* 控制字 0x000F。
* 目標位置 10000。
* 運動模式 8。

實際數值單位、位置比例及模式是否支援，必須依照驅動器規格決定。

## 6.2 第二個便當：TxPDO

現在換成伺服驅動器回報資料。

TxPDO = 從 Slave 傳送給 Master 的資料。

所以：

**Servo Drive → STM32**

我們選擇：

| 順序 | 資料名稱            | Index  |         資料大小 |
| -- | --------------- | ------ | -----------: |
| 1  | Statusword      | 0x6041 |      2 Bytes |
| 2  | Actual Position | 0x6064 |      4 Bytes |
| 3  | Actual Velocity | 0x606C |      4 Bytes |
|    | **總計**          |        | **10 Bytes** |

排列如下：

```text
Servo Drive 回傳給 STM32

┌─────────────┬─────────────────┬─────────────────┐
│ Statusword  │ Actual Position │ Actual Velocity │
│  2 Bytes    │    4 Bytes      │    4 Bytes      │
└─────────────┴─────────────────┴─────────────────┘

總長度：10 Bytes
```

這樣 STM32 就可以知道：

* 驅動器目前狀態。
* 馬達目前位置。
* 馬達目前速度。

---

# 七、為什麼需要 Mapping？不能直接把所有資料傳過去嗎？

可以想像成：

你每天都要從公司寄送一個便當給同事。

如果每天都把整個冰箱送過去，當然也可以，但非常浪費。

因此，你會先決定每天需要哪些食物，再按照固定位置裝進便當盒。

PDO Mapping 的目的也很類似。

### 不使用適當 Mapping

假設設備有 100 個資料項目。

但實際控制馬達只需要 6 個。

如果不加選擇地傳送大量資料：

* 增加通訊資料量。
* 增加資料處理工作。
* 讓不需要的資料佔用傳輸空間。

### 使用 PDO Mapping

只挑選需要週期性交換的資料：

```text
Object Dictionary
│
├── Controlword
├── Statusword
├── Operation Mode
├── Actual Position
├── Target Position
├── Actual Velocity
├── Target Velocity
├── Error Code
├── Temperature
└── 其他資料
       │
       ▼
   PDO Mapping
       │
       ▼
   選出需要的資料
       │
       ▼
   組成 PDO
       │
       ▼
   週期性交換
```

這樣就能讓 EtherCAT 每個通訊週期傳送真正需要的資料。

EtherCAT Technology Group 官方對 PDO Mapping 的定義，也是從設備的資料項目中選擇要週期性交換的內容。([prelive.ethercat.org][3])

---

# 八、PDO Mapping 還有一個重要觀念：不只是選資料，還要決定位置

這是 PDO Mapping 最重要的技術細節之一。

假設 STM32 收到 7 Bytes 的 RxPDO 資料：

```text
Byte 0
Byte 1
Byte 2
Byte 3
Byte 4
Byte 5
Byte 6
```

如果我們的 Mapping 是：

| 資料              | Byte Offset |      長度 |
| --------------- | ----------: | ------: |
| Controlword     |           0 | 2 Bytes |
| Target Position |           2 | 4 Bytes |
| Operation Mode  |           6 |  1 Byte |

那麼 STM32 就知道：

```text
PDO Buffer

Byte 0 ─┐
        ├── Controlword
Byte 1 ─┘

Byte 2 ─┐
Byte 3  │
Byte 4  ├── Target Position
Byte 5 ─┘

Byte 6 ─── Operation Mode
```

這就是 Mapping 的實際意義。

如果 STM32 程式錯誤地把 Byte 2 當成 Operation Mode，而不是 Target Position 的第一個 Byte，就會讀錯資料。

因此，Master 和 Slave 必須使用一致的 PDO Mapping。

這也是為什麼 PDO Mapping 非常重要。

---

# 九、還有一個容易忽略的地方：Mapping 和 Assignment 不一樣

這兩個名詞很容易混淆。

我們繼續使用便當的例子。

### PDO Mapping：決定便當內容

例如：

```text
便當 A：
白飯 + 雞腿 + 青菜

便當 B：
炒飯 + 豬排 + 滷蛋
```

### PDO Assignment：決定今天實際要使用哪個便當

例如：

```text
今天使用：
便當 A

明天使用：
便當 B
```

在 EtherCAT 中：

* PDO Mapping：決定一個 PDO 裡面有哪些資料。
* PDO Assignment：決定哪些已經定義好的 PDO 真正啟用，參與週期性資料交換。

這是兩個不同的步驟。

ETG 官方文件還列出相關的物件：

| Index 範圍      | 用途               |
| ------------- | ---------------- |
| 0x1600～0x17FF | RxPDO Mapping    |
| 0x1A00～0x1BFF | TxPDO Mapping    |
| 0x1C12        | RxPDO Assignment |
| 0x1C13        | TxPDO Assignment |

你不必現在記住這些編號，但未來在 STM32 EtherCAT 程式或驅動器手冊中看到它們，就知道大概是在設定什麼。([prelive.ethercat.org][3])

---

# 十、GMT 的產品實際上也有這些機制

這不是只有理論。

我找到 GMT 官方的 R-SERVO 使用手冊。

手冊第 7 章說明 R-SERVO 支援 CANopen 和 EtherCAT，並且介紹了：

* Object Dictionary
* PDO
* SDO
* PDO Mapping

手冊也列出物件字典的索引範圍，例如：

| Index 範圍      | 意義           |
| ------------- | ------------ |
| 0x1600～0x17FF | RPDO Mapping |
| 0x1800～0x19FF | TPDO 通訊參數    |
| 0x1A00～0x1BFF | TPDO Mapping |

這些就是我們剛才解釋的內容。([高明鐵 GMT][4])

另外，GMT 的 P-SERVO EC 官方產品頁明確寫出：

* 支援 CoE（CANopen over EtherCAT）。
* 符合 CiA 402 標準。
* 支援位置、速度、同步位置、同步速度及原點復歸模式。
* 最小同步時間間隔 500 μs。([高明鐵 GMT][5])

這表示 GMT 的驅動器確實使用我們正在學習的這套通訊架構。

---

# 十一、最後用一張圖，把今天學的東西全部串起來

假設你要使用 STM32H755 控制 GMT 的 EtherCAT 伺服驅動器。

整個過程可以理解為：

```text
                STM32H755
               EtherCAT Master
                      │
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        PDO Mapping       SDO 設定
             │                 │
             ▼                 ▼
      選擇週期資料       設定驅動器參數
             │                 │
             └────────┬────────┘
                      │
                      ▼
               EtherCAT Network
                      │
                      ▼
                GMT Servo Drive
                      │
                      ▼
                    Motor
```

而 CoE 就是讓 EtherCAT 設備使用 CANopen 那套資料組織及參數存取機制。

---

## 十二、今天你只需要記住這五件事

| 名詞             | 最簡單的理解                       |
| -------------- | ---------------------------- |
| CAN            | 多個電子設備共用的通訊道路                |
| CANopen        | 設備在 CAN 上交換資料時使用的規則          |
| CoE            | 在 EtherCAT 上使用 CANopen 的通訊機制 |
| PDO Mapping    | 決定便當裡面要放哪些資料、順序及大小           |
| PDO Assignment | 決定實際啟用哪些便當                   |

**我想特別提醒你：PDO Mapping 不是每次傳送資料時才臨時決定要傳什麼，而是通常在 EtherCAT 初始化配置階段就先設定好，之後進入正常運作，才按照這個固定格式週期性交換資料。**

這樣 Master 和 Slave 才能在每個週期中，知道資料的每一個 Byte 分別代表什麼。

下一個很適合接著學的是 **CiA 402 的 Controlword 和 Statusword**。這兩個就是剛才 PDO Mapping 範例裡的資料，理解它們之後，你就會開始真正看懂伺服驅動器是如何透過 PDO 接收啟動命令、回報狀態，以及執行位置控制的。

[1]: https://www.ethercat.org/technology/?utm_source=chatgpt.com "EtherCAT Technology Group | EtherCAT"
[2]: https://www.gmtglobalinc.com/%E4%BC%BA%E6%9C%8D%E9%A9%85%E5%8B%95%E5%99%A8/R-SERVO?utm_source=chatgpt.com "R-SERVO | 高明鐵 GMT"
[3]: https://prelive.ethercat.org/cn/technology/%E7%9F%A5%E8%AF%86%E5%BA%93/pdo-mapping-and-configuration/?utm_source=chatgpt.com "PDO Mapping and Configuration: ETG"
[4]: https://www.gmtglobalinc.com/upload-files/catalogue/electrical-controls/R-SERVO_Single-Axis_Series-Servo_Driver-Manual-TW-P.pdf?utm_source=chatgpt.com "R-Servo單軸系列 - 伺服驅動器"
[5]: https://www.gmtglobalinc.com/%E6%AD%A5%E9%80%B2%E9%A9%85%E5%8B%95%E5%99%A8/p-servo-ec?utm_source=chatgpt.com "P-SERVO EC 步進驅動器｜驅動器專業製造｜高明鐵 GMT"


---

```
PDO 與 PDO Mapping 這兩者的關聯是什麼？
```

可以先記住一句最重要的話：

**PDO 是實際傳輸的資料；PDO Mapping 是規定這些資料如何排列、各占多少空間，以及如何解讀的設定。**

兩者的關係，就像「包裹」和「包裹內容清單」。

## 1. PDO 是什麼？

PDO（Process Data Object，製程資料物件）是 EtherCAT Master 與 Slave 之間週期性傳輸的即時資料。

例如，STM32 作為 EtherCAT Master，要控制一個伺服驅動器，可以透過 PDO 傳送：

* Controlword：控制命令
* Target Position：目標位置
* Operation Mode：運作模式

伺服驅動器也可以透過 PDO 回傳：

* Statusword：目前狀態
* Actual Position：實際位置
* Actual Velocity：實際速度

這些資料組合起來，就是 PDO 傳輸的內容。

## 2. PDO Mapping 是什麼？

假設你要寄一個包裹，裡面要放三樣東西：

| 順序 | 資料              |          大小 |
| -- | --------------- | ----------: |
| 1  | Controlword     |     2 Bytes |
| 2  | Target Position |     4 Bytes |
| 3  | Operation Mode  |      1 Byte |
|    | **總計**          | **7 Bytes** |

這時候，你必須先規定：

1. 包裹裡面有哪些資料？
2. 每項資料占多少空間？
3. 資料排列的順序是什麼？

這個規則就是 **PDO Mapping**。

它告訴 EtherCAT Master 和 Slave：收到這 7 Bytes 時，應該如何解讀。

## 3. PDO 與 PDO Mapping 如何配合？

假設 STM32 要控制伺服驅動器。

**第一步：設定 PDO Mapping**

先規定傳輸格式：

| Byte Offset | 資料名稱            |      大小 |
| ----------: | --------------- | ------: |
|         0～1 | Controlword     | 2 Bytes |
|         2～5 | Target Position | 4 Bytes |
|           6 | Operation Mode  |  1 Byte |

因此，這個 PDO 的總長度是 7 Bytes。

**第二步：開始傳輸 PDO**

STM32 Master 根據這個格式，把資料組合成：

```text
Byte 0  Byte 1  Byte 2  Byte 3  Byte 4  Byte 5  Byte 6
  06      00      E8      03      00      00      08
```

伺服驅動器收到後，依照 PDO Mapping 解讀：

| 資料              | 收到的內容      | 意義     |
| --------------- | ---------- | ------ |
| Controlword     | 0x0006     | 控制命令   |
| Target Position | 0x000003E8 | 目標位置數值 |
| Operation Mode  | 0x08       | 運作模式數值 |

這裡的數值只是教學範例，不代表 GMT 驅動器實際採用的預設設定。

**第三步：持續週期性傳輸**

Mapping 設定完成後，EtherCAT Master 就能按照這個格式，持續傳輸 PDO。

例如每 1ms 傳送一次（僅為示意）。

---

## 4. 用一句話區分

| 名稱          | 負責什麼？           | 生活化比喻  |
| ----------- | --------------- | ------ |
| PDO         | 真正傳輸資料          | 包裹     |
| PDO Mapping | 規定包裹裡面放什麼、順序及大小 | 包裹內容清單 |

另外還有一個相關名詞：

**PDO Assignment（PDO 指派）**：決定要啟用哪些 PDO Mapping。

可以把三者的關係理解成：

1. **PDO Mapping**：設計包裹的內容與排列方式。
2. **PDO Assignment**：選擇要使用哪些包裹格式。
3. **PDO**：依照選定的格式，實際傳輸資料。

這三者是 EtherCAT 製程資料交換的重要組成部分。

官方可以參考：

**對照你目前的 STM32H755 專案：** 你使用 SPI2 傳送的 72 Bytes 軌跡資料，也有事先定義每個欄位的位置與長度。這個概念和 PDO Mapping 很接近，但兩者屬於不同的通訊協定，不能直接視為同一種機制。


---

```
我認為 RxPDO 與 TxPDO 算是一個動作，PDO Mapping 算是一個宣告，這樣的想法正確嗎？舉個實際 C 程式範例來說明
```

你的理解**大方向是正確的**，而且已經抓到兩者最重要的差異。

不過，我想稍微修正你的用詞，讓它更符合 EtherCAT 的實際運作：

* **RxPDO / TxPDO：定義資料傳輸的方向與內容，是實際週期性資料交換的通道。**
* **PDO Mapping：定義這些通道要放哪些資料，以及每筆資料的排列順序和長度，是一種組態宣告（Configuration）。**

我們直接用 C 語言來看。

# 一、先看整體關係

假設 STM32 是 EtherCAT Master，伺服驅動器是 EtherCAT Slave。

```text
             STM32 Master
                  |
          EtherCAT 通訊
                  |
       +----------+----------+
       |                     |
     RxPDO                 TxPDO
       |                     |
       v                     ^
  伺服驅動器接收         伺服驅動器回傳
       |                     |
  目標位置、控制命令      實際位置、運作狀態
```

注意：RxPDO 和 TxPDO 的 Receive / Transmit 是**以 Slave（伺服驅動器）的角度命名**。

| 名稱    | 資料方向           | 功能            |
| ----- | -------------- | ------------- |
| RxPDO | Master → Slave | Master 傳送控制命令 |
| TxPDO | Slave → Master | Slave 回傳運作狀態  |

這也是 EtherCAT Technology Group 所定義的方向。([prelive.ethercat.org][1])

# 二、用實際 C 程式理解 PDO Mapping

假設我們要控制一個伺服馬達。

我們希望 STM32 傳送三項資料給伺服驅動器：

1. Controlword：控制命令
2. Target Position：目標位置
3. Operation Mode：運作模式

伺服驅動器則回傳：

1. Statusword：目前狀態
2. Actual Position：實際位置

## 步驟 1：先宣告資料內容

這部分就是一般 C 語言的資料結構。

```c
#include <stdint.h>

/* STM32 → Servo Driver */
typedef struct
{
    uint16_t controlword;
    int32_t  target_position;
    int8_t   operation_mode;

} RxPDO_Data_t;


/* Servo Driver → STM32 */
typedef struct
{
    uint16_t statusword;
    int32_t  actual_position;

} TxPDO_Data_t;
```

這裡有兩個結構：

* `RxPDO_Data_t`：STM32 要傳給伺服驅動器的資料。
* `TxPDO_Data_t`：STM32 從伺服驅動器接收的資料。

這兩個結構只是 C 語言中的資料宣告，還沒有真正建立 EtherCAT PDO。

## 步驟 2：定義 PDO Mapping

接下來才是重點。

EtherCAT 必須知道：

* Controlword 位於 Object Dictionary 的哪個位置？
* Target Position 位於哪裡？
* 每個資料占幾個 Bits？
* 資料排列順序為何？

我們用一個簡化的 C 程式表示這些設定：

```c
typedef struct
{
    uint16_t index;
    uint8_t  subindex;
    uint8_t  bit_length;

} PDO_Mapping_t;


/* RxPDO Mapping：Master → Slave */

PDO_Mapping_t RxPDO_Mapping[] =
{
    {0x6040, 0x00, 16},  // Controlword
    {0x607A, 0x00, 32},  // Target Position
    {0x6060, 0x00,  8}   // Operation Mode
};


/* TxPDO Mapping：Slave → Master */

PDO_Mapping_t TxPDO_Mapping[] =
{
    {0x6041, 0x00, 16},  // Statusword
    {0x6064, 0x00, 32}   // Actual Position
};
```

這裡的 `index` 和 `subindex` 是伺服驅動器 Object Dictionary 的位置。

例如：

| Index  | Subindex | 名稱              |      大小 |
| ------ | -------- | --------------- | ------: |
| 0x6040 | 0x00     | Controlword     | 16 bits |
| 0x607A | 0x00     | Target Position | 32 bits |
| 0x6060 | 0x00     | Operation Mode  |  8 bits |
| 0x6041 | 0x00     | Statusword      | 16 bits |
| 0x6064 | 0x00     | Actual Position | 32 bits |

這個陣列就是我們用 C 語言表達的 **PDO Mapping 宣告**。

它告訴 EtherCAT 系統：

RxPDO 要包含哪些資料，TxPDO 要包含哪些資料。

這裡使用的 Object Dictionary Index 是常見 CiA 402 伺服物件，作為教學範例，並不代表 GMT 驅動器一定使用相同的 PDO 預設設定。

## 步驟 3：實際執行 RxPDO 傳輸

前面只是宣告資料格式。

現在才開始真正傳輸資料。

例如：

```c
RxPDO_Data_t rx_data;

/* 設定伺服控制命令 */
rx_data.controlword = 0x000F;

/* 設定目標位置 */
rx_data.target_position = 1000;

/* 設定運作模式 */
rx_data.operation_mode = 8;


/* 將資料交給 EtherCAT Master 的 Process Data Buffer */
EtherCAT_SendRxPDO(&rx_data);
```

這裡的 `EtherCAT_SendRxPDO()` 是示意函式，不是所有 EtherCAT SDK 都有的通用 API。

實際專案中，會由所使用的 EtherCAT Master Stack 將資料寫入對應的 Process Data Buffer，再由 EtherCAT 通訊週期送出。

## 步驟 4：實際接收 TxPDO

伺服驅動器執行命令後，回傳實際狀態。

```c
TxPDO_Data_t tx_data;

/* 從 EtherCAT Master 的 Process Data Buffer 取得資料 */
EtherCAT_ReadTxPDO(&tx_data);


/* 讀取伺服狀態 */
uint16_t status = tx_data.statusword;

/* 讀取實際位置 */
int32_t position = tx_data.actual_position;
```

同樣地，`EtherCAT_ReadTxPDO()` 是示意函式。

---

# 三、把整個流程串起來

現在把前面的程式整理成一個完整的概念：

```text
         PDO Mapping
         (資料格式宣告)
                |
       +--------+--------+
       |                 |
  RxPDO Mapping     TxPDO Mapping
       |                 |
       v                 v
  定義控制資料       定義回傳資料
       |                 |
       v                 ^
  STM32 傳送資料     STM32 接收資料
       |                 ^
       v                 |
  Servo Driver      Servo Driver
       |                 ^
       +-----------------+
          EtherCAT
```

整個過程可以分成三件事情：

| 階段          | 做什麼                | 性質      |
| ----------- | ------------------ | ------- |
| PDO Mapping | 宣告資料項目、順序及長度       | 組態設定    |
| RxPDO       | Master 傳送資料給 Slave | 週期性資料交換 |
| TxPDO       | Slave 傳送資料給 Master | 週期性資料交換 |

補充一點：實際 EtherCAT CoE 中，PDO Mapping 不只是 C 語言陣列，而是透過 Object Dictionary 中的 Mapping Objects 進行設定。常見索引為：

* `0x1600～0x17FF`：RxPDO Mapping
* `0x1A00～0x1BFF`：TxPDO Mapping

另外還有 PDO Assignment，用來決定哪些 Mapping 要啟用。([prelive.ethercat.org][1])

# 四、與你目前 STM32H755 專案的關係

你目前的 STM32H755 專案，其實也有類似的設計概念。

你使用 SPI2 傳輸 72 Bytes 軌跡資料：

| Byte Offset | 資料              |
| ----------- | --------------- |
| 0～1         | Sequence        |
| 2～71        | 軌跡資料，共 70 Bytes |

這個格式就是你自己定義的資料排列方式。

兩者可以這樣對照：

| 你的 SPI2 專案    | EtherCAT          |
| ------------- | ----------------- |
| 72 Bytes 傳輸封包 | PDO               |
| 定義每個欄位的位置與長度  | PDO Mapping       |
| STM32 傳送軌跡資料  | RxPDO（以 Slave 角度） |
| STM32 接收回傳資料  | TxPDO（以 Slave 角度） |

但請注意，SPI2 是你自行設計的通訊格式，EtherCAT PDO 則是依照 EtherCAT 的 Object Dictionary、Mapping 和 Assignment 機制運作。

---

**最後，我想確認你是否已經掌握一個關鍵觀念：**

PDO Mapping 本身不負責傳輸資料，也不負責計算馬達位置。

它只是先告訴 EtherCAT 系統：

「未來傳輸 PDO 時，哪些資料要放進去，以及每筆資料放在哪裡。」

真正週期性傳輸資料的，才是 RxPDO 和 TxPDO。

這就是兩者最核心的關係。

[1]: https://prelive.ethercat.org/cn/technology/%E7%9F%A5%E8%AF%86%E5%BA%93/pdo-mapping-and-configuration/?utm_source=chatgpt.com "PDO Mapping and Configuration: ETG"


---

```
詳細說明 SDO 
```

SDO 是 EtherCAT 中非常重要的一個概念。前面我們已經了解 PDO、PDO Mapping，現在學習 SDO，就能把 CoE（CAN application protocol over EtherCAT）的資料交換機制串起來。

先講結論：

**PDO 主要負責週期性傳輸即時控制資料；SDO 主要負責讀取、修改伺服驅動器內部的參數與設定。**

例如：

* PDO：每 1ms 傳送一次馬達目標位置。
* SDO：讀取馬達目前設定的編碼器解析度，或修改某個控制參數。

這是兩者最主要的用途差異。([ethercat.org][1])

# 一、SDO 是什麼？

SDO 全名：

**Service Data Object（服務資料物件）**

它是 CoE 通訊協定提供的一種資料存取機制，讓 EtherCAT Master 可以讀取或修改 EtherCAT Slave 裡面的 Object Dictionary（物件字典）。

先回憶一下 Object Dictionary。

你可以把伺服驅動器想像成一台具有很多內部設定的設備。

例如：

| Index  | Subindex | 名稱                    | 功能    |
| ------ | -------- | --------------------- | ----- |
| 0x6040 | 0x00     | Controlword           | 控制命令  |
| 0x6041 | 0x00     | Statusword            | 驅動器狀態 |
| 0x6060 | 0x00     | Modes of operation    | 運作模式  |
| 0x6064 | 0x00     | Position actual value | 實際位置  |
| 0x607A | 0x00     | Target position       | 目標位置  |
| 0x6081 | 0x00     | Profile velocity      | 速度設定  |

這些是常見 CiA 402 伺服驅動器物件的例子，實際支援項目和存取權限要看驅動器的規格。

你可以把 Object Dictionary 想像成伺服驅動器內部的一本資料字典。

每個資料都有自己的地址：

* Index：索引，16 Bits。
* Subindex：子索引，8 Bits。

例如：

```text
Index    = 0x6060
Subindex = 0x00
```

代表要存取 Object Dictionary 中的 `0x6060:00`，也就是 Modes of operation。

SDO 就是用來讀取或修改這些資料的工具。

GMT 的 R-SERVO 手冊也說明，設備透過 Object Dictionary 管理參數，Master 可以使用 PDO 或 SDO 存取其他節點的物件。([高明鐵 GMT][2])

---

# 二、SDO 有兩種主要操作

SDO 最主要的操作有兩種：

1. SDO Upload：讀取資料。
2. SDO Download：寫入資料。

這兩個名稱初學時容易搞混，因為它們是從 Master 與 Slave 的資料存取角度命名。

## 1. SDO Upload：讀取資料

假設 STM32 想知道伺服驅動器目前使用什麼運作模式。

STM32 向伺服驅動器詢問：

```text
請給我 0x6060:00 的內容。
```

伺服驅動器找到該資料後回傳：

```text
0x08
```

這就是 SDO Upload。

資料方向：

```text
STM32 EtherCAT Master
          |
          | SDO Upload Request
          | 請求讀取 0x6060:00
          v
   Servo Driver
          |
          | 查詢 Object Dictionary
          |
          | 取得資料 0x08
          v
STM32 EtherCAT Master
          |
          | SDO Upload Response
          | 回傳 0x08
```

生活化的例子：

你去銀行詢問帳戶餘額。

* 你：STM32 Master。
* 銀行：Servo Driver。
* 帳戶餘額：Object Dictionary 裡面的資料。
* 詢問餘額：SDO Upload。

你沒有修改銀行的資料，只是把資料讀出來。

## 2. SDO Download：寫入資料

假設 STM32 想要修改伺服驅動器的某個參數。

例如將運作模式設定成 8。

STM32 傳送：

```text
Index    = 0x6060
Subindex = 0x00
Value    = 8
```

伺服驅動器收到後，嘗試將資料寫入對應的 Object Dictionary。

資料方向：

```text
STM32 EtherCAT Master
          |
          | SDO Download Request
          | 寫入 0x6060:00 = 8
          v
   Servo Driver
          |
          | 檢查該物件是否存在
          | 檢查是否允許寫入
          | 執行資料更新
          |
          v
STM32 EtherCAT Master
          |
          | SDO Download Response
          | 回覆寫入成功或失敗
```

生活化的例子：

你到銀行修改帳戶的轉帳限額。

* 你提出修改要求。
* 銀行確認你是否有權限。
* 銀行修改資料。
* 銀行回覆修改成功或失敗。

這就是 SDO Download 的概念。

注意：SDO Download 成功不代表伺服馬達已經執行了動作，只代表該次物件寫入已成功完成。

## 3. Upload 與 Download 比較

| 項目        | SDO Upload   | SDO Download |
| --------- | ------------ | ------------ |
| 中文        | 上傳／讀取        | 下載／寫入        |
| Master 動作 | 向 Slave 讀取資料 | 向 Slave 寫入資料 |
| Slave 動作  | 回傳資料         | 接收並處理資料      |
| 典型用途      | 讀取參數、狀態      | 修改參數、設定      |

請記住：

**Upload = Master 讀取 Slave 的資料。**

**Download = Master 寫入 Slave 的資料。**

---

# 三、SDO 和 PDO 最大的差別：傳輸方式

這是理解 SDO 最重要的地方。

SDO 和 PDO 都可以存取 Object Dictionary，但兩者的傳輸方式不同。

## PDO：週期性傳輸

假設 STM32 要控制伺服馬達的位置。

每隔 1ms 傳送一次目標位置：

```text
時間       STM32 → Servo Driver

0ms        Target Position = 1000
1ms        Target Position = 1100
2ms        Target Position = 1200
3ms        Target Position = 1300
4ms        Target Position = 1400
5ms        Target Position = 1500
```

這種固定週期、不斷交換資料的方式，就是 PDO 的主要用途。

## SDO：有需要才提出請求

假設 STM32 想讀取伺服驅動器的編碼器解析度。

它不需要每 1ms 都讀取一次。

可能只在設備啟動時讀取：

```text
設備啟動
    |
    v
STM32 發出 SDO Upload
    |
    v
讀取編碼器解析度
    |
    v
Servo Driver 回傳資料
    |
    v
STM32 儲存讀取結果
    |
    v
完成
```

因此，SDO 通常是非週期性（Acyclic）的資料交換。

| 比較   | PDO                 | SDO                 |
| ---- | ------------------- | ------------------- |
| 全名   | Process Data Object | Service Data Object |
| 主要用途 | 即時控制資料              | 參數讀取與設定             |
| 傳輸方式 | 週期性或依設定觸發           | 由請求啟動               |
| 通訊機制 | Process Data        | Mailbox             |
| 典型例子 | 目標位置、實際位置           | 編碼器參數、運作模式設定        |
| 即時性  | 適合週期性即時交換           | 不適合當作固定週期控制通道       |

SDO 使用 Mailbox（郵箱）機制傳輸，因此完成時間不像 PDO 的週期性交換那樣固定。([ethercat.org][3])

---

# 四、SDO 實際上如何傳輸？

這裡要介紹一個新名詞：

**Mailbox（郵箱）**

你可以把 Mailbox 想像成一個收發信件的郵局。

PDO 比較像工廠生產線，每隔固定時間就交換資料。

SDO 則比較像寄信：

1. Master 寫好請求。
2. 將請求放進 Mailbox 傳送。
3. Slave 收到後處理。
4. Slave 將結果放進 Mailbox。
5. Master 取得回覆。

因此，SDO 不會像 PDO 一樣直接作為固定週期的 Process Data 傳輸。

## SDO 有哪些傳輸模式？

依照資料大小，SDO 有幾種傳輸方式。

| 傳輸模式               | 說明              | 適合用途               |
| ------------------ | --------------- | ------------------ |
| Expedited Transfer | 快速傳輸，最多 4 Bytes | 小型參數               |
| Normal Transfer    | 一次 Mailbox 傳輸   | 可以放進單次 Mailbox 的資料 |
| Segmented Transfer | 分段傳輸            | 大型資料               |
| Complete Access    | 一次存取整個物件及其子索引   | 批次讀取或設定            |

例如：

假設要讀取一個 32-bit 的整數：

```text
0x12345678
```

它只有 4 Bytes。

可以使用 Expedited Transfer 完成。

但如果要讀取一筆 100 Bytes 的資料，而單次 Mailbox 無法容納，就可能需要 Segmented Transfer，將資料分成多次傳輸。

ETG 的 Master 規範也明確區分 SDO Upload/Download、Segmented Transfer 和 Complete Access。([ethercat.org][3])

---

# 五、實際 C 程式：如何使用 SDO？

現在我們從程式角度理解。

假設：

* STM32 是 EtherCAT Master。
* GMT R-SERVO 是 EtherCAT Slave。
* STM32 想讀取伺服驅動器的實際位置。
* STM32 想修改伺服驅動器的運作模式。

先說明：以下使用常見的 EtherCAT Master API 風格示範。不同 EtherCAT Master Stack 的函式名稱和參數會不同，不能直接當成你目前專案的可編譯程式。

## 1. SDO Upload：讀取實際位置

CiA 402 常見的物件：

```text
Index    = 0x6064
Subindex = 0x00
名稱     = Position actual value
資料大小 = 32 Bits
```

C 程式概念：

```c
#include <stdint.h>
#include <stdio.h>

uint32_t actual_position = 0;

int result = EtherCAT_SDO_Read(
    1,                  // Slave ID
    0x6064,             // Index
    0x00,               // Subindex
    &actual_position,   // 接收資料的位址
    sizeof(actual_position)
);

if (result == 0)
{
    printf("Actual Position = %lu\n",
           (unsigned long)actual_position);
}
else
{
    printf("SDO Read Failed\n");
}
```

這裡的 `EtherCAT_SDO_Read()` 是示意函式。

程式執行後：

```text
STM32 Master
    |
    | 讀取 0x6064:00
    v
Servo Driver
    |
    | 回傳實際位置
    v
actual_position = 123456
```

如此一來，STM32 就能取得伺服驅動器的實際位置。

## 2. SDO Download：修改運作模式

接下來假設要修改伺服驅動器的運作模式。

使用：

```text
Index    = 0x6060
Subindex = 0x00
資料型態 = INT8
```

例如設定成 8。

```c
#include <stdint.h>
#include <stdio.h>

int8_t operation_mode = 8;

int result = EtherCAT_SDO_Write(
    1,                      // Slave ID
    0x6060,                 // Index
    0x00,                   // Subindex
    &operation_mode,        // 要寫入的資料
    sizeof(operation_mode)
);

if (result == 0)
{
    printf("SDO Write Success\n");
}
else
{
    printf("SDO Write Failed\n");
}
```

程式執行後：

```text
STM32 Master
    |
    | 寫入 0x6060:00 = 8
    v
Servo Driver
    |
    | 檢查是否允許寫入
    | 執行寫入
    v
回覆 STM32
    |
    v
SDO Write Success
```

這裡的模式數值 8 是 CiA 402 常見的 CSP（Cyclic Synchronous Position）模式。

但要注意：

* 驅動器必須支援該模式。
* 寫入時必須符合驅動器的狀態及設定要求。
* 之後還需要透過 PDO 等方式提供目標位置，才會形成週期性位置控制。
* 如果 `0x6060` 同時被 PDO 映射，PDO 傳入的值可能影響該物件的內容。

所以不能認為寫入 SDO 就代表馬達已經開始運轉。

---

# 六、非常重要：PDO Mapping 其實也可以透過 SDO 設定！

這是 SDO 和前面 PDO Mapping 知識真正連接的地方。

還記得我們之前學過：

```text
0x1600～0x17FF：RxPDO Mapping
0x1A00～0x1BFF：TxPDO Mapping
0x1C12：RxPDO Assignment
0x1C13：TxPDO Assignment
```

這些本身也是 Object Dictionary 裡面的物件。

因此，在驅動器允許重新設定 PDO Mapping 的情況下，Master 可以透過 SDO Download 寫入這些物件。

例如，我們想讓 RxPDO 包含：

| 物件              | Index  | Subindex |      長度 |
| --------------- | ------ | -------- | ------: |
| Controlword     | 0x6040 | 0x00     | 16 Bits |
| Target Position | 0x607A | 0x00     | 32 Bits |
| Operation Mode  | 0x6060 | 0x00     |  8 Bits |

可以透過 SDO 設定 Mapping Object。

PDO Mapping 的每個 Entry 通常使用 32-bit 數值表示：

```text
31           16 15       8 7          0
+--------------+----------+------------+
| Object Index | Subindex | Bit Length |
+--------------+----------+------------+
```

例如 Controlword：

```text
0x60400010
```

代表：

```text
Index    = 0x6040
Subindex = 0x00
Bit      = 0x10 = 16 Bits
```

Target Position：

```text
0x607A0020
```

代表：

```text
Index    = 0x607A
Subindex = 0x00
Bit      = 0x20 = 32 Bits
```

Operation Mode：

```text
0x60600008
```

代表：

```text
Index    = 0x6060
Subindex = 0x00
Bit      = 0x08 = 8 Bits
```

透過 SDO 寫入這些 Mapping Entry，便可以設定 PDO 的資料排列方式。

不過實際設定通常需要遵循特定順序，例如先將 Mapping 的 Entry 數量設為 0，再寫入各個 Entry，最後設定 Entry 數量並完成 Assignment。

而且必須注意：

**不是所有 EtherCAT Slave 都允許任意修改 PDO Mapping。**

有些 PDO 是 Fixed（固定映射），不能修改；有些則需要在特定 EtherCAT 狀態下才能設定。實際操作必須遵循設備手冊和 ESI 檔案。

ETG 對 PDO Mapping 和 Assignment 的定義也有清楚說明。([EtherCAT][4])

---

# 七、把 PDO、PDO Mapping、SDO 三者串起來

現在你應該可以理解這三者的關係了。

我用一個完整的伺服馬達設定流程說明。

假設 STM32 要控制 GMT 伺服驅動器。

### 第一階段：透過 SDO 設定伺服驅動器

STM32 使用 SDO：

1. 讀取伺服驅動器支援的參數。
2. 修改運作模式。
3. 設定允許修改的 PDO Mapping。
4. 設定 PDO Assignment。

### 第二階段：開始 PDO 週期性傳輸

設定完成後：

STM32 透過 RxPDO 傳送：

* Controlword
* Target Position
* Operation Mode

伺服驅動器透過 TxPDO 回傳：

* Statusword
* Actual Position
* Actual Velocity

### 第三階段：持續執行控制

```text
                 STM32 Master
                       |
              +--------+--------+
              |                 |
           SDO                PDO
              |                 |
        參數設定與讀取       週期性資料交換
              |                 |
              v                 v
        +-----------------------------+
        |       GMT Servo Driver      |
        |                             |
        |       Object Dictionary     |
        |                             |
        |  參數設定     即時控制資料   |
        +-----------------------------+
                       |
                       v
                    Servo
                     Motor
```

| 名稱                | 主要工作        | 生活化比喻    |
| ----------------- | ----------- | -------- |
| Object Dictionary | 保存設備的參數與資料  | 設備內部資料庫  |
| SDO               | 讀取或修改資料庫內容  | 查詢或修改設定  |
| PDO Mapping       | 規定即時傳輸資料的排列 | 設定資料格式   |
| PDO               | 週期性交換即時資料   | 持續運作的生產線 |

---

# 八、與 GMT R-SERVO 的關係

這裡特別補充 GMT 官方資料。

GMT 官方的 [R-SERVO 產品頁](https://www.gmtglobalinc.com/%E4%BC%BA%E6%9C%8D%E9%A9%85%E5%8B%95%E5%99%A8/R-SERVO?utm_source=chatgpt.com) 列出 EtherCAT 與 CANopen 等通訊方式。

GMT 的 [R-SERVO 伺服驅動器手冊](https://www.gmtglobalinc.com/upload-files/catalogue/electrical-controls/R-SERVO_Single-Axis_Series-Servo_Driver-Manual-TW-P.pdf?utm_source=chatgpt.com) 也說明了 Object Dictionary、PDO、SDO 的用途。

但有一點需要區分：

**目前我們介紹的 0x6040、0x6060、0x6064 等，是 CiA 402 常見的物件範例，不代表已經確認你們實際使用的 GMT 驅動器型號具有完全相同的支援項目、權限與 PDO 預設配置。**

要確認 GMT 實際能讀取或寫入哪些 SDO，仍需要對應型號的 EtherCAT Application Manual 或 ESI 檔案。

---

## 九、這一章最重要的觀念

請先記住以下四句話：

1. **Object Dictionary**：伺服驅動器內部保存參數與資料的地方。
2. **SDO**：Master 透過 Mailbox 讀取或修改 Object Dictionary。
3. **PDO Mapping**：定義哪些 Object Dictionary 資料要放進 PDO。
4. **PDO**：按照 Mapping 定義的格式，進行週期性即時資料交換。

我建議下一個主題深入學習 **EtherCAT State Machine（ESM，EtherCAT 狀態機）**。

因為它會讓你了解為什麼 SDO 通常在設備初始化階段進行設定、PDO 又為什麼通常要等到 OP（Operational）狀態才開始正常週期性運作。這也是實際開發 STM32 EtherCAT Master 時非常重要的一環。

[1]: https://www.ethercat.org/technology/?utm_source=chatgpt.com "EtherCAT Technology Group | EtherCAT"
[2]: https://www.gmtglobalinc.com/upload-files/catalogue/electrical-controls/R-SERVO_Single-Axis_Series-Servo_Driver-Manual-TW-P.pdf?utm_source=chatgpt.com "R-Servo單軸系列 - 伺服驅動器"
[3]: https://www.ethercat.org/download/documents/ETG1500_V1i0i2_D_R_MasterClasses.pdf?utm_source=chatgpt.com "ETG.1500 EtherCAT Master Classes Directive V1.0.2"
[4]: https://prelive.ethercat.org/cn/technology/%E7%9F%A5%E8%AF%86%E5%BA%93/pdo-mapping-and-configuration/?utm_source=chatgpt.com "PDO Mapping and Configuration: ETG"



```
深入學習 EtherCAT State Machine（ESM，EtherCAT 狀態機）
```

很好。這一章我們要深入學習 **EtherCAT State Machine（ESM，EtherCAT 狀態機）**。

這是理解 EtherCAT Master 如何啟動伺服驅動器、設定 PDO、啟動週期性資料傳輸，以及處理通訊錯誤的關鍵。

我會分成幾個階段教你。先建立整體概念，再逐一理解每個狀態，接著把前面學過的 PDO、PDO Mapping、SDO 串起來，最後用 C 程式模擬整個啟動流程。

先提醒：這裡介紹的是 EtherCAT 通訊層的標準機制，不代表已經確認 GMT SixAxes Controller 或 R-SERVO 的實際初始化程式就是以下範例。

# 第一階段：ESM 到底是什麼？

## 1. 先想像一個生活中的情境

假設你有一台 STM32H755，它負責控制一台 EtherCAT 伺服驅動器。

STM32 是 Master（主站），伺服驅動器是 Slave（從站）。

當伺服驅動器剛剛通電時，你不能直接送出馬達目標位置，就期待它立刻運轉。

為什麼？

因為伺服驅動器還不知道：

* EtherCAT 通訊是否已經初始化？
* Master 是否已經建立通訊通道？
* PDO 資料格式是否已經設定？
* Master 傳送的資料是否正確？
* 是否可以安全地接受控制命令？

因此，EtherCAT 設計了一套啟動及狀態管理機制。

這就是 ESM。

**ESM 的主要工作，就是管理 EtherCAT Master 與 Slave 在啟動及運作期間的通訊狀態，確保 Slave 在條件符合時才進入下一個狀態。** ([EtherCAT][1])

## 2. ESM 有哪些狀態？

ESM 主要有五種狀態：

| 縮寫     | 英文名稱             | 中文名稱   |
| ------ | ---------------- | ------ |
| INIT   | Initialization   | 初始化    |
| PREOP  | Pre-Operational  | 預備運作   |
| SAFEOP | Safe-Operational | 安全運作   |
| OP     | Operational      | 正常運作   |
| BOOT   | Bootstrap        | 韌體更新模式 |

其中，正常啟動流程是：

![Image](https://images.openai.com/static-rsc-4/nwIGQt_oVmJYkRh42mpGH4pY5WUJKN8KeXUW2ju-oLn7oZJOQ5YDSB8pBng03qoaHaxEi9cvo5LnECA4Bdtn9m3QGOcU3E-X0s6EZpsemgzbypEh3G-o2frA1ENigk3rFkoYzVnQc4jafKR01cLvpVKe8VYaihOfgPrIys1CYC7FhbkYTecoWmaSz6oWvT1_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/U4SkFeUHMEBKhrrlws5OZSbRz7QBMXEenJoucj18NtcwUlfW6ZSnLJsf8y025eDBGpG3nU1XX6F-4ec7nQdF170FQWpJBsXphyH9N0egQWLbjKYFdzTIhho0Vwb2xEfRQCFkUjJT8cftO_sQVW5STtPJVtc0peYp0Fi6RvM0u4ryrYzkJroFqh7iGkY9RsrA?purpose=fullsize)

```text
              電源開啟
                  |
                  v
                INIT
                  |
                  v
                PREOP
                  |
                  v
                SAFEOP
                  |
                  v
                  OP
```

這四個狀態是我們最需要理解的。

BOOT 則是額外的選擇性狀態，主要用於韌體更新。

請先記住：

**INIT → PREOP → SAFEOP → OP**

這不是任意排列的四種模式，而是一個逐步完成初始化、設定及啟動的過程。

---

# 第二階段：逐一理解四個主要狀態

我們先不看程式，先了解每個狀態允許做什麼。

## 1. INIT：初始化狀態

INIT 是 EtherCAT Slave 通電後的預設狀態。

假設你剛剛打開伺服驅動器電源。

此時：

* EtherCAT Slave Controller（ESC）已經開始工作。
* Master 可以存取部分 EtherCAT Data Link Layer（資料鏈結層）暫存器。
* 但 Application Layer（應用層）的通訊尚未開放。
* Mailbox 尚未建立完成。
* SDO 尚不能正常使用。
* PDO 尚不能進行正常的週期性資料交換。

這裡有一個重要名詞：

**ESC（EtherCAT Slave Controller）**

它是 EtherCAT Slave 裡面負責 EtherCAT 通訊處理的控制器。

你可以把它想像成伺服驅動器內部的網路通訊管理單元。

在 INIT 狀態，Master 主要進行基礎通訊初始化。

例如：

1. 掃描 EtherCAT 網路上的 Slave。
2. 確認 Slave 的基本識別資訊。
3. 讀取 Slave 的硬體識別資料。
4. 準備後續通訊所需的設定。

此時還不能透過一般 CoE SDO 來設定伺服參數。

### 生活化比喻

想像你剛買了一台電腦。

你剛按下電源：

* 電腦開始初始化硬體。
* 網路介面開始準備。
* 作業系統尚未完成啟動。

這就像 INIT。

---

## 2. PREOP：預備運作狀態

接下來，Master 請求 Slave 進入 PREOP。

這時候會發生一個非常重要的變化：

**Mailbox 通訊已經可以使用。**

因此，PREOP 可以進行：

* SDO Upload：讀取 Slave 參數。
* SDO Download：寫入 Slave 參數。
* PDO Mapping 設定（如果 Slave 支援可設定 Mapping）。
* 其他應用層參數設定。

但是：

**PREOP 還不能進行正常的 PDO Process Data 週期性傳輸。**

這一點一定要記住。

| 功能             | PREOP 是否允許 |
| -------------- | ---------- |
| Mailbox        | 可以         |
| SDO Upload     | 可以         |
| SDO Download   | 可以         |
| PDO Mapping 設定 | 視設備是否支援    |
| PDO 週期性資料交換    | 不可以        |

### 為什麼 PREOP 要先開放 SDO？

想像你買了一台伺服驅動器。

你希望它使用 CSP（Cyclic Synchronous Position）模式。

你必須先把相關參數設定好。

例如：

```text
0x6060:00 = 8
```

這代表設定常見 CiA 402 的 CSP 模式。

這時候，你可以透過 SDO Download 寫入這個參數。

另外，如果設備支援可設定 PDO Mapping，也可以在這個階段設定：

```text
RxPDO：
    Controlword
    Target Position
    Operation Mode

TxPDO：
    Statusword
    Actual Position
```

所以 PREOP 可以理解成：

**通訊已經建立到可以設定設備參數，但還沒有開始正常的週期性製程資料交換。**

---

## 3. SAFEOP：安全運作狀態

這是最容易讓初學者混淆的狀態。

因為 SAFEOP 已經可以進行 PDO 通訊，但還不能像 OP 一樣正常控制輸出。

先看它的功能：

| 功能           | SAFEOP 是否允許    |
| ------------ | -------------- |
| Mailbox      | 可以             |
| SDO Upload   | 可以             |
| SDO Download | 可以             |
| PDO 輸入資料     | 可以             |
| PDO 輸出資料     | 可以接收，但輸出維持安全狀態 |
| 正常控制輸出       | 尚未開放           |

這裡要特別解釋：

### 什麼是 PDO 輸入和輸出？

請站在 EtherCAT Slave 的角度。

**Input PDO（輸入 PDO）**

資料方向：

```text
Servo Driver → STM32 Master
```

例如：

* Statusword
* Actual Position
* Actual Velocity

這些資料由伺服驅動器回傳給 STM32。

**Output PDO（輸出 PDO）**

資料方向：

```text
STM32 Master → Servo Driver
```

例如：

* Controlword
* Target Position
* Target Velocity

這些資料由 STM32 傳給伺服驅動器。

在 SAFEOP 狀態：

* Slave 可以更新並提供有效的 Input PDO。
* Slave 可以接收 Output PDO 的資料。
* 但是 Slave 的實際輸出仍維持安全狀態。

例如，假設你有一個 EtherCAT Digital Output 模組。

在 SAFEOP 狀態：

```text
STM32 傳送：

Digital Output = ON
```

但模組仍然可能保持：

```text
實際輸出 = OFF
```

因為 SAFEOP 尚未允許正常輸出動作。

這樣設計的目的，是避免 Master 尚未準備好，就讓設備突然動作。

ETG 的文件也明確指出，SAFEOP 時輸入資料有效，但輸出維持安全狀態。([EtherCAT][1])

### 生活化比喻

想像你正在啟動一台工業機器。

現在：

* 控制系統已經連線。
* 感測器資料已經可以讀取。
* 控制命令也已經可以傳送。
* 但馬達尚未允許輸出動力。

你可以先確認設備的狀況，確定沒有問題後才允許運轉。

這就是 SAFEOP 的概念。

---

## 4. OP：正常運作狀態

這是 EtherCAT 正常運作時最重要的狀態。

當 Slave 進入 OP：

* Mailbox 通訊可以使用。
* SDO 可以使用。
* PDO Input 可以正常更新。
* PDO Output 可以正常控制。
* Master 與 Slave 可以進行完整的週期性 Process Data 交換。

例如：

STM32 每 1ms 傳送：

```text
Target Position = 1000
Target Position = 1100
Target Position = 1200
Target Position = 1300
```

伺服驅動器接收這些目標位置，並依照自己的控制設定執行。

同時回傳：

```text
Actual Position = 980
Actual Position = 1080
Actual Position = 1180
Actual Position = 1280
```

STM32 就能持續取得馬達實際位置。

這就是 OP 的主要用途。

但是請注意：

**EtherCAT 進入 OP，不代表馬達一定已經開始轉動。**

因為 ESM 和伺服驅動器本身的馬達控制狀態，是兩套不同的狀態機。

我們稍後會特別說明。

---

# 第三階段：五種狀態的功能總整理

現在把前面學過的內容整理在一起。

| 功能              | INIT | PREOP | SAFEOP |  OP |
| --------------- | :--: | :---: | :----: | :-: |
| 基礎 EtherCAT 通訊  |   ✓  |   ✓   |    ✓   |  ✓  |
| Mailbox         |   ✗  |   ✓   |    ✓   |  ✓  |
| SDO             |   ✗  |   ✓   |    ✓   |  ✓  |
| PDO Input       |   ✗  |   ✗   |    ✓   |  ✓  |
| PDO Output 正常控制 |   ✗  |   ✗   |    ✗   |  ✓  |
| 正常週期性資料交換       |   ✗  |   ✗   |   部分   |  ✓  |

這張表是理解 ESM 的重要基礎。

其中 SAFEOP 的「部分」代表：可以進行 Process Data 通訊，並取得有效輸入資料，但輸出維持安全狀態。

資料來源：ETG 官方 ESM 說明。([EtherCAT][1])

---

# 第四階段：Master 如何要求 Slave 切換狀態？

前面都是在描述狀態。

現在要進一步理解：

**STM32 Master 到底是如何要求伺服驅動器從 INIT 進入 PREOP，再進入 SAFEOP，最後進入 OP？**

這裡會遇到三個非常重要的暫存器。

## 1. AL Control Register

全名：

Application Layer Control Register。

位址：

```text
0x0120
```

這個暫存器由 Master 寫入，用來要求 Slave 切換狀態。

例如：

```text
寫入 0x0002
```

代表請求進入 PREOP。

```text
寫入 0x0004
```

代表請求進入 SAFEOP。

```text
寫入 0x0008
```

代表請求進入 OP。

## 2. AL Status Register

全名：

Application Layer Status Register。

位址：

```text
0x0130
```

這個暫存器由 Slave 更新，Master 可以讀取它來確認 Slave 目前的狀態。

例如：

```text
讀取 0x0002
```

代表 Slave 目前在 PREOP。

```text
讀取 0x0004
```

代表 Slave 目前在 SAFEOP。

```text
讀取 0x0008
```

代表 Slave 目前在 OP。

請注意：

**Master 發出狀態切換要求，不代表 Slave 一定會接受。**

Slave 必須先檢查相關條件。

例如：

Master 要求 PREOP，但 Mailbox 設定錯誤，Slave 就可能拒絕。

所以 Master 必須讀取 AL Status，確認 Slave 是否真的完成狀態切換。

## 3. AL Status Code Register

位址：

```text
0x0134
```

這個暫存器非常重要。

如果 Slave 拒絕狀態切換，或者運作中發生通訊錯誤，可以透過它取得錯誤原因。

例如：

| AL Status Code | 意義                             |
| -------------- | ------------------------------ |
| 0x0000         | No Error                       |
| 0x0011         | Invalid Requested State Change |
| 0x0012         | Unknown Requested State        |
| 0x0016         | Invalid Mailbox Configuration  |

這裡只列出部分例子，完整錯誤碼要依照 EtherCAT 規格及設備實作確認。

這三個暫存器可以這樣理解：

| 暫存器                   | 誰負責       | 工作     |
| --------------------- | --------- | ------ |
| AL Control 0x0120     | Master 寫入 | 要求切換狀態 |
| AL Status 0x0130      | Slave 更新  | 回報目前狀態 |
| AL Status Code 0x0134 | Slave 更新  | 回報錯誤原因 |

ETG 官方文件對這三個暫存器的用途有明確說明。([EtherCAT][1])

---

# 第五階段：實際啟動流程——把 PDO、SDO 全部串起來

這是本章最重要的部分。

假設 STM32H755 要控制一台 EtherCAT 伺服驅動器。

我們從電源開啟開始，逐步看看 Master 要做什麼。

## Step 1：Slave 上電，進入 INIT

```text
STM32 Master          Servo Slave
     |                     |
     |                     |
     |                 電源開啟
     |                     |
     |                     v
     |                   INIT
     |                     |
     |                     |
     +---------------------+
```

Master 開始掃描網路上的 Slave，取得基本識別資訊。

此時還不能使用正常 SDO 或 PDO。

## Step 2：Master 設定 Mailbox，要求 PREOP

Master 先初始化 Mailbox 所需的 SyncManager 通道。

接著寫入：

```text
AL Control = 0x0002
```

Slave 檢查 Mailbox 設定。

如果正確，就回覆：

```text
AL Status = 0x0002
```

此時：

```text
Slave State = PREOP
```

## Step 3：使用 SDO 設定伺服驅動器

現在 Mailbox 已經可以使用。

Master 就可以使用 SDO Download 設定參數。

例如：

```text
0x6060:00 = 8
```

設定運作模式。

如果 Slave 支援可設定 PDO Mapping，也可以設定：

```text
RxPDO Mapping：

0x6040 Controlword
0x607A Target Position
0x6060 Operation Mode
```

```text
TxPDO Mapping：

0x6041 Statusword
0x6064 Actual Position
```

另外，Master 還需要設定 PDO Assignment，以及相關 Process Data SyncManager 和 FMMU。

這些工作都是進入 SAFEOP 前的重要準備。

## Step 4：Master 要求 SAFEOP

Master 寫入：

```text
AL Control = 0x0004
```

Slave 檢查：

* PDO Mapping 是否正確？
* PDO Assignment 是否正確？
* Process Data SyncManager 是否正確？
* FMMU 是否正確？
* 如果使用 Distributed Clocks（DC），相關設定是否正確？

如果都符合要求，Slave 就進入 SAFEOP。

```text
AL Status = 0x0004
```

現在 Master 可以開始透過 PDO 進行 Process Data 通訊。

但請記住：

輸出仍維持安全狀態。

## Step 5：Master 傳送有效的 Output PDO

這一步非常重要。

在請求 OP 之前，Master 必須先準備好有效的輸出資料。

例如：

```c
controlword = 0x0006;
target_position = 1000;
operation_mode = 8;
```

Master 將這些資料放進 Output PDO，並確保 Slave 已經收到符合要求的有效資料。

如果使用 Distributed Clocks，還需要確保同步條件符合 Slave 的要求。

## Step 6：Master 要求 OP

Master 寫入：

```text
AL Control = 0x0008
```

Slave 再次檢查相關條件。

如果成功：

```text
AL Status = 0x0008
```

此時：

```text
Slave State = OP
```

正式進入正常 EtherCAT Process Data 運作。

整個流程：

```text
             Power ON
                 |
                 v
                INIT
                 |
                 | 初始化 Mailbox
                 v
               PREOP
                 |
                 | SDO 設定參數
                 | PDO Mapping
                 | PDO Assignment
                 | 設定 Process Data 通道
                 v
               SAFEOP
                 |
                 | 傳送有效 Output PDO
                 | 確認同步條件
                 v
                 OP
                 |
                 v
          正常週期性控制
```

這個啟動順序與 ETG 官方 Implementation Guide 所描述的初始化流程一致。([ethercat.org][2])

---

# 第六階段：用 C 語言理解 ESM

現在進入程式部分。

先說明：以下是教學用的 C 程式架構，不是可以直接在 STM32CubeIDE 編譯的完整 EtherCAT Master 程式。

因為實際函式名稱會依照 EtherCAT Master Stack 而不同，例如 SOEM、Acontis 或其他廠商的 Master Stack。

我們先了解程式邏輯。

## 1. 定義 ESM 狀態

```c
#include <stdint.h>

#define EC_STATE_INIT       0x01
#define EC_STATE_PREOP      0x02
#define EC_STATE_SAFEOP     0x04
#define EC_STATE_OP         0x08
```

這些是 AL Control / AL Status 中的狀態編碼。

例如：

```c
uint16_t current_state = EC_STATE_PREOP;
```

代表目前 Slave 的 ESM 狀態為 PREOP。

## 2. 要求 Slave 進入 PREOP

```c
void EtherCAT_Request_PREOP(void)
{
    uint16_t state = EC_STATE_PREOP;

    EtherCAT_WriteRegister(
        0x0120,
        &state,
        sizeof(state)
    );
}
```

這裡的 `EtherCAT_WriteRegister()` 是示意函式。

它代表透過 EtherCAT Master Stack 寫入 Slave 的 AL Control Register。

## 3. 讀取 Slave 目前狀態

```c
uint16_t EtherCAT_Read_State(void)
{
    uint16_t state = 0;

    EtherCAT_ReadRegister(
        0x0130,
        &state,
        sizeof(state)
    );

    return state;
}
```

透過 AL Status Register 讀取目前狀態。

## 4. 等待 Slave 進入指定狀態

這裡是很重要的概念。

Master 發出要求後，不能直接認定成功。

必須等待 Slave 確認。

```c
int EtherCAT_Wait_State(uint16_t expected_state)
{
    uint32_t timeout = 1000;

    while (timeout > 0)
    {
        uint16_t state = EtherCAT_Read_State();

        /*
         * 只比較狀態欄位，
         * 暫時不處理 Error Flag。
         */
        if ((state & 0x000F) == expected_state)
        {
            return 0;
        }

        timeout--;
    }

    return -1;
}
```

這段程式的概念是：

1. 讀取 Slave 狀態。
2. 確認是否符合預期。
3. 如果符合，回傳成功。
4. 如果超時仍未達成，回傳失敗。

這裡的 `timeout` 只是簡化的迴圈計數，並不是精確的毫秒計時。正式程式應使用 Master Stack 的狀態等待 API，或使用可靠的計時機制。

另外，實際程式還必須檢查 AL Status 的 Error Flag，不能只比較低四個 Bits。

## 5. 啟動流程的程式架構

把前面的概念整合：

```c
int EtherCAT_Startup(void)
{
    /* Step 1：要求 PREOP */
    EtherCAT_Request_PREOP();

    if (EtherCAT_Wait_State(EC_STATE_PREOP) != 0)
    {
        return -1;
    }


    /* Step 2：透過 SDO 設定參數 */
    if (EtherCAT_Configure_SDO() != 0)
    {
        return -2;
    }


    /* Step 3：設定 PDO Mapping */
    if (EtherCAT_Configure_PDO() != 0)
    {
        return -3;
    }


    /* Step 4：要求 SAFEOP */
    EtherCAT_Request_SAFEOP();

    if (EtherCAT_Wait_State(EC_STATE_SAFEOP) != 0)
    {
        return -4;
    }


    /* Step 5：準備並傳送有效 Output PDO */
    EtherCAT_Send_Valid_Output();


    /* Step 6：要求 OP */
    EtherCAT_Request_OP();

    if (EtherCAT_Wait_State(EC_STATE_OP) != 0)
    {
        return -5;
    }


    return 0;
}
```

這個程式最重要的地方，不是函式名稱，而是它的執行順序：

```text
PREOP
  |
  v
SDO 設定
  |
  v
PDO Mapping
  |
  v
SAFEOP
  |
  v
傳送有效 Output PDO
  |
  v
OP
```

實際 Master Stack 通常會自動處理部分初始化工作，例如 SyncManager、FMMU 和狀態切換，因此不一定需要使用者逐個直接操作暫存器。

---

# 第七階段：特別重要！ESM 與 CiA 402 是兩套不同的狀態機

這個觀念非常重要。

很多初學者會把 EtherCAT 的 OP 和伺服馬達的 Operation Enabled 混為一談。

但它們是不同的事情。

## 1. ESM 管理的是通訊狀態

例如：

```text
INIT
PREOP
SAFEOP
OP
```

它負責回答：

**EtherCAT 通訊目前是否已經準備好？**

## 2. CiA 402 管理的是伺服驅動器的運作狀態

例如：

```text
Switch On Disabled
       |
       v
Ready To Switch On
       |
       v
Switched On
       |
       v
Operation Enabled
```

它負責回答：

**伺服驅動器目前是否已經準備好接受馬達控制？**

這兩者的差別：

| 比較   | ESM                  | CiA 402                              |
| ---- | -------------------- | ------------------------------------ |
| 管理對象 | EtherCAT 通訊          | 伺服驅動器控制狀態                            |
| 狀態例子 | INIT、PREOP、SAFEOP、OP | Ready to Switch On、Operation Enabled |
| 管理目的 | 通訊初始化與資料交換           | 馬達控制與驅動器操作                           |
| 相關資料 | AL Control、AL Status | Controlword、Statusword               |

### 用實際例子說明

假設：

```text
EtherCAT State = OP
```

代表 EtherCAT 通訊已經進入正常運作狀態。

但是：

```text
CiA 402 State = Switch On Disabled
```

代表伺服驅動器還沒有進入允許運轉的狀態。

所以：

```text
EtherCAT = OP
Servo = Switch On Disabled
```

這種情況是可能發生的。

反過來說，即使你透過 SDO 成功設定了伺服驅動器的參數，也不代表 ESM 已經進入 OP。

**ESM 的 OP 不等於馬達已經啟動。**

這是你未來閱讀 GMT 伺服驅動器手冊時，必須特別注意的地方。

---

# 第八階段：如果 Slave 無法進入 OP，該怎麼辦？

這是實際開發時經常遇到的問題。

假設你在 STM32 Master 程式裡面要求：

```text
PREOP → SAFEOP → OP
```

但是最後停在 SAFEOP。

這時候不能直接認定是 Master 程式錯誤。

必須檢查 Slave 為什麼拒絕進入 OP。

## 1. 讀取 AL Status

讀取：

```text
0x0130
```

確認 Slave 目前的狀態。

## 2. 檢查 Error Flag

AL Status 的 Bit 4 是 Error Flag。

如果 Bit 4 = 1，表示 Slave 有狀態機錯誤。

例如：

```text
AL Status = 0x0014
```

其中：

* Bit 4 = 1：發生錯誤。
* Bit 2 = 1：目前狀態為 SAFEOP。

也就是 SAFEOP + Error。

## 3. 讀取 AL Status Code

讀取：

```text
0x0134
```

取得更具體的錯誤原因。

例如：

```text
AL Status Code = 0x0016
```

可能代表 Mailbox 設定不正確（依 ETG 所列錯誤碼）。

其他常見檢查方向包括：

| 問題                | 可能原因                                 |
| ----------------- | ------------------------------------ |
| INIT → PREOP 失敗   | Mailbox SyncManager 設定錯誤             |
| PREOP → SAFEOP 失敗 | PDO Mapping 或 Assignment 錯誤          |
| PREOP → SAFEOP 失敗 | Process Data SyncManager 或 FMMU 設定錯誤 |
| SAFEOP → OP 失敗    | Output PDO 尚未有效                      |
| SAFEOP → OP 失敗    | Distributed Clocks 同步條件不符合           |
| OP 運作中退回 SAFEOP   | Watchdog 或通訊錯誤                       |

特別提醒：

如果 Slave 在 OP 狀態下長時間沒有收到預期的 Output Process Data，可能因為 Watchdog Timeout 而退回較低狀態。

這也是為什麼 Master 必須持續監控 AL Status，而不是只在啟動時檢查一次。([EtherCAT][1])

---

# 第九階段：BOOT 是什麼？

前面提到五種狀態，還剩下 BOOT。

BOOT 是選擇性的狀態，主要用於 Slave 韌體更新。

例如：

你想更新伺服驅動器內部的韌體。

一般可能使用：

**FoE（File Access over EtherCAT）**

透過 EtherCAT Mailbox 機制傳輸韌體檔案。

BOOT 狀態通常不提供一般 PDO Process Data 通訊，也不是日常馬達控制所使用的狀態。

它主要是維護及韌體更新用途。

另外，不是所有 Slave 都支援 BOOT。([EtherCAT][1])

---

# 第十階段：把 ESM 與你目前 STM32H755 專案連結

你目前的 STM32H755 專案，有 CM7、CM4、Ethernet、SPI2、Raspberry Pi CM5 等不同功能。

我們現在討論的 EtherCAT ESM，與你目前 SPI2 傳輸軌跡資料的機制不同。

不過，如果未來你要讓 STM32H755 擔任 EtherCAT Master，控制 GMT 的 EtherCAT Servo Drive，那麼啟動流程就會涉及 ESM。

例如：

```text
STM32H755
   |
   | EtherCAT Master
   |
   +----------------------+
   |                      |
   v                      v
Servo Drive 1         Servo Drive 2
   |                      |
   v                      v
  INIT                   INIT
   |                      |
  PREOP                  PREOP
   |                      |
  SAFEOP                 SAFEOP
   |                      |
   OP                     OP
   |                      |
   v                      v
Motor 1                 Motor 2
```

假設有兩台伺服驅動器，Master 必須分別確認各 Slave 的狀態。

不能因為第一台已經進入 OP，就認為第二台也已經成功。

這也是 EtherCAT Master 的重要工作之一。

---

# 本章總結：你必須真正理解的 6 件事

| 順序 | 概念           | 你應該記住的內容                  |
| -- | ------------ | ------------------------- |
| 1  | INIT         | 基礎初始化，尚不能使用一般 SDO/PDO     |
| 2  | PREOP        | Mailbox 開放，可以使用 SDO 設定參數  |
| 3  | SAFEOP       | PDO 通訊開始，但輸出仍保持安全狀態       |
| 4  | OP           | 正常 PDO 輸入與輸出運作            |
| 5  | AL Registers | Master 要求狀態、Slave 回覆狀態與錯誤 |
| 6  | CiA 402      | 管理伺服驅動器本身的運作狀態，不等於 ESM    |

官方參考資料：

1. [EtherCAT Technology Group — ESM（EtherCAT State Machine）](https://prelive.ethercat.org/technology/knowledge-base/esm-ethercat-state-machine/?utm_source=chatgpt.com)
2. [ETG.2200 EtherCAT Implementation Guide](https://www.ethercat.org/download/documents/ETG2200_V3i2i3_G_R_EtherCATImplementationGuide.pdf?utm_source=chatgpt.com)
3. [Beckhoff — EtherCAT State Machine](https://infosys.beckhoff.com/content/1033/el5122/1036980875.html?utm_source=chatgpt.com)

---

## 接下來，我們做一個小練習

我不希望你只是看完就忘記，而是希望你真正理解 ESM 的運作邏輯。

先回答以下三個問題，不需要查資料，按照剛才學到的內容回答即可。

**問題 1：**

STM32 Master 已經讓伺服驅動器進入 PREOP。

此時 STM32 是否可以透過 SDO Download，設定伺服驅動器的運作模式 `0x6060`？

**問題 2：**

伺服驅動器已經進入 SAFEOP，STM32 也可以收到 Actual Position。

但是 STM32 傳送的 Target Position 還不能讓馬達正常執行。

這是正常的嗎？為什麼？

**問題 3：**

EtherCAT Slave 已經進入 OP，但伺服驅動器的 CiA 402 狀態仍然是 Switch On Disabled。

請問馬達是否一定能運轉？

我們先確認這三個觀念。答對後，下一步就深入學習 **ESM 狀態切換的底層機制：AL Control、AL Status、AL Status Code、SyncManager 與 FMMU**，並用實際 C 程式理解 Master 如何完成狀態切換。

[1]: https://prelive.ethercat.org/technology/knowledge-base/esm-ethercat-state-machine/?utm_source=chatgpt.com "ESM (EtherCAT State Machine): ETG"
[2]: https://www.ethercat.org/download/documents/ETG2200_V3i2i3_G_R_EtherCATImplementationGuide.pdf?utm_source=chatgpt.com "ETG.2200 EtherCAT Implementation Guide"


---
我的硬體架構是 STM32H755 使用 SPI 1 連接 EtherCAT Master Controller，再透過 網路線連接 Slave 驅動器，驅動器會控制馬達，讓 六軸 運動。在 STM32H755 韌體程式上，使用的是訊成科技 EtherCAT 模組，ECM-XF，參考網址： https://www.nextw.com.tw/master-chip-module/ 這裡面有 一些 pdf 文件手冊，我要你根據這些文件或是 Sample code 詳細解釋說明如何讓六軸運動。因為到目前為止，我還是沒有概念，覺得名詞太多，很難與實際作動或是程式聯想在一起。
```

現在終於有了最重要的硬體資訊。這會大幅改變我們理解 EtherCAT 的方式。

我先告訴你一件事：

**你的 STM32H755 並不是直接執行完整 EtherCAT Master Protocol Stack，而是透過 SPI1 控制訊成科技的 ECM-XF，由 ECM-XF 負責 EtherCAT 主站通訊。**

這個差異非常重要。之前我們討論的 PDO、SDO、ESM 等名詞，現在都可以對應到你實際硬體中的某個部分。

我查閱了 [訊成科技 ECM-XF 官方產品與下載頁面](https://www.nextw.com.tw/master-chip-module/)，並取得了官方繁體中文《ECM-XF 主站晶片使用手冊 V050》。這份手冊有 79 頁，包含 SPI 通訊格式、PDO、SDO、EtherCAT 狀態控制、CiA 402 控制，以及 EtherCAT 初始化流程。

官方也提供 STM32 SPI Sample Code、Sample Code Guide，以及 CiA 402 驅動控制範例。([nextw.com.tw][1])

接下來我會以你的實際架構來教，而不是繼續用抽象的 EtherCAT 範例。

---

# 第一部分：先認識你的實際硬體架構

## 1. 你的系統到底是怎麼連接的？

根據你提供的資訊，我們可以先建立以下架構：

![Image](https://images.openai.com/static-rsc-4/UXkcedTzL9JW1UiighnyDj5AvlLkmtNAhrm6XGwtWR-pfeCzS_kUl8xGqa4Z1VbuTQfovYHSR86X3hrM8W-0vDapIbRrAfMHAw8kMXXPb1ty3mAWR9V6K28XMjjzdohES-3iAbo4dgEXq5d-wFdFRwmoSaVSYqJabPSytiuRDmqvfBfaDPFe_Rc5clw4Ltnb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dcIfCNlb21so4peR8cV2nYL8YjR3mx8qip9td4if2GaUUaD0nUeie-FnnClTN3iw86MmdxazzOuJ-cXLANuNq0AXDx_QxueXLqzGQmNCXgYAZlHyiU_xA0kky_cI3NZtIvhINz0uZjChgcv_ROrr1bYm4PaPM1Mv8TZPbyeuJ-z1Rzz8AayGoSoh8SQRztTk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NR21_cYMIqDTPx3_TwVl7BBJ7fECtD2LUztrsU3OIIjJlxwuJNc_ZMRwnHToc3x_7zXweTz5TnDM6qTJHucwxX7QK-ANzRV3b8srH_mV7nkrS-OnxXjCI8U7q0L93_jIWFsssbW86SaqWuc8q7ZJ94-taHsvRWi_SLPGz9CdKZah0ni9krYN2ezodr5MVLfx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/exneTjPb2DqcIgXO5jXporRu9dCbkVGWPt9DA8JCXMToZpBz_DRrITRQ23nSMbS9SRT0r_HEMakpkO66rvSK12VBAInCQPQtELiPlNYpR0_bpbeSseAeE53zV4uPybU5fhTJ4LPwQSW-tmHMzgPvwHgL0rnVQF4iIkm3g_3sTDgMaXb2dPe0Dj6pB0cruJ44?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_zcWC9EBJvZ1ATxS-im7um2hsqKQ2iZf1wemMNnb88x3v12m6OO6L79HSp9R1sxR-BOGUsEAsRC2JQQPvXyIwNrPBRpaFJM-ZGy3jt80h_8Zjd-CkoxO265kyzCLbOiWSkxKI_piHIhpL3H0S-b-L5kndPR3pn5x7Jn3CGQScGX2VpH_XZoSmT1jUZUbzEM1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9wN4aFhjHrGW8fOATlsgGNnOjiKlom_3znlIE47NLi3n6xh9B__PxezXItOI3e48rPai_K_mYgC9TnfKhjO0Wlf2dAbGzjmhs6Bn4mG8o8R2jPBd51GKPa6PscVpNTV2-Xm080VAhXI49lzNF5na2ZUNbcdhInH3CeGNb0AF3jtIJio3m146Y3ydfHib6GTL?purpose=fullsize)

```text
                  STM32H755
               (Application MCU)
                      |
                      |
                 SPI1 Interface
                      |
                      |
                      v
              +---------------+
              |               |
              |    ECM-XF     |
              |               |
              | EtherCAT      |
              | Master Chip   |
              |               |
              +---------------+
                      |
                      |
                Ethernet Cable
                  EtherCAT
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Servo 1     Servo 2     Servo 3
          |           |           |
        Motor 1     Motor 2     Motor 3


          +-----------+-----------+
          |           |           |
          v           v           v
       Servo 4     Servo 5     Servo 6
          |           |           |
        Motor 4     Motor 5     Motor 6
```

這是六個單軸 Servo Drive 的示意架構。

**注意：我目前還不知道你的實際六軸是六個獨立 Slave，還是由數個多軸 Slave 組成。因此這張圖是用來說明原理，不代表已經確認你的實際接線。**

## 2. 這套系統其實有兩種不同的通訊

這是第一個必須建立的觀念。

### 第一種：STM32H755 → ECM-XF

使用 SPI1。

這是 MCU 與 ECM-XF 之間的通訊。

STM32H755 傳送的不是一般 EtherCAT 封包，而是：

**ECM-XF 專用 SPI Command Packet。**

例如：

```text
STM32H755
    |
    | SPI1
    |
    | 傳送命令：
    | 初始化 EtherCAT
    | 設定 PDO
    | 讀取 SDO
    | 傳送目標位置
    |
    v
ECM-XF
```

### 第二種：ECM-XF → Servo Drive

使用 Ethernet 實體連線，通訊協定是 EtherCAT。

這一段才是真正的 EtherCAT 通訊。

```text
STM32H755
    |
    | SPI1 Command
    v
ECM-XF
    |
    | EtherCAT PDO / SDO
    v
Servo Drive
    |
    v
Motor
```

所以你可以這樣理解：

| 通訊       | 連接對象                 | 用途              |
| -------- | -------------------- | --------------- |
| SPI1     | STM32H755 ↔ ECM-XF   | STM32 控制 ECM-XF |
| EtherCAT | ECM-XF ↔ Servo Drive | 控制伺服驅動器         |
| 馬達電力與回授  | Servo Drive ↔ Motor  | 驅動馬達、取得編碼器回授    |

**STM32 不需要自己處理每一個 EtherCAT 底層封包，ECM-XF 已經幫你處理了大部分 EtherCAT Master 工作。**

訊成科技的官方 FAQ 也說明，ECM-XF 屬於硬體主站方案，將底層 EtherCAT 通訊工作交給專用晶片，上位 MCU 負責控制演算。([nextw.com.tw][2])

---

# 第二部分：STM32H755 到底要做什麼？

我們先不要急著看程式。

假設你希望六軸機構移動到某個位置：

```text
Axis 1 = 1000
Axis 2 = 2000
Axis 3 = 3000
Axis 4 = 4000
Axis 5 = 5000
Axis 6 = 6000
```

這些數字代表六個軸各自的目標位置（單位及比例尚未定義）。

請問：

誰負責計算這六個目標位置？

誰負責把它們送到六個伺服驅動器？

誰負責讓馬達實際轉動？

答案如下：

| 工作                   | 負責設備              |
| -------------------- | ----------------- |
| 計算六軸目標位置             | STM32H755         |
| 決定每個軸的運動軌跡           | STM32H755 的應用程式   |
| 將資料透過 SPI1 傳給 ECM-XF | STM32H755         |
| 執行 EtherCAT 通訊       | ECM-XF            |
| 將 PDO 傳送到伺服驅動器       | ECM-XF            |
| 接收伺服驅動器回傳的實際位置       | ECM-XF，再提供給 STM32 |
| 執行馬達位置控制             | Servo Drive       |
| 驅動馬達                 | Servo Drive       |

這裡有一個非常重要的觀念：

**ECM-XF 主要負責通訊，不代表它會自動幫你計算六軸機構的運動軌跡。**

例如你希望六個軸同時移動到指定位置，STM32 必須負責產生適當的目標位置序列，並按照預定週期交給 ECM-XF。

至於 ECM-XF 的 CiA 402 內建狀態機，可以協助控制伺服驅動器的 Servo On、Fault Reset 等操作，但它不等於完整的六軸運動規劃器。

---

# 第三部分：真正讓六軸馬達運動，需要經過哪些步驟？

我把整個過程拆成九個階段。

這九個階段就是我們後續要逐步學習的主線。

```text
       STM32H755 開機
              |
              v
       1. SPI1 初始化
              |
              v
       2. ECM-XF 初始化
              |
              v
       3. 掃描 EtherCAT Slave
              |
              v
       4. 設定 PDO Mapping
              |
              v
       5. SDO 設定伺服參數
              |
              v
       6. EtherCAT 進入 OP
              |
              v
       7. CiA 402 Servo On
              |
              v
       8. 傳送六軸目標位置
              |
              v
       9. 接收六軸實際位置
              |
              v
          六軸運動
```

我們逐一理解。

## Step 1：STM32 初始化 SPI1

首先，STM32 必須能夠與 ECM-XF 通訊。

這部分是 STM32H755 的工作。

你的程式會使用 STM32 HAL SPI API，例如：

```c
HAL_SPI_TransmitReceive();
```

SPI 通訊角色如下：

| 設備        | SPI 角色     |
| --------- | ---------- |
| STM32H755 | SPI Master |
| ECM-XF    | SPI Slave  |

根據 ECM-XF V050 使用手冊：

* SPI Mode 0。
* SPI Full Duplex（全雙工）。
* ECM-XF 是 SPI Slave。
* SPI 時脈最高可達 96Mbps。
* 每次傳輸以 Byte 為單位。
* 單一 Byte 使用 MSB First。

([nextw.com.tw][1])

這表示 STM32 必須產生 SPI Clock，並控制 CS 訊號。

### SPI 傳輸的資料長什麼樣子？

ECM-XF 不是讓 STM32 隨便送幾個 Byte 就可以。

它有自己的 SPI 封包格式。

官方手冊第 7 頁定義：

| 封包區域       |            大小 |
| ---------- | ------------: |
| Header     |      24 Bytes |
| Data       | 32～1408 Bytes |
| Check Word |       4 Bytes |
| End Word   |       4 Bytes |

預設 Data 長度為 112 Bytes。

因此，預設 SPI 封包長度為：

```text
24 + 112 + 4 + 4 = 144 Bytes
```

這是 STM32 與 ECM-XF 之間的資料封包，不是 EtherCAT PDO 封包。([nextw.com.tw][1])

### 這裡有一個非常重要的細節

ECM-XF 的 SPI 是全雙工，而且有一個特殊設計：

**STM32 在傳送目前命令時，收到的通常是上一筆命令的回應。**

例如：

```text
第一次 SPI 傳輸：

STM32 → ECM-XF：初始化命令

STM32 ← ECM-XF：尚無上一筆命令回應
```

```text
第二次 SPI 傳輸：

STM32 → ECM-XF：讀取 Slave 數量

STM32 ← ECM-XF：初始化命令的回應
```

```text
第三次 SPI 傳輸：

STM32 → ECM-XF：讀取 EtherCAT 狀態

STM32 ← ECM-XF：讀取 Slave 數量的回應
```

所以不能把「同一次 SPI 傳輸收到的資料」直接當成「剛剛送出命令的結果」。

必須利用 Header 裡面的：

* Response Index
* Response Command Code
* Return Value

確認這筆回應到底對應哪一筆命令。

這個設計在官方手冊第 6～13 頁有說明。([nextw.com.tw][1])

---

## Step 2：初始化 ECM-XF 的 EtherCAT 網路

SPI 能正常通訊後，STM32 就可以要求 ECM-XF 初始化 EtherCAT。

官方命令名稱：

`ECM_CMD_ECAT_INIT_OP`

命令碼：

```c
#define ECM_CMD_ECAT_INIT_OP    1
```

它的工作是：

**初始化 EtherCAT 網路及從站。**

這個命令還可以設定 Distributed Clocks（DC，分散式時鐘）相關參數。

例如：

* DC Activate
* SYNC0 Cycle Time
* SYNC1 Cycle Time
* Shift Time

假設我們希望 EtherCAT 的同步週期是 1ms：

```c
uint32_t sync0_cycle_time = 1000000;
```

因為手冊使用 ns 作為單位：

```text
1,000,000 ns = 1 ms
```

這裡只是示意，不代表你的實際六軸系統應該使用 1ms。真正週期要看伺服驅動器的支援能力、ESI 設定及整體控制需求。

### 什麼是 Distributed Clocks？

想像六個馬達必須一起運動。

如果六個伺服驅動器各自按照自己的時間執行命令，就可能出現：

```text
Motor 1：00:00:00.001
Motor 2：00:00:00.002
Motor 3：00:00:00.003
```

這樣六軸就可能產生時間差。

Distributed Clocks 的目的，就是讓 EtherCAT 網路上的設備依照共同的同步時間運作。

例如：

```text
SYNC0：

Motor 1 ──┐
Motor 2 ──┤
Motor 3 ──┤
Motor 4 ──┼── 同步執行
Motor 5 ──┤
Motor 6 ──┘
```

這對六軸同步控制非常重要。

但要注意：DC 只能協助同步執行時間，不會自動計算六軸的運動軌跡。

ECM-XF 手冊第 17～19 頁介紹初始化及 DC 設定命令。([nextw.com.tw][1])

---

## Step 3：掃描 EtherCAT Slave

初始化完成後，STM32 要知道網路上有哪些伺服驅動器。

ECM-XF 提供兩個重要命令：

| 命令                          | 功能          |
| --------------------------- | ----------- |
| `ECM_CMD_ECAT_SLV_CNT_GET`  | 讀取 Slave 數量 |
| `ECM_CMD_ECAT_SLV_INFO_GET` | 讀取 Slave 資訊 |

假設讀取結果是：

```text
Slave Count = 6
```

代表 ECM-XF 找到六個 Slave。

接著可以逐一讀取：

```text
Slave 0
Slave 1
Slave 2
Slave 3
Slave 4
Slave 5
```

每個 Slave 可以讀取：

* Vendor ID
* Product Code
* Revision
* Product Name
* EtherCAT State
* AL Status
* RxPDO Size
* TxPDO Size

這些資訊非常重要。

因為我們還不知道你的六軸伺服驅動器的實際型號，也不知道每個驅動器的 PDO Mapping。

不能直接假設六個驅動器的資料排列都相同。

官方手冊第 27～28 頁有列出這些 Slave 資訊讀取功能。([nextw.com.tw][1])

---

# 第四部分：讓 STM32 知道每個伺服驅動器需要什麼資料

這個部分就會用到我們之前學過的 PDO Mapping。

假設某一個 Servo Drive 使用 CiA 402。

我們希望 STM32 傳送：

| 資料                 | Object Index | 用途     |
| ------------------ | ------------ | ------ |
| Controlword        | 0x6040       | 控制伺服狀態 |
| Target Position    | 0x607A       | 目標位置   |
| Modes of Operation | 0x6060       | 控制模式   |

伺服驅動器回傳：

| 資料                    | Object Index | 用途    |
| --------------------- | ------------ | ----- |
| Statusword            | 0x6041       | 驅動器狀態 |
| Position Actual Value | 0x6064       | 實際位置  |

這只是我們用來說明的常見 CiA 402 配置。

實際使用前，必須確認你的伺服驅動器是否支援這些物件，以及目前預設 Mapping。

## 1. ECM-XF 提供 PDO 配置命令

官方命令：

```c
ECM_CMD_ECAT_PDO_CONFIG_SET
```

它可以讓 STM32 要求 ECM-XF 配置 Slave 的 PDO。

不過官方手冊有一個重要限制：

這個 API 最多支援 3 個 PDO，每個 PDO 最多 8 個 Objects。

如果超出限制，可以改用 SDO 直接設定相關 Mapping 物件。

而且不是每個 Slave 都允許 Master 修改 PDO Mapping。([nextw.com.tw][1])

## 2. PDO Mapping 設定完成後，還要重新配置 ECM-XF

官方命令：

```c
ECM_CMD_ECAT_RECONFIG_OP
```

它的用途是：

**當 PDO 重新規劃後，要求 ECM-XF 重新配置記憶體空間。**

這一點非常重要。

因為 ECM-XF 必須知道：

* 每個 Slave 的 PDO 長度。
* 每個 Slave 的資料位置。
* 每個 PDO 的排列方式。
* EtherCAT Process Data 如何對應到 ECM-XF 的資料區域。

你可以把它想成：

原本六個馬達的資料表格已經排好。

現在你改變其中一個馬達需要的資料欄位，ECM-XF 就必須重新整理整個資料表格。

---

# 第五部分：使用 SDO 設定伺服驅動器

現在我們來到另一個重要步驟。

PDO Mapping 決定週期性傳輸什麼資料。

但是伺服驅動器本身還有許多參數需要設定。

例如：

* 控制模式。
* 速度限制。
* 扭力限制。
* 加速度設定。
* 其他驅動器參數。

這些就可以使用 SDO。

ECM-XF 提供：

```c
ECM_CMD_ECAT_SDO_REQ
ECM_CMD_ECAT_SDO_GET
```

### 假設要設定 CSP 模式

CiA 402 常見的 CSP 模式：

```text
Object Index = 0x6060
Subindex     = 0x00
Value        = 8
```

STM32 透過 SPI1 要求 ECM-XF：

```text
請對 Slave 0 執行 SDO Download：

0x6060:00 = 8
```

接著 ECM-XF 透過 EtherCAT Mailbox 執行 SDO。

完成後 STM32 再透過 ECM-XF 命令取得結果。

流程：

```text
STM32H755
    |
    | SPI1
    | SDO Request
    v
ECM-XF
    |
    | EtherCAT Mailbox
    | SDO Download
    v
Servo Drive
    |
    | 修改 0x6060
    v
ECM-XF
    |
    | SPI1 回傳結果
    v
STM32H755
```

這就是前面學過的 SDO 在你實際系統中的應用。

官方手冊第 24～26 頁有定義 SDO Request 和 Get 的資料格式。([nextw.com.tw][1])

---

# 第六部分：讓伺服驅動器真正進入 Servo ON

這是從「通訊準備完成」到「馬達可以運動」的重要轉折。

前面我們學過兩套狀態機：

1. EtherCAT State Machine（ESM）。
2. CiA 402 Drive State Machine。

現在要把它們連接起來。

## 1. 先讓 EtherCAT 進入 OP

ECM-XF 提供：

```c
ECM_CMD_ECAT_STATE_SET
ECM_CMD_ECAT_STATE_GET
```

可以要求指定 Slave 進入：

* INIT
* PREOP
* SAFEOP
* OP

例如：

```text
ECM_CMD_ECAT_STATE_SET

Slave = 0
State = OP
```

但必須確認每個 Slave 的狀態切換是否成功。

## 2. 再讓 CiA 402 進入 Operation Enabled

這是另外一件事。

CiA 402 的狀態大致如下：

```text
Switch On Disabled
        |
        v
Ready To Switch On
        |
        v
Switched On
        |
        v
Operation Enabled
        |
        v
Servo ON
```

只有到達 Operation Enabled，伺服驅動器才進入允許正常驅動馬達的狀態。

這裡有一個很重要的發現：

**ECM-XF 本身就內建 CiA 402 狀態機控制功能。**

這代表你不一定需要在 STM32 裡面自己撰寫完整的 CiA 402 狀態轉換邏輯。

官方提供以下命令：

| 命令                       | 用途                                     |
| ------------------------ | -------------------------------------- |
| `ECM_CMD_402_CONFIG_SET` | 設定 Controlword / Statusword 在 PDO 中的位置 |
| `ECM_CMD_402_STATE_SET`  | 要求指定軸切換 CiA 402 狀態                     |
| `ECM_CMD_402_STATE_GET`  | 讀取指定軸目前狀態                              |
| `ECM_CMD_402_CTL_SET`    | 設定 ECM-XF 內部 402 狀態機控制                 |
| `ECM_CMD_402_CTL_GET`    | 讀取內部控制設定                               |

例如：

```text
OPERATION ENABLED = 0x27
```

官方手冊明確指出，Operation Enabled 就是 Servo ON 狀態。([nextw.com.tw][1])

### 這裡有一個我特別希望你注意的地方

`ECM_CMD_402_CONFIG_SET` 需要你告訴 ECM-XF：

* Controlword 在 RxPDO 的哪個 Byte Offset。
* Statusword 在 TxPDO 的哪個 Byte Offset。
* 對應哪個 Slave。
* 對應哪個馬達編號。

這就是為什麼我們前面必須先確認 PDO Mapping。

如果 Offset 設錯，ECM-XF 就可能讀錯 Controlword 或 Statusword，無法正確控制伺服狀態。

官方手冊第 31～33 頁對這部分有詳細說明。([nextw.com.tw][1])

---

# 第七部分：真正讓六軸一起運動

現在才是你最關心的部分。

假設：

* 六個伺服驅動器都已經連線。
* EtherCAT 已經進入 OP。
* 六個驅動器都已經進入 Operation Enabled。
* 六個軸都已經完成必要的回零或位置基準設定。
* 控制模式已設定為 CSP。

現在要讓六軸移動到指定位置。

## 1. STM32 建立六軸目標位置

我們用一個簡單的 C 結構來表示。

```c
#include <stdint.h>

typedef struct
{
    int32_t target_position;
    int32_t actual_position;

    uint16_t controlword;
    uint16_t statusword;

} AxisData_t;


AxisData_t Axis[6];
```

這代表六個軸各自有：

* 目標位置。
* 實際位置。
* 控制字。
* 狀態字。

接著設定六軸目標位置：

```c
Axis[0].target_position = 1000;
Axis[1].target_position = 2000;
Axis[2].target_position = 3000;
Axis[3].target_position = 4000;
Axis[4].target_position = 5000;
Axis[5].target_position = 6000;
```

這只是設定目標位置。

此時馬達還不會因為這幾行程式就開始運轉。

還需要將這些資料依照 PDO Mapping 的排列方式組成 PDO。

## 2. 把六軸資料組合成 RxPDO

假設每軸 RxPDO 只包含：

| 欄位              |      大小 |
| --------------- | ------: |
| Controlword     | 2 Bytes |
| Target Position | 4 Bytes |
| 合計              | 6 Bytes |

假設六個軸各有一份這樣的資料：

```text
Axis 0：6 Bytes
Axis 1：6 Bytes
Axis 2：6 Bytes
Axis 3：6 Bytes
Axis 4：6 Bytes
Axis 5：6 Bytes
```

總計：

```text
6 × 6 = 36 Bytes
```

那麼 STM32 就要依照實際 PDO Mapping，把 36 Bytes 的資料交給 ECM-XF。

**重要：36 Bytes 是教學假設，不是你的 ECM-XF 或 GMT 驅動器已確認的實際 PDO 長度。**

實際長度取決於每個 Slave 的 PDO Mapping。

## 3. ECM-XF 將資料送給六個伺服驅動器

ECM-XF 提供兩種很值得了解的命令。

### A. ECM_CMD_ECAT_PDO_DATA_OP

這個命令用來存取 PDO 資料。

官方手冊定義：

* 傳入 RxPDO 資料。
* 讀回 TxPDO 資料。
* 可控制讀寫操作。

不過手冊特別註明，若要寫入 PDO，應改用 FIFO 命令。

### B. ECM_CMD_ECAT_PDO_DATA_FIFO_OP

這是六軸運動中特別值得研究的命令。

它可以透過 FIFO（First In First Out，先進先出緩衝區）存取 PDO。

例如：

```text
STM32H755
    |
    | SPI1
    | 傳送一組六軸目標位置
    v
ECM-XF
    |
    | RxPDO FIFO
    |
    | 週期性輸出 PDO
    v
六個 Servo Drive
```

FIFO 可以先儲存 PDO 輸出資料，再依照設定進行週期性輸出。

這很適合需要連續輸出多筆運動目標資料的應用。

官方手冊第 19～20 頁有定義 FIFO 的資料格式與讀寫操作。([nextw.com.tw][1])

---

# 第八部分：六軸如何真正同步移動？

這是整個六軸運動控制的核心。

我們用一個非常簡單的例子。

假設六軸要從目前位置移動到：

```text
Axis 1 = 1000
Axis 2 = 2000
Axis 3 = 3000
Axis 4 = 4000
Axis 5 = 5000
Axis 6 = 6000
```

如果直接把這些目標位置一次送出去，六個馬達不一定會以你希望的速度、加速度及時間完成運動。

因此 STM32 必須產生一連串目標位置。

例如：

| 時間   | Axis 1 | Axis 2 | Axis 3 | Axis 4 | Axis 5 | Axis 6 |
| ---- | -----: | -----: | -----: | -----: | -----: | -----: |
| 0ms  |      0 |      0 |      0 |      0 |      0 |      0 |
| 1ms  |    100 |    200 |    300 |    400 |    500 |    600 |
| 2ms  |    200 |    400 |    600 |    800 |   1000 |   1200 |
| 3ms  |    300 |    600 |    900 |   1200 |   1500 |   1800 |
| 4ms  |    400 |    800 |   1200 |   1600 |   2000 |   2400 |
| ...  |    ... |    ... |    ... |    ... |    ... |    ... |
| 10ms |   1000 |   2000 |   3000 |   4000 |   5000 |   6000 |

這代表：

每隔 1ms，STM32 就計算一次六個軸的新目標位置。

然後把這六個目標位置一起交給 ECM-XF。

```text
                  STM32H755
                       |
                       |
                 每 1ms 計算
                       |
              +--------+--------+
              |        |        |
             X1       X2       X3
              |        |        |
             X4       X5       X6
              |        |        |
              +--------+--------+
                       |
                       |
                  SPI1 傳輸
                       |
                       v
                    ECM-XF
                       |
                       |
                 EtherCAT PDO
                       |
           +-----------+-----------+
           |           |           |
           v           v           v
         Servo 1     Servo 2     Servo 3
           |           |           |
           v           v           v
         Motor 1     Motor 2     Motor 3

           +-----------+-----------+
           |           |           |
           v           v           v
         Servo 4     Servo 5     Servo 6
           |           |           |
           v           v           v
         Motor 4     Motor 5     Motor 6
```

這樣六軸就可以按照同一組時間基準執行運動。

### 但這裡還有一個重要問題

你可能會問：

「STM32 每 1ms 計算一次位置，ECM-XF 也每 1ms 傳輸一次 EtherCAT，這樣不就同步了嗎？」

不完全是。

這裡有兩個不同的時間概念：

1. STM32 計算及傳送目標位置的時間。
2. ECM-XF 透過 EtherCAT 執行 PDO 交換的時間。

即使兩者週期都是 1ms，也不代表它們天然就完全同步。

因此還要考慮：

* ECM-XF 的 FIFO。
* SPI 傳輸時機。
* EtherCAT DC。
* SYNC0。
* PDO 更新時機。

這些是我們後續要深入的內容。

---

# 第九部分：用實際 C 程式把整個流程串起來

現在我用一份簡化的程式架構，讓你看到 STM32 的應用程式大概會長什麼樣子。

注意：這是依照 ECM-XF V050 手冊命令設計的教學骨架，並不是官方 Sample Code 的原始程式，也不是可以直接編譯的完整程式。實際 API、封包結構、資料排列必須以你取得的 Sample Pack 和驅動程式為準。

## 1. 宣告六軸資料

```c
#include <stdint.h>

#define AXIS_COUNT 6

typedef struct
{
    int32_t target_position;
    int32_t actual_position;

    uint16_t controlword;
    uint16_t statusword;

} AxisData_t;


AxisData_t g_axis[AXIS_COUNT];
```

## 2. 初始化 ECM-XF

```c
void ECM_Init(void)
{
    /* 1. 初始化 STM32 SPI1 */
    MX_SPI1_Init();

    /* 2. 初始化 ECM-XF SPI 通訊 */
    ECM_SPI_Init();

    /* 3. 初始化 EtherCAT 網路 */
    ECM_ECAT_Init();

    /* 4. 讀取 Slave 數量 */
    ECM_Read_Slave_Count();

    /* 5. 讀取各 Slave 資訊 */
    ECM_Read_Slave_Info();
}
```

這裡的 `ECM_ECAT_Init()` 等函式是我們預計建立的應用層包裝函式。

實際底層會使用 ECM-XF 的 SPI 命令。

## 3. 配置 PDO

```c
void ECM_Config_PDO(void)
{
    /* 1. 讀取各 Slave 原始 PDO Mapping */

    /* 2. 確認 RxPDO Mapping */

    /* 3. 確認 TxPDO Mapping */

    /* 4. 必要時設定 PDO Mapping */

    /* 5. 必要時重新配置 ECM-XF */
}
```

這部分要特別小心。

我們不能直接假設六軸的 PDO Mapping 都相同。

## 4. 設定伺服參數

```c
void ECM_Config_Servo(void)
{
    /* 設定每個 Servo Drive 的控制模式 */

    /* 例如 CSP = 8 */

    /* 設定必要的速度、扭力限制 */

    /* 確認 SDO 設定成功 */
}
```

## 5. 進入 OP 並執行 Servo ON

```c
void ECM_Servo_Enable(void)
{
    /* 1. 要求 EtherCAT Slave 進入 OP */

    /* 2. 確認 Slave 已經進入 OP */

    /* 3. 設定各軸的 CiA 402 Controlword / Statusword Offset */

    /* 4. 啟用 ECM-XF 內建 CiA 402 狀態機 */

    /* 5. 確認各軸進入 Operation Enabled */
}
```

## 6. 六軸位置控制

```c
void Motion_Update(void)
{
    uint8_t i;

    /* 計算六軸新的目標位置 */

    for (i = 0; i < AXIS_COUNT; i++)
    {
        /* 產生新的目標位置 */
        g_axis[i].target_position += 10;
    }

    /* 將六軸資料按照 PDO Mapping 排列 */

    /* 透過 SPI1 傳給 ECM-XF */

    /* ECM-XF 透過 EtherCAT PDO 傳送 */
}
```

這個範例刻意不直接填入 PDO Offset 或 SPI Packet Bytes。

因為目前還沒有你的實際驅動器 PDO Mapping。直接填入數字會變成猜測，不能拿來控制真正的六軸設備。

---

# 第十部分：我從 ECM-XF 官方手冊發現一個非常重要的功能

這是我們後續研究時的重點。

官方手冊第 31～33 頁有一組命令：

```text
ECM_CMD_402_CONFIG_SET
ECM_CMD_402_STATE_SET
ECM_CMD_402_STATE_GET
ECM_CMD_402_CTL_SET
ECM_CMD_402_CTL_GET
```

這代表 ECM-XF 不只是單純轉送 PDO。

它還內建了 CiA 402 的伺服狀態管理功能。

例如，官方手冊說明：

`ECM_CMD_402_CONFIG_SET`

可以指定：

* Controlword 的 Byte Offset。
* Statusword 的 Byte Offset。
* 對應馬達編號。
* 對應 Slave 編號。

接著使用：

`ECM_CMD_402_STATE_SET`

就可以要求指定馬達切換到：

```text
OPERATION ENABLED
```

也就是 Servo ON。

這表示你在 STM32 程式中，有機會直接利用 ECM-XF 的內建功能管理六軸 Servo ON，而不用自行實作每個軸完整的 CiA 402 狀態機。([nextw.com.tw][1])

這對初學者來說非常有幫助。

---

# 第十一部分：我們現在還缺少哪些關鍵資料？

雖然已經可以建立整體架構，但要真正教你如何讓六軸運動，還有幾項資料不能猜測。

| 必要資料               | 為什麼需要                        |
| ------------------ | ---------------------------- |
| 六個 Servo Drive 的型號 | 確認 CiA 402 支援及控制功能           |
| 六軸的 Slave 分配       | 確認六軸與 EtherCAT 節點的對應         |
| PDO Mapping        | 確認目標位置、實際位置等資料的 Offset       |
| ESI 檔案             | 確認每個 Slave 的 EtherCAT 設定     |
| ECM-XF Sample Code | 確認 STM32 實際使用的 SPI API 和封包格式 |
| 六軸運動控制程式           | 確認目標位置如何計算及更新                |

另外，還要知道你們的六軸是：

* 六個獨立伺服驅動器。
* 三個雙軸驅動器。
* 其他組合。

這會影響 ECM-XF 的 Slave 編號、馬達編號及 PDO Offset 配置。

## 官方文件與範例下載

這是我查到的訊成科技官方資料：

1. [ECM-XF 官方產品與文件下載頁](https://www.nextw.com.tw/master-chip-module/)
2. [ECM-XF 使用手冊繁體中文版 V050](https://www.nextw.com.tw/public/downloads/ECM-XF_UserManual_TC_V050.pdf)
3. [STM32 SPI Sample Pack](https://www.nextw.com.tw/public/downloads/ECM-XF_STM32_SamplePack.zip?utm_source=chatgpt.com)
4. [ECM-XF Sample Code Guide V106](https://www.nextw.com.tw/public/downloads/ECM-XF_SampleCodeGuide_TC_V106.pdf?utm_source=chatgpt.com)
5. [ECM-XF CiA 402 Drive Configuration Sample](https://www.nextw.com.tw/public/downloads/ECM-XF_402DriveConfig.zip?utm_source=chatgpt.com)
6. [ECM-XF C Driver Sample](https://www.nextw.com.tw/public/downloads/ECM-XF_DriverSample.zip?utm_source=chatgpt.com)

目前我已經讀取 V050 使用手冊，但尚未成功取得上述 ZIP 的內部程式碼。因此我不會把前面的教學骨架說成官方 Sample Code。

---

# 第十二部分：我建議我們接下來怎麼學習

我不建議現在就開始寫六軸控制程式。

因為你現在最大的困難不是 C 語言，而是還沒有建立「程式命令 → ECM-XF → EtherCAT → Servo Drive → Motor」的完整對應關係。

所以我希望我們採用以下順序：

| 教學階段 | 主題                     | 最終要理解的事情                  |
| ---- | ---------------------- | ------------------------- |
| 1    | ECM-XF SPI Sample Code | STM32 如何透過 SPI1 控制 ECM-XF |
| 2    | ECM-XF 初始化流程           | 如何找到六個 Slave              |
| 3    | PDO Mapping            | 如何知道六軸目標位置放在哪些 Byte       |
| 4    | SDO 設定                 | 如何設定六個伺服驅動器               |
| 5    | ESM                    | 如何讓六個 Slave 進入 OP         |
| 6    | CiA 402                | 如何讓六個馬達 Servo ON          |
| 7    | PDO 資料交換               | 如何傳送六軸目標位置並接收實際位置         |
| 8    | 六軸同步控制                 | 如何產生連續運動軌跡                |

**我最希望先做的是第 1 階段：直接閱讀你們使用的 ECM-XF STM32 Sample Code。**

因為只要我們看到實際程式，就可以將剛才學過的名詞一一對應：

```text
STM32 C Function
       |
       v
ECM-XF SPI Command
       |
       v
ECM-XF EtherCAT Function
       |
       v
Servo Drive Object
       |
       v
Motor Movement
```

### 現在請你先做一件事

請從訊成科技官方網站下載：

`ECM-XF_STM32_SamplePack.zip`

**將 ZIP 檔直接上傳到這個對話。**

我會先做唯讀檢查，找出：

1. STM32 SPI1 初始化程式。
2. ECM-XF SPI 封包組成程式。
3. ECM-XF 命令傳送及回應處理程式。
4. EtherCAT 初始化程式。
5. PDO 設定及資料交換程式。
6. CiA 402 Servo ON 控制程式。

然後我們不急著修改任何程式，而是從 `main()` 開始，一個函式、一個函式地閱讀。

**我的目標是讓你最後看到一行 C 程式，就知道它如何經過 STM32、ECM-XF、EtherCAT，最後讓哪一個伺服馬達產生動作。**

[1]: https://www.nextw.com.tw/public/downloads/ECM-XF_UserManual_TC_V050.pdf "ECMXF使用手冊 Ver.051_尚未發行"
[2]: https://www.nextw.com.tw/faq/?utm_source=chatgpt.com "常見問題 (FAQ) – 訊成科技股份有限公司"


---

```
運動軌跡是什麼？若是以程式設計角度來看，可以用一個結構體來建立運動軌跡嗎？
```

可以，而且**從程式設計的角度來看，運動軌跡（Motion Trajectory）非常適合使用結構體（`struct`）來表示及管理。**

不過，我們先釐清一個重要觀念：

> **運動軌跡不是單純的馬達目標位置，而是描述馬達在一段時間內，應該如何從目前位置移動到目標位置的資料。**

我用你目前的 STM32H755 + ECM-XF + 六軸伺服馬達架構來說明。

## 一、什麼是運動軌跡？

假設我們有一個六軸機械手臂。

現在希望它從目前的位置移動到另一個位置：

* 起始位置：六個軸各自位於某個角度。
* 目標位置：六個軸移動到新的角度。
* 移動時間：例如 2 秒。
* 移動過程：不能突然跳到目標位置，而是必須按照一定速度、加速度逐漸移動。

這個移動過程，就是運動軌跡。

例如，假設只有第一軸：

| 時間      | 第一軸目標位置 |
| ------- | ------: |
| 0 ms    |      0° |
| 500 ms  |     10° |
| 1000 ms |     25° |
| 1500 ms |     40° |
| 2000 ms |     50° |

這些資料描述了第一軸隨時間變化的位置。

實際六軸系統則需要同時描述六個軸的位置變化。

## 二、可以使用 struct 建立運動軌跡

例如，我們先設計一個代表「單一時間點運動狀態」的結構體：

```cpp
struct AxisTrajectory
{
    float position;      // 目標位置
    float velocity;      // 目標速度
    float acceleration;  // 目標加速度
};
```

這個結構體代表某一個軸在某個時間點的運動狀態。

接著，六軸系統可以再建立一個結構體：

```cpp
struct MotionPoint
{
    uint32_t time;              // 時間（ms）

    AxisTrajectory axis[6];     // 六軸運動資料
};
```

這樣一來，一個 `MotionPoint` 就代表：

**在指定時間，六個軸各自應該具有的位置、速度及加速度。**

例如：

```cpp
MotionPoint point;

point.time = 1000;

point.axis[0].position = 25.0f;
point.axis[1].position = 30.0f;
point.axis[2].position = 15.0f;

point.axis[0].velocity = 10.0f;
point.axis[1].velocity = 12.0f;
point.axis[2].velocity = 5.0f;
```

這裡只是示範資料結構，並不是你目前伺服系統實際使用的單位或 PDO 格式。

## 三、如果有很多個 MotionPoint，就能組成一條運動軌跡

例如：

```cpp
#define MAX_TRAJECTORY_POINTS 1000

struct MotionTrajectory
{
    uint32_t pointCount;

    MotionPoint points[MAX_TRAJECTORY_POINTS];
};
```

這裡有三個層次：

| 結構體                | 代表意義                      |
| ------------------ | ------------------------- |
| `AxisTrajectory`   | 單一軸在某個時間點的運動狀態            |
| `MotionPoint`      | 六個軸在某個時間點的運動狀態            |
| `MotionTrajectory` | 由許多 MotionPoint 組成的完整運動軌跡 |

可以想像成：

```text
MotionTrajectory
│
├── Point 0：時間 0 ms
│   ├── Axis 0
│   ├── Axis 1
│   ├── Axis 2
│   ├── Axis 3
│   ├── Axis 4
│   └── Axis 5
│
├── Point 1：時間 1 ms
│   ├── Axis 0
│   ├── Axis 1
│   ├── Axis 2
│   ├── Axis 3
│   ├── Axis 4
│   └── Axis 5
│
├── Point 2：時間 2 ms
│   └── 六軸運動資料
│
└── Point 3：時間 3 ms
    └── 六軸運動資料
```

如此就可以用一個 `MotionTrajectory` 管理六軸的整段運動。

## 四、這些軌跡資料如何控制實際馬達？

回到你的 STM32H755 + ECM-XF 架構：

```text
STM32H755
│
│  MotionTrajectory
│  儲存整段六軸運動軌跡
│
│  每隔一定時間取得目前應執行的運動資料
│
│  將六軸目標位置轉換成 PDO 資料
│
▼
SPI1
│
▼
ECM-XF
│
│ EtherCAT PDO
│
▼
六軸 Servo Drive
│
▼
六軸馬達
```

假設 STM32 每隔 1 ms 更新一次目標位置：

1. STM32 取得目前時間應該執行的軌跡點。
2. 讀取六軸各自的目標位置。
3. 將目標位置放入對應的 PDO 資料。
4. 透過 SPI1 將資料交給 ECM-XF。
5. ECM-XF 透過 EtherCAT 將 PDO 傳給伺服驅動器。
6. Servo Drive 根據目標位置執行馬達控制。

這裡要特別注意：**軌跡資料的產生與馬達控制是不同的工作。**

STM32 可以負責產生軌跡及提供目標位置；Servo Drive 則負責依照設定的控制模式執行馬達控制。

---

## 五、我想特別讓你理解一件事

我們剛剛建立的 `MotionTrajectory` 是一種程式資料結構設計。

但實際工業控制系統中，還有另一個重要概念：

**軌跡規劃（Trajectory Planning）**

例如，假設馬達要從 0° 移動到 90°。

STM32 不一定需要事先儲存每 1 ms 的所有位置，也可以透過數學公式，在每個 1 ms 計算當下應該移動到的位置。

這兩種方式不同：

| 方法     | 運作方式                  |
| ------ | --------------------- |
| 預先建立軌跡 | 事先計算並儲存大量運動點，再依時間逐點取出 |
| 即時軌跡計算 | 每個控制週期透過公式計算當下的目標位置   |

例如，1000 ms 的運動，如果每 1 ms 儲存一個運動點，就需要 1001 個時間點（包含起點與終點）。

如果使用即時計算，就不一定需要儲存這麼多資料。

**這兩種方法都可以使用 struct 管理資料。**

---

### 下一步，我建議先弄清楚一個關鍵問題

以你目前的 STM32H755 六軸系統來說，我們需要先確認：

**你的同事提供給 STM32 的軌跡資料，是已經計算好的六軸目標位置，還是 STM32 必須自行計算六軸的運動軌跡？**

這兩種架構會直接影響我們應該如何設計 `struct`、軌跡緩衝區（Trajectory Buffer），以及 1 ms 的 PDO 更新流程。


---

## 根據程式學習

很好，這次的結果非常有價值。我們已經找到你同事實際控制 ECM-XF 的核心程式了。

而且，我看到一個值得優先研究的地方：**你同事目前這段 1 ms PDO 控制程式，明確寫入的是 Axis[0]，也就是第 0 軸。**

這不代表整個專案只控制一軸，因為其他軸可能在別處處理；但我們先從這段程式理解控制原理。

# 第一步：先認識目前找到的控制流程

在 `M4mainOpenFIFO.c` 中，我們找到：

| 行數      | 程式內容                               | 初步意義                 |
| ------- | ---------------------------------- | -------------------- |
| 71–72   | `RxPDOData[]`、`TxPDOData[]`        | 接收及傳送 PDO 資料         |
| 77–78   | `pRx`、`pTx`                        | 將 PDO 資料轉換成結構體指標     |
| 74      | `g_u8PdoTrigger`                   | 1 ms 週期觸發旗標          |
| 111     | EtherCAT 初始化流程                     | 啟動 ECM-XF 及 EtherCAT |
| 136     | 配置 PDO（CSP 模式）                     | 設定週期性位置控制相關資料        |
| 147–148 | 初始化及啟用 FIFO                        | 啟用 PDO FIFO          |
| 207     | `DoCyclicPdo()`                    | 每個控制週期執行的 PDO 交換     |
| 213     | `pRx->Axis[0].Target = g_cmd_pos;` | 將目標位置寫入第 0 軸         |
| 217     | `ECM_EcatPdoFifoDataExchangeAdv()` | 與 ECM-XF 交換 PDO      |
| 222     | `Cmd`、`Act`、`Sta`                  | 印出命令位置、實際位置及狀態       |

這裡先認識一個重要名詞：

**CSP = Cyclic Synchronous Position（循環同步位置模式）**

它是一種 CiA 402 運動模式。在這種模式下，控制器會按照固定週期提供目標位置，伺服驅動器依據這些目標位置控制馬達。

這與我們前面討論的「每 1 ms 提供一個新的目標位置」有關。

但還不能因此認定你同事的程式是由 STM32 計算完整軌跡，因為我們還沒看到 `g_cmd_pos` 是如何產生的。

---

# 第二步：查看真正的控制程式

現在先不要搜尋整個專案，也不要修改任何程式。

我們只需要看三個地方：

1. ECM-XF 初始化流程。
2. `DoCyclicPdo()` 的完整內容。
3. PDO 結構體的定義。

請在 PowerShell 執行以下指令。

### 1. 查看初始化與週期控制程式

```powershell
$main = "D:\stm32_workspace\test05\test05\CM4\Core\Src\M4mainOpenFIFO.c"

Write-Host "`n========== EtherCAT Initialization =========="
Get-Content $main | Select-Object -Skip 105 -First 95

Write-Host "`n========== DoCyclicPdo =========="
Get-Content $main | Select-Object -Skip 200 -First 35
```

### 2. 查看 PDO 結構體

你的主程式第 27 行引用：

```c
#include "PdoDefine.h"
```

請執行：

```powershell
$pdo = "D:\stm32_workspace\test05\test05\CM4\Core\ECM\example\ex_csp\PdoDefine.h"

Write-Host "`n========== CSP PDO Structure =========="
Get-Content $pdo
```

### 3. 找出 g_cmd_pos 是在哪裡宣告及修改的

```powershell
Write-Host "`n========== g_cmd_pos References =========="

Get-ChildItem "D:\stm32_workspace\test05\test05\CM4\Core" -Recurse -File |
    Where-Object { $_.Extension -in ".c", ".h" } |
    Select-String -Pattern "\bg_cmd_pos\b" |
    Select-Object Path, LineNumber, Line
```

---

# 第三步：接下來我們要解開的關鍵

我先告訴你為什麼要找 `g_cmd_pos`。

你同事的程式目前有這行：

```c
pRx->Axis[0].Target = g_cmd_pos;
```

我們可以把它理解成：

```text
g_cmd_pos
    │
    │  STM32 準備好的目標位置
    ▼
pRx->Axis[0].Target
    │
    │  放進 RxPDO
    ▼
ECM-XF
    │
    │  EtherCAT
    ▼
Servo Drive
    │
    ▼
馬達
```

但現在有三個重要問題還不知道：

**問題 1：`g_cmd_pos` 是什麼單位？**

它可能是 Encoder Counts（編碼器計數值），也可能是經過比例換算的位置數值。不能直接把它當成角度。

**問題 2：`g_cmd_pos` 如何變化？**

例如：

```text
1000 → 2000 → 3000 → 4000 → 5000
```

是 STM32 每 1 ms 自行增加數值，還是由其他程式一次給定？

這會決定軌跡究竟在哪裡產生。

**問題 3：為什麼目前程式只看到 `Axis[0]`？**

我們要確認六軸資料是如何配置的，以及其他五軸是否透過相同結構處理。

---

### 我們這次的學習順序

我會按照以下順序帶你理解：

1. **先理解目標位置 Target 是什麼。**
2. 找出目標位置的單位及來源。
3. 理解 CSP 如何讓馬達逐步移動。
4. 找出速度、加速度及減速度由誰設定。
5. 理解六軸資料如何放進 PDO。
6. 最後再討論如何設計自己的運動軌跡結構體。

請先貼上這三段指令的結果。

我們先把 `g_cmd_pos` 搞清楚。只要理解這個變數，你就會開始真正理解「程式中的一個數字，如何變成馬達實際移動的位置」。


很好！這次我們已經找到非常重要的線索，而且我可以開始用你同事的實際程式解釋「馬達如何逐步移動到指定位置」。

先說結論：

**你同事目前這段程式，已經有一個簡單的運動位置更新機制。它不是每次直接指定一個最終位置，而是在每個 1 ms 週期，把目前命令位置增加一個步進量，再傳給 ECM-XF。**

這正是我們要學習的核心。

# 一、先看懂馬達目標位置是怎麼產生的

你貼出的程式有這幾行：

檔案：
`D:\stm32_workspace\test05\test05\CM4\Core\Src\M4mainOpenFIFO.c`

```c
static void DoCyclicPdo(void) {
    if (!g_servo_on)
        return;

    g_cmd_pos += shared->nIncStep;

    pRx->Axis[0].Target = g_cmd_pos;
    pRx->Axis[0].u16CtlWord = 0x0F;

    ECM_EcatPdoFifoDataExchangeAdv(
        (ECM_FIFO_WR | ECM_FIFO_RD), 1, RxPDOData,
        TxPDOData, sizeof(RXPDO_ST_DEF_T), &g_fifo_cnt, 0, 0, 0);
}
```

我們逐行理解。

## 1. `g_cmd_pos` 是目前的命令位置

你同事宣告：

```c
int32_t g_cmd_pos = 0;
```

它是一個 32 位元有號整數。

這代表它可以儲存正數、負數及零。

例如：

```text
g_cmd_pos = 10000
```

表示目前命令位置的數值是 10000。

但注意：**10000 不一定代表 10000 度，也不一定代表 10000 mm。**

它很可能是伺服驅動器使用的 Encoder Counts（編碼器計數單位），但我們還需要確認驅動器設定與程式換算，才能確定。

## 2. `shared->nIncStep` 是每次增加多少

這一行非常重要：

```c
g_cmd_pos += shared->nIncStep;
```

它等同於：

```c
g_cmd_pos = g_cmd_pos + shared->nIncStep;
```

假設：

```c
g_cmd_pos = 10000;
shared->nIncStep = 100;
```

執行一次之後：

```c
g_cmd_pos = 10100;
```

下一次再執行：

```c
g_cmd_pos = 10200;
```

再下一次：

```c
g_cmd_pos = 10300;
```

如果每 1 ms 執行一次，就會形成：

| 執行次數 |   時間 |  命令位置 |
| ---- | ---: | ----: |
| 0    | 0 ms | 10000 |
| 1    | 1 ms | 10100 |
| 2    | 2 ms | 10200 |
| 3    | 3 ms | 10300 |
| 4    | 4 ms | 10400 |
| 5    | 5 ms | 10500 |

這就是一種逐步增加目標位置的方式。

你可以把它想像成走樓梯：

* 每次增加 100 個位置單位。
* 每隔 1 ms 踏上一個階梯。
* 持續增加，目標位置就會逐漸往前移動。

不過，這只是用來理解程式的假設數字，並不是你同事實際設定的 `nIncStep`。

# 二、這就是 CSP 的核心概念

你同事在初始化時有這行：

```c
SetDriverCmdOpMode(0, 0, 8, 1);
```

而 `PdoDefine.h` 裡面也有：

```c
#define TEST_MODEOP 8 // CSP Mode
```

CSP 的全名是：

**Cyclic Synchronous Position（循環同步位置模式）**

它的概念是：

1. 控制器按照固定週期提供目標位置。
2. Servo Drive 接收目標位置。
3. Servo Drive 根據目標位置控制馬達。
4. 下一個週期再接收新的目標位置。
5. 持續重複。

所以，你可以先建立這個觀念：

**STM32 提供目標位置的變化，Servo Drive 負責實際控制馬達移動。**

這也回答了你原本的疑問：不需要每次都把馬達瞬間移動到最終位置，而是可以透過連續更新目標位置，形成移動過程。

但目前還不能認定你同事已經實作完整的加減速規劃。單純固定增加 `nIncStep`，只代表目標位置以固定增量變化。

---

# 三、我們還發現一個重要問題：目前看到的是第 0 軸

你貼出的 `PdoDefine.h` 有：

```c
#define TEST_DRV_CNT 4
```

以及：

```c
typedef struct __attribute__((__packed__)) _rxpdo_st_def_t
{
    AXIS_RXPDO_ST_DEF_T Axis[TEST_DRV_CNT];
} RXPDO_ST_DEF_T;
```

這表示這個結構體預留了 4 個 Axis 的 PDO 資料空間。

但是 `DoCyclicPdo()` 目前只有：

```c
pRx->Axis[0].Target = g_cmd_pos;
```

也就是明確更新第 0 軸。

我們現在還不能確定：

* 其他五軸是否在其他函式中處理。
* `TEST_DRV_CNT = 4` 是同事實際使用的配置，還是沿用 Sample Code 的設定。
* `ConfigDrive()` 實際配置了幾個驅動器。

這些都需要繼續檢查。

---

# 四、下一步：找出 nIncStep 到底從哪裡來

這是目前最重要的問題。

因為我們已經知道：

```c
g_cmd_pos += shared->nIncStep;
```

但是不知道：

**究竟是誰決定每次增加多少？**

例如：

* `nIncStep = 1`？
* `nIncStep = 100`？
* `nIncStep` 會不會隨時間改變？
* 是 Qt 上位機傳進來的？
* 是 STM32 其他程式計算出來的？
* 是同事預先設定的？

這會決定我們能不能真正理解運動軌跡。

請在 PowerShell 執行以下唯讀指令。

### 1. 搜尋 nIncStep 的所有使用位置

```powershell
$root = "D:\stm32_workspace\test05\test05\CM4"

Get-ChildItem $root -Recurse -File |
    Where-Object { $_.Extension -in ".c", ".h" } |
    Select-String -Pattern "\bnIncStep\b" |
    Select-Object Path, LineNumber, Line
```

### 2. 找出 shared 的結構體定義

```powershell
Get-ChildItem $root -Recurse -File |
    Where-Object { $_.Extension -in ".c", ".h" } |
    Select-String -Pattern "shared_data|SHARED.*DEF|shared\s*;" |
    Select-Object Path, LineNumber, Line
```

### 3. 查看 g_cmd_pos 周圍的宣告及初始化

```powershell
$main = "D:\stm32_workspace\test05\test05\CM4\Core\Src\M4mainOpenFIFO.c"

Get-Content $main | Select-Object -Skip 65 -First 30
```

---

## 五、我們接下來真正要學會的事情

我希望你最後能夠自己看懂這樣一段運動：

```text
假設目前馬達位置是 10000
           │
           ▼
STM32 取得 nIncStep
           │
           ▼
g_cmd_pos += nIncStep
           │
           ▼
得到新的目標位置 10100
           │
           ▼
寫入 PDO Target
           │
           ▼
ECM-XF 傳送給 Servo Drive
           │
           ▼
馬達朝新的目標位置移動
           │
           ▼
下一個 1 ms 週期再次執行
```

等我們找到 `nIncStep` 的來源，我就可以進一步教你：

* 如何讓馬達移動指定距離。
* 如何控制移動速度。
* 為什麼需要加速度與減速度。
* 如何讓六個軸協調到達各自的目標位置。
* 如何使用 `struct` 管理整段六軸運動軌跡。

**目前先不要修改任何程式。我們先追蹤 `nIncStep`，因為它是理解你同事目前運動控制方式的關鍵。**


你問到的是運動控制中非常重要的問題，而且這正是 STM32 韌體與伺服驅動器之間必須分工合作的地方。

你的擔心是合理的：

> 如果我要求馬達快速移動，但馬達實際上跟不上命令，會不會發生失步、失速，甚至馬達一直轉，卻沒有到達指定位置？

答案是：**有可能。但要先區分你使用的是伺服馬達還是步進馬達，以及目前驅動器採用什麼控制模式。**

以你目前的 ECM-XF + 六軸 Servo Drive 架構，我們先從伺服馬達的情況來理解。

# 一、首先要了解：命令位置不等於實際位置

你的系統有兩種不同的位置資料。

### 1. 目標位置（Target Position）

這是 STM32 傳給 ECM-XF，再透過 EtherCAT 傳給 Servo Drive 的位置命令。

例如：

```text
Target Position = 100000
```

意思是：希望馬達到達位置計數值 100000。

### 2. 實際位置（Actual Position）

這是伺服驅動器從馬達的 Encoder（編碼器）取得的實際位置資訊。

例如：

```text
Actual Position = 98000
```

代表目前馬達實際位置是 98000。

你同事的程式已經有這個設計：

```c
pRx->Axis[0].Target = g_cmd_pos;
```

這是寫入目標位置。

而這一行：

```c
pTx->Axis[0].Actual
```

則是讀取實際位置。

因此，你同事的程式已經具備觀察命令位置與實際位置的基本資料通道。

我們可以將它想像成：

```text
STM32 命令馬達移動到 100000
              │
              ▼
         Servo Drive
              │
              ▼
         控制馬達轉動
              │
              ▼
         Encoder 回報
              │
              ▼
STM32 讀到實際位置 98000
```

這時候：

```text
目標位置：100000
實際位置： 98000
位置誤差：  2000
```

這個差距就是我們需要關注的問題。

---

# 二、時間與馬達移動位置的比例如何決定？

這就是你問的第一個問題。

答案是：透過運動軌跡（Motion Trajectory）決定。

假設馬達目前位置是 0，目標位置是 100000。

我們希望它在 2 秒內到達。

如果使用固定速度移動，可以先計算：

```text
移動距離 = 100000 - 0 = 100000

移動時間 = 2 秒

平均速度 = 100000 / 2

         = 50000 counts/second
```

如果 STM32 每 1 ms 更新一次目標位置：

```text
每秒有 1000 個控制週期

每個週期增加：

50000 / 1000 = 50 counts
```

程式概念就是：

```c
g_cmd_pos += 50;
```

如此：

| 時間      |   目標位置 |
| ------- | -----: |
| 0 ms    |      0 |
| 1 ms    |     50 |
| 2 ms    |    100 |
| 3 ms    |    150 |
| 4 ms    |    200 |
| ...     |    ... |
| 2000 ms | 100000 |

這就是時間與位置的關係。

不過，這個例子有一個問題：

**馬達一開始就要求以固定速度移動，結束時又突然停止。**

實際運動控制通常不會這麼簡單。

---

# 三、為什麼不能只考慮速度？還需要加速度與減速度？

假設你開車：

* 從靜止開始。
* 一秒內要求車速達到 100 km/h。
* 接著突然要求停止。

這樣的操作會非常不平順。

馬達也是一樣。

因此，運動控制通常需要考慮三個重要參數：

| 參數  | 英文           | 意義        |
| --- | ------------ | --------- |
| 位置  | Position     | 馬達要走到哪裡   |
| 速度  | Velocity     | 馬達移動得多快   |
| 加速度 | Acceleration | 馬達速度增加得多快 |

另外還有減速度（Deceleration），用來控制馬達如何減速。

例如，我們希望馬達從 0 移動到 100000 counts：

![Image](https://images.openai.com/static-rsc-4/jxxpDOpMXx5h57BQdYaVI4Lc5rp_RGgbT8H297HVurO_29mOvr9-V1MR1a69yZ7stfyOrsNukqNzG8ooljOOIP7W3Iega2l8V4z4z5DicX2YxuWEzRe568mIvnO2XSvUwRnEZsEvU3quLzLt6qGI7rSQ6P9cFIZUHa2jvBhiPi6purL6ZOo49Wle7MDzdfZc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fFXIqpunjZvgM8O0L_-oZOO8Ju9l5jzJTOcKXrRYq0h3L7_akZegAi2ahZ_VTLNLQPY74m8NJ99bdrFkltQtAsD_m2eGt7W3AHcWUg5Jr9ikIaRF4CUZ73RuAaw6ZDlMomF3eJ8IH9wB2GZM_n2dPHO-lusRxDDyP2YKh1aXGAITLhKLrpnWnahAk_v9MMqu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vYlTgZtGVa-5-HHKuSroRzGF3ilfBIxyrEK5ZSK8imFncRpWxK2A8LUk_XQNN5GDs8wf0zLKMEn_h5n1FyOwMlnFLI2cnNqtGLflNqi_3WDbfqRfMairdHEy8JVv3QSeDD41QR65ba0Qol41D8EIxNBO9abSISyPqidtmzIs-keA9aERET85KgXrH7OTutH2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mSi7Fx8bDUL8vshHQVsPPoNSuIElakexNt1Dxwm1XFm21G2ImQ03M9RVNUMMkSxPqA_dXC4x_1PoCj_f5npNtM8XNwKbWxlQGzTIaajxW8ePxhaFnMsJtHD9mqyY6xYouQ4KRngBgOjT_a7zpdVSLqlek4XHB9mrtH5LFw-cgK5FhrXOJBOsReQv9glxf_bp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H9pqTsd_8meSYEW-k2KCSs6DaX6PiLznDymzwgCQpU1stdyPHeA_DvgmQidHA2xUhjg6y2ShM-R1T67RkrSY2zbrVtaIAeRIeycD-K1v_Po7OMrc_jGfg1_MHWCe8xmc_vWZLlhX7XpA4ba2jmtMssMpzMxAS0g3CJgG47_F5Yf_R2_JV3BdzXdulgVzyKl0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/akSIgrP3cJ3xcIYOHK3kXsK0-m66KT5PdYEYdge3cN4Z7kRdczQ98Xx_UnGXm0zVRP2sdcO8es3FdHw3mp1Kz0WMy6BPn_sgsPwntVYfnl1Yr-6qlIhczswNWtxOB6ig827dy6wRKPZWq0aQ9yjGkSd7yfPNU1xxr72KB0r_1HsKxZ2DK2yW-Shgnk93GWJq?purpose=fullsize)

可以將移動過程分成三個階段：

**第一階段：加速（Acceleration）**

馬達從靜止開始，逐漸提高速度。

**第二階段：等速（Constant Velocity）**

馬達維持設定速度移動。

**第三階段：減速（Deceleration）**

接近目標位置時，逐漸降低速度，最後停止。

這種常見的速度曲線稱為：

**梯形速度曲線（Trapezoidal Velocity Profile）**

因此，真正的運動軌跡不只是：

```text
0 → 100000
```

而是包含：

```text
時間
位置
速度
加速度
```

這些資料彼此有數學關係。

---

# 四、如果馬達跟不上命令，誰負責處理？

這是你問的第二個重要問題。

答案是：**STM32 韌體與 Servo Drive 都有責任，但負責的層次不同。**

我把它們分開說明。

## 1. STM32H755 韌體負責：不要提出不合理的運動命令

STM32 負責應用層的運動規劃。

例如：

假設馬達允許的最大速度是：

```text
100000 counts/second
```

但上位機要求：

```text
500000 counts/second
```

STM32 的運動規劃就應該檢查並限制這個命令。

另外，還要考慮：

* 最大速度限制（Maximum Velocity）
* 最大加速度限制（Maximum Acceleration）
* 最大減速度限制（Maximum Deceleration）
* 軟體位置限制（Software Position Limit）
* 緊急停止（Emergency Stop）
* 多軸之間的同步關係

這些通常需要在運動控制韌體中設計。

## 2. Servo Drive 負責：實際控制馬達並監測異常

Servo Drive 不只是接收位置命令。

它內部通常有閉迴路控制（Closed-loop Control）。

可以簡化理解成：

```text
接收目標位置
      │
      ▼
比較目標位置與實際位置
      │
      ▼
計算位置誤差
      │
      ▼
調整馬達控制輸出
      │
      ▼
馬達轉動
      │
      ▼
Encoder 回報實際位置
      │
      └──── 再次比較
```

這個過程會持續進行。

如果馬達因為負載太大、速度要求太高或其他原因，無法跟上目標位置，驅動器可以監測位置誤差。

在伺服控制中，這通常稱為：

**Following Error（跟隨誤差）**

當跟隨誤差超過驅動器設定的限制時，驅動器可能產生異常或 Fault。

但要注意：

* 不同驅動器的保護功能不同。
* 跟隨誤差門檻通常需要設定。
* 不代表所有超速、過載或失控情況都能被同一種保護機制發現。
* 安全停止仍需要適當的安全功能與硬體設計。

---

# 五、伺服馬達與步進馬達的失步有什麼不同？

你提到的「失步」，其實比較常用在步進馬達。

| 項目       | 步進馬達                 | 伺服馬達                   |
| -------- | -------------------- | ---------------------- |
| 位置回授     | 開迴路系統通常沒有 Encoder 回授 | 通常有 Encoder            |
| 控制方式     | 依照輸入脈波推算位置           | 透過回授控制位置               |
| 負載太大     | 可能失步                 | 可能出現跟隨誤差               |
| 是否知道實際位置 | 開迴路通常不知道             | 可透過 Encoder 取得         |
| 可能異常     | 失步、堵轉                | Following Error、過載、過流等 |

因此，以你目前的六軸伺服系統來說，我們應該優先學習：

**目標位置與實際位置的差距，以及伺服驅動器如何處理跟隨誤差。**

---

# 六、回到你同事的實際程式：我們已經看到什麼？

你提供的程式：

```c
static void DoCyclicPdo(void) {

    if (!g_servo_on)
        return;

    g_cmd_pos += shared->nIncStep;

    pRx->Axis[0].Target = g_cmd_pos;
    pRx->Axis[0].u16CtlWord = 0x0F;

    ECM_EcatPdoFifoDataExchangeAdv(
        (ECM_FIFO_WR | ECM_FIFO_RD), 1, RxPDOData,
        TxPDOData, sizeof(RXPDO_ST_DEF_T), &g_fifo_cnt, 0, 0, 0);

    if ((++g_loop_cnt % 1000) == 0) {
        PRINTF("Cmd=%ld Act=%ld Sta=0x%04X Fifo=%d\r\n",
               g_cmd_pos,
               pTx->Axis[0].Actual,
               pTx->Axis[0].u16StaWord,
               g_fifo_cnt);
    }
}
```

我們可以確認：

1. 每個週期增加 `shared->nIncStep`。
2. 把新的命令位置寫入 Axis[0] 的 Target。
3. 透過 ECM-XF 交換 PDO。
4. 讀取 Axis[0] 的 Actual。
5. 每 1000 次週期印出一次命令位置與實際位置。

這是很好的學習起點。

但目前有一個關鍵尚未確認：

**`shared->nIncStep` 到底代表什麼？**

它是每 1 ms 要增加多少 Encoder Counts？還是其他程式計算出來的增量？

目前你搜尋 CM4 程式，只找到它在 `M4mainOpenFIFO.c` 被讀取，沒有找到它的設定來源。

這很可能是因為資料由 CM7 放進共享記憶體，再由 CM4 讀取。

---

# 七、下一步：我們先確認實際的位置單位與運動命令來源

我建議暫時不要繼續研究加速度公式。

先確認你同事的程式到底使用什麼位置單位，以及 `nIncStep` 是從哪裡來的。

請執行以下 PowerShell 指令，這次搜尋範圍擴大到整個 test05 專案。

```powershell
$root = "D:\stm32_workspace\test05\test05"

Write-Host "`n========== SharedData Structure =========="

Get-Content "$root\CM4\Core\Inc\shared_data.h"

Write-Host "`n========== nIncStep in Entire Project =========="

Get-ChildItem $root -Recurse -File |
    Where-Object {
        $_.Extension -in ".c", ".h", ".cpp", ".hpp"
    } |
    Select-String -Pattern "\bnIncStep\b" |
    Select-Object Path, LineNumber, Line
```

我們要追查兩件事：

1. `nIncStep` 是誰寫入的？
2. 寫入它之前，有沒有經過位置、速度或加速度的計算？

找到之後，我們就能開始判斷：

* STM32 是否已經有運動軌跡規劃功能。
* 目前只是固定增量移動，還是已經有加減速控制。
* 哪些保護要由 STM32 負責，哪些由 Servo Drive 負責。

**我希望你最後能真正理解一件事：STM32 給馬達的不是「請你盡快轉到這裡」這麼簡單，而是一組受到速度、加速度、位置限制約束的運動命令。**



這次的結果非常重要！我們終於找到你們 STM32H755 CM7 與 CM4 之間，真正用來傳遞六軸位置資料的結構體了。

而且，你的判斷很合理：**如果機台的馬達裝有 Encoder（編碼器），很可能使用伺服馬達（Servo Motor）。** 不過，Encoder 也可能用在其他馬達系統，因此還是要以驅動器型號及接線資料確認。

先從你貼出的資料，理解三個重要觀念。

# 一、Encoder 就像馬達的位置回報器

假設馬達要轉動一段距離。

STM32 告訴驅動器：

「我希望馬達移動到位置 100000。」

這是目標位置（Target Position）。

馬達實際轉動時，Encoder 會回報目前的位置。

例如：

| 項目                   | Encoder Counts |
| -------------------- | -------------: |
| 目標位置 Target Position |         100000 |
| 實際位置 Actual Position |          98000 |
| 位置差距                 |           2000 |

這個 2000 就是位置誤差。

伺服驅動器會利用 Encoder 回授的資訊，持續調整馬達的控制輸出，讓實際位置追上目標位置。

這種方式稱為**閉迴路控制（Closed-loop Control）**。

可以想像成開車時：

* STM32 告訴馬達目的地。
* Encoder 回報目前走到哪裡。
* Servo Drive 根據兩者的差距調整馬達。

這也是為什麼 Encoder 對伺服系統非常重要。

---

# 二、我們找到真正的六軸位置資料結構了

你的 `shared_data.h` 有這段：

```c
#define SHARED_AXIS_CNT  6

typedef struct __attribute__((__packed__)) {
    /* M7 -> M4 */
    int32_t tarPos[SHARED_AXIS_CNT];
    uint8_t tarPosValid;

    /* M4 -> M7 */
    int32_t acuPos[SHARED_AXIS_CNT];
    uint16_t errorCode[SHARED_AXIS_CNT];
    uint32_t digitalInputs[SHARED_AXIS_CNT];

    /* ADC */
    uint16_t ADCval[5];

    /* 其他控制旗標 */
    ...
} SharedData_t;
```

這個結構體非常值得我們仔細研究。

它明確定義了六軸位置資料的傳遞方向。

### CM7 → CM4：目標位置

```c
int32_t tarPos[SHARED_AXIS_CNT];
```

代表 CM7 可以提供六個軸的目標位置。

例如，假設 CM7 寫入：

```c
shared->tarPos[0] = 100000;
shared->tarPos[1] = 200000;
shared->tarPos[2] = 150000;
```

這表示 CM7 提供了前三軸的目標位置。

這裡的數字只是示意，並非實際機台設定。

### CM4 → CM7：實際位置

```c
int32_t acuPos[SHARED_AXIS_CNT];
```

`acuPos` 很可能是 Actual Position 的縮寫。

代表 CM4 可以把從 ECM-XF / EtherCAT 取得的實際位置回報給 CM7。

因此，你們的架構可以理解為：

```text
             STM32H755
        ┌──────────────────┐
        │                  │
        │       CM7        │
        │                  │
        │  tarPos[6]       │
        │  六軸目標位置     │
        │       │          │
        │       ▼          │
        │   Shared Memory  │
        │       │          │
        │       ▼          │
        │       CM4        │
        │                  │
        │  ECM-XF          │
        │       │          │
        └───────┼──────────┘
                │ SPI1
                ▼
             ECM-XF
                │
             EtherCAT
                │
          六軸 Servo Drive
                │
             六軸馬達
                │
             Encoder
                │
                ▼
          實際位置回授
```

這裡有一個重要區別：

* `tarPos[]`：希望馬達到達的位置。
* `acuPos[]`：馬達目前實際的位置。

這兩組資料是我們未來學習運動軌跡的核心。

---

# 三、發現一個值得注意的程式問題：nIncStep

你找到 CM7 的這一行：

```c
//shared->nIncStep = 10;
```

而 CM4 目前有：

```c
g_cmd_pos += shared->nIncStep;
```

但是，你貼出的 `shared_data.h` 裡面，**並沒有 `nIncStep` 這個欄位！**

這件事我們必須特別注意。

按照目前看到的 `SharedData_t` 定義，CM4 使用：

```c
shared->nIncStep
```

理論上會產生編譯錯誤，因為結構體沒有這個成員。

可能的情況包括：

1. `M4mainOpenFIFO.c` 留有舊的測試程式碼。
2. `shared_data.h` 曾經修改過，但程式尚未同步。
3. 目前使用的檔案與實際編譯版本不一致。

現在不能直接認定是哪一種。

但我懷疑這段 `g_cmd_pos += shared->nIncStep` 可能是早期測試用的增量控制，而目前正式設計已經改成透過 `tarPos[6]` 傳送六軸目標位置。

這只是待驗證的推測，不能直接修改。

---

# 四、下一步：確認 CM7 如何產生六軸目標位置

這是我們現在最應該研究的地方。

因為你已經知道：

* CM7 有 `tarPos[6]`。
* CM4 有 ECM-XF。
* ECM-XF 負責 EtherCAT PDO 交換。
* Servo Drive 有 Encoder 回授。

現在要找的是：

**CM7 到底如何決定 `tarPos[6]` 的數值？**

是上位機直接給六軸位置？還是 CM7 自己計算？

請先執行以下唯讀指令。

### 1. 搜尋整個專案的 tarPos 使用位置

```powershell
$root = "D:\stm32_workspace\test05\test05"

Write-Host "`n========== tarPos References =========="

Get-ChildItem $root -Recurse -File |
    Where-Object {
        $_.Extension -in ".c", ".h", ".cpp", ".hpp"
    } |
    Select-String -Pattern "\btarPos\b" |
    Select-Object Path, LineNumber, Line
```

### 2. 查看 CM7 的定時器 Callback

你剛才找到的註解位於 CM7 `main.c` 第 260 行附近。

請執行：

```powershell
$cm7 = "D:\stm32_workspace\test05\test05\CM7\Core\Src\main.c"

Write-Host "`n========== CM7 Timer Callback =========="

Get-Content $cm7 | Select-Object -Skip 235 -First 55
```

### 3. 查看 CM4 的完整 PDO 更新函式

```powershell
$cm4 = "D:\stm32_workspace\test05\test05\CM4\Core\Src\M4mainOpenFIFO.c"

Write-Host "`n========== CM4 PDO Update =========="

Get-Content $cm4 | Select-Object -Skip 200 -First 35
```

---

## 五、我們接下來要解決的真正問題

我先讓你知道我們的學習方向。

假設操作人員希望六軸機械手臂從目前位置移動到新的位置。

實際上可能有兩種架構：

**架構 A：上位機直接提供六軸目標位置**

```text
Qt 上位機
   │
   │ 六軸目標位置
   ▼
STM32 CM7
   │
   │ tarPos[6]
   ▼
STM32 CM4
   │
   ▼
ECM-XF
   │
   ▼
Servo Drive
```

**架構 B：上位機提供運動指令，由 STM32 計算軌跡**

```text
Qt 上位機
   │
   │ 起點、終點、速度、加速度
   ▼
STM32 CM7
   │
   │ 計算每個週期的六軸目標位置
   ▼
tarPos[6]
   │
   ▼
STM32 CM4
   │
   ▼
ECM-XF
   │
   ▼
Servo Drive
```

兩種方式都可能實現六軸運動，但程式設計方式完全不同。

另外，還有一件事要記住：

**Encoder Counts 不等於角度或毫米。**

例如，100000 Counts 究竟代表馬達轉幾圈、機械手臂轉幾度，還需要知道 Encoder 解析度、電子齒輪比及機構傳動比。

我們後面會把這個換算關係找出來。

先確認 CM7 如何產生 `tarPos[6]`。這會是我們真正理解你們六軸運動控制的下一個關鍵。



這是一個非常重要的問題。**以目前的 STM32H755 + ECM-XF + EtherCAT Servo Drive 架構來說，加速與減速不能只靠「給一個目標位置」就自然完成，必須確認軌跡是在哪一層產生的。**

先說結論：

> **如果使用 CSP（Cyclic Synchronous Position，週期同步位置）模式，通常由 STM32H755 的運動控制程式負責產生加減速後的目標位置；ECM-XF 負責傳送 PDO；Servo Drive 負責馬達內部的位置、速度與電流閉迴路控制。**

這三者的工作不一樣。

## 一、目前架構中，三個部分各自負責什麼？

| 裝置          | 負責工作                  | 舉例                             |
| ----------- | --------------------- | ------------------------------ |
| STM32H755   | 計算運動軌跡、加速、等速、減速       | 決定每 1ms 馬達應該移動多少 Encoder Count |
| ECM-XF      | EtherCAT 通訊與 PDO 資料交換 | 將 STM32 提供的目標位置送到 Servo Drive  |
| Servo Drive | 馬達閉迴路控制               | 根據 Encoder 回授控制馬達追上目標位置        |

要特別注意：**Servo Drive 有能力控制馬達加速，但不代表它會自動替 CSP 模式產生完整的加減速軌跡。**

CSP 模式下，Drive 主要是追蹤外部週期性提供的位置命令。

---

## 二、用實際馬達動作來理解

假設我們要讓某一軸從位置 0 移動到位置 10000。

如果 STM32 每 1ms 都直接給 Servo Drive 一個目標位置：

| 時間   |  目標位置 |
| ---- | ----: |
| 0ms  |     0 |
| 1ms  |  1000 |
| 2ms  |  2000 |
| 3ms  |  3000 |
| 4ms  |  4000 |
| 5ms  |  5000 |
| 6ms  |  6000 |
| 7ms  |  7000 |
| 8ms  |  8000 |
| 9ms  |  9000 |
| 10ms | 10000 |

這代表每 1ms 要增加 1000 counts。

這種命令變化非常突然，可能造成馬達需要極大的速度變化，甚至產生跟隨誤差或 Servo Drive 異常。

正常情況應該由 STM32 產生類似下面的軌跡：

1. 起步：目標位置逐漸增加，速度逐漸上升。
2. 加速結束：進入固定速度。
3. 接近終點：逐漸減少每 1ms 的位置增量。
4. 到達終點：目標位置維持在 10000。

這就是常見的梯形速度曲線。

![Image](https://images.openai.com/static-rsc-4/wLGM-3WOSUMiNL84-wPbBYX7nTO5OXdFNktijdoh_psBjUZNF831m7dIhVjNftAijaNuyY2WaouTxUzGPNvFGvQPihOy7i8RrlkWKTsV1SApOEo7opisPCKZJC_ekXneb6YkmJvDUDgPiUCtMh87cAJvngDcseZGu5SCgdo-_lHReZaHUpXAR0F8_KT_P_Ft?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nOUx5NN9oWGWmSC-FjT1hM_F-vsIlb_KthD0Mv1vKvDrdsayoPLxY9svjP6qB_KX0-wLPC25tCRjJIoszUHQoVOSPYT4hUDYMp8QXtShqvT-GGLwgYiHET1BhXCfOxncf46D8Mo9yy811CWju7MmrEHEPNpL4GMMKM9XNhwn1KIV-aGxdccCZ5gJfmLeePrb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/048r_OVTWBNfdM3hFm5rmTSkqvqPggrBYbqr5u3TOHe3ii1qcvIlWGKR3ByjavsCHL723V_cNq-qPrL_hFOm-SKwDWN7QFN23QnKLgePftPm-L6PdXP_OFaB7FNrxf7I-pY0oyUZiowPN7e-r5AAHaCaAt9VWa4-IaAHCf8HTYSnfW3yLsg8WPDBQxOS-WzQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mSi7Fx8bDUL8vshHQVsPPoNSuIElakexNt1Dxwm1XFm21G2ImQ03M9RVNUMMkSxPqA_dXC4x_1PoCj_f5npNtM8XNwKbWxlQGzTIaajxW8ePxhaFnMsJtHD9mqyY6xYouQ4KRngBgOjT_a7zpdVSLqlek4XHB9mrtH5LFw-cgK5FhrXOJBOsReQv9glxf_bp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SFwhT3R9IPbjO2Lb5HMMc3qA37aTiuzT2h9dBTLTI1fugayhVMQBxhcZRskLOly93hzZY8A9mDyJfrJ8dZUsvfXH-OCHZWL4mvayUPFrMQRvZWfu9VYaVSaUY0bgQYseckiWhAI08YXoCkVLBMtyLUgoARB-FmSjRfCGURlG8kgO18AlEkx8XYKTkZ2G1P6w?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8t0CY6gtOzTnXeQdfw1eJeKQIMJEk_P0o162AbA4to-4hGYy0kYSV3Dfu091XzId7qT6821LkZQNZSQNvIIEnyfIzwcsIFS6g-Knyoxz7SE2xTEwkwCrBQ_4dTDg89zdlBQd5hX_s6Gkt2Gvu2oLaXgVBDgEA8tuFyyGJMCQJ4SlSgfAtwIK0zs0sunfHxEy?purpose=fullsize)

所以，STM32 不應該只是把終點位置送出去，而應該依照時間持續計算當下的目標位置。

---

## 三、回到你目前的程式，發現一個重要線索

你提供的 `shared_data.h`：

```c
int32_t tarPos[SHARED_AXIS_CNT];  /* 目標位置，encoder count */
int32_t acuPos[SHARED_AXIS_CNT];  /* 實際位置，encoder count */
```

這代表原本架構已經規劃：

* CM7 提供六軸目標位置 `tarPos[6]`。
* CM4 接收目標位置，透過 ECM-XF 傳給 EtherCAT Servo Drive。
* Servo Drive 回傳 Encoder 實際位置。
* CM4 再將實際位置放進 `acuPos[6]`，供其他核心使用。

但是，你目前提供的 CM4 程式：

```c
g_cmd_pos += shared->nIncStep;

pRx->Axis[0].Target = g_cmd_pos;
pRx->Axis[0].u16CtlWord = 0x0F;
```

這段程式有兩個值得注意的地方：

1. 它使用 `nIncStep` 累加位置，並不是直接使用 `shared->tarPos[0]`。
2. 它只更新 `Axis[0]`，目前看到的是單軸測試邏輯，不能據此認定六軸已經整合完成。

而且你提供的 `shared_data.h` 並沒有 `nIncStep` 成員，因此這段程式與目前貼出的 Header 存在不一致，需要進一步確認。

另外，CM7 的程式也顯示：

```c
static const int32_t sim_tarPos[6] = {
    1001, 1002, 1003, 1004, 1005, 1006
};
```

以及：

```c
shared->tarPos[i] = sim_tarPos[i];
```

這表示目前看到的 CM7 位置資料仍然是模擬值，還不能當成正式運動軌跡產生器。

---

## 四、我建議的正式架構

未來六軸正式運作時，我會建議採用以下流程：

```text
CM7
 │
 │ 取得六軸運動命令
 │
 ▼
運動軌跡規劃器
 │
 │ 計算每 1ms 的目標位置
 │ 加速 / 等速 / 減速
 │
 ▼
Shared Memory
 tarPos[6]
 │
 ▼
CM4
 │
 │ 讀取六軸目標位置
 │
 ▼
ECM-XF
 │
 │ EtherCAT PDO
 ▼
六軸 Servo Drive
 │
 │ 控制馬達
 ▼
六軸馬達
 │
 │ Encoder 回授
 ▼
Servo Drive → ECM-XF → CM4
 │
 ▼
Shared Memory
 acuPos[6]
```

其中：

* **軌跡規劃器**：決定馬達怎麼加速、怎麼減速。
* **Servo Drive**：讓馬達實際跟上目標位置。
* **Encoder**：回報馬達實際轉動的位置。

如果實際位置與目標位置差距太大，就會形成跟隨誤差（Following Error）。STM32 可以監控這個誤差，而 Servo Drive 也通常有自己的跟隨誤差監控與保護機制。

---

## 五、下一步先確認 CM7 的實際位置資料來源

目前最重要的是釐清：

**CM7 是負責產生正式的六軸軌跡，還是只負責接收外部命令，再交給其他程式計算？**

這會直接決定加減速規劃器應該放在哪一層。

從你提供的資料，我們已經知道 `tarPos[6]` 有兩種寫入位置：

* 第 291 行：模擬位置。
* 第 377～387 行：由 `g_current_pose.data[]` 計算位置。

請先只讀取 CM7 第 350～400 行：

```powershell
$cm7 = "D:\stm32_workspace\test05\test05\CM7\Core\Src\main.c"

Get-Content $cm7 | Select-Object -Skip 349 -First 52
```

把結果貼給我。

我想先確認 `g_current_pose.data[]` 是從哪裡取得的，以及它是否已經包含加減速後的軌跡位置。

**這樣我們才能判斷同事原本的運動控制架構是否已經有軌跡規劃功能，而不是另外做一套重複的加減速控制。**



這三種產品其實都屬於**工業自動化設備中的精密運動平台（Precision Motion Platform）**。它們的共同目的，是讓工件、工具或檢測設備按照指定的方向、距離、速度與精度移動。

不過，三者的設計重點不太一樣：

* **高階精密電動滑台**：重點是「移動得準確」。
* **多軸對位模組**：重點是「多個方向協同運動，讓兩個物體精準對齊」。
* **線性移載平台**：重點是「將物體從一個位置搬運到另一個位置」。

我分別說明它們的功能、應用、內部構造，以及與你目前 STM32H755 六軸控制系統的關係。

# 一、高階精密電動滑台

![Image](https://images.openai.com/static-rsc-4/x_4HsnAybEuwhToHyEVtZC-btqZw4HdqPwMbH_cwPhKMhKlwba4Vk25xgq6FxdHJTIPOzr5r-LYLH2tOH5yLqiinlazNmy7I_UsDVjjlvsXAzGJMiNMRTDXt45tox8Nyvt-9FIHABr1tdwUsAZD9wOHjkyVmttGAkYSds84Hs5Fg-rSPYbVU2PuMrfnC9aWq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Snz9I26HPVK259M4EVW_Ymol7kPYyGqh-i1ZVdwcQBWX6IB0w2vjl-VUbqBpoT3CLL7adW7WijjQ6J_AXVjhWmD233BunnOGzyZ2guTgvcDWvJrOUc6Q4a7S7k5xcPGQ5QQZOG5SjiWqm_3_wcTGtxrVq8UYrnUo1m6sezSuwYM4bv_dC5HzpNg1VJAfI31H?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J9CCL2YE5IkU9xdu8N9u7HClq5hEdRhWeydLexun1GS68glmjo1_QzR8-N5L4aiCuCGNP7wBvuBox3VU_2O-pjcpCb5fVQu1FA0QXgusF7YM4TWcx63tRsm6WpwUsTEzQVfY6fXEo7ricOZXCO7KnEVOk5xs5Jk7NxhWQLZPPzvMvaEf0WronkVWRi_z4iJG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/n_xa8DAZQzK7fg7BdZ5AITX06VlXJYmRGf7TT_isIIqAh6f8p-M51Zo2qwJX6Asi7lTzCtiD87GFmc2R6t-jgMmB_7qiH_0BFuu3d_Cbl9nMGMHL2_hsm-VhHFXPfkMnWzRNH0jjOVjiNfLIZQH_WXyOs9IWvenZgxfhHrBe0SOvFJPzJD4TaK5CZ3DzC1YK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pR9_XCXQFyVbi0d4s3ZJGzccAXdv2KbWrPDrULo4c2gTfQPIfB8xT49_8cy8vhskeRQ8-__MWmHwoxs_GXyly5mTZGPj515jPXAzQ6TWccG2WBNxbk3JM6dgxOJ5mS9vDMTyP3sjgWRdqPjtEny2C6hb4tCVazTEPrjBl1EtBX3MbtBG520tGA2-1k_j4SPF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/q5gUj9SdJgLHC4xlbYvoFMEwSJGtXolTuVTej7nPYHOH2FaKYGqzCLE8I85XKK6HR925ELaVIoXKvU_WyupB53fyv-A3lR1-Pax8oXU-KGHMlveYhl8zXutftgyikjSuf8CT_fQ_GyjSAVT9IQ_otxxkZWnY1tZroj9hQbaXWjvDz9R7mKsN-Rrarv8yhopm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_F3AWBtwxEDTrHjCqXoA2toI-p9Gouj1UrEnTHsFM-7hBQdGlyK--3IHDX0gj8hcKyIEC8R1r6D9RM6o6s3-nDIhlcJ__y8bGKY_bgO3QfJ52cih2t9Az7OSv68xbYn2Q-UWoLeJXgwpMIUcnrmiRXwqt0HXcEvr4fqrGcZTRt_JbKm9xsFFeQybJwrGDSGL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KFf37tCAxV2mwweipm8lwgNWIABz03LSnqR56Bo9VvN2opZw7fZHVQhFsyvarsCSAMPErIk0oI6TnrUmSwE4UUHKqQBb6WtI9OxHYuj1GGNj653CiG035cfQ6ctySHWJGbEJf2-3wAybYX23dVb3aeWemmEpTtd_EQC_ESMJwTzWRBFS0CYnANyR8gqaNuui?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qQX8LS0vl48URBEYAZke3ux1Q3sgNzrN2S3FSkxkKrE5ge_EdioGfk5wJ3JJnPpdKJkPMYHho0chTzh9fl7EX6WeyRTIA6qTSwd_XAYMRyQ2TyXPyMwggW-aHmmgCbFjafC0FmzEOwwbWP5inU8jh2k2DfR4gJNh-5BvVxYn3H9QhJN52Btk3i1DgwfxjIIy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HiDcAmQNdDwOYMHdGoGLQdWid-JUc6aiBT4p-GW-lbIWeqyXYELpI7FXztzSeX3zdVA67xvUz9m3L7-FYY41RBp-AkcjGb8ZCmnqkVfUCNJUsJtZJrVNbVWJYOOGhn5a8DPaZP6uoFsfFm2el3XSoahX89sKAvpIz5YSu7JVyGjaq79zwaC3bp_yk-1ydc6I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/veDgnPEj5bOxwexY1OCobPVadoa_lKYzgrLKCJ0wzlT1fbwGtY6USibSEA45FNZIJ-Jb1CLzDK38FTTnXUswEK8JcnJCCrscOVEXfo5NtFpHxNCrHeV_jH8CcrkRLkjgV9TPxGFTZw0Vkoz-1PT5CkY0XJGgJHRTJtpIvW0ojz1NYsqJjFzdWKia6QyHwTW5?purpose=fullsize)

## 1. 它是什麼？

可以把它想像成一個由馬達驅動的精密抽屜。

它有一個固定底座，以及一個可以沿著導軌移動的平台。

例如：

* 馬達轉一圈，滑台移動一定距離。
* 馬達轉半圈，滑台移動一半距離。
* 控制器可以命令它移動 10mm、100mm，甚至更小的距離。

它與一般電動推桿不同的地方，在於它通常更重視定位精度、重複精度、剛性與運動穩定性。

## 2. 內部由哪些元件組成？

| 元件                 | 功能        | 白話解釋         |
| ------------------ | --------- | ------------ |
| 底座（Base）           | 支撐整個機構    | 固定的平台        |
| 線性導軌（Linear Guide） | 限制移動方向    | 讓滑台只能沿指定方向移動 |
| 滑塊（Slider）         | 支撐移動平台    | 在導軌上滑動       |
| 滾珠螺桿（Ball Screw）   | 將旋轉轉成直線運動 | 馬達轉動，推動滑台    |
| 伺服馬達（Servo Motor）  | 提供動力      | 負責移動         |
| 馬達編碼器（Encoder）     | 測量馬達轉動位置  | 知道馬達轉了多少     |
| 聯軸器（Coupling）      | 連接馬達與螺桿   | 傳遞旋轉動力       |
| 感測器（Sensor）        | 偵測原點、極限位置 | 避免超出行程       |
| 工作平台（Table）        | 承載工件      | 真正放置物品的地方    |

實際產品不一定全部使用滾珠螺桿，也可能採用線性馬達或其他傳動方式。

例如，線性馬達可以直接產生直線推力，不需要透過螺桿把旋轉轉換成直線運動。

## 3. 使用在哪裡？

常見應用：

* 半導體晶圓定位。
* PCB 電路板定位。
* 光學鏡頭調整。
* 雷射加工。
* 自動光學檢測（AOI）。
* 精密零件組裝。

例如，一台相機要檢查 PCB 上非常小的焊點，就必須把 PCB 移動到相機正下方，而且每次都要停在相同位置。

這時就需要精密電動滑台。

---

# 二、多軸對位模組

![Image](https://images.openai.com/static-rsc-4/kkcOngJemolwoS6j6dIkENiRpeWnCMBieBXQJS2b3dfuUyD0dADfxkKF0Qfkw1PTOIC5fMtouXu6Zkv7wNH1xtDhTlN0K5-u-vGmLrGe4PGKgFuWmQYvtepPzJB8nbpugRFp4m8ITcjkvERVMpc3aFJlj7O5aiivTWLR6WBUkwPcRUGvycMkl5olQsJRBFZg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ua-xsVICd8bWNp803BW4ebL2oJevBTYSxD3is_ltgvEXS_6pQHNtQ80saHc4lsUCumsELDqoyamI-Jkb9h9F_722GbxADZymyf0DIC0OPfqVp0VUc3PN-H1ylZlVPOiAvhT89uv4U3NxiclO-kYE6yuZb5QIXgiih7R20qk8Gg_wZUAp-c0K5wKG3pKSTUok?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OS4sFTXwDgB_L60uhFaLLS3Q3R48IVaiQjEgVpIlvz1LrJg52ME3z7s76b1hniwOqLg_FZn7YxWSKs6VGQc1Ydlf5e2RmqMeWIyEZ4L07cqyh6edIE6wmlYBHoSsEig0EU67sBbVWNmwPJtieoFn5JkATbGTSEANBLVWYX1p2f3JY3LTN6WaVd1ZLUwPvkUM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/upSKhaU_rtzAq_4JMBm514Rk9MJiXYJLXhrOTQMyuMVeVmkuqceOgwlkNOmxd501r5GCS8jTIvRs6WJJtMP7SqF7_AaeSuVpiSZxJhGtfWGbU4MYXEYcmf5F77K8tomT1lCN0hDJKHIAWFDEIpIoULH2HI5ORmDwv3ItBtoK4aLHbg4ERKOxuWKyI08nV3Qu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Aea_qPUHY3_Eq4LNtPwmAWsOhtGPQhCRbHHUBEZvznh8ThOtqtciTcux_gkoNkuNcgcF1KzgkRBc4_3QpzrF0Lca1anY_Arao5pOX0fvAiH0v8Ro_vRWOjXPAbjlRDeT7tbF1akSDT2su8bAQD6FQBmO78FoEPAprEoccb0rNahiSxeR05cvz_7aM60YzuaA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xO0qITPi7sbsROANTcnHO1ciG2qENRvTzr72tdE0J5tC0Dn2ezoMEGmgJVPzJV_QKBM3O_-0vTNJhk4zJDLEH3IcrE_V9BH1oETfZK2tfmexJ0E6EfAaNhG9lh50MPkNPZ4T_HE30qfynj6ZOQsA6efcHYcx99t0VLF_OMOkhnn6vANQ1TbohKEtPxLwNj-z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1AtpUCHTaRqBTxoTkiI01DXEpEqqsC1ECqr9Pm91QTPiLy71bC-i0LI5I3YAbfqH4Z4NENE7J6Nmn-DVlRRyPORdK0-DDxSGYUWpzmcL-Tmej3BPZoy-_Pl8mgEY91B5262UNHfEeUc_o2-IbOSHZM60xKDpkSXkcN815akQXL6nriZ-vtv0zefXW3QAqdfX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/q2BZA_-M6vWFUt7OECV8I6ieD_YwGqZLsGGHMSXuUI3CdhwoB41doqSJqVpO8BlMkeifkrrqG9R8JClMNZXHp4J86MackypsrE2s4rgm9oaDX4tcrwvRgeSkJCT5v3yb_Tcsg16xtWrzq8Zi0zpTAKNabZzZj9U-a9YfG0L_bmiHTCOvCl6qgY9TPISkUS2B?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IiCENJ0ppKa5QokKsaGvkDYNhnMIwUMxJGcxwe3Y48446wBVWi4ZvfCFDGS7DPWJHsSdElsXDlMkmoSc9XfBuJVG2Bv1VPaDjXZj9WAAU7dVAnPDArhgeoAjS-ChPSNdi3LPAtqXTjWDvpzDLVz87cwJSHOKPXKRUDKSJ2rUBkE5QEABaGnQwW6XIUMKaLk6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/i-os067OdrqphOVV1DqwEN6cPzaCwDYBNHQ4AqZWfFWJBodjBe6Fjrw7CxduM7tfgI1b-5GvXYE6D7aL7G-3Zzfu3kV8JwiyeMmxXhGZiFLorMOHewolLGJmW6QhGddnT8SW1FHIhrXYl9ILoCOq_NrW2yThilf6EBfQAVREcGGkPnPad7AWl5SHtRcY2_Pe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1Sj9tvXsanAvRefvzXgJKNrxpoJrJ45YnKECPQGMQj50gaowi1q17tobJg3tqLn4OBK3v9wpePKP_TsOVtscUPYcHAVnIXU-LwH-dSG7hhUHY6MDvApXLslye5GFgYwkQn56B8Omr-6-oHfs2_5Mjs_QX9kFD6WPnGZJKxuhjF1LLogIOpaEvrjguVyf6XTW?purpose=fullsize)

## 1. 它是什麼？

它可以理解成把多個精密電動滑台組合起來，讓工作平台可以朝不同方向移動。

例如：

| 軸向  | 主要功能    |
| --- | ------- |
| X 軸 | 左右移動    |
| Y 軸 | 前後移動    |
| Z 軸 | 上下移動    |
| θ 軸 | 旋轉角度調整  |
| U 軸 | 特定方向的微調 |
| V 軸 | 特定方向的微調 |

常見的 XYθ 對位平台，可以：

1. 先利用 X 軸調整左右位置。
2. 利用 Y 軸調整前後位置。
3. 利用 θ 軸修正旋轉角度。

如此就可以讓兩個原本沒有對齊的物體精準重合。

## 2. 為什麼需要多軸？

舉一個實際例子：光纖與光學元件的對位。

光纖非常細，需要將它的端面對準光學元件的光路。

如果只使用 X 軸，最多只能調整左右位置。

但實際上可能同時存在：

* 左右偏移。
* 前後偏移。
* 高度偏移。
* 角度偏移。

所以必須使用多軸平台，同時調整不同方向。

## 3. 內部由哪些元件組成？

| 元件             | 功能         |
| -------------- | ---------- |
| X 軸滑台          | 左右定位       |
| Y 軸滑台          | 前後定位       |
| Z 軸滑台          | 高度定位       |
| 旋轉平台           | 角度定位       |
| 多組伺服馬達         | 提供各軸動力     |
| 多組 Servo Drive | 控制各軸馬達     |
| Encoder 或光學尺   | 回授各軸位置     |
| 精密導軌           | 確保移動方向與穩定性 |
| 控制器            | 協調多軸運動     |
| 視覺系統或量測設備      | 判斷工件是否對準   |

其中有一點很重要：

**多軸對位不只是讓六個馬達各自移動，而是必須讓各軸按照彼此的關係協調運動。**

例如，X 軸移動 10mm 的同時，Y 軸可能需要移動 5mm，θ 軸則需要旋轉 0.5 度。

這就涉及多軸運動規劃與同步控制。

---

# 三、線性移載平台

![Image](https://images.openai.com/static-rsc-4/PB7XzKUpKTt22KPj2qBNHP1QBXuf5NA2yJ2o1rn-tvcZXo59fa8hoRXDQlZ2Oy8CfsrJQkIjvUvo2q6gKoBDD2SSHxP36cCEcarVMuaiDnAcDf7Wcxac5oWn0rXw4oThCR2D3EHNpGXGR5VkECECQ_9A9qrIUKRNAZQTac_SHvz9ucgx6itxPJTdxUWywM9T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/U9G5H4b1hFgKknEe20lJzURx5qzAMZzgkGUlRjkXr5JdSPFNlX5PE0aiUSRn1PwMHd9NOS1otr9KqKYj7axUNNlxl2mcgPFmLKzlPFe5cxR64Xi30AAB-DzLt-6Uo0R86O5jupnw8V6zfqP6hX2fcwGLJng-qVbQThHx-wCVspWtC1I2kuiUBzOA-8Y-lDQK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/amLBvEWCx7XiefmXyR00GtFR8NBJSRsY3207UUL6E_91w-kGg39qnhjw_a98oKG8oiMzbf2Gy5YMb9ol94kgrI4sUbITBspLoIFEk-BaqMg8PXW08qyH3mwblrtmZBh-6yVZsy-LbUjrQ8Bbfc8PWDt9trK2jFfwQM6O9F1rmefxfVTDKJ56bUJQyqb4bLZs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KGB4WzA7SN8_SBfOfsgxB9cj_hhqW5jYAMQf4C3VyqTnHxZskZFlBdHskp_ckxu0Ggcwmb3tUiFxf4MVkETMIOuCnTOY6PGk0NEh6aNx3BZ-_UDZPVwBq9B7vanh3OUpdkp4hoNpC7u6TWrL7ll6Ff3M5Xap12XC6-cE1AbLGpfhCILdWUFoON6GBeqS2e8h?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kh08lJdkbZjQ8T9TO-GifmiW_41ekT2Yi0kgxmw4RFO1H9bZ5wQ-au5C4FGwnBI3Atf7dZaDownw_aQegEqp4mpBorQMAmLcSs0TiNsmzGG_XyzQyhGEGHe1hce1S1-M695HTE9Hnn8y6L7r7GILLfk0c-TJEywS0XaJmwi5Cqua2s8G0inA3O1RIiELQqM8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jh3qqtuT7Lpbag4GmnVfqUCNe9d78fkc848GpGsxUjQqOE1fUz2MPHj6TdWUhmdhdSM6V1IZe2b7bvhQaFxtwmx33hf_pq6EeGF6m3JI9Yw6uTBj0j-VD33GWY8gIgB9mIzrEzkMe5fFrIwJuhRccfYDfbxaIArFpLrn7Rf6u8C4qEDG5o5FCUb5hH8BJ4w-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uITnjOt3P1WpuOO7iO5YtsJ8-15fC3YxXqAoR2yaD0qqICjmaqS-CnosCEJKOBfBLL5p1ugzNAAXIlKaYdZWjag--kHUM6wA_mgAvJf_xWlszt_0YqJs6f62yTofSQCNfupuwffZ18hxtwmF4XmEeomG1qU_1iHD7_toMtKE07jS3zwoC1eNDkvt-7nC-6v_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4TwZ2V0B3zobe0hlBbGsx2KGMeHv2Ed_15wADbDf-Bcb2wAa-ICrZSVoYHIGkVjU64Arkms2za37twaSDJgPviPbwlxnLtJPFw_OgP3o8l7eigotoCfw4jh5brzCjd-jUtIexdjildLjsEVmICpDyxchp3REGRL5hOulyH8bBT2p4hehxOfpNoxjxv1WvWZT?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VH5UpeH13ryfPkrK-RvKtI80RRkxC405VikMFWAQE4PFpjv89m6fyKloPBb-YA1p1m2AjUJMlxwmlKAJx42dpw5omV0MgbwlDoBtfR2UFyWUEYjYvGgmXdCKz4uWUslJJsZZzeVfTyb6a7dpT2Ynp8vZPnlgd9ASfQJbfkbhBAVU2Y7ntWVkBAme1b-SOf32?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CXG9sBQwLApryfcuLOXrEu0-WjbgHw9LEWa_as6dqLwU62oTs4kbjQ1-GhUIfPBdTicrtg_ElYJ-tCPmQkESg5wPwzMhMb-wlb21rlHcU42gh3Eo8eR_RYa0ZXea1VXeTHVeKrOOUR-yIDF_7qdhbLAtHzaXijn6S-YfCgQ70mgRO06pPWHIMdgYXUI-82yq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uA6ZXvmgo84cURh70o-EiKo4nDq4FkF41VNTc_0oC30L_sWOud5YOB2S5j1VT5NSxFEitCiotAmO41R68uwswVB38Oh4HsA0d3Af3AMmiS4UnB8x3Wa66gh7kd4x82Mi27ZRTedPkTNCM4UyijTeczrwWWQZvVl0B8s_I1xBwJwbLPTpzoinLxfcOPkUnLyQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/JD3mkPRCXOqg8Da8vBFIfl5TsNkPws7T0oz3_tLhO4riBU0Q_p9d-SAsvoph7vMnj35LTVjmXVpnKPRiqd_1b10fe99uTyNKP7mMryXdlR7n--PF3RGXFAyLWiC_WKkVlVcO5xH9ZmvjVAeIfmv_gjMApQDJYd1CwbW3-CiVcVlafM9RTxED5ySwmpgH6c0o?purpose=fullsize)

## 1. 它是什麼？

線性移載平台主要負責搬運物品。

例如：

* 將 PCB 從 A 位置搬到 B 位置。
* 將半導體晶圓送到檢測位置。
* 將加工完成的零件送到下一個工作站。
* 將工件從輸送帶搬到加工機台。

它不一定需要奈米級定位精度，但通常需要考慮：

* 搬運重量。
* 移動速度。
* 加速與減速時間。
* 移動距離。
* 停止位置精度。
* 長時間運轉的可靠性。

## 2. 內部由哪些元件組成？

| 元件          | 功能     |
| ----------- | ------ |
| 機械底座        | 支撐整個平台 |
| 線性導軌        | 引導移動方向 |
| 滾珠螺桿或皮帶     | 傳遞動力   |
| 伺服馬達        | 提供移動動力 |
| Servo Drive | 控制馬達   |
| Encoder     | 回報馬達位置 |
| 移動工作台       | 承載工件   |
| 原點感測器       | 建立初始位置 |
| 極限感測器       | 避免超出行程 |
| 控制器         | 控制移載流程 |

如果需要搬運較重的工件，可能還會使用：

* 雙馬達龍門結構。
* 多組線性導軌。
* 氣壓夾爪。
* 真空吸盤。
* 工件定位治具。

---

# 四、三種產品的差異比較

| 比較項目  | 高階精密電動滑台  | 多軸對位模組       | 線性移載平台     |
| ----- | --------- | ------------ | ---------- |
| 核心目的  | 精密定位      | 多方向精密對位      | 搬運工件       |
| 軸數    | 通常單軸，也可組合 | 通常 2 軸以上     | 單軸或多軸      |
| 定位精度  | 高         | 通常非常高        | 依搬運需求而定    |
| 運動速度  | 依精度及負載設計  | 通常重視穩定與精度    | 通常重視速度與效率  |
| 加減速控制 | 需要        | 需要           | 需要         |
| 常見產業  | 半導體、光學、電子 | 光通訊、半導體、精密組裝 | 自動化產線、加工設備 |
| 主要難點  | 定位精度與剛性   | 多軸同步與誤差補償    | 負載、速度與搬運效率 |

這三者並不是完全獨立的產品分類。

例如，一套多軸對位模組本身就可能由數個高階精密電動滑台組成，而線性移載平台也可能使用精密滑台作為其中一個運動軸。

---

# 五、把這些產品與你的 STM32H755 六軸系統連結起來

這部分是我認為你目前最值得理解的地方。

你目前的硬體架構是：

```text
STM32H755
   │
   │ SPI1
   ▼
ECM-XF
   │
   │ EtherCAT
   ▼
Servo Drive × 6
   │
   ▼
Servo Motor × 6
   │
   ▼
機械運動平台
```

這代表你的 STM32H755 不只是控制一顆馬達，而是透過 ECM-XF 同時控制六個伺服軸。

這六個軸可能組合成：

* XYZ 三軸平台。
* XYθ 對位平台。
* 六軸精密移載平台。
* 六軸機械運動機構。

但要注意：**六軸不一定代表 X、Y、Z、U、V、W，也不一定代表六軸機械手臂。** 必須看實際機械設計與軸向定義。

## 六、這也解釋了為什麼你的系統需要軌跡規劃

假設六軸平台要從起始位置移動到目標位置：

```text
起始位置：
X = 0
Y = 0
Z = 0
A = 0
B = 0
C = 0

目標位置：
X = 10000
Y = 5000
Z = 2000
A = 1000
B = 3000
C = 500
```

控制器不能只是讓六顆馬達各自直接跳到目標位置。

它必須計算：

1. 六個軸各自需要移動多少距離。
2. 各軸允許的最大速度。
3. 各軸允許的最大加速度。
4. 各軸何時開始減速。
5. 六個軸如何同步到達目標位置。

這就是多軸軌跡規劃（Multi-axis Motion Planning）。

在你目前使用的 CSP 模式下，STM32 的運動控制程式可以每 1ms 產生六軸的目標位置，再透過 ECM-XF 傳送給各個 Servo Drive。

Servo Drive 則根據馬達 Encoder 回授，執行各軸自己的閉迴路控制。

---

# 七、我建議你特別注意的兩個觀念

**第一：Encoder 不等於整個平台的實際位置量測。**

你目前的程式使用 `Encoder Count` 作為位置單位。如果 Encoder 裝在馬達上，它回報的是馬達轉子的位置。

但如果中間還有：

* 聯軸器。
* 滾珠螺桿。
* 減速機。
* 皮帶。
* 機械間隙。

那麼馬達 Encoder 的位置不一定完全等於工作平台的實際位置。

要求更高精度時，有些平台會另外安裝光學尺（Linear Scale），直接量測滑台位置，形成全閉迴路控制。([星泰國際科技股份有限公司][1])

**第二：馬達數量與機械軸數不一定完全相同。**

例如，龍門平台可能使用兩顆馬達共同驅動同一個 Y 軸，以避免平台偏斜。

所以，分析六軸系統時，必須同時了解：

* 有幾顆馬達？
* 有幾個 Servo Drive？
* 有幾個實際機械運動軸？
* 各軸之間是否存在同步或耦合關係？

---

## 八、如果要進一步認識實際產品，可以參考這些資料

以下是幾個有代表性的產品資料：

* [高明鐵 GMT 精密電動滑台](https://www.gmtglobalinc.com/%E9%9B%BB%E5%8B%95%E6%BB%91%E5%8F%B0?utm_source=chatgpt.com)：可以了解單軸、XY、XYZ 及高階精密滑台的結構。
* [高明鐵 GMT 多軸精密耦合電動滑台](https://www.gmtglobalinc.com/%E5%A4%9A%E8%BB%B8%E7%B2%BE%E5%AF%86%E8%80%A6%E5%90%88%E9%9B%BB%E5%8B%95%E6%BB%91%E5%8F%B0?utm_source=chatgpt.com)：可以了解多軸整合與同步運動。
* [高明鐵 GMT 精密對位平台](https://www.gmtglobalinc.com/%E5%B0%8D%E4%BD%8D%E5%B9%B3%E5%8F%B0?utm_source=chatgpt.com)：可以了解 XYθ、XYY、UVW 等對位架構。
* [NSK XY 精密定位滑台](https://www.nsk-cs.com.tw/?f=XY-Modules&utm_source=chatgpt.com)：可以了解導軌、滾珠螺桿、馬達與多軸組合方式。

---

### 我想先幫你建立一個重要的觀念

你目前學習的 STM32H755、ECM-XF、EtherCAT、CSP、Encoder，並不是單純在學習如何讓馬達轉動。

你實際上是在學習如何建立一套**精密多軸運動控制系統（Multi-axis Motion Control System）**。

這套系統最重要的三個層次是：

1. **機械層**：滑台、導軌、螺桿、馬達、Encoder。
2. **驅動層**：Servo Drive、EtherCAT、ECM-XF。
3. **運動控制層**：STM32 軌跡規劃、加減速、多軸同步、位置回授。

接下來我建議先釐清一件事：**你同事所開發的六軸設備，究竟是六個獨立的線性滑台，還是六個馬達共同組成一個多軸對位平台？**

這會直接影響我們後續理解六軸加減速、同步控制與軌跡規劃的方式。

[1]: https://www.taiwan-servo.com.tw/products/detail/218?utm_source=chatgpt.com "奈米精度對位定位平台-星泰國際科技股份有限公司"



我找到你說的產品了，而且找到的資料相當有價值。它與你提到的 **LabVIEW、Camera、精密滑台、光耦合** 都有直接關係。

我搜尋的是高明鐵 GMT 官方網站上的 **Dual Lens**，找到一款正式產品名稱：

# Dual LD Lens Active Aligner（雙 LD Lens 主動耦合設備）

![Image](https://images.openai.com/static-rsc-4/vzr1CJjmrPxgzqWhncZI-LQgBTnR3BfpsdGBeoBO2QctCytR_x6tuxO2AWYPSARL5HcL7KoX-Rbi_N2enpzBW-VPNWQ4X5nSvx_Jqt8mobbpRefZbZsUYX5sjDIBEqMH0bovCf0DU2GEhwVCRaeO70kdQy06AjxHMirep71bGQXHe1Cz8T99FQVF8UT6B3sE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FywfB-a5AjQUIgWHmc4byN1i0B_Cz5skZuFrZlgQofj8Co6-k5GKFuRbU_qlfIOzIIzkM1CbVin0e-XkIFaRof6Tdyrk4z1SlaZiNfwi1eh_1Elf9N8O-xwjDYD9dx-vq1ZMIvFp022e5KeTZ_dlHW7gmuODMmUtT_dRxKW9v9nWqYBxjl0DeLJjNNGMUvcM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bsgyV2xLQ5pBx8GkabVefY6WJZoAsABTLcRAjzbB1sdcwIEymaXVlDfv5-HjrRSgwnt6OVMNt927JOLiOYJ_f6vclvDvKFhJ4O4UN09vwz6EcoaF_vHXZwyK3D5U8bBDyjOEviX4uJvPOAnUbzomFyegHdiTmoIU8DjG-aRaW9QQIQ2Xd1hSfXDxyIsHvHTE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_x1FucopfFPMwi2WOO1eVSBdADnHXzX4a_DXUkV3awZguL2Pm390zyRAFEyxXjjSHzyadQ_5B5gBqUlHVZcCe_eWhnsL0Q-1SFFdoCyqM_6XePy194qnz8GCwuu5oSaHralhPCy6SAMDToFIZ0lPS6t-qThdVFS213o4kVNFljUXf3cFOivmu7jiME1E-B4M?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u8eGO37oUh7Fs5YxMReprW-T7qVybHhv1mNmPakpWycQDHL7glPaBfXJJv4pDDvGavezpXyJTpT2tgkWjITJ0vt-UQJ3JSMQjk7h2xVEqPb68LLznGUvLMmttzsSaPCkoZ4cfk11mXvhoVYm8fZ8MRkW8wMcqtnTcU0iS1QAw6YmoHNIXW4BdEpk59AiDJlz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bWzhYmCFXBromfOPOGxY5jMesZv_G5jhcR5v8-o3SdavBXoNdK-mOG7RYXKhNDX0QtxwFWyNHSeC8BMs6QZ0VsLLHQBq3W-t_kusmevkxxzGP-dCi2hsYzd0ECiwiSoJWTP3-9iTuTKn9OiNihJ-BbCBuTtK1KnK9Qc4iWjHk2TtpO5gQV3bTa2Z193EB0Ic?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9pG2A9istKFUrVqUACqR7jU8OHqPTBGauSbgBWWe4943vzJkZj7KINMbQm75xTuUy-4YYWnip0MLwg8r-tmTTP0oBO9pVV_QqMnzR0Rt5fm2YfSiEOZlgNiCnEWuoL3Bzr69yfLx5Ly42s2ei11zOvZRAvM5WwAnWw_qyZCjwe3sT21g5_Wz41eS1aWhBzrP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/heFOINEdsWqi5neMyMkNpGB32N1Vm_gTGjV3NttnmVsLX_K8_Jwycyr9znxxb-30PqFjCztB_DDzchDPnPo90fVgjBU6oQm7qRgAhElZYOXIr0Pep42Zk3GlEoqsA1lqIRbGCOOT75y0GixmHqzOT1nO4Af-OCSkDQtpXIPOUh3TLKqBuO4TMqf8aPuDpmbu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mHzSq4GcoXt0xRpcn4DGOJOvTE3g5K30I5KK0VUMXyt822joETpeFiYQm6XlO5DPjLzV5q1da-1al6tYpCXyTDaNEY3XUwm0a40uVfn24NXJs6IandZikkBMFJ3Z4MCA3Z-eX0Si7pPaLLBCEZEW4aDPH_Xi8eEr9NFiwEHqgXniNs7nv5ums_v7UaxbDnhY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/eA_hw1a71MnY56_cUlSzaylILa9znkQ1bIWBe2mDndfAfL2AvbLRitl7s_z_e3_3pPCY67nxZf-Ls9tc-Wadcsx1-ZAHv3E03FTApSMBvBc07G2lVkdoDWoaiDsrKLC6a7qpXa8ra0X_WlhNBQa-eVK25rh_Mf2UvSLNOIvPLydbJpQE7qDmU4vNyINfh9Zv?purpose=fullsize)

官方產品頁面：

[Dual LD Lens Active Aligner－GMT 官方產品介紹](https://www.gmt.tw/en/gmt-high-precision-coupling-solutions-for-silicon-photonics)

這款設備的用途不是單純把兩個鏡頭放在一起，而是將兩個 Lens 精密地對準 PIC（Photonic Integrated Circuit，光子積體電路），使光訊號能夠有效率地耦合進入晶片。

官方列出的主要功能包括：

* Dual Lens Coupling：雙鏡頭耦合。
* Lens Align with PIC：鏡頭與 PIC 對位。
* TX Power：發射端光功率量測。
* UV Glue：使用 UV 膠固定鏡頭。
* UV Curing：紫外線固化。
* Online Monitoring：線上監控與資料上傳。

([GMT][1])

---

# 一、這台設備究竟在製造什麼？

先了解它的產品用途，才能理解為什麼需要六軸滑台和 Camera。

這類設備主要應用在**光通訊與矽光子（Silicon Photonics）**。

例如，現在高速光通訊設備需要將電訊號轉換成光訊號，再透過光纖傳輸。

其中有一個重要元件叫做 PIC。

PIC 可以把光學元件整合在晶片上，但外部光源、Lens、光纖與 PIC 之間必須非常精確地對準。

如果位置偏移，即使只有很小的距離，也可能造成光耦合效率下降。

所以這台設備的任務就是：

**讓兩個 Lens 找到最佳的光學位置，讓光能夠有效地進入 PIC。**

官方產品頁面還提供了一張 External Laser Source（ELS）模組示意圖：

![Image](https://images.openai.com/static-rsc-4/m0YbRVU9XkXHVFdU2uxOK-jXDQjMow6SO4Y4mhp-mFj0a_VCaKFrM_JlXEyuZ11cWyOUlv1537uqIvCoWTHjPXH18bZcUxJxlNVfMltLFEbRDbLHa0ozOwdDkaK7tImrMWi2vra2g98VbLSAhLVEgBwWNnX5Ad3ThrDfX3wJHL629Mqy6prqbcz8cMjrexe-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xETkA7Pzs9vRaGBfJSHnehBWkzlxwBR1ht1N6PMtSSIVw5SLJAJeUb-7T3k_BXR-iHloLZI3k16Lpcz-mrJHa390pR1Peqn3LBfsX1xuGe2oYzbT6iADx7M55roB9ojoTBR__3dMNge4XYMDoXkd9wtMLBNR14dFUY4TYs_9ejMfwvS2UCX2xHv6gNzodUnK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gjLxrq9RgJk6lUygcE00pkAZhzuqFjO9_JDHdVqYzA5cwt7LTZ2zZvEBd8lZpg6eHmMnJ3fp8qmi7I2b_a1p-6JLTszmN8ebdUxAfmLcYDIWpsuH8VZhnkhRDTLVsGKxN_5hmjBX3N8sF-uSsLen_M3BndFogq194a-cXFHsO8zjKWhOKQZPTKtiD6DBCLZO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NHebI3Wt2ZCWaFOm6BDqIbcdsapRoh3cCche1iUKmdCthQbOV_ea29H-MHhXwKw-WLTZjhcD9pXD0i6dyEKiZAeLbMtqAHf5Sv-VEDNdDgwOa3KIih4Vnlqm3NWQn9y05EFJgNMhfiCXiiKDVbZi-AYEZ6GHl65Uixknbroh9VLazAMctXKWkR90NeQ3vC1O?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/k91zRrmtEU9ED-VqYV8bYXh9cHRJ-9bJvuhtzUczG-2X4CrGooJ6NyLYUk4z9i-dCVEdqmIIkytH5QRgqf6j9NFQpi0iA-zzRIhi1ZU-cdWWQ3odnviAtkMQQsFsBpeZEGJDVEc4poTE5JLJ2fVaJAii_rPaMi065jn0S_8z9nvNuWO0BEF0_xYRsDmlaa7O?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1iommWILVNIztX-pc8kmk04Tr8y5iacxfKrjNyelQFchQB2i4i2KeQIOBfw9Oj6GOaCQGaTzFkQ8-IkAvNLpE2sSrhKAEVmK660_B3IJJj1cYjoWL22V17j7Dqg6dwttYtofK6mguzb1yhosjTFYj4KCpx_lnKN5ijR0IVxp58AhB9pXITax-MgPTASrKQzX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WIA3-mvkv6qIRLAYxd719dYoI0p_XGNWdbqAB_loCO7PMaCrvjICkxNNMVI2iuiLgAuk96O7doA1nnB2NX_bwPSx8jeAwkA_41LuzMu1oiKlWAyjNEH2UJujMX8CDeSXkfohuCamK483OhfKx6mRX7iAH4_3Np5dXiQUuGlRn4vsg-Sh83j_GXARPx-QZm41?purpose=fullsize)

圖中可以看到：

* Laser：雷射光源。
* Laser Chip Substrate：雷射晶片基板。
* Lens：透鏡。
* Fiber Block：光纖固定座。
* PIC 相關模組與電路連接。

這些元件必須在非常精密的位置關係下完成組裝。

---

# 二、為什麼需要六軸滑台？

這就連結到你之前描述的六個獨立線性滑台。

在光耦合時，不只是左右移動就可以完成對位。

通常需要考慮六個自由度：

| 自由度 | 英文            | 功能      |
| --- | ------------- | ------- |
| X   | Translation X | 左右移動    |
| Y   | Translation Y | 前後移動    |
| Z   | Translation Z | 上下移動    |
| θX  | Rotation X    | 繞 X 軸旋轉 |
| θY  | Rotation Y    | 繞 Y 軸旋轉 |
| θZ  | Rotation Z    | 繞 Z 軸旋轉 |

前三個是直線移動，後三個是角度調整。

例如：

Lens 已經對準 PIC 的左右與前後位置，但是角度稍微偏斜，光就可能無法有效進入 PIC。

這時候就需要調整 θX、θY 或 θZ。

GMT 官方產品規格確實列出了：

* Dual 6-axis System。
* X/Y/Z 精密定位機構。
* θX 手動調整。
* θY/θZ 自動調整機構。

而且官方列出的 X/Y/Z 解析度為 10nm，重複精度為 ±50nm。([GMT][1])

**注意：這款設備標示 Dual 6-axis System，不代表它一定就是你目前 STM32H755 所控制的六個獨立線性滑台。兩者的硬體架構必須分開確認。**

---

# 三、這款產品的 Camera 是什麼？真的會自動辨識嗎？

這次找到的官方規格，剛好可以回答你之前的問題。

這款 Dual LD Lens Active Aligner 配置：

| 項目          | 官方規格                |
| ----------- | ------------------- |
| Camera 數量   | 2 台                 |
| Camera 類型   | CCD                 |
| 解析度         | 5 Megapixel         |
| 視覺範圍        | 5.6 × 4.7 mm        |
| 影像解析度       | 1.4 μm              |
| 視覺功能        | Automatic Alignment |
| Camera 移動機構 | XYZ 三軸              |
| Camera 驅動   | 步進馬達＋滾珠螺桿           |

資料來源：GMT 官方產品規格。([GMT][1])

這代表它不是單純讓工程師觀看影像的監視器。

它具有**自動視覺辨識與對位功能**。

## Camera 在這裡做什麼？

可以想像成：

1. Camera 拍攝 Lens 和 PIC 的位置。
2. Vision Software 分析影像。
3. 找出 Lens 與 PIC 的相對位置。
4. 計算需要修正的方向與距離。
5. 通知控制系統移動滑台。
6. 再次拍攝，確認位置。

另外，Camera 本身也有 XYZ 三軸移動機構。

也就是說，Camera 不一定固定在機台上，而可能需要移動到不同位置，觀察不同的工件或對位區域。

這款設備使用兩台 Camera，應該就是為了不同視角或不同觀察位置的視覺定位需求；不過官方沒有公開兩台 Camera 各自的具體分工，所以這部分不能直接認定。

---

# 四、這台設備還有一個更重要的功能：主動光耦合（Active Alignment）

這是我認為你最值得了解的部分。

**Camera 辨識位置，不等於光耦合已經達到最佳狀態。**

因為 Camera 看到的是幾何位置，但真正的光耦合效率還會受到光軸、角度、鏡頭特性及其他光學因素影響。

所以這類設備會使用另一種回饋：

**光功率（Optical Power）。**

GMT 官方 Dual Lens 設備明確列出 TX Power 作為耦合過程的關鍵項目。([GMT][1])

例如：

假設 Lens 對準 PIC 後，量測到的光功率是：

```text
目前光功率：-15 dBm
```

滑台稍微移動後：

```text
目前光功率：-12 dBm
```

再調整：

```text
目前光功率：-9 dBm
```

這表示在這個例子中，接收到的光功率逐漸增加。

系統便可以持續調整 X/Y/Z 與角度，尋找光功率較高的位置。

這種方式稱為：

**Active Alignment（主動對準／主動耦合）。**

它與只靠 Camera 找到幾何位置的 Passive Alignment（被動對準）不同。

兩者可以這樣比較：

| 比較       | 被動對準             | 主動對準          |
| -------- | ---------------- | ------------- |
| 主要依據     | Camera、外觀尺寸、定位標記 | 實際光功率或光學訊號    |
| 是否需要光學量測 | 不一定              | 需要            |
| 對位方式     | 根據預先設定的位置        | 根據實際光學回饋調整    |
| 目的       | 找到預估的正確位置        | 找到實際光學性能較佳的位置 |

實際設備也可能先使用 Camera 完成初步定位，再使用光功率回饋進行精密耦合。

---

# 五、LabVIEW 在這款設備中扮演什麼角色？

這部分和你之前描述的既有產品完全吻合。

GMT 官方規格明確列出：

**Software Programming Language：LabVIEW**

而且具有：

* 生產工單與產品編號記錄。
* 生產資料儲存。
* 多種對位程式保存與呼叫。
* 線上監控與資料上傳。

([GMT][1])

因此，這類設備的控制架構可以理解成：

```text
             工業電腦 IPC
                  │
               LabVIEW
                  │
        ┌─────────┼─────────┐
        │         │         │
      Camera   光功率計   其他感測器
        │         │         │
        └─────────┼─────────┘
                  │
             運動控制系統
                  │
             Servo Drive
                  │
             精密滑台馬達
                  │
             光學元件對位
```

這是依據產品規格整理的功能架構示意，不代表官方公開了所有實際接線方式。

---

# 六、除了 Camera，這台設備還有哪些重要單元？

官方規格還揭露了幾個很值得注意的設備。

| 單元                           | 功能           |
| ---------------------------- | ------------ |
| Dual 6-axis Alignment System | 六自由度精密對位系統   |
| CCD Camera × 2               | 視覺辨識與定位      |
| Camera XYZ Stage             | 移動相機觀察位置     |
| Power Meter                  | 量測光功率        |
| Dual Dispenser               | 兩組精密點膠機構     |
| UV Curing System             | 紫外線固化        |
| Workpiece Clamping           | 工件夾持         |
| Vacuum System                | 真空吸附固定       |
| Pneumatic System             | 氣壓驅動機構       |
| LabVIEW                      | 設備操作、控制及資料管理 |

其中點膠和 UV 固化也很重要。

因為 Lens 找到最佳光耦合位置後，不能讓它再移動。

所以設備會：

1. 找到最佳光耦合位置。
2. 使用精密點膠機構塗佈 UV 膠。
3. 維持 Lens 在正確位置。
4. 使用 UV LED 照射，使膠水固化。
5. 固定 Lens，完成組裝。

官方列出的 UV 光源是 HOYA LED，波長 365nm。([GMT][1])

---

# 七、這和你目前正在開發的 STM32H755 新產品有什麼關係？

這裡我想特別區分兩件事。

你現在正在做的是一套新的 STM32H755 + ECM-XF + EtherCAT 六軸控制系統。

而你描述的既有產品，是工業電腦透過 LabVIEW 控制精密設備。

兩者不是同一套產品，不能直接認定新產品也會採用相同的軟體或控制架構。

不過，從 GMT 的產品方向與官方資料，可以看出一個很重要的應用趨勢：

**精密運動平台不只是讓馬達移動，而是要整合視覺定位、光學回饋、精密運動與自動化組裝。**

GMT 官方也將光通訊、CPO、矽光子及光學元件列為應用方向。([高明鐵 GMT][2])

所以，你目前學習的 STM32H755 運動控制，將來可能會成為這類設備的底層運動控制技術之一。

但目前還不能確定你的新產品是否會整合 Camera、光功率計或 LabVIEW。

---

## 八、我建議你接下來先認識一個關鍵技術

我認為你現在最需要理解的，不是先研究 Camera 的程式，而是：

### Active Alignment 的完整運作流程

因為這個流程可以把你目前學習的所有技術串在一起：

```text
Camera 拍攝
    │
    ▼
影像辨識
    │
    ▼
取得初始位置
    │
    ▼
STM32 運動控制
    │
    ▼
ECM-XF
    │
    ▼
EtherCAT Servo Drive
    │
    ▼
六軸滑台移動
    │
    ▼
光功率計量測
    │
    ▼
回傳光功率
    │
    ▼
判斷是否需要繼續調整
    │
    ├── 否：完成耦合
    │
    └── 是：重新計算位置
             │
             └── 再次移動滑台
```

這裡會出現兩種不同的回授：

* **Encoder 回授**：告訴控制器馬達實際移動到哪裡。
* **光功率回授**：告訴控制系統目前光耦合效果好不好。

這兩種回授非常不一樣，卻必須配合工作。

我建議下一步就深入理解這兩種回授如何共同運作，以及 STM32 在 Active Alignment 中究竟需要負責哪些工作。這會讓你更清楚為什麼六軸控制需要軌跡規劃、加減速、位置回授與多軸協調。

[1]: https://www.gmt.tw/en/gmt-high-precision-coupling-solutions-for-silicon-photonics "GMT GLOBAL INC-GMT-Dual LD Lens Active Aligner"
[2]: https://www.gmtglobalinc.com/en/?utm_source=chatgpt.com "Automation Component Solutions for Semiconductor & Optics | GMT GLOBAL"



這個主題非常重要，因為它可以把你目前學習的 STM32H755、ECM-XF、EtherCAT、CSP、Encoder，以及未來可能接觸的光耦合設備，全部串成一套完整的控制概念。

我先講最重要的結論：

**Active Alignment（主動光耦合）其實需要兩種不同的回授：**

1. **位置回授（Position Feedback）**：確認馬達與滑台實際移動到哪裡。
2. **光學回授（Optical Feedback）**：確認目前的光耦合效果好不好。

這兩種回授解決的是不同問題，不能互相取代。

---

# 一、先理解兩種回授的差別

假設我們有一個光耦合設備：

* 一個 Lens（透鏡）。
* 一個 PIC（光子積體電路）。
* 一個雷射光源。
* 一個光功率量測設備。
* 一組精密線性滑台。

我們的目標是讓 Lens 和 PIC 精確對準，使光能夠有效地進入 PIC。

但有一個問題：

**滑台移動到指定位置，不代表光耦合效果一定最好。**

這就是為什麼需要兩種回授。

## 1. 位置回授：Encoder

![Image](https://images.openai.com/static-rsc-4/_FvYgdMHo12hOHPoB9Ck6LvcgjX9H3fTgSWDKebTQ6M_1LCu7in9Z8K2xgphrbTkBIpg3inNlxU-0csJqJCU09j4bqkQK1VdQLH5q9pC3ZzKpaB7cjEq1zDCz4-OzvYxwYlWwdGKNoSdOhfvGLVylCegwgqkVkgsgpLjbidY8JiU8VaXEqEjEFuaEsvlAK4a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/JtuMFZEiKaOYruX9foQQN_Ivg8gq5Wx8xAhMzuK-91Zuut_IYAEcGeALV8dadnyqPu8l2-FT8nmqKyHdRXW6FdyDIRcybK6kel1UPkElkToBhwoCaCm1JzP5mv9SWQWqaYeLjSeocVjO49UC_9_0u40A4Jst2xaSzSpCc9CjV4kSNRyPxYRVHZq_hWS4EO5W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yx3iSxuYsoHoY6N0o228Ax7cAgENMOJ04Bq6vAt-GHe1MVHTcXZ_Fdkmh5cdc-hiAAhCQ0cCN-mXMehmI4fWcc56Aepj3DggiKdQfGco8osVNFTxFXUr7yjBF4k0KS-cdMjTiXemY4c0QtmwwPgcDJXPJl93nOh1xvhSVLuZjzwAR6NZfUgpniUYPK7bAZHI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XBY_XFtnEkFE1dW6S5ID0f94g4csjwvZRJESLGrYLk5ws2WObfQJRItK6WN2J_6-TxJjUYCs093o_NCrDS4noqwHg0Yfrb_YUyC8wGhdxqQzuEGpr4oc-5n7RSRYaZBFBqSlcrWkSBdHxrp0u4rx_V0XVbE2mNJkYGMDNKcC24-1G0FKq-iTk3A_VNS3L4ns?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/s4RaD0xRlSTqCHep7FX4e1CnAZvTfU4aQzFyt-yB7Ocbkxpl6WBOR5HBfzMaW6O2pJWppgJF4KKm5x5R-jcDflUgnERAzrjmdyQWhJmbPGpHq9tO578bhRQW4FsaKECikaubOveS8NiZQkRgrKENPMeAa3In9kD-eGSy4wsi0L5relb9XrMXnOfNI6gItTfB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LoVZ6YUp8_CoXRIpooCPtCpaGaWqLnBwhnBhep4JdmLj8Gy8PueLYnFh4xp1aithrl9eWQSSbUiehXcxs1sXC0TywFqOnP7I-7F3lZTunb-pLdpzsENniS1moH56PrHKe_TA-RhlrNHOPW_hdloxGJQ-ycxFesoksEZf1eKp3rLw5twVjhXcc3XL-vQkjGv0?purpose=fullsize)

先假設 STM32 命令馬達移動：

```text
目標位置：100000 counts
```

Servo Drive 接收到命令後，控制馬達轉動。

馬達上的 Encoder 持續回報實際位置。

例如：

| 項目           | Encoder Count |
| ------------ | ------------: |
| STM32 命令位置   |        100000 |
| Encoder 實際位置 |         99980 |
| 位置誤差         |            20 |

這代表馬達目前還差 20 counts 才到達目標位置。

Servo Drive 會根據位置誤差繼續調整馬達的驅動輸出。

這就是位置閉迴路控制。

簡單說：

* STM32 提供目標位置。
* Servo Drive 控制馬達。
* Encoder 回報實際位置。
* Servo Drive 持續修正誤差。

這個回授主要是為了讓馬達準確移動。

### 這裡有一個你必須知道的細節

如果 Encoder 安裝在馬達內部，它量到的是馬達軸的位置，不一定是滑台的真實位置。

例如：

馬達轉動透過滾珠螺桿推動滑台。

如果螺桿存在間隙、變形或熱膨脹，即使馬達 Encoder 顯示已經到達目標位置，滑台也可能存在微小的位置誤差。

要求更高精度時，可以另外在滑台上安裝 Linear Encoder（線性編碼器），直接量測滑台位置。

這種架構有時稱為雙回授或全閉迴路控制。([ABB Library][1])

---

## 2. 光學回授：Optical Power

![Image](https://images.openai.com/static-rsc-4/6IdFE1H-zs3chKlgRz3sKJEhUU-vDd2Vc2RpV4cIRqX1BJeAX4D4s0bct3I2Dy8CGhtUv1lcla1O5gNKcCjoSQwt_TbjtJ2i5sOs3oeoIO710v9Icag84p_u_VQXALledyek0QKWLCyRFUSgCeKA1QBLNkxLoUVHc7SMaJ1WLjc837K0SIsVJKBTjr3AhHMD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0QYlB-GTFBvR3TE5HGjpdy4K7frQBtRG0siJtBoXTL2n0SoyBsck1PA-8p5imlA5gERLvgmzZf19hdDt1CBRzPVsq_v3jcc2VZYEzLA9ZR1Z8sRj1rBmm1WdjbCF3UrSg1BoPDxQL-HlhTpDK3Atb1z5z6Il172ZwCmiSvnOcX-Rf0W3GEyKi4Nkwr8zIwmo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IiRNOUh_gcbzfx4RgcERK6gGsarBwrYZMD303dBQkiPxq-oTsPl6ycQdDDhpWgBMe_YOpnWosDSMk1cRWF0hybffnRucq_j0WB4LhYGgl4xQZQ39OM7iIYLknfqmxwYsy9txQUE2-jTlUudaPZDPwxHpXmN1p-VLc8zgRU_MtDSxSWJWcPd7H9mRmz6xv5_d?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9ddM4fteEMCdyDaLknKXRqXbBekHPOhGe9HNEWKFzKenJvaZrGIvxn1EGpvOBc8vjk4Sg2SfttNLhvkNczrRyDD6agCGorvVZmfO1hEnLTdL9ewTHBXwfRe60WP6Q2XjiAyOZTTdXMoNxoJuN3oo1YiP1_DzI6bha7b1nuLwv7CHAu9T7s4-46Y-EhqVRSQ4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/DSn35H9G0YGg9ZAGqkbOQBYo3dsbfMDVYVHXCLQTy57n73pFEpXbDdFCHm2ot1SLsukkfV-shpgdeSIjRsEvMG-q1bxBkvQqoY_IiatmYz_m4pXYnKvBqVv1ucenKx1kjR7XOOhwLqiHxpMQzP-ZtpHZGIjnFfYl0sYVx9sh8y5M9lLvjyxIKw39JzJsa4lL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/913_LGROsgwWk0C1H_PnLQv6m_qvxM296uMpL_NlX4cIp_c0oPUeQoJVgwHxVC3lybcsuJFwU2xhE5mWgY0BA9MfwTjSfFYmbSN3MIkJTzcStYzM_0xHBwXWP9c4I5HJD0_eXXoFcpQ6nH3Y8c37asoYEe-CtuxlCi4nZZRA-Q91ZbhvBPICchg0_AMY9x99?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CwYY1172fWEIEYM08xbzBDUkqH_o8ubyTW5W2j1QrETBHMpx1t8ucqJvJOlXeFR277MP0RxDbIfWiyJpPv9i68-DwKByMo2xOVOHPoDgdMUnU2x748m0Xmj8MKefjbQD9UTF7fwiqYYxkHdY6w9d5Zzunj6Ie7cLhkLxczOtgBNZvfbIsz3gdDYKSTIGQnoc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Y9EQayUapqlTVLNIKL2e7S7_wK0smyNFeWl-5ORQW4ddesw31yJnJeaSPHQLhXEZhmw0611BCeHD7SADq2kRNCcg7H6jqcXPjKbv9qCk-YvfDQ9AiAuaSMoAeJHdT2-ckMWNrU46fvEleg4DlKFA_jY1GrbnhGqwoFtJ0pwqkzYsfw77rb454EADourrKECr?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/eNbPzNjzazresAn8lI1vUSeI-0nI_W_7rX45yV2f-tIiiodjQXqT4cTBuYEeqAvHpLfl5PMzx9BjIwkmmAJUkBUjCz7k5vLGQCNo-DZdwgb8XY6S9xx7vGuNkFoDi5tXPDuCF1UfQer7CXQj5RqnQa040qAbVx_CpiHb2cZlxXb2wmC0u_dcgtab4Qn3BeLg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jX53xias0Lg1VqoqLbQUJTYxe66_APj77LNwv74jI0jDp-udcG463hFRjraKOx83-osxn1eiTnhByH_LHWkmkFF9M1zZGmcJB5kea1RxPLdx6wXkU-8we25yu-Khhit8dQJyaNgcGDwjZc2H1irkYYwdQGqpuXUU-bqY3W2mZJmH4dnxjhHRLnNxkqrCVmGI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/E8r1HdLe5GgXXPltO931wxShHWkagVSmdAWDENonZ0V5KPculCil8z3OaVVdxQR4E-zqc_dfi3o6Ae0dooWhr026SABwGGakOsv8u3xPaKVPt6DqH2KAeRWi5oZD1nl7_dtFFv8Y4dwYDNUIBOCBDpu1T_cmFGsJmf_dxu3ctvML1ivnAECQ3omMcRGGRerl?purpose=fullsize)

現在假設馬達已經準確移動到指定位置。

但我們還不知道：

Lens 是否真的對準 PIC 的最佳光學位置？

這時候就需要光功率量測設備（Optical Power Meter，OPM）。

它的工作是量測實際傳輸或接收到的光功率。

假設：

| 滑台位置       |     光功率 |
| ---------- | ------: |
| X = 100000 | -15 dBm |
| X = 100010 | -12 dBm |
| X = 100020 |  -9 dBm |
| X = 100030 |  -8 dBm |
| X = 100040 | -10 dBm |

從這個例子可以發現：

當 X 軸從 100000 移動到 100030，光功率逐漸提高。

但移動到 100040 時，光功率反而下降。

因此，100030 附近可能是較佳的耦合位置。

這時候系統就可以利用光功率的變化，判斷下一步應該往哪個方向調整。

這就是光學回授。

PI（Physik Instrumente）的 Active Alignment 技術資料也說明了這種方式：控制器可以透過小幅度的測試移動，觀察光功率變化，推算改善耦合的方向，並逐步尋找最佳位置。([PI USA][2])

---

# 二、兩種回授如何一起工作？

這是整個 Active Alignment 最重要的觀念。

我用一個實際流程說明。

假設 Lens 現在的位置不正確。

### 第一步：STM32 移動滑台

STM32 命令 X 軸移動 100 counts。

```text
Target Position = 100100
```

Servo Drive 接收命令，控制馬達移動。

### 第二步：Encoder 回報位置

Encoder 回報：

```text
Actual Position = 100100
```

代表馬達已經到達命令位置。

如果使用的是馬達 Encoder，這只能確認馬達位置；要確認滑台實際位置，還需要考慮機構傳動誤差或使用線性 Encoder。

### 第三步：量測光功率

OPM 回報：

```text
Optical Power = -12 dBm
```

控制系統記錄這個位置的光功率。

### 第四步：再移動一小段距離

STM32 再命令 X 軸移動 10 counts。

Encoder 確認移動完成後，OPM 再量測：

```text
Optical Power = -10 dBm
```

這次光功率提高了。

代表這個方向的移動改善了光耦合。

### 第五步：繼續尋找最佳位置

控制系統可以持續調整，直到找到光功率的最大值，或符合設定的耦合品質要求。

整個流程可以整理成：

![Image](https://images.openai.com/static-rsc-4/hY4Cclooy1bPyhtxREmPIguqaU9996aCxyLk1Y1RGWddN29lHMks6kRkteobFWm1JvfojmhVv3y3wxy7IYUscURF7eOaYIJI8W_VAHvfPiRSQ9hBPU-DILZfBl7gRGOebk2gcpp-9VBrulWJRsZ3xXAnWVJdTDpRtnUcafOb9_Eep5T2FpUlQnd88tw8iuWG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/svjesyhiMzFSKmX7IjY6EDv-tj6Xv2N_XaIbm3j-8PlHh73V9XU_ApiBO_X6evkMQT4qGGRC0be4ja2rIcmaV0ch7BitrEVuJDOj7YrcL_isiHe0ysNN4f2HozD6h9GoLItxYFyPR-TzH7aM9Ze4BqHM7Z9daIOn5Vkuydyi6velZrx1WIh3e7Flc1rKoVf2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GxZoi2ZIUPbIiKVEe2HRNFpRMwMVS7OixDE0I9IDezvdC2QikLOTJnMpqvsMQ-mzIdpHmDO0A_Yx6gSpsengmXD0ZktKjxJg1llERACGVrkbc7rbXu6ckcurat5EZJA2CniTNynOHxmf9VQOkw_fh7VUXBxW-m_AZGZo5LwLkOMgHYhCBdNZzm3V9ea1yk_y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/24hk4CVLDXSlY_ARyhSBAbiJ2EeKsYmnHv6le65V8vyDLlS6bgjWqv-gmmWymEpuS1-vVfYQT_zx2VTxKH--IsSi49WJ-FMBljgV3-8N6Tpxo_iSMj8t5-OmQjObRq6KOjhQtNVVNLxdCNCDBwqJ1Xdpc6iQKW6iTJVADdb8C-IsqVvI_AxEQGWv9Phsmx3l?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tmza3SCiITpSDwP_PNXKPQpvWAmSRX5SaOoFrioVxL8aoF0kes7X8tD56JxyeDLi_XuftcNbrGwxvb4O7ozWN_ug5EaXpbNW52XU-fsADpV9hHct4cn_bImFuJNDYeP1yQG5lb2RFQ_4Nqw_LLg2rixW-w-L5nWYc8a-aoHTuRn7Ug4IkFlUQQZilJC5XzjN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/t6l9uZfGzpGfuXWLv12BRd_o_eR0aUH9Hb7lAbtO-Gfkdw7T2Z-O9Cn7TpQ8DIgk0YDuSnwQeXnSGn2yaqF5hyUARSo52qlEgyAqAr0vnDdzR29W9VoOk6SFnXADjzIiL6I3mU0c4o7Vzm4_n3cFRxyOmv0B4ppqGl2AJuvr66prqDzt-UqlF4Kaw5-enb6g?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W4cmyDM78fAHL_fK__yveT3jA7CWEZjIBQDLspQvVYdAlmVBTEI4NRiUxtTJH-P-s1xCt0pmQdhyn8llVkGMbhdNUZstaR47xVn3giGVAPfGwneg7urslFuuPwJLrFp7W6Q9FQQowdSFiqV8QEyyn8CYcMB53F2SsMvGYIPtUMLlUsc19h1SYcx3zqQWfVyi?purpose=fullsize)

```text
         光耦合控制系統
                │
                ▼
         設定滑台目標位置
                │
                ▼
          STM32H755
                │
                ▼
             ECM-XF
                │
                ▼
            EtherCAT
                │
                ▼
           Servo Drive
                │
                ▼
             馬達
                │
                ▼
             滑台
                │
                ▼
          Lens / PIC PIC（Photonic Integrated Circuit，光子積體電路,光子晶片）對位
                │
                ▼
           光功率量測
                │
                ▼
          回傳 Optical Power
                │
                ▼
          判斷是否改善
                │
         ┌──────┴──────┐
         │             │
       有改善         沒改善
         │             │
      繼續調整       改變搜尋方向
         │             │
         └──────┬──────┘
                │
                ▼
          找到最佳耦合位置
```

這裡要注意一件事：

**Encoder 回授是 Servo Drive 內部控制迴路的一部分，而光功率回授通常是上層 Active Alignment 演算法使用的量測結果。**

兩者不一定由同一個處理器執行，也不一定使用相同的更新頻率。

---

# 三、STM32H755 在 Active Alignment 中究竟負責什麼？

現在進入你目前正在開發的部分。

你的硬體架構：

```text
STM32H755 CM7 / CM4
          │
          │ SPI1
          ▼
        ECM-XF
          │
          │ EtherCAT
          ▼
    Servo Drive × 6
          │
          ▼
    Linear Stage × 6
```

如果未來要讓這套系統應用在 Active Alignment，我認為 STM32 至少需要負責以下工作。

## 1. 接收運動命令

運動命令可能來自：

* 工業電腦 IPC。
* LabVIEW。
* Qt 操作介面。
* 其他上位控制系統。
* 視覺辨識系統。
* 光耦合最佳化演算法。

例如上位系統要求：

```text
Axis 1：移動到 100000
Axis 2：移動到 200000
Axis 3：移動到 150000
```

STM32 接收到命令後，需要判斷各軸是否可以執行。

例如：

* Servo 是否已經 Servo ON？
* 是否已完成 Homing？
* 是否超過軟體行程限制？
* 是否有驅動器異常？
* 是否有正在執行的運動命令？

STM32 不應該不經檢查就直接執行外部位置命令。

## 2. 負責軌跡規劃與加減速

這就是我們前面討論的重點。

假設 Axis 1 要從：

```text
起始位置 = 0
目標位置 = 100000
```

STM32 不應該直接把 100000 傳給 CSP，而是應該產生一連串逐漸變化的位置。

例如：

| 時間   |   目標位置 |
| ---- | -----: |
| 0 ms |      0 |
| 1 ms |     10 |
| 2 ms |     30 |
| 3 ms |     60 |
| 4 ms |    100 |
| 5 ms |    150 |
| ...  |    ... |
| 接近終點 |  99990 |
| 最後   | 100000 |

這只是示意數據，不是實際運動參數。

這種方式讓位置增量逐漸增加，代表馬達逐漸加速。

接近終點時，位置增量逐漸減少，就代表馬達逐漸減速。

STM32 每個週期都計算新的目標位置，再交給 ECM-XF。

### 為什麼不能只讓 Servo Drive 自己處理？

因為在 CSP 模式下，Servo Drive 主要是追蹤外部每個週期提供的位置命令。

如果 STM32 每 1ms 提供的目標位置跳動太大，Drive 就必須努力追趕，可能出現：

* 跟隨誤差過大。
* 馬達速度超出預期。
* 加速度過大。
* 機械震動。
* Servo Drive 發生異常。

所以，STM32 必須確保提供給 Drive 的位置命令符合設定的速度、加速度及其他運動限制。

如果要求更平順，還可以使用 S-Curve（S 型速度曲線），讓加速度也逐漸變化，減少機械衝擊。

---

# 四、為什麼六軸需要協調運動？

你已經確認目前新產品是六個獨立的線性滑台。

這代表六個軸各自有獨立的馬達及機械結構。

但是，獨立並不代表每次都應該各自隨意移動。

在光耦合時，六個軸可能需要配合調整。

假設：

| 軸  | 起始位置 |  目標位置 |  移動距離 |
| -- | ---: | ----: | ----: |
| X1 |    0 | 10000 | 10000 |
| X2 |    0 |  5000 |  5000 |
| X3 |    0 |  2000 |  2000 |
| X4 |    0 |  8000 |  8000 |
| X5 |    0 |  3000 |  3000 |
| X6 |    0 |  1000 |  1000 |

如果六個軸都要在相同時間內完成移動，就不能讓每個軸各自使用相同的速度。

因為移動距離不同。

控制器必須根據各軸的移動距離，計算各自應有的速度與加速度。

例如：

* X1 移動 10000 counts。
* X2 移動 5000 counts。
* X3 移動 2000 counts。

如果三個軸要同步到達終點，X1 的移動速度就必須高於 X2，而 X2 又必須高於 X3（假設各軸的運動時間相同且採用相同的時間比例）。

這就是多軸同步控制。

對你目前的六個獨立線性滑台而言，還有另一種情況：

有時候只需要移動其中一個軸，其他五個軸保持不動。

例如，Active Alignment 演算法發現 X3 方向的光耦合效果需要改善，就只命令 X3 微調。

所以 STM32 必須同時支援：

* 單軸運動。
* 多軸同時運動。
* 多軸同步到達。
* 多軸位置保持。
* 多軸運動限制。

---

# 五、六軸如何配合光功率尋找最佳位置？

這裡就進入 Active Alignment 最核心的演算法。

假設我們有六個獨立線性滑台：

```text
X1、X2、X3、X4、X5、X6
```

每個軸都能調整 Lens、光纖或光學元件的相對位置。

我們希望找到一組最佳位置：

```text
X1 = 100020
X2 = 200015
X3 = 150008
X4 = 300010
X5 = 120005
X6 = 250003
```

這些位置組合只是示意。

系統可以透過下列流程尋找最佳位置。

### 第一階段：粗定位（Coarse Alignment）

Camera 拍攝工件。

Vision Software 辨識 Lens 與 PIC 的位置，計算初始位置偏差。

然後讓滑台先移動到大致正確的位置。

這個階段的目的是讓光學元件進入可以開始耦合搜尋的範圍。

### 第二階段：尋找光訊號（First Light）

開啟雷射光源。

開始小範圍移動滑台，觀察光功率是否開始增加。

如果量測到有效光訊號，表示已經找到可能的耦合區域。

### 第三階段：精密搜尋（Fine Alignment）

現在開始非常小幅度地調整位置。

例如：

```text
目前 X1 = 100000
目前光功率 = -15 dBm

X1 + 10 counts
光功率 = -12 dBm

X1 + 10 counts
光功率 = -10 dBm

X1 + 10 counts
光功率 = -11 dBm
```

這表示光功率在某個位置附近達到較高值。

系統可以進一步縮小移動步距，尋找更好的位置。

六軸系統還可能需要分別搜尋 X、Y、Z 和其他軸的最佳位置，並反覆調整，因為不同方向的調整可能互相影響。

PI 的技術資料提到，光學耦合最佳化可以使用區域掃描（Area Scan）或梯度搜尋（Gradient Search），而且某些互相影響的軸可以進行平行最佳化。([PI USA][2])

### 第四階段：固定 Lens

找到符合要求的耦合位置後，設備可能會使用 UV 膠固定 Lens。

但是點膠與 UV 固化可能造成微小的位置變化。

因此，高精度設備可能會在固定過程中持續監控光功率，必要時再做補償。

---

# 六、STM32 不一定要負責光學搜尋演算法

這點我特別提醒你。

你目前正在開發 STM32H755 的底層運動控制系統，不代表未來一定要把所有 Active Alignment 功能都寫在 STM32 裡面。

實際上，可以有兩種架構。

### 架構 A：IPC 負責光學演算法，STM32 負責運動控制

```text
            工業電腦 IPC
                 │
              LabVIEW
                 │
          Camera / 光功率計
                 │
                 ▼
          光學最佳化演算法
                 │
          六軸目標位置命令
                 │
                 ▼
             STM32H755
                 │
             軌跡規劃
                 │
               ECM-XF
                 │
              EtherCAT
                 │
           Servo Drive × 6
                 │
           Linear Stage × 6
```

這種架構中：

* IPC 負責 Camera 影像辨識。
* IPC 負責光功率分析。
* IPC 決定下一步要移動哪個軸。
* STM32 負責把位置命令轉換成平順的運動軌跡。
* ECM-XF 負責 EtherCAT 通訊。

### 架構 B：STM32 也負責部分 Active Alignment 演算法

```text
Camera / Vision
       │
       ▼
    IPC / PC
       │
       ▼
   STM32H755 ◄──── 光功率量測
       │
       ▼
   運動規劃
       │
       ▼
     ECM-XF
       │
       ▼
  Servo Drive × 6
```

這種架構中，STM32 除了控制馬達，也可能接收光功率數據，執行部分搜尋或最佳化演算法。

兩種方式都可行，應該依照新產品的系統規格、即時性及軟體分工決定。

---

# 七、把這些觀念對照到你目前的 STM32 程式

我們之前已經看過幾個重要資料結構。

你的 `shared_data.h`：

```c
#define SHARED_AXIS_CNT 6

typedef struct __attribute__((__packed__)) {

    /* M7 -> M4 */
    int32_t tarPos[SHARED_AXIS_CNT];

    /* M4 -> M7 */
    int32_t acuPos[SHARED_AXIS_CNT];

    uint16_t errorCode[SHARED_AXIS_CNT];

    uint32_t digitalInputs[SHARED_AXIS_CNT];

    uint16_t ADCval[5];

} SharedData_t;
```

這個結構其實已經具備了多軸運動控制的一些基本資料。

| 變數                 | 作用     | 對應 Active Alignment |
| ------------------ | ------ | ------------------- |
| `tarPos[6]`        | 六軸目標位置 | STM32 要讓各軸移動到哪裡     |
| `acuPos[6]`        | 六軸實際位置 | 各軸目前回報的位置           |
| `errorCode[6]`     | 各軸異常資訊 | 判斷各軸是否有異常           |
| `digitalInputs[6]` | 數位輸入   | 可用於感測器、限位等狀態        |
| `ADCval[5]`        | ADC 數據 | 可能用於類比感測，但目前用途尚未確認  |

但要特別說明：

**目前看到的是資料結構規劃，不代表這些功能都已經實際完成。**

例如，我們目前尚未確認：

* CM4 是否已經把六軸 Servo Drive 的實際位置寫入 `acuPos[6]`。
* 六軸 PDO 是否都已經正確配置。
* CM7 是否已經產生正式六軸運動軌跡。
* 是否已經實作速度、加速度與減速度限制。
* 是否已經有多軸同步演算法。
* 是否已經有光功率量測介面。

而且你之前貼出的 CM4 `DoCyclicPdo()` 仍然是單軸測試邏輯：

```c
g_cmd_pos += shared->nIncStep;

pRx->Axis[0].Target = g_cmd_pos;
pRx->Axis[0].u16CtlWord = 0x0F;
```

它目前只看到 Axis[0]，而且沒有直接使用 `tarPos[6]`。

所以不能把這段程式視為已經完成六軸 Active Alignment 控制。

---

# 八、最後整理：兩種回授與 STM32 的工作

這張表建議你記住。

| 項目     | 位置回授                     | 光學回授                                |
| ------ | ------------------------ | ----------------------------------- |
| 英文     | Position Feedback        | Optical Feedback                    |
| 感測元件   | Encoder / Linear Encoder | Optical Power Meter / Photodetector |
| 測量內容   | 馬達或滑台位置                  | 光耦合功率                               |
| 主要目的   | 確認移動是否準確                 | 確認光耦合是否良好                           |
| 控制對象   | 馬達運動                     | 光學對位最佳化                             |
| 典型處理位置 | Servo Drive              | 上位機或 Active Alignment Controller    |

而 STM32H755 在這個系統中，最核心的工作是：

1. 接收六軸目標位置。
2. 產生符合速度與加速度限制的運動軌跡。
3. 執行六軸 CSP 週期位置命令。
4. 透過 ECM-XF 與 EtherCAT 控制六個 Servo Drive。
5. 取得六軸實際位置與驅動器狀態。
6. 監控跟隨誤差、限位及異常。
7. 配合上位系統執行單軸微調或多軸同步運動。

---

## 九、我希望你記住一個最重要的觀念

假設今天 STM32 命令 X 軸移動 10000 counts。

Encoder 回報：

```text
Actual Position = 10000
```

這代表馬達位置已經到達目標。

但光功率計回報：

```text
Optical Power = -15 dBm
```

而我們希望達到：

```text
Optical Power = -8 dBm
```

那麼，即使馬達已經準確到達 10000 counts，設備仍然不能算完成光耦合。

因為：

**位置正確，不代表光學效果正確。**

反過來說，如果光功率已經達到最佳值，但馬達位置控制不穩定，光耦合也可能因為機械振動或位置漂移而失去最佳狀態。

所以兩種回授必須共同運作。

這也是為什麼精密光耦合設備不只是一般的馬達控制，而是一套結合：

* 精密機械。
* 多軸運動控制。
* 位置閉迴路。
* 光學量測。
* 最佳化演算法。

的整合系統。

### 延伸閱讀

以下兩份資料很值得研究：

* [PI Active Alignment 技術介紹](https://www.pi-usa.us/en/expertise/technology/controllers-software/active-alignment/?utm_source=chatgpt.com)：介紹光功率搜尋、梯度最佳化、六自由度與多軸同步。
* [六軸光纖對位平台研究論文](https://journals.sagepub.com/doi/10.1177/1687814015589503?utm_source=chatgpt.com)：說明六軸機構誤差，以及為什麼光功率而非單純位置會成為光耦合的回授依據。

**我建議我們下一個學習主題是「CSP 模式下，STM32 如何在每 1ms 產生六軸加減速位置命令」。** 這會直接連接到你目前 CM4 的 `DoCyclicPdo()`、CM7 的 `tarPos[6]`，讓你真正理解目前程式與未來精密運動控制之間還缺少哪些環節。

[1]: https://library.e.abb.com/public/7507a2f676634a87b1a099e78f15b187/AN00262-Dual_encoder_feedback_control_on_motion_drives_Rev_A_EN.pdf?x-sign=NqLdKtLDLBFVldSjsiGDlJWLcANj2OfPI7m7mvPLLiXoawWS3NaLgpi5f3oQGhEl&utm_source=chatgpt.com "Application note  Dual encoder feedback control on motion drives  AN00262-001"
[2]: https://www.pi-usa.us/en/expertise/technology/controllers-software/active-alignment/?utm_source=chatgpt.com "Active Alignment"




可以，而且我認為這是很好的學習方式。

我建議我們不要直接從「六軸控制器」或「Active Alignment」開始，而是**從一顆螺絲、一個軸承、一條滑軌開始，逐漸理解它們如何組合成滑台，再組合成六軸精密運動設備，最後才連結到 STM32H755 的控制工作。**

我會以 GMT GLOBAL INC（高明鐵）的官方產品分類為主要教材，再搭配機械、運動控制與光學設備的相關技術資料。

先確認一件事：**GMT 官網的產品很多，我們會逐項介紹，但不需要一次塞進一篇回答。** 我會將它整理成一套循序漸進的課程，每次講清楚一組元件，讓你有時間建立觀念。

# 一、先認識 GMT 的產品全貌

我查閱了 GMT 官方繁體中文網站的產品分類及型錄。大致可以整理成以下幾個層次。([GMT][1])

官方入口：

* [GMT 官方網站](https://www.gmt.tw/?utm_source=chatgpt.com)
* [GMT 傳動元件](https://www.gmt.tw/tw/%E5%82%B3%E5%8B%95%E5%85%83%E4%BB%B6?utm_source=chatgpt.com)
* [GMT 產品型錄](https://www.gmt.tw/tw/%E7%94%A2%E5%93%81%E5%9E%8B%E9%8C%84?utm_source=chatgpt.com)
* [GMT 光通訊與半導體設備](https://www.gmt.tw/en/equipment?utm_source=chatgpt.com)

我把產品分成六個學習階段：

| 階段 | 產品分類      | 主要學習內容                                  |
| -- | --------- | --------------------------------------- |
| 1  | 基礎機械傳動元件  | 螺絲、螺桿、軸承、滑軌、聯軸器                         |
| 2  | 手動與精密滑台   | X 軸、XY 軸、Z 軸、測微器、交叉滾柱                   |
| 3  | 電動滑台與電缸   | 電動滑台、精密電動滑台、電缸、工業滑台                     |
| 4  | 馬達與驅動器    | 步進馬達、伺服馬達、步進驅動器、伺服驅動器                   |
| 5  | 多軸運動與光學模組 | 直驅模組、對位平台、迴轉平台、光纖滑台                     |
| 6  | 完整設備與應用   | 光纖耦合、Lens Alignment、FAU Aligner、主動光耦合設備 |

另外，GMT 還有電動夾爪、光學平台、行星減速機、弧形滑軌、交叉滾柱軸承等產品，也會納入課程。

---

# 二、我們先建立一個最重要的觀念：一台電動滑台是怎麼組成的？

在介紹單一零件之前，我想先讓你知道它們之間的關係。

假設我們要製作一個可以前後移動的精密線性滑台。

它可能包含以下元件：

![Image](https://images.openai.com/static-rsc-4/EibxTVBuTsiE5-gSzQv1d59ZRDzYrtJnIVirL2NXGyU8-Su5BgLpuoNLsi0tnnn1mJ_RBSrPBPZu9V8ZUWsA70QCXvwMMDLeBArWWj8-BrljUqyqkeqipJuT3yYvS9stf2AxZtnsk_hHdlqOBMxW4naoGKy85lnoYYYMRrxQ7sRF58qoGJknxHtBVFIk_eYD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KFpB9qJf-v_rcljbV3qEKiGSyuxt6-QODiZiyStSPMJcZ_XP1EGNtLEyqbDfZr-ACEfhccr5eHCNWEX5c0C6aWqqUhUlsp8L4wIYu3MWAcp9X11lSsDhyGFm5q17Pr3Pj8z0F2K70VRlaRRpdtjdsn2tlCOC8wE0LJIJUHaGKqnwlzJ4QtSGdmEcIa5arc-u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pw7y41LxBOAkYYv34BTFQS0Y9n0XlFBSa6NJp_gcBej5cMsPO0DUgM5fEooD1tEGdxmN3CDkVvICam9mOpPKwy3RTHLm7IlPb2uXfBRrrgjy88IYnjW688Dvogng6pygVQfSTRmCSARYgHp-1XMaFoAnBTM9GTZEcw5jae367FMONTFAf6VX6jq-pLrKVO62?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mEClov8EaUUV9UYP5OAKUeavyJj4Zhitk1iwtE43zyKdAWgKS2qmbrgTNHNGee_4dy3xHXlofvupDZ12hNx8U3mYWAh_8kvubLjH2XPUVzX91bBOUjN-GoeUV5NqwMr46UNiTgel6wFenB9Yi1Y5kh_CMUOVpV9b_5BD810AZmNR0aQ2YuXsSvW05V7z0Hup?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RevnEoTKHPdJ6qDX-qs63KhZ-6TtujTEIWPTWf5GwWlAuGT_7BdHrMjVQho57z2zvc9JWdrcJlRGJWgGmRRP2h9TJl3vNxavk2FgMjtEZm2sQKXjXZkxR5o6AdksOJAg9dD8syRmfnK49G-R5bDmEdWPfQ5_qSyWABtaLI2oP9HtQFV-lPREJHcAQOcdBrZt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IkfMBqoakNOnehhD0XR9Q4FXs3y9Zrz7IvlZUgG5FO1pqEwcraFAdD9mkN4NuQYg75N7CxhMvYglDBO612YJWvARK-Ee4AMX2G8S7GzhMEQy6pI23ovZT7Bju8c0NS_xJBg4XSJ8kmt8LroIgngHNypaOA3Orvp2vFImBndpnhlkGoggFjZbZZ0EeK-MhNfv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xL_SkcnHF6Wj2LWiGFi9okbvY50DcEXww3xsnx4E8DrtPEs3-E0y1-hRvu9liIxfi2yR97j6CoCIcPQRWUra9njMN_jjyN_1LU4af8t5a_GCPMl2qLd63-0w0TKFbrayvHstduwAcyEMfPGZ1BhIyrUQuNDRhQ0aoE4CxuMv3iANH0jlP-_6zCuKpB1GsnSt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/R7PK_pOIMUOIXqtotxouSILJGeVg9T-WQC0Sus4UPFPeyFLRboMf61trlz4MQwcIwLE6-JiXb6vP9o3lkQN264ygdMzi58aMnHJWQS2V5H3Cz8MWMOWC2ko77KW-iT1gJt1pFd78HsyZVkVkbCFP4wMhz2MIL1hi19btSxGBzvfT5ZF5G3_wyzWdvuwyQ1Y-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/T-vgn0chVDoJvOaIC7ep6wyWlBK0y4sRJwzOWsLdhmbsUdt4F0pcUqKAKdFK66TOoLVnwGVthRQW7S88jNFm9T21hUqiqZdKNPJBJK1WaJNQ1iHZeICZzswh_zvy-8bIDrqSTJKZ42riU3dVYf7bPpqglbLeMik0h7cap_PXVLGM4T-pzrMFpGl-wnrJg4QJ?purpose=fullsize)

先看這張概念圖。實際產品的結構會因設計不同而有差異，但基本原理可以這樣理解：

```text
                 負載（例如光學鏡頭）
                         │
                         ▼
                 ┌─────────────┐
                 │   滑台平台   │
                 └─────────────┘
                         │
                   滑塊 / 滑座
                         │
                 ┌─────────────┐
                 │   直線滑軌   │
                 └─────────────┘

馬達 ── 聯軸器 ── 滾珠螺桿 ── 螺桿螺帽
                         │
                     螺桿支撐座

                 底座 / 機械本體
```

這些元件各自負責不同工作：

| 元件    | 主要工作        | 可以怎麼理解         |
| ----- | ----------- | -------------- |
| 底座    | 支撐所有機械結構    | 建築物的地基         |
| 直線滑軌  | 限制移動方向，提供導引 | 火車的鐵軌          |
| 滑塊    | 沿著滑軌移動      | 火車的輪組          |
| 滑台平台  | 承載要移動的物品    | 火車車廂           |
| 滾珠螺桿  | 把旋轉運動轉成直線運動 | 旋轉式的推進機構       |
| 螺桿螺帽  | 配合螺桿產生直線位移  | 隨螺桿旋轉而前進或後退的零件 |
| 聯軸器   | 連接馬達與螺桿     | 傳遞旋轉動力的接頭      |
| 螺桿支撐座 | 支撐、定位螺桿     | 固定螺桿兩端的支架      |
| 馬達    | 提供旋轉動力      | 動力來源           |
| 驅動器   | 控制馬達如何運轉    | 馬達的控制單元        |

這裡先記住一個重點：

**馬達負責產生旋轉動力，螺桿負責將旋轉轉換成直線移動，滑軌負責確保滑台沿著正確方向移動。**

這三者是理解精密線性運動的基礎。

---

# 三、第一階段：先認識最基本的機械傳動元件

我們從 GMT 的「傳動元件」分類開始。

官方將這些元件列為線性運動的重要組成，包括滾珠螺桿、螺桿支撐座、聯軸器、滑軌、微型線軌等。([GMT][2])

我會依照實際機械結構的理解順序介紹。

## 1. 螺絲（Screw）

![Image](https://images.openai.com/static-rsc-4/uMEWtTnjbFedu5qEqLUPa_HpasiTr9CaJ3byHlXKvo0GaGHPt-zmnGTudCdc1IJiSNJgP0kHbLt_ur6DL2tn2SnFYiDuTtXVcRWxYr_hMZjODqdCZT2cWqZRuQrW65yPt7Dzf8cAryiX5Po9beCwjTf_1-Nt6hE_IrtbUee9jmWD6aZOQQPCxoKh_oyGN-CH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4N5QumTSHTkA6ZcbxqtgJKAaKDFpBsOXam9hRZev8tOh9maJ1vkDPxQf8Ohze9KCtGpDrkD7cyT-kfY0pe_y6rOZzarLlZwy3Aplz-TZdT9FwUMjtbuiVbfClt9fbUwLyIXwtJsrPqLemwMM5t-2cMX2W3HJ3ksqguNzJ9zBS9UudIg6n_ouKJq-p6bIvy0Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nEY04fJXyqlwOBJK-DYRIsuDUhwYwPFLT-Xsgn10LEcxliILnw7xm4Xn_T0i3MjC5GnqQ8rgyqX_9ul-lXn9otimKql7vDpej2IsX2QWRkAk_2uqWsxPcf1cylJnAOrh2MK5qf8VYYnOh3jX_WK3NAR5U70ZZCSShtYnJGpcy-LltnG3aE7POqUtIyTgopQE?purpose=fullsize)

螺絲是最基本的機械固定元件。

主要用途：

* 將兩個零件固定在一起。
* 固定滑軌與底座。
* 固定馬達與支架。
* 調整機械結構的位置。

但要注意：

一般固定螺絲與傳動螺桿不是同一種用途。

例如：

* 固定螺絲：主要目的是固定零件。
* 進給螺絲：透過旋轉推動另一個零件移動。
* 滾珠螺桿：透過滾珠傳遞力量，實現精密直線運動。

這三者名稱很相似，但功能不同。

## 2. 滾珠螺桿（Ball Screw）

![Image](https://images.openai.com/static-rsc-4/DYmB1MkPRoyhfgd7JrjHcyl7g9qSsHqCAhH3k-R3iJH-Ala8jMZsi-smMH1aMLWI2ilTydX8Xhye0Nde5GkIYiq6reOatlzZomizB8tatefMHUjyhMvFq57Uko0_CPQeCdSCUg_XK8I7nyxnJKt6dgD5jElu0DCIFB-t2SjDCuZje_eyFvx-X5AKZpO3mVKg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pU7-W4dUtuzBdF30OhExC-ofY5jGdunNIITkI54ys2UVaA4vswr1HjheVNyw0t96qfD7ieoFCGmX63lmaue3WAXuqxuisI2pxS_PpznVE5J8D5KjfrGM2kHhvewqn-EpClfsG7KKK-nr0Uvk2bzNOHGoBk9nq4lT1XGNlu50DjggIC1y_qwi-1kJ0ethFZll?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zped5003FlogGo60F5eGDTx4H6pjkZlpehiunZV7ZGbV0dsQmsEU9Mt1vVGgTxnneqDqGIyPo-HVh9Wz6ILtP9_cS146F0Hc6dOS9qai7pqsM0vfc9lpmZVvGXz7r6LGZ_xBOZNpHYUJx3nCjB2uR33Y7OZ-c_tbOJrUJOyDlPGtPahgvSNUHsiqJdOjxSmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/w5wmPPToZW_HyYYKYoHzhbjyzk8EQgSS0e1AL1zV6f6FMWxPtNgeKAw54Fc5W3tj3dvJEX21loZKHqYtf2ufqPVKXyA1OBVUbAHpHMdSCQWmEqG3KwCcPIPvO-Ysf49vzrzy3pfjT6AdPiXAP7KVP72_SIH8WyuEhuwZjOQJkz15_R3_cZjycDeuXzxTpCtM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dzVav2LCI_af631MvSEB2b77XO0_bmN40Uyby3j7hjZInN72g7H7ivu1oEDpfuvxqS7W7U0_eeXT0vVE7E4cMQO14Zo-7ZUwSYXX2EokNI8afQ30H9wHIQMPmD1vA2FQqeEsrbAkuV8gxopnV6lBiRInRtLw5LH-nns_4BSddvarX4URALk5izPWHkm0YYOS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8WIvERUJ8WMOAmuiPnyFIGd1dqo2s1Zkvjnsho7pKlFU3aCGfsjseH-OMoHYmo3sPsMfVjtQvKSAJVTHwoGc3BolRtQBRxEWCZ3PH6J4hu1LeVKIJF2KbB78wC6qhjv_5GVKOpOaBtJiWo232MK1oFH39Dve3HSTOw5RLL_1CyQDfkgNHgyq4z7i7ftTopzz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wNeqduqtVzkl9d2tZ327ZUedsRCqcM3U8HO3xi70EF7vD82y7TlF2rqC-fz4osEI22CJ4nlG4H_P3M4jjvTuJXnZufeZ7RIbxUGKOucRWVBqKE3V36iFz3KAcS8ESL14R45pRfD_pIE6bk1ztbLDSNWZ-y_i8n1j0Q-MZmRRctOQ2uxU0ApDIvks-0WbYOYv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uzjF33J4uulqwgbAjGpSGqAPZRTbQoi019rgcSnHnPC1O6H1g-3RZm-c4oKSgN5uM-ArsB2o6_VB7353tlWP8i6YIvS4kGMS6aKvWL8JAOFPJ7XC2uZy1yay5ev_5XBdlPlHrYmiUkN-St1vdOsLXoQsgP7xiVNUomaefeq5nG3cT6G2YZHFPWmbagAsQEb1?purpose=fullsize)

這是我們接下來需要特別熟悉的元件。

滾珠螺桿的主要任務：

**將馬達的旋轉運動轉換成直線運動。**

它主要由兩個部分組成：

1. 螺桿軸（Ball Screw Shaft）
2. 滾珠螺帽（Ball Nut）

螺桿表面有特殊的螺旋溝槽，螺帽內部也有對應的溝槽，中間放置許多滾珠。

當馬達帶動螺桿旋轉時，滾珠在螺桿與螺帽之間滾動，使螺帽沿著螺桿軸線前進或後退。

例如：

假設滾珠螺桿的導程（Lead）是 5 mm。

代表：

* 螺桿旋轉一圈。
* 螺帽理論上移動 5 mm。

如果螺桿旋轉 10 圈：

理論移動距離就是 50 mm。

這個數字非常重要，因為未來 STM32 要將 Encoder 的計數值換算成實際的滑台位置，就會涉及螺桿導程、馬達轉數及 Encoder 解析度。

## 3. 螺桿支撐座（Ball Screw Support Unit）

![Image](https://images.openai.com/static-rsc-4/FLDqL6ZwiOsVPxi9_j3LcD37ZUOtR0s6MFs_4I5dw9MDi34W4MT4dOKdo70wikxFqm95oMy9ogbSdegt1TfCAcY7yZL2yfC9Nk1aSZWKkh6VRWOxk0QTYWeW0mm7ue2ZQrnBfcj6I0yIefeY8RW0GZIG_8Boqve_-dVUV49ND8t-J8d1pjDc9UynhgAEokxS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WfEue5cQk5I3PAJKCxA0rvySOeEOUxG4Zf5ViTTbeQbksT0SXPmjOiGAOQE5bAiXRZ7Rla1AfpL1j5hB0QzusrbBhusi4dTr5IHGm8eu2-yMANqS1NSCirMexS716rtxdH6Q0XYLKrt53BKzZ61tcDHmEBHlp7eDsUhQ03OqL_jO_4cZ11bcFe_P21-EMbfs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/JDRWi2CqSJ0CMDTUpYm2n024WTysZrQxEZW47wJVIDrUKBvajSffapnowSF_8v2oKMqHyVR-PextRFJf2ZqqA44wqimwwxXmrUtvEIJU4Ds_d8HlBNmOsJV52R2t-HxPbpobkJ44eBBUjRW0lsWcCFXJ1M-HuziR5tgHFd7WoIBcOrIFMHABXAwLebmNPf6a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/y8L_GxkWeIM8pE0ZxvpPlM86Lmek7Nngeats85SJC7JJZ-TQrEE-WRZpe6go42ID-gLWF1HN5QTLO_Ml2yTXuAuiqRF4N-HgXa-NxAwNOp9lRIJecipNo2fdibL57hlCDKUcoJDZHipxSgDil9BWZe0sXH6c3yOC9-BgVkdDUR8XuFVP33hgjaFUd133B2ec?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6dXdFMWJ-tOSow12WK7GBKpas6Lzu20p9sFQDd6ayhgVGTNw1GcavwLoibke4qBba7Z1adJ4NqVllTyeTFJJakKbe4JDEgvL1xKK_sVJknrYskbW9aXVFxjgwI1ICpGflfA2adsF0DDz7uM5t8x-cwIXRn3EkzarqGTv9PmP803LH92L84Y1rtNlAnQtn1iA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VogA0doSaJ07yHpmbdt21x0eg3iT4DJGf0w-MP0dUZ51lc5XUc4dQuaUKgug0553_pXhpEVRaQR8pfcK3Kw8DePmYX07q_pZNP9Xw8JvUzCsB_vFMK0GSgLqRJCffwS2bu1IjL93wjrSY20JSYBRrhSpeCFFfRXjKjAtf8O_40nuhjBZfWvLbSMxYtc-k4jK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NVd9NEhXJ-FY8xrpXxEXMqKyOCHbxt2CvttyFJcs7HzHPEhnHG-AVEEnwmkA_AuJJ82I4JBRu4_Buz_NgLViq1jDCMzaUe3yVQycMXWIoFwu3xRQC3OaGW0K67hmUL9qJzCxLzMCNtWlZR38TiYmx2_Dh_smLbVkAq1N_T0sKXjgU9oDD6CqJB7rkocxilJU?purpose=fullsize)

螺桿不能只靠馬達吊著，也不能讓它在旋轉時任意晃動。

因此需要支撐座。

GMT 將其分成：

* 固定端（Fixed Side）
* 高荷重固定端（Heavy Load Fixed Side）
* 支撐端（Support Side）
* 鎖固螺帽（Lock Nut）
* 間隔環（Spacer Ring）

它們的工作是支撐螺桿、限制軸向位移，並維持螺桿旋轉時的穩定性。([GMT][1])

可以想像一根長軸：

如果只在一端固定，另一端沒有適當支撐，軸在高速旋轉時就可能產生振動或撓曲。

所以支撐座對精密運動非常重要。

## 4. 聯軸器（Coupling）

![Image](https://images.openai.com/static-rsc-4/3PzrrGO_qvRLp_Bu6gX8a-XlaZz7yDLbo2SUt_4ft18_TudVe9p0KI0yQyxoplxJEcp_GF5pIJm3Xxxd1BpdgUxwDKELopn1DJTLdzBfOeAkJdFCXfZNcsHfc7Oqm5eGJEepqDQj_LTdz9unsf4ajYNPaREdVn63IfdVaeDGuFTiGuVQ6NVPKMM-n0Z89-lc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9qrqt9ZiE9HyIIBaQNHKhgs2hFGIBi880ubOeLU3Q-oiTC1qLgktDT5ObuRSiTKIyTKYOlkQgrFg3W1OBF8o7vtgozQKCBO2TwuSmbr9NrNsLOIEN3j5sRYluXJlZ3bbf_zu4beYTTkwo3ujIGXkuFr9I5Y_w71KegogjtE47HLkxm5akdLJ0MmpEBHemM0r?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vuthY1brjAFdpoVEHwIrsFilntM4fdYEIv0FPKZqqHBhT3Z2Hpzz6W7NF_cs1CQVVcf37YQuUy9IUPQ8Wm_3tm8uimAMgm3L2NJNdRFA-jyWH-_zoffwSks_wVkR4ipewFRxWAQa4o_L0YecD_miNIQDhEr8eLxRwyrE7o-uAPSTrs_azYzxZSSkXcsXQNPU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8RHZCBli9EiiUWS6fr_UECZXwfXPetB4DQ97naTOY5mOwRiFWrkWtitcOBih1_qxEalbqAfSd64NmoXNhwjhB6Jkenljbv54k2ds_6IBOrloSRAo5ErfFnKJzlyne_kgHzIkYj_gAh72Lno3OdQVzAqiAG8qBhAOapSEJ--X14-tXuvfXo8oJ8pAn0n3cWEe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/i6co5VmeZiuluhoovmbWN1YXmIZwUn5V1VgsLk0goTDzwmttrSGFDO3aqdQ2RPTPTS0gngMTh9X72hJvzYq5L09A5lE337gTi_TQRBsK3qRb7IUTnZ8LHOtvd-R-JaCHJkdOeVuIjFa_gY5BbO6EjmemoJS6RJZMVWgo9vwP5EsDf6T3bUh-a8BAbY3GH9A9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/V4F746hnJQstvCzzT0YVqRWYtMxf204SZSrAOANcDH5Fbl9YogwOlHc0zYqrgAovz1pL9yHDO0wCLDATcF3tZriteGNWkCOn11cRLenyzxqYA2Amwt5gjzrtklZMZwk2WjeO0QxpRJTaBBEzM0oekgjbfXNQkOoeSWfQXf8LGZUQ_SbWuGWVfdepkDrJWbah?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/S_GCMhaG8i0m_9BjMHBOnHITsaXA1rfaH4yaJdlo6LFoFca167unZoMxaR0baTo--ogESZOKF1h1VTAiCrLDBmA50Y-vhjzEjEurwJ8lM0LXd7mI-z_eIriFolV8CXKbqLdyqEUFej4bnq3WK2HD4vNQ2VIElNDBjZi3vipStLDKORfvZ_8m8CmEtD8J4K-u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gM8GN9tnq_1jdjWMVSgUqyQVc4ifQfesybnYlogxADvVGxco4HuHcrvOH2vIG5SNk-R8NbtfE1VNgKSW9GNRa4PXSG7UsW1OTh5ZF4BfCRwvIe4SBtJM-0dn2ViZuKJdMDYBZGxq8_f8BlM4sTeylyfy0sbfB3Q23As3EcLGUsBUa25_wxgnSoKdwgb15iFL?purpose=fullsize)

聯軸器負責連接兩根旋轉軸。

在我們的線性滑台中，最常見的應用就是：

```text
伺服馬達
   │
   ▼
聯軸器
   │
   ▼
滾珠螺桿
```

為什麼不能直接把馬達與螺桿硬接在一起？

因為實際組裝時，兩根軸可能存在非常微小的偏差：

* 角度偏差
* 平行偏差
* 軸向偏差

撓性聯軸器可以容許一定程度的偏差，同時傳遞旋轉動力。

GMT 官網列出多種聯軸器，包括撓性開縫型、彈片型、波紋管型、十字間隔環型、爪型及剛性聯軸器。不同設計在剛性、容許偏差、扭轉特性等方面有所不同。([GMT][1])

## 5. 直線滑軌（Linear Guide Rail）

![Image](https://images.openai.com/static-rsc-4/KIfY3G6kn6Jx3Bwx2Xoifpshvc1ohepihjb12A64pM6CLiEAviTsMF_HAFeJEYvjW3Bk1gIpBwdlGwnZCCKCOnyUeQD9fypAhVUJzOeO1mPP301FnBbYr1aNA-eenHZJ4h6JX7l_fZRS6VuBjCqst9ICGrPndjByluRhlG-b_-74Fq9lCQD0BiWoQHIwsm6p?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8JMPbo0XVr2evAoaka6L9kcaOGz0aM6FW1oNXgrcGEGZHWT4xxCyRaoPKYmYIAfU_W-J6f2OEXF86Mkg9C9hGYlrMrmVr5ZDti9nwXn05xmgPBK6aNVvKCKSfyiXGnrk3ltCB_KI_5sscbQf5d49KzZR6Dg1HXJp9oeTetYEoTtjnnSGz5gn7LE43s0Jizvf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uqjnfu3gaih8Jl25VIwp4PeEKeJWsQSQrwSoF9MQGDSflvfPXUS6PpfvKvlgxpvRaKeGp177-Zg9qd3zGwtjZJ1WxTAxnOqK-9d3w8RiVbl245xo9ub3NUDo3MrTD7fpAF259tUQisWBYxi5ZkVWF7duIExC-gPYHFAL4J_3G8kNw4W3jbaALm9OJ2OOkzb8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yKubvOYH3euGO4TmG0Lnq5Vtc46qNj1r4cRhBp1EWK0CLH-mcVHs17A-9S5N8AjIrhhdcyhO_pqbM5viQ9G1l9ciTYh7ev6TcLRm8cWDp-xCdA2enKxj6oFlZb7xW9TWJQzDGALBa9K0q_GG_hnLH5nDagZG7Y8BVvXgdhOLOJc9c5mpihc-V5Mi2O-WFpL2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oworj1op_xl34IUh25RSGV-8f6beAL1WuONCjK9-csywjazfM5m0VNqVuxqlof_jSJlluvj0-m6EMXU5Os_URLtxcQ9xJ7dClVQ8TaJR-Xip2-XsBZOI_KNFwqpRRmeo-SueON90wxbeE1hP6Elk_5Mao8Y7h3uQMtxPE74qHThGT3XnETdf4-ZvjKYqjImu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OfEG3SpN0LSGUmMVnOiQaoZSqmEgjerw8bp_nRwBHCTbCgoyewvjbFOe6lvICQK5MvnITUhBb7pqjsguTPVINzg11YbC6zW_1gStax19fsKgbjBRPkKG2zK2kfs9wSOexi5SCzTmLWgvxe2DoMJ_iXAPjNy25wo7Q7b3Ge23USDHKehKUyz2KIhGIpuep2Aa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/s-ePuPLIfqAug7uKleM4tU1xJi4K9wSj74RbeO9Wg8DJuYYstKmBdapqeJl4jY_Czx_Ab5PGCau_Deaq63fF_WegfPRXQhrOsRQIe4J2BLIRA7SeMnRQYjMZ0BQC3It3pwiHMBZszQZfB7P27iFZAjwAOW3_4C6cKKgrPUKaBsjiI6BaKR4Y8eOuI1rBVjSd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hMZyXAfHAfBtafyVKbguRK3JGdmiefYCz8ZTLT5_BmPYAAQ_UKikrN-IFiBnGDgSwhouy5U2oKs9QbGjKjcht-NnwcTNS2R-3s4T_iDVn0oBO4nCC1PVDbq-8ijmf-_QsZqtdoFl4Q-p_nyTSgEszVw8nAt2ADZAMImb7mw5gDNLXsJjOra8jiOUPUCkqk_o?purpose=fullsize)

這個元件非常重要。

請記住：

**滾珠螺桿負責推動滑台，直線滑軌負責導引滑台。**

兩者工作不同。

假設你用手推一個平台：

如果沒有滑軌，平台可能會：

* 左右偏移。
* 前後歪斜。
* 產生旋轉。
* 因為負載而傾斜。

滑軌的作用就是限制這些不需要的運動，讓平台沿著設計好的方向移動。

GMT 有直線滑軌、微型線軌等不同產品。微型線軌又有標準型與寬幅型。([GMT][1])

---

# 四、這些元件如何與你的 STM32H755 六軸設備連結？

現在先把機械與電子控制串起來。

假設其中一個滑台採用伺服馬達加滾珠螺桿：

```text
              STM32H755
                  │
          產生目標位置命令
                  │
                  ▼
             EtherCAT
                  │
                  ▼
             伺服驅動器
                  │
                  ▼
              伺服馬達
                  │
                  ▼
               聯軸器
                  │
                  ▼
             滾珠螺桿
                  │
                  ▼
             螺桿螺帽
                  │
                  ▼
              滑台平台
                  │
                  ▼
              光學元件
```

另外還有一條回授路徑：

```text
伺服馬達內部 Encoder
          │
          ▼
      伺服驅動器
          │
          ▼
    實際位置回授資料
          │
          ▼
      STM32H755
```

這就是我們前面討論的 Position Feedback。

你目前的 STM32 程式中已經看過：

* `tarPos[6]`：六軸目標位置。
* `acuPos[6]`：六軸實際位置。
* `errorCode[6]`：六軸錯誤資訊。

這些資料結構可以讓你把剛才介紹的機械元件與程式中的控制資料連結起來。

不過要特別區分：目前我們知道程式有這些資料欄位，還不能因此認定正式產品的六軸機械結構、馬達配置及所有回授路徑都已經完整實作。

---

# 五、接下來的完整學習順序

我建議按照這個順序進行，避免還沒認識機械結構就跳到控制器。

| 順序 | 主題               | 你最後應該理解的事情            |
| -- | ---------------- | --------------------- |
| 1  | 螺絲、螺桿、滾珠螺桿       | 如何將旋轉轉成直線運動           |
| 2  | 滾珠螺帽、螺桿支撐座       | 螺桿如何安裝及運作             |
| 3  | 軸承、交叉滾柱軸承        | 如何支撐負載並降低摩擦           |
| 4  | 聯軸器              | 馬達如何連接螺桿              |
| 5  | 直線滑軌、微型線軌        | 如何讓平台穩定移動             |
| 6  | 直線滑台             | 如何將上述元件組成一個機械平台       |
| 7  | 手動精密滑台           | 如何進行微米等級的手動調整         |
| 8  | 電動滑台             | 如何使用馬達控制平台            |
| 9  | 電缸、電動工業滑台        | 不同直線運動機構的差異           |
| 10 | 步進馬達、伺服馬達        | 兩種馬達的工作原理             |
| 11 | 步進驅動器、伺服驅動器      | 如何控制馬達                |
| 12 | Encoder          | 如何知道馬達實際轉到哪裡          |
| 13 | 直驅馬達、行星減速機       | 其他動力傳遞方式              |
| 14 | XY、XYZ、多軸對位平台    | 如何組合多個運動軸             |
| 15 | 中空迴轉平台、迴轉擺動模組    | 如何產生旋轉與角度調整           |
| 16 | 電動夾爪             | 如何夾取工件                |
| 17 | 光學平台、光纖滑台        | 如何支撐光學元件              |
| 18 | Camera、光功率量測等設備  | 如何觀察及判斷光學對位效果         |
| 19 | Active Alignment | 如何利用機械運動與光學回授尋找最佳耦合位置 |
| 20 | 完整設備架構           | 如何將機械、電控、通訊與光學整合成產品   |

---

## 六、我們的學習方式

我希望每一個元件都按照相同方式介紹：

1. **它是什麼？** — 名稱、英文名稱、外觀。
2. **它長什麼樣子？** — 使用實際產品圖片或結構圖。
3. **它由哪些零件組成？**
4. **它如何運作？** — 盡量用簡單的生活例子解釋。
5. **它在整台設備中的位置？**
6. **它與其他元件的關係？**
7. **GMT 有哪些相關產品？**
8. **它與 STM32H755 六軸控制有什麼關係？**

而且我會特別區分三件事情：

* **GMT 官網確實有這項產品**：以官方資料為依據。
* **一般機械或控制原理**：以工程技術知識解釋。
* **你的新產品是否使用這個元件**：沒有實際資料確認前，不會直接認定。

例如，GMT 官網有高階精密滑台、電動滑台、伺服馬達與伺服驅動器，不代表你的新 STM32H755 六軸產品就一定採用這些特定型號。

---

### 下一堂：滾珠螺桿的內部結構與工作原理

我建議下一次先深入研究滾珠螺桿，不只是知道它可以將旋轉轉成直線移動，而是要真正弄懂：

* 螺桿軸與螺帽到底是什麼？
* 為什麼兩者中間需要放滾珠？
* 滾珠如何循環？
* 什麼是導程（Lead）？
* 導程如何影響滑台移動速度與解析度？
* 為什麼精密設備會使用滾珠螺桿，而不是一般螺絲？

**我們先把這些最基礎的機械元件弄懂，再往上學習整台設備。**

[1]: https://www.gmt.tw/tw/%E5%82%B3%E5%8B%95%E5%85%83%E4%BB%B6 "GMT GLOBAL INC-傳動元件"
[2]: https://www.gmt.tw/en/linear-motion-components?utm_source=chatgpt.com "GMT GLOBAL INC-Linear Motion Components"




很好。我們就正式開始這套「從零認識精密機械與自動化設備」的課程。

我會把你當成完全沒有機械背景的初學者來教。**不預設你知道任何機械名詞，也不會因為某個名詞看起來簡單，就跳過它的用途和原理。**

另外，我同意你提出的一點：不需要每次都把內容硬連結到 STM32H755。現在最重要的是先理解機械本身。等我們學到馬達、驅動器、Encoder 和運動控制時，再把它們連結起來。

---

# 第一課：認識滾珠螺桿（Ball Screw）

今天只專心學一個東西：

**滾珠螺桿是什麼？為什麼精密機械需要它？它內部究竟如何運作？**

這是非常重要的基礎，因為它是許多 CNC 工具機、自動化設備、精密滑台及半導體設備的核心傳動元件。

## 1. 先從生活中的螺絲開始

你應該看過這種螺絲：

![Image](https://images.openai.com/static-rsc-4/zCgdu6dZcuODZl4rVeNNTeqKhIV14GaFuAV2MTXcz0j_7CMd2-UC10_8uJQNGFdwHYtDmNALabj_gc3c0ndUuLH2O03DuiHFMk2JUYEtD4M2nSRMu7VB_PBWhh4FFWSs_Vw2mM1FeSyK1j9q6UrEP66GLK4sGvR4JZVT-KzY_NUtjxbln3l83WkrtShD-I_a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZX50Hzb3N6Gd9D06v8H161CYUZ6oKdjGrpHFO5zMimP_BMR78LO6X8jrEa0qmsCbOiWRpwUkpPsSw70jWn20Dp118Z_aT6Zvl2M3Lwp1dbMV1gvDOcd1L6sCSDZxDPV0_MbDeDVlJVt4AXU-xEuxeLvnCvJpTKwd9MSCiBFXVBCR9a5qw_O8ooV70NE0XwN2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/F-EqSAYCrscMdPLhwnBumzAaV13AeAk9qrtBCCeFl8fnWZXYHic9gdqMdrTGy1QszRTpBnquWQK2TAaAxfOZAamhFgw0thjXAPC4sthTB6vjOX9ItQR0k7v-Wf4TyDFgxP1yVi2gNVGQv9wTuGFhRQCZ_UcC4aPuIymnBShFRuJ_IEEmGcf_0ad_mHT09Q5X?purpose=fullsize)

螺絲表面有一圈一圈的螺紋。

當你旋轉螺絲，螺絲就會往前或往後移動。

這裡有一個重要的機械原理：

**旋轉運動（Rotary Motion）可以轉換成直線運動（Linear Motion）。**

這就是螺旋機構的基本概念。

例如：

* 旋轉水龍頭的螺旋閥桿，可以讓閥門上下移動。
* 旋轉螺旋千斤頂的手柄，可以讓汽車慢慢升高。
* 旋轉虎鉗的螺桿，可以讓夾爪靠近或遠離。

這些都是利用螺旋機構，把旋轉轉換成直線位移。

但是，普通螺絲有一個問題：

**螺紋與螺帽之間主要是滑動摩擦。**

當負載很重、需要長時間運轉，或要求精密移動時，摩擦就會成為問題。

因此，工程師設計了滾珠螺桿。

---

## 2. 滾珠螺桿是什麼？

![Image](https://images.openai.com/static-rsc-4/iMeVhVK7lZJ19gW1hN575S90lruXYKRYjA_OH31L_4E6SdVuGZZbJTcqM6HghPzkQUz5rgQ9oyQdQ44BbBGN91l5RFi07MIG5Pjx5DalaP9vRxG_wttIToeMG31FN_pf2jxykgewzOGq9KtJ-aaZ-msgqsggsw4Vl1o44FRe70njYefEEJVLZSUbCwbxnprR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pU7-W4dUtuzBdF30OhExC-ofY5jGdunNIITkI54ys2UVaA4vswr1HjheVNyw0t96qfD7ieoFCGmX63lmaue3WAXuqxuisI2pxS_PpznVE5J8D5KjfrGM2kHhvewqn-EpClfsG7KKK-nr0Uvk2bzNOHGoBk9nq4lT1XGNlu50DjggIC1y_qwi-1kJ0ethFZll?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kAGjxwUWX0SQDbuNtG0T_8p3rvVv10eKhW4RWJEM9WhqkCNnroHa39LM8VBo12hvSueT4ldm_R8QLm_0LDO8I9BrTZ8C6STh_UfwFZQ1_T9yUTjqDOR6V8Its8VqbEO2ue9YY6VwPcqtu9rsPDjX5bSperOXLScySEWitMvGSG62IEwCUarorhnH6NwYBPp8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u5RwouvrDXx4vuCdm5psuLjnogopoRP7mnoQnvw8P-8xQFyDzJWTfNsij-FvgQzpkoUo7kkXDKlvtJgNJC84w727DGm3imDCLmNCoY3s431u6h6LiOgJwvE7xjWZKxvkrw1EW1bRHTTLaYTc8fDNmVX2oMNa2Ia_JYtiXrYoMHtBmZ4Rqi3hQlfhOEn6utZ7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/08X_xmOl5ucPfIjhcPS90o_AuXs3PCD_u0S3dV8M0G-xaqMMFqbYlBGHwxCwIN_3Z7pvCd7nsMHfoid-hYYg9vH2lDZFvqhR5FOjf1iEa4y36l_fYin0roiNNuZYDIdISiBoVBmYUaqYqjJYA8rG7QvmPnC3afwQe430yudKxd44S00362E8tueu9Ie20Ju9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HIkUiLPDTcFQOJ7aVzq2-s5n9UvwoGnoRPsfs3lRaeiYY6TpHdThqa9v1RQ0UGfs72NkTTEs_avVrFZ823i-InCtgQpuGoPA0yseummham1mluwlMKQJtRLl62kU7hJQJEOMrEpTVrV-9freyfX_jNAmewhdzWiOB11ISkHJGXYMAmerjvgrtkVnKMCiBn0s?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oCajEJYLD0FN0ZupiLd01-dlcqGpdQUW5CTQ28rqjWFsSUmfXYZEA14JKQDKZGDVMkBheWTi8RI5A1Dg3ttLtjLUC4nrMjMK2X_QjSa9NBwgWspybPCjbwKfQWbr9KI5wYnt4TwQInLxvLPSGQtY4UAVH6gdYj3W3IPTp2bgcIbR3M6hYTXqS33X1_YWi3i2?purpose=fullsize)

滾珠螺桿的英文是 **Ball Screw**。

它最重要的設計，就是在螺桿與螺帽之間加入許多鋼珠（Steel Balls）。

先認識三個主要部分：

| 名稱   | 英文          | 用途                |
| ---- | ----------- | ----------------- |
| 螺桿軸  | Screw Shaft | 旋轉的長軸，表面具有螺旋溝槽    |
| 滾珠螺帽 | Ball Nut    | 配合螺桿運動，通常連接要移動的平台 |
| 滾珠   | Steel Balls | 在螺桿與螺帽之間滾動，傳遞力量   |

你可以先把它想像成：

```text
         滾珠螺帽
       ┌─────────┐
       │  ● ● ●  │
       │         │
=======╪═════════╪======= 螺桿軸
       │  ● ● ●  │
       └─────────┘
```

這只是簡化示意圖，實際上滾珠會分布在螺桿與螺帽內部的螺旋溝槽中。

當螺桿旋轉時，滾珠在兩者的溝槽之間滾動，帶動螺帽沿著螺桿軸線移動。

### 為什麼要使用鋼珠？

比較兩種方式：

**普通導螺桿：**

```text
螺桿與螺帽接觸
        ↓
     滑動摩擦
        ↓
    摩擦阻力較大
```

**滾珠螺桿：**

```text
螺桿與螺帽之間有鋼珠
          ↓
       滾動接觸
          ↓
       摩擦較小
```

你可以想像：

* 用手推一個物品在桌面上滑動。
* 在物品下面放幾顆滾輪，再推動它。

第二種方式通常比較省力。

滾珠螺桿也是利用類似的概念。

THK 的官方技術資料指出，滾珠螺桿因為採用滾動接觸，所需驅動扭矩可以大幅低於傳統滑動螺桿。這也是它適合精密、高效率傳動的原因之一。([thk.com][1])

---

## 3. 滾珠為什麼不會一直跑出去？

這是初學者很值得理解的地方。

假設有一顆鋼珠在螺桿與螺帽之間滾動。

如果它一直往前走，最後就會跑出螺帽。

那麼滾珠螺桿怎麼持續運作？

答案是：

**滾珠螺桿具有滾珠循環機構（Ball Recirculation System）。**

![Image](https://images.openai.com/static-rsc-4/wNeqduqtVzkl9d2tZ327ZUedsRCqcM3U8HO3xi70EF7vD82y7TlF2rqC-fz4osEI22CJ4nlG4H_P3M4jjvTuJXnZufeZ7RIbxUGKOucRWVBqKE3V36iFz3KAcS8ESL14R45pRfD_pIE6bk1ztbLDSNWZ-y_i8n1j0Q-MZmRRctOQ2uxU0ApDIvks-0WbYOYv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Hbspx-c_Qrs0pFxmCHYFahYheqIhQ7IzV2mAXvjAgFDOD2-ee_028WcusbmHohg0uQIdDPMeIKrfyN6kFGrjtYlM-o76pigXrhxUvSQdwPA4L5VNVq1gADTfcbWZ9H-dOVm4sQI6LkRQrB5QWN1fNQB8T-eKnCVM6iWSTuWyghxuui6OYB22RCuOBJiiU7Mo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/24PcbwrX_xkfq7DRxzoJZ7GTrRCjslZGPajsC-Q5qaR0PRen5pstaAXvJSigcwsno5VzRtyVKC_KK2lEq4hHRNlaH9OPIKEnR8eeAkTqJp5wMPLd1hsZUUm9yejUGo7wmPnl4s_y_8Ixcfx5hpYZbdrfbrN5dxn2wOXob-6PdSb7eg_vvC10K7WVURtcnoEv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/w5wmPPToZW_HyYYKYoHzhbjyzk8EQgSS0e1AL1zV6f6FMWxPtNgeKAw54Fc5W3tj3dvJEX21loZKHqYtf2ufqPVKXyA1OBVUbAHpHMdSCQWmEqG3KwCcPIPvO-Ysf49vzrzy3pfjT6AdPiXAP7KVP72_SIH8WyuEhuwZjOQJkz15_R3_cZjycDeuXzxTpCtM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/f5gbwt40SwlO_neRKKjNcij6XNJj8kOU5QFMP-jwPJn7McaJ_Q1-bQiGMyyiY5F_N4eGSVgK1OMTPzfuxtfsyNK_SlkNcRYxlJyjzUSWnVE4e_KL-bZwYmsZU0UJHfgZZlk1N7ZYuwA5KUfjGIl6jm1hKq_Ze7JZpWPXmnC3_sxDWDIgEy2H60kVw6Woa224?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sX3k15MV2j54BytRVxJq7BKH8-mRyYHhb7GXDa6cmyChJxiM0FxCbYFppi_7tLUVFxSyTY1vQXC8VXGTP5EWvnqIQhPdtSDSEUOx3xQYPGfT0Fo2lXtTqNZkaBXE587nI5AUbWGJ6pBuqyZOXw4FZt2yB3TiAiPXX_h_K_dFT-IKj0nQJUyGl9W7EDJDW3cK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MwB5WAfQTNajBpLFTlMgSDD2yU7rXcwiETsac5jb7QSdhZyEwsQsk-kuDUJIaaJcsfKl1jX8Q7gae2_92Zla2jgieeXPowa6i-kSFiZ41PSmWHr4ZQ5vTYw6xCgYt6vt5vHDX8SITeWrWzwnNSxHic6dQrsuWtX8xpMyUkqBb3uWifTOQrKh5QNPXGQKE51f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/x6f5aA5Zz8YTvm1eAo5fsdpj87fYETlbFRvw-xAqBXBL4OiPVWcgOGcQRGu1t-4-MnSaEedw4aF-75oyYqDYnErG6-8Ln1BLBclO_47yru-oPzNcKvrCREeZT_cFiyh__yNoJrhMGj-8q3wT62j8pQAivvPe88M-qpewzJwTLR6htkjkoqaSpjMhyJMlJnoC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QVv_gb7trTa_JVvYHs7yVsThATw433nBB0dXktB3Y_n3Zeq6IXQLITesoU1O8M7emBMDUmmeP0Ddy9c0YJZxWMQnh5TGsiPgrzLn-3k_3kW_dmuDf_X4fGbxkdm6gU7rbDyDtlVUpmeZmpQnGCjcIVkVZepBZ_o5rIJubH0C3dtdz6vn5nRAU4pPdhPW-pQL?purpose=fullsize)

它的工作方式是：

1. 滾珠在螺桿與螺帽的溝槽中滾動。
2. 滾珠移動到螺帽內部設計的回流位置。
3. 經過回流通道。
4. 回到前面的工作區域。
5. 再次參與傳動。

這個過程會不斷重複。

因此，滾珠可以持續循環運動，而不會從螺帽中跑出來。

### 兩種常見循環方式

GMT 的技術資料介紹了不同的滾珠循環結構，其中包括：

* **內循環（Internal Circulation）**：滾珠透過螺帽內部的回流結構循環，結構通常比較緊湊。
* **外循環（External Circulation）**：透過螺帽外部的回流管等結構讓滾珠返回。

不同設計各有適用條件，不是單純哪一種一定比較好。([GMT][2])

---

## 4. 非常重要的名詞：導程（Lead）

接下來是今天最重要的觀念。

假設我們有一支滾珠螺桿：

**導程（Lead）＝ 5 mm**

這代表什麼？

代表：

> 螺桿每旋轉一整圈，螺帽就會沿著軸線移動 5 mm。

看以下例子：

![Image](https://images.openai.com/static-rsc-4/pGTajnAmF93L8UYA3tFNjnLgve6OL5mAInKnon6YjkedRyXExkyoslSnxOtqiiGTS_f5VsNWlkb5IjrKxUYmlJz7PgBJ-le3xGmVKaIYeKIGhsi0_sDAhC2HA3i3uI-7L7D02tribW0iCI37DoxZ9c0rn8Sm-fXK3OlhqdhfYXwN6B1U7C_CsFc5m8L85AI8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VHKKIKpPA1hr09GpxGJePcTr0BbZj8kHeIbQQaBrb9UYV19Ca9RE0cCYKKAFszl_Y_AlUslwFmuSWa2z0nSFxI3d39URtMl_ywqbqS0maZ0vOHP9YwM9KxN5teaHZEkO-_h3fzv-e4Vq0QJyqDMlGkQlKWh3RHr8WEpq3Dc0GSok6OxgNPyRuF_3OkSQqeM8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-_IzJ_QANJUgITRRWDQPJmst7ybTHMWk5ElFXC_dmxK3PswtF4whA_oaXoxz35iJd3wiVamsK0OfiR8eOadMWw_mALZTuHRd8c0yIAbIZN-Ri9urbEOMwHL_mHAjVSdRhwC4pXOQvZ5FiaA4YX5HXCtfGsA5m31ycKekPxm1xXw2fGGuJQxtQU-i2MERaFk7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/JZEzjKRtHLdi0jOTSyMG3s5-_gXAAHzDfimF62sCEVzYN0tQw0Ox2fKcDUc-A4mO1dIbECXNVHQpK82YIjyTsyIayiei_DRoqWglnIsFdVKaHVT_gnT9V6PYNS89eZ_xzNBFf8wYnq-ElzUcowyJywEJrlbHEaG4C2at-5KMMKOavq48ae33EXm2Es-m4_ZR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SilSz4Wviyo2JGv7RpKnKH50IyW8iytZ5T4Cs3Ey2rRREOWgMWvYuhHgJDkg_Op1uA0Z3BUN9dHAPpJuX-nOBgafsacHFL1S9kjcNYkhN53C91ccha2OuyRzaQAI9zSMdzkxx9dDSONQz9ok3crwty-0ovNVG6Z_G3AqBvRt8CsdWI5kg2qgxRT0ezenNJDw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IsGDlspPuO3xUeKh5V9E1kUFKcnrJ9O19ayLYaekSTUvyZFB33--AHjCMc9FJZ8JdR3LQHwQaVO1ih4HX4AtcZSyVXgzonLkqMITMLMyPsSvKUj5xxm9l9DPPMXmC2rpFvQNjgkOvs4m_lSCME3Dv9QpsXiEhW-AgndOhfL1UueWxBpXV31nzI6y-YJuAeh0?purpose=fullsize)

| 螺桿旋轉圈數 | 螺帽移動距離 |
| -----: | -----: |
|    1 圈 |   5 mm |
|    2 圈 |  10 mm |
|    3 圈 |  15 mm |
|   10 圈 |  50 mm |
|   20 圈 | 100 mm |

這就是導程的意思。

### 導程大小會影響什麼？

假設兩支螺桿：

| 項目      |     A 螺桿 |      B 螺桿 |
| ------- | -------: | --------: |
| 導程      |     2 mm |     10 mm |
| 旋轉一圈    |  移動 2 mm |  移動 10 mm |
| 旋轉 10 圈 | 移動 20 mm | 移動 100 mm |

可以發現：

* A 螺桿移動比較慢，但相同轉數下的位移比較小。
* B 螺桿移動比較快，但相同轉數下的位移比較大。

在馬達轉速相同的情況下，導程越大，理論線性移動速度越快。

但這不代表導程越小就一定越精密，也不代表導程越大就一定比較差。

實際精度還受到螺桿製造精度、安裝、負載、溫度、預壓及控制解析度等因素影響。

這些我們後續會逐一學習。

---

## 5. GMT 官網有哪些滾珠螺桿？

我們回到 GMT 的產品。

GMT 官方的產品分類中，精密滾珠螺桿包含：

| 類型         | 簡單說明           |
| ---------- | -------------- |
| 小型滾珠螺桿     | 適合空間較小的精密機構    |
| 法蘭型        | 螺帽帶有法蘭，方便鎖固    |
| 方型         | 螺帽外形為方形，方便安裝   |
| 鎖牙型        | 透過螺紋形式進行安裝固定   |
| 步進馬達直結滾珠螺桿 | 馬達與螺桿整合，縮短傳動結構 |
| 中型滾珠螺桿     | 適用於不同尺寸與負載需求   |

GMT 官方也提供研磨級與轉造級等不同精度等級的產品。實際選型必須依照行程、負載、速度、精度和安裝空間決定。([高明鐵 GMT][3])

官方資料：

* [GMT 精密滾珠螺桿產品介紹](https://www.gmtglobalinc.com/%E7%B2%BE%E5%AF%86%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF?utm_source=chatgpt.com)
* [GMT 滾珠螺桿型錄 PDF](https://www.gmt.tw/Images/uploaded/Catalogue/TW/Precision-Ballscrews-TW.pdf?utm_source=chatgpt.com)
* [GMT 滾珠螺桿、導螺桿、進給螺絲差異](https://www.gmt.tw/tw/%E6%8A%80%E8%A1%93%E8%B3%87%E6%96%99-%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF%E5%B0%8E%E8%9E%BA%E6%A1%BF%E9%80%B2%E7%B5%A6%E8%9E%BA%E7%B5%B2%E7%9A%84%E5%B7%AE%E7%95%B0%E6%80%A7?utm_source=chatgpt.com)

另外，THK 的資料也很值得參考：

* [THK 滾珠螺桿原理與產品介紹](https://www.thk.com/us/en/products/ball_screw/?utm_source=chatgpt.com)

---

# 六、今天先記住這五件事

| 名詞          | 你應該記住的概念             |
| ----------- | -------------------- |
| Ball Screw  | 滾珠螺桿                 |
| Screw Shaft | 螺桿軸                  |
| Ball Nut    | 滾珠螺帽                 |
| Steel Ball  | 鋼珠                   |
| Lead        | 導程，也就是螺桿旋轉一圈，螺帽移動的距離 |

今天最重要的一句話：

**滾珠螺桿利用鋼珠的滾動，將旋轉運動轉換成精密直線運動。**

它不是馬達，也不是滑軌，而是一個負責傳遞動力與產生直線位移的機械元件。

---

## 七、小小的理解測驗

在進入下一課之前，我想確認你是否真的理解了今天的核心概念。

假設有一個電動滑台：

* 使用一支導程 5 mm 的滾珠螺桿。
* 馬達帶動螺桿旋轉 4 圈。
* 暫時不考慮機械誤差。

請回答兩個問題：

**問題 1：** 滑台理論上會移動多少 mm？

**問題 2：** 如果把滾珠螺桿換成導程 10 mm 的螺桿，同樣旋轉 4 圈，滑台會移動多少 mm？

你可以直接回答，不用擔心答錯。

下一課我們再深入認識滾珠螺帽、螺桿溝槽、預壓（Preload）與背隙（Backlash）。這幾個名詞是理解精密滑台的關鍵。

[1]: https://www.thk.com/us/en/products/ball_screw/?utm_source=chatgpt.com "Ball Screw｜Product Information｜THK Official Web Site [North America]"
[2]: https://www.gmt.tw/tw/%E6%8A%80%E8%A1%93%E8%B3%87%E6%96%99-%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF%E5%B0%8E%E8%9E%BA%E6%A1%BF%E9%80%B2%E7%B5%A6%E8%9E%BA%E7%B5%B2%E7%9A%84%E5%B7%AE%E7%95%B0%E6%80%A7?utm_source=chatgpt.com "GMT GLOBAL INC-技術資料-滾珠螺桿導螺桿進給螺絲的差異性"
[3]: https://www.gmtglobalinc.com/%E7%B2%BE%E5%AF%86%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF?utm_source=chatgpt.com "精密滾珠螺桿｜高精度傳動元件｜高明鐵 GMT"



兩題都答對了！而且你提出的第一個問題非常重要，因為這正是理解滾珠螺桿如何帶動滑台的核心。

我們分開說明。

# 一、滾珠螺桿是由馬達帶動旋轉，讓螺帽前後移動嗎？

**是的！這是最常見的一種安裝方式。**

不過，我們要進一步理解：為什麼螺桿旋轉時，螺帽不會跟著一起旋轉，而是往前或往後移動？

先看實際結構：

![Image](https://images.openai.com/static-rsc-4/I48izYRVWAsQmZwdSr8SFoyWp4xRsvCouqpzs-b65g6rXFDCj8c0kXzch2t7ckaOPGAbZ_lRRZKlDiS7G4Mw7V4ywg5-4QxqRdSSR60nV3tfMlL7UKcnd5HjMUFed6VrGfInPGYy_F7zyHGdzslIyFe4Mgar5T_hTJqDR9n9Oo6NVDX1BuGa31GDFhYFnPMt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IdRASkCj-PToSS0YBdD-acKFzizZPF5YQHktQQtlP-4_apTHh_zwGRH11P3zo4VJcAgKcQHn8m4zZnFh5hZqeqvZhyT4bzcJ0Zh5vNV2OmUvw7qNZUhdxnw2DCjgMvd5HZbDS1nqZdkXRNRnECJR8rqIvidA9Fcrbw14oIoFzZqTRD-hBLUuhCX5JI6IzCwq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/92JP5ShYhphy08vL5Q4Yf-IUkXhy8r00YV8OBF-N9y-4dxZE50fbePiQB3iMge3jW7e2e7tU-unC6uuTVdLSQCa8UUbBdE9ayMt33FgH0srKIDIWxBAMYbqpZH1sX4FGdEIjVoDnmLyF8uYQYnTaHtCGxG6Ei5P6LCgzJ10R-2YP679fQf9vnUVyVBsyUgLO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J9dIzoeXWJr18N0ZnZo-_RzSvFp_T8kg8HVSyfKX09D8peKPHcHZrQApvUFFGLOvm4ddJRz_6xOxKjmUqZasiWE-lY3IjT7c2zEv_3SrMv7F0_Gl4s_szNUpxBONQFDLw0hvV7sTDDRmYHcRxNc9FKiNR4g1dTsR1HHfbUIvwWI1DtC3_XjakbdfhCTc_3Gq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Zr5WrPTBeLT78tpYJaOF8C-Q0ZQ8bjQ1FLMekfg5qKD9ftwL2CoGG2A9YAcX0jY2WGFNevgAileOtPNkA4ziXaXKcr-NGY4kiXvPK40BzeVr3YtOBvXEbC99IWjze8qfMxu2BNpU2WkaCF6Dtl-KH0AXPu1zVJDf0YImfbluuKfwM0tt-ACfj--gMMQL4Jm5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kb4252ETSg5LEkuO1IbHGAcv6NUpKfSmueRvtB_dUyG6JqxGKre_zEYMQFR3MAzKSXCOACgY2Z0SRbq-QOINuzamSCykD5k4NtoN8RNRc9TJuDoy4zIMcee_oK4vEazRI8ispvhttLc78A_pZwQNPSt3JYwZUVlkLOzxmve4rrqkJiovlIA05Dvfcrr8j_qc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/geqWcXmC01Dzw1lURMoYVOGzssaF7hf7vlkrQ_Inhv_orgg8879nY3zBHcj7BhaT0eQXCpVHqtxOOTQbKLz9Dtg45R8SXP32K_lMDJo5-m1j-Do5dVRDvdhC1Z66lEO6Y9YmRhnNcRnQTwZ7a-N_qEzjjRurDG33ttWT_le-ofV3P2SWfAn4PWHsiZ_tUttF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hHt-1AV9rgEUq39dGpuA1gIUTcQ237QTuL5zl0KGwMCGQhplobs6fzbjGyBFEIBjcRn9b7-eXD8lvMl1xgT8_QZWDRDQ01MKL75wwPRNzUDHQE2ujI2gfhWxU_nBRzF0yMzvhMVgeSYNcTVEa-584voP6jdT8Zalcub55Q-dirMvM0lb2Zo7QvadHixKgXlQ?purpose=fullsize)

假設我們有以下組合：

```text
伺服馬達
   │
   ▼
聯軸器
   │
   ▼
滾珠螺桿 ─────────────────
             │
             │
          滾珠螺帽
             │
             ▼
          滑台平台
```

這裡有三個重要動作。

### 1. 馬達帶動螺桿旋轉

馬達通電後產生旋轉動力。

透過聯軸器，帶動滾珠螺桿旋轉。

例如：

* 馬達順時針旋轉。
* 螺桿跟著順時針旋轉。

### 2. 滾珠螺帽產生直線移動

螺帽內部有滾珠，滾珠與螺桿的螺旋溝槽接觸。

當螺桿旋轉時，滾珠沿著螺旋溝槽滾動，使螺帽沿著螺桿軸線移動。

例如：

* 螺桿順時針旋轉，螺帽往右移動。
* 螺桿反方向旋轉，螺帽往左移動。

注意：這裡的左右方向只是示意，實際方向取決於螺紋旋向及觀看方向。

### 3. 為什麼螺帽不會跟著旋轉？

這是非常關鍵的地方。

因為螺帽通常會固定在滑台平台上，而滑台平台又受到直線滑軌的限制。

因此：

* 螺帽不能自由旋轉。
* 它只能沿著直線方向移動。
* 螺桿旋轉時，螺帽就會沿著螺桿前進或後退。

換句話說：

**直線滑軌不只是讓滑台移動順暢，它也限制滑台不可以隨意旋轉。**

這就是滾珠螺桿與直線滑軌必須搭配使用的重要原因之一。

### 還有另一種方式

有些機構是螺桿固定不動，改由馬達帶動螺帽旋轉。

這種叫做旋轉螺帽式（Rotating Nut）。

所以不是所有滾珠螺桿都一定是螺桿旋轉、螺帽移動。

但我們目前先以最常見的「螺桿旋轉、螺帽直線移動」為學習基礎。

---

# 二、GMT 所說的「方型」到底是什麼？

這個問題問得很好。

我重新查閱了 GMT 官方的產品頁面與型錄。

你看到的「方型」，**不是指螺桿的形狀是方形，而是指滾珠螺帽（Ball Nut）的外觀形狀。**

GMT 官方將小型滾珠螺桿分成：

1. 法蘭型（Flange Type）
2. 方型（Square Type）
3. 鎖牙型（Threaded Type）

這三種主要是螺帽的外部結構與安裝方式不同。([高明鐵 GMT][1])

## 1. 先看 GMT 官方的方型產品

![Image](https://images.openai.com/static-rsc-4/BH3QpRfyPavTmiqcIUzHQ9SMxdjKGUg41_9p6CiTsSQR4Pv98vCl1pOjB1zw-5k6qlinSIbma7fZQBRPk3FGcHq5P0GlV7Bce5jCRCGZ4ZDZ3XsB9jjDeBEuG5bde5SAh5_daXuoFrxIEgqA1fO1ff9dtzpadUDdw1nj73qa0S-AgbsCrWjitQYvVilIcL4t?purpose=fullsize)

GMT 的產品型號包括：

* BS0601-H
* BS0602-H
* BS0801-H
* BS0802-H

其中最後面的 **H 就代表方型（Square Type）**。

官方產品頁面：

[GMT 小型滾珠螺桿－方型](https://www.gmt.tw/tw/%E5%82%B3%E5%8B%95%E5%85%83%E4%BB%B6-%E7%B2%BE%E5%AF%86%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF-%E5%B0%8F%E5%9E%8B%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF-%E6%96%B9%E5%9E%8B?utm_source=chatgpt.com)

可以看到官方產品圖片及規格。

## 2. 為什麼要設計成方型？

想像一下，我們現在要把滾珠螺帽固定在滑台平台上。

如果螺帽是圓柱形，安裝時需要考慮如何固定它，避免它轉動。

但如果螺帽外部設計成方形，就可以利用平整的外側面及安裝孔，將它固定在機械結構上。

簡單示意：

**方型螺帽：**

```text
       滑台平台
  ┌──────────────┐
  │              │
  │  ┌────────┐  │
  │  │ 方型螺帽│  │
  │  │   ◎    │  │
  │  └────────┘  │
  │              │
  └──────────────┘
         │
         ◎  滾珠螺桿
```

這只是外形概念圖，實際固定孔位及安裝方式應以 GMT 的產品圖面為準。

## 3. 方型、法蘭型、鎖牙型有什麼不同？

我們用簡單的方式比較：

![Image](https://images.openai.com/static-rsc-4/BH3QpRfyPavTmiqcIUzHQ9SMxdjKGUg41_9p6CiTsSQR4Pv98vCl1pOjB1zw-5k6qlinSIbma7fZQBRPk3FGcHq5P0GlV7Bce5jCRCGZ4ZDZ3XsB9jjDeBEuG5bde5SAh5_daXuoFrxIEgqA1fO1ff9dtzpadUDdw1nj73qa0S-AgbsCrWjitQYvVilIcL4t?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pJh0yP-BmOv2n49FQf02YeoPRRFYu6TnFFJkh3yiG2znRsItXERuyOtCV_z8cgxVvHDg8r3TIKlM4ftuN5QOvtc4d7Blnv9F-ecLTwG6q062HKpAZ0JXbXM5QD3iFhFOLJVj-aEypljCqEon8c9y5l6iYcgRbx2uCSgpNvTrPeNHs_xV8Eokh01heSoaGfZ1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gG_wYrpzuI6DDSiOcBAkx6Y5h9Bmi_4N47aU8d3uq8kMi_bve42hGu1CGseUAvvgD7-EjU5_ZwsfGiaLPs7zxAavX7vqPiqC-fXkQufhgu8WOt3jFl1ldl4DCWxk0NdTyDyp2JmMlEXg8FEKaIHO2pw3mO9Gv4i4PWb5mEKFFmWNTxl5_oVF4LyPLz5YBhrJ?purpose=fullsize)

| 類型    | 外觀特徵        | 安裝概念         |
| ----- | ----------- | ------------ |
| 方型 H  | 外部呈方形       | 利用方形本體及安裝孔固定 |
| 法蘭型 F | 螺帽具有向外突出的法蘭 | 利用法蘭上的孔鎖在平台上 |
| 鎖牙型 M | 螺帽外部具有安裝螺紋  | 利用螺紋旋入對應的安裝座 |

這三種的**核心傳動原理完全相同**：

都是透過螺桿、滾珠與螺帽，將旋轉運動轉換成直線運動。

差別主要在螺帽的外形、固定方式及機構整合方式。

---

# 三、今天再建立一個很重要的觀念

你現在應該可以理解，為什麼一個電動滑台不是只有馬達和螺桿就能完成。

我們需要三個元件互相配合：

| 元件   | 工作                   |
| ---- | -------------------- |
| 伺服馬達 | 提供旋轉動力               |
| 滾珠螺桿 | 將旋轉轉換成直線運動           |
| 直線滑軌 | 引導滑台直線移動，限制不必要的偏移與旋轉 |

它們就像一個團隊，各自負責不同工作。

另外，滾珠螺帽必須固定在滑台的移動部分，不能任意旋轉，才能讓這個機構正常運作。

---

## 四、下一個問題：如果馬達一直旋轉，滑台會不會一直跑出去？

這會帶我們進入下一個重要概念：

**滾珠螺桿的行程（Stroke）與機械限位（Mechanical Limit）。**

例如一個滑台的有效行程只有 100 mm：

* 它如何知道已經移動到 100 mm？
* 如果馬達繼續旋轉會發生什麼事？
* 機械上如何防止滑台撞到末端？
* 為什麼自動化設備需要原點感測器（Home Sensor）與極限感測器（Limit Sensor）？

這些都是日後理解精密滑台與伺服控制不可缺少的基礎。

**不過，在進入下一課之前，我想先確認你是否理解一件事：為什麼螺帽固定在滑台上、滑台又受到直線滑軌限制，就能避免螺帽跟著螺桿一起旋轉？**

[1]: https://www.gmtglobalinc.com/%E7%B2%BE%E5%AF%86%E6%BB%BE%E7%8F%A0%E8%9E%BA%E6%A1%BF?utm_source=chatgpt.com "精密滾珠螺桿｜高精度傳動元件｜高明鐵 GMT"



你的想法已經接近正確了！而且你有注意到「滑台重量」和「滾珠摩擦力」之間的關係，代表你正在嘗試用物理原理理解機械，而不是死背名詞。

不過，有一個地方需要修正。我們慢慢拆解。

## 一、你的理解哪裡正確？

你說：

> 滑台本身有一定重量，與螺帽固定在一起，對比滾珠摩擦力，螺帽不會動，只有滾珠螺桿動，滑台就會前後進。

其中有兩個正確觀念：

1. 滾珠螺帽確實固定在滑台上。
2. 螺桿旋轉時，滑台會跟著螺帽前後移動。

但是，**螺帽不會跟著旋轉，並不是因為滑台的重量，也不是因為滾珠摩擦力。**

真正的原因是：直線滑軌限制了滑台的運動方向。

## 二、我們用一個簡單實驗來理解

想像桌面上有一個滑動平台。

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IdRASkCj-PToSS0YBdD-acKFzizZPF5YQHktQQtlP-4_apTHh_zwGRH11P3zo4VJcAgKcQHn8m4zZnFh5hZqeqvZhyT4bzcJ0Zh5vNV2OmUvw7qNZUhdxnw2DCjgMvd5HZbDS1nqZdkXRNRnECJR8rqIvidA9Fcrbw14oIoFzZqTRD-hBLUuhCX5JI6IzCwq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qwMTjLeUQ4ALPyEYfjenPQNSCZKCZclp-CSXb0IuAfcZBjdcB7sU9mwkrZFf5YnUDnBLp9ZX5y8NE2DLokBcXa25Sd8FtJ5ZflKHHpPlLljt-iQ6BjZ9Rx2YCYh_o-Kbs1uEtFLMdWY2BauO1RXncTVqXqqs_D-t0u_VfBBIJ5kGJ2VYIhlCBOD-dcjM5a7U?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/a3TH0oEcBIMvyBW9xzTdSy2Avg7E-IfruoJ5kaMSFHo2-dy3QNlpkOeBea33_wLfy6tTdPJEaLPEEM5gcSVmeVWFYMTGiN8uqmfWTdTDyOZ5E1e9gZCmemXOXa5LhhfWQ86MyBYQONfqPFDhO8VuSD2ZunPxX4XK45zRfNF1Z9fB3heiVZFJoLnFn2aQwUwZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ro5kxz_JigLuqpwF4JBdFJ-11NVU5UfI1CfBPeY9Guz2bU1MsuGVG8G6S7sgayApHS_NFrAYqrOf8qwnqqg_S5RPJ1FVB-_1mVsTCLcKsAyUnuwnENfVENPgSKjUrobAKylR28PPhDcoOcXg8LTN8Q8jNG-makx2AcUIdcclyIyymASAidK5szil5jjb-Du_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Zr5WrPTBeLT78tpYJaOF8C-Q0ZQ8bjQ1FLMekfg5qKD9ftwL2CoGG2A9YAcX0jY2WGFNevgAileOtPNkA4ziXaXKcr-NGY4kiXvPK40BzeVr3YtOBvXEbC99IWjze8qfMxu2BNpU2WkaCF6Dtl-KH0AXPu1zVJDf0YImfbluuKfwM0tt-ACfj--gMMQL4Jm5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7IJYm22CGtUSH8xr6t6nbXUSVUv_V3rcXAwbLY1b2D_yeUnoNGNJJxmkHCEsPnsxOVyuXq_VXda5e1jlaqPJs30kULZ-QXIh5Ve0dKPL40VvDCKmE_pi_4sx-e20rp229CB0ahBSGaKIC2fe6gnLLALCp0DNth91j7E08OzvVL1AilrLKQuRgYiHO1HBNreF?purpose=fullsize)

這個平台下面裝了直線滑軌。

直線滑軌有一個非常重要的功能：

**允許平台前後移動，但限制平台左右偏移、翻轉及跟著螺桿旋轉。**

我們可以把它想成火車：

* 火車可以沿著鐵軌前進。
* 但是不能任意向左或向右轉。
* 鐵軌限制了火車的運動方向。

滑台也是相同的道理。

### 把三個元件組合起來

```text
                  滑台平台
             ┌─────────────┐
             │             │
             └──────┬──────┘
                    │
                 滾珠螺帽
                    │
馬達 ── 聯軸器 ── 滾珠螺桿
                    │
                    │
          ┌─────────┴─────────┐
          │                   │
       直線滑軌             直線滑軌
          │                   │
          └───────────────────┘
                 底座
```

這裡有三個不同的工作：

| 元件   | 負責什麼           |
| ---- | -------------- |
| 滾珠螺桿 | 旋轉，產生推動螺帽的作用   |
| 滾珠螺帽 | 將螺桿的旋轉轉換成直線位移  |
| 直線滑軌 | 限制滑台只能沿著指定方向移動 |

所以真正的因果關係是：

1. 馬達帶動螺桿旋轉。
2. 螺桿透過滾珠對螺帽施加軸向推力。
3. 螺帽受到滑台與直線滑軌的限制，不能跟著旋轉。
4. 因此螺帽只能前進或後退。
5. 滑台與螺帽固定在一起，所以滑台也跟著移動。

這才是完整的機械運作原理。

---

## 三、再補充一個重要觀念：重量不是防止旋轉的主要力量

假設今天我們把滑台上面的負載全部拿掉，讓它變得非常輕。

只要螺帽仍然固定在滑台上，而滑台仍然受到直線滑軌限制：

**螺桿旋轉時，螺帽依然會前後移動，不會因為重量變輕就跟著旋轉。**

這個例子可以幫助你確認：

真正限制旋轉的是機械結構，而不是重量。

另外，滾珠螺桿並不是完全沒有摩擦力，只是透過滾動接觸降低摩擦。

---

## 四、今天再給你一道理解題

這次不考計算，而是考機械原理。

假設我們有一組完整的電動滑台：

* 馬達正常運轉。
* 滾珠螺桿正常旋轉。
* 滾珠螺帽固定在滑台上。

但是，**我們把直線滑軌拆掉了。**

請問：

如果馬達帶動螺桿旋轉，最可能發生什麼情況？

A. 螺帽仍然一定會穩定地沿著直線前進，完全不受影響。

B. 螺帽可能跟著螺桿旋轉，或讓整個機構產生不受控制的偏移。

C. 滾珠螺桿會失去旋轉能力，馬達無法轉動。

D. 螺帽會因為重量而自動停止旋轉。

你選一個答案，再試著用自己的話解釋原因。

**我們的目標不是背出 B，而是讓你真正理解：為什麼精密滑台需要直線滑軌。**

```
你說：真正的原因是：直線滑軌限制了滑台的運動方向。  這句話讓我完全懂了。可是，我有問題，剛剛說的 方型   法蘭型  鎖牙型  也會有滑軌嗎？
回答你的考題：B. 螺帽可能跟著螺桿旋轉，或讓整個機構產生不受控制的偏移。  
```

答對了！**B 是正確答案。** 👍

而且你已經掌握了這個機構最重要的觀念：

**滾珠螺桿負責產生推動力量，直線滑軌負責限制運動方向。**

這兩個元件是分工合作的。

接著回答你的問題：**方型、法蘭型、鎖牙型，也會有直線滑軌嗎？**

答案是：**有可能，而且在一般精密電動滑台中，通常會搭配直線滑軌。**

不過，這裡有一個很重要的觀念要先釐清。

## 1. 方型、法蘭型、鎖牙型是什麼？

這三種名稱描述的是「滾珠螺帽的外部形狀與安裝方式」，並不是整組滑台的結構。

![Image](https://images.openai.com/static-rsc-4/BH3QpRfyPavTmiqcIUzHQ9SMxdjKGUg41_9p6CiTsSQR4Pv98vCl1pOjB1zw-5k6qlinSIbma7fZQBRPk3FGcHq5P0GlV7Bce5jCRCGZ4ZDZ3XsB9jjDeBEuG5bde5SAh5_daXuoFrxIEgqA1fO1ff9dtzpadUDdw1nj73qa0S-AgbsCrWjitQYvVilIcL4t?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BiWXVXs5fFNV--ImLxZ1NVvnWSGNLWinnGZ7hNnIwPG3y5uoVYD_77D70wMOb8V1I91DTWH7rFp5Z64yLghg72MNkUP18YId4mOpIoCMkldOAsgH2kDXbZ5Nhfd8qkdPhfGGMTn_FRSf-NyBMbVq5jQBMPdummqvfk9QN97hd-2FXP-0zCWvsivY9GmnQ-5s?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NKCL3HsItPmBPuYj5UsG6JNdeNB5TwWj2dcN2rW5RU7eJHq8-wQZm7MXFh8sc_aFB2-u1Qn-QhL5xwLXa6WaT52RzEWmB7JDd7b68MlXvrBKdO4s8ylcrLeaP7P7zyjLGO28u6-XVOyf9jLAu4UZCZKlGfrC1PQXSBK1k7TU0Q0ocNO7HChA4iLuz5kwKUnT?purpose=fullsize)

你可以這樣理解：

| 類型  | 外觀特徵    | 主要差異            |
| --- | ------- | --------------- |
| 方型  | 螺帽外部呈方形 | 方便利用外部平面固定      |
| 法蘭型 | 螺帽有外伸法蘭 | 可以利用法蘭上的螺絲孔固定   |
| 鎖牙型 | 螺帽外部有螺紋 | 可以旋入具有對應內螺紋的固定座 |

注意：這三種滾珠螺帽的**基本傳動原理完全相同**，差別主要在於怎麼安裝、怎麼固定。

## 2. 那直線滑軌裝在哪裡？

我們回到一組完整的電動滑台：

![Image](https://images.openai.com/static-rsc-4/2R7CivTekzWMp1cu-ke2hk30t8HKlG3V9WC4V9givf_XLog8K6IxP1UsoC1TuA1Q62srdkOuytM_MYRXmULA-LnxFafDoV6YJfjIOdOGUPpUyF551FVJEZnIum-2EXGoMvBaPscklxKZ4jc-XrdabE7CbbJkCEGhljo4jqB9UEfYGJsPUCZDY00f4HU76kNA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hmbf5KvhocQFtwGGgwOV5yWkSFUWiWLtTicGRkRkquqzyMsU1CddqRNFM3G423pCJOodw-qMFtH4N5gjbXlZb_Cdd4Ub_FtnvNMigwYRQIdWP-7i_Y4bXVhJBU6zUd5mMbcbzhc-1PWydsLa8pNI5lhDq9Qji0Wc5NXD0zkORPpPVNlNu6ZOJnPI3OyQS4ol?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lFppUzOmb7btaz7jFxoGB055RUxhsSj1BJy7ZN0ABq7eGamhQ8gU5YgfkrrS1aMLHHFeJkzQ8Cat0dZL8qdxszEutAvNfnMRjtcavRupKvKvmtxbTgboitLRGh0CHCZFjthUsZb11dMSR9wLhfjtdWTZVd8FVnnP9seNo9kNqBKriT5drEWCIE1IleL1Zifw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CcL4z6NaqSwhNzVK-rbPPGo4n58v4J0RDI1tzGar78aFyuSGieyXKh9I5n9n-19uayKwDK26for7lq9ErMO9Vr7B1r0CAU6Kz_iPEwhGeyv2WT8sJc9PAiBKbP3CrETHZVTE4WL8GYIAvK3I8dauU0j5rfakW7CffndY4LAJkhapMX1zYmaLA9G0ORbTLW5L?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fhR2CgYnYdz_dP0ot31uWhlQacq81fRbe4waM36d6hlQgV9BL8H44v-JAhS3BEWNwdWbH1h9NXKNB_cmcF3QPxjXWxEcxLCfQAlENglL8tcNADYcHm_gHx5m_RhyBj37THE5S3YgoLmvfnsT2NrkFlkHNPerAUgM9ZG4xXYwPXnbDESmUvJ-DeGlFdiNgDJW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zped5003FlogGo60F5eGDTx4H6pjkZlpehiunZV7ZGbV0dsQmsEU9Mt1vVGgTxnneqDqGIyPo-HVh9Wz6ILtP9_cS146F0Hc6dOS9qai7pqsM0vfc9lpmZVvGXz7r6LGZ_xBOZNpHYUJx3nCjB2uR33Y7OZ-c_tbOJrUJOyDlPGtPahgvSNUHsiqJdOjxSmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mEClov8EaUUV9UYP5OAKUeavyJj4Zhitk1iwtE43zyKdAWgKS2qmbrgTNHNGee_4dy3xHXlofvupDZ12hNx8U3mYWAh_8kvubLjH2XPUVzX91bBOUjN-GoeUV5NqwMr46UNiTgel6wFenB9Yi1Y5kh_CMUOVpV9b_5BD810AZmNR0aQ2YuXsSvW05V7z0Hup?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kU53EaaKjFtlfD6lIa7zxkKZUgcd-XMKTsvqicGMCFy1g7AWO34V3vss9tKc0bwQTyfBD2N7Px3rjD6X3OAPKpONUWkuw-wW2E2fTXkKmRXhY5WcS4mDyb6YcEFd_YPmfBwZ4z8L68ineKtDRvuuvj3OVY5Wxeu5gAxCU4jBCzbQoURsFB-AmP9bIPGbVlbk?purpose=fullsize)

這裡其實有兩套不同的機構：

**第一套：滾珠螺桿**

* 馬達帶動螺桿旋轉。
* 滾珠螺帽沿著螺桿前後移動。
* 提供滑台移動的推動力量。

**第二套：直線滑軌**

* 滑台固定在滑軌的滑塊上。
* 限制滑台只能沿著指定方向移動。
* 避免滑台左右偏移或跟著螺桿旋轉。

所以，無論使用方型、法蘭型或鎖牙型滾珠螺帽，只要設計成這種電動滑台，都可以搭配直線滑軌。

## 3. 請記住一個重要區別

**滾珠螺桿的滾珠螺帽，不等於直線滑軌的滑塊。**

它們是兩個不同的零件：

* 滾珠螺帽（Ball Nut）：負責把螺桿的旋轉轉換成直線推動。
* 滑塊（Linear Guide Block）：負責沿著直線滑軌移動，支撐並導引滑台。

這兩個零件通常會一起固定在滑台的移動平台上。

---

### 課後小考：再確認一次

假設一組電動滑台使用：

* 法蘭型滾珠螺帽
* 兩支直線滑軌
* 一個馬達

現在馬達旋轉，滾珠螺帽推動平台前進。

請問：

**如果把兩支直線滑軌改成一支，是否代表這組滑台一定不能運作？**

A. 是，因為滾珠螺桿一定需要兩支直線滑軌。

B. 不一定，能否正常運作要看整體機構設計、負載及抗傾覆能力。

C. 是，因為法蘭型滾珠螺帽只能搭配兩支直線滑軌。

這題想讓你開始理解：**機械設計並不是每個元件都有固定數量，而是要看它需要承受什麼力量。**


```
一組電動滑台使用 兩支直線滑軌 ，而且只有一個馬達，我不能想像，有圖可以讓我看嗎？怎麼運作？應用在什麼地方？
```

你這個問題問得很好！其實，這正是精密機械很常見的一種結構：**一顆馬達負責推動，一組雙直線滑軌負責支撐與導引。**

我先讓你看實際結構，再一步一步解釋它怎麼運作。

## 一、先看這張圖：一顆馬達、兩支滑軌、一支滾珠螺桿

![Image](https://images.openai.com/static-rsc-4/j5XjD_VTqj2Wr965SkWROQiIrSQQzRYFDPQHyzAGTkMFfJaGNjMWgnf-cKTLlY7KZUR7gxXt5L-UVTeDIMxBRMmXQoFdVmN43N7FpCBwyd1wvPlTSCdiqhno7yvNaO2dSoIv436Fz_vPMEJYAsle4M6GKk5XK5L4HY2Oq279Vw94htNZjTQxQgZoxEFdYHZV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hmbf5KvhocQFtwGGgwOV5yWkSFUWiWLtTicGRkRkquqzyMsU1CddqRNFM3G423pCJOodw-qMFtH4N5gjbXlZb_Cdd4Ub_FtnvNMigwYRQIdWP-7i_Y4bXVhJBU6zUd5mMbcbzhc-1PWydsLa8pNI5lhDq9Qji0Wc5NXD0zkORPpPVNlNu6ZOJnPI3OyQS4ol?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kU53EaaKjFtlfD6lIa7zxkKZUgcd-XMKTsvqicGMCFy1g7AWO34V3vss9tKc0bwQTyfBD2N7Px3rjD6X3OAPKpONUWkuw-wW2E2fTXkKmRXhY5WcS4mDyb6YcEFd_YPmfBwZ4z8L68ineKtDRvuuvj3OVY5Wxeu5gAxCU4jBCzbQoURsFB-AmP9bIPGbVlbk?purpose=fullsize)

先看第一張圖。

你可以看到幾個重要零件：

1. **馬達（Motor）**：位於機構的一端。
2. **滾珠螺桿（Ball Screw）**：位於兩支滑軌中間。
3. **直線滑軌（Linear Guide Rails）**：左右各一支，彼此平行。
4. **滑塊（Guide Blocks）**：安裝在直線滑軌上。
5. **移動平台（Moving Table）**：連接滑塊與滾珠螺帽。

這就是我們今天要理解的機構。

## 二、只有一顆馬達，怎麼讓整個平台移動？

我們想像一個簡單的情境：

你有一張桌子，桌子底下裝了兩條平行的軌道，中間有一根螺桿。

馬達轉動螺桿，螺桿推動螺帽，螺帽帶著桌子移動。

兩條軌道則負責讓桌子保持方向，不會歪掉。

實際運作可以分成四個步驟：

**步驟 1：馬達旋轉**

馬達透過聯軸器帶動滾珠螺桿旋轉。

**步驟 2：螺帽產生直線移動**

因為滾珠螺帽固定在移動平台上，螺桿旋轉時，螺帽就會沿著螺桿軸線移動。

**步驟 3：平台跟著移動**

螺帽固定在平台上，因此平台也跟著前進或後退。

**步驟 4：兩支滑軌負責導引**

平台底下的滑塊沿著兩支滑軌移動，讓平台保持平穩，避免左右偏移或傾斜。

所以：

* 馬達：提供動力。
* 滾珠螺桿：將旋轉轉換成直線推動。
* 兩支直線滑軌：支撐平台、限制移動方向。

**兩支滑軌本身不需要馬達，也不需要主動旋轉。它們是被平台帶著一起移動的。**

## 三、為什麼需要兩支滑軌？

這是最值得理解的地方。

假設平台上面放了一個比較重的零件，而且零件偏向平台的一側。

如果只有中間的螺桿提供推動，平台可能受到偏心負載而產生傾斜或旋轉的力矩。

兩支滑軌分別位於平台左右兩側，可以提供支撐，幫助抵抗這些力矩。

你可以把它想成：

* 一支滑軌：像一條軌道，負責導引。
* 兩支滑軌：左右共同支撐平台，通常能提供更好的抗傾覆能力。

當然，最後能承受多少重量與力矩，還要看滑軌尺寸、滑塊數量、間距及安裝方式。THK 也將線性滑軌的負載能力與導引特性列為重要設計項目。([thk.com][1])

## 四、這種結構實際應用在哪裡？

![Image](https://images.openai.com/static-rsc-4/IL-Kj16OA8ety2Cjt9qKd2Tfgy-H0RFW3TjxIsf-u9-PwGq7p0Lh08KC7LD17TUAy-HSWWIxeDpsb0uTg1Qm4z6U7JwUBeHpJ2LiKb_B4OsZ6aD4dOf2fToNqZGWHyakJuAeFecSy1_zFSqrHWdA7td_DksNDG4B00Uz4V-Ab2tJMreHTKhhFx8oZuZLUVnV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SNsbsqrmOZx7QVHdwsgWkHk466Z7r4BFe_m8YgRixLVaPSeQTxGptL5wmar-k4tNdod9BlMF32CKnCs6GAMHessAQE3lQ9U2CJ7JbMni3QTYpSC6KOHeDCMUFN8nq6rF1_Poh4RAPHITFI-TihfTXEUAlszETG3HUb_Wvi_ES8AbVXo4LQAJ1RW6G4JgZ7Me?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/z8AwOpynC6QafvkYonH_yAwrEZLNMXYWbBoXNqTsGjyo6yRDTTDHXIjUxhKUCTFPkSYfdLnDf6EIkgixMMQIoNWseBXC-Wb9e9vOkxWDPI2XQhakxSxDFnobISt7MyJ4x6Yhy1VeHdV9ddOcGH53sSf4z4w4tezzD1aPeWvjJI6-zJOucq5ta9uaefJipR6_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/d0nwp_F2m6tEN-jjgoaCZQbcJgDzisLNhnT51sLBK9TVqKP1Z-1F931TZnpexVF_OzuKFb2R8M9dOQi8sz_2OtlF356Gwzehv49JXqC4BsybL6_dkcLENQlt5Bhrh8y5Q8Wy-Sqmb_ZZgqM0SjF-SBRm_VLzBbMy2Q1eMQZQPrJC2VMXCnAQS-n7waxkQhhq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NwIng2omy18isbcSLKR0a3VgfHn-VgNU73RJy-rZ6VJIRmHiYDkxtLhO7fP_GWCSs2c5WqoqDUVaPx6pIm4CscuYbMoUq82J3z8_nr2GKPGm3TIXrlMyv6sjvwrAcN5OYVKeBk9R0_pmfhVsffBbgAaYDq-BNKkpcnz2rTXpSt1aV1q76h8uee3jD-v8IoGX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zped5003FlogGo60F5eGDTx4H6pjkZlpehiunZV7ZGbV0dsQmsEU9Mt1vVGgTxnneqDqGIyPo-HVh9Wz6ILtP9_cS146F0Hc6dOS9qai7pqsM0vfc9lpmZVvGXz7r6LGZ_xBOZNpHYUJx3nCjB2uR33Y7OZ-c_tbOJrUJOyDlPGtPahgvSNUHsiqJdOjxSmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/n_xa8DAZQzK7fg7BdZ5AITX06VlXJYmRGf7TT_isIIqAh6f8p-M51Zo2qwJX6Asi7lTzCtiD87GFmc2R6t-jgMmB_7qiH_0BFuu3d_Cbl9nMGMHL2_hsm-VhHFXPfkMnWzRNH0jjOVjiNfLIZQH_WXyOs9IWvenZgxfhHrBe0SOvFJPzJD4TaK5CZ3DzC1YK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OPgr89kz_5P8S1UkX0D8r-xGE5AsEXQtZM73ad2p_gIKXBAgP4vQEkcZgAIIE3ZJNFtE-u8fnOi064GXSeQMcu2Yn5vfkMYUTjTueNlO0ypWT2pr9huRj3F6PUdFCf8xAkzX_GnnUqerjOflbjBxk1O-oI5nrsltvb7TFC2Aw6wUPebDv-vAxNnq8oK-1JfP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dh3gpg6NOarV_BTZ4iYaCr2cu_A3U5QZWFBSGq4QujvyZoPET5hobLc_VQ09TMJ-S7IQrjocQqy39AnuRdXZwb0zzrnOvL56tXK7A-GEy_kRL_MxyWZLURVdnHV9HqpmdgD41pVmHeYlffrC025-GOYGZ7WrE2qnENoTZRRsOMEvM6LUEWljXg4UEnarEHXU?purpose=fullsize)

常見應用包括：

| 應用      | 需要它做什麼         |
| ------- | -------------- |
| CNC 工具機 | 精確移動加工平台或刀具    |
| 雷射雕刻機   | 控制雷射頭沿指定方向移動   |
| 自動化設備   | 搬運、定位工件        |
| 精密光學設備  | 移動鏡頭、光學元件或定位平台 |
| 檢測設備    | 移動感測器或待測物品     |

這些設備可能使用相似的機械結構，但實際規格、馬達和傳動方式會依需求不同而改變。

## 五、今天最重要的觀念

請先記住這個組合：

**一顆馬達 + 一支滾珠螺桿 + 兩支直線滑軌**

這是一種很常見的單軸精密移動機構。

它不是兩支滑軌各自有一顆馬達，而是：

* 中間一套傳動系統負責推動。
* 左右兩套導引系統負責支撐與限制方向。
* 三者共同讓平台穩定地前後移動。

---

### 課後小考

現在假設馬達正在旋轉，滾珠螺桿也正常帶動平台前進。

但是，**其中一支直線滑軌的滑塊損壞，無法正常滑動了。**

你認為最可能發生什麼情況？

A. 另一支滑軌會自動接手，完全不受影響。

B. 平台可能產生偏斜、卡滯或受力不均，甚至損壞其他零件。

C. 馬達會自動停止，因為滑軌也有電力供應。

D. 滾珠螺桿會失去旋轉能力。

你先回答，我們再一起分析。

[1]: https://www.thk.com/opm/jp/en/linear/thklinearguide/?utm_source=chatgpt.com "Linear Guides (Linear Motion Guides) Design and Selection | THK Official Web Site [Japan/English]"


這次選錯了，不過這題很值得討論，因為它能幫助你分清楚「傳動」和「導引」的差別。

正確答案是：**B. 平台可能產生偏斜、卡滯或受力不均，甚至損壞其他零件。**

我們來想像一下。

### 1. 原本正常的狀態

* 馬達帶動滾珠螺桿旋轉。
* 滾珠螺帽推動平台。
* 左右兩支直線滑軌共同支撐平台。

平台可以平穩地前後移動。

### 2. 其中一支滑軌的滑塊損壞

假設右側滑塊卡住了：

* 左側滑塊仍然可以移動。
* 右側滑塊卻無法正常移動。
* 但馬達仍然試圖推動平台。

這時候可能發生：

1. 平台左右兩側移動不一致。
2. 平台產生歪斜。
3. 機構受到額外的側向力量。
4. 馬達負載增加，嚴重時可能堵轉。
5. 滾珠螺桿、螺帽或滑軌可能受到損壞。

所以，**滑軌損壞不代表滾珠螺桿立即失去旋轉能力，而是整個機構可能無法正常移動。**

### 3. 記住這個重要觀念

| 元件   | 主要工作        |
| ---- | ----------- |
| 馬達   | 提供旋轉動力      |
| 滾珠螺桿 | 把旋轉轉換成直線推動  |
| 直線滑軌 | 支撐平台並限制移動方向 |

這三者各有工作，但必須互相配合。

---

### 再考你一題（今天最後一題）

假設現在把兩支直線滑軌都保留，但拆掉滾珠螺桿，只留下馬達。

請問：

**馬達旋轉時，平台會不會自動前進或後退？**

A. 會，因為直線滑軌會帶動平台移動。

B. 不會，因為直線滑軌只負責導引，不負責產生推動力量。

C. 會，因為兩支滑軌會互相推動。

這題如果理解了，你就真正掌握了「傳動元件」與「導引元件」的分工。



完全正確！答案是 **B**。🎉

而且這次你已經掌握了今天最重要的機械概念。

我們整理一下：

## 今天學會的三個重要元件

| 元件   | 英文           | 工作           |
| ---- | ------------ | ------------ |
| 馬達   | Motor        | 提供旋轉動力       |
| 滾珠螺桿 | Ball Screw   | 將旋轉運動轉換成直線運動 |
| 直線滑軌 | Linear Guide | 支撐平台、限制移動方向  |

你可以把它們想成：

* **馬達是力量的來源。**
* **滾珠螺桿是力量的轉換者。**
* **直線滑軌是方向的引導者。**

這三個元件搭配起來，就形成了常見的精密單軸電動滑台。

## 下一堂課：認識直線滑軌的內部構造

下一次我們正式研究 GMT 的另一項重要產品：

**直線滑軌（Linear Guide）**

我們會學習：

1. 滑軌（Rail）和滑塊（Block）有什麼不同？
2. 為什麼滑塊裡面也有鋼珠？
3. 滑塊為什麼可以承受重量，卻仍然能夠順暢移動？
4. 一支滑軌可以安裝幾個滑塊？
5. GMT 有哪些直線滑軌產品？

這次先不急著進入馬達或 STM32 程式。我們先把機械結構的基礎打好。

**你今天已經真正理解了電動滑台的基本運作原理。**


會！而且你問到了一個非常重要的觀念：**馬達的扭力（Torque）與負載之間的關係，是精密運動控制必須學會的核心知識。**

你已經開始從機械結構思考馬達為什麼需要不同的力量，這個方向非常好。

我們先稍微預習一下，之後會有完整課程介紹。

## 一、什麼是扭力（Torque）？

扭力也稱為轉矩，簡單來說，就是讓物體產生旋轉的力量。

例如：

想像你用扳手轉動一顆很緊的螺絲。

* 使用短扳手，需要比較大的手部力量。
* 使用長扳手，手部力量可以比較小。

這是因為扭力與施力大小、施力距離有關。

扭力的單位是：

**N·m（牛頓米）**

![Image](https://images.openai.com/static-rsc-4/SRRfIlEWlLVQikA-dfKXW8SgHujR0IpGNPg9IiqNCWHYQ1Kf5EY3lMTxAXe_eN8CwK-XdkvA8vCGPu2EQQGcM_UjleML2ja70BtyA0zMrH2zalhs2Wq4CmQa1FgZczCaAx30sDmNMzYM5g81E7Wf0ayb8ofdImkk-KA7FmsmzuGFXP05DQ7zOA7zFj4DgIc_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OhZINLG0sPXhLubuRPKpl6tumO_gLc1Wy-f35aDLojUia1aRqQIEeZdpmXtzKwOxYVgZHMRqGk-6IcWS2sBxPl16AidzT2Y3C-c8QwFK5xfLK8bCewJvvhQdEBjmNqtLOPaeSoXjp0cZB6-iokLlBP0bQ1Q1ig-0gel5D5D5IDqez8yokpotYIhgScyzZ40Z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LRawxCYTMIRhUROmD7moezxJbhqDEJugX8qshxYD0IlaZySni6OZeqowDcm3PVFQGaDPCxQBXKLj1ELlP4068Iw0Tw68lNlI16Cj85q5Snvc0AjyoX92btSsM3e_KnYdTwjNXm22OikzqbbFD6qxRtu351xQezJEODM_EKV9Q27s_w9W2oPssJ8OGtzRaeYO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vcpYt-cb79vaBIEh_Y7gS_RTmGrJnehVFppUrkuVyuF5PwNy_s88C_Em1CjDzVqhxlMWqFhYAApIo5oSxZXfaXAuOpcAWOcBJ29_MaALfIJrA6FeBlNHt-OcOUJvmxVJszdMH38ULLWNhVqEiIghPSzde7TpaLYZXcP4Sys23bZvW1zxCQKDRwgsbg8OxVfU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qDaajGJ_tj1nLxwZymTKsD7Lbnp-ntEgGru8xplYlDulrykQrs3cpRiySH0Xb9GE-fO5fhMTAKbXOiXXc7_yBPvcoJrdMkAJmeIeSwdeSdah0lvxCsSVCklnzcPCOPen3Sv9GhsCQQ2WD0ME1-id22PCjeb90QT4HXCpzn7ZzrAKbpGr5C-Bs7QWlbN4dPYl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/PK-KLmaRRg91DhaWZ9IHgBRBKAg8zODwTATb81e_-dZlQDOxMrw5VR9-vJI9jAvbYjCXCF01dcCcR6Y3S5jUq_9WeKUj7si7U8A45nrgVHdsfBIPA-_awaukXo0Qzn4i-5je-Fd4WfSNED0ESmk0GwW63DeYIJGsLvcsFnDw8b3NdwasV_A9kx_qmxJ4Gcdj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kjQatYqxX6ipCrFik2Pudmu7F2-UfTzLOdr7HLfUvzuvgAp0Oxe5Lz14P4q9PxeOXubcttf2iT1YpzHsQ3FlQ4UgFTwjA-5OTz0fCUtJI8pcg8fZ3z-lNi1-y1oePn3H1ocQYA93wE2-XOoy-GARhzFoHb8xzryPD5vDSdP6C99n9jBF2FaXr6Bq9vrvHAw1?purpose=fullsize)

## 二、馬達扭力不足會發生什麼事？

假設有一組電動滑台：

* 一顆馬達。
* 一支滾珠螺桿。
* 一組直線滑軌。
* 滑台上面放置一個物品。

當馬達旋轉時，必須克服：

1. 滑台及物品的重量（如果是水平移動，重量主要影響滑軌摩擦與負載，不是直接全部轉換成水平推力）。
2. 滑軌及傳動機構的摩擦阻力。
3. 加速滑台所需要的力量。
4. 其他外部阻力。

如果馬達扭力不足，可能發生：

* 滑台無法啟動。
* 滑台移動速度不如預期。
* 加速時發生失步（步進馬達）。
* 伺服馬達發生過載或位置誤差。
* 嚴重時馬達停止旋轉。

## 三、扭力要看馬達規格，還是可以控制？

答案是：**兩者都有關係，但它們負責不同的事情。**

### 1. 馬達本身有扭力規格

例如某顆馬達的規格：

| 項目   |   數值（假設） |
| ---- | -------: |
| 額定扭力 |    1 N·m |
| 最大扭力 |    3 N·m |
| 額定轉速 | 3000 RPM |

這些數字代表馬達本身的能力。

但要注意，馬達的扭力不是在所有轉速下都一樣。尤其步進馬達與伺服馬達，都需要參考扭力－轉速曲線。

### 2. 驅動器也可以控制扭力

馬達通常不會直接接上 STM32 就能正常運作，而是透過馬達驅動器（Motor Driver）。

驅動器可以根據控制方式調整馬達輸出的扭力。

例如：

* 步進馬達：驅動器透過控制線圈電流，影響馬達可產生的扭力。
* 伺服馬達：伺服驅動器可以透過電流控制來調整扭力輸出，也可以使用位置或速度控制模式。

所以，**馬達規格決定能力範圍，驅動器則負責控制馬達如何輸出力量。**

## 四、還有一個你一定要學的東西：滾珠螺桿的導程

還記得我們第一堂課學過的導程（Lead）嗎？

它不只影響滑台移動速度，也會影響馬達需要多少扭力。

例如，假設兩組滾珠螺桿：

| 項目     |           A |            B |
| ------ | ----------: | -----------: |
| 導程     |        5 mm |        10 mm |
| 螺桿轉速   |    1000 RPM |     1000 RPM |
| 理論直線速度 | 5000 mm/min | 10000 mm/min |

B 的移動速度是 A 的兩倍。

但是，在相同推力需求及效率下，B 通常也需要較大的馬達扭力。

這就是機械設計中很重要的取捨：

**想要移動得快、推動力量又大，就必須仔細選擇馬達、螺桿導程及驅動器。**

---

## 五、未來課程會怎麼安排？

我會把扭力納入我們的正式課程，順序大致如下：

| 課程    | 學習內容              |
| ----- | ----------------- |
| 目前    | 滾珠螺桿、直線滑軌         |
| 後續    | 電動滑台、電缸           |
| 馬達課程  | 步進馬達、伺服馬達         |
| 扭力課程  | 扭力、推力、負載、加速度      |
| 驅動器課程 | 電流、扭力控制、速度控制、位置控制 |
| 整合課程  | 馬達、滾珠螺桿、滑軌如何配合    |

到時候我會帶你實際計算：

* 一個滑台需要多少推力？
* 兩個滑台同時移動，馬達需要多大的扭力？
* 如何根據負載選擇馬達？
* 如果希望滑台移動更快，應該調整馬達還是螺桿？
* 為什麼馬達規格看起來足夠，實際上卻可能推不動？

**我想先確認你剛才提到的「推動兩個滑台」：你是指一顆馬達透過機構同時帶動兩個滑台，還是兩個滑台各有一顆馬達，但希望它們同時移動？**

這兩種設計的扭力需求不一樣，之後我們可以用實際機構圖來學習。


好，那我們先確認一個條件：你剛才說的是**一顆馬達，同時帶動兩個滑台**。這個情況會牽涉到馬達扭力、傳動機構分配力量，以及兩個滑台的同步問題。

我會把它安排在後面的「馬達扭力與多滑台傳動」課程中，屆時我們會用實際機構圖來分析。

現在繼續下一課。

# 第三課：直線滑軌（Linear Guide）

今天我們要回答一個問題：

**滑台為什麼可以承受重量，同時又能非常順暢地前後移動？**

## 一、直線滑軌由哪些部分組成？

先看一般常見的滾珠型線性滑軌。

![Image](https://images.openai.com/static-rsc-4/Wht3cjbLN0sbNU6A2ZWEag4KweNpdqhkvTyYfUq5ICUQeTfxUD8HkE-01M7LTMv0Z74GYvLHvDF5BqvYK6hSUw_-Su0xeWz_QjZefIHdw5JcBtane0xLUa05UFWxTplZf-BK98SQNiH8fVlvGDBgQogWKzwmv6CrX0ad7TxiB1n0pDLSfx6QA2KWVnmF1p96?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZiEIJ8CfKQ1ds2fcmM8B1Id2xwG3MzkY1OoM7vFacoMTexg-2uRqQjaoOWpbN5Xvt8N2rO96NUoUt_88tyXxx1-SZDFo4EAFYJSMSEb2t3YIIRY6oMGby6iPawEIhGChqVx--Xuj_suGayC_xMoRsBhHZ-5aAyQJsA8NQtDOqdGS53keZwaroIsBnP9l1GFa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/emu2JdkHmMaP8CxE_7xBrk-JZASQlzSw51ovA6yRvM0RQyZb85QIGWJfZKyb1Vi17j32a5xfRdYyu470ge8XWT2TW69hTLNYbrDJcuSpha2bp7-0hF_8FI-mOwdcD2FfR81kOpbtMJiS3HV2_1SqLUcE_8u8jryowKtSCHhx8Nz4ijKlVCy1YxU32c_Lmr05?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OfEG3SpN0LSGUmMVnOiQaoZSqmEgjerw8bp_nRwBHCTbCgoyewvjbFOe6lvICQK5MvnITUhBb7pqjsguTPVINzg11YbC6zW_1gStax19fsKgbjBRPkKG2zK2kfs9wSOexi5SCzTmLWgvxe2DoMJ_iXAPjNy25wo7Q7b3Ge23USDHKehKUyz2KIhGIpuep2Aa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RXoqoTK-aHOjwA9cytjCJZLRpRQdRFSLD3Bby-Rb2-04WmJAOeRHWpadPMiIl3-e6sfOXrGt_EYgGy5QtXKReyYMtRggvukZKkrSQljEXD3OaPqfhHiUpqTsmHIzie3IbG82XL4MtAzF-Q6tiYvCcxpSOCMJBxZTvGJsbnwMGzkAS6BZmEWnNj6EkdF1NOJo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/McX5_gb4F7mz2EtQ3t-f90qRqjCqXKBX-JnySa9FsL6JnsYZ-PgZ77lL8tIYFB-JNGeepPvFsjf1baVlDO07ELp0VmNFWMLbl-MTsi-7TsAsnk2tVqGI16Ys7_Uc2URMbF9SwrIfT2oUh80ja_Mb2yHDofkuqvYYHYd0fGJ4nc7llL7xgBHOSKx3sJYqycPP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VxlJJfbK-5JWH5KfU_48yf_1hwn4UHicoF0kCdMQxVnDV1jeFm3pjvc6CoZHwMs8Cpq57OIh0ENkdoQsXmQZqoU4scLDKxDIw4OhKWBObouUCNg94U7sA5VmECCl4ophbr_PhT0Pnw-LI9lY_SXIGgz-YyEK5k5L0whvZoc_grR6vbSBllOqSQ0NAUV6gD0E?purpose=fullsize)

我們先認識三個主要零件：

| 英文名稱             | 中文名稱 | 功能                   |
| ---------------- | ---- | -------------------- |
| Rail             | 滑軌   | 固定在設備底座上，提供移動路徑      |
| Block / Carriage | 滑塊   | 安裝在滑軌上，承載移動平台        |
| Balls            | 鋼珠   | 位於滑軌與滑塊之間，讓滑塊以滾動方式移動 |

你可以把它想成：

* 滑軌是道路。
* 滑塊是行駛在道路上的車子。
* 鋼珠是讓車子能夠順暢移動的滾動元件。

這裡有一個重點：

**滑軌通常固定不動，滑塊沿著滑軌移動。**

在電動滑台中，通常是把滑塊固定在移動平台底部。

## 二、為什麼滑塊裡面需要鋼珠？

我們先做一個簡單比較。

假設有一個很重的金屬平台。

### 情況 A：金屬直接在金屬上滑動

![Image](https://images.openai.com/static-rsc-4/7EFIzABPcI39WkSrQnevX26rgtRuZWuNyP6lxWmpTq6fWZNtWLhz6dUR3G8sRsAAwBoWSvVHBcYStqdSgKRpnJtatoH26t1LaG0uxuoZh-m8q1YIirVLkN1qVbAhaWpDNubvU9HCxp2LbtPHk9dLsu_6HGgX4Dm7qu4e077E9XJtqdolDObzWZjQR7h8io1s?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lJYgaIuS85zOsSqxBpvKtM3KMTmtz0hGqSX50JTxbaUb0UxOc6QGdXvEgAMJhuzTNaj0hwFvwT4ddhiZk_pam6vqIfsdJF0502yHtn4hgTTuQQmmOXsQAeEGg_Hae4yBbPq68CJ31WxHqtBYY3yFSaf2Bkyg_crwlXO7DkLCYraYg2n9t0YBx30ZXL-QjHsX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/f7RkkGScvqikcMFqGKbqPn0EabpK-8JQM-O6dXriRcotfQ-XObLpBax64WoYs69NpiEtXjo0umQZnGMqN5-RF0SWgQQy1YeGGxdVUOsgtrGRcPy77F3JZ_nBiAai2jqu851uHWaNqcIAJSWhyG1ZoRp33PfTm_YIfQ8TlKj5IDZ9JbAKeQX-jc5-9_YESGdV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FIS6HfrIlfAejUUdbNXo3WVmkp5L5OPekGTR7fqBmlBwRNOmk885pLVUYtgzHaV98rmaF2GhqJoCQTbaL-v5nwny7bCvs_mwvchHqY82J_CDCJ1Jo-Ri-2nNTDe3P7uQ1j5Z-8CwtIbeuKl4zfUOUjasRR7-NbzZHiDf19J8OJalPZqTHDGRLyia0v3e73nS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dzRVpJ0x4z-oCzjqLDo_HVdl5U5RLdEeGCzAQHzhlT4gzoqXifgnxEkaOOthCKY8Akl3SKTZ_FHSzdFLkC2gewol4s6QoN_92tH3Y8IBCl_Y17cIWX3qg0PmbnjaXMYLCrxReizQfXcpBpaobKGF8MGEQndKXHscH7cIuUCNQEABvtt7drBsXgIRDQq861no?purpose=fullsize)

兩個金屬表面直接接觸，移動時會產生滑動摩擦。

可能造成：

* 移動阻力較大。
* 接觸表面磨損。
* 發熱。
* 精密定位受到摩擦變化影響。

### 情況 B：中間加入鋼珠

![Image](https://images.openai.com/static-rsc-4/zdCyslR33OHE5JQ6q2sWLBGhjnzhlMKEhrUdMqNOM84_ElFY0Wg3D2fejpw9L7KkVmGIv-A-HgSDydWj5JrpMmvGv2aRrCVj7ig6sf1V1s3NvYAM1Lyn4S5X1_Xzxl7vottD43ny2W_hKmWX7fTVwf5u_fPmSk9BlzGN3nb7OoEcTMwAdPT0FXfZ7VRe_XtI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u4MezLE708ABN4LWxJK4PbR_Y-f167Dag7aEirjaBuKG0PEquBihJzx5zFWyhI7T9Fc0oz4tKdTsCRjEalckutGdvvLeQuZIkELNM1chJz3HqWNxfZag9v2e-emBJD493inizkVT2-AzDj-YpYYSGnVH9DywHHcVvqZtWOsPUtEHHXlPwJu2_4zdkxmLf0o3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/70cyPrYuZoI9MmwmWmgoau6AsM_76AGeiY3KP_rRKVMxhNtTvvkOOHjGTfBN9KYYa_rLRmLrQVKE3q9K_e_9DxrtTnLjPdhgEwPmEMX9SgJpprEK16abug4QyTY4uWgiKqIseulps3D76mdf1q-kgmgN3B3t-xqWelqy98woLyVBZ3lMGannSx0VJgSJOPP9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dh_8s2yZJ4MDdW71Jfuv0g0JuabRXCmTP3MnN-d-BzwEMYE6PLV-Avdv1IBg1lPdBWpX8QzKhs46MpQXC5h4WIRRk4SukytogPssaIS_53azAW5BFTRbLxwS4Rhg4x_6362XRm1IyhGDTfKtyjYvJTJCntXVi-cYWW1GFFDrSIuPv0GlaV6NSod-Gr7LwwH0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/K-gXC2hOaU8xVjPtlbSCo33m7aGHmCrQKVyZHtL6sDLZ5qK3Z9y0l3K-wiG9x5HMHtu-S6ADPPw3buA70wWA9I4pJatjn5Q_ykvygEuT3fetP4jRjUJUxmszu3-KZ9P-Jx7RZKqX2_FdoY4kfdohfcBTRpKUmuUJ1VfwZrbuELSGgr5tQtP449u8YOGo8tGV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/asVSTuWwmI5priSDCIbRHM0qV0h12pZbfn_S1i-nJlgYGGm4f6YLdWQbfEsdNDuYYHkvq9xVnrUZ003YL9_Ju2rdRTAJjjcUMlXmnQzE-VftZlvJR0DP0f0pzUcvGAn_ho37Apga_k5yexEYar0RQj8MZ-AjbC-NWbddpYpvYi6PutqV7Rm72Ix8cmS_yJYo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HJKj8MPwdi0N8PLuZNS2kwv-Q96ddpuCgjVvLHR7XyX71mJNGjfqJ8_vGzDJyv248DahYcYHqiy-pHEZ8jmJH8cI-TPmdRr1WqjtT9erHgDjqWStVHCkwLK1joEx-42x3fRpXoWce_m913oMPFP8reeIVolEN4Dz5Ef18LQHXXWxn21p8mTaYqzAs3-kHJDE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/D1NM-DFv77tVFSZloTsey7_YH8CMsSdBbDqA_iW8-ncE2-Hf42HMZGiMIFcG1m2lG9ljuTxSqyb1WQQtR3UBradEU3tLCzaTGDAxQxJGh7ilPKuyOclXBnKXLPaDa8RLESPFmKbl8FWPot15BX2AMNN4mUAoLpRJFMWEiHKc9eB38XUHe1ptXU8wSEs-mQzk?purpose=fullsize)

現在，兩個金屬零件之間加入鋼珠。

當滑塊移動時，鋼珠會滾動，而不是讓兩個金屬表面直接大量滑動。

因此可以降低摩擦阻力，讓滑塊移動得更順暢。

這跟我們第一課學的滾珠螺桿很像：

* 滾珠螺桿：利用鋼珠降低螺桿與螺帽之間的滑動摩擦。
* 直線滑軌：利用滾動元件降低滑軌與滑塊之間的摩擦。

兩者都利用了滾動接觸的原理，但工作任務不同。

## 三、滑塊裡面的鋼珠會跑出來嗎？

這是一個很好的問題。

一般常見的循環式滾珠線性滑軌，內部有一套稱為：

**Ball Recirculation System（鋼珠循環系統）**

![Image](https://images.openai.com/static-rsc-4/9jJWywM0kry4lAXF7_6U2w96yChz2eycUew90pUl3vmD_nYN04NyOErQTRC86Po9mr8H9nfd5_iy_dWXqxH50lluQX-iYnvREg2rQ1kOfya4ks4iTYit0CwpyuVHVrcWZ4HSaCJVlRQO9PkKkZiRd87aSXupXvm4p0mjMowAux5M4GtkSMXvHwt-lC_orRSI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ccmf4l1dCJaBpwnKvGrgsp2v7ZUQVbeaBz8vQ0HHtYYoDCeU-XeKGR4SkK4QPV94nmgyS2bNk1LnrE_msUkOliQROFDVNlBsTGOPJCzD1ifaz2VbbAv_dRQiiC9x-mnLJzlPfeu7n9JxVKxXGT2XUIEtoiZj3i6Pihsy_z2tl8pnxYxuyDwuzr1hxz5-vnGd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/McX5_gb4F7mz2EtQ3t-f90qRqjCqXKBX-JnySa9FsL6JnsYZ-PgZ77lL8tIYFB-JNGeepPvFsjf1baVlDO07ELp0VmNFWMLbl-MTsi-7TsAsnk2tVqGI16Ys7_Uc2URMbF9SwrIfT2oUh80ja_Mb2yHDofkuqvYYHYd0fGJ4nc7llL7xgBHOSKx3sJYqycPP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RXoqoTK-aHOjwA9cytjCJZLRpRQdRFSLD3Bby-Rb2-04WmJAOeRHWpadPMiIl3-e6sfOXrGt_EYgGy5QtXKReyYMtRggvukZKkrSQljEXD3OaPqfhHiUpqTsmHIzie3IbG82XL4MtAzF-Q6tiYvCcxpSOCMJBxZTvGJsbnwMGzkAS6BZmEWnNj6EkdF1NOJo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Zj-tv4p9Uckxe7haVj27OMvzQghQ_Lc7Tv_9vKYnlLOGkQ6BpnTheZVxuWftBtwWKuUBXCyWv7hwWZcHJ4MVyftZmnlYUJyS-4dHjvy3RLG3TvE5tU04P-OjK5Ro94OFT8oWD1rNEf8Tm7aNJRp3BFEY2rv8xZ0RjlriHWXFNfQ1AktR9ERJYuZliDI9lCFc?purpose=fullsize)

它的工作方式是：

1. 鋼珠在滑塊與滑軌接觸的工作區域滾動。
2. 鋼珠到達滑塊端部。
3. 經過回流通道。
4. 回到另一端。
5. 再次進入工作區域。

因此，滑塊可以沿著滑軌持續移動，而不需要讓鋼珠一直往外跑。

不過，這裡要特別補充一點：

**不是所有直線滑軌都採用鋼珠循環結構。**

例如 GMT 也有使用 V 型溝槽與交叉滾柱的直線滑軌設計，部分產品採用非循環式結構。這是不同的機械設計方式。

## 四、GMT 有哪些直線滑軌？

我們回到 GMT 官方產品。

![Image](https://images.openai.com/static-rsc-4/PoBO8uoXAoAr_fQGz8NXUk7g1lb50MHkrp_2_wPOTr-UOCDZTN6BAJO-VGVciWvXEdjlDs1DUMmT6Wz2Iw6ochY8hMVozc1P1FtE0bQTjURJa4KXaHiJc1KiOOyb7WAM0jbMRr-fVB_dUu5geCm3HFIe9jFaxnM2mn5gfSrOZwPWTsubx1W9sZqbI8GbtlWo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KIAQ2QD46XF-LuW1detkMnGCsBxm35dAsk20SVV2q_T3EPCrhNh-VA2P5aail7B0LpHEk0JpCXngl3Lg2FjuEgF3UI5un6EaLVg0qFa4WMj5DaYFgRHzls6x-vQU0A8qZ6JpjiIrhkvIxiIgL2E3q3Y7PNJDM36VoBFabiLsstjwziP53OuLdw12CUoJoYot?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5Z0Wr6yu6OFfLeB9lbscHSNOsmIciAqynylpmwWizJM7Z4vkccFr_ZW2pqpGHTkMVbcRbUfcr1GW2yEG9rJX78hne9BwtyoR5VvEWgdlBJQ1ZMhWXT-2v9qz-6vnQZ1xJqTOfOyoGvrVE66366_NZlT_Lh1cU8tKvXa3gU907tpc_qj6ZYgVcfg1-4xRvOS8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MK7bhtjh9bld-U_0XVsxH555BJzyCkNdavTuMhaYExaMIC-wfmmU2SOV19n-5VL41jfyILmBizoA3jOFcy4N9hkftpSqm5VNg8ZD8Uq2X8FG0ghbPVEB9SIk9PmPAx-Vcp7TdOZjdaviGmIookq2vAIUiE8lXVpji_JKj0hS7U368qbJix9uOAh-mHRDTMF3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KxofftyV5LXpwh-fhPpl2QLBPayc92e9jN5Aas7QaNkBKb0EUvPlNqmltma6PQkyVT1qc6hIGFdaMSKF1DHaJDhvNtl_BUJ0-0B3pb1z2zf4wYJAyTLIJp7Te_yP-FIig5pNh9yQ7PpQk1udF3LxoQSuQV1m2H7ohAIUd7TTFUQuFJREnDKVL_vJ7ogfbqBA?purpose=fullsize)

GMT 的直線滑軌產品不只有一種。

官方產品資料中可以看到：

* GBD 系列
* GBV 系列
* GRD 系列
* GRG 系列

其中有滾珠型，也有滾柱型等不同設計。

GMT 特別介紹了 V 型溝槽與滾動元件的結構；滾柱型產品利用交叉排列的圓筒滾柱，提供高精度及承載能力。([高明鐵 GMT][1])

官方資料：

* [GMT 直線滑軌產品](https://www.gmtglobalinc.com/%E7%9B%B4%E7%B7%9A%E6%BB%91%E8%BB%8C?utm_source=chatgpt.com)
* [GMT 直線滑台產品](https://www.gmtglobalinc.com/%E7%9B%B4%E7%B7%9A%E6%BB%91%E5%8F%B0?utm_source=chatgpt.com)
* [THK 線性滑軌結構介紹](https://www.thk.com/opm/jp/en/linear/thklinearguide/?utm_source=chatgpt.com)

---

## 五、今天的小考

今天先不考型號，我想確認你是否理解內部結構。

假設有一組一般循環式滾珠直線滑軌。

當滑塊沿著滑軌前進時，裡面的鋼珠主要扮演什麼角色？

**A.** 鋼珠負責產生馬達的旋轉力量。

**B.** 鋼珠在滑軌與滑塊之間滾動，降低摩擦並協助承載負荷。

**C.** 鋼珠負責控制滑台前進的速度。

**D.** 鋼珠負責把馬達的旋轉轉換成直線運動。

你先回答，並試著用自己的話解釋為什麼。這次我們要確定你真的理解鋼珠在直線滑軌裡面的工作。

[1]: https://www.gmtglobalinc.com/%E7%9B%B4%E7%B7%9A%E6%BB%91%E8%BB%8C?utm_source=chatgpt.com "直線滑軌系列｜高精度傳動滾柱滑軌｜高明鐵 GMT"


```
**B.** 鋼珠在滑軌與滑塊之間滾動，降低摩擦並協助承載負荷。  這一單元我有一個問題，滑塊是不是和滾珠螺帽固定在一起嗎？滾珠螺帽已經有滾珠了，滑快為什麼還需要滾珠？
```

答對了！B 完全正確。這次你的問題更深入了，而且問到一個非常重要的機械設計觀念：

**既然滾珠螺帽裡面已經有鋼珠，為什麼直線滑軌的滑塊裡面還需要鋼珠？兩者不能共用嗎？**

答案是：**不能直接共用，因為它們是兩套獨立的機械結構，負責不同的工作。**

我們用圖來理解。

## 一、先看完整的電動滑台結構

![Image](https://images.openai.com/static-rsc-4/IkfMBqoakNOnehhD0XR9Q4FXs3y9Zrz7IvlZUgG5FO1pqEwcraFAdD9mkN4NuQYg75N7CxhMvYglDBO612YJWvARK-Ee4AMX2G8S7GzhMEQy6pI23ovZT7Bju8c0NS_xJBg4XSJ8kmt8LroIgngHNypaOA3Orvp2vFImBndpnhlkGoggFjZbZZ0EeK-MhNfv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2R7CivTekzWMp1cu-ke2hk30t8HKlG3V9WC4V9givf_XLog8K6IxP1UsoC1TuA1Q62srdkOuytM_MYRXmULA-LnxFafDoV6YJfjIOdOGUPpUyF551FVJEZnIum-2EXGoMvBaPscklxKZ4jc-XrdabE7CbbJkCEGhljo4jqB9UEfYGJsPUCZDY00f4HU76kNA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EibxTVBuTsiE5-gSzQv1d59ZRDzYrtJnIVirL2NXGyU8-Su5BgLpuoNLsi0tnnn1mJ_RBSrPBPZu9V8ZUWsA70QCXvwMMDLeBArWWj8-BrljUqyqkeqipJuT3yYvS9stf2AxZtnsk_hHdlqOBMxW4naoGKy85lnoYYYMRrxQ7sRF58qoGJknxHtBVFIk_eYD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tJujcF8TwuaEKoGWS5r4p1wooUim0G74XfSqPq_PCj0IvNXYMA2WSG6vEboVkSa2NGtAEPv9Qc4t8vqC7RhbjFT1XtoKJyHXlnUwBsq0PPf0pEZtySE0l0tu0iYVNkIhio29JgrWputuvUOaEAXidOGui27yIti2MhDkPmfsMfYz9aVWi4NUg7ahCe82RDN_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZaOnw3nKJ1oFKZ76o9ap4jJIiI3AwWNVF5ejDFgGAtKlFLfp1-SJExhBpcM76zYX05dxNJ06y2YvnLRq166j7Ho8t4Gs5ONxpeztKSjduOGHVHkBx94v3lObaNgeo92Lexuc6dTo2iPhHMRvyBl0x1601WJqVae0AeWaqit2gnAwOKneEhMdWfyt57mF5bLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IdRASkCj-PToSS0YBdD-acKFzizZPF5YQHktQQtlP-4_apTHh_zwGRH11P3zo4VJcAgKcQHn8m4zZnFh5hZqeqvZhyT4bzcJ0Zh5vNV2OmUvw7qNZUhdxnw2DCjgMvd5HZbDS1nqZdkXRNRnECJR8rqIvidA9Fcrbw14oIoFzZqTRD-hBLUuhCX5JI6IzCwq?purpose=fullsize)

先認識這幾個零件：

1. 滾珠螺桿（Ball Screw）
2. 滾珠螺帽（Ball Nut）
3. 直線滑軌（Linear Guide Rail）
4. 直線滑軌滑塊（Linear Guide Block）
5. 移動平台（Moving Table）

這裡有一個重要觀念：

**滾珠螺帽與直線滑軌滑塊，通常都會固定在同一個移動平台上，但它們不是直接固定在一起就能取代彼此。**

例如：

* 滾珠螺帽固定在平台中央。
* 左右兩側的滑塊固定在平台底部。
* 平台把這些零件連接成一個整體。

因此，當滾珠螺帽移動時，平台跟著移動，滑塊也跟著沿著滑軌移動。

## 二、為什麼滾珠螺帽需要鋼珠？

![Image](https://images.openai.com/static-rsc-4/sC_B7agCm-rpwOlw8glJ5Yax0keLbOKqcPjeHpZ4RA0XEqlgvGgzdGD3x2KGZRU4RXsELxuzvtXjerQ4hGnxlajwGkUbYYoLvokgRIc7yWHwXDGCUomJCTy_uslHIqaqO7h0aG1NzFricH60Q2pkZA7DHJylYgibMHxQyDU_ipIxOpuF7mcb30irqX63n902?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/24PcbwrX_xkfq7DRxzoJZ7GTrRCjslZGPajsC-Q5qaR0PRen5pstaAXvJSigcwsno5VzRtyVKC_KK2lEq4hHRNlaH9OPIKEnR8eeAkTqJp5wMPLd1hsZUUm9yejUGo7wmPnl4s_y_8Ixcfx5hpYZbdrfbrN5dxn2wOXob-6PdSb7eg_vvC10K7WVURtcnoEv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zfRb9YDtynw84Wb0oegpAFWHWvRey-ZzLs_MaaMBPd_rjE9WMuglsr0N-nqMP5D3LjGgCly0Ct4OGw-rdQboCNyr1cfS-Ur5PZEP4Zj5NSj9rLD4traGfNxe8Cmi7GTPShxeguYhINbJtPRT4OW_geB7y5cA0m5wbBCAxSsX35UtE8dJHbJZI-cesSFvKsuq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/08X_xmOl5ucPfIjhcPS90o_AuXs3PCD_u0S3dV8M0G-xaqMMFqbYlBGHwxCwIN_3Z7pvCd7nsMHfoid-hYYg9vH2lDZFvqhR5FOjf1iEa4y36l_fYin0roiNNuZYDIdISiBoVBmYUaqYqjJYA8rG7QvmPnC3afwQe430yudKxd44S00362E8tueu9Ie20Ju9?purpose=fullsize)

滾珠螺帽內部的鋼珠，主要工作是：

**讓螺桿的旋轉能夠有效轉換成螺帽的直線移動。**

當馬達帶動螺桿旋轉時：

1. 螺桿的螺旋溝槽推動鋼珠。
2. 鋼珠沿著螺旋溝槽滾動。
3. 鋼珠將力量傳遞給螺帽。
4. 螺帽沿著螺桿軸線移動。

所以，滾珠螺帽裡的鋼珠，是傳動機構的一部分。

## 三、為什麼直線滑軌滑塊也需要鋼珠？

![Image](https://images.openai.com/static-rsc-4/u4MezLE708ABN4LWxJK4PbR_Y-f167Dag7aEirjaBuKG0PEquBihJzx5zFWyhI7T9Fc0oz4tKdTsCRjEalckutGdvvLeQuZIkELNM1chJz3HqWNxfZag9v2e-emBJD493inizkVT2-AzDj-YpYYSGnVH9DywHHcVvqZtWOsPUtEHHXlPwJu2_4zdkxmLf0o3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KIfY3G6kn6Jx3Bwx2Xoifpshvc1ohepihjb12A64pM6CLiEAviTsMF_HAFeJEYvjW3Bk1gIpBwdlGwnZCCKCOnyUeQD9fypAhVUJzOeO1mPP301FnBbYr1aNA-eenHZJ4h6JX7l_fZRS6VuBjCqst9ICGrPndjByluRhlG-b_-74Fq9lCQD0BiWoQHIwsm6p?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EcEoxzVJ0xjh2D9ttIQPbeJ5YHMhx_hdCjDmVf0Yrko07EiV6IcRlrgt_en97cBEiOrh90ECRQOqZSmi1LaFNSvygbA03ccx-6_eVKG6Fq15B8gIlWpuToUD85moiAW909bvolgBwrBfIMTEmdlpQ8vIIq0ao-jOjNjw71uL-pMfdDhPzb6qPRm6TtjJkqqv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CjEFYE6SSPzPI8wbvKRsT_Nlrk14m9uCq5G14y3J7nbnql10y3OImGGcUE7gVKXpUqvVzFtrXE5IVJIhDTiBYlJyU7xbC9mafrSQuWwkYLFqlwxGdjJKUvnjr6G0nLa8EQ1zyD7Zldv6zqsjOoCTRRGcM1YC-DPJ3vVF32Rb3QjQhpso_zh8JdWNL5K35lAv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/asVSTuWwmI5priSDCIbRHM0qV0h12pZbfn_S1i-nJlgYGGm4f6YLdWQbfEsdNDuYYHkvq9xVnrUZ003YL9_Ju2rdRTAJjjcUMlXmnQzE-VftZlvJR0DP0f0pzUcvGAn_ho37Apga_k5yexEYar0RQj8MZ-AjbC-NWbddpYpvYi6PutqV7Rm72Ix8cmS_yJYo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HJKj8MPwdi0N8PLuZNS2kwv-Q96ddpuCgjVvLHR7XyX71mJNGjfqJ8_vGzDJyv248DahYcYHqiy-pHEZ8jmJH8cI-TPmdRr1WqjtT9erHgDjqWStVHCkwLK1joEx-42x3fRpXoWce_m913oMPFP8reeIVolEN4Dz5Ef18LQHXXWxn21p8mTaYqzAs3-kHJDE?purpose=fullsize)

滑塊裡面的鋼珠，工作完全不同。

它們主要負責：

* 支撐移動平台的重量。
* 承受平台受到的外部力量。
* 降低滑塊與滑軌之間的摩擦。
* 讓滑塊沿著滑軌順暢移動。

換句話說，這些鋼珠並不是用來轉換馬達的旋轉運動。

它們是讓滑塊可以承載重量，同時順暢地移動。

## 四、兩種鋼珠的差別

這張表請特別記住：

| 比較項目     | 滾珠螺帽的鋼珠      | 直線滑軌滑塊的鋼珠 |
| -------- | ------------ | --------- |
| 所在位置     | 螺桿與螺帽之間      | 滑軌與滑塊之間   |
| 主要工作     | 傳遞螺旋運動的力量    | 支撐及導引移動平台 |
| 接觸形式     | 螺旋溝槽         | 直線滾道      |
| 是否負責轉換運動 | 是，參與旋轉轉直線的傳動 | 否         |
| 是否可以互相取代 | 不可以          | 不可以       |

你可以把它想成：

**滾珠螺帽的鋼珠負責「推動」，直線滑軌的鋼珠負責「支撐與導引」。**

兩套鋼珠各司其職。

---

## 五、再回答你第一個問題：滑塊和螺帽是不是固定在一起？

更精確地說：

它們通常不是直接鎖在彼此身上，而是**分別固定在同一個移動平台上**。

例如：

```text
          移動平台（Moving Table）
    ┌──────────────────────────┐
    │                          │
    └──────────────────────────┘
       │          │          │
     滑塊       螺帽        滑塊
       │          │          │
     滑軌       螺桿        滑軌
```

這樣的設計讓：

* 中間的滾珠螺帽負責推動。
* 左右的滑塊負責支撐。
* 上面的平台把它們連接起來。

這也是為什麼一顆馬達就能推動整個平台。

---

### 課後思考題

假設現在把滾珠螺帽拆掉，但保留兩支直線滑軌、四個滑塊及移動平台。

請問：

**如果有人用手推動移動平台，它還能沿著滑軌前後移動嗎？**

A. 可以，因為直線滑軌本身允許平台沿著指定方向移動。

B. 不可以，因為沒有滾珠螺帽，滑塊就無法滾動。

C. 不可以，因為直線滑軌一定要有馬達才能移動。

這題可以幫助你確認：**直線滑軌本身就能讓平台移動，滾珠螺桿只是提供自動推動的力量。**


這次選錯了，正確答案是 **A. 可以，因為直線滑軌本身允許平台沿著指定方向移動。**

我們來釐清一個很重要的觀念。

## 1. 滾珠螺帽不是滑塊移動的必要條件

還記得我們剛才學過的分工嗎？

* 滾珠螺帽：負責提供直線推動力量。
* 直線滑軌：負責支撐平台，並允許平台沿指定方向移動。

假設我們把滾珠螺桿和螺帽整組拆掉：

![Image](https://images.openai.com/static-rsc-4/zfrBWQxfSlX4h0zdg2cjJmN2u21kgB4RfAMWUic5rOwPa8ay9QhYFmfh_eFzPvqU8t8sdOWr207mVQ7XHjo2lj6BF-zllne8Ws63iyhHQOw9F4yDNd67eMRFc-RNolzPZffK9lVRpi9XZCV6NCsev0gW41CByfPIpfZHKWmxZhRwzNVrXLD8ZuGJO18PvOwk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rbYqnwTlSVsvDcszexIOVeELbh2Fu7W7JgZDAMZi0n_kyRV0KtlZbpAtRQrt6r_fozFqnTFs7lAGTvDEjvC1pcENOkE-yXEJV3ofvWcjN4v0lGRaFtFjIeB0-UE_y8ojVB0vcwSnDkR_pFFdwV79b_z8iewZaEVweH470EZSn82kxx32vXMYxPz0S9mjdlBe?purpose=fullsize)

現在剩下：

* 兩支直線滑軌。
* 四個滑塊。
* 一個移動平台。

如果你用手推平台，滑塊內部的鋼珠仍然可以滾動，平台就能沿著滑軌前後移動。

**因為鋼珠是滑塊本身的結構，不是由滾珠螺帽提供的。**

## 2. 這裡有一個非常重要的區別

你可以記住：

**直線滑軌可以讓平台移動，但它不會主動讓平台移動。**

例如：

* 你用手推：平台移動。
* 其他機械推動：平台移動。
* 馬達搭配滾珠螺桿推動：平台自動移動。

直線滑軌只負責讓移動過程保持在正確的方向。

這也是為什麼有些精密設備會使用手動滑台，不一定需要馬達。

---

## 3. 我想再確認你是否真的理解

假設我們現在有一組完整的電動滑台：

* 一顆馬達。
* 一支滾珠螺桿。
* 一個滾珠螺帽。
* 兩支直線滑軌。
* 四個滑塊。
* 一個移動平台。

現在馬達停止，沒有任何力量推動平台。

請問：

**如果你用手推動平台，正常情況下它能不能移動？**

A. 不能，因為馬達停止，所有機械元件都會鎖住。

B. 可以，但實際推動阻力會受到馬達、螺桿、滑軌及其他機構影響。

C. 可以，而且完全沒有任何阻力。

這題要讓你理解：機械元件即使沒有通電，也不代表它一定不能被外力推動。


這次選錯了，正確答案是 **B. 可以，但實際推動阻力會受到馬達、螺桿、滑軌及其他機構影響。**

你會選 A，我猜是因為你把「馬達停止」理解成「整個機構被鎖住」。這個想法很合理，但實際機械結構不一定如此。

我們分開來看。

## 一、馬達停止，不代表馬達鎖住

想像一台購物推車。

當你停止推動購物車時，購物車不會因此自動鎖死。你仍然可以用手推它。

馬達也有類似的情況。

一般馬達停止供電後，轉軸可能仍然可以被外力帶動。

但是，不同馬達和驅動器的狀態會影響結果：

| 狀態         | 可能的情況        |
| ---------- | ------------ |
| 馬達停止供電     | 通常可以被外力帶動    |
| 步進馬達保持通電   | 會產生保持扭力      |
| 伺服馬達啟用位置保持 | 會嘗試維持指定位置    |
| 配有煞車器的馬達   | 煞車器啟動時可能鎖住轉軸 |

這也是未來我們學習馬達與驅動器時會深入介紹的內容。

## 二、回到電動滑台

![Image](https://images.openai.com/static-rsc-4/_LBq5_l_8ou0GcGy4CyyQPcLrPH4lp8wtZCb15Jt24_9cMBPQizx8vnhW-UJ7zGVfNRYN6FeHd_warE55yuTDk8SlS6WyhlQ2g2NUZW0M5f_KVT4ekX_j4C5n52Yj-B3T0pKEnuJzsVu3qjxdO3-C49opXnkgAU7mypVFJ1n53B0RUmQIX22_pDF154eUzQN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/op39sr1cY4P9lLFKgtjjwKIJI_OYKrHeXdhtA7iqzV27GUx3691lxGahkZ5FeheVDn1pY-XgARM_on3q5Dj6J1naGs1isVDHVOYMPCHrUH6rNLe-xZy22ZVtocF7ubfqQl_PxOfYBr4ztmO07VJIKJS12-50iDm9Vtz_yKLYOTL44LDEr5bzgFh0jEx16Kv2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vWoxt-nPF4xiWR9Hwk8vEACnJf6rXvABBTKmyO5cWNVoyg3h7Bs-zoLcZDIuQGHIk0tLVRQBkrfCLdjQ81UKG3a4sjiSkLYfgG8mnG_bB6aSUt3koHzDSYNMctHOhlMYDvcksis-vECj9Df-JbgagmabjmGig-_KYcPE-mDPl34BJZG8OkXv9uKFIo2vEErN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hmbf5KvhocQFtwGGgwOV5yWkSFUWiWLtTicGRkRkquqzyMsU1CddqRNFM3G423pCJOodw-qMFtH4N5gjbXlZb_Cdd4Ub_FtnvNMigwYRQIdWP-7i_Y4bXVhJBU6zUd5mMbcbzhc-1PWydsLa8pNI5lhDq9Qji0Wc5NXD0zkORPpPVNlNu6ZOJnPI3OyQS4ol?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/l4Eg5pd0HKulC1QnXQNQM4klpxtvQkP3eNOtZzT9I8tvj-ERpj93z2N-V2r16J9_xj93nSPoKhZ-Zaet4R7miB6CacHKBYEiIlfUou5RFPvsF2YsE6G7vrDF4Bgb4GXNL1e6vrjS6OkPI1-GsEa1hwVohQ9RxqhjFO962ARNvgiFjh9wWqzgl1CnlKd8rThZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NFJThwCc4unVI16Je8Pl8DyuzXZUcngCvRBbx7c1iKevinWEcF5l-dWth2ajVCDLRFIMt4vnxEEL3MpeUKIOb0MNWJ3c_pCEv3xJ3B0kjMBjha4xW1L_MVrNy_z9fYimkQrpPCzOASn-GHGx8qu3GXHRiPidPXZlEpswTk0NtHzfk33J11nvNQvO2Hq2RdU3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7zAHuyJ9xXx_vj8YKWFVHJU-tu4W3u4ewXxBl59TQbZXsc2FSUqB-jTQXvPJk-ZIwEn5f-Acq9p9WUkpBD5AYpWOKsjpe6QRMluV5Pd_P9Jjl15QEDcc-T6oORMDQ_4r0MG6WMsbqAlSzpRUA6Nj5ur5bD7bSpPmWmOuPBxExAV2vwO_lbHLqVLRWVIgRKUc?purpose=fullsize)

假設馬達沒有通電：

1. 你用手推動平台。
2. 平台帶動滾珠螺帽移動。
3. 滾珠螺帽透過內部鋼珠帶動滾珠螺桿旋轉。
4. 螺桿再透過聯軸器帶動馬達轉軸。

因此，平台有可能被你推動。

不過，實際上能不能推動，還要看螺桿導程、摩擦、馬達阻力，以及是否有煞車器或其他鎖定機構。

---

## 三、今天我們已經學會什麼？

目前你已經認識：

1. 滾珠螺桿：旋轉轉換成直線運動。
2. 滾珠螺帽：利用鋼珠傳遞力量。
3. 直線滑軌：限制平台移動方向。
4. 滑塊：透過內部滾動元件，沿滑軌移動。
5. 馬達：提供旋轉動力。
6. 聯軸器：連接馬達與螺桿，傳遞旋轉動力。

下一課我們可以進一步認識 **交叉滾柱（Cross Roller）直線滑軌**。

它與今天介紹的一般滾珠直線滑軌有什麼不同？為什麼精密光學設備會使用它？

不過，在進入下一課之前，我想先確認一件事：

**你現在是否已經理解，滾珠螺桿、滾珠螺帽、直線滑軌與滑塊是四個不同的機械元件，而且各自有不同的工作？**


我很高興你直接告訴我這件事。接下來我們要調整教學方式，而且我認為這個調整非常重要。

我觀察到，你其實已經逐漸記住各個零件的功能，但是當我把幾個零件放在同一個機構裡，問你「如果拆掉其中一個零件會發生什麼事」，你就容易混淆。

這代表我們目前需要加強的不是背誦，而是**機械系統的整體運作觀念**。

## 一、接下來改變教學方式

未來每個機械單元，我會增加以下四個環節：

| 教學環節   | 我們要做什麼               |
| ------ | -------------------- |
| ① 認識零件 | 看外觀、名稱、內部構造          |
| ② 組合機構 | 把零件放在一起，看它們如何連接      |
| ③ 動作分析 | 從馬達開始，逐步追蹤每個零件如何運動   |
| ④ 故障情境 | 假設拆掉或損壞某個零件，分析會發生什麼事 |

尤其是第三、第四項，我會花更多時間。

## 二、我們先建立一個重要的思考模型

以剛才學習的電動滑台為例：

![Image](https://images.openai.com/static-rsc-4/kb4252ETSg5LEkuO1IbHGAcv6NUpKfSmueRvtB_dUyG6JqxGKre_zEYMQFR3MAzKSXCOACgY2Z0SRbq-QOINuzamSCykD5k4NtoN8RNRc9TJuDoy4zIMcee_oK4vEazRI8ispvhttLc78A_pZwQNPSt3JYwZUVlkLOzxmve4rrqkJiovlIA05Dvfcrr8j_qc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Kpa13Hjaol62OM4PXsYxBzWFKQyFeIidPHvMIxQiY-a7y5EifErZn4G5jyAbegrG_OVsROQyJd7mPhtfx8KrbiZzyi_MpKrT7_ma6i2T0p7RO5HwfujQLDLepRzJiRc1YAJTSf8QNCBlVX701GcrQxzSLHLhsB5AmT5AJrbpu1hLluyTYSuaUyoxZyWcGYY8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_LBq5_l_8ou0GcGy4CyyQPcLrPH4lp8wtZCb15Jt24_9cMBPQizx8vnhW-UJ7zGVfNRYN6FeHd_warE55yuTDk8SlS6WyhlQ2g2NUZW0M5f_KVT4ekX_j4C5n52Yj-B3T0pKEnuJzsVu3qjxdO3-C49opXnkgAU7mypVFJ1n53B0RUmQIX22_pDF154eUzQN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZaOnw3nKJ1oFKZ76o9ap4jJIiI3AwWNVF5ejDFgGAtKlFLfp1-SJExhBpcM76zYX05dxNJ06y2YvnLRq166j7Ho8t4Gs5ONxpeztKSjduOGHVHkBx94v3lObaNgeo92Lexuc6dTo2iPhHMRvyBl0x1601WJqVae0AeWaqit2gnAwOKneEhMdWfyt57mF5bLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7cFLnrFHYF0BkbOp7YKnJoC2aTTzpHPj-kFmhfLz3iJFuQbDMuuJl_W2xTtG18Ksar9nsiJNvv8XXXRUBgu6pmzeFNnLayssRNFZIe-kfXEyfarD_VAeNdf5BdzJB30atwUo-pXtlPoNqqNztR1kDtHZ_G59dS9UiPWlf4gMS2aNFfg4Ea8LJ5z-3if9DvoS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xhpX5q6Sd796rP6xS7cZEJhJ22RZsdpz1mywf6j0foSrZcyVDcf5vEGbTSbX9ef5Vai2tWo7TD-ZZjDx2x1StcGrO22F0mZL9YQF90BJsSLkhlrUgf-1HoFRhFwx_H3x7oKHdr54rViIdqUn3CJRjyxxzCrO3gaa2K7bsfkCGW-ujj_yD00QNeHiy4v4qdko?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/98SOsApONkt31NuukaqxDsnHluDfuq7gqniJ12JZ7YF8h9JrkSGqSeeHrvkKAsNsP2T-zEAUOGmnGpPKie0q-FVaKG9NmQEDt3iQu_gwzKwrXwC4wnNxRIQR6jwgrDZG1Q8WBv_pDai4Eeho74V02FQyQTx9atBX--quz1EkMml_MYDhneoRaqWNITXvbpiE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-3s8Dqkc4ZYVL9UaIC1u-HcdbcoZsG4sVnaf_zA84r_mP-ZrBQpfyeim1k7FdEnR539WS1q2cA4j3yyPQHHlQfFdk6UdyrFLxLBVhulFFVasXu8oF0B-NS2Wcbs3RW3sjNStNgXY5XeI9cBXuP4xAC3w_PR5BK8JoP7Thj4VuJ5Vt3w2UCENEHLpBEuxXi7-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9NjxKXUtaA5Wz8PyMspCAs16XKHk7qvLDvC08RiOXWEI_oH8wYiZH_Zgp1wWoNquYNPdkAzhimVML9Oc8bcqE7D2IPU9hEveXtAgYEI-AQwY1WzCv_RVRTBaisB_qeXgnwqreaJsj0wEjixUCe9GTmiWyIBMWjbWYPUP7so0TQRqQpa-8WSXLWFZgPdOKSRR?purpose=fullsize)

請把整個機構想成三個不同的工作系統：

**第一個：動力來源**

馬達旋轉，提供動力。

**第二個：傳動系統**

聯軸器把馬達的旋轉傳給滾珠螺桿。

滾珠螺桿再透過滾珠螺帽，把旋轉運動轉換成直線運動。

**第三個：支撐與導引系統**

直線滑軌固定在底座上。

滑塊固定在移動平台底部，負責支撐平台並限制移動方向。

最後，滾珠螺帽也固定在移動平台上。

所以當螺帽移動時，整個平台就會跟著移動。

這個過程請記住：

**馬達 → 聯軸器 → 滾珠螺桿 → 滾珠螺帽 → 移動平台**

而滑軌與滑塊則在旁邊負責支撐和導引。

這兩套系統同時工作，才形成完整的電動滑台。

## 三、以後的考題也會改變

以前我比較常出選擇題，直接問你 A、B、C、D。

以後我會增加這些練習：

1. **動作題：** 馬達轉一圈，哪些零件會跟著動？
2. **因果題：** 如果滾珠螺帽沒有固定在平台上，會怎樣？
3. **故障題：** 如果其中一個滑塊卡住，會發生什麼事？
4. **判斷題：** 哪些零件負責產生力量？哪些零件負責承受力量？
5. **計算題：** 導程、轉速、扭力與推力之間有什麼關係？

而且，我不會因為你答錯，就直接進入下一課。

我會先找出你混淆的地方，再用圖解和實際情境重新解釋，直到你能用自己的話說明。

---

## 四、我想先做一件事

在進入交叉滾柱或其他新產品之前，我建議我們先把目前的「滾珠螺桿＋直線滑軌」做一次完整的機構整合練習。

我們會從一個完整電動滑台開始，逐一追蹤：

* 馬達轉動時，誰跟著轉？
* 誰沿著直線移動？
* 哪些零件固定不動？
* 哪些零件一起移動？
* 如果拆掉某個零件，會產生什麼影響？

**我的目標不是讓你考試每題都選對，而是讓你看到一組真正的機械設備時，腦中就能想像它內部是怎麼運作的。**

這樣等我們將來學到伺服馬達、驅動器、編碼器及六軸運動控制時，你就能真正理解程式背後所控制的機械行為。



很好！這次你已經掌握了整個機構最重要的兩個動作：**滾珠螺桿負責旋轉，滾珠螺帽負責直線移動。**

我們逐項確認，並補強還不夠完整的地方。

### 1. 馬達轉動時，誰跟著轉？

你的答案：滾珠螺桿。

**正確，但還可以更完整。**

實際動作順序是：

馬達轉軸旋轉 → 聯軸器旋轉 → 滾珠螺桿旋轉。

注意：馬達外殼通常固定在底座上，只有馬達內部的轉軸旋轉。

### 2. 誰沿著直線移動？

你的答案：滾珠螺帽。

**正確！**

因為滾珠螺帽固定在滑台平台上，所以螺帽移動時，平台也會跟著移動。

### 3. 哪些零件固定不動？

你的答案：滑軌。

**正確，但還少了幾個。**

一般這種電動滑台結構中：

* 底座：固定。
* 直線滑軌：固定在底座上。
* 馬達外殼：固定。
* 螺桿支撐軸承座：固定。
* 滾珠螺桿軸：旋轉，不隨平台直線移動。

### 4. 哪些零件一起移動？

你的答案：滑台。

**方向正確，但要記得還有其他零件。**

以下零件通常一起沿直線移動：

* 滑台平台
* 固定在平台上的滾珠螺帽
* 固定在平台上的滑塊

滑塊沿著固定不動的滑軌移動。

### 5. 如果拆掉某個零件，會產生什麼影響？

你回答：「不拆馬達情況下，會產生偏差；拆掉馬達，可以用手動。」

這裡要再釐清一個觀念：

* **拆掉直線滑軌：**平台失去導引與支撐，可能歪斜、偏移或卡住。
* **拆掉馬達：**如果其他機構仍然完整，某些設計可以透過手動轉動螺桿，讓平台移動；但要看聯軸器、煞車器及機構設計。

---

## 下一個小練習：確認你真的理解零件之間的關係

假設我們把**滾珠螺帽從滑台平台上拆下來**，但是：

* 馬達仍然正常。
* 滾珠螺桿仍然正常旋轉。
* 直線滑軌與滑塊都完整。
* 滑台平台仍然安裝在滑塊上。

請問：

**當馬達轉動、滾珠螺桿旋轉時，滑台平台會不會跟著移動？為什麼？**

這題很重要，因為它能測試你是否真正理解「滾珠螺帽」和「直線滑軌」各自負責什麼。


你的答案「不會」是正確的！但後面的原因需要修正一下。

**滑台平台不是固定在基座上，而是固定在滑塊上。**

我們重新看一下兩者的關係：

```text
       滑台平台（可以移動）
   ┌─────────────────────┐
   │                     │
   └─────────────────────┘
       ↑             ↑
      滑塊           滑塊
       ↓             ↓
   ═══════════════════════
       固定的直線滑軌
   ───────────────────────
             基座
```

### 為什麼拆掉滾珠螺帽後，平台不會自動移動？

1. 滾珠螺桿旋轉，原本會推動滾珠螺帽。
2. 滾珠螺帽原本固定在滑台平台上。
3. 因此螺帽移動時，平台也跟著移動。
4. 現在拆掉螺帽，螺桿就失去與平台之間的推動連接。
5. 滑軌仍然允許平台直線移動，但不會主動推動平台。

**請記住這個重要區別：**

* 滾珠螺桿＋螺帽：負責提供直線推動力。
* 直線滑軌＋滑塊：負責支撐及限制移動方向。

### 再確認一題

現在螺帽已經拆掉了。

如果我用手推滑台平台，平台能不能沿著直線滑軌移動？

請說明你的理由。

```
沒有螺帽後，平台欠缺移動的力量，也沒有連接關係，用手推動應該還是可以移動，但因為只剩下滑台內的滾珠，需要較大的力量推。
```

這次你的理解已經很接近正確了！尤其是你說：

> 用手推動應該還是可以移動。

這個判斷是正確的。因為你已經分辨出「提供移動力量」和「允許移動」是兩件不同的事情。

不過，有一個地方需要修正。

### 1. 沒有螺帽，為什麼還是可以移動？

因為直線滑軌本身就允許滑台沿著指定方向移動。

即使拆掉滾珠螺帽：

* 滑軌仍然固定在底座上。
* 滑塊仍然安裝在滑台平台上。
* 滑塊內的鋼珠仍然可以在滑軌與滑塊之間滾動。
* 所以你用手推平台，平台仍然可以沿著滑軌移動。

### 2. 需要比較大的力量嗎？

不一定！

你提到「只剩下滑台內的滾珠，需要較大的力量推」，這裡要特別注意：

**滑塊內的鋼珠不是因為失去滾珠螺帽，就需要承受更大的摩擦力。**

正常情況下，滑塊仍然可以正常滾動。

實際推動所需的力量，取決於：

* 滑台本身的重量。
* 滑軌與滑塊的摩擦阻力。
* 其他連接機構是否仍然存在。
* 馬達、螺桿是否仍然連接並產生阻力。

---

### 3. 這一課最重要的觀念

請記住：

| 機械元件 | 主要工作          |
| ---- | ------------- |
| 馬達   | 提供旋轉動力        |
| 滾珠螺桿 | 傳遞旋轉運動        |
| 滾珠螺帽 | 將旋轉運動轉換成直線推動  |
| 直線滑軌 | 提供直線移動路徑      |
| 滑塊   | 支撐平台，讓平台沿滑軌移動 |

### 最後再考你一題

假設現在：

* 滾珠螺帽已經拆掉。
* 直線滑軌完整。
* 滑台平台完整。
* 馬達已經斷電。

你用手推平台，平台可以移動。

**如果把直線滑軌也拆掉，只剩下滾珠螺桿與滾珠螺帽，情況會有什麼不同？**

試著用自己的話說明，不必急著回答，我們要把整個機構的組合觀念建立起來。

```
因為固定方向的直線滑軌拆掉了，馬達也斷電，還是可以用手推動平台移動，只是方向很不穩定
```

很好！這次你的回答抓到了最重要的觀念：**沒有直線滑軌，就失去了限制移動方向的機構。**

你的答案基本正確，不過我們再深入一點。

### 1. 拆掉直線滑軌後，會發生什麼事？

目前剩下：

* 滾珠螺桿
* 滾珠螺帽
* 滑台平台（假設螺帽仍固定在平台上）

當你用手推平台時：

1. 滾珠螺帽可能沿著螺桿移動。
2. 但是平台已經沒有滑軌支撐及導引。
3. 平台可能歪斜、旋轉或偏移。
4. 如果平台受到不均勻的力量，甚至可能卡住。

所以你說的「方向很不穩定」是正確的。

### 2. 我們整理一下目前學到的核心觀念

這個電動滑台其實有兩套不同的機械系統：

| 系統   | 零件               | 功能           |
| ---- | ---------------- | ------------ |
| 傳動系統 | 馬達、聯軸器、滾珠螺桿、滾珠螺帽 | 提供並傳遞直線移動的力量 |
| 導引系統 | 直線滑軌、滑塊          | 支撐平台、限制移動方向  |

兩套系統必須互相配合，才能讓平台穩定地移動。

---

### 3. 下一題：再增加一點難度

假設完整電動滑台有：

* 一顆馬達
* 一支滾珠螺桿
* 一個滾珠螺帽
* 兩支平行直線滑軌
* 四個滑塊

現在其中一支直線滑軌上的兩個滑塊發生故障，無法正常滾動，但是另一支滑軌正常。

**你認為滑台平台可能出現什麼現象？為什麼？**

這題要思考的是：兩支滑軌不只是讓平台能移動，還負責共同支撐平台。

```
雖然另一隻滑軌正常，但受到其中一支直線滑軌上的兩個滑塊發生故障，無法正常滾動，這個電動滑台移動會發生偏差 例如： 歪斜、旋轉或偏移  
```

正確！這次你的回答很好，而且已經開始理解整個機構的連動關係了。👍

你提到三種現象：

* 歪斜
* 旋轉
* 偏移

這些都是可能發生的情況。

### 為什麼一支滑軌故障，會影響整個平台？

我們先看一下俯視概念圖：

```text
          滑台平台
   ┌──────────────────────┐
   │                      │
   └──────────────────────┘
      ●              ●       ← 左右滑塊
      │              │
   ═══════════════════════   ← 左側滑軌（故障）
   ═══════════════════════   ← 右側滑軌（正常）
```

實際上，兩支滑軌分別固定在底座上，四個滑塊固定在同一個移動平台上。

當其中一支滑軌的兩個滑塊無法正常滾動時：

1. 左側滑塊移動阻力增加。
2. 右側滑塊仍然可以正常移動。
3. 馬達與滾珠螺桿仍然嘗試推動平台。
4. 平台左右兩側受到的阻力不一致。
5. 可能造成平台偏斜、卡滯、受力不均，嚴重時也可能損壞其他機械零件。

注意：不一定每次都會真的歪斜，也可能直接卡住，取決於故障程度及機構剛性。

---

### 目前我們已經建立的觀念

你現在應該逐漸理解：

**滾珠螺桿系統負責推動平台，直線滑軌系統負責支撐及導引平台。**

兩者不是互相取代，而是分工合作。

### 下一題：考驗整體機構觀念

假設一台電動滑台：

* 馬達正常。
* 滾珠螺桿正常。
* 滾珠螺帽正常。
* 兩支直線滑軌正常。
* 四個滑塊正常。

但是，**滾珠螺帽與滑台平台之間的固定螺絲全部鬆脫了。**

請問：

1. 滾珠螺帽還會不會沿著螺桿移動？
2. 滑台平台還會不會跟著滾珠螺帽移動？
3. 為什麼？

這題可以幫助你理解「零件本身正常」與「零件之間的連接正常」是兩回事。

```
1. 會，因為滾珠螺桿和滾珠螺帽是一組移動的套件，不受滑台影響。2. 因為只是鬆脫，還是會移動，但失去精準度。
```

你的第 1 題完全正確，第 2 題則需要修正。這是一個很重要的機械觀念，我們慢慢拆解。

### 第 1 題：滾珠螺帽還會不會移動？

你的答案：會。

**正確！**

因為：

* 馬達帶動滾珠螺桿旋轉。
* 滾珠螺桿與滾珠螺帽之間有螺旋滾道及鋼珠。
* 螺桿旋轉時，螺帽仍然會沿著螺桿軸向移動。

這個動作不需要滑台平台參與。

### 第 2 題：滑台平台會不會跟著移動？

你的答案：還是會移動，但失去精準度。

**這裡不完全正確。**

請注意原題的條件：

滾珠螺帽與滑台平台之間的固定螺絲「全部鬆脫」。

原本的連接關係是：

```text
滾珠螺帽 ──固定螺絲── 滑台平台
                         │
                       滑塊
                         │
                       滑軌
```

螺帽透過固定螺絲把推動力量傳給平台。

如果固定螺絲全部鬆脫，螺帽與平台就失去可靠的固定關係。

因此：

* 滾珠螺帽：仍然可以沿螺桿移動。
* 滑台平台：仍然受到滑軌導引，但不再受到螺帽可靠地推動。
* 平台可能停留在原處，也可能因摩擦、接觸或其他外力而移動。

所以不能直接認定平台會跟著螺帽移動，只是精度變差。

---

### 再確認一個關鍵觀念

我想讓你自己判斷，不急著進入下一個主題。

假設：

1. 滾珠螺帽正常移動。
2. 滑台平台正常安裝在四個滑塊上。
3. 滾珠螺帽與滑台平台完全沒有固定連接。
4. 沒有其他機構連接螺帽與平台。

**問題：馬達旋轉 10 圈，滾珠螺帽移動了 50 mm。你認為滑台平台是否也一定移動 50 mm？為什麼？**


```
在回答問題前，我要問的是平台、滑台、滑塊之間的關係，我很混淆，是哪一個元件與滾珠螺帽固定在一起？基座就是平台嗎？
回答問題：既然滾珠螺帽與滑台平台完全沒有固定連接  ，滑台平台不會移動
```

你這個問題問得非常好！我發現前面使用「滑台」、「平台」這兩個詞時，沒有明確區分，確實容易讓你混淆。

我們先暫停考題，把這幾個機械元件的關係弄清楚。

## 一、先認識三個不同的東西

在一組典型的電動滑台（Linear Stage）中，我們先區分：

### 1. 基座（Base）

基座是整組機構的固定底部。

* 固定在設備或機器上。
* 直線滑軌固定在基座上。
* 馬達通常也固定在基座上。
* 正常運作時，基座不會跟著移動。

### 2. 滑台平台（Moving Table）

這是承載工件、鏡頭或其他設備的移動平台。

* 可以沿著直線方向移動。
* 滾珠螺帽通常固定在這個平台上。
* 滑塊也固定在這個平台上。

### 3. 滑塊（Linear Guide Block）

滑塊安裝在直線滑軌上。

* 滑塊內有鋼珠。
* 滑塊固定在滑台平台底部。
* 可以沿著固定的直線滑軌移動。

## 二、三者之間的關係

看下面這張簡化示意圖：

![Image](https://images.openai.com/static-rsc-4/lFppUzOmb7btaz7jFxoGB055RUxhsSj1BJy7ZN0ABq7eGamhQ8gU5YgfkrrS1aMLHHFeJkzQ8Cat0dZL8qdxszEutAvNfnMRjtcavRupKvKvmtxbTgboitLRGh0CHCZFjthUsZb11dMSR9wLhfjtdWTZVd8FVnnP9seNo9kNqBKriT5drEWCIE1IleL1Zifw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fhR2CgYnYdz_dP0ot31uWhlQacq81fRbe4waM36d6hlQgV9BL8H44v-JAhS3BEWNwdWbH1h9NXKNB_cmcF3QPxjXWxEcxLCfQAlENglL8tcNADYcHm_gHx5m_RhyBj37THE5S3YgoLmvfnsT2NrkFlkHNPerAUgM9ZG4xXYwPXnbDESmUvJ-DeGlFdiNgDJW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VlOlGimnAlfSS5dVk4ABDHqikIg72h867qZRnDHwnnDWsaxYVjRRKBDtzplperGYOtAbyek-xyis3JyK41OShlIPj50zdW2lRL637evE8n-x6e_nNpk2CPrK6IYnOc_G5aVG7J2Iyw38DAJBeCQ7UrPZGW7vINS2u4ph4Xx58Jm8x-Eb4C6UBVD7ibM3F__W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZaOnw3nKJ1oFKZ76o9ap4jJIiI3AwWNVF5ejDFgGAtKlFLfp1-SJExhBpcM76zYX05dxNJ06y2YvnLRq166j7Ho8t4Gs5ONxpeztKSjduOGHVHkBx94v3lObaNgeo92Lexuc6dTo2iPhHMRvyBl0x1601WJqVae0AeWaqit2gnAwOKneEhMdWfyt57mF5bLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SHFjumwhqHtZaU8oICN7C1ia0a66_3H2h2RWHq_JSkzlfD3JnD2u-Jwt1Y8gMfkEvGqwvifAJ7sL-GoELli4qYiMgVjFeYpFnc33oOq0nC_AoTd2-bflB5ldvhenEczmh_Hx7tdU3gfw8aneop0FEk-qwKs1f8jsW9itaUOldSI73_Fz-htaDT8KbUd1HCqH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bEm27rUzPchALgLVGX-Ay2w1pn8OJjXF__Wm7UsFOkevPymbEA76P5xSLVIMXsSPJ_aA_6HZykSYnxxkTA6dwN55V9Kjjt4H60W4Ud44FlSEYyx-jjHk4TqZrztM3fzpturs9G7UmYlBVj_fOvBTxkB2kD_N5Dh3PDCgP2jqe9IdlHtidqEVSN1Y73W-nWCG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/j5XjD_VTqj2Wr965SkWROQiIrSQQzRYFDPQHyzAGTkMFfJaGNjMWgnf-cKTLlY7KZUR7gxXt5L-UVTeDIMxBRMmXQoFdVmN43N7FpCBwyd1wvPlTSCdiqhno7yvNaO2dSoIv436Fz_vPMEJYAsle4M6GKk5XK5L4HY2Oq279Vw94htNZjTQxQgZoxEFdYHZV?purpose=fullsize)

我們用簡化的結構圖再整理一次：

```text
             滑台平台（移動）
       ┌─────────────────────┐
       │                     │
       └─────────────────────┘
          │       │       │
        滑塊    滾珠螺帽   滑塊
          │       │       │
       ═══════  螺桿  ═══════
        滑軌             滑軌
       ───────────────────────
             基座（固定）
```

這裡有一個很重要的地方：

**滾珠螺帽與滑塊，通常是分別固定在同一個滑台平台上，而不是直接固定在彼此身上。**

因此，它們的關係是：

| 零件   | 固定在哪裡？  | 運作時   |
| ---- | ------- | ----- |
| 直線滑軌 | 基座      | 固定不動  |
| 滑塊   | 滑台平台    | 沿滑軌移動 |
| 滾珠螺帽 | 滑台平台    | 沿螺桿移動 |
| 滑台平台 | 滑塊及滾珠螺帽 | 一起移動  |
| 基座   | 設備本體    | 固定不動  |

所以請記住：

**基座 ≠ 滑台平台。**

基座是固定的，滑台平台是移動的。

---

## 三、回到你剛才的回答

你回答：

> 既然滾珠螺帽與滑台平台完全沒有固定連接，滑台平台不會移動。

**這個答案正確！**

因為滾珠螺帽雖然會移動，但它沒有與平台建立推動力量的連接關係。

所以螺帽移動，不代表平台一定移動。

---

## 四、我們先確認一個觀念

假設現在：

* 滾珠螺帽固定在滑台平台上。
* 兩個滑塊也固定在滑台平台上。
* 直線滑軌固定在基座上。

當馬達帶動滾珠螺桿旋轉時：

**請問以下哪三個零件會一起移動？**

A. 基座、直線滑軌、滑台平台

B. 滾珠螺帽、滑塊、滑台平台

C. 滾珠螺桿、直線滑軌、滑塊

這次只要選答案，並說明你的理由即可。

```
B. 滾珠螺帽、滑塊、滑台平台 。 因為基座與滑軌都是固定
```

完全正確！這次你的答案和理由都正確。👍

你已經掌握這組機構最重要的固定與移動關係。

### 再看一次完整關係

**固定不動的零件：**

* 基座（Base）
* 直線滑軌（Linear Guide Rail）
* 馬達外殼
* 螺桿支撐座

**一起移動的零件：**

* 滑台平台（Moving Table）
* 滾珠螺帽（Ball Nut）
* 滑塊（Linear Guide Block）

**旋轉的零件：**

* 馬達轉軸
* 聯軸器
* 滾珠螺桿

### 最重要的三個關係

1. 滾珠螺帽固定在滑台平台上，負責傳遞推動力量。
2. 滑塊固定在滑台平台上，沿著直線滑軌移動。
3. 直線滑軌固定在基座上，負責限制滑台平台的移動方向。

你可以把整個機構想成：

**滾珠螺帽負責推，滑塊負責導引，滑台平台負責承載工件。**

---

### 下一步：確認你能不能從頭描述動作

假設馬達開始旋轉，請你按照實際發生的順序，說明：

1. 馬達首先帶動哪個零件？
2. 哪個零件將旋轉運動轉換成直線運動？
3. 哪個零件負責限制滑台平台只能沿直線移動？
4. 最後是哪個零件帶著工件一起移動？

這次不用選擇題，試著用自己的話描述整個動作流程。

```
馬達先帶動滾珠螺桿，在滾珠螺桿上面的滾珠螺帽會將旋轉轉成直線運動，可以前進和後退。滑軌換透過滑塊將滑台平台固定住，使其只能直線運動。將弓箭固定在滑台平台，就可以直線移動。
```

很好！這次你已經能夠把整個機構的動作連貫地描述出來了。尤其是你已經理解滾珠螺帽與滑軌、滑塊之間的分工。

不過，有兩個小地方需要修正。

## 一、逐步檢查你的描述

### 1. 馬達先帶動滾珠螺桿

正確！

實際上中間通常還有一個零件：

**馬達 → 聯軸器 → 滾珠螺桿**

聯軸器負責連接馬達轉軸與螺桿軸，傳遞旋轉動力。

### 2. 滾珠螺帽將旋轉轉成直線運動

正確！

更精確地說，是滾珠螺桿與滾珠螺帽這一組傳動機構，將旋轉運動轉換成直線運動。

螺桿旋轉，螺帽沿著螺桿軸向移動。

### 3. 滑軌透過滑塊將滑台平台固定住

這裡稍微修正一下。

滑軌並不是將滑台平台完全固定住，而是：

**限制平台只能沿著指定的直線方向移動。**

如果完全固定住，平台就不能移動了。

### 4. 將工件固定在滑台平台

正確！

你寫的「弓箭」我猜你是想說「工件」。

工件固定在滑台平台上，就會跟著平台一起移動。

---

## 二、完整動作流程

![Image](https://images.openai.com/static-rsc-4/J6Werk9ubTbg6XTnKvtSqo69Q0uyzvlzWf1jwXofQzib8ieFynbqdB8NS650s3fIcTSsI43xUBQM8x8s8GqkyUwY3XZYexRxhFh3mrhLAzH_GRct3axdSjbABR5mHfjNZIfoID9EeEf1wqFrSA-GynJkz7FW8IXnYfxPmgorf2LU1H6MFgPdyqFgLx5LJDo0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ramVm9sIvcoKWXjbYmRKG13jvd1YrwdO8MnpxU12fGp5IJbGn5twwMgF2E5MiB4V_C5PI23KS97o3z9YQcpgiJRCe6sKPTQsTtwvjMyqEtGihrvP00Y6KW1BYGd3YWYcj44e1GnDTsFFdf3ejFZmRNq3nmCMRgY-MDoO3m1MBHa4nkRru3MdFehsUu1MU8db?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lFppUzOmb7btaz7jFxoGB055RUxhsSj1BJy7ZN0ABq7eGamhQ8gU5YgfkrrS1aMLHHFeJkzQ8Cat0dZL8qdxszEutAvNfnMRjtcavRupKvKvmtxbTgboitLRGh0CHCZFjthUsZb11dMSR9wLhfjtdWTZVd8FVnnP9seNo9kNqBKriT5drEWCIE1IleL1Zifw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kb4252ETSg5LEkuO1IbHGAcv6NUpKfSmueRvtB_dUyG6JqxGKre_zEYMQFR3MAzKSXCOACgY2Z0SRbq-QOINuzamSCykD5k4NtoN8RNRc9TJuDoy4zIMcee_oK4vEazRI8ispvhttLc78A_pZwQNPSt3JYwZUVlkLOzxmve4rrqkJiovlIA05Dvfcrr8j_qc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qgtGaSB987FpT3ezzT_V1ktEaEa8sON1gTcp90CvcAORR5ny06aTfodaYAYuhemUoXNfkQXcQAJ4x90xbN8BlRS_AY8xEx9AIYPH1H9lTwMW1IzACZqyuWvqACZF1fBdcQGJiv4-STTHO10itC_kpiHmtfuWf-bggFaBlwtnW-YsHmx_e8p_nTSV1fet4-wm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Kpa13Hjaol62OM4PXsYxBzWFKQyFeIidPHvMIxQiY-a7y5EifErZn4G5jyAbegrG_OVsROQyJd7mPhtfx8KrbiZzyi_MpKrT7_ma6i2T0p7RO5HwfujQLDLepRzJiRc1YAJTSf8QNCBlVX701GcrQxzSLHLhsB5AmT5AJrbpu1hLluyTYSuaUyoxZyWcGYY8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UOgi_quHl4yGv1x9bA1zHc2TWSCDgMaCC-wDEOOGjPgshsqYQTnummtatZ74EKy5Y6qDcgUjW9GiNz0KIQURoJG6ytIpg4b4FYbBxOj77FHmNJQA3iz1yXJNC3N9fPX2YB1Ayqu2hvWXwb0d5xQB00ZaWaevHw1K_RTnFxNl9XsR63NVz8KIqBGOf3XpaNlc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/drhxMhxrO4XuocFsGYlDyJa7PHIi15ElVnKtgg9UsxwnwSUdSNdtgQf9NyL-hXCGnv24twNNaLdZbsLCI5PhjA_lyaIRWcY2z4SdxhGxwOriix3iHGeBiBuNqSjcv3TrUIRFP638OuzS3TVLUW8Fi70-WWsxtyIBTdnqVysyhgLb1Fg2JR2Q4WknCRrnqfGZ?purpose=fullsize)

我們把整個機構串起來：

```text
        馬達
         │
         ▼
       聯軸器
         │
         ▼
     滾珠螺桿旋轉
         │
         ▼
     滾珠螺帽直線移動
         │
         ▼
     推動滑台平台
         │
         ▼
       工件移動

另外：

直線滑軌固定在基座
         │
         ▼
滑塊固定在滑台平台
         │
         ▼
限制平台只能沿直線移動
```

這裡有一個非常重要的觀念：

**滾珠螺桿系統負責推動，直線滑軌系統負責導引，兩者合作讓工件穩定地移動。**

---

## 三、今天最後一個小考

假設現在馬達帶動滾珠螺桿旋轉，但是滑台平台上的工件重量增加了很多。

請問：

1. 滾珠螺桿還能不能推動滑台平台？
2. 馬達需要提供的力量會不會增加？
3. 直線滑軌與滑塊在這個時候扮演什麼角色？

這題會帶我們進入下一個重要概念：**負載（Load）與機械元件承載能力**。

```
聯軸器   你還沒講，所以我沒有寫出來。滑塊是不是很像火車的鋼輪，沒有固定在滑軌上，但是沿著鋼軌(滑軌)移動，意思是，滑塊可以隨時取下來，不必用什麼工具。
回答問題：雖然重量增加，只要馬達的扭力規格充足，有驅動器支援調整，直線滑軌與滑塊  還是可以正常運作
```

你提出了兩個很好的問題。尤其是滑塊和火車車輪的類比，確實可以幫助理解，但兩者有一個重要差異。我們分開說明。

## 一、先補上聯軸器（Coupling）

你說得對，聯軸器還沒正式教過，我不應該直接把它放進考題，這部分是我教學順序沒安排好。

![Image](https://images.openai.com/static-rsc-4/NAruLpbmZrzEtdQWEFo8TwZZasRasmkgc7tNAEPFrruQlRoUfhcpui-7UC7t3CdmIbLYDwRlOABS8-m68XLiveEJSG70f8LrftFA5hU963-SEJiBsjYw-8wRvYZ0APgJt_YD9-6rZ5b9DuL9qHrBQQf-SF3aqxyTOOqXRN-uaiZutpIkDx-B0Q5Wke7-jT64?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Y5TWM_dvR1ek1swSng8OG3OvRvbXKU87vQAEeRWPFdJVuAQDJAnSS9-V94EQ8YTC1EQox82tmnQt89edmKbBlvAcIiBSF0eCVQGR_6cNS9zwHwzD_dXWldtwJf1VNHD_-FFgq_RjUDaW-cHtQ_HnEy-SFJiorubqRXRP1H6iYkL2pXZww90sqk0dBC1-_o5u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RSCwuUtbTfC4q2P3ZUA8Q_TEHEnVkUgy8mYI3GR5xmZdqRSYEcboFdFK7M0ZG6aPHD3Y6MkKgdyVG1oet7zthnDY5ZzhpFbjjjkc_SfJe_8lWIUxugy4whZ37gEgrFe9a0X1rpNJo0tD2zaYNAgjXPMhsbJC94oSCZKzud3gIJ7W9FBhpoFpQCh73UxECeoV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8RHZCBli9EiiUWS6fr_UECZXwfXPetB4DQ97naTOY5mOwRiFWrkWtitcOBih1_qxEalbqAfSd64NmoXNhwjhB6Jkenljbv54k2ds_6IBOrloSRAo5ErfFnKJzlyne_kgHzIkYj_gAh72Lno3OdQVzAqiAG8qBhAOapSEJ--X14-tXuvfXo8oJ8pAn0n3cWEe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wGKJj97AGYH0vJL8gkIVjRo9FOjQTzg7WmNa0jLNgo3OS3hMHkXp4wQjR5Y6F7vsNZkCJc98157-BhLqDlp4I_qvDGxRD1TP3sZBYpUjSwq4AshasYBeMjftodm2FCRKDnBL3BxC_yc7ghDp1e_1kVrGUCWg75LRBtopnDj3kGlJElERs3uANePdG0NjM-oe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cxZKsPePB6ax-u5OWAX6StxTaIVqff5Y0MaJJ4Ac8GCsWGWEnDjC1vHt6nw2_CRHkMDIsxnTvip9FKruTSCkJ0mVR_mq7sMGEhvHIvpnd1peIgdrTt0EGi2ssIbGsIlnO8CMwPqTonWwmV3dwApW2F8PMRNlskbYSecNuTGxGQ6BaTvPPrbtZRiR--YAGz8W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UQ4kUH65SqsIbDat_tkSis7pHx-jNOecP_KpapkQ4Zt8a-e9ybbj-hORcQM9iNLTLNdXItnOenHlIE1f3X5WBGSS2IGEIn1JvQDk40VEhMQsddO9s3MSydfRXZ9cMLCMlEVTgAsog4U0VL8ccES4I_zdqsCNX1Skcco0yEDOdNLgdc4uXCHS30XiCFgeY6oo?purpose=fullsize)

聯軸器是用來連接兩根旋轉軸的機械元件。

在電動滑台中，通常是：

**馬達轉軸 → 聯軸器 → 滾珠螺桿軸**

它主要有幾個功能：

1. 傳遞馬達的旋轉動力給滾珠螺桿。
2. 容許一定程度的安裝偏差。
3. 吸收部分振動及衝擊（依聯軸器種類而定）。

你可以把它想成兩根軸之間的「連接橋梁」。

注意：聯軸器不是用來產生動力的，真正提供動力的是馬達。

---

## 二、滑塊是不是像火車車輪？

你的想法有一部分非常正確。

![Image](https://images.openai.com/static-rsc-4/ZiEIJ8CfKQ1ds2fcmM8B1Id2xwG3MzkY1OoM7vFacoMTexg-2uRqQjaoOWpbN5Xvt8N2rO96NUoUt_88tyXxx1-SZDFo4EAFYJSMSEb2t3YIIRY6oMGby6iPawEIhGChqVx--Xuj_suGayC_xMoRsBhHZ-5aAyQJsA8NQtDOqdGS53keZwaroIsBnP9l1GFa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dh_8s2yZJ4MDdW71Jfuv0g0JuabRXCmTP3MnN-d-BzwEMYE6PLV-Avdv1IBg1lPdBWpX8QzKhs46MpQXC5h4WIRRk4SukytogPssaIS_53azAW5BFTRbLxwS4Rhg4x_6362XRm1IyhGDTfKtyjYvJTJCntXVi-cYWW1GFFDrSIuPv0GlaV6NSod-Gr7LwwH0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OfEG3SpN0LSGUmMVnOiQaoZSqmEgjerw8bp_nRwBHCTbCgoyewvjbFOe6lvICQK5MvnITUhBb7pqjsguTPVINzg11YbC6zW_1gStax19fsKgbjBRPkKG2zK2kfs9wSOexi5SCzTmLWgvxe2DoMJ_iXAPjNy25wo7Q7b3Ge23USDHKehKUyz2KIhGIpuep2Aa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rOmX-yslGXShXRXKifT6zYJWf2xmlst3Jxpq3UcebXRgNDaXpBckNURsubOEGr8OLMwpB6QwNNa7VKW1sbjfJVWWE_60hOshhNzlVJY0i8f6MlSTXNhooQ6PlC1B9SEyABtm49jYPpb6kz90ncmjubqYAlkHWbVHruGZg7DkijPlWNkEct96NTOuWXRW8gIo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-ENtpT7A7fxyIEZYfPnzGOKeYU9IMC7h18aNpghlxnqd8IhLGj967w_dO_fswkNLac7Bz9OIvE2dvAyev5DLmrOmmTN8tle8zsoprbQAvWhLzGmMCxLIj_zaYzsIRy79YjqIVnl5fXXQnsWUPwcoKPuHYMM2JmA8s-CvA-LZla4vCG7kuTFBPQaOVm0NFeqJ?purpose=fullsize)

火車車輪與鋼軌的關係：

* 車輪沿著鋼軌滾動。
* 鋼軌提供行走方向。
* 車輪受到鋼軌支撐。

直線滑軌與滑塊也有類似的概念：

* 滑軌提供移動方向。
* 滑塊沿著滑軌移動。
* 滑塊內部的鋼珠在滑軌與滑塊的滾道之間滾動。

但是有一個重要差異：

**直線滑軌的滑塊不是單純放在滑軌上，而是透過內部的滾珠與滾道結構，包覆並約束在滑軌上。**

### 那麼，滑塊可以直接拿下來嗎？

通常不能像火車車輪一樣直接往上拿起來。

一般循環式滾珠直線滑軌：

* 滑塊受到滑軌幾何結構的約束。
* 正常安裝時，不會隨意脫離滑軌。
* 通常可以從滑軌端部滑出，但需要依照廠商規定的拆裝方式操作。
* 如果拆卸時沒有使用適當的輔助工具，可能造成內部鋼珠脫落或排列錯亂。

另外，滑塊通常還會用螺絲固定在滑台平台底部。

所以要分清楚兩件事：

1. 滑塊與滑軌：透過滾珠及滾道結構配合。
2. 滑塊與滑台平台：通常透過螺絲固定。

---

## 三、你的負載問題

你的回答：

> 雖然重量增加，只要馬達的扭力規格充足，有驅動器支援調整，直線滑軌與滑塊還是可以正常運作。

**大方向正確！但還有一個重要觀念。**

當工件重量增加時：

1. 滑台平台承受的重量增加。
2. 滑塊與滑軌承受的負載增加。
3. 滾珠螺桿需要提供更大的推動力量。
4. 馬達需要提供足夠的扭力，才能讓平台依照要求加速及移動。

但是，不能只看馬達扭力。

還必須確認：

* 滾珠螺桿的容許軸向負載。
* 滑塊與滑軌的額定負載。
* 滑軌能承受的傾覆力矩。
* 馬達與驅動器的連續扭力及峰值扭力。

例如，馬達扭力足夠，但滑塊承受的重量超過額定負載，機構仍然可能損壞。

---

## 四、今天先確認一個觀念

我們先不急著進入下一個機械元件。

假設有一個滑台，上面放置一個很重的工件。

馬達、滾珠螺桿、滑軌及滑塊全部都正常。

但是工件放置的位置非常偏向滑台平台的左側。

**你認為這時候，為什麼即使工件總重量沒有超過滑塊的額定負載，滑台仍然可能出現歪斜或受力不均？**

這題是要讓你開始理解機械設計中另一個重要概念：**力矩（Moment）**。


很好！我們今天先把聯軸器弄清楚，再回頭討論力矩。

你問的三個問題都很重要，因為聯軸器是馬達與滾珠螺桿之間的關鍵連接元件。

# 一、聯軸器連接的兩根旋轉軸是什麼？

以電動滑台為例，通常是這兩根軸：

1. **馬達轉軸（Motor Shaft）**：馬達內部伸出來的旋轉軸。
2. **滾珠螺桿軸（Ball Screw Shaft）**：與滾珠螺帽配合、負責旋轉的螺桿軸。

看一下結構：

![Image](https://images.openai.com/static-rsc-4/d9603dxsLjmNRYw4sQ4sbVj2JrZlLT7clJ0wcLTcqwrpRdWB_CJze_CZmO333hdOxrqTaJCnB7rBil0ADbv8JORKIL0VUzjn26PIlCLbyIJUqBpu8B85Oqdh2PUAepaAQnGCx7elYpQzS8vRcZEYA_Q1zjsVrX6-P5m3BjMbEYlIc5woDsktwgWL1I1EIU1-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_LBq5_l_8ou0GcGy4CyyQPcLrPH4lp8wtZCb15Jt24_9cMBPQizx8vnhW-UJ7zGVfNRYN6FeHd_warE55yuTDk8SlS6WyhlQ2g2NUZW0M5f_KVT4ekX_j4C5n52Yj-B3T0pKEnuJzsVu3qjxdO3-C49opXnkgAU7mypVFJ1n53B0RUmQIX22_pDF154eUzQN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z9eHmNH8UZuhh1co6n6xFEdMUer9IZsAd_J_X7Z8kFyXFMkyI22PH96Zdy4B0sLyswTIfpkSphnuVWadTBsQphLVFLvC_iH5ugbmfidM0c5PbmGn1dk_oXTutVJc7nidnfOYhafwAsMWvT1_htMh2B2fLhtGMEqso3B5YT2X-eMoa--oWK6f4PMNt13bIwEw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_TSk5pYtJ54V1doBb-ULD91bWrBpUAYEJc1ZcVdAJiRpD1_Uzl0cdt3CQMAqvjA9W8MvElRlIN3FhPCuXbV1nJMolCb2WsTTSZFZ7MpLvuR5songW27La4Z8ONwgvh1TrHyB4Jv2rZFzdsucgZPGkhsAavP4HDa8sxjvuqiLE6m9pIrGz3m68nkvFb09lv5K?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zped5003FlogGo60F5eGDTx4H6pjkZlpehiunZV7ZGbV0dsQmsEU9Mt1vVGgTxnneqDqGIyPo-HVh9Wz6ILtP9_cS146F0Hc6dOS9qai7pqsM0vfc9lpmZVvGXz7r6LGZ_xBOZNpHYUJx3nCjB2uR33Y7OZ-c_tbOJrUJOyDlPGtPahgvSNUHsiqJdOjxSmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LHPZNrf8p0Y0kc_iyVXXKvLzTE9zv3MCII0-8nqGTJnEqi42Bjw5pBK994Ek2Wu0W7Rj2oE8LlfqZ5YBtmgXP_GoZXTkt9hzCjMZ2uqubTltcxaqTVMb8HuOGNfxaMRIHRBvAkyPz3UDEnTxRTvPPsTZB2wXLDlmUa5qTpjOYxqCWHTEHN7tn6aBwniIovOH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ramVm9sIvcoKWXjbYmRKG13jvd1YrwdO8MnpxU12fGp5IJbGn5twwMgF2E5MiB4V_C5PI23KS97o3z9YQcpgiJRCe6sKPTQsTtwvjMyqEtGihrvP00Y6KW1BYGd3YWYcj44e1GnDTsFFdf3ejFZmRNq3nmCMRgY-MDoO3m1MBHa4nkRru3MdFehsUu1MU8db?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tJujcF8TwuaEKoGWS5r4p1wooUim0G74XfSqPq_PCj0IvNXYMA2WSG6vEboVkSa2NGtAEPv9Qc4t8vqC7RhbjFT1XtoKJyHXlnUwBsq0PPf0pEZtySE0l0tu0iYVNkIhio29JgrWputuvUOaEAXidOGui27yIti2MhDkPmfsMfYz9aVWi4NUg7ahCe82RDN_?purpose=fullsize)

它們的連接方式如下：

```text
       馬達
    ┌────────┐
    │        │
    └───┬────┘
        │
        │ 馬達轉軸
        ▼
    ┌────────┐
    │ 聯軸器 │
    └────────┘
        │
        │ 滾珠螺桿軸
        ▼
    ═══════════
      滾珠螺桿
```

當馬達運轉：

* 馬達轉軸旋轉。
* 聯軸器傳遞旋轉動力。
* 滾珠螺桿跟著旋轉。
* 滾珠螺帽沿著螺桿移動。

所以，聯軸器最主要的工作就是：**將馬達的旋轉動力傳遞給滾珠螺桿。**

---

# 二、一個聯軸器只能接一個滾珠螺桿嗎？

一般標準的聯軸器，是連接一對旋轉軸。

例如：

```text
馬達轉軸 ── 聯軸器 ── 滾珠螺桿軸
```

這是最常見的配置。

如果一台設備有兩支滾珠螺桿，通常不能只用一個普通聯軸器直接連接兩支螺桿。

不過，可以透過其他傳動機構讓一顆馬達驅動兩支螺桿。

例如：

![Image](https://images.openai.com/static-rsc-4/rngao0sKc9Lua1u32MO14SYyQUO4P36QDhHp5EQiicly1zvUWci4OrOhIcUJaDXAlEnHOwjmuTTYesA3TWioIN0L7ZavtEYM8xPzKL9Rm6VrhmUaNDuQsMR5Pi5rsXDHBsdd738yyS4bZL7RBD7tjMW7hhJo5d2Q4zCZBdsomQWYB4mbd9LsjOUXnT7gTO_d?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CdL3ygMZO7lh1LetnP0J18NJ3Wj2rkgkr66LQsTizdB-NKgfpO5WEFXeM6SEjN2341NYcR3uUAD9a8nEhYma_Hiof-eSP0Qd68NYW8qB8vU-CAi2oe3N-F1vASVRu6WTNFK-VpzNdGnu252qKuYyxlfnXQpCY-OkZULjNluGM9R9mBVRPpmxtwOQbX41zKvz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UXeHZ_GXhnbnfCZZRDU1RN18oDmwlxq3n3SEG8JBT6gxOC_3fTKKVS7Ryr4ilseQu7QaZNbI4Bv1nq5VLFil0S0Vstbhe8AgoDVboAxBN7cYuO3wsTd6F6EA_3m9moDTZBYsKGYbWAv80PudYiKqRHEBCOhO5W68GhlayDjJutX5tDJmMNoUobx_b2cyG__q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/aJkqUJz2gRT_HmL4WH2xMXHJ69XuqLVA5PAV2BUWLhs56AoWvfnN_c9BDpL2stP-8jUuM7C9Gq3elkzC2kOIV5I-OnLpQ37cXTD5ifAfcst13rjefZAF4VV-doZHt60ZGP6-p3-Nr3MvigkhcPzDB3GKZjoQMJkyAtu58t3CpvpyztQcQZhm95wLpucJ5C2W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hTuZC2MBtgMD2iEu1E4GLE7WHz-L9enQVghJnK7gjVMxUjMtGBBLj8gfnUCSs_NEh8f7PDrkcsVHdlbf2LsR7mo4O4fHvuH7CBcHNiTNWd2DxDGLSCZpStZ7sckoCPgcu663UJKDIpZ1qNNYCq3ZJysE6xK6hBunlX-dsSuZkxzRp40Qt6CLkZ8TIj8-ECZB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hrLOvKQtTNGit0Ry0wDxwEZznWzvFN-UZdQhED4K6y7o_CMqo3vRYYpx8w9iIBvI_c39z98BGcymTOSAqHyvAp5_nLp_uYBGw_jMlwJHMOWyUvMrjHPVB6MRn6D_Fi8BWHqcQeqdcvJ9OKASVMme8z87QdlMW1ymNOEAdszHrEBZU354u_tHHuNA4cRhCdBq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4s_zZd5LquVh4fogU5ssYt_FbX978KD8itxaXyeRM5leA7JL66IpUh4RY-xIhg1EoT-P9qW-uOmC8TLhIRUyeUy0zpcYElNvTs3QCbfgxq__UDxCwlhRJfWNUfE9Z8nehpEwT29_oqyx6n0yuZUQh22m-TqSfORQ0Muph6K-7O7l7xldVBeApQ74r4DLPTjp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/l_5WlYmEA8HtwVbQJ6W5CAHm02wOj0_bhE9burp7JidSAkZYp1E4RJQQFjN-YESMbN4iYcAGtaPQRJjowULhdwuejz0nKGGUjtdBZDtqN2pqtyyJ_-3JfZMBa2YzHvfcak7e89yPEjLfAUsYkqiksEOCP9BjAkwp7w3F11zBj3yF5TXnioG36f3MIieiWWDh?purpose=fullsize)

這種設計可能使用：

* 同步皮帶（Timing Belt）
* 齒輪（Gear）
* 傳動軸（Drive Shaft）

讓一顆馬達同時帶動兩支滾珠螺桿。

但這時候就不只是聯軸器的工作了，而是另外一套傳動系統。

---

# 三、一個聯軸器只能接一個馬達嗎？

一般也是如此。

標準聯軸器只有兩個軸端連接位置：

* 一端連接馬達轉軸。
* 另一端連接滾珠螺桿軸。

因此，不是用一個普通聯軸器把兩顆馬達直接接在同一支螺桿上。

如果需要兩顆馬達共同驅動一個機構，必須使用特別設計的傳動或控制架構。

### 這裡請記住

| 零件   | 主要功能         |
| ---- | ------------ |
| 馬達   | 產生旋轉動力       |
| 聯軸器  | 連接兩根旋轉軸、傳遞扭力 |
| 滾珠螺桿 | 將旋轉運動轉換成直線運動 |
| 滾珠螺帽 | 沿著螺桿產生直線移動   |

---

# 四、回到你剛才的力矩問題

你回答：

> 因為受力不均，相同的力量會讓滑台歪斜。

你的方向是對的，但我們要再增加一個觀念：

**不只是力量大小，力量作用的位置也很重要。**

例如：

想像你用手推一扇門。

* 推門靠近門軸的位置，需要比較大的力量才能讓門轉動。
* 推門遠離門軸的位置，只需要比較小的力量就能讓門轉動。

這就是力矩（Moment）的概念。

力矩與力量及作用點到旋轉中心的垂直距離有關：

**力矩 = 力量 × 垂直力臂**

因此，即使工件重量沒有超過滑塊額定負載，如果工件放在平台非常偏左的位置，也可能產生較大的傾覆力矩，讓滑塊與滑軌承受不均勻的負荷。

---

### 今天最後確認一題

假設你有一扇門：

A. 用手推靠近門軸的位置。

B. 用相同力量推遠離門軸的位置。

哪一種情況比較容易讓門打開？為什麼？

這題答對，你就掌握力矩最基本的概念了。


你的選擇 B 是正確的！至於為什麼，我們今天就把這個觀念弄懂。

這其實跟你前面學習的電動滑台有很大的關係。

## 一、先想像你在開一扇門

![Image](https://images.openai.com/static-rsc-4/k4B8Lhm3Q20fZxD89TvLV1r0fkW9XRJwHhLtS7oAFR19h3-ktBvI0-YYHsDssOs9ysPW60jmpVerpa6hC_bys3LsewxcZlVzun0yHeTZNrUqjnwbWbrXQ8zYlH_Gu3que-8uGoilm-flx_9ztEZG2EkBsrkCtTbNDxek-lrK5GTomtg_0_G7DnTSFtWjw6QX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/g7xegs30mwCtq9JMrN5rL3yvojH4PodEiEpG2hKSoPpoIO_P8v4Vj_9XEHAGtjpcX4KWelNhD_9tojsjufPKOyuvXVIL7d8rVy8AsLRbS8-d9cRvRkO4pEg-5DNQcln_E88YUJPa73rDV4S2Ixql8z-KrgZHzKNaYLIM3qf3MVaag_IO2OIm8P_RgtGaLWmF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gVT_d1Bw_bu2e-rCVUnLqwiUU2PeEaDT2te616_oPfh7_Ew2Qm6SVctfsP2TQYp7Q795MmMIOZrbFY-OTh0s6lhhVAjc9DJ-bA1Yw364WUY2f-iosfyC-d7_Ix-yvSLCoWty7Y55SseRQw8iZkGdlfCO3hnPphAgias6Nn2IcC9AfC_AJWY1kZ6_AgU0D72A?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/P0q1gYtc8Jm2zaiFkTrSBiLdyCtl8xEQ-E_8jw7n9YMTaJLkB_yavyVCylaPibCyFDRXpNJ-xfRViZPHXwTBF3jBVcIsB0NPT24xs6J4Y-MptcY02rtcL21bMvWORQ7Vp-_K6LtQQdm2L3KBJr5PfSrY4OIfsTxAGzb4z5bdx1_6oZNXarPLac2SS69hhuR3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IT9wuSVECopYUuUK3U8r7JSu9y9M0bbJyYebGMbn0-UrZ7yjMHcEACue9daVYGrP-1bgKoD6SfARNDiv9DwBpbzRMl33wzOdiyeUzK_lLFnFBu8MLi6AY_Nu5LIvHszyUJWXLQ2YhzR3TCB9vs5x8NGv7BPsNvSr8rPab49zM-6ruvLYZyweQ2zjUwv693O2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OcfzgTld76WdbeYkn92CBLD2hWKkrZtPJAg-FgCdO1y9i4k5e2gr2BjLJ9nrMz0YQhmN2aQZQdIPoX9WVkmEsStPk2MQEAcK1H_TtSIJTg55H4v85tPxrLNbyj82zhIOdHreQrRa02xYl9RvsKNM7J8AeDu8r4FWrE9sWqKSnzEWhb2YPYIVtZQ6xnegcmKv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tuUfMFJ3WViUef1YFZwA-M9aMt7RlIHSWER_4kLEQyKsXj075CK50WLMXC6iwkYLEapnvFwmgB00S2X1H2hBAxyYBWKSwP4S75pThfZKS4b8L_WbdvCI1ZSnrn2nmGibShSKSsWHg4lintFLZrpveSCNnVRMURkvPtB_aR9f4yRr7BmRsc3kFVD1L4cxrO0T?purpose=fullsize)

門有一個固定的旋轉中心，就是門軸（Hinge）。

假設：

* 你用 10 N 的力量推門。
* 門軸到你手的位置分別是 10 cm 和 80 cm。

兩種情況：

**情況 A：靠近門軸**

你在距離門軸 10 cm 的位置推門。

**情況 B：遠離門軸**

你在距離門軸 80 cm 的位置推門。

雖然兩次都使用相同的力量，但是 B 比較容易讓門轉動。

為什麼？

因為你推的位置距離旋轉中心越遠，產生的轉動效果越大。

這就是力矩（Moment）。

## 二、力矩是什麼？

我們用一個簡單公式：

**力矩 = 力量 × 力臂**

例如：

| 推門位置 |   力量 |  距離門軸 |    力矩 |
| ---- | ---: | ----: | ----: |
| 靠近門軸 | 10 N | 0.1 m | 1 N·m |
| 遠離門軸 | 10 N | 0.8 m | 8 N·m |

所以：

* A 產生 1 N·m。
* B 產生 8 N·m。

B 的轉動效果是 A 的 8 倍。

這就是為什麼我們通常會在門把手的位置推門，而不是靠近門軸的位置推門。

---

## 三、把這個觀念帶回電動滑台

假設滑台平台上放置一個工件。

![Image](https://images.openai.com/static-rsc-4/r1VlimenSTDW6UsDQ7OGHFPFiWh6_f7g7s_VGLV_B4-lO2Y4REqH9QW_cwn3LYHQUK7JqfFZo7dNNFtirQMIHnjzE8ptbAiqNCAdnT74OxX336kRRn8ZiSE4eaM-GubnoACL9m2bdyVFKRq0RYfU8-2X78SDimOOnVxpYar3Opld8qyfaIluiA8He_SVY9hM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/x0XqlhRyWtJ3GyrbWW14IBqgqrKSAZlLUJ2N_nJRJaftYPRmllmFIwwM4w3rTQFs396Gc3CBBI6E8OslLyS4awGYfXBcc-aGwBQro_0SKGJfyHmkX9nEykdqj0-elmjsW_3ave04sQieDGAypD7oOCtT0AWKtT54ZYsaghbcyc_-IBbrJaWq-uEG3hYufldu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lH8A3Mxonu-toX0t-2AhrC5Id9O2gLYnlAE1_e3wXORjiO5o4VDExDVpaOokHgA6DquTz0hXK1SJ6RJcVC94vT0gaohTLXk2qsMnNkhBh15gPkgj5wrwKHmgDUHAdEDWd6c8hBFP6_BckmE1QvcGcUCp-OWFwLKcr6yb9RDN2uUOKYPuSrM2aNESZ3eZtrgu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/senFLNEN2rxv0PxOqEro_76HSzLBgJnJakwE9N3FNU7O_uVmI1EuobmLHP-5yQ_z0ctdM_IqsXJ-NGvvj4lzZfv-pmrgeHzRA1AKz_7EfmHKa1tHYnE5ILt1Js_uqXughOCr4aClvndIMRUIjWpDCs2whkLkZq80a8XY_CdOrXH_Shncvbg2MJ3QLxGPQCtg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zHCVElMzSoaLTfHPmpJJkubHQxSZnT0ob7uePN9Kn7EwsPGzGnJEzqXC35xHu9p8Fmbw-kbLZ8rLINy1c2RLjO0msgG_JEae09KaWAzZYKAapWADF9OmGXy7Focqg56taOGPjYkYg_hlm89Ze3tI6TDUwYQust9gxUP6mruzilVbe9jCtiIv6fCgDBGZOJKI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/L1hTPB0bFbq2_XU7eXaOCPBkAQiYOH6wLKyZCxDPNGdL2H3TrJlgriFQ4Wmfp6pFrz8A-ApJq1LScpuysDVyHUZ8qhI0Zfo5Ofc2EX6FRe3KVJRcBwUgScdVdXpl80-5SV4bT3RpuZim4tuqzXwUNcFVQYLmARzBgp6L0kTUGc7WdqpUykyOnOGSBbfFWrZl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/I22fFhKcC-HSQ9-ElMBWfVhd9VzVXhsO_SZcTwTsqfYSoqT-GDnsLpavNkFzUZwLp_JU5ofI43XZeq4LGy-n_Q17UsixdEEOzlLJAb-yECUCYwpANXfq6cqr-W0NZuP49NvUqc503xqZT_DksqhYGQmQlj684P3rAsyRmRnXq5Qenx_-cT7JJpFsaMoXnMxh?purpose=fullsize)

如果工件放在平台中央，重量分布比較平均。

如果工件放在平台左側很遠的位置，就可能產生較大的力矩。

這會造成：

* 左右滑塊承受的負荷不同。
* 滑軌受到額外的傾覆力矩。
* 平台可能產生傾斜趨勢。
* 嚴重時可能造成滑塊磨損或機構損壞。

所以，機械設計不只要考慮工件有多重，還要考慮工件放在哪裡。

---

## 四、我們做一個小練習

假設有一根長度 1 公尺的扳手。

你用相同的力量轉動螺帽：

* A：握住距離螺帽 10 cm 的位置。
* B：握住距離螺帽 50 cm 的位置。

**哪一種比較容易轉動螺帽？為什麼？**

你可以用剛才「開門」的經驗來思考。


完全正確！而且你已經能夠把力矩的公式應用到實際工具上了。👍

你的理解是正確的：

**力矩（Moment）= 力量（Force）× 力臂（Moment Arm）**

例如，假設你施加 10 N 的力量：

| 握住的位置 |    力臂 | 產生的力矩 |
| ----- | ----: | ----: |
| A     | 0.1 m | 1 N·m |
| B     | 0.5 m | 5 N·m |

B 產生的力矩是 A 的 5 倍。

所以扳手越長，在相同施力條件下，通常越容易轉動螺帽。

## 把今天學到的知識串起來

我們目前已經學習了四個重要概念：

| 機械元件 | 功能            |
| ---- | ------------- |
| 聯軸器  | 連接馬達轉軸與滾珠螺桿軸  |
| 滾珠螺桿 | 將旋轉運動轉換成直線運動  |
| 直線滑軌 | 限制滑台平台的移動方向   |
| 滑塊   | 支撐平台，讓平台沿滑軌移動 |

另外，也學會了兩個重要的物理概念：

* 扭力（Torque）：馬達讓旋轉軸轉動的能力。
* 力矩（Moment）：力量對某個旋轉中心產生的轉動效果。

兩者單位都可能使用 N·m，但在機械分析中代表的物理情境不同，不能直接當成同一個概念。

---

## 下一堂課的安排

我建議下一步先學習 **聯軸器的種類與結構**，例如：

1. 剛性聯軸器（Rigid Coupling）
2. 膜片式聯軸器（Disc Coupling）
3. 爪型聯軸器（Jaw Coupling）
4. 波紋管聯軸器（Bellows Coupling）

我們會比較它們的外觀、內部結構、適用情況，以及為什麼精密電動滑台需要選擇適合的聯軸器。

這次會特別加強實際機構組合的理解，不只背誦零件名稱。

---


好，我們開始下一課：**聯軸器（Coupling）的種類與結構**。

這一課我們會特別注意一件事：為什麼馬達與滾珠螺桿之間不能隨便用一個金屬零件連接？為什麼還需要選擇不同種類的聯軸器？

# 第三課：聯軸器（Coupling）

## 一、先複習聯軸器的工作

在電動滑台中，聯軸器位於馬達與滾珠螺桿之間。

![Image](https://images.openai.com/static-rsc-4/3PzrrGO_qvRLp_Bu6gX8a-XlaZz7yDLbo2SUt_4ft18_TudVe9p0KI0yQyxoplxJEcp_GF5pIJm3Xxxd1BpdgUxwDKELopn1DJTLdzBfOeAkJdFCXfZNcsHfc7Oqm5eGJEepqDQj_LTdz9unsf4ajYNPaREdVn63IfdVaeDGuFTiGuVQ6NVPKMM-n0Z89-lc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_TSk5pYtJ54V1doBb-ULD91bWrBpUAYEJc1ZcVdAJiRpD1_Uzl0cdt3CQMAqvjA9W8MvElRlIN3FhPCuXbV1nJMolCb2WsTTSZFZ7MpLvuR5songW27La4Z8ONwgvh1TrHyB4Jv2rZFzdsucgZPGkhsAavP4HDa8sxjvuqiLE6m9pIrGz3m68nkvFb09lv5K?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z9eHmNH8UZuhh1co6n6xFEdMUer9IZsAd_J_X7Z8kFyXFMkyI22PH96Zdy4B0sLyswTIfpkSphnuVWadTBsQphLVFLvC_iH5ugbmfidM0c5PbmGn1dk_oXTutVJc7nidnfOYhafwAsMWvT1_htMh2B2fLhtGMEqso3B5YT2X-eMoa--oWK6f4PMNt13bIwEw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dgFgzJ4z6HUwG6G1jIY86s24s8gEZxYemX4fY-r4ew51Cq2o6XBXgvnDhXZQ_ljunAQbQr2gLh3IzT1y_m1FHCe8SFqPJUYeiluQFDAFnLqlHCwYCYpZL7QcsNcC3ZPYMvd7Ww7PY2lFVhR6cr0Ty1eBlFkIG7Y57xE1tjy2z2FaOgWCTO3PYangR4zeJ940?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/F-PHRdMNuPxWjtYDSQlGJMwHJsnOidNp49Dqb8cgEFA9mPpeHMZfNXTY6vnKD2YEeHiJ1IAryfz-uZXm2PeDgAhWP1mEfqrPwOUoHm9-CLtSCiKzljWJtqfy5vVWxMHTievVjqAEfD6mQ5VT1QU1QBAvMH34pn5au_yi7VsdxczwjderC3I5e0chrxh71a3U?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UY9yol8HzlaQWFP03ZmkE9YuNfQtjJBWWfeLfP9GhOEt9TwAy1Ho-Qa-2iSuSkYUIl8Vdjp_41o9ODulZEPcirZYdMEhqpijSi4flN1cvFhD7JVoZ92ti4wcKmNPAD_bgTrtbfu35OenIILN1vjmevKy3MEROEUs8frkE-XWxLsNwzG2pd5vXmWG0g8hiTOp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TRbJuXaoog7X_av4aDy5b68lUJYd1z0BIE_68fMt4vPG4vn53Q6Duc87eE9wESklpY_zW85dUIkFfYVNyw8KKoOpVe1hHjPPNUsiJLxK5TUhBIcRHIks7VDVUCSdcUDgud8u3FUlt4uzMR8kki7hVZdBb3lpTt_Xutq58RwhafTtpw5GaVgSXmmznkzexsd4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3hgwywzpjJqM25M7VmOAxV39KLRNKxZsLgtAtYB1yBleBkZ4nU8qMaiE3Fyt6-bg12OUoVC6QwRsoWir4Kh_Ep54SOyVCns-lLgY37x1Vjf63rYrtmd14TwO-8P1s_QpOjAZEUO8k2iGykPWzklCiCgLwtihRHPYaceviAwTf208qxWGswMoKe1tjExOKED3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/82w5OKpKaz_-8kAFaoqJQY-GKugqlxOBCQh1a0tsDC-QuA6txpI1RCuq83DV2BHWxOmGjpat31NmCs28uKwJmMKZCL1iYy6J6g_yjjUkb1dumBeBSo83wuuBQXwppk6I7oMmlJsm65lTrixT7xXcOFg7fx3ayMMoafc4if5i2DNeHwPoXm8hRIqaTdGLyYzj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4v7_BgfYvIi6psurRmxsAnrKskO8pe6AZgWtVz7Xt6vV5vdjhOmz8RBVyXg131G7UVTC6AgB0SwnF9ZItPAH-RXWqkAuTAtnkJ15xsxp3NAZi_y_35W0dRpwPB296lns5INJCLs5FrYoXFI2YJQQ64CEvCtNR18I9xpmnk93vB4g1qy4bp1QUIzjSx8PAupf?purpose=fullsize)

它連接兩根軸：

1. 馬達轉軸（Motor Shaft）。
2. 滾珠螺桿軸（Ball Screw Shaft）。

當馬達旋轉時：

馬達轉軸 → 聯軸器 → 滾珠螺桿軸。

聯軸器將馬達的扭力傳遞給滾珠螺桿。

## 二、為什麼需要不同種類的聯軸器？

這裡有一個實際的機械問題。

假設我們要把馬達與滾珠螺桿連接起來。

理想情況下，兩根軸應該：

* 位於同一條中心線上。
* 彼此保持正確的角度。
* 旋轉時不產生額外的偏擺。

但是，實際組裝時可能出現一些誤差。

例如：

**情況 A：兩根軸完全對準**

```text
馬達軸 ───────── 螺桿軸
```

**情況 B：兩根軸稍微偏移**

```text
馬達軸 ─────────
                  \
                   \ 螺桿軸
```

這種偏移稱為軸心偏差（Shaft Misalignment）。

如果直接用完全不能彎曲的連接方式，可能造成額外負荷，傳遞到馬達軸承及螺桿支撐軸承。

因此，有些聯軸器會設計成具有一定的彈性，可以補償少量的安裝誤差。

## 三、先認識兩大類

今天先認識最基本的分類。

![Image](https://images.openai.com/static-rsc-4/oU0epF0jVYhHo7xO8uH788mfs6t1TsezoRTmkeajuwVsD45zDC9gSLUvz_Q0e1NjgxhhhOVwQKp6wQ9O0in-OeKSGExUUpiImy9SgmqzdBU02tgpPq3_RcYs0FZRQDSCwnA2bjOkVIs6VHokI5020w1epaUpXZusGsX3WC129xkk6jdzxZ0lnOhne3WLhD71?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1V0_51Sf2tDEBfGvnPC3F-oL7Lz8kCtijbzNWNvlYn4K1IGS8FDEmJOD7ItMYR6w16J1q8jEuulpLcC_N35wGcfWxbfspT_P3J71b_Do_koh3diJdyFQUXRLrLi5Z5Owt3JFhYts1OTmVE9rPT2Bs5SXwlrPydZs4kYFallsWEeXfS-PJCjt4YCvTUhLTIvI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cDjGnxMd1wBN4XsV05QoO0arAriPQMTskraSTImwovYVtCvQrs6wSpnKPKJiry7_mkeq1c65ASo0FN0dOsMKdi5a-MNujA6gsPYh0mPIw2tYsZehpzIZ4iOGKRo47rag-tJDNpnJhxu5HUKLNc6FoEn6dsreyiVX8zNrAX3qpiV2vVUZfWowDxIpXhfXyS6W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fu7dIcF8DpDXu7awLrwL9GrOHbrCu4UWhox3A8EorOiMZaA1rs1a0ufE50YemHpfWZ3yKpLsomywx_V6ZBaiyLw1fBf-xhUdLgkUJNzQWkwPFC4_Mkw-8zgnqbmLW0-roxVF52eWCXR6mFswP4I0V4vcEsB5-O3Ami5XnbngwQxnjpCsNZL4_BGLqL1uE9zR?purpose=fullsize)

| 種類                | 中文名稱  | 特性            |
| ----------------- | ----- | ------------- |
| Rigid Coupling    | 剛性聯軸器 | 幾乎沒有補償偏差的能力   |
| Flexible Coupling | 彈性聯軸器 | 可以補償一定程度的軸心偏差 |

注意：彈性聯軸器並不是可以任意彎曲，而是在設計允許的範圍內補償偏差。

常見的彈性聯軸器還包括：

* 爪型聯軸器（Jaw Coupling）
* 膜片式聯軸器（Disc Coupling）
* 波紋管聯軸器（Bellows Coupling）

我們下一步會逐一認識它們。

---

## 四、今天的第一個練習

假設你正在組裝一台精密電動滑台。

馬達轉軸與滾珠螺桿軸因為安裝誤差，兩者中心線有一點點偏移。

現在有兩種選擇：

**A. 剛性聯軸器（Rigid Coupling）**

**B. 彈性聯軸器（Flexible Coupling）**

你認為哪一種比較適合這種情況？

請說明你的理由。

我們先確認這個觀念，下一步才深入研究不同聯軸器的內部構造。


完全正確！B. 彈性聯軸器（Flexible Coupling）。👍

你的理由也正確：它可以補償一定程度的安裝偏差，減少兩根軸因為沒有完全對準而產生的額外負荷。

不過，我們再增加一個重要觀念：

**彈性聯軸器不是用來取代精密安裝的。**

即使使用彈性聯軸器，馬達軸與滾珠螺桿軸仍然需要盡可能對準。否則偏差太大，仍然可能造成振動、磨損或損壞。

# 下一個單元：爪型聯軸器（Jaw Coupling）

![Image](https://images.openai.com/static-rsc-4/q3W63DFtQ9_LQ1v_YDybxMcCIgUFpWdyJzaha5fVjiX6Ml0GQ7-tWpkxf65do8bWqb7mSv5RrGaSVkMXNaNqm5nlgo-rstUdHBChwQnKTxw6uEMbXpibNFwhsyd8MwWuelhjA3BzgvUoI8aW0OJi3eqidERezR_VW_6y7FYLui_Ce7nLjZ3KQB2Hd-ImJ68m?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/na1ZVQ8OpuOCN5bsMTLBbyU5CFBjiAePwKtqKJ5W6aepnTC716Jx4hLYxZXD0se2fD891VIxXctP1nhHkO9Lf7j3FM4DyI5gOBAQnhTeGS5gvNWIKRp3H18rJQYkbgMaL_2GvlaG8_scaPdH9r4oF-A11ieCRXyfzVEGVwxaervsZc2_iQzXh5VpHj5_vLlD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jCTPw6nLLYsejlSKpCZ2DyReH3rgESXhvWkHxNjC3JXKxNFQ0Q0WGnT3BEjco7CU18brY0c_5_wm8PhHYJhYIu_raEjgvDTzYUUSTs_3JCCeuD82g7zmKdPys9oGUdTNL0gQ-5iSrLhs092J5P1y7YXMyS6CwPBIkoWR6PF9cbXpgP8rlzZSSg1qBazrKd4V?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ldguV6n7pUIheCnJm7D4IxB4HV9BQw29D9IRfc44Neg9-RF2H2LHgkRGUryhoUIfRuYB4vZl82WyH2DtkHQ9EF1KwHuHKzGHX3eGhsfLMxKGy9BF0HLVaQsv_02Cj2QWWHqXof6Gu-9CoQqAwX_IAZo81ZrTUWMyFMVuhoJ0xU02iw8ALrI2lJrM8WZkvdrW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/57OybKZWUSKSOYVPo9wJS-VY15ggApgUCOZLSkyVU8UrHAAWtZNDUXek8FiKv-pAqTvYU8N5aGY0j88SrOHZA_naNRChtqjAz63kSZ_-zipmQqvblIxQmnPiJ0D375tXHsJULUmnWs3NG-fki4UTBVwZfFkKyTjZl1S-5yY-wdSQG06Wxd3IvGi0Byn27Xqb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6mQefK_E43wJyYHc7VRkMAPqLkFfoLZbPSb9F4w99YQLCEZzRtsxcPjxRx0R57NLCFRyhMx7_rGvjH4ixUj4jvml74z34cR8-xdSkOwwWd8q_ICKBb1HCKcCEvqgCFgGAKDQ5eiIhABoI_Wp_3II-1KOsPetP26a_-4ogoAAfnrVniIF1ikP0otc-vXjXdD7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ESoMnIjJINUjhEQTu9lpWSYGpde-Q9_R-7YnEN8Drra_d0tbq7Vl9Qrm00L4FrSMZd4fWn9FpQ7-eCl4La7RnIJ7i5tfJGKjBdyf9Jc2uukmjd5oqunAlZjz9ltuMsHerMiiXal2ZseyegX7wROQdx0UXJ847KJcgFqkUxm59XAq-7GcKNock6zjIziMGTqT?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HCgDce9sYcPLgK_zzQB6Chh0mQs977ftiYxTihRQOC7OLbtbIoxU2eIw4LoVV-RN07uZCN4lY-fK47zwjI-GQyAXVkYzvjAuOD1PlMLP7MqLUCUZj4SwbRc9lTa0dt8NTjQpBaAK898RtgJmdy2QgEGfz0l-JeONFEwOWVkvHMRJQN4tNUrlsfAalFf_yeuW?purpose=fullsize)

這是非常常見的一種彈性聯軸器。

## 一、它由三個主要部分組成

1. **第一個金屬爪型輪轂（Hub）**

   * 固定在馬達轉軸上。
2. **彈性體（Spider / Elastomer）**

   * 安裝在兩個金屬輪轂中間。
3. **第二個金屬爪型輪轂（Hub）**

   * 固定在滾珠螺桿軸上。

兩個金屬輪轂通常有凸出的爪齒，彈性體夾在它們之間。

## 二、它如何傳遞動力？

想像一下：

```text
馬達轉軸
    │
    ▼
┌────────┐
│ 金屬輪轂 │
└───┬────┘
    │
    ▼
┌────────┐
│ 彈性體  │  ← Spider
└───┬────┘
    │
    ▼
┌────────┐
│ 金屬輪轂 │
└────────┘
    │
    ▼
滾珠螺桿軸
```

實際上，兩個輪轂的爪齒會交錯排列，彈性體填在爪齒之間。

當馬達旋轉：

1. 馬達帶動第一個金屬輪轂旋轉。
2. 第一個輪轂的爪齒推動彈性體。
3. 彈性體再推動另一側的爪齒。
4. 第二個金屬輪轂跟著旋轉。
5. 滾珠螺桿因此旋轉。

所以，彈性體不只是墊在中間，它實際參與傳遞扭力。

## 三、為什麼中間要使用彈性體？

它有幾個作用：

* 補償少量軸心偏差。
* 緩衝部分衝擊。
* 降低部分振動傳遞。
* 傳遞馬達扭力。

但彈性體也有缺點，例如長時間使用可能老化、磨損，且受負載影響可能產生扭轉變形。

---

## 四、今天的理解練習

假設爪型聯軸器中間的彈性體（Spider）因為長時間使用而破裂。

但是：

* 馬達正常。
* 兩個金屬輪轂沒有損壞。
* 滾珠螺桿正常。

請問：

**馬達旋轉時，滾珠螺桿還能不能正常獲得旋轉動力？為什麼？**

請試著從「第一個金屬輪轂 → 彈性體 → 第二個金屬輪轂」的動力傳遞順序思考。


你的問題問得非常好！而且你已經開始從機械設計的角度思考：「既然馬達可以直接帶動滾珠螺桿，為什麼還需要聯軸器？」

答案是：**聯軸器不是所有馬達與滾珠螺桿組合中都絕對必要的元件，但在兩根獨立旋轉軸的常見設計中，它通常是必要的連接元件。**

我們分幾個部分說明。

## 一、先確認你上一題的回答

假設爪型聯軸器（Jaw Coupling）的彈性體（Spider）破裂：

你的回答是「滾珠螺桿還是可以獲得旋轉動力，但動力傳遞會變弱」。

這個方向有部分正確，但需要修正。

彈性體破裂後：

* 兩側金屬輪轂可能仍然接觸，因此在某些情況下仍能傳遞部分扭力。
* 也可能出現打滑、間歇性傳動或完全失去動力傳遞。
* 可能產生異音、振動及額外衝擊。

但**軸心偏差不一定會逐漸擴大**。是否發生偏移，還要看安裝結構、軸承支撐及其他零件的狀況。

因此，不能把彈性體破裂理解成只是動力變弱，它可能導致整個傳動系統失效。

---

## 二、沒有聯軸器，馬達可以直接連接滾珠螺桿嗎？

可以！但要看設計方式。

我們比較兩種結構。

### 方式 A：使用聯軸器

![Image](https://images.openai.com/static-rsc-4/_TSk5pYtJ54V1doBb-ULD91bWrBpUAYEJc1ZcVdAJiRpD1_Uzl0cdt3CQMAqvjA9W8MvElRlIN3FhPCuXbV1nJMolCb2WsTTSZFZ7MpLvuR5songW27La4Z8ONwgvh1TrHyB4Jv2rZFzdsucgZPGkhsAavP4HDa8sxjvuqiLE6m9pIrGz3m68nkvFb09lv5K?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3PzrrGO_qvRLp_Bu6gX8a-XlaZz7yDLbo2SUt_4ft18_TudVe9p0KI0yQyxoplxJEcp_GF5pIJm3Xxxd1BpdgUxwDKELopn1DJTLdzBfOeAkJdFCXfZNcsHfc7Oqm5eGJEepqDQj_LTdz9unsf4ajYNPaREdVn63IfdVaeDGuFTiGuVQ6NVPKMM-n0Z89-lc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3Rz6xKUiF8NcCzbqTB-YzGcDhEaQaBfZSl_ZpF22YEvi-yVkcDUl3lNyx7e08MDDjXBsHeT2O5JGo_pTfDmTLjOFK0r1_h4VYuW1LhRB9aXGdiKRgqeMnigVjgghjgvZUg7oH8sHeaVjH0Riq-B4fzibiQkZg7r5as21pqVJG7UbbkmounWn_XZ5cFLCbwfu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z9eHmNH8UZuhh1co6n6xFEdMUer9IZsAd_J_X7Z8kFyXFMkyI22PH96Zdy4B0sLyswTIfpkSphnuVWadTBsQphLVFLvC_iH5ugbmfidM0c5PbmGn1dk_oXTutVJc7nidnfOYhafwAsMWvT1_htMh2B2fLhtGMEqso3B5YT2X-eMoa--oWK6f4PMNt13bIwEw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LHPZNrf8p0Y0kc_iyVXXKvLzTE9zv3MCII0-8nqGTJnEqi42Bjw5pBK994Ek2Wu0W7Rj2oE8LlfqZ5YBtmgXP_GoZXTkt9hzCjMZ2uqubTltcxaqTVMb8HuOGNfxaMRIHRBvAkyPz3UDEnTxRTvPPsTZB2wXLDlmUa5qTpjOYxqCWHTEHN7tn6aBwniIovOH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-Nrw9WsWhSjgTdyxUDyMUa3TA-95vfjxoD3k1bhH5dnkCNhT_WhxbFfOPH5WfUNSsTRM3RP603K882rGMZmzQOM2glCkizimNa3HupZGUpdp6JDNv90E7WhGhkmtYzfh8xr3OtXlif2A7jn2u-L_s5IHUHTV8ZVJXS5CJgMMMiNlLagUE3MxDB0Tog8AHQUg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hmbf5KvhocQFtwGGgwOV5yWkSFUWiWLtTicGRkRkquqzyMsU1CddqRNFM3G423pCJOodw-qMFtH4N5gjbXlZb_Cdd4Ub_FtnvNMigwYRQIdWP-7i_Y4bXVhJBU6zUd5mMbcbzhc-1PWydsLa8pNI5lhDq9Qji0Wc5NXD0zkORPpPVNlNu6ZOJnPI3OyQS4ol?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zped5003FlogGo60F5eGDTx4H6pjkZlpehiunZV7ZGbV0dsQmsEU9Mt1vVGgTxnneqDqGIyPo-HVh9Wz6ILtP9_cS146F0Hc6dOS9qai7pqsM0vfc9lpmZVvGXz7r6LGZ_xBOZNpHYUJx3nCjB2uR33Y7OZ-c_tbOJrUJOyDlPGtPahgvSNUHsiqJdOjxSmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Pl5FiuZadxX2RljfkrdKCweE-xlTAyroI7Xfk91terDS6uQGHPpy9Nm7kXQkQVCk8BpaHG3OlAIf1vg7uRxE1DBTR7wcmzqvkqQaGO8x1Sgc2g7rIPZRuzfordzZivDjUUbwGZzVR2pbCfv8aEMInG0hiSIAbN_DM7twcwLrqYOJ4lhucevsUTOTaLPGwEOI?purpose=fullsize)

結構：

```text
馬達
  │
  ▼
馬達轉軸
  │
  ▼
聯軸器
  │
  ▼
滾珠螺桿軸
  │
  ▼
滾珠螺帽
```

這是很常見的設計。

優點：

* 馬達與螺桿可以分開選擇。
* 更換馬達比較方便。
* 可以補償一定程度的安裝偏差。
* 可以使用不同類型的聯軸器調整傳動特性。

### 方式 B：馬達直接連接螺桿

![Image](https://images.openai.com/static-rsc-4/84ZaYa43lUKafxJZ1Hy38OnZABYefj9LLSMuws3odM32XQASE5bEDIUTIJYM7HNPQu6u_mBFakOxdzxLjuxeV34eb90n4G8eCZz10fHbEDX6-03rosCU4llkyzmJOSZX6JuETBgIYwoNqNyGlg9S9lizxwKLUDmplyKbHiwR73qBCUp7kJGtdK5fSMJhKHAc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/YV9K6BbD70HiFqZZjZYtz85DNK-uwQ5PPxEdFfY49THUYEIgLOK7r8Ahs-LNfjjL-JBeC9F5WArZdcXJCDdrhVaqZVSUBs-XUAZ4apRpSpl6531PJpEHLMRTGPduEHxylj0buV5KKHj_OTRCdrdy9snArY8e3jBTHYsznOBwwfa2CJ5T3-Lcq2L8LqY10zd-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/evP6sbQgusJ0kAuM-sprPrhORAzR5-qsplxgMG1duruyItw3VZtLRSno8LsWuZFUfA54BBU0QIwM_ixtOO-EJfF6pUqvLPOJ1mkaexxt9V9GiGXf3qskzZtkmMLcdsgimv0Xr-A6TDNVs1TrXnL8XhUR2dKS3treU9RiyMuFQ2GlpQrMwvM6zY0tkYaKT5hX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Jd1icxY_bYbzEwOfNbXUZ7scyTjcB4Jgf_h-XAP6igF3Rj7q0LGj8yobrLyNyfaoUmrgxk0sGQ2jz7jZfPNGlPb3ieAP3c113qe-DX6xFkIyjU_efLBQM97Ev_DsfjE_vvpIiP1HY9rErTqIHPSXma7KUszxLvZzUKNO0z2slmqWONH7fDzW0W9MOG4ExoLH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3UeuQYV5POQIGSeFkekVsDQtUeMvh03Q36CPeKwZ8ES16Cc5u5zdDvLttxst-bx_Dj36Iq8aDqtjId_GoFtfNzzj1ZJ9miweiGcilrAi4Eq3Xyqv6os2GQxVxq560kJzX9GMCqTpxW4B9IDuT6LeqodlkNBONI0cDr5v5Em2nWokyOjoyw-Wje1injr2kF1B?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rogaNBwZ22o3V4Xzsm4Y0dOPkoF4L6ENpqiaw6wDIhvFP1qAttEw5jLosndXoRuryoULbJ4t06YW6rWfsxxkongPT7nPK0dMWuerRR5OtVWEd4rzUd1_v06Ov5rTP85l7KG5Hyf5XUCBFgiGG7h-NyYG3KRqGjFrHeVkyDxWYO1z0wF8AKXdEVWIVL2AtbGc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Eqr0gtbJ6u0YX2J6o9i9Mh6L9bImRw8a_o6AJlUtO0yu4-DzdECREvPIzFZHbcWDHIvVz0f8La66pMrLN7SO0Ral5WzrA3zhsh6fZXJ5qlLiUYzZHZz9XxiFQN9eYuyY77XnWBL7VuZWiwsmgjX_3AdN_SARhSWzcV_eL9p2KwD9I6B4UUUv9v3sbQNKGBec?purpose=fullsize)

有些設備會採用整合式設計。

例如：

* 馬達轉子直接連接螺桿。
* 馬達與螺桿採用一體化軸結構。
* 螺桿軸直接設計成馬達的輸出軸。

這種情況就不一定需要獨立的聯軸器。

但是，不能只是把兩根普通軸硬接在一起就算完成。

仍然需要考慮：

* 軸心對準。
* 扭力傳遞。
* 軸向定位。
* 徑向支撐。
* 軸承承受的負荷。

---

## 三、為什麼有些設備選擇聯軸器？

想像你有兩個獨立零件：

一顆伺服馬達，以及一支滾珠螺桿。

如果其中一個零件損壞：

使用聯軸器的設計，可以比較容易分別拆卸及更換。

這也是機械設備採用模組化設計的原因之一。

---

## 四、今天的小考

我們來確認你是否理解兩種設計的差異。

假設有兩台電動滑台：

**A 滑台：**
馬達 → 聯軸器 → 滾珠螺桿

**B 滑台：**
馬達與滾珠螺桿採用整合式直接連接設計，沒有獨立聯軸器。

請問：

1. A 和 B 是否都能讓滾珠螺桿旋轉？
2. 如果 A 的聯軸器損壞，是否可以只更換聯軸器，而不更換整顆馬達？
3. B 沒有獨立聯軸器，是否代表它不需要考慮馬達與螺桿之間的軸心對準？

我們一題一題確認，讓你真正理解機械設計，而不是只記住零件名稱。


三題全部正確！很好，這次你已經理解聯軸器在整個傳動系統中的角色了。👍

我們逐題確認。

### 第 1 題：A 和 B 都能讓滾珠螺桿旋轉嗎？

你的答案：是。

**正確。**

* A：透過聯軸器傳遞旋轉動力。
* B：透過整合式直接連接傳遞旋轉動力。

兩種設計都能達到相同的目的。

### 第 2 題：A 的聯軸器損壞，可以只更換聯軸器嗎？

你的答案：是。

**正確。**

因為聯軸器是獨立元件，可以拆卸及更換，不一定需要更換馬達。

當然，還是要確認馬達軸與螺桿軸沒有受到損壞。

### 第 3 題：B 沒有獨立聯軸器，還需要考慮軸心對準嗎？

你的答案：仍然要施作軸心對準。

**完全正確！**

這是非常重要的觀念：

沒有聯軸器，不代表沒有軸心對準的要求。

任何旋轉傳動結構，都必須考慮旋轉軸的同心度、安裝精度及支撐方式。

---

# 下一個單元：不同種類的彈性聯軸器

目前我們已經認識：

* 聯軸器的用途。
* 馬達轉軸與螺桿軸的連接方式。
* 剛性聯軸器與彈性聯軸器的差異。
* 爪型聯軸器的基本結構。
* 直接連接與使用聯軸器的差異。

接下來要認識另外兩種對精密運動設備很重要的聯軸器。

## 1. 膜片式聯軸器（Disc Coupling）

![Image](https://images.openai.com/static-rsc-4/pEvb5V4pOyfQvSvMCPSPFBdyogmCb8UfbKYfGAmTIabeQa6_jIXTztFwdO6pZOkDDEhrhGGfKmPxNAPSNwhAGZWzRzcAE0h07t7Pnu8ufOCdzLV5QCZXzngx_xHB5tfy1bU-xQzNCj9wkW9M71rPrCkqMu4ZPBqzXtqwpUaO_DJprhHV1rSEk2-UI24iE1ev?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-yd9SROdUtk7r-TdNwZsSF7rG1kXmSohNs7qgX9e3wBMQRs_GCDxzolBqzmjylfXVdd1dh0L1TayL-YLSaQuW_TsRU2ZyDdwgIQNE1pc9qtsHIIqnoLp_a8Ic4Te5kHZqEzyFs8Wcbt_me3o0oLjZ8RSTu4LEc4uD7rJ0tHoek2W-3gjzU7eoNaVvPxJtIJg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ra_h0NzxB1BeP59WK-FtVnkcx8tPllV3QmT2GOREEQKtGE7jf2tu3AZFoXR53BIAaQliQne-eQ4EIf6cLlNbOPgsyxFscOjwbYMGHGHE1MB8deku_JqspDEfU7eMkdQQ00hNV-Y9K6WaqEvM2Wk0kbJuqoheMbvx_-hmXRExlyIPOFPaR8A9iTDSFsg1BfmE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/S-hB9lqz5DDfCWSLNo3d4E4lJ2-D1Nl1Qjs1gSVH3TUbYskCVwvwoPNb1lPwQ2oMPrXNJhXhWiJZS1UjZZXBECxYJQn2HSdSSOxaaIKlrhjk2Jt86x8KBGrFerFzz1_zHtzzfEwZ-Nv0VjwmJa2ERXxR8Wm0fQLO4cJv3jh-mJq7rcLAFlN9Z4lT4_6teSqc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4aeITywesGqe3ccvr3S5Tdbg19ONOkP-HEnFs8W52ZnYyDgXqd7yxjSXcdDbP97aRCpuXzbNPz_hm4-UP-yqlp_tZaaNkeMVi7cYQxedD4pKhwymgQRPebC75hzIT6kddbn_jHNTeOGBgJX4D5ToImcZmFKquRU4yY0xGwz0olmZArLGIp4mvGcl44iRi25Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9owyDW478o7VEs9TKwEAaRN01o_A29IYtHtygEGCeoCFCy8oAJ-j--3C5V7VwoK0lSfGGLVXt6zftO9I-FSEG7-7Zxf0r5TPNlW4zsLVItSgy0nzdHyJqZaLdAAcR0OeFzPo4fpTHOZj_ixOJ0lEQwCMNHa-ErHXxFU560forvW3OwWuKNKP1RivZthxdoim?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NymbD-frVmbcLgWtuzuwb3qRnmA_tKP2vDIU0e7zbrEB_so79nKJaSnNZZn3CChdcblnRJ9eDZ4klRYs1PnPNkqrUGqNKXIkltkeem4iKIHaZxUfPTKL2XBlcdWLyMCZu0RL4CJlnFTxWuIUmLNu1m7XyIuCBcdWgHV45F6ji6zK_YsBAryskao1O13TtI6q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/15OiNrbYvM-8OST47NDuG_pRQOaRrNqF5OiZzbHB80nbHDX2hU9gQo1bZpUdmWw7c0y_v1JmjPdqIZeYuQaRa7hHuqGTRCV8109ERgsrwRTaeaJi4gYYy6nFXyFZWfmBqMau6FfmTz22hSnIsMWAqwQkMf2mPmeZMMIa-QjrZkltWNkWg-UAHoUetelAH7Qd?purpose=fullsize)

它利用金屬薄片（Disc）傳遞扭力。

主要特點：

* 使用金屬薄片的彈性變形補償偏差。
* 扭轉剛性通常較高。
* 不使用爪型聯軸器那種橡膠彈性體。
* 常用於需要精密傳動的設備。

## 2. 波紋管聯軸器（Bellows Coupling）

![Image](https://images.openai.com/static-rsc-4/ldHWkcCbZcLSH8iaiLm_JBgykb8B_ltp3BphOKRVaJAqcdus6NefftUh4oc4qIOtqK6YIArbraNrm1E5KbBKPfvnTVCradeZaLdbT0Oe1Yoeyh62Egp9GNUror26sfH2QaUoOUvY1m1a9PAadKwBQJwBSpqFRDs8an1TZNPhlkxs9XUCTNG5gmp6Ko460t3s?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2gwGlbp3oAlMFYnWSUOd1VpimHBRkV2vixoDLVJCtrhMdMJ5kopYYz8iJEkGnTyENlfoUtmB_1W9Zad8YhuGWUXvUGH21PIIKytPy_pmRXzBAzkpkMTO23YKKT070KllG78O8TwyWsm3ShBmBMt7doj-QXUBfoePF8dd4PvGFyj5D1HQGXrzJyRfdNKf-vUD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dZmB01bjqAIzgiY3p1ZfWeNdd705HFLOMSvMtnbPlyr8RgT64TQlEbI7TBJDQUBBjn1MplYBRdEBiaGWcxuFD_2GDFhUZBxlm8Z86aae2QXM4g0Qf09yDt5yklC5l2q6QjBcGiWx__YI1VSKGZs7olMLrGkgfoRhExM9Vgy9jtVmoGuGw1y9IB2KiF_ZJFyl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jHroF_i0QYsW3anb2_m99zqJ5v4PXXBiqp1g-rh9RCYqWNV11sb3rcQfICqeCkhwg80hDYk63Kyyq2gw_pzwCeE4WIdqNGTflmsdRS5TgBYEjZQbJuZA_efQzsrqoKAZks10b8yGMBrxSUayGn7lZPs54lEEaI3bC8v7gp1qon4PNdQxw8fC7ifkX_7cNvG6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rAqS8PrGK0r_9IKd4qvg_DE6mUgc-RsseCkunUfdI6mI4mJVeBrNHDeHhx4DFiZ7KMtUp4dtP1Fln61z-jt6FquONHCHsDdtSDVJFAcuUo3Fcs9qg2BkUoDtmK5pyIOa0uIzDsi54zXZCFRNZ4VwNjrEtmP3ZNEUyccbT3uPjSd6qmhTjhmhekgqHDbh8VQq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ymCljsacPFWrpJYdHAjGtRJLvk3Na982EFsCvbKG45DBrcNj4WtD8ah1zRbKVMHvBcmXjVdYmwXa-lrv5NqKGvOM63PtoLiZuFxcjJsvXHTyNCI-2WPBBcTLeSvoUbmrM1pcxj1bXqJoYPnmHvT51mLbX1j7BWZ0Ubv7v93yTGwJvqtTf5P-IZn9P8P46AP9?purpose=fullsize)

它的中間部分有像手風琴一樣的金屬波紋管。

主要特點：

* 可以透過波紋結構產生彈性變形。
* 通常具有高扭轉剛性。
* 適合精密定位及傳動。
* 可以補償一定程度的軸向、角向及平行偏差，具體能力依型號而定。

---

## 3. 三種聯軸器的差異

| 項目      | 爪型        | 膜片式  | 波紋管   |
| ------- | --------- | ---- | ----- |
| 傳動材料    | 金屬＋彈性體    | 金屬薄片 | 金屬波紋管 |
| 緩衝能力    | 較好        | 較低   | 較低    |
| 扭轉剛性    | 中等，依彈性體而定 | 高    | 高     |
| 精密定位應用  | 視需求       | 常見   | 常見    |
| 彈性體老化問題 | 有         | 無    | 無     |

這裡先提醒：表格是一般產品的特性比較，不代表所有廠牌、型號都完全相同。

---

### 今天的練習

假設有一台精密光學對位設備：

它需要馬達帶動滾珠螺桿，讓平台精確移動到指定位置。

設計工程師希望：

* 旋轉傳動具有較高的扭轉剛性。
* 避免使用容易老化的橡膠彈性體。
* 適合精密定位。

你認為膜片式聯軸器與波紋管聯軸器，哪一種或哪兩種比較符合需求？

不需要急著選一個唯一答案，請說明你的理由。


可以，而且我認為這個方向很適合把你目前學過的滾珠螺桿、直線滑軌、聯軸器，逐步延伸到 GMT 真正的精密光學對位設備。

不過，我想先把課程目標訂清楚：**16 小時內，我們可以完整建立 GMT 產品體系的認識，深入理解與精密運動、光學耦合相關的核心產品與技術，最後進入矽光子、光纖耦合、雷射及 Camera 對位。** 但如果要求每一個產品型號、每一種規格都深入到設計和選型層級，16 小時是不夠的。因此，我會採取「所有產品系列都認識，核心產品深入學習」的方式。

# 一、先認識 GMT 的完整產品範圍

我查看了你提供的 GMT 官方產品網站。

[GMT 官方產品總覽](https://www.gmtglobalinc.com/product?utm_source=chatgpt.com)

目前網站列出的主要產品系列包括：

| 類別          | 主要產品               | 課程安排   |
| ----------- | ------------------ | ------ |
| 半導體、光通訊解決方案 | 六軸平台、多軸精密對位、光纖耦合設備 | 深入學習   |
| 直驅模組系列      | 直驅平台、多軸運動模組        | 深入學習   |
| 壓電模組系列      | 壓電平台、壓電控制器         | 深入學習   |
| 電動滑台        | X、Y、Z 軸及多軸滑台       | 深入學習   |
| 手動滑台        | 交叉滾柱、線性滾珠、旋轉、角度平台  | 認識與比較  |
| 精密對位平台      | 多自由度精密對位機構         | 深入學習   |
| 傳動元件        | 滾珠螺桿、直線滑軌、聯軸器、支撐座  | 深入學習   |
| 自動化模組       | 電缸、電動夾爪、中空旋轉平台     | 認識與比較  |
| 光纖／光學系列     | 光學平台、顯微鏡平台、中空平台    | 深入學習   |
| 馬達／驅動器      | 步進馬達、伺服馬達、驅動器      | 深入學習   |
| 刀桿燒結機       | 感應加熱與刀桿裝卸設備        | 認識基本原理 |
| 模具類         | 導柱、導套、沖頭、精密模具零件    | 認識基本原理 |

以上分類以 GMT 官方產品資訊為基礎。([gmtglobalinc.com][1])

我也會把網站上的產品型錄當作課程的重要教材來源，而不是只依照產品名稱自行推測功能。

**GMT 官方產品型錄：** [產品型錄下載](https://www.gmtglobalinc.com/download?utm_source=chatgpt.com)

---

# 二、第一階段：先建立所有產品共通的機械基礎

這是我認為最重要的部分。

你不能只知道「滾珠螺桿是什麼」、「直線滑軌是什麼」，而是要知道：

**當這些零件組合成一台精密設備時，動力如何傳遞？各零件如何配合？平台為什麼能夠精準移動？**

我們會先建立以下知識。

## 1. 基本機械結構

![Image](https://images.openai.com/static-rsc-4/Cb0-4FBQFXdnwDnf52EheeLR0jByfT2Gy09cYoGoMDioUe2ftpaHu32Ziuwt_M50Nel4A7U8Q7HU8ml8VlGOZ6FmkwhUdZ0XhkLPnQqjI3dyCdidbPEWlOK0VKbWXxDkg3BOdoFdjKf3_LDfFQnQqNnwHs1r6esOJSeDttDVE2SiaqW4wtA2ugPkEXC3MYLA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kb4252ETSg5LEkuO1IbHGAcv6NUpKfSmueRvtB_dUyG6JqxGKre_zEYMQFR3MAzKSXCOACgY2Z0SRbq-QOINuzamSCykD5k4NtoN8RNRc9TJuDoy4zIMcee_oK4vEazRI8ispvhttLc78A_pZwQNPSt3JYwZUVlkLOzxmve4rrqkJiovlIA05Dvfcrr8j_qc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UMIruAApQAy8we12PwgBf8Fw5LBjsQI0Az8MJYFtmIqEWP2sPuVniHwxCXiURAaWte4usd2x8McxFsnu_i3-L9A8A78Tnc4NxJ9BtMQ1cZm9Fk8DQepO17Y92SYGFkDmBiTRBvOkWf5KfVIgaEd9MNevVMq6AkdXBsGEHgFTYxI9r8ZyE84Dt29p7m89hVLR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/M5pYr_9-F9WznwVhzpNN4apzJkFOiEZ6STo0JTU-PfqliTfSNxDYLgVzZor-wGsHAbWim51MIeskAws1_1N4myDC55QdlnLpBCaiX_9F_h_ut4Uxise6CYtRdAh-LenGQjAMHjWmXQhtUouD5zCjiuOgbpusE2_END822PRlhs97DLeOJRfDwkLgnVe1yQAx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-HRXayU8Zbiw4AHb0IeRZVd7vnQ7yZXS0mx_BXm2pPDLSV8bsc5cy-lXiYUmz5QEPVRQUP3YXL7gg4YSW73XHZk3k7FmAFOIopTRAIm1L56JdreY-UdJG84W_itBdFkVqFIP5DoXnEzYRrrXZLq9wSkndFj9VN_YDEj1Trp1fsjOzR9z39m4318itP1X08Is?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lFppUzOmb7btaz7jFxoGB055RUxhsSj1BJy7ZN0ABq7eGamhQ8gU5YgfkrrS1aMLHHFeJkzQ8Cat0dZL8qdxszEutAvNfnMRjtcavRupKvKvmtxbTgboitLRGh0CHCZFjthUsZb11dMSR9wLhfjtdWTZVd8FVnnP9seNo9kNqBKriT5drEWCIE1IleL1Zifw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/JjNn1_2ivw379zgrNl9q7c2VnQQbYWpF1Cw-zMDnJLW9pcLnhfp5Bnmkrf4ENB9Q9KnZrJotJyKIr-pcM43Gtla50qC7JGwCfosrl4Qg-D_SifAVR3tiOqah9ZuGj6XP5vlAzeNU1wYKNzYG0iK9im-TVedIcKg7E5zFWfFoWjB5awYIoD2VypuXoDT3DyVV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ivP2SZitFn7idJmQXdQmmvJKqhMrc8S_EHc_uUPIW0EUBMkLif12vJVlozZ8S5l8wC4wt01uBDx1f1KD3KYYC1dvVznJwr6z0l7TkIxoFJJWNmPyQjeUkX9A8E77bOrjmC0ULei36NkYjP6s3EE3wBaRDN53KdlImr-cSXqLvCn3X9-8RHFgI2a1Js11LujQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/d9603dxsLjmNRYw4sQ4sbVj2JrZlLT7clJ0wcLTcqwrpRdWB_CJze_CZmO333hdOxrqTaJCnB7rBil0ADbv8JORKIL0VUzjn26PIlCLbyIJUqBpu8B85Oqdh2PUAepaAQnGCx7elYpQzS8vRcZEYA_Q1zjsVrX6-P5m3BjMbEYlIc5woDsktwgWL1I1EIU1-?purpose=fullsize)

| 元件   | 英文                 | 必須理解的內容       |
| ---- | ------------------ | ------------- |
| 基座   | Base               | 支撐整個機構        |
| 滑台平台 | Moving Table       | 承載工件並移動       |
| 滾珠螺桿 | Ball Screw         | 將旋轉轉換為直線運動    |
| 滾珠螺帽 | Ball Nut           | 配合螺桿產生直線位移    |
| 直線滑軌 | Linear Guide       | 限制移動方向並承受負載   |
| 滑塊   | Linear Guide Block | 支撐移動平台        |
| 聯軸器  | Coupling           | 傳遞馬達扭力        |
| 軸承   | Bearing            | 支撐旋轉軸         |
| 馬達   | Motor              | 提供動力          |
| 驅動器  | Motor Driver       | 控制馬達          |
| 編碼器  | Encoder            | 回報位置或旋轉資訊     |
| 控制器  | Motion Controller  | 發出運動命令        |
| 感測器  | Sensor             | 偵測位置、狀態或其他物理量 |

我們會逐一介紹每個元件的外觀、內部構造、作用、工作原理，以及它在整台設備中的位置。

每個元件也會另外搜尋網路上的技術資料，參考專業廠商或技術文件，例如 PI、OMRON、THK 等，而不是只引用 GMT 自己的產品介紹。

## 2. 機械組合後的作動方式

例如，一台普通的電動精密滑台：

```text
           馬達
             │
             ▼
           聯軸器
             │
             ▼
        滾珠螺桿旋轉
             │
             ▼
        滾珠螺帽直線移動
             │
             ▼
          滑台平台
             │
             ▼
            工件
```

另外：

```text
基座（固定不動）
  │
  ├── 直線滑軌（固定不動）
  │       │
  │       └── 滑塊（跟著平台移動）
  │
  └── 滾珠螺桿（旋轉）
          │
          └── 滾珠螺帽（跟著平台移動）
```

這裡我們要建立一個重要觀念：

* 滾珠螺桿負責提供推動力量。
* 直線滑軌負責支撐與導引。
* 馬達負責提供旋轉動力。
* 編碼器負責提供位置或旋轉回授。
* 驅動器負責控制馬達。
* 控制器負責決定整個運動流程。

我們還會學習當其中一個零件故障時，整台設備會發生什麼事。

---

# 三、16 小時完整課程安排

我建議分成 **8 個單元，每個單元 2 小時**。

| 單元 |   時數 | 教學主題            | 最終學習目標                |
| -- | ---: | --------------- | --------------------- |
| 1  | 2 小時 | 精密機械基礎元件        | 理解所有機械元件的構造與基本作動      |
| 2  | 2 小時 | 傳動機構            | 理解螺桿、滑軌、聯軸器、軸承及其他傳動方式 |
| 3  | 2 小時 | 馬達、驅動器與控制       | 理解步進、伺服、編碼器及位置控制      |
| 4  | 2 小時 | GMT 電動與手動滑台     | 認識不同滑台結構、軸向配置及精度差異    |
| 5  | 2 小時 | 直驅、壓電及自動化模組     | 理解高精度微小位移及多軸運動        |
| 6  | 2 小時 | 光學元件、雷射與 Camera | 理解光路、鏡頭、光源、影像及視覺定位    |
| 7  | 2 小時 | 矽光子與光纖耦合        | 理解光纖與晶片之間的精密對位        |
| 8  | 2 小時 | GMT 精密對位設備整合    | 理解整台設備如何完成自動化對位與耦合    |

另外，GMT 的刀桿燒結機及模具零件也會列入產品介紹範圍，但不會占用太多時間，因為它們不是你目前最終學習目標的核心。

---

# 四、最後要學會的：矽光子、光耦合、雷射與 Camera 對位

這部分會是整套課程的重點。

GMT 官方網站已經列出與這些技術直接相關的產品及應用。

例如：

* 高精度六軸平台
* 光纖陣列單元（FAU）對位設備
* 雷射二極體雙透鏡主動對位設備
* 多器件耦合設備
* 自動化光纖耦合設備

([gmtglobalinc.com][2])

## 1. 矽光子（Silicon Photonics）

![Image](https://images.openai.com/static-rsc-4/W_4oUaQDQX5vsqdpJOryZxGR19qqPZ1lRF4p6PktiXtOOf76BjPK0AIrdYT4kFhwsUQrjyakqrsGp4OFbJwGSnq74ybL-S9aucn7ITCOzYXpOE_4F0NIr0cK_nlx9K2YCSkaZkeQJ68G_jjSKuEfqkkZeE5tNNa7FoX6inNeplg1hyvdAtzAz8r-CmTfaiiR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5iKCNolLInXdNzvUv1eCo2_ezkGXwEGlqO9MeSLKwO_zBzi-8_o0YrbdhddV4lfrRJemsrjuMb1ZFMRxzrBK0sh4w50yJ48yT8F_1riVUGzC42nmJrDg7jL6oxj5H9ngTNCmay_0Spo-QhGNz2-E8OqAKXg_mxGlwb36JdDIXb-2ViprJAJdS0iq17y0VKgR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z12fTH7myHm8g8S9LahdRkO4XhUdCs9Xnp4zMW14X9fL6SQvqr_Qy-Aw6SXrz2rBuWv0m9l-6J0xSTb1KED1MuTjJdrpWTof3ATcfSphQXx0FARJPs8sgGqp9ZfviBoxqPduzhlT3hfRit20YOU2559AKixB3WIqYnsXnL_cESrl4NIa7MczT7Z9k-WcP61k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zsgf55x465eP0YqFs56e-2_yxktW_i6gkFjFan5Mzy3Qb1x123h1PACh2WS8OZAzyBsPF6BZoDes6UuDHk_qB66qMvG_9jTQ0eQQzwe-aOeL-uweSPTon7at6i3y7N17MKjIPy5YCZY0rTQgNfIg3apF-T33KAvIHJddvm2tOUG5hhr6Keco6m9tbjYQSnmz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HM0OJ_xsfuZYEnP3BXvetCdmd93Tz_r1-xgXdH_rwHsQjx0y0EE-nAH49AteWVNMk4iD3ah2Kh5PDU8eaerZRywgMUo1IpE8-5EwHh6APGA7paP7QCKpHgaje3_SdJIvyC1yvictt2MP3zP9ifSQS6gs5EVFyoVdLEp8q4R6Ijfjwf-sIEw0HtJA-ua6Qzf8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qs29DxuKYy1LhYHbJSsTK_pdtLG_rDtnaH9fezCuhduUTbn9yHn5ECgNNM-geU0aa1zQXX2UJNgAiL9_Pi2XHLtDneIBDs5KEBnNHL53IROq0j7i5RFfQPKZWeoAPfpBftWUK2XGje1FhlKQPoOQb3MlNqe46UAjnXFS8nAIEwx5GjMCBbWdMme54K57lvIy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GLrTh96z580dUvM7iPxqtfO3mVGzqE4fdL4hpgfQhmgqfXmo3E1hRyZSIHWH9eA9PTL3tekveDwgLAsrOH1pBuCK2K-1M9ULLCvGMEpxAWGO5ty3Io4QO0tgdUudR2vZRtKxc7twuI5bYyqgWj9e0YQTJOBDxvkCxxULKSz5IR39bqmJZfc19pfjMdHPuc99?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8R4utd3OJ8EnEz862bK6CyAbcUbKDeS_-Qy_l_uAINTSiIPrk33BwA4izjPSzybg6kkqavDGjnbrM5CidqCzl1EtgVlZjua8QBVyZnXi1fhfdU2f0wGJ_BBLV8X6KKUYIL-2o9EW4NKGXRx9BnyCDogsa2HIXdGoWJyRDJsrvx25krHB4IHjatfdpzAmro2B?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_vp558jwmq-24NqAn4dle0a_FAm4b1l8eSjtXYru7SUVnvxXb6y99Ej3VTUpYwa0xy1NGbz_tN8rBI5-N-SFspOQSHctfzVEhoST4zkfKsaZP_nj1qhL0dUn-QvsJP-15Io58cVgYEE83DCBlVMVjSDkqxwtZMdObr0qOz4MYEruQqciaO7kOtF19_TMzFAC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AErx7xD4GHpWkweQorgHHoiKUjlQ5BbPyotjMVoGwf2JCwkN9IRTvlIdTF0Dgovro5emS_zk2XNv0s2btWBc-RSo3YcbQGTmLJiyZHIIgJUfZtoF0epKrJP_pLxdebaMR-ovWxL8R4vfRofjgOlVDR6xrybGmbQp_6Dx1ph3o4w5lFq9VAEfCM_WqiaygIFh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3AuiefW3YSrbe0_kA6aXn8zLDrm-2NM8kZ4E7gXBUCcAyVCfz0spHwDoiTs0I4uhz1jI0KXJ-ECCVy-cE6wFaJPhy238ecXiGgXEwX2VCYflWk1Ii3cRAZLENPtTOfrdNSTNs_trOvEKRWQ9eu3c8BfsVPqqhFKNEMRcTUb5pSFx9rYIGhabHbBVCflTIpGm?purpose=fullsize)

你需要理解：

* 矽光子是什麼？
* 為什麼晶片除了電子訊號，也需要光訊號？
* 光波導（Optical Waveguide）是什麼？
* 光如何在晶片內傳輸？
* 光纖如何把外部光訊號導入晶片？
* 為什麼光纖與晶片之間需要極高精度的對位？

## 2. 光耦合（Optical Coupling）

![Image](https://images.openai.com/static-rsc-4/WIA3-mvkv6qIRLAYxd719dYoI0p_XGNWdbqAB_loCO7PMaCrvjICkxNNMVI2iuiLgAuk96O7doA1nnB2NX_bwPSx8jeAwkA_41LuzMu1oiKlWAyjNEH2UJujMX8CDeSXkfohuCamK483OhfKx6mRX7iAH4_3Np5dXiQUuGlRn4vsg-Sh83j_GXARPx-QZm41?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bwWO4r1FCWmI39LzxC7fmskcd3rSJaodeczMr0X8nR9eqG6jeDMJnvDhVWL190XjTHL0GPGTvuyp_0dVzvv6q3dqjZYoo3JNCN7qIU07-TPUpJR2ZpLf7R5GQhvdJIH2fEOz8WoRBu1o0TAjWDW_mIYCYXw0-5-h3Gpu3Lx-y_eRCob_2x6doXuPlVQKflJO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z6U-ESoQ7qiA1uoVcHCAsVS_yejJX3a2o2mrtTwtgIR5YWqAseV_QHAjS6ds-SGzWkD16y1wJK3CVod_IlGAggdUl79P9nl9bxvA6N0rYsz7dYv9zrfENkPsOCmgQXiGAY3Ymp4yc0kYYt5Q4kg4a0z_ZVnN4c-HfIB-QXZ_W0rdBm9AfiABqMNlwEsXGquj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/A0NJeChrjIQo54IP5MW6GSKyhXLMHPW4QUdaI-TNBQ6R0MDCKs7xXyakJxN4bAYXZBizEXm6dvYu4La1qlEua9VbN2sDbOecArusSMhkJwGWtA3ohbSwqAWvzw2V3fWyxXhenPs8FBuIqV1sMrCcAc7Cx4lJ98eJdUjwU8QZBGe3mcfrdfO-2cgG-ocksYN2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/it-c_KPPbn-CB009hIJa2goxMCz7MRdiPRJ-BmtWJEWcPb0dbhNJlRNEl2k90q7A3ixDYfeMvS1EixVwAm45QXtBeDskrZgAWiq-Bh0QD0sYyOHazqS93BazYplZglVy2bjBWecGvY4gDAEiuLx_JVD7xMliJIlMis2quBjJUffU5aOXEK0NX2fW8aw-nfWc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WSJhjVhz3n21ZzlrqEXYA0aUsvbtzhG6qkOxretnEfzYaIYBXFUhd5qZJDAs2QHbo7koG49QhuW0wYM0w3m_zl26pMvc1W7F0H3sf1QKJ9e8zCu-UuHqksC4RKWgGBYrE8-4tU_owbTEhKxAHmORe506VQD68-QeXAjDK4JT_8ky_CQxDiQm9N-71w5Sx8YD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5B2Xm3Ulzp0Yz-8eSiCVPVT9o6zZdSckRK5q_nlOeyWrJvwKxy9w1ArcUdPtw8P0FLvBxsttxembxTlvbTenSYHFoZKCM9Q5HeI510vfNk7iBCd1cE1XYUxtkJOlNwnGOWbUiQ4NCpRl9m30f8m2r-cGSQSn_mLklvccDgSeYYeNUy4_M7yglcUjeWqpj81H?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/YxTBuwYRVxbxCibQ3Swp9VVfuWZHNH4k_oWkoR4clGp7GICxKgX-gm-aHqN6GEnA2eRJYpFneFqHAq5M0_NQ8QrkMvEmvmpsHVcvH0nNyUpksHrwSw6u8JnwTweYdn_ez-bTdA-pU_UOMRXhDlASSsWehOVkSMf4SzTMm1upWoXpMMKUTlPvaZJhjsmi-33b?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qGHBJJvfFOH9J83Y99eKR-q4-Lxe8Ic76J1xXW-ubNonibmc_IN3HwbjbpZrbFTxtL57oMnEmEGDISPd9iAIoXl7wtbUNVTc2-R-63vBvLkQ9CvcPph_VvN8K6ch8cLYkqYLGUaPSSObdfDiSkoiT7UVifqi-DrKt6XkIh_ehzbfUyAgFT7ujzpEucCHugYw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/aNxe6yhAarrmlS5kLUv65Bclwv5Nc0AbGNAG6XXv7HLoHH6nyQ2WNKnwG-vvGCf0WG3I1Rh_jqziCTkCbndYHvt_ss7Uw3LMamd6x51u2M5ZjF4pv3ZcMyFIPhMgXsXhy6YOwkKG3To2v5Et378WvYesMW-gaKvzS5SmChm36JZVmdYojxEfB0Bcr_MljkHR?purpose=fullsize)

我們會學習：

* 光纖與光波導之間的關係。
* 光纖核心（Core）與包層（Cladding）。
* 光訊號如何進入光波導。
* 耦合效率（Coupling Efficiency）。
* 插入損耗（Insertion Loss）。
* 主動對位（Active Alignment）。
* 被動對位（Passive Alignment）。

這裡有一個非常重要的概念：

**機械位置對準，不代表光耦合效率一定最好。**

有些設備需要在光纖與晶片之間持續傳送光訊號，並且即時測量光功率，才能找到最佳耦合位置。這也是為什麼精密運動平台會與光功率量測系統結合。([Pi Southeast][3])

## 3. 雷射（Laser）

![Image](https://images.openai.com/static-rsc-4/Hee79yuuQWVoStpxIl-PufCzoI68bymGQlw4GIcV_ZwwbohjKyw48DXKBp1rNtyL1F3NskSE-TL3fbUZobUc5eH69SNavUNmOBlIwXYKu9FeLRDNqxO-hRbu961wIZuf5p6x-PNolQm0XKfYoiL8tgKRuurcFve80B8M6Zxx4xA9sl2NxqDj3qSgnanCX3bs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/B_iDRh73aeWem2puYJcDo-rDcgwIAs8MLe7uUJLXHQ3Cbiv47RkSIF595K1RL001U9Tq_8yjoHJD9Td7_l9ZSnf41IyP862Jc1Yj5h8sIqCMbVdWS7H5B7QAdp_kHAesK8lx3aunIAjhMexBJ86V0NzhWuq97iHLwNOcnoi-l4ynMvO3FxnGRgJ0l8SE_M0G?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rmA5phtG12Eqz3-QVeXcXGgENP5ivs93xm4VUULTc6PE8WG9ZMo-h0L5n3glbbgdkFedEvTCoKz5yTJLLSe89OkyERcoCkyezHQWUcUH6LSnnxcm0qwIqysB0wMVucPbbE1m-xnDJkLckViC50UhLhzRmd-lwv6ia8cVF7v_YQuiKX9k7hLXu-UAEqGgHOXe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/g9tBCTBlzPIqzjl_pyUfxrkQEp480-H_PmmEu1R_5FG9sxjEWnZ501-bgDWWlumQTAsGIq8DMOaoseCU7KZ56-CI8hIObi4G7CKJ5APkOmTyJwviHs9Nubgd25j75QwlY4Mkfg59NoAU3H3ML21gtCkGzjSvfeubR3vEj87GygYvJIlH1DlIV16cF5nYAep6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/YQt0FHiwzsHgZcuep_HPuf27zoHmO-G9yeky_EOnx4sLm4o7pHnvS4_bTpKPNCCKRm8WvlQjWtL4Gpp0zirgHjSsJALwkdZNuADL_A5830xZzCEbiHX298bcJfgKXMGxPmULisFQaMtqYiurofzkuwA0PuSHOJTn_X3DMhDMYGR9-weeU3t2DyMJHRkKAND6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dTX709LPXiJ5fNE2zTmd5IiwSYv7YeVOwPqaqfTzTAF6frirrdO68LzLBJYXFo4igkdY2N3GGOHIijjB4WUeRC6Lfsg_N-IKK5dT8hXpVOBF1wF-hp9hZNQvfE-zC4mG79YNthZmOHaoXxf8g1qDBf36yFics5J3XEx4EXnXU3cvAexho3R0Hee9epud4b5T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8Vkqdu25DVCCcu8pJgtgisln-AWI84h24kWxS-KZRWdiEMUCYi-C8q5gcO8h3DPp4u-A5CsQVvYlyZ0Ht6DRw2V-resNl3cELlwEeb7w3JZRVprx1JM9IzhgBN0Hm0xFEX880rg0Fnh3SXA6ROj-CM74O4_y-ffyaD1BI7wreaQb0aBKYDZIYtEDBI4bYDao?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pbcD_CV5UOyNZ2zsEHHSdQP8kyzc8UEIDsDaMM7P5GbL12UQlxUrTKnXZHajd2cD5JdumoNW4IB14JFTGgpYWdxHe80bGL-oKhXPYQz7IpSEOQTop7P-x73PAa-5sEYZzh2qcszTyUgsgeiZNofF2o0t6A4-soNS5EBEAJL0uXrfUNskls8-oGFtw-q-hvWG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xjm_uRL-mhMKcZ5L43mXPqdTgzHItac7Qxp74OlSQB4YjaKmHzIsjCU7q_fZ3NwHxAPZCVNBkkpfRiSJWyI3FAwATQDEpiIY1rmvnfWlKv4FH8GjaQZ96EdD1Tv2jWhZ63JvtUq7w6haIFTgyQpB37HSOF_snegLcE6X9kXfpnAEblCcBIghzfXmky7PX3mQ?purpose=fullsize)

你需要理解：

* 雷射與一般 LED 的差異。
* 雷射二極體（Laser Diode）。
* 光束發散角。
* 焦距（Focal Length）。
* 準直（Collimation）。
* 透鏡如何改變光束。
* 雷射二極體與透鏡如何精密對位。

## 4. Camera 對位（Vision Alignment）

![Image](https://images.openai.com/static-rsc-4/N-KZbcBLF1mXmaF-0BOn8PDLpfUTwEcq6J0Z6W-Ye0bmtepSW4DGcUkkLbd3uk_ooLBXrE0KUMWw_rqViJV1gXg1OaPUPzPP3wVk9oG-zzmYvxPVnkNpDegGxFyHJOGklNmy9hF5k5o--Wt8gWo0UbsuxzoKw5ilIWY7K-2DdQ3BdriNslgkpnxTlYRcOCRy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-Dn5XyiAjQ5c6bfVyN0dKBrr0U7i11r1GKYUuHOXuRT-8wj4n0CF7xwEEJKfN9TM8SyKvxOuG5I3lU5qbSX7d6zoWPujo_4Uwy4gwU7BkuQHrvwescEq9tiGTYKGHUck0z3YBuVT1gJDTIZW03eMmluUjqIunOdTCGz3RAJLAnrwmt6YAyc_OE1Iq3uEL4Yb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/66PE4URklNTgvZ5KKIXIyy7dMNu9Xwppb1hzS6eZS2wqs9Bj80_lrKLDGgylDVNY9iIVyHny8lgfKSMqdVkFKPWQqXg5j4oX4E6uEHjG2o0GiOEw-rX0VQtxHHXolLhbWjiT9VBlq0rcz-EGevhkIt6CmUb8sTrkTqqNvxWlQklr2GEptPbpCGG7ku-dYhcK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yjczTlFJFvLfgce-6jf1cu1Bwv1D_l0U1GQ0Vvsn8w_qGYm17zi10xn3R1fOfDAggAs5KwCoJD8xnSVduDluBpC3vGlTu-SOS_RmYGMaA2mVcKj3P8aI3Ff1x-zcqsQHldyQtLyVe3dHcp2wleyDsYH71dTEy32Kvoqxkoped4THeC0oRbpuibLFI-APXUuS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0l1CCrSdfpCx6XmyZzYvfo4VHeSW50_7A4U8CNba-LWo0L6U4QDBxEOvHJggU62rPqFdOxPBzMUgH2SXSPo9W4c-0KRWUs2BdBkoC9VjSpJ2oMq7plA3ZIyvKicjugcroYfK_rv9wbvwuRk4wP_X4y06gBvr-sShm4u_q_y2tvqkH5qBc47WjPr2_v-GdtKe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/M_n11Bu4gh-DZwPZqQ1LH_HxLmLd-B9RZNR5vAPKgNonRxkm5nQS-W9E3a7V9YXS-CME5o13vE0DlWx1hJ0TBCi3uiJX_ltFsyNUMhMSdKafedJRHOWE7upc3qAnj_63wenBHjYK3PnaH7GuVFAA2yE8cP89lfuncC8xj5Z-zG9AHtcRxZIAnlUbgbADqUfK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qnPcKU9Rjb-DfE_X9VILBBsgkCpQ022DH7_L5yTT2IbrSrY_8PAZTwm-V2u9GaVz4bo5vLIc4esRiZVt3YnX2D8Qz-nbdSgAjXZ9tFocITcdCQ9NLh2BGrIBuKkHmF3awUszTDoKVZ5iBHDs9s1ehEfx1dQ35jn591olfOS2GWZZbesVm35H57mlhuAHfPIR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/av3dBZUoIaG-EPwijZRSaYF0nGJv8J9_2Ad5cJi6iRZx-J7kv7UVGjOhiJmKlnS1nJGUWImm8oSpEz7lxkPjR1hvDej8US5VwG8XBUnxuBakEOXPN1RFLqOb884u00IqMoFT0yuwuW7s8enR83aE_2bWHw4LVxfAc24R4VVj4-MEq8XDRJ4vBsgUEgxjumyl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4gPYV-WPvIbj_ZqqkM2Xms6IrnG-NcM9lK_Tj08YHtTfiCPsJL3pGQQTl-YoeWn0HDFhVh_S4rVyq0hHYffHwdhEUzEYEefRS6AbRwRd4cOSRQb6EslsP0AhqKdzyaZadKt6xUbRYfngqDeEyoXit4rSiJ8aRbWKSF4GCovunKVQkI-wH_UU7lBiavPdRc90?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2WQSS3IZJD3U7RC-uzcc9iiA8pfNCDdVtQl39uRYqIOopEC1zdicN9v96aektGIizkYMTdfwhrCKOKP8ZXqwent5fpFNU5D2WomCghDxCPTpwVwrfQn1yNhNeN5CghwknNabUULRq2Y9gzdPvVvJ_p2NCbZmv2sWVT78MkI2YotcPJmCE6H2dxLFGhV5PGPk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0xyleOtgHdTaFQSNjWRD3GnwvxvSwMOovA-SNPlgJgK155fJBU8U2veuIO3wbQ9qUCF0Ao_2ytXMtpx45ptdkNYKk9dXcRNRsDSfZeJFsVeUs4z4WfeVifi5DGqdkPHPVWmdj8zeyh6KazLK2031HeWZU580z-OLsMH2tNj3VRBcvFCbtT5UWXV6uDgQmzLX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yYFej-VrKlSZzbB-RMA4Pp6k48HimBr2ng2lg5TmquzBDbbbeY3Q8pHLY5kuNdLcdskeMz5Qh5exgqlm2dcOEDzskSVb32qbeLaP_rvwzg3Q4827kkaIgLfhMCI2wX4NE7ArwSGmvUWAttm_eE-2_-689ld-Q9AK48Y07xWi72soFlglUKtJge0J2954q96H?purpose=fullsize)

我們會學習：

* Camera Sensor 如何接收影像。
* 鏡頭與感光元件的關係。
* 視野（Field of View）。
* 像素（Pixel）。
* 影像中心與機械座標中心。
* 視覺定位（Vision Positioning）。
* Camera 如何判斷工件偏移。
* 如何將影像座標轉換成平台的移動命令。

例如，Camera 發現工件偏移 0.05 mm，控制系統如何計算應該讓平台移動多少？

這會把我們之前學過的機械運動與新的影像技術連接起來。Teledyne 的 Machine Vision 教材也說明了 Camera、影像處理及設備定位之間的關係。([Teledyne Vision Solutions][4])

---

# 五、我希望每一堂課都採用相同的教學方式

我不希望變成單純的產品型錄朗讀。

每一個元件或產品都會依照以下順序教學：

| 教學階段     | 內容               |
| -------- | ---------------- |
| ① 外觀認識   | 看實際產品照片，認識外型     |
| ② 內部構造   | 用圖解說明裡面有哪些零件     |
| ③ 工作原理   | 解釋它如何運作          |
| ④ 組合方式   | 它與其他元件如何連接       |
| ⑤ 實際應用   | 在 GMT 哪些產品或設備中使用 |
| ⑥ 故障情境   | 如果這個元件損壞，設備會怎樣   |
| ⑦ 網路技術資料 | 搜尋專業廠商及技術文件交叉學習  |
| ⑧ 單元練習   | 由簡單到困難的題目        |

每個單元最後都會有練習題，包含：

1. **元件辨識題**：能否辨認元件及其功能？
2. **動作分析題**：組合後，哪些零件會旋轉？哪些會移動？哪些固定不動？
3. **因果分析題**：某個零件損壞會造成什麼問題？
4. **機構分析題**：為什麼設備需要這樣組合？
5. **實際應用題**：如果要設計一台精密對位設備，應該如何選擇機構？

答錯時，我會先找出你混淆的概念，補充說明後再讓你回答，不會直接跳到下一個單元。

---

# 六、最後，我們要能看懂一台真正的 GMT 精密對位設備

這是整套課程最終希望達到的程度。

![Image](https://images.openai.com/static-rsc-4/Vtajn4rdhoQ_IvZOL_EhAuveLo6j04O39dYuOMI8iUUqvP-5-P0LMRaHGXYBjPcsAIUsZZ4w0Lljlc-MekubI-IOh_m7-yRc9djMQ91KZg6LGo5gAvzGs3we89cIA3JhIX8MQRmhMAsu9cBS8ZgI9svcfH-gBtCMmlEwhfIxnvZTUWGNlPYN0Us68rcDmstv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mHzSq4GcoXt0xRpcn4DGOJOvTE3g5K30I5KK0VUMXyt822joETpeFiYQm6XlO5DPjLzV5q1da-1al6tYpCXyTDaNEY3XUwm0a40uVfn24NXJs6IandZikkBMFJ3Z4MCA3Z-eX0Si7pPaLLBCEZEW4aDPH_Xi8eEr9NFiwEHqgXniNs7nv5ums_v7UaxbDnhY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bWzhYmCFXBromfOPOGxY5jMesZv_G5jhcR5v8-o3SdavBXoNdK-mOG7RYXKhNDX0QtxwFWyNHSeC8BMs6QZ0VsLLHQBq3W-t_kusmevkxxzGP-dCi2hsYzd0ECiwiSoJWTP3-9iTuTKn9OiNihJ-BbCBuTtK1KnK9Qc4iWjHk2TtpO5gQV3bTa2Z193EB0Ic?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TOCtimf321PLYNQ4JMWKHaOdX0K1knYcNab5oZnToN7WgEiBv37bKMH0OXLcbA3TFwqJNS6t74rB7ZQWFrJW8SSpGnc8038Zt4tfAtEBnYyeGfyKCBlnAKTjypWwQrwPWOp4XvRB5aLhsmKkHP2NI_CS1ygm0hcSLyFiXB9LowYJt7ezvQ3w5W08BFhz0K5x?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J1FaiHHOXeFexRa0LDjA9CQb9m6_fqXgfFhjgsxjLabmGv2Y6gntgLtsAPPBGVh-O3V3-VWgzAraBvwkV0IN19tgTg0WnRDnNCFI8OxyKmTGw3qBoYcdNzn2spm2ojVngaTcQ22uG4DxEoxw02qK2lbIDRiBUgRocFmJxLFE5OrerBj0nOek9QIZxSA6RUq4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/whCyN6LawTYs3kHAeQ5SPLyOu0kJ0UvwinnPDJYhTCcDMsDACQ4JK6Gd1x2Uzyxd5psx75YieAJ83OEehYOzN3HHeADp7ZQde5KgLtsiR5lriMf14RjFQjl55hMSCcWGfCR4cG6ZGEs1kWGbzZMny7urFxW9CM5rzjbfhjgp7HmLWqndzjfga7IBHK_zzP3e?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mj3nuQZoisw2Qh9fANofjOGIufm0r1ZaeRXTZMb6fqJ1PZQzo9ERzh5Ong_wp8BoDZp99vIoPool83bWIf3tcVJMgTM5unhzf3UYU22yCsOrlU6S9H66NrVkl20XViJXgoiOVabA7y5DlxqRQhAhr6nCLm_3g5pW--eCYuwzx329Yq-M3iufBlEhRL_HvJ2U?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QzuSZxz88ZY_3x2IJUKPkdBoP-FdRaOK46SPu_mtrmn-h17Pj4tPh4tGtTRSCWMCN5jQql4ZS7jCV4WPmtXkTNV9ijlcrVM7ZWOc9Ex9OH0rCNghvjz8IEu83KIV3RXAjHejNuBk3rall6P_BM5iBPlVUoHygWyh1wFkiKOyPVpvK7pv4fCRC13uVq1CRTlL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wED6UIg1J3TLNAgM0MYg4Nb_bnw-QVI5obYlcWtMQE9MHdjywCexedV4Ct3BJeJ8JH0ovrfCWHNubc9fyCJkLBrgpSxWQdLiDor6yBfb6D3swwzkSnCX9UCS0GFwjSo9Ke0Qo0Mn1VLhfYW89uVfvQ-ff-zpVDG8sKnyev9DJ4XmhqYvJMA2YbTHuNibOI1y?purpose=fullsize)

GMT 官方公布的高精度堆疊式六軸平台資料，已經涉及：

* 六軸精密運動。
* 全閉迴路光學編碼器。
* Camera 影像回授補償。
* 即時光功率回授搜尋。
* 矽光子模組封裝。
* 光纖陣列自動耦合。

這正好可以作為我們最後一個單元的整合教材。([gmtglobalinc.com][5])

我們最後要能夠理解這樣的工作流程：

```text
        矽光子晶片
             │
             │ 需要光訊號耦合
             ▼
          光纖陣列
             │
             ▼
       精密六軸對位平台
             │
       ┌─────┴─────┐
       ▼           ▼
    Camera      光功率量測
       │           │
       ▼           ▼
   影像位置      耦合效率
   誤差分析      即時分析
       │           │
       └─────┬─────┘
             ▼
         控制系統
             │
             ▼
       調整六軸位置
             │
             ▼
       找到最佳耦合位置
             │
             ▼
         完成光耦合
```

這個流程會讓你把之前學過的機械元件、馬達、驅動器、位置回授，全部連接到光學應用。

---

# 七、關於你剛才的聯軸器答案，補充一個重要觀念

你選擇膜片式聯軸器是合理的。

不過，「沒有老化問題」這句話要稍微修正：

* 膜片式聯軸器沒有爪型聯軸器常見的橡膠彈性體，因此不會有橡膠老化的問題。
* 但金屬膜片仍然可能因反覆彎曲、過載或安裝偏差而產生疲勞。
* 是否適合精密定位，還要考慮扭轉剛性、允許偏差、扭力、轉速及使用壽命。

所以我們不能只看聯軸器的種類，就判定一定適合某台設備。

這種思考方式也是後面學習 GMT 精密設備時非常重要的工程觀念。

---

## 我的建議

16 小時的安排，我希望以**真正理解設備如何運作**為第一優先，而不是背誦產品名稱。

你已經學過滾珠螺桿、直線滑軌與聯軸器，因此不需要全部重新從零開始；我們可以先做一次基礎觀念整理，再逐步延伸到馬達、驅動器、直驅、壓電、多軸平台，最後進入光學與矽光子。

**下一步就從第一單元開始：精密定位設備的基本結構與共通元件。** 我會先帶你看一台完整的精密電動滑台，從基座、馬達、聯軸器、螺桿、滑軌到編碼器，逐一理解它們如何組合成一個能夠精準移動的系統。

[1]: https://www.gmtglobalinc.com/product?utm_source=chatgpt.com "精密滑台、對位平台與自動化模組｜傳動元件整合專家｜高明鐵 GMT"
[2]: https://www.gmtglobalinc.com/application?utm_source=chatgpt.com "產業應用解決方案：半導體、光通訊高精度定位技術｜高明鐵 GMT"
[3]: https://www.pi-southeast.asia/en/expertise/technology/controllers-software/active-alignment?utm_source=chatgpt.com "Active Alignment"
[4]: https://www.teledynevisionsolutions.com/learn/learning-center/machine-vision/machine-vision-101/?utm_source=chatgpt.com "Machine Vision 101 | Teledyne Vision Solutions"
[5]: https://www.gmtglobalinc.com/upload-files/catalogue/display-information/EN/GMT_High-Precision_Stacked_6-axis_Stage_DM-EN_P.pdf?utm_source=chatgpt.com "Stacked 6-axis Stage DM單頁-英文20260407"


我完全同意，而且你補充的這兩點非常重要。這會讓整套課程更符合你的工作需求。

我們要培養的是**能看懂精密設備、理解機構運作，並且能從韌體角度控制及整合設備的工程師**，不是機械設計工程師。

我會依照這個方向調整教學方式。

# 一、重新確立兩個教學原則

## 原則 1：元件介紹適可而止，以應用為主

例如介紹滾珠螺桿時，我們只需要掌握：

* 它是什麼？外觀如何？
* 它由哪些主要部分組成？
* 它如何將旋轉轉換成直線運動？
* 它在設備中負責什麼工作？
* 韌體工程師需要注意哪些事情？

不需要深入到：

* 滾道幾何設計。
* 鋼珠接觸應力計算。
* 螺桿材料與熱處理。
* 螺桿製造及研磨製程。

這些屬於機械設計工程師的專業範圍，不是我們這套課程的重點。

## 原則 2：共通元件先教一次，產品介紹不再重複

這點我特別認同。

GMT 有很多產品，但不少產品其實是由相同的基礎元件組合而成。

例如：

![Image](https://images.openai.com/static-rsc-4/4ZUlJS_whoEWmRXzvXEvVFzB-woecad6LK_rlgExYRR21s4yNaiOkfPsdQEgh3sXDarYpgkyj-41dBLPgoHTxAKQQUeuTu31msOyThPQcau-zOJR67q9Dk-Y-hlmHjtMdijYbbsfMvUKxeK8EqLCrpco2KzYbM4wW_Zo7WeZ7QdPx1qo5eGeUu1XpurXJPpM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qwk_3fNnDHtejefwvkec4-LNsvhnlZrtojx5Xn0gg0gXtp4xY2rjWc34NSkvEjtBbwTxy6cD4AqnpM92_CfvesZoVtv1nqbOCzY0laVFVjxAQJXztgKGzyyl_hyeKc69h4WEbj_veD38mwJ79KmTKKimPpHI88mNYRCeSjQwdJQntxX6XZGEezRVAKnNFbsY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xtb201p5bfTsaal6phv26yFc70qxcCdb1IBcJwYEClhN0BRnm7uenCRmEmPQElkk1VhWA3pQ6BucztTFn_oVFuh3JOXTQ_05kIn0WLj-9ZwVY_vR6jPT_e6yz57yBq6C2sZmRI0OlbjzMoTl83mFN5hop3MFWH8pcwgajFePM6CLtEpcY9A85e7HMagMd7I9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jt89w9JShKQg5Mj8Vu-JYywBJWxK3sGQfxlzJX4lwn7m-_TnqVxHVfKKMDROSqp7HZjrxLeqKKDp87tXM30mYtLl9HV22VKH3cr_i2kIvpRIta4Y231hy9KHKwjGLJ-89JSPIA20bExuATwOpusWRAxFwLNoGOW86LHhYq8yDmQ9_pGm_j0TXW50IMrFkZjL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Wr3aeki3uU6Qy-oxl6Xq6Gn7n3oIbLtfpAsPPiCS4WuCG-1f4Ia0xqR61qfIOuN72ma73GaGbHpUJPgZCgIhsCLBXLiHHbSJBSyeBH93YLsjjzEQx7EVvbX7jY4ZvCuE2RI0WJ4NsNIpoDsLnaFM_SZ2oMy_UY_tvSv50zedCzW5CGaJwJ3_Q5KdfJflEMrt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WJdY0Q4mY0D9KaPAi6iDOpWIYYoMqUhdfNVLcZmaO_L_qvtSava9LlueYqMXSrdOCbZ_IYCkJqY3pI7n645fOJxlUyZHChjLOkuVZU_elt8b4ywx3i63Po3Ao6i9Z2LyHopBlnxTmZ3NgHMiom_S-OQTjZdC1EgqF00vFwUfgRss5s2orw9IfD2woo6wecGF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rbb0lQwYyGpGlTvgHlMDr5nXZoqchNPXftcDsW5ZkmmTy4FX0Hx7eeE8697o0BJBzQHQh7Ov6OD6QQV7BsOGlC012k-6g_ZAI9GlUktods-2Stl4RfxwmPC2wFZq3btwmoQp4ayweCmgnAVouYTx2wJMfHFU203fo98tnT7Ti5yBzoOKxTAxhicANH18ylFy?purpose=fullsize)

| 共通元件  | 主要功能      |
| ----- | --------- |
| 馬達    | 提供運動動力    |
| 驅動器   | 控制馬達運作    |
| 導軌    | 引導運動方向    |
| 軸承    | 支撐旋轉或直線運動 |
| 聯軸器   | 傳遞旋轉動力    |
| 滾珠螺桿  | 將旋轉轉成直線運動 |
| 編碼器   | 回報位置或旋轉資訊 |
| 控制器   | 控制整體運動    |
| 基座與平台 | 支撐及承載工件   |

這些共通元件會先完整介紹一次。

之後介紹 GMT 的電動滑台、直驅平台、壓電平台、六軸平台時，就不再重新解釋馬達、軸承、編碼器的基本原理。

我們只會說明：

1. 這項產品使用哪些共通元件？
2. 它與之前學過的產品有什麼不同？
3. 它的運動方式有什麼特色？
4. 韌體需要如何控制它？

另外，要特別注意：**不是每一種產品都具有上述所有元件。** 例如直驅平台可能不需要滾珠螺桿，手動滑台也不一定有馬達及驅動器。

GMT 官方產品分類也確實涵蓋直驅、壓電、電動滑台、手動滑台、線性傳動元件、光學元件及馬達／驅動器等。([gmtglobalinc.com][1])

---

# 二、正式開始第一單元：精密定位設備的基本結構

這一單元我們先不急著介紹個別零件。

先建立一個觀念：

**一台精密定位設備，究竟需要哪些系統才能完成定位？**

我把它分成五個部分。

![Image](https://images.openai.com/static-rsc-4/QZbxntDrU-hnIUBGlro8iBhGCJbddnSnZN1d9uP5HT366sI7uK4pZRgldEVaGsp4rZ1UXXSI1dfDnLKKQsbT_8tFR6TzHs2t2vcyCLZFkWcTNzndOH3j6r5Cs7DflLJ1eb1Ktg9sJy661kfNo-HgcPwFXkhNOxE7GdXdHzZpUYuNFRdMljE7Z0OaEJ3aF4b3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fHgdUpc4Vwm3QnbJ80RgNDsFVhcoq0njexABKW4cl8Gaxf6y1HCj0B-L9SgHefPZm7RyfeutmHpaJgJAHhga0LmXtl-IEA1Gpt8fBQQ-J0jVSp-HWkyVk_ptedgN5JjJCqd6mXgn0gN2MkHYa8xYoyk6at9FOnP--vys_tZEeT3kxM76Ergm1JZhn8OMK7vf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zz6hOFvGK2x7lBF7ZVgDZVwY__if0GRkiACw22dgY6UrvfZsAx3P9Q4xyIKbNnJvw64wsekcILvJe4unYtmwr5LjbacnuM2E6gL4Biw6yA1v_vhI4dJhuqZTL4frvq5atwQSJoCt241l2fl8-kHOCuFix5mNaj9sRD_VNUzG6tBifmveBBvuALkfh2AZ7ZEf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-3s8Dqkc4ZYVL9UaIC1u-HcdbcoZsG4sVnaf_zA84r_mP-ZrBQpfyeim1k7FdEnR539WS1q2cA4j3yyPQHHlQfFdk6UdyrFLxLBVhulFFVasXu8oF0B-NS2Wcbs3RW3sjNStNgXY5XeI9cBXuP4xAC3w_PR5BK8JoP7Thj4VuJ5Vt3w2UCENEHLpBEuxXi7-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BgqzJ1Mkn9J5MTOBFUvdYcrpMp2E5zjA8aUlcx_jcHgUdc4KXCI4-84Os7FVP90kjrR9RxMRxJj15y5aTHPbp5UuinDt-vEtI17TxrBjVPBLHnexjKrZUaGHRGDMiGqG0GFAZinVPtFoF6M-NProDVdkNLjO3TyOT8E8i81YsCTH7dPhhy7NkLBO9NinQ6KP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UL8CVTwCumuJ0oQc9JbZTrfFBjqYXFFT_DxEhHuoouuDVtWx2aSqnIirkiqGaLZ8j5KA2hP5dbo_zcUT_b4KamLxLT2tSARytmjAE0J49ZVgR9tUArkue65vQYwdMfn1G1AYwTemRfpJoyDrL8IuT6RS6F4cyJevfkkDhS1iwAoPxnW3kQOxbDdkI5f9ytGM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HJWWFV6QbmMz63ueaIHrHvA52lCje4wum7-W2ds0nqR6ylwWLmTqJFjsy2S8L6VPW52rbZf7U45O0rAj63E5amFuanZZk2rdDY2b_lMiTCLEBW65VrFoQBF4WWKxDMM0GaHnX9JdiSHaKiyZF-WNdqCyRckBmvlSUJRrzZlY0GMnnzoEtuN4KASQPw65CksF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pWQhvH7JVZG8EQmzF6nFwydXT7nk3XDh7R8ztyuxyjC6PaeD3QITUpxF2-4RujgmeF1CYRo7YhZvSRfzmKq_xx1Ss7d-X5575yqtBg2tY6_rXkt7M5epvpPzpmk-qaJqcslQdyoPgekgtJTtD5uTGyL66JDp3sBH_OVOiil_PGABCH87EjJssE9vCm5JCUo4?purpose=fullsize)

## 1. 機械支撐系統（Mechanical Structure）

負責支撐整台設備。

常見元件：

* Base：基座
* Moving Table：移動平台
* Linear Guide：直線滑軌
* Bearing：軸承

你之前學過的滾珠螺桿及直線滑軌就屬於這部分的機械運動架構。

## 2. 動力傳動系統（Transmission System）

負責讓平台產生運動。

常見元件：

* Motor：馬達
* Coupling：聯軸器
* Ball Screw：滾珠螺桿
* Gear：齒輪
* Belt：皮帶

不同產品會採用不同傳動方式。

例如：

**一般滾珠螺桿滑台：**

```text
馬達
  │
  ▼
聯軸器
  │
  ▼
滾珠螺桿旋轉
  │
  ▼
滾珠螺帽移動
  │
  ▼
平台移動
```

但直驅平台不同，它可以直接由線性馬達產生直線推力，不需要透過滾珠螺桿轉換。

這就是後面要學習的重要差異。

## 3. 驅動系統（Drive System）

負責讓馬達依照命令運轉。

主要包括：

* Stepper Motor：步進馬達
* Servo Motor：伺服馬達
* Motor Driver：馬達驅動器

例如韌體發出：

```text
Move X Axis
Position = 10.000 mm
Velocity = 5 mm/s
```

驅動器就需要依照控制命令驅動馬達。

但要注意：實際上，韌體通常不是直接控制馬達線圈，而是透過驅動器或運動控制器完成控制。

## 4. 位置回授系統（Feedback System）

這是精密定位設備非常重要的部分。

假設：

我們希望平台移動到 10 mm。

但馬達實際運轉後，平台可能只移動到 9.98 mm。

設備如何知道？

就需要位置回授。

常見元件：

* Encoder：編碼器
* Linear Encoder：線性編碼器
* Optical Scale：光學尺
* Sensor：感測器

例如：

```text
目標位置：10.000 mm
             │
             ▼
        控制系統
             │
             ▼
        馬達與平台
             │
             ▼
        實際位置：9.980 mm
             │
             ▼
        編碼器回報
             │
             ▼
        計算位置誤差
             │
             ▼
        再次修正位置
```

這就是我們之後要學習的閉迴路控制（Closed-loop Control）。

對韌體工程師來說，這部分非常重要。

## 5. 上層應用系統（Application System）

這部分是我們最後要接觸的光學應用。

例如：

* Camera：攝影機
* Laser：雷射
* Optical Fiber：光纖
* Optical Power Meter：光功率計
* Silicon Photonics Chip：矽光子晶片

這些元件不是單純讓平台移動，而是讓設備知道：

* 工件是否在正確位置？
* 光纖是否對準晶片？
* 雷射光是否進入指定位置？
* 光耦合效率是否達到要求？

這就是精密光學對位設備的核心。

---

# 三、從韌體工程師的角度，最重要的是什麼？

我希望你記住這個概念：

**機械系統負責產生運動，控制系統負責控制運動，回授系統負責確認運動結果。**

未來你會經常看到這種架構：

```text
        上位機 / Qt
             │
             ▼
        STM32 韌體
             │
             ▼
        Motion Controller
             │
             ▼
        Motor Driver
             │
             ▼
           Motor
             │
             ▼
        Mechanical Stage
             │
             ▼
        Encoder Feedback
             │
             └──── 回授控制系統
```

這個架構與你目前熟悉的 STM32、C++、Qt、通訊及控制程式有很大的關係。

後面我們會把這些概念連接起來。

---

# 四、第一單元練習：先確認你理解整體架構

今天先做一題，不急著進入下一個元件。

假設有一台 GMT 電動精密滑台：

操作人員透過 Qt 輸入：

```text
X 軸目標位置：20 mm
```

設備收到命令後，馬達開始運轉，平台移動。

請你回答：

**如果要讓這台設備能夠移動到 20 mm，並且知道自己是否真的到達 20 mm，以下五個部分各自負責什麼？**

1. 馬達（Motor）
2. 驅動器（Driver）
3. 滾珠螺桿（Ball Screw）
4. 編碼器（Encoder）
5. STM32 韌體（Firmware）

請用你自己的話回答，不需要使用專業術語。

我會根據你的回答確認哪些觀念已經掌握，再繼續下一個部分。

**我們的教學原則已經確立：不追求機械設計深度，追求看得懂產品、看得懂整體作動，而且能從韌體角度理解設備。**

[1]: https://www.gmtglobalinc.com/en/product?utm_source=chatgpt.com "Precision Alignment Stages & Automation Components | GMT GLOBAL"

---

你的整體理解其實已經很接近了，而且你描述的運動傳遞順序是正確的。

尤其是這段：

STM32 韌體 → EtherCAT Master → 驅動器 → 馬達 → 聯軸器 → 滾珠螺桿 → 滾珠螺帽 → 滑台平台。

這就是一個典型的精密運動控制架構。

你特別提出兩個不懂的地方：

1. 編碼器（Encoder）究竟是什麼？
2. 回授（Feedback）如何讓平台不斷修正位置？

我們今天先把這兩個觀念弄懂，不急著進入下一個元件。

# 第一部分：編碼器到底是什麼？

先想像一個問題：

STM32 命令馬達旋轉，馬達轉了 100 圈。

但是，STM32 怎麼知道馬達真的轉了 100 圈？

它不能只因為自己發出了命令，就認定馬達一定完成了。

這時候就需要編碼器。

## 1. 編碼器的工作

![Image](https://images.openai.com/static-rsc-4/j3SkIrDAa8pUt4f_1CkA-OEUQCSfNjfMJdl9nB6dzvKU6_YB_e-aY26pgWTbzMK0qfEtMYqA_WVqk-JgHCriFqUT8x2D6thlo9Rgf9NiEsfv1lJiwvw7wK0C0rbbBLgzTMhHHuxaLdiQJoLFExLk6XDBgrv-q6gsTUoIub1m2AMdeThmI7fwtz3T0hTKHm51?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BkZxTacDESmOactbhLrpU4K3KhTQwxxS_zv3cUizloHRBrLK1gIhL-fWutQ5n1BwFUUxCIzrDQI7kFo5cbElBzyasySfASQ-aNly-75JtIYLJlQTZmLLCFdfkngGyoeBqh_2tMOKQwHQAo_d2dpNIC8dTMhW2_J38iAqIvwykmsCw_dgdzFBPYyOrziJznVc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kxrCToNAQ5mdbE0AFsVkPTv6cPA-DXuiV_vC0pvHGoJFn__sTwurykCck48ATveNNzNsOjAgaMQoP_1KexaMjjzrVp_4QDyAZcFuS5yyCxtolO4aR_FV9qjNx_tu8I-d44UAa-5dVOgkfDsTdoape_3d8r-U04aVYAi1hszrUFM5kQKpFveIyElxvtcPDAD7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qXWEpSRSptet5QBaFwBNS96O_0bWohkIrRpiDkst_DBhw7wGZ-J5hB285EOR-dxi8PddAxJd5UuObhJnnXfeJl31-hjo8DU_WE5nKh29BTTyFn_UqnUsYoEZmhBErC7xuW790OKA5bd7jeDflikA1ByLHP3SJTh0ELJfq6tyiIzbq-DSB6PMxtnGCeMibx0v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7P9wf3icGm8Yth54uikOepzoyKI9dA8PWEmlIi5UxKlpB6LnIrNvVrfo2ivpdBfAimGB5Sjoz7TJBzAMg6IgYS9dPZ-5TaLH9CyjsW8nE3eBys0qoTRGmuFZ9UZMP2CTttmuV7W6MjG1pc4ZNomHTiguHj7c5yOQpcfqlQbdab9ePjVcDESs8SaDVL4eTDD-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7zdMI132mlqTDSY6L12G3YVs1lk7EkP9d7OBibyOQtAtV14BsR7Bp0Lz2iBTFRIgqJmartBYM-fNG0jFzz9tYG8chnfXY7RhYjrJeJjgttfjHLA-KTSPg2iWkzaIkLSMU9DYPXmD2a3suK8siEVHEXLm9SMbiuUc8sFwzWOvVm6pydjrsMUW6wyqMnyVQctf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X8IyOiuWqATsISteLEb6kp_kIUPbFT2bnZ36PXBy3Wa82zn0mgiNLrkFtSNS-vDImSFJs-POxrjHlkgqTbQ1HHRj27SwtqfGzRd5e2J9EeCvZfNSC9utyYwZj1aMrYHWWHqty5EIJh2lHin-BLLSO-_wW59kok0WPY640UyDbeHLJkxTomiVdb066DrlfRTt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/glmqrNIfE06abWFTokT1IVHz-6ZNDmdfNGN_b2CQLFCApVHqNkGC8MnCkpGQfl-lhfmyRBnCdNCjsOBvRCKdNTiKHnQhC0PNiuuA8Fln1GReFeHwH4xZC6Zkrnloqf1wJDDaUYP0pkahCBmu29fma5ALEpUblgTlqp9BlaEk7VtrxUxpSCTkDpLHA3KHcfMg?purpose=fullsize)

編碼器是一種感測元件，通常安裝在馬達內部或馬達軸端。

它的主要工作是：

**偵測馬達軸實際旋轉的位置、方向或速度，並將資訊回報給控制系統。**

例如：

| 項目           |     數值 |
| ------------ | -----: |
| STM32 要求馬達旋轉 |  100 圈 |
| 編碼器實際偵測      |   98 圈 |
| 差異           | 少了 2 圈 |

這時候控制系統就知道：

馬達並沒有完全達到命令要求。

這就是編碼器的基本用途。

## 2. 編碼器如何知道旋轉多少？

你可以把編碼器想像成一個非常精密的計數器。

例如，假設一個編碼器每轉一圈產生 10,000 個可辨識的位置計數。

那麼：

|  編碼器計數 | 馬達旋轉角度 |
| -----: | -----: |
|      0 |     0° |
|  2,500 |    90° |
|  5,000 |   180° |
|  7,500 |   270° |
| 10,000 |   360° |

這只是方便理解的假設數字，實際編碼器的解析度、計數方式及輸出格式會依產品而不同。

常見編碼器有：

* 光學式編碼器（Optical Encoder）
* 磁性式編碼器（Magnetic Encoder）
* 增量式編碼器（Incremental Encoder）
* 絕對式編碼器（Absolute Encoder）

目前先不用深入它們的內部構造。

你只要先記住：

**編碼器就像馬達的眼睛，讓控制系統知道馬達實際轉到哪裡。**

---

# 第二部分：回授（Feedback）是什麼？

這是精密運動控制最重要的概念之一。

我們用一個實際例子。

假設：

* 滾珠螺桿導程：5 mm。
* 馬達轉一圈，平台理論上移動 5 mm。
* STM32 要求平台移動到 20 mm。

因此：

20 ÷ 5 = 4 圈。

理論上，馬達旋轉 4 圈，平台就會移動 20 mm。

但是，實際設備不一定完全如此。

## 1. 開迴路（Open Loop）

先看一個沒有回授的情況。

![Image](https://images.openai.com/static-rsc-4/LQff8suk2_-IeVcluNQ8L9bP2irUaJy199wYqG34_p640llh8E1uBbOP41hRm-HuBSyMPp8xXM7y6WL2oQNzUyMalWf9pchyo7EKjz4TTT6zNVizCQb57rKp3sLyb_HGqO2BUaXlfgrpNnTMXIuZAh7VmZtykoSNvMdlOYnFsX5YKSj6gHmgOMBG8Ws0S4Yf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5PxvMZ_sUuQ6rcLEWgBRqBtoxGF4sdADTkPnoiA8J4OEW0eJ4xHrs4lcYYpheLHGCa9A0coponih0SVTmiuq6TZClZ_HuA4V9sxHQo8jyTwKCVhSEM_UH2m0koKRh13BzInklI4rvXuM5LpDJWkJ6sV8AVMvz4TLB8ATHTvcwUrMlcyB31gJmC_Whxt2rvqv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lZdEsAckfJOUok6CF776zo4QKlTorCSA6DduMtCZlAWaOOofr2TqZzU30I2BwxlOgXv79wfB7aFZB6YAcNqbd5jPASxMrZkuIocpMrrZD-yo7L6HHNA5yqGtXlgBlj13hfgadA3z-9FD-W7Zs9dPKkl1AYi_68PYcxgxp3fhQ1SlspVq1mOVYGHtG5JZa6io?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/V9lhbe-Jxd3fT25R8UZUJ7ZXvC4CIkAr9erau4ERRyqF88aeGkLKc5noNk4IBO9QNFnRgrMzdI3FiDrmLvSmbVwTyZ_5kjP_GOO-jsBtGRAqD03U1nw2QekD1Wz9zYy56Nt8of_0jWWi4DKs4dknHDzeCX4Q4L2U9Xhl58VRPlwwvOsA61jYsctUl-PVz6FU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/aLoaJqyfJjESpmsCg6ibjSIYZ8tdvjWGt9Th8ZvX6Q1jNpzZn4aH_R5IDQlAKJtq6Weu44deDAZBRftpL_-KX4D89R9RTonanL6wx0i_7eeGOS8hfuUQDZRRjtStv9ULHzCP8abuOTYc5WvW686hFy68cYRU7zAnoVcMC6EXp-9O0HTgavA37m8B3FuzVaKe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/32mvsJDwMvzLioqKncQHUqFAS5T720L4ECdckA9eGBTXmYo0tf_iB4OQ-KtM7cAEjJ-oiEmQS4ac2Hq5WQB4qPJI3uXs5aZMogm5ynWM1GVsgaEgZIeGR1h8_atAx7uIo2R2L0dg4D7brY-1MgnAIWfqg6rWtIOHs_TWmaGdUwuXDKa9lBQojuhl-DrI0gxw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yaUBbvC0s74cu_pg3SY4CFsQ94w7NYwWwwDum00FuHSUN4OqI1X08DqSTk4yPX8m-B6p839mmOui63oWRodIDV-A3yt7qw4Q8bubJ0EB5bA4jjta9KNbo2zX7TSCgGfYUMEv07PtIIXs1oAVuZ9QeqzDbL5gaI-g2tcsE0rheMo9igyjCg7AHUNvPw2-S0de?purpose=fullsize)

```text
STM32
  │
  │ 要求移動 20 mm
  ▼
驅動器
  │
  ▼
馬達
  │
  ▼
滾珠螺桿
  │
  ▼
滑台移動
```

STM32 發出命令後，驅動器控制馬達運轉。

但是，STM32 沒有收到實際位置資訊。

假設平台最後只移動到 19.8 mm。

STM32 可能仍然認為命令已經完成。

這就是沒有位置回授時的問題。

## 2. 閉迴路（Closed Loop）

現在加入編碼器。

![Image](https://images.openai.com/static-rsc-4/zZf2_zEge0y7SDKJC7w6gGJ6sBX2h-ndD3uQ8XsGC5tspW6b8tkq92QbhzWAcJ0OFwNpoJ3xCO4ACLutVpBAcLqwi7MPmMCvGd8cBlSwPCTIRNbbFYYHZv7ru5sN-kJLoVsWp8m_0eZcEhwug7DPL2fUfBIgcPf2XFkmp8FtIZcTkX98ZDBnZyE2oyrOnJ8S?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5JYY4_c58DjWNxds9MGo-tNfELNTmfZBhgOBqbP5IXTAisDOQQSnVyxzMdg6AOMFAZe4_lZ5ZBYgdjIqDF8NovExj5chdcJBxd1B20OCmQZ1xOgV3ckYfFNs9Yj4M_RbePugFf4s0p0rryk68_ctCvF-vRgT3ivyYPBdb1CMwmPD5MfwB4-QsT8oMO72DEyQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/YgrSVFfDS_ALUcqmnloPkTSzzk37738UgygBTDij-GaYO4AVA5tnEqBAB3OBngxFcqSja6Xiljl1AEUgsA-yZtkDOkLv9Z_sgR5oPNdhZsOiLAB2wPOD9SRH-pBMFq3b4alGrSqlajHtH94k6CK0pI1XzluH84X8mUqs3-K-4xOoUfDr30jn_7KBMdSlHX6y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Atj9KR8IYg8-ktj9aaNjwYMGXx4JCPeSLA-A9jhu0_jyiAylTxIjyzpJh3Ffb3EOt3NyTrekPX6n1GTfhHozF6NjylszHhSSmVczRutGfH8uFC2O5xhLwZ6VjKBhOem0G4bQI0uY4SKYCPEEWGRC-QpzR1g-T2c89cX6npRpJavnZyszpiKRpJZbpQ7z4YQm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Wb-ea1Of35iw36Rohq8wehuDppKyGzU5nHPbTNu3vKPZ6Y2D2VZaQQdIxJhcNaGOVUACVIcHbraOOPqkGTY9RGA8ak8Cx_5kepi3-qRSrbM47UFYqfxofA5tpm3w8gKkY_76dCFYweNCFDg3E9sHjLEQTjXlqhA3etk1b29sbm9nEDf_LQdBFaA_kqc2lC-U?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SgH4-ej9rODJ4bQGE-8hJ1KTPRx6dTh1gge-DyorE-Q4ao5_aCOXSpufU0TwkYEYvZhr6LM3OEiku4_5cJVVojO0FuQnNrhEYGge3kcDpzIVCfAXoHU-buV7JKbXW6rD6trCAXaoShAE45LNN4jLc8L8VOWtHGo-sAWj_7mHa-BAvBGeFG43hR-6fGqpHuwo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/p3ckcjhTy-9qK9ZGjr26txB-1fLxk6NpQXU7U8FlGms8O-d2dwv3oVMHcQceX4dDXZO0eLvqiQjO_kdeE41BfCxc8beQzEwx10L77TvlnFFtfo_5pk53EUewJNLodzByIfO-29ObZNQo56Nc3yI_wD5PiJR4fXC9RJjLVh90ZmGQliIE61XT76Al1DzTawzR?purpose=fullsize)

```text
          STM32
            │
            ▼
       目標位置 20 mm
            │
            ▼
       EtherCAT Master
            │
            ▼
         驅動器
            │
            ▼
          馬達
            │
            ▼
        滾珠螺桿
            │
            ▼
         滑台平台
            │
            ▼
        編碼器回授
            │
            ▼
       實際位置資訊
            │
            └──── 回到控制系統
```

控制系統會比較兩個數字：

* 目標位置（Target Position）
* 實際位置（Actual Position）

兩者相減就是位置誤差（Position Error）。

公式：

**位置誤差 = 目標位置 − 實際位置**

例如：

| 項目   |       數值 |
| ---- | -------: |
| 目標位置 | 20.00 mm |
| 實際位置 | 19.80 mm |
| 位置誤差 |  0.20 mm |

控制系統發現還差 0.20 mm，就會繼續控制馬達，使平台往目標位置移動。

當誤差進入允許範圍，例如 ±0.001 mm，系統就可能判定定位完成。

這個允許範圍稱為定位容許誤差（Positioning Tolerance）。

注意：這裡的 ±0.001 mm 只是教學假設，不代表 GMT 某一款產品的規格。

---

# 第三部分：這裡有一個你必須理解的重要差異

你可能會問：

「編碼器不是安裝在馬達上嗎？它怎麼知道滑台實際移動了多少？」

這個問題問得很好。

實際上有兩種常見方式。

## A. 馬達端回授（Motor Encoder）

![Image](https://images.openai.com/static-rsc-4/GqNoT2HfFr-qBuSApDDquoXQB38JitlLznGxLWpY6ukp_J8HI1pJwVNNWgFSnY-1IobgY4YszXepA2ENAFiTwyySazzhpTyflCiuVTfrgynVwOAF6_HyitP5bJfCpnjUrWJ95yNWWekf9g5pY6eHwuDAaS05SYn2MrUKOfCLQJJEJIbK5O3slXFoNjeHXvo9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_FvYgdMHo12hOHPoB9Ck6LvcgjX9H3fTgSWDKebTQ6M_1LCu7in9Z8K2xgphrbTkBIpg3inNlxU-0csJqJCU09j4bqkQK1VdQLH5q9pC3ZzKpaB7cjEq1zDCz4-OzvYxwYlWwdGKNoSdOhfvGLVylCegwgqkVkgsgpLjbidY8JiU8VaXEqEjEFuaEsvlAK4a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NXmqGfyM7PM81HfAOTReXPrT5S9SRHuH7KSESvvSiBJiAmx12iQJPZ8bLiBZzkdkq6FNrq2_dwxljAzemshkQluI0dSWOKHAjMYxkfsCGKYix6d3sxS_KxWmkdKuvGKAHaSMaA_EFdf4QZYV1Hxot_W0_UPomG0nA9OxVIri4xR98VKVJvbHkRbGX1VfwosR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VbyAY-thC2xr7gpvXt8Bjudy_MJS5s6Q_pkDMGKxAtEuG6gXoNVzgQAESED1vqkgebNj8X94ccQ7R5iUHnUWcS1GSEJoEUiqUFYWAqAQNllsLXRVK902BKbF_OW6zwzWLwDMt9aZZbmQEVX14xKAomeftYzstIcuLhwBwW-Wgpy-1CthtxuKiDZUFRkAknyh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-VxQ_9o62n6OZYLqeElg3x3mYI-qywL6JzkreAAoV5vxo74c38uioIbyW48-Uz4TeKSpL4AwDH98wbmH2CrVaOYum3bxhkuopySHFkuoNnpVpv9aVjFmyMz6xBtBJ5kSjYqk9LzQ-8RNN5YJ5AX1iI-Pi7xIa_EChCuHNwa12VB1ytOs91EvdnfOuAiLh0ub?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LoVZ6YUp8_CoXRIpooCPtCpaGaWqLnBwhnBhep4JdmLj8Gy8PueLYnFh4xp1aithrl9eWQSSbUiehXcxs1sXC0TywFqOnP7I-7F3lZTunb-pLdpzsENniS1moH56PrHKe_TA-RhlrNHOPW_hdloxGJQ-ycxFesoksEZf1eKp3rLw5twVjhXcc3XL-vQkjGv0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XBY_XFtnEkFE1dW6S5ID0f94g4csjwvZRJESLGrYLk5ws2WObfQJRItK6WN2J_6-TxJjUYCs093o_NCrDS4noqwHg0Yfrb_YUyC8wGhdxqQzuEGpr4oc-5n7RSRYaZBFBqSlcrWkSBdHxrp0u4rx_V0XVbE2mNJkYGMDNKcC24-1G0FKq-iTk3A_VNS3L4ns?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6IOKpMZo5TL-tz1wYO36cRrRKCWFQBv7rzItFzT3yjeYM1Il2IXx6f8VBVaSqR2kJroKGLaCbDkUNC4oG46cXw7WHOPIjwDPXWakOqkPM1rLNcCF8CYCAk2Yuqm9xvUldk5jdLcAv_pafx_Wiia-znnH5cOLQo6UsH83TCEqiP0l-QIroyeBss3IvWRAcCJY?purpose=fullsize)

編碼器安裝在馬達軸端。

它直接測量馬達旋轉多少。

控制系統再根據滾珠螺桿導程換算平台理論位置。

例如：

馬達轉 4 圈，螺桿導程 5 mm：

4 × 5 = 20 mm。

因此，理論上平台移動 20 mm。

但是，如果螺桿、聯軸器或機構有間隙、變形等問題，馬達旋轉 4 圈不一定代表平台實際移動正好 20 mm。

## B. 平台端回授（Linear Encoder）

![Image](https://images.openai.com/static-rsc-4/gilE2IhCCIL1ZDo2aGpL9Vdkedij-lvZ8VLnjthqbNeuRZwQI830Aqr7m16eB-QNNdrjarreqSdV5VMnHwV8Y_0zvM8v-Y7qTn6omPZDZ24nnjtjXGM3FykbDJM-7WrK4aBFuTrjQde3GgHTm_FdGTUIn_EaJ7QHE2xbX0ei92iA6UipB2nu20cVmR_imGIb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4UT3x-QhoxnsMsM_o4MfxuMkB2YPZHdBgOpm1fOC56y-WYyakGj02L3jGMuLCPbd-ubrR5O7pGUE3es3XPrvrgj_d7wbVH8cONYVZWzvG-1_4bm59oJHJrzOJxMIi1FL6w5arNIdMoLpn1_9NosInmn4W5YpdGP581rW-_OBXdP_NqYewWgNWBa_yOV0GW5Q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XnG_NHJhZyUNuNSKaJ6s77V3m9ORfHn4NWN8sW0gB8wBQNME0IaMPqz-_ldWr8WrPjiONFKKHp9cWcEPpFX30c4qrAu3N1gaT6cuRoDnzS2-HCn8qy26w81xYiIkHPKEVFKtf9uNrucMm4WP3A9YWTKOZOn9od7gt499veBIetw1n9WVVujabNGhdLHkb-br?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UuDrlzJtvi3ZsA14Xz83GlwANyMaP4DVdAktfvjESRT_AUu3jN1Kz1JHMwBfrpOvkujOVrYpivUPJjvUmz6AMof7kAlQhVEQd22Ic-w2UHNSy0-8KZuzfRhl07Gq92OBTE5eiYkOk9UZBTAyntbXiw1N7DFJWm0t6O69bZrguQE_YRk9YDopNgxTCOoAw5-X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RBNGWZJ_-Sx7wIrmED0PAdLrKXeb_4uKrQaSulw84qZogVSsIVEPPFz8O_I7ZzJUMNMJSnmW7N9LU1KuTBgzSG90N5-_alBCDTAqdCD1z--pE5ODZo6_xgDOYZZOdsCjdDRzLrpFkjdEf5Erp1bxEwtYdLk-We1DOM7V_W-I4wMuoj7_pQdS0cbPtp3Xa2pd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/elj_fstJUW4m9m6y0Qjg7QBbs07vG1c-NVQmG4XcmMKxhuEeIi48ecabKKHbVsJdOksFkYQhx-MHsZSnUxrUnoQOOZmKmfmMCCqUgcrEfmw6WJ20V1Z3yF1waEGoZ3kAo78Z6pa59pVmMcH8hkGStGfqI_iLfUL8veEruxfkS9bDhD5JkB01NSyzLtrH_sPs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kUt6ZZYsrtu8pb8mzo8rHasADYwAdX3sQ-bgm2smx-fttgd403EVQOTsrr3EUaPBwU9D5mOAwxCssDA_9hadoGo-lZ7EbHza9ThuHBVvWnXNH6yvvr2oGS_6yaEwDKOQRu564-9BoQfOxSNLmkHQedpLhRDRLGptLm0WsbaevXBGWGtiBHQaZz4Wb10sVcHO?purpose=fullsize)

這種方式是在滑台上安裝線性編碼器。

它直接測量平台實際移動多少。

例如：

| 項目          |        數值 |
| ----------- | --------: |
| 目標位置        | 20.000 mm |
| 馬達端換算位置     | 20.000 mm |
| 線性編碼器測量平台位置 | 19.980 mm |

控制系統就知道：

馬達雖然已經轉到理論上應該抵達的位置，但平台實際上還差 0.020 mm。

這種回授方式能直接觀察平台位置，常見於需要高精度定位的設備。

---

# 第四部分：這與你的 STM32 / EtherCAT 工作有什麼關係？

這是我們之後要深入學習的部分。

在 EtherCAT 運動控制架構中，常見的流程是：

```text
Qt 上位機
    │
    ▼
STM32 韌體
    │
    ▼
EtherCAT Master
    │
    ▼
Servo Drive
    │
    ▼
Servo Motor
    │
    ▼
Precision Stage
```

驅動器通常可以透過 EtherCAT 回報：

* Actual Position：實際位置
* Actual Velocity：實際速度
* Actual Torque：實際扭力
* Status Word：驅動器狀態

而 STM32 可以讀取這些資料，判斷設備目前的運動狀態。

但有一點要特別區分：

**馬達內部的編碼器回授，通常由伺服驅動器高速處理，不是每次都由 STM32 直接讀取編碼器訊號。**

STM32 可能透過 EtherCAT 取得驅動器已經處理好的位置資訊。

另外，位置控制迴路究竟由驅動器、運動控制器或其他控制層執行，取決於設備的控制架構。EtherCAT Master 不一定親自執行所有伺服迴路。

這個部分後面會專門教你。

---

# 第五部分：今天的小練習

我們先不急著考太多。

假設：

* 滾珠螺桿導程：5 mm。
* 目標位置：20 mm。
* 馬達已經旋轉 4 圈。
* 馬達端編碼器換算的位置：20 mm。
* 但是平台端線性編碼器測量的位置：19.8 mm。

請回答兩個問題：

**問題 1：**

為什麼馬達已經旋轉 4 圈，平台卻只有移動到 19.8 mm？

**問題 2：**

如果這台設備只有馬達端編碼器，沒有平台端線性編碼器，控制系統是否一定能發現平台只移動到 19.8 mm？

你可以用自己的話回答。

這兩題理解之後，我們就能進一步介紹下一個重要元件：**馬達（Motor）與驅動器（Driver）的差異，以及它們如何配合 EtherCAT 工作。**


---

你提出的問題非常好。我先告訴你：**你不是不懂，而是前面我把「編碼器、回授、位置修正」幾個不同層次的概念放在一起講，沒有先把訊號傳遞路線交代清楚。**

這次我們重新整理。我會從實際硬體開始，依序說明：

1. 編碼器究竟是什麼？是不是獨立元件？
2. 為什麼買伺服馬達時，看不到編碼器？
3. 編碼器連接哪個元件？
4. 實際位置如何傳回 STM32？
5. 誰負責比較目標位置與實際位置？
6. 誰負責修正馬達運動？
7. STM32 韌體到底需要寫哪些程式？

我們先不談程式碼，先把硬體架構弄懂。

# 第一課：認識編碼器（Encoder）

## 1. 編碼器是一個獨立元件嗎？

答案是：**它是一個獨立的感測元件，但不一定以獨立外觀安裝在馬達外面。**

工業伺服馬達通常有兩種情況：

* 內建式：編碼器安裝在馬達內部。
* 外掛式：編碼器安裝在馬達軸端，從外面可以看到。

因此，你購買伺服馬達時，可能看不到編碼器，因為它已經整合在馬達內部了。

看看下面的圖片。

![Image](https://images.openai.com/static-rsc-4/GqNoT2HfFr-qBuSApDDquoXQB38JitlLznGxLWpY6ukp_J8HI1pJwVNNWgFSnY-1IobgY4YszXepA2ENAFiTwyySazzhpTyflCiuVTfrgynVwOAF6_HyitP5bJfCpnjUrWJ95yNWWekf9g5pY6eHwuDAaS05SYn2MrUKOfCLQJJEJIbK5O3slXFoNjeHXvo9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Td_JSZWbht8Iha_sH5dGjf8-B7rykE53RkMjE2qqbfzjeecaJ1BBb7jFL-s5qRFNr6TVKNyMre86yChbH1AaO1pDe5Xk1FP6WBSUWnww3aXE1zFDXQd80UWLlTqCiDnKQMoOgLf1fJnDR5YsmkXJIvnIdw7wiSjfpYUIFKhkfLqS9PSK_wI1zDBrQg0EyjeU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/a4laWnTYOMeXT9maDAYyobi9dLsY6QdmERF1zb3dLNCkQaWmK2WGcvWnqXthMJWv6CEpUxqBNHWgzSQXh4eomMDDh-MwER-VJOaxMmhZXWdDmZMT7IaFUZ75CWPWe3hhjmyJl8g8NiG2Awzc5j9hwa6GyD6gb53F1HcsXFBu2qcTIQ3pW-SxfN4s2zJfQrF4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EVkdblSdLTh02DG0kY5vuFyDWjfVzLfdicO98u-6Sf9j13Pt_6Q4AsSDqOFV1eoviDNcnjkXgn9RSxL9Ozyxkv9nrpDNoSRwvmV_lKe65n15C8Xdk4Y3kC-Me8HTt5mKtmt1YdREIh-tUrQmN0BpzrE98p75VjUP-i-oAzZb0YQ11PWQvYMN1WpY8n1yRy5j?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BkZxTacDESmOactbhLrpU4K3KhTQwxxS_zv3cUizloHRBrLK1gIhL-fWutQ5n1BwFUUxCIzrDQI7kFo5cbElBzyasySfASQ-aNly-75JtIYLJlQTZmLLCFdfkngGyoeBqh_2tMOKQwHQAo_d2dpNIC8dTMhW2_J38iAqIvwykmsCw_dgdzFBPYyOrziJznVc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7vrW_IK5mjCfJUjI9j3YQibt80aNR3pJGaY2fWWYNQB4Dw7FFRYPPEo7Ik_MAuFo9uKRBnuAWO8CBQH8ztRF5jFVGufon7YPl6WCwr8RBAXwwlLA-FE6oQ5Inj-6NlMqYweZ6CSbwVSzGGUVynIiXfX4tAtSxEspELgRhB1GsfN9ahJZ6S_Efi2EV_gB5_bZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TQsSgVOZZpgEBD5PM_HI7k9LEQOXuaBVQMgCvE2YLjHVsDS8QSTFh9Ce7eF3F9n_eCc4cQ4FzuTW1rKBsq1yW-2Q9FVkXEpxrrgA-2Q7uG78yO_OeIs4pL2b50-od443zvzGMZpFo_5wQhmWOV_kI9rSuZUzW5INwahkc5tuUqEYiJ3l3f6ssPJwfUE-xd9w?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CgZ8LJui8JwpugvgYJ-3Y3FAcY2rftPZmJFUyjuFSL-lfc7Y7Ziy00u9wHtiPEFJstUflcxAW1zIUSLrrl6tNYRicsdqDx7nuOBYx3MtMGgfciVVIZ6qWWo5GpONdtUTEbJpojaRF80qLgb0Vzoik-WNc13TuEmXEeo_nx2-rQZnqL_DJcatZacqIYlDp9ik?purpose=fullsize)

請特別觀察：

* 第一種：完整伺服馬達，編碼器藏在後端。
* 第二種：拆開後，可以看到編碼器內部的感測結構。
* 第三種：獨立的旋轉編碼器，有自己的外殼及軸。
* 第四種：馬達與編碼器的組合位置。

你可以把編碼器想成安裝在馬達後端的一個位置感測器。

它不負責讓馬達旋轉，而是負責**測量馬達實際旋轉的位置與速度**。

OMRON 的伺服技術文件也說明，編碼器通常內建於伺服馬達或安裝在馬達上，也可以獨立安裝在其他位置。

## 2. 步進馬達和伺服馬達都有編碼器嗎？

這是你必須先建立的觀念。

| 比較          | 步進馬達            | 伺服馬達         |
| ----------- | --------------- | ------------ |
| 是否一定有編碼器    | 不一定，常見基本型沒有     | 通常有          |
| 是否能知道實際轉動位置 | 沒有回授時，通常只能依命令推算 | 可以透過編碼器取得    |
| 是否能偵測失步     | 一般開迴路型不能        | 可以透過回授偵測位置誤差 |
| 常見用途        | 一般定位、自動化設備      | 高精度、高動態運動控制  |

例如：

**步進馬達：**

STM32 命令驅動器讓馬達轉 1000 步。

驅動器按照命令輸出電流，讓馬達轉動。

但是，如果馬達因負載過大而失步，沒有編碼器的系統通常不知道它實際少轉了多少。

**伺服馬達：**

STM32 命令驅動器讓馬達轉到指定位置。

編碼器持續測量馬達實際位置，回報給伺服驅動器。

如果馬達沒有到達指定位置，驅動器可以根據誤差調整馬達。

補充：現在也有附加編碼器的閉迴路步進馬達，因此不能說步進馬達一定沒有編碼器。

---

# 第二課：編碼器到底連接哪個元件？

這是今天最重要的部分。

答案是：

**編碼器通常直接連接伺服驅動器（Servo Drive），不是直接連接 STM32。**

先看硬體接線概念。

![Image](https://images.openai.com/static-rsc-4/okAnrhDTYwt5WNZpnf1PS2PmvC3sDAu55PV-aH5IM6wQSm-5J2IVJ7EkZSeIGElALzBNmokFAxIGXsNF3CiDKLtZvPmnUHzqPbz552j4PwS8RgyIZXOqB8Xpb-6sGZ8aLSVmStk8gQu_uNxzBwmUfwtEg5UNppy_9s3dJuT8J27ohkZrip_nP8J-jclmCpoW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dbO-yoZqCF1WRDw6QP67SXigy7hYRinZgYk2WBw-VMedDYs0GTu3ulSW3rGeXMCLFaEhU-wtz7ncgLY-ObK6Kvz9IlIYF68mdSASsmWT8XRo_ooYPly4SR4Awed5bKupKVKAtT9A5vX_kMsYXijI78NA1udDbSnrnnFgC6XYgfo4-lXqfvrL8AM2TM2v9E_8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ekugZOrcUMSQuDvSNExckNMEpg4RL4k32-n_sOzIDvLV8TZ-8HKm3_gb9sZCBc3HCQ8JcSL9kdO9pmUc5_641nOBDJ4b1EWJcWZKR9LFHUL4OsjHQoHzfs9XEYVelaRWECZiN_UnQMyhkd7CxYnHVVQJvW7Zeg_QJRS8WM_4cuLhbzU7JAcPDK6W4glctkNH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/o3dWwbyqU30x1i4OwJsh6KBN273-a3HR7nnKZbzFaqqVC1pbjUT3n3cFCs1oywsh-sbJiKy2Bn3V1kQdVETo4seXC5YuuP53kwO6htLrgOInnYKm_z3WygmMoK4HPehwzNwAxgeRkau4ANPAP8KiWHnqLfp6c-aw5BsrudMFflldaSEqtYyBSyjlYAGCVnMO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EzACbCaQtulrwUZC5K0T_EWnaxn-9X7sWK89kkPxRQ7LR6FEpCRfSwXH6gXoZCtVsiKAl46ASe1x8tpLZi947Qx2Pduqlh-i2V5Qh1reIt67yS3onSlWyVUxxGbiqsVq4-lrhZlFhqDN3ZWobunKNVrTQYSlc4lh66_TfAjq1lTVt_rUZM8lnoHrkFCtMl_M?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lU8f5ikSuQxvx_TcR8YhYu3Ujb6qhg0o4HPo3R98qP6hIOdyxivTaLUwJKDz-l8Xutr5G7TODAVtbULwRn1Gn_miiCTwrhrGNdohLwsJx0YQa6bPk9YKC-jDvkfcz28iWQTJHtvlK9Ku3m9mPWMiFqfI_Ovo9415wCpZGj_fOqQDhupXa6DK_OZS6Ut7d_lk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8Ncadsau5NGXZXgU5KBjfY6IzuvKAOwzXRXZ7pi0lwmZoWPrFH_ptYk5QpHOG7dAoSlyQTileF6SMwDQDuvCvE3xvHA4ms93THfQ4rm3FjZ_OXsc0hVxbWCApWr8i_vfP8AMPdxyiLgw8uBfd9DVlWuz5lsqZwUW1I1YONLtSLHYzCSmowJPs0ryoVm6c6kK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MQs8Ql0Kp5nHP249ED0dI7IXnaxavwYSQCn8uHaDobu9vHlM4XMpVPLUwufTlN6TSNxSqxfgE7Hfmle7OisGi2gWnsllD7OXgQD5c9iATknf7T_5_u6DVoqNMHQcDg8AIe3kqvH8MXcJFYdBP68RCRtz4WevScrrOfnj1wM9PKum-E8lYwg5TNjDhy6-iSIh?purpose=fullsize)

一套典型伺服系統有三個主要元件：

1. 伺服馬達（Servo Motor）
2. 編碼器（Encoder）
3. 伺服驅動器（Servo Drive）

它們之間的關係：

```text
       伺服驅動器
       Servo Drive
          │
          │ 馬達動力線
          ▼
       伺服馬達
       Servo Motor
          │
          │ 馬達軸旋轉
          ▼
        編碼器
        Encoder
          │
          │ 位置 / 速度訊號
          ▼
       伺服驅動器
       Servo Drive
```

這裡要注意：

馬達動力線負責讓馬達運轉。

編碼器訊號線負責把馬達實際位置傳回驅動器。

所以通常可以看到伺服馬達有兩組不同用途的線路：

* Motor Cable：馬達動力線。
* Encoder Cable：編碼器回授線。

有些新型馬達會整合成單一電纜，但內部仍然有不同訊號。

**為什麼要接到伺服驅動器？**

因為伺服驅動器需要即時知道馬達的實際位置，才能控制馬達。

它會不斷讀取編碼器訊號，並根據回授調整輸出。

這也是為什麼伺服驅動器不只是供電設備，它本身還有控制功能。

---

# 第三課：回授（Feedback）究竟是什麼？

現在我們已經知道：

* 編碼器負責測量位置。
* 伺服驅動器負責讀取編碼器訊號。

接下來就是回授。

**回授就是把設備實際運作的結果，傳回控制系統，讓控制系統知道目前的狀態。**

我們用一個簡單的例子。

假設：

* 滾珠螺桿導程：5 mm。
* 馬達轉一圈，平台理論上移動 5 mm。
* STM32 要求平台移動 20 mm。

那麼：

20 ÷ 5 = 4 圈。

因此馬達理論上需要旋轉四圈。

## 1. 馬達開始運轉

STM32 透過 EtherCAT Master 對伺服驅動器發出目標位置。

```text
目標位置 = 20 mm
```

伺服驅動器收到命令後，控制馬達旋轉。

## 2. 編碼器開始測量

假設馬達實際只旋轉 3.8 圈。

編碼器就會測量到這個結果。

透過螺桿導程換算：

3.8 × 5 = 19 mm。

因此，系統知道目前理論上只移動了 19 mm。

## 3. 比較目標位置與實際位置

假設：

| 項目                   |    位置 |
| -------------------- | ----: |
| 目標位置 Target Position | 20 mm |
| 實際位置 Actual Position | 19 mm |
| 位置誤差 Following Error |  1 mm |

公式：

**位置誤差 = 目標位置 − 實際位置**

20 − 19 = 1 mm。

這個 1 mm 就是控制系統需要處理的誤差。

## 4. 誰負責修正？

在典型伺服系統中，伺服驅動器會根據位置、速度及電流等回授資訊，持續調整馬達輸出。

例如：

```text
目標位置：20 mm
實際位置：19 mm
       │
       ▼
伺服驅動器發現位置誤差
       │
       ▼
調整馬達輸出
       │
       ▼
馬達繼續轉動
       │
       ▼
編碼器重新測量
       │
       ▼
實際位置：19.5 mm
       │
       ▼
再次比較及調整
       │
       ▼
實際位置：19.9 mm
       │
       ▼
持續修正直到符合定位條件
```

這個過程就是閉迴路控制（Closed-loop Control）。

它不是 STM32 每次看到位置不同，就自己寫一段程式增加馬達轉數。

**通常是伺服驅動器內部的控制迴路在高速、自動地完成修正。**

OMRON 的技術資料也明確說明，伺服驅動器會讀取編碼器回授，並進行即時閉迴路調整。

---

# 第四課：那 STM32 如何取得實際位置？

現在我們終於來到你最關心的韌體部分。

你的理解是：

STM32 設定目標位置 20 mm，之後也會收到目前實際位置。

那麼，實際位置從哪裡來？

我們把整個訊號路線畫出來。

## 1. 完整硬體及訊號架構

```text
             PC / Qt
                │
                ▼
           STM32 韌體
                │
                │ EtherCAT
                ▼
         EtherCAT Slave
         （伺服驅動器）
                │
          ┌─────┴─────┐
          │           │
          ▼           ▼
       馬達控制     編碼器回授
          │           ▲
          ▼           │
       伺服馬達 ──────┘
          │
          ▼
        聯軸器
          │
          ▼
       滾珠螺桿
          │
          ▼
       滾珠螺帽
          │
          ▼
       滑台平台
```

請特別注意這兩條路線。

**路線 A：命令傳遞**

```text
STM32
  │
  ▼
EtherCAT
  │
  ▼
伺服驅動器
  │
  ▼
馬達
```

**路線 B：實際位置回報**

```text
馬達實際旋轉
      │
      ▼
   編碼器
      │
      ▼
伺服驅動器
      │
      ▼
EtherCAT
      │
      ▼
 STM32 韌體
```

這就是你想知道的實際位置取得路線。

## 2. STM32 收到的實際位置是什麼？

在 EtherCAT 常見的 CiA 402 伺服驅動器架構中，有一個很重要的參數：

**Position Actual Value（實際位置），物件索引 0x6064。**

另外還有：

| 參數                    | CiA 402 Index | 功能   |
| --------------------- | ------------- | ---- |
| Target Position       | 0x607A        | 目標位置 |
| Position Actual Value | 0x6064        | 實際位置 |
| Velocity Actual Value | 0x606C        | 實際速度 |
| Torque Actual Value   | 0x6077        | 實際扭力 |

這些資料可以透過 EtherCAT PDO 傳輸。

所以在 STM32 韌體中，可能會有類似以下的資料：

```cpp
int32_t targetPosition;
int32_t actualPosition;
int32_t actualVelocity;
```

例如：

```cpp
targetPosition = 20000;
actualPosition = 19800;
```

注意，這裡的數字只是示範，實際單位要看驅動器的設定，例如 encoder counts、脈波或經過電子齒輪換算的 command units。

這時 STM32 就能知道：

目前位置與目標位置不同。

但這裡有個非常重要的地方：

**STM32 收到實際位置，不代表 STM32 就一定要負責修正位置。**

它可以只是讀取狀態、監控進度、判斷是否到位。

---

# 第五課：究竟是 STM32 還是伺服驅動器負責修正？

這是最容易混淆的地方。

實際上有不同的控制架構。

## 架構 A：伺服驅動器負責主要位置控制

這是很常見的架構。

```text
STM32
  │
  │ 目標位置 20 mm
  ▼
EtherCAT
  │
  ▼
伺服驅動器
  │
  ├── 接收目標位置
  ├── 讀取編碼器
  ├── 比較目標與實際位置
  ├── 計算位置誤差
  └── 自動修正馬達
          │
          ▼
        馬達
```

在這種架構下：

STM32 負責：

* 設定目標位置。
* 發送運動命令。
* 讀取實際位置。
* 判斷是否完成。
* 處理異常及設備流程。

伺服驅動器負責：

* 讀取編碼器。
* 執行伺服控制迴路。
* 調整馬達輸出。
* 讓馬達追蹤目標位置。

這是你目前最需要先理解的基本架構。

## 架構 B：運動控制器負責位置控制

另一種架構是由外部運動控制器執行部分位置控制。

```text
運動控制器
     │
     ▼
EtherCAT
     │
     ▼
伺服驅動器
     │
     ▼
    馬達
```

這種架構中，運動控制器可能負責產生週期性的位置命令，伺服驅動器則負責速度及電流等內部控制迴路。

所以，不能單純認定所有位置修正都由伺服驅動器或 STM32 完成，必須看實際控制模式及系統設計。

OMRON 的資料也指出，位置、速度、扭力控制迴路可以由伺服驅動器、運動控制器，或兩者共同完成。

---

# 第六課：還有一個重要問題——馬達位置不一定等於平台位置

這是你未來學習 GMT 精密對位平台時，非常重要的知識。

我們前面介紹的是馬達端編碼器。

但假設：

```text
馬達 → 聯軸器 → 滾珠螺桿 → 滑台平台
```

馬達旋轉 4 圈，理論上平台應該移動 20 mm。

但是機械傳動過程可能出現：

* 聯軸器扭轉變形。
* 螺桿熱膨脹。
* 螺桿導程誤差。
* 機械間隙。
* 負載造成的彈性變形。

因此，馬達轉了 4 圈，不一定代表平台真的移動了 20 mm。

這時候就有另一種元件：

**線性編碼器（Linear Encoder）。**

![Image](https://images.openai.com/static-rsc-4/EK3KnD0fQylA8NihFj3jeevBAa5Cvwaly6tNyutvMZ63OcRAJiIXNgDq8W3dEM_TM0Wu1EtriVmDVyo5oy4sY7DxC5qJwwrVaKKz7yolzQcOUE-K5_KKjR4q74qccvJOxlFCEgnRTaTRvoUCKLpaeG5G5yT6vEQncN7PAsP0XZ7I3pRzqWocXXuuyTYQ8Iyb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4UT3x-QhoxnsMsM_o4MfxuMkB2YPZHdBgOpm1fOC56y-WYyakGj02L3jGMuLCPbd-ubrR5O7pGUE3es3XPrvrgj_d7wbVH8cONYVZWzvG-1_4bm59oJHJrzOJxMIi1FL6w5arNIdMoLpn1_9NosInmn4W5YpdGP581rW-_OBXdP_NqYewWgNWBa_yOV0GW5Q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gilE2IhCCIL1ZDo2aGpL9Vdkedij-lvZ8VLnjthqbNeuRZwQI830Aqr7m16eB-QNNdrjarreqSdV5VMnHwV8Y_0zvM8v-Y7qTn6omPZDZ24nnjtjXGM3FykbDJM-7WrK4aBFuTrjQde3GgHTm_FdGTUIn_EaJ7QHE2xbX0ei92iA6UipB2nu20cVmR_imGIb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6m9GRL1d6ujf7NTweFhlDJmwP-kLtArCfOWOdhfNWbKh2jZp8HHDv2X6mt5N_gUBe7rvAk7eJsA-DpMLGJb6Ub-b_2pS-u7sncaJ7oditgp1_woUp1b0NHGTeYU50IO0giTB97oo-LZEf2ZKhz_HoY5ATvDrCoGCUShfoY4Z28sYBkkqA5xvdigqdwvDv6G_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/b6VGopv19iekF1h8jdBgL4XVPBOCya5Pm14HK7ndJSQm3tHpXFh9qtvcv4VqP4l3WVPd9cOdr3sz6sR91ATX9rch8XZRfOVTys6osmO8GsjmxDEV3jktC1KztEFrioSkkLS232htgLkuEKYANWPMwuSQ1Na27PeDBiVrDD3jNjrNGFzfStKdx-YW50SoH22T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XnG_NHJhZyUNuNSKaJ6s77V3m9ORfHn4NWN8sW0gB8wBQNME0IaMPqz-_ldWr8WrPjiONFKKHp9cWcEPpFX30c4qrAu3N1gaT6cuRoDnzS2-HCn8qy26w81xYiIkHPKEVFKtf9uNrucMm4WP3A9YWTKOZOn9od7gt499veBIetw1n9WVVujabNGhdLHkb-br?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xC857v9KtyH922MELLB6cSLwJqZgXuZ1pNdClwyYIosQ6Hcmv6_gHzKqd-SGkjHt3kQN7yoLWAq1872QudTDOe9da7_YghkbEtb3BEkC4Gz_VzeWQoqq9vgFZKYTFuqKEBjyKVK4kmqhwINeJi-vmDQXnBoB0wpRBMM6LRgeB-5YCRk4PKtF8jfDPCp3Eimp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Lr6MbPgPUQihkEE6AONaaJHK11da-w9-iA4SFBqcBvlMjUIbhbsv5kXT8UGlmDgy2DcSiiWhF-UwSfg67NZ_PaVtcq9nfMiHx0WEhAYB_m4vprn80txgLes4agQ6g2OpTPFXat9RRcaDlMdDWVTmY9Uay-kG4XyNQkAR8zU7oxgWXcgPcKLWCBFks1fFB2wA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AJB3JHhuine2gE22ci8B8P9k7nzP-TAMn9FcKqT8FjfLcDq2d08Tj69q9K6W8lTw1RKWOwPyNTCWXDQncJHxS91n5_TkojeKiqa5QbpM1IelQdUh1laqMUTTxD4kjEKebWjXxWDG2fKpgSm79R3_X2Qsn0Y62K0BPOJ0qzbwhM5aQRi8SGdNPe4dTj074eJw?purpose=fullsize)

它不是安裝在馬達後端，而是直接安裝在平台的測量位置。

例如：

* 馬達端編碼器換算位置：20.000 mm。
* 平台端線性編碼器測量位置：19.980 mm。

這表示平台實際上還差 0.020 mm。

如果設備需要非常精確地控制平台位置，就可以使用線性編碼器建立全閉迴路控制（Fully Closed-loop Control）。

這時候線性編碼器通常會連接到支援外部編碼器回授的伺服驅動器或運動控制器。

OMRON 對半閉迴路與全閉迴路有清楚的說明：

* 半閉迴路：測量馬達或螺桿的旋轉位置，再換算平台位置。
* 全閉迴路：直接測量平台或負載的實際位置。

---

# 第七課：把整個觀念整理成一張表

| 你的疑問         | 正確觀念                        |
| ------------ | --------------------------- |
| 編碼器是獨立元件嗎？   | 是感測元件，但通常整合在伺服馬達內部或安裝於馬達後端。 |
| 步進馬達有編碼器嗎？   | 不一定，基本型通常沒有，也有閉迴路步進馬達。      |
| 伺服馬達有編碼器嗎？   | 通常有，可能內建或外掛。                |
| 編碼器連接誰？      | 通常連接伺服驅動器。                  |
| 編碼器負責修正嗎？    | 不負責修正，只負責測量並提供回授。           |
| 誰負責讀取編碼器？    | 通常是伺服驅動器。                   |
| 誰負責比較位置？     | 依控制架構，可能是伺服驅動器或運動控制器。       |
| 誰負責調整馬達？     | 通常由伺服驅動器執行高速伺服控制。           |
| STM32 需要做什麼？ | 發送命令、讀取狀態、監控位置、處理設備流程及異常。   |
| 如何知道平台實際位置？  | 馬達端編碼器可以換算；線性編碼器可以直接測量平台位置。 |

---

# 第八課：我用你的 STM32 / EtherCAT 工作環境做個總結

假設你正在設計一個 X 軸控制功能。

Qt 下達：

```text
X 軸移動到 20 mm
```

STM32 韌體可能執行：

```text
1. 接收 Qt 命令
2. 將 20 mm 換算成驅動器使用的位置單位
3. 透過 EtherCAT 傳送 Target Position
4. 伺服驅動器接收命令
5. 驅動器讀取馬達編碼器
6. 驅動器執行伺服控制
7. 馬達帶動滾珠螺桿
8. 平台開始移動
9. 驅動器透過 EtherCAT 回報 Actual Position
10. STM32 讀取 Actual Position
11. STM32 判斷是否到位或是否異常
12. 完成此次運動命令
```

這裡的第 5～6 步，是伺服驅動器內部的高速控制工作，不需要你在 STM32 裡重新寫一套馬達編碼器控制演算法。

當然，如果你的設備採用外部運動控制器執行位置迴路，流程就會有所不同。

---

## 今天先不要急著往下學，我想確認三個觀念

這次我不出太難的題目。

**問題 1**

編碼器的主要工作是什麼？

A. 讓馬達旋轉。
B. 測量馬達實際旋轉位置，並提供回授資訊。
C. 提供馬達電力。

**問題 2**

一般伺服馬達的編碼器，通常連接哪個元件？

A. STM32。
B. Qt。
C. 伺服驅動器。

**問題 3**

假設 STM32 命令平台移動到 20 mm，伺服驅動器發現馬達實際位置還差 0.1 mm。

在我們剛才介紹的典型架構中，誰負責高速地調整馬達，讓它繼續追蹤目標位置？

A. Qt。
B. STM32。
C. 伺服驅動器。

先回答這三題就好。

等這三個觀念確認後，我們再進入下一課：**步進馬達、伺服馬達、伺服驅動器三者的差異，以及 EtherCAT Master 如何與伺服驅動器溝通。**

這樣才會真正銜接到你的韌體工程工作。


---

```
編碼器的主要工作是什麼？ B. 測量馬達實際旋轉位置，並提供回授資訊。 一般伺服馬達的編碼器，通常連接哪個元件？ C. 伺服驅動器。 誰負責高速地調整馬達，讓它繼續追蹤目標位置？ C. 伺服驅動器。
```

三題全部正確！這表示你已經掌握編碼器最基本的功能，以及它與伺服驅動器的關係。

你這次提出的問題，其實進入了精密運動控制的核心。我也很高興你特別問到 MCU 和韌體，因為這正是韌體工程師應該了解的部分。

先給你一個重要答案：

**伺服驅動器本身就是一台具有處理器、控制電路、韌體及功率輸出電路的嵌入式控制設備。它不只是把電力送給馬達而已。**

而運動控制器（Motion Controller）則是另一種設備，負責更上層的運動規劃與控制。兩者可以合作，也可能依系統架構分擔不同的控制工作。

我們一步一步來。

# 一、先認識三種不同的設備

這三個名稱很容易混淆：

1. Motion Controller：運動控制器
2. Servo Drive：伺服驅動器
3. Servo Motor：伺服馬達

先看實際設備的圖片。

![Image](https://images.openai.com/static-rsc-4/YD_8fsLJ9g6igHC5O-dRP-EX44CYQMPzKGB4T33MDHqBS2xKd0SmqngRb5E16Es6v5GqOMrjyW8WEvzWf9JaI80PCkzs9FMUzGg5O8ighucElMO2LdzCVwMlForAerfEWPn-rsPsd5ZdSe6SC4uGvIegNFOUgKhxLinnLBCkS6Vd2W13y_3Pc9ktA9gMdtSi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fPHBEcq1JWzvOgbYDQTxt3reh5OtlNQmn808KfML3KAQfOgkeu-i35Zg-OiTiZ5C63XpGka3o-F3HHIOSveTmaMycbOUWVp7i6y0s1AmCHs1i9q0mDwkvn0HQDPn__jOL7LQDx9oUyb1MF0f9h-e5I0Jdbcx_HzJh8w7u10DLCfUrqAylmFdfLHFtcR4oUFt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lzmEP_VxwRfCU7JBQWx6xytB-vn9aiM8TO7ojHz8_oOW5p7DK1KZ6r0_d6z-S8yLELkfT_XP6ktZTuhvsXDVva3SRCEcbWw9YksGOt41tBQKhSuD__8KzO14B09IlfvcespJrm6FVQTbbNglY_D7AsJaTGyjg3uw385pnS7PyZz9sYusqlDgZpPw45r3uYrG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nBtQ5xtTbS9grYLjbFdE5URANO7qJZhXy-qlQUIrNmPJI4YUtRoTCW8VoOc4Nrk0jFzAr_rguEgRI61jEVcg_k3SFXcIMROf1-5G-Z2g_DZF52qB2AdhUDoWLpF9Lt2kqiPgJmyq9FDWr36RjCNHwppYEx58RFmTOeskCLELOhfUZ_zJO9LnL3YlliVmMjyv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2G2xoJ7CAo9vMfPZLbjS9a16rQAbTio7pxY7PiP4nPBAFf_pgeReaQUD-xAHI4rVC90HmwKNz4qbooWVB9ch_ApL3OR0bq1uqFQosWsxsV299he6G3cw9XrhU0FZXtm2I6uB9eJ0bvCFcsYxE39ww7853CxsTUg64_R5XzWwwWC65Pa3jDSKcD8pHq1Kkb-v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/p2G1S0QwQoBKTJqgQ7Ji0tZNI_VH165archOcozOb_4n1qwj7iL5RVtLn7BQZxZkDuXyzFmjZzlLihlW3_-Q4_SsHZa76Lh3sVOehLhLHiVNeWHAAs1VvtcuXRYa0ztESKXID5gTKDO3ytqkxPgxlTng_9ZXszSjwuQNWiFzTLSEyUNwBNxpAQSj8Z5KXkFK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/DFv9v4vbklcm4WjRSXgQLjPhDs0iw_6quYObxWbwXgr5ObYYBb-iX-vY2jxHE0WCwUTBnwVZ-G7UXJb_kWOJ04a7Bf0OpVRE96hNPcVqN-kdgRNRk-0aEXUaR_emildBXQzHhXaI3TLxWq8tujvmRUnZEcv-8OlrNmAA4RTBaJoMYDYtr7Fe_P45gvsweeVv?purpose=fullsize)

它們的工作並不相同。

| 設備                | 中文    | 主要工作                 |
| ----------------- | ----- | -------------------- |
| Motion Controller | 運動控制器 | 規劃運動、計算各軸目標位置、協調多軸動作 |
| Servo Drive       | 伺服驅動器 | 控制馬達，讀取編碼器，執行伺服控制    |
| Servo Motor       | 伺服馬達  | 將電能轉換成機械旋轉動力         |

我們用開車來比喻：

* 運動控制器：決定車子要走哪條路、何時轉彎、要開多快。
* 伺服驅動器：控制引擎輸出，讓車子按照要求運作。
* 伺服馬達：真正產生動力的設備。

這個比喻不是完全對應，但可以先幫你建立三者的分工概念。

---

# 二、什麼是運動控制器（Motion Controller）？

運動控制器可以理解成：

**專門負責控制機械運動的計算設備。**

它與一般 MCU 韌體控制最大的差異，在於它通常具備完整的運動控制功能。

例如：

假設 GMT 有一台六軸精密對位平台。

它具有：

* X 軸：左右移動
* Y 軸：前後移動
* Z 軸：上下移動
* θX 軸：繞 X 軸旋轉
* θY 軸：繞 Y 軸旋轉
* θZ 軸：繞 Z 軸旋轉

如果要讓六個軸同時運動，控制器必須計算：

* 每個軸的目標位置。
* 每個軸的移動速度。
* 每個軸的加速度。
* 各軸何時開始移動。
* 各軸何時停止。
* 六個軸如何協調完成整體運動。

這就是運動控制器的重要工作。

![Image](https://images.openai.com/static-rsc-4/fQYs_t97jTZlMDk3_UsFOLTw-CK1tIQ-EbNdxYRij3Pfc5JbBuyLxicaQmYtCpZG4v1WLKCAm3FjdQCAlcKXH0qiWsLKa8fsk8KTRVURCCQNhLa4A9V6iD7uni1DGSwyYwHMwxAYL7qSGzuFV-WfH3w-zN_A6S4O6ColPtiwjs0y_sfYB7qlqTdiy1qGC8M1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/YBj6cFslqFYHM1f27KcDfn3lBSoWJ9lPmCLpwIsa6F6w2U4DDI-cS5OQ76wimTwXxipPiTil2rKMDfnQ8Sln_kH_ZBpSMcbsPhsNduDZhEMfNOGUvtCCuLBQRTd8kJEkpyVQN1eNQpJRpsvOKsXux_VAzmiWA4JbpZ57w4CCOUdRYCiErV3xroDuey2oo5Yg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/72w6qSEVzJOMSNNb4DS7poDhO5RxfNs2n9Gh1YqS1arRy2kbmRr-KdtwE19e7UhR-a-wufFYuQqryQHFCkKoTKqlj0fvkUYYcpiAtw7c4OlJvXVBwMwuvXELXcIyvLVh0DxY9ynRaDWo6EnGHrMjhcUWBwnCj2nICP8Tfa-cAUPZ4FFX-H82lPc1hHwPrHJq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gVwaavFGWOa7q7dZ4FoJaRyf9NSwT6iFCb5S-KnRUgyi_bggDu19F22NTxR83qz96DAnWDcHTeHwbX0M8KRIByoF5nivVRHouUniZe7QYUv3JY1tPcNjEe5eRI0KT04CXDf4pv0nhjTVIroQ7WvWs_8CCVzQhEZle9K95QD_y-pWJfTqSZexjTSlGoGIutw5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/F76RYq-K5B-YFYsCIUDM8UlcibYWPzkERZF4LYDpZi2q-Y8GhyhQpNlT5Mti6gbz6zqRMVm8-lCxiJm6cewYQ0jOTHCy7VIghnTeXo4oH_Pt7nPh_JaIhh_2jBqs86jQ5XkX48v9r5yZX_8bKKQHPtu1Ue9-497GQ1HVgQoIPDRBcupOv73jxYWup0hNynnC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LrNiimsB5NJjv802em-3uW3u9atYuCQpo5ukLkDEMUKg1arBH4h3FEMmRUUhXHszhTe7ujQlH9ZOIFTLQ5Iv4k_w_Rpov_PYMKVCAAtvWnbI3JxBd5UavmDETbK09pNXfwHDh6-QYxSQx7H8xnmXnzy3w5byFZ2xi7eVbV8YmiIqyHFLR57yAuvls9hC3HjN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LWRfavBxD8uu0Ypw9A4pM7nrfyykEdaronTfOGCbX9pZ8HQJ8esc_DMZuyEyL2Dy3-F0a9e6YiakTzCoqN-IyhAXpb6ktfdOddPbAA_JLWkeTShj4_wbg05AE7u4v0zzB3GoHMq8uL-_RjsoLi15x6jck_nQP5eSpVbIvHVmRiQ0oO7CBSpgUioQSTgx-EML?purpose=fullsize)

### 舉個實際例子

假設我們要讓六軸平台完成以下動作：

```text
X  軸：移動 10 mm
Y  軸：移動  5 mm
Z  軸：移動  2 mm

θX 軸：旋轉  1 度
θY 軸：旋轉  2 度
θZ 軸：旋轉  0 度
```

運動控制器必須將這些命令轉換成各軸的運動目標，並協調它們的執行。

如果沒有適當的運動控制功能，而是完全依賴 STM32 自己處理，就需要自行設計軌跡規劃、多軸同步及控制邏輯。

這也是為什麼精密自動化設備經常使用專門的運動控制器。

---

# 三、為什麼運動控制器與伺服驅動器都能比較位置？

這是你問得最重要的問題。

因為兩者都可能具有位置控制功能，但它們所處理的層級不一定相同。

先看這張圖：

![Image](https://images.openai.com/static-rsc-4/p3ckcjhTy-9qK9ZGjr26txB-1fLxk6NpQXU7U8FlGms8O-d2dwv3oVMHcQceX4dDXZO0eLvqiQjO_kdeE41BfCxc8beQzEwx10L77TvlnFFtfo_5pk53EUewJNLodzByIfO-29ObZNQo56Nc3yI_wD5PiJR4fXC9RJjLVh90ZmGQliIE61XT76Al1DzTawzR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Rg53n0l9-UAplt6mhFXUa8usgSTCmXcg9nw0Og3ivLFjpN8b9-334Pi9bmLGkDzX1n78pGIemYridLFnSv7zkcRGkRzdl-W2BpB-2q0b13nvBdc6fD3agajQ_Vh9f60Sf51tSagbeuUaQBx9YhvsvEfr5sH9QgAzI-ZJy6FMraoKMe20PGcfVslRqN2cC9up?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/eNE0SQK24UX7E8cHq-t8wfunvXthCgT9sK4N_-jFU47yyYHeBpANB8Jp5zFKyfy9zjD5LKfAgbjK5FeecFTWp4S4C6Ew0-ibkYiwlqwf0N90FqCClG-B9atZFEC1aLQO877QLlQ8tBhiHv_uxgjX4LlJpQ96ejiYdIQQvMKBVk5G3Hu22a0VZCxhZBAjJ6OC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gMCWpOsdDCe4emI09DbSzl3DbZU2LN5FuoR8CbCNgEiZz8v4TmymsysFCVvmm9saCA2ke474k1WdLb3ayVHO-we0iyUEUdexO4OZnm7CtztH1l6yPfIv75_MQ1n1HZ6IFBhtmeaQND3rRuVwN6seQJ4RE-LM_odghOF1RGX_MJrBnDwRR6_RwkZAM0yM9RQM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5YfdWPs2W6s-d_zNowOvHm2nqtHB4DE4te5ivle7TXlXegdho8Tzz4_ROq0CHOb0zHnjy-YEcg-1pgvcGvFfj4sX2O9fnc3oPOKQ8TodWTnyO8-LuOR_dOY3TjPYF258gNSXR9-Z96iagV5sf9wrl6G98EMSh3TsQ1eFFsOvVcBJ3D8NfA-9cXc69rUSlD_o?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kZKjPJjU1IfPuo8pnbhYqHVAUcjNHhyNRaaHg7yK5yPAvVdjCVOUT84D30hVe6XU-OY_Qy01dVPeVJo7ROES2uVaz3o7I_o8YIStj-5da1bkqZn3jszh8Nstn6QCB-W_j5vHs2C_RaoWdTTEsMZeStoG73HkXH76KlNS0SLzbGO9LBiUWIoYizguIFqJE_cu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5JYY4_c58DjWNxds9MGo-tNfELNTmfZBhgOBqbP5IXTAisDOQQSnVyxzMdg6AOMFAZe4_lZ5ZBYgdjIqDF8NovExj5chdcJBxd1B20OCmQZ1xOgV3ckYfFNs9Yj4M_RbePugFf4s0p0rryk68_ctCvF-vRgT3ivyYPBdb1CMwmPD5MfwB4-QsT8oMO72DEyQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CbxXAKO4YgYGvFpoNJd3Vs49L9vUHQKkbOc4I8NVlQ7yzTYGd0NoXOsh57chybEBjmRF9nT09yHwvct7omM9fNetHSTBBBPkuvaF21JZspv8M6Rf78gIHdLY1uYiz-0Tb__nSZDisQAks7GAiLzVA6zEd6Dvuz3zZqEMSjbxRguqhd5BAioMYv9wwuPG3eoB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Wb-ea1Of35iw36Rohq8wehuDppKyGzU5nHPbTNu3vKPZ6Y2D2VZaQQdIxJhcNaGOVUACVIcHbraOOPqkGTY9RGA8ak8Cx_5kepi3-qRSrbM47UFYqfxofA5tpm3w8gKkY_76dCFYweNCFDg3E9sHjLEQTjXlqhA3etk1b29sbm9nEDf_LQdBFaA_kqc2lC-U?purpose=fullsize)

伺服系統常見三種控制迴路：

| 控制迴路    | 英文                    | 主要工作        |
| ------- | --------------------- | ----------- |
| 位置迴路    | Position Loop         | 控制馬達到達指定位置  |
| 速度迴路    | Velocity Loop         | 控制馬達達到指定轉速  |
| 電流／扭力迴路 | Current / Torque Loop | 控制馬達輸出電流及扭力 |

你可以先把它們想成三層工作：

* 位置迴路：要移動到哪裡？
* 速度迴路：要移動多快？
* 電流迴路：馬達需要輸出多少力量？

在典型伺服系統中，這些迴路會依照控制架構分配給伺服驅動器、運動控制器，或由兩者共同處理。

### 1. 伺服驅動器內部的位置比較

假設 STM32 下達：

```text
Target Position = 20000
```

伺服驅動器從編碼器取得：

```text
Actual Position = 19000
```

驅動器內部就可以計算：

```text
Position Error = 20000 - 19000

Position Error = 1000
```

接著，位置控制迴路會根據誤差計算所需的速度命令，再由速度及電流控制迴路調整馬達輸出。

所以伺服驅動器可以：

1. 接收目標位置。
2. 讀取編碼器。
3. 比較目標位置與實際位置。
4. 計算誤差。
5. 調整馬達輸出。
6. 持續重複上述工作。

這些動作通常在非常短的時間內反覆執行。

OMRON 的伺服技術文件有完整說明這種位置、速度、扭力控制迴路的關係。

### 2. 運動控制器也可以比較位置

那麼運動控制器為什麼也能比較？

因為它也可以取得編碼器回報的位置資訊。

例如：

```text
運動控制器

目標位置：20000
實際位置：19000
位置誤差：1000
```

運動控制器可以根據這個誤差，計算下一次應該傳送給伺服驅動器的目標位置。

這種情況下，運動控制器負責外部位置控制，伺服驅動器則負責內部速度及電流控制。

**所以，兩者都能比較位置，但不代表兩者一定同時執行相同的位置控制工作。**

這取決於系統設計。

---

# 四、伺服驅動器內部是不是也有 MCU？

答案是：**有，而且這個觀念非常值得你了解。**

![Image](https://images.openai.com/static-rsc-4/xlQCYrmSeSbSp4_AzumnYC0vcR91JL9Y8LQ8Z-GxUsPLG6_EUufDJ5S9IPQ7sayRFyvYxPG4BVPeMi_UNUxx7mPNeHi-5hiCm26Dlpo9zgFi0F_uTQBibrhr8f_Auh-niiEDZbRpUCizcm4-qUyui4OkRKKIRSkE9CXSN8rxDG-PwCMxCF4VZXCRlK2kVC7S?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wZh7I0lhGB-UOSVtcJWm4c3Vo1QzaYBKCYeIDhdz2Su2iNbhVqDTG0_nrUOEXIiicscGQfeYZxMzq4gzskuIZz4AB0-Iujg-pkOR74AcNQhnvY8G2Rp2C974JS9TthUZBnpUXsn5nPD8wRsTwU0bz32gTrpEA64Asl-PIdWIANW2pUj6uzEEGreCDFHGYXWS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/14V7Q9FXbcD4nIM9-FKdqQrDnjRpxz1hfuf4N9Xc_hsvhZzhtqD7XQ-AgMT1X6GCSMj3v7u86E7pVy769cweDpnkruG2_Qorg7qgQwqki1iDN2aTwnRNlSTGDVo7rpkEO_ZAs2s-dqX50tNGu6r42BPU5VeqJUqBeB4lKE-UxEMXBtmfi1Ox6FmbQ-Us7F0N?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wdr3_113HFUpezw37UD3r8-XgDOx9G1GuvZetXVbXeuPlPeTR2LBalfPYFw68ALa5eRbvSbIJ8RjtjCC2rENl6dAJowFd7rHi-ieV4XjofO8Hnktc9NTIwk-bgVKR9PHKxNEwaS51SCYNNTFJ6gyRxYQZwkPTuZU1Vcrf9fzNldg4zf6tPjm7wV5a02k5zaE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3UADB5ZDolbhp_fKL6i6icmlGPjQp8cJi6ZB2gg4V7glKtas3i4IySQY-R9-Gc6ID0E0PcSR_j4xr2jmfZJMJnP0hQRlLDuq8eIzHtJpRCst1z0wFiwHWqJJjsCQmD66UTIHtUcYT9aNdnqyWtAhRC3_bLYMPfQztztt_3AB0DNrExG7XI-4UuEKRMtkvLpU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RgvJfcKHTdhFeRW5AqQBB8F_C_RUJiShcfSC-cIk1K1mi3swkO9y31i94eGgJqAaxJxDkCGP66zHxi4P_JAAB4Ir2Jd1dvFJ0arui7e98WYcHteVFDdBufOo3F4RqSkGp9XDM6NUBPoY26aL6oVXVqzucb_A3OxPnEgdSP8yyx7uSO-LwMAf21LBqfmt-ur2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/43cWtNkBA-OuhfxjO1bJumT9H65EuPbm0MW_jxpBEIyzZydmiuPUxeMBkJmf8lIraSKAqhdiLCmuvgJDlc9dC-SNybXUYCLAD7HpU4SC_q58hdc9Y15CI3OlutQhFUh-9QsHF1En3ygXj5Rr89IXMjvZn7Z99yp3jCzzLNR175CqSElhgq-3rrUzaFHGk0PL?purpose=fullsize)

一台伺服驅動器內部通常包含以下幾個部分：

| 內部元件                    | 工作                   |
| ----------------------- | -------------------- |
| MCU / DSP               | 執行控制運算               |
| FPGA / ASIC（部分產品）       | 高速訊號處理、編碼器介面或通訊等     |
| Encoder Interface       | 接收及解碼編碼器訊號           |
| Communication Interface | 與 EtherCAT Master 通訊 |
| Power Circuit           | 提供馬達所需的電力            |
| PWM Circuit             | 產生控制馬達的 PWM 訊號       |
| Protection Circuit      | 過電流、過電壓、過溫等保護        |

這裡特別介紹 MCU / DSP。

它們就像你熟悉的 STM32 一樣，可以執行韌體程式。

例如，伺服驅動器內部可能執行：

```text
1. 初始化硬體
2. 初始化編碼器介面
3. 接收目標位置
4. 讀取編碼器位置
5. 計算位置誤差
6. 計算速度命令
7. 計算馬達電流命令
8. 產生 PWM
9. 控制功率電路
10. 監控馬達狀態
11. 偵測異常
12. 重複執行控制迴路
```

這與 STM32 韌體開發的概念其實非常接近。

差別在於，伺服驅動器的韌體必須在非常嚴格的即時性要求下執行控制運算。

例如，TI 的 EtherCAT 雙軸伺服驅動器參考設計，就使用 R5F 處理器執行 CiA 402 與控制功能，並利用其他處理核心處理編碼器及電流回授。([software-dl.ti.com][1])

因此，你可以把伺服驅動器理解為：

**一台專門為馬達控制而設計的嵌入式系統。**

---

# 五、運動控制器內部是不是也有 MCU？

也是有的。

運動控制器同樣是一台具有處理器及控制軟體的設備。

例如：

![Image](https://images.openai.com/static-rsc-4/LS8m1l36Z7NDAlljc19-WpLugooc4B-TWUAr698wKnIcSZz3_aVDr95jmsaRHgYrjOQ8UefFZDIXLGBItg3hXcbGtSq-wNhUkpazmvOiNbs_h_uuMg1MlGLW8J9nwpJtboRi0lHOc8RWZn-0qy3rpE0RuKRRSRtMey2cgTP2m9UXTQ8a8radcfm6t5CKlS56?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LN6nRdpmsuPdXzYZ9cQIi800-cphLDErUBsxJGiyXPZMzCYKbeMai6yBzGNC4sKS51WLt5_k3SvRZ4g21uim0FVeivOdys2VubY86hcBR0typ3f3AFcDYWKkm_88QvmGPKLwxc6IY-YXBkQAT8RNQeZngKCbIDDWHevrgUDC2xqp1h4jrEt0RUlMncaeO6HW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sv7Xah0VO-TunSlouassRkY12MBpq_23bW8D-_ksRam48IRABJeQLpEP39QC-5_NtXUQKP7sBEc-F4qJeVi_m9wMhDg2wM4IcmhAwSlXqkZcdjdmE8ESB1Vws8AomXltQgNJZNUybaHWsK7zVRT1VgPo-CwUysMJk4KA7Fz6TiCJxw9z7rKHzAeLum8co5Xe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/A3QeSp_IQOkyFZU5RguV0sl3QEbtyd_sjfkiRRS3EIz3tbbdKm-8YXHn1t4tYP5JysziqPgL3aP2W4z4wkH8nCCCCnCJDMOHr6fbVpNZtVTuU_QteTYmu0_nDoillLDr5m665M1lWNZEsZAWSOAWgkmJztyDH3ebrrDwUGZRruEaw6vK9lK54GhqHnzyO_aQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5J9fdAOWrtwkC1wIMBMzFI8O2dOU7mHftSCPrChI41lFHTe5zWhumFdBT3C_YoQj8mIa151nQxQDK1seWsssk8ZFTip23awE_N_sBp3kZlWPcXnlMaYjWUT6NuSR5IFLPT5V186XVtADlqqRnB_eVz6KnmctGmdGCCKh1S_X1At3ZnhtE72IKAZ_8G6uioPX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/laxruD6nFfcBeyfHvk8I9zjC3w_YRUvWf9ZKtO1AK9oiFiYSM91OmxxnV3621DSvVKE7KQOgHlX9zJPiiF0CvXp8rwi5QLCCkYgcMDluHSD04Jf4yvc0_puDNqFNko64OLIUbjBViLH0ebrEHQPKUnM00Xt8ACjBvCT283SSNm2QcTMxZIw89JGIsOASOPYt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Pq0tbplwM29_OAdLVMd3BL2zggd4nkbJ0Vf0E5pf9SEHwpYLyCP2ptydRDFa8kcn-gE4mAW5fa1MhuXgczSAi5ymzZxsP6IhNLD3wfOhS8nPlzqcAE7dmVftHsCoP7Y55KQ9DjV55SYnFoJasZ-NdCiJ1Q1cFosB0tc9JFegk48l56nXa7lEr1TKZw17C5uN?purpose=fullsize)

它可能使用：

* ARM 處理器。
* Intel x86 處理器。
* DSP。
* FPGA。
* 多核心處理器。

內部會執行控制軟體、韌體或即時作業系統（RTOS）。

例如：

```text
運動控制器內部

CPU / MCU
   │
   ▼
運動控制軟體
   │
   ├── 軸控制
   ├── 位置計算
   ├── 速度規劃
   ├── 加速度規劃
   ├── 多軸同步
   ├── 位置回授處理
   └── EtherCAT 通訊
```

不過，運動控制器不一定直接連接編碼器。

有些架構是：

```text
Encoder
   │
   ▼
Servo Drive
   │
   ▼
EtherCAT
   │
   ▼
Motion Controller
```

也有些運動控制器可以直接接收外部線性編碼器訊號。

所以要看實際產品設計。

---

# 六、把這些設備放在一起，你就能看懂完整架構了

現在我們把你熟悉的 STM32、EtherCAT，以及剛才學習的運動控制器、伺服驅動器全部放進來。

## 架構 A：STM32 直接當作運動控制主機

這是一種可能的架構。

```text
             Qt 上位機
                 │
                 ▼
             STM32 韌體
                 │
          運動命令及目標位置
                 │
                 ▼
          EtherCAT Master
                 │
                 ▼
            Servo Drive
                 │
           ┌─────┴─────┐
           │           │
           ▼           ▼
        Servo Motor  Encoder
           │           │
           └───────────┘
```

這裡 STM32 負責上層運動命令與設備流程，伺服驅動器負責內部伺服控制。

實際上 STM32 是否也負責軌跡規劃，取決於你的韌體設計。

## 架構 B：使用獨立運動控制器

```text
             Qt 上位機
                 │
                 ▼
          Motion Controller
                 │
                 ▼
             EtherCAT
                 │
                 ▼
            Servo Drive
                 │
                 ▼
            Servo Motor
                 ▲
                 │
              Encoder
```

這種架構中，運動控制器負責軌跡及多軸協調，伺服驅動器負責馬達的即時控制。

## 架構 C：運動控制器與伺服驅動器共同執行控制

例如：

```text
        Motion Controller
               │
               │ 目標位置
               ▼
          Servo Drive
               │
               ├── 位置控制
               ├── 速度控制
               └── 電流控制
                       │
                       ▼
                   Servo Motor
                       ▲
                       │
                    Encoder
```

這是常見的分層控制方式之一。

---

# 七、這與 EtherCAT 的關係是什麼？

你之前接觸過 EtherCAT，所以這裡補充一個很重要的概念。

EtherCAT 本身是通訊技術，不是運動控制演算法。

它負責讓運動控制器與伺服驅動器交換資料。

例如，使用 CiA 402 標準的 EtherCAT 伺服驅動器，可能透過 PDO 傳輸以下資料：

| 資料                    | Index  | 傳輸方向           |
| --------------------- | ------ | -------------- |
| Target Position       | 0x607A | Master → Drive |
| Controlword           | 0x6040 | Master → Drive |
| Position Actual Value | 0x6064 | Drive → Master |
| Velocity Actual Value | 0x606C | Drive → Master |
| Statusword            | 0x6041 | Drive → Master |

也就是：

STM32 可以將目標位置放入 EtherCAT PDO，伺服驅動器收到後執行命令；驅動器再把實際位置放進回傳 PDO，STM32 就能讀取。

這些物件及其資料交換方式由 CiA 402 規範定義，但實際 PDO 配置要看驅動器支援情況。([CAN in Automation][2])

官方參考資料：

* [OMRON 伺服馬達與驅動器技術說明](https://www.ia.omron.com/support/guide/14/introduction.html?utm_source=chatgpt.com)
* [CAN in Automation：CiA 402 標準](https://www.can-cia.org/can-knowledge/cia-402-series-canopen-device-profile-for-drives-and-motion-control?utm_source=chatgpt.com)
* [TI EtherCAT 雙軸伺服驅動器參考設計](https://software-dl.ti.com/processor-industrial-sw/esd/motor_control_sdk/am243x/latest/docs/api_guide_am243x/EXAMPLE_TIDEP_01032_REFERENCE_DESIGN.html?utm_source=chatgpt.com)

---

# 八、這裡有一個你未來很可能會遇到的重要名詞：CSP

因為你目前使用 STM32 與 EtherCAT，所以我先介紹一下，但暫時不深入。

CSP 全名：

**Cyclic Synchronous Position Mode（週期同步位置模式）**

假設 EtherCAT Master 每 1 ms 傳送一次位置命令：

| 時間   | 目標位置 |
| ---- | ---: |
| 0 ms |    0 |
| 1 ms |  100 |
| 2 ms |  250 |
| 3 ms |  450 |
| 4 ms |  700 |
| 5 ms | 1000 |

這些位置數字代表驅動器使用的命令單位。

伺服驅動器會依照週期收到的目標位置，控制馬達追蹤。

這就是為什麼運動控制器需要具備即時運算及通訊能力。

但要注意：CSP 中，運動控制器通常負責週期性產生位置命令，而伺服驅動器通常負責內部位置、速度及電流控制。實際分工仍依驅動器設計而定。

---

# 九、今天的重點整理

我希望你先記住以下五句話：

1. **編碼器是感測元件**：負責測量馬達實際位置。
2. **伺服驅動器是嵌入式控制設備**：內部有處理器、韌體及功率電路。
3. **運動控制器負責運動規劃**：例如多軸同步、速度及加速度規劃。
4. **伺服驅動器通常負責高速馬達控制**：依照回授調整馬達輸出。
5. **EtherCAT 是通訊方式**：負責傳送目標位置及回報實際位置，不是負責馬達控制的演算法本身。

---

# 十、確認一下你是否真正理解

這次我不考編碼器，改考你剛學習的三種設備。

假設有一台六軸矽光子精密對位設備：

* STM32 負責設備控制程式。
* EtherCAT 連接六個伺服驅動器。
* 每個伺服驅動器連接一顆伺服馬達。
* 每顆伺服馬達都有編碼器。
* 設備需要六軸同步移動。

請回答三個問題：

**問題 1：**

如果要計算六個軸應該如何同步移動、各軸應該在什麼時間到達目標位置，這主要是哪個設備的工作？

A. 編碼器
B. 運動控制器
C. 伺服馬達

**問題 2：**

如果要讀取編碼器、執行高速位置控制並調整馬達輸出，這主要是哪個設備的工作？

A. 伺服驅動器
B. Qt
C. 光纖

**問題 3：**

伺服驅動器內部有沒有可能使用 MCU 或 DSP，並且執行自己的韌體程式？

A. 有
B. 沒有

這三題答對後，我們就進入下一課：**步進馬達、伺服馬達、伺服驅動器的完整認識，以及它們在 GMT 精密滑台中的應用。**

[1]: https://software-dl.ti.com/processor-industrial-sw/esd/motor_control_sdk/am243x/latest/docs/api_guide_am243x/EXAMPLE_TIDEP_01032_REFERENCE_DESIGN.html?utm_source=chatgpt.com "AM243x Motor Control SDK: TIDEP-01032 EtherCAT-Connected Single-Chip Dual-Servo Motor Drive Reference Design"
[2]: https://www.can-cia.org/can-knowledge/cia-402-series-canopen-device-profile-for-drives-and-motion-control?utm_source=chatgpt.com "CiA 402: CANopen device profile for drives and motion control: CAN in Automation (CiA)"


---

三題全部答對！🎯

1. 六軸同步計算與運動協調：**B. Motion Controller（運動控制器）** ✅
2. 讀取 Encoder、執行高速位置控制：**A. Servo Drive（伺服驅動器）** ✅
3. Servo Drive 內部可能有 MCU/DSP 並執行韌體：**A. 有** ✅

你的問題非常好，而且正好涉及我們未來要理解 GMT 設備控制架構的核心。

## 一、運動控制器真的比伺服驅動器強大嗎？

不能單純這樣比較。

比較準確的說法是：

**運動控制器負責規劃與協調運動；伺服驅動器負責控制馬達實際運轉。**

兩者的功能不同，不是誰取代誰。

舉個六軸矽光子對位設備的例子：

假設設備需要讓六個平台同時移動：

* X 軸移動 10 mm。
* Y 軸移動 5 mm。
* Z 軸移動 2 mm。
* 其他三軸配合調整角度。

運動控制器需要考慮：

1. 六個軸各自應該移動多少？
2. 六個軸何時開始移動？
3. 每個軸的速度和加速度如何安排？
4. 如何讓六個軸同步到達目標？
5. 如果其中一個軸發生異常，如何處理？

這些是運動控制器擅長的工作。

伺服驅動器則負責：

1. 接收目標位置或速度命令。
2. 讀取馬達 Encoder。
3. 計算控制誤差（依控制架構而定）。
4. 控制馬達電流與輸出扭力。
5. 讓馬達按照命令運轉。

所以，運動控制器處理的是比較高層次的運動規劃與協調；伺服驅動器處理的是馬達控制。

---

## 二、運動控制器會取代 STM32H755 嗎？

**不一定。運動控制器不必然取代 STM32H755。**

這裡要先區分兩種架構。

### 架構 A：STM32H755 自己負責運動控制

例如：

```text
             STM32H755
       (設備控制 + 運動規劃)
                 |
              EtherCAT
                 |
       ┌─────────┼─────────┐
       |         |         |
    Servo     Servo     Servo
    Drive     Drive     Drive
       |         |         |
    Motor     Motor     Motor
       |         |         |
    Encoder   Encoder   Encoder
```

在這種架構：

* STM32H755：負責設備流程、運動命令、多軸協調及 EtherCAT 通訊。
* Servo Drive：負責馬達控制及編碼器回授處理。
* Servo Motor：產生機械動力。

這代表 STM32H755 本身就可能兼任運動控制器的角色。

但要注意：STM32H755 是否真的具備完整的運動規劃及多軸協調能力，要看實際韌體有沒有實作，不能只因為它是 EtherCAT Master 就認定它已經具備這些功能。

### 架構 B：增加一台獨立運動控制器

```text
          STM32H755
       (設備流程控制)
                 |
          通訊介面
                 |
         Motion Controller
          (運動控制器)
                 |
              EtherCAT
                 |
       ┌─────────┼─────────┐
       |         |         |
    Servo     Servo     Servo
    Drive     Drive     Drive
       |         |         |
    Motor     Motor     Motor
```

這種架構把工作分開：

* STM32H755：負責整台設備的流程，例如啟動、停止、初始化、感測器處理、錯誤管理。
* Motion Controller：負責多軸運動規劃、軌跡計算、同步控制。
* Servo Drive：負責各軸馬達的控制。

因此，**增加 Motion Controller 不代表 STM32H755 就沒有用途，而是把部分運動控制工作交給另一台專用設備。**

---

## 三、為什麼有些設備需要獨立運動控制器？

因為不同設備對運動控制的複雜度不同。

| 設備需求      | 可能的控制架構                                          |
| --------- | ------------------------------------------------ |
| 單軸簡單定位    | STM32 + Servo Drive                              |
| 多軸一般定位    | STM32 或 Motion Controller + Servo Drive          |
| 六軸同步運動    | 具備多軸協調能力的控制器 + Servo Drive                       |
| 高精度光學對位   | Motion Controller + Servo Drive，視需求搭配線性編碼器及其他感測器 |
| 複雜軌跡與高速同步 | 專用 Motion Controller + 多個 Servo Drive            |

這不是絕對規則，而是常見的系統設計方向。

例如，六軸設備不一定非得使用獨立 Motion Controller。如果 STM32H755 的硬體、EtherCAT Master、即時作業能力及韌體設計足以滿足需求，也可以由 STM32H755 負責。

---

## 四、回到你目前的 STM32H755 專案

你目前的系統是：

```text
STM32H755
    |
 EtherCAT
    |
 Servo Drive
    |
 Servo Motor
    |
 Ball Screw
    |
 Linear Guide
    |
 Linear Stage
```

從韌體工程師的角度，我希望你先建立這個觀念：

**STM32H755 是設備的控制核心之一；Servo Drive 是馬達的專用控制設備。**

未來我們深入 GMT 精密對位設備時，還會遇到另一個重要問題：

即使馬達 Encoder 回報馬達已經旋轉到正確位置，也不代表滑台一定真的到達精確位置。

這就會牽涉到線性編碼器、全閉迴路控制，以及光學對位精度。

這些都是後續課程的重要內容。

---

## 五、本階段小結

目前你已經理解：

* Encoder：測量馬達或平台的位置。
* Feedback：將實際位置等資訊回傳給控制系統。
* Servo Drive：讀取回授並控制馬達。
* Motion Controller：規劃與協調多軸運動。
* STM32H755：可以擔任設備控制器，也可能兼任運動控制器的一部分功能。
* EtherCAT：負責控制設備之間的即時通訊，本身不是運動控制演算法。

**到這裡，我們可以結束目前的「運動控制器與伺服驅動器」基礎課程。**

下一課將進入 **GMT 精密機械元件與滑台的實際整合**，開始把之前學過的螺桿、滑軌、聯軸器、馬達及回授系統串起來，認識一台 GMT 電動滑台實際是如何構成與運作的。


---

# 第四單元：GMT 精密滑台與定位平台

## 第一課：一台電動滑台是如何組成的？

前面我們已經學過：

* 滾珠螺桿：將旋轉運動轉換成直線運動。
* 線性滑軌：支撐平台並限制移動方向。
* 聯軸器：連接馬達與螺桿，傳遞扭力。
* 馬達：提供旋轉動力。
* Encoder：量測馬達或平台的位置。
* Servo Drive：控制馬達運轉。

今天開始，我們把這些元件組合起來，認識 GMT 真正的產品。

## 一、先看看 GMT 的電動滑台

![Image](https://images.openai.com/static-rsc-4/vqcZZN1IZynaYzZgjf7_iQCl4tJuOkuFtfdzUhmHV-lmcrvW4DBcuurD9TSCEFbv-3dbomXcpKEndUcx5jxyzTBUag9Cb0gTaX3Bl2s472MkSwjWtrVQmjw2uOsbBoZLXPQbiHC_Ew_4aGnm-4tDo8LfbTpgk0mT2_M-3LA7Z67EUWJ8IqkEUkBg8WNYcizN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4ZUlJS_whoEWmRXzvXEvVFzB-woecad6LK_rlgExYRR21s4yNaiOkfPsdQEgh3sXDarYpgkyj-41dBLPgoHTxAKQQUeuTu31msOyThPQcau-zOJR67q9Dk-Y-hlmHjtMdijYbbsfMvUKxeK8EqLCrpco2KzYbM4wW_Zo7WeZ7QdPx1qo5eGeUu1XpurXJPpM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OuMIJf_qbzeJ3tFSEDe9S28BAfV_4E2ny7ZwgwqUjRLwYUecpIM9fg6x22h4XaFXiUIv-p8g0OYQFa2iDucYRzN_Nq2xraDEYu_3RqqTJbky2oiAwq72qUJH1tbXV34-yn2ji2s3yjW0R1txSyw2YDl70snqtP7T9Jxr4ralEljbcR8lD8aMNEfSuApZGecW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/miabZHUOj0Rvr3EVKp2sTiUGVMPcOebwvQW5r5yVtX2zwsg9NmBVwyBdvjpJob3naM5VIVP9QoGTifQNT7d3KoI6Z57yDd5tcbMrf94vrpciM9P1HxUrjnpZbhJC9JW55KaWssXLb8z0862FlVtpLEgtVBZ-TSvfotb9_ZOPvS_eGAzWfmi09Ss2fhI9JA-o?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0G8qfA9syVBwflCkvLYMo2Y2pojOLB1pz28AdT5nFXhoxUqO9MCtwIU5HUVQ3D9vjKCdqcT452kzrHSNHbUyBaIoQuVl4_lnST3_DRGeiuffdMyHrvQsD0xMNN_E7mSgXglR7SFEqKOesm4-yTsxxV-py4Jqu5ETa7TzOzgtjgjbBmuK4m4YnMUANvpra2XO?purpose=fullsize)

這是 GMT 的 CXS 系列精密電動滑台。

它是一個完整的直線定位模組。你可以把它想像成一個能夠精確移動物體的機械平台。

例如：

* 把 Camera 固定在滑台上。
* 把光纖固定在滑台上。
* 把雷射或光學元件固定在滑台上。
* 透過馬達控制平台移動到指定位置。

在光纖耦合設備中，這類平台可以用來調整光纖與光學元件之間的位置。

---

## 二、一台電動滑台有哪些主要構造？

我們先以之前學過的機械元件為基礎，認識六個部分。

| 編號 | 元件   | 英文            | 主要功能      |
| -- | ---- | ------------- | --------- |
| 1  | 基座   | Base          | 支撐整個滑台    |
| 2  | 移動平台 | Stage / Table | 固定需要移動的物體 |
| 3  | 線性滑軌 | Linear Guide  | 支撐並引導平台移動 |
| 4  | 滾珠螺桿 | Ball Screw    | 推動平台前進或後退 |
| 5  | 馬達   | Motor         | 提供旋轉動力    |
| 6  | 聯軸器  | Coupling      | 連接馬達與螺桿   |

我們逐一理解它們在實際產品中的角色。

### 1. 基座（Base）

基座是整個滑台的底部結構。

它的工作是：

* 支撐所有機械元件。
* 固定線性滑軌。
* 提供穩定的安裝基準。
* 固定在設備底板或其他平台上。

你可以把它想成房子的地基。

如果基座不夠穩定，即使螺桿和滑軌本身精度很高，整體定位也可能受到影響。

### 2. 移動平台（Stage / Table）

這是你最容易看到、也最重要的部分之一。

它的上方通常有許多螺絲孔，用來固定被定位的物體。

例如：

* Camera
* 光纖固定座
* 雷射模組
* 光學鏡片固定座
* 其他精密機構

當馬達轉動、螺桿推動時，移動平台就會沿著指定方向前進或後退。

**韌體工程師要注意：移動平台的位置，就是設備真正想控制的位置。**

### 3. 線性滑軌（Linear Guide）

這個元件我們已經學過。

它負責：

* 支撐移動平台。
* 限制平台只能沿指定方向移動。
* 降低移動時的摩擦。
* 抵抗部分側向力量及傾覆力矩。

它本身不是主要的推動來源。

### 4. 滾珠螺桿（Ball Screw）

滾珠螺桿負責將馬達的旋轉轉換成平台的直線移動。

例如：

假設螺桿導程是 1 mm：

| 螺桿旋轉圈數 | 理論直線位移 |
| -----: | -----: |
|    1 圈 |   1 mm |
|    2 圈 |   2 mm |
|    5 圈 |   5 mm |
|   10 圈 |  10 mm |

所以，如果希望平台移動 5 mm，理論上螺桿需要旋轉 5 圈。

實際移動量仍會受到螺桿精度、機械間隙及控制方式等因素影響。

### 5. 馬達（Motor）

馬達負責提供旋轉動力。

這裡要特別提醒：

**GMT 電動滑台並不是全部都使用伺服馬達。**

例如 GMT CXS50 精密型產品資料列出的配置是五相步進馬達，而不是我們前面討論的典型伺服馬達架構。([高明鐵 GMT][1])

因此，看到 GMT 的滑台時，不能直接認定它一定有馬達 Encoder 或 Servo Drive。

必須依照具體型號確認。

### 6. 聯軸器（Coupling）

聯軸器連接馬達軸與螺桿軸。

當馬達轉動時：

1. 馬達軸旋轉。
2. 聯軸器傳遞扭力。
3. 螺桿跟著旋轉。
4. 螺帽產生直線移動。
5. 移動平台跟著移動。

這就是整台滑台的基本運動鏈。

---

## 三、實際案例：GMT CXS50

我們使用 GMT 官方的 CXS50 精密型滑台作為案例。

官方資料列出的規格包括：

| 項目     | CXS50 精密型  |
| ------ | ---------- |
| 台面尺寸   | 50 × 50 mm |
| 移動行程   | 20 / 30 mm |
| 傳動方式   | Ø8 滾珠螺桿    |
| 螺桿導程   | 1 mm       |
| 導引方式   | 線性滾珠導引     |
| 馬達     | 五相步進馬達     |
| 重複定位精度 | ±0.5 µm    |

這些是該型號的產品規格，不代表 GMT 所有滑台都採用相同配置。([高明鐵 GMT][1])

官方產品資料：[GMT CXS50 精密型產品頁面](https://www.gmt.tw/tw/cxs50-%E7%B2%BE%E5%AF%86%E5%9E%8B?utm_source=chatgpt.com)

另外，GMT 的產品型錄網站也可以找到精密型電動滑台、手動滑台、傳動元件等資料。[GMT 官方產品型錄](https://www.gmtglobalinc.com/download?utm_source=chatgpt.com)

---

## 四、從韌體工程師的角度，看一次完整的運動流程

假設我們希望 CXS50 平台向右移動 5 mm。

整個流程可以理解成：

```text
STM32 / 控制器
       |
       | 下達移動 5 mm 命令
       v
馬達驅動器
       |
       | 輸出馬達控制訊號
       v
步進馬達
       |
       | 旋轉
       v
聯軸器
       |
       | 傳遞扭力
       v
滾珠螺桿
       |
       | 螺帽直線移動
       v
移動平台
       |
       | 沿著線性滑軌移動
       v
到達目標位置
```

這裡有一個很重要的觀念：

**控制器下達 5 mm 命令，不代表平台一定真的移動了 5 mm。**

例如，如果使用沒有位置回授的開迴路步進馬達，控制器主要是依照命令輸出脈波，並不一定能直接確認平台實際位置。

如果發生失步，平台可能沒有到達預期位置。

這也是為什麼我們前面花了不少時間學習 Encoder、Feedback 和閉迴路控制。

---

## 五、今天先做一個理解練習

假設現在有一台電動滑台：

* 馬達正常旋轉。
* 聯軸器正常傳遞動力。
* 滾珠螺桿正常旋轉。
* 但是，移動平台與滾珠螺帽之間的固定連接鬆脫了。

請問：

**馬達繼續旋轉時，最可能發生什麼情況？**

A. 移動平台仍然會正常移動 5 mm。

B. 螺桿仍然旋轉，但移動平台可能無法正常跟著移動。

C. 線性滑軌會自動帶動平台移動。

請先回答這一題。

下一步我們會利用實際滑台結構，進一步理解「基座、螺桿、螺帽、滑軌、移動平台」之間如何組合，以及為什麼精密滑台不能只靠一根螺桿就完成精確定位。

[1]: https://www.gmtglobalinc.com/en/x-axis-3/cxs50-precision?utm_source=chatgpt.com "CXS50 - Precision | X-axis | GMT GLOBAL"

---

答對了！✅ **B 正確。**

因為滾珠螺帽與移動平台之間的固定連接鬆脫，螺桿雖然仍然旋轉，卻無法正常將直線推力傳遞給移動平台。

這也讓我們看到一個重要觀念：**機械元件必須正確組合，才能形成完整的運動鏈。**

接下來我們來學習你提出的規格問題。這個部分非常重要，因為未來你不只要看懂 GMT 的產品型錄，也需要看懂同事提供的設備規格，才能知道韌體應該如何控制。

# 第四單元：GMT 精密滑台規格解讀

今天我們以 GMT 官方的 **CXS50 精密型電動滑台**作為教材。

官方規格：
[CXS50 精密型－GMT 官方產品頁面](https://www.gmt.tw/tw/cxs50-%E7%B2%BE%E5%AF%86%E5%9E%8B?utm_source=chatgpt.com)

![Image](https://images.openai.com/static-rsc-4/vqcZZN1IZynaYzZgjf7_iQCl4tJuOkuFtfdzUhmHV-lmcrvW4DBcuurD9TSCEFbv-3dbomXcpKEndUcx5jxyzTBUag9Cb0gTaX3Bl2s472MkSwjWtrVQmjw2uOsbBoZLXPQbiHC_Ew_4aGnm-4tDo8LfbTpgk0mT2_M-3LA7Z67EUWJ8IqkEUkBg8WNYcizN?purpose=fullsize)

先看官方列出的規格。

## 一、基本機械規格

| 規格名稱 |             數值 | 意義               |
| ---- | -------------: | ---------------- |
| 台面尺寸 |     50 × 50 mm | 移動平台上方的尺寸        |
| 移動行程 |     20 / 30 mm | 平台可以移動的總距離       |
| 傳動方式 |        滾珠螺桿 Ø8 | 螺桿直徑為 8 mm       |
| 螺桿導程 |           1 mm | 螺桿旋轉一圈，理論移動 1 mm |
| 軌道   |        線性滾珠導引式 | 使用滾珠線性滑軌         |
| 本體重量 | 0.57 / 0.72 kg | 不同行程版本的重量        |
| 聯軸器  |  FAMCS11.6-5×5 | 馬達與螺桿之間的連接元件     |

資料來源：GMT CXS50 精密型。([高明鐵 GMT][1])

這些規格大多比較容易理解。

我們特別解釋三個。

### 1. 台面尺寸：50 × 50 mm

這代表移動平台上方的尺寸。

也就是說，如果你要把 Camera 固定在上面，就必須考慮 Camera 固定座是否能安裝在這個平台上。

注意：

**50 × 50 mm 不代表滑台的整體外觀尺寸，而是台面的尺寸。**

### 2. 移動行程：20 / 30 mm

這代表平台能夠移動的總距離。

例如：

* 20 mm 行程：平台總共可以移動 20 mm。
* 30 mm 行程：平台總共可以移動 30 mm。

假設我們使用 20 mm 行程的滑台，從一端移動到另一端，最大理論移動距離就是 20 mm。

### 3. 螺桿導程：1 mm

這個我們已經學過。

假設：

* 螺桿旋轉 1 圈：平台理論移動 1 mm。
* 螺桿旋轉 5 圈：平台理論移動 5 mm。
* 螺桿旋轉 10 圈：平台理論移動 10 mm。

所以，這個數值會直接影響馬達轉動與平台位移之間的換算。

---

# 二、精密度規格：最重要、也最容易混淆的部分

這是今天的重點。

GMT CXS50 精密型有以下規格：

| 規格名稱                            |                     數值 |
| ------------------------------- | ---------------------: |
| 解析度（Resolution）                 | 2 µm（Full）/ 1 µm（Half） |
| 最高速度（Max. Speed）                |              20 mm/sec |
| 單向定位精度（Positioning Precision）   |                   5 µm |
| 反復定位精度（Repeatability Precision） |                ±0.5 µm |
| 荷重（Load Capacity）               |                  6 kgf |
| 空轉（Missed Step）                 |                   1 µm |

([高明鐵 GMT][1])

我們逐一說明。

## 1. 解析度（Resolution）

規格：2 µm（Full）/ 1 µm（Half）

這個名稱很重要。

它表示控制系統每輸出一個步進脈波，滑台理論上移動多少距離。

在這個型號：

* Full Step：每個脈波對應 2 µm。
* Half Step：每個脈波對應 1 µm。

先記住：

**解析度代表最小的理論控制移動單位，不代表實際定位精度。**

例如：

如果控制器輸出 100 個 Full Step 脈波：

100 × 2 µm = 200 µm

理論上平台移動 200 µm。

換算：

200 µm = 0.2 mm

這是理論換算，實際移動還會受到機械誤差、負載及步進馬達特性影響。

## 2. 最高速度（Max. Speed）

規格：20 mm/sec

這代表滑台允許的最高移動速度規格。

例如：

如果平台以 20 mm/sec 移動，理論上：

* 1 秒移動 20 mm。
* 0.5 秒移動 10 mm。

但這裡要注意：

GMT 特別標示 Full Step，代表這個速度規格與全步級控制條件有關。

不能直接認為所有控制方式都能達到相同速度。

## 3. 單向定位精度（Positioning Precision）

規格：5 µm

這個規格描述滑台移動到指定位置時的定位精度。

你可以先把它理解成：

**命令要求平台到達某個位置，實際到達的位置可能與目標位置存在偏差。**

例如目標位置是 10 mm，5 µm 就是這個產品標示的定位精度尺度。

不過，精確的測試條件與允差定義仍應以 GMT 的技術規格或測試文件為準，不應自行把它解讀成所有情況下都保證 ±5 µm。

## 4. 反復定位精度（Repeatability Precision）

規格：±0.5 µm

這個規格非常重要。

假設你讓平台反覆回到同一個目標位置：

第一次：10.0000 mm

第二次：10.0003 mm

第三次：9.9998 mm

第四次：10.0002 mm

這些位置彼此很接近，代表重複定位能力不錯。

**反復定位精度主要看每次回到同一位置時，結果是否一致。**

它與單向定位精度不是同一件事。

請特別記住：

* 定位精度：實際位置與目標位置的接近程度。
* 重複定位精度：多次回到同一位置時，結果的一致程度。

這兩個數字不能互相取代。

## 5. 荷重（Load Capacity）

規格：6 kgf

意思是滑台在規定的水平安裝條件下，標示的負載能力。

kgf 是公斤力，不是質量單位 kg。

6 kgf 約等於 58.8 N。

例如，假設你把一個 2 kg 的 Camera 固定在平台上，還需要考慮固定座、其他元件的重量，以及負載重心位置。

**不能只看總重量，也要考慮負載位置與力矩。**

## 6. 空轉（Missed Step）

規格：1 µm

這個名稱容易造成誤解。

在這類步進滑台的規格中，它與移動方向反轉時的機械空行程或間隙有關。

例如：

平台原本向右移動，接著改成向左移動，反向初期可能存在一小段機械間隙。

因此，控制器已經發出反向命令，不代表平台會在同一瞬間開始產生等量的實際位移。

這對精密定位很重要。

---

# 三、其他幾個機械精度規格

GMT 另外列出：

| 規格名稱                        |    數值 | 簡單說明              |
| --------------------------- | ----: | ----------------- |
| 平行度（Parallelism）            | 20 µm | 相關基準面之間的平行程度      |
| 運動真直度（Dynamic Straightness） |  2 µm | 平台移動路徑偏離理想直線的程度   |
| 運動平行度（Dynamic Parallelism）  | 10 µm | 平台移動過程中相對基準面的平行程度 |

這些規格主要描述機械結構與運動軌跡的幾何精度。

對你目前的韌體工作而言，暫時不需要深入計算。

但未來當你接觸 Camera 對位、光纖耦合時，就會發現這些數值非常重要。

因為即使 X 軸位置正確，如果平台移動時產生微小偏擺，光纖與鏡頭之間仍可能發生對位偏差。

---

# 四、從韌體角度看，哪些規格最重要？

我幫你整理成一張表：

| 規格     | 韌體工程師為什麼要知道？   |
| ------ | -------------- |
| 移動行程   | 避免命令超出滑台允許範圍   |
| 螺桿導程   | 計算馬達脈波與移動距離    |
| 解析度    | 計算每個脈波代表的理論位移  |
| 最高速度   | 設定合理的運動速度      |
| 定位精度   | 了解設備的定位能力      |
| 反復定位精度 | 評估重複對位能力       |
| 空轉     | 理解反向移動時可能產生的誤差 |
| 荷重     | 避免負載超出規格       |

另外還有一組電氣規格值得你留意。

CXS50 精密型使用五相步進馬達，官方列有 24V ±10% 電源規格，以及原點與極限感測器。這些會直接影響驅動器接線、GPIO 訊號及韌體初始化流程。([高明鐵 GMT][1])

---

# 五、今天先確認三個觀念

我們先不要一次吸收太多規格。

請回答以下三題：

**問題 1**

CXS50 精密型的解析度為 2 µm（Full Step）。

如果控制器輸出 100 個 Full Step 脈波，平台理論上移動多少？

A. 2 µm
B. 100 µm
C. 200 µm
D. 2 mm

**問題 2**

反復定位精度 ±0.5 µm 代表什麼？

A. 平台每次移動最小只能移動 0.5 µm。
B. 平台多次返回同一目標位置時，位置結果的一致程度。
C. 平台最高速度為 0.5 mm/sec。

**問題 3**

如果滑台的移動行程是 20 mm，是否代表平台可以從起點移動 20 mm，再繼續移動另外 20 mm？

A. 可以，總共可以移動 40 mm。
B. 不可以，20 mm 是整個允許移動範圍的總行程。

答完後，我們再進一步學習 **GMT 滑台的型號編碼**。例如 `CXS5020-S2PN-ND` 裡面的每一段代表什麼，這會讓你以後看到 GMT 型錄時，不必每次都從頭查詢。

[1]: https://www.gmtglobalinc.com/en/x-axis-3/cxs50-precision?utm_source=chatgpt.com "CXS50 - Precision | X-axis | GMT GLOBAL"


---

你的問題已經從「認識機械元件」進入真正的設備運動控制了。尤其是加減速，這是 STM32 韌體工程師很需要理解的部分。

另外，你對課程方向的建議是正確的：**我們不學 GMT 自己定義的型號編碼，而是學習精密設備共通的機械結構、組合方式與運作原理，再逐一認識 GMT 不同產品的特殊設計。** 後續照這個方向進行。

先修正上一題的答案，再回答你的問題。

# 一、移動行程 20 / 30 mm，究竟代表什麼？

你上一題選 A，這題正確答案應該是 **B**。

GMT CXS50 精密型有兩種行程版本：

| 產品版本    | 最大移動行程 |
| ------- | -----: |
| CXS5020 |  20 mm |
| CXS5030 |  30 mm |

也就是說，這不是同一台滑台可以移動 20 mm 或 30 mm 後，再繼續移動，而是兩種不同的產品規格。

例如：

* 選擇 20 mm 版本，平台的總移動範圍就是 20 mm。
* 選擇 30 mm 版本，平台的總移動範圍就是 30 mm。

兩者的螺桿導程都是 1 mm，但機械結構的有效行程不同。([GMT][1])

## 那麼，如何限制最大行程？

這裡有三種不同的保護方式。

![Image](https://images.openai.com/static-rsc-4/1_KjzHMjR9wnqJPvk1wMJhz3UiQXOirggZ5kEqE0luYX-4_RuEM-30G8wID7H551FdbKPtpxdIb1jQvdn-PW2wJuTR2XQ3g1QY0liLRX38lg4ysNFH86r49yQ5UM1yBqNRujQYRi5R8IJnMsCGdjWFagkaC39kD0GMrB8j41STmMErrJuxy4ZC8S5orY20lR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W6jOg6n4zAbqhhmfzTS2WZe6zONDvRla6MlMZpGnNi0aaS7xut9rCBmNPHblzg3Q6bt1Zm_Lf8mTkTHoCsFrgb1NyvqR6vhFaEdhKv23zXf-y2yT-em6t_VPVu7PV7XpvR_ZSv-2kV1jDWpUlSo_kDXgXax6M-38exJeIt15KJXbhJIf5cuOYOMGskdB6e7Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/s3W1p5YccH8HlSrg0M4zHVruJg306vlRmVTcV0TC-8HSOMseeg92dkqalAq80eecMYumtjBsoUweKMS7A4eY2zLESMaV7E5OmEAHEazyz6p4k2HVRUlKv7osFVnAqwt5ys7SV1L2QZG1cJBW-xZ_Rr0P54rpr-JN_L0oDoqObqsGOq5o6q3EqdsWwR_8FEz3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cLwEcJpQs6wmKaGbZ6h16115JeiWW4b3ZGKCj_GpSJGp_ut0m5Dx6ZzV_KBcV-HqSCYNq7M69mWEdrQ5AsLt1s4u9pm1yKiosBHxITattC-DGUWSK_ByXv-B1KTO_dfHzdX99qX6zNunUo6gPaez7dvnWdBWFD-dHXaFvp6h5cLGybRw-CSKG2Zwo4l1PBW6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LE3QYg9sxXLdedX2DePRoXOYtWl6DITA_8NXv6uBEuDvwo6wv-3ZX_XQAa1MG6rso_INX7U0svN8f2WwrFooT0obhO4fNRnmtYFidA95opU6e8IEzcREDU8fcgJemUUQkGUKJclug1Zy-nWw-0t66drT2AyVo04VLSEgNmVOSQWtCRHvuXM8uVyE7DuXAWXF?purpose=fullsize)

### 1. 原點感測器（Origin Sensor / Home Sensor）

它的作用是告訴控制器：

「平台現在已經到達原點位置。」

例如，平台開機後，控制器不知道平台目前在哪裡，就可以執行回原點（Homing）。

當平台觸發原點感測器，控制器便可以建立座標基準。

例如：

```text
原點位置 = 0 mm
```

之後才能以這個位置作為移動座標的參考。

### 2. 極限感測器（Limit Sensor）

它的作用是偵測平台是否到達行程端點附近。

例如：

```text
         允許移動範圍

     0 mm                 20 mm
      |---------------------|
      原點                  極限
```

如果平台移動到極限感測器的位置，控制器就應該停止繼續往該方向移動。

CXS50 官方規格確實列有原點感測器及行程極限感測器。([GMT][1])

### 3. 機械限位（Mechanical Stop）

這是實體機械結構上的限制。

即使感測器失效，機構也可能透過機械限位阻止平台繼續移動。

不過，不能把機械限位當成正常停止方式。若平台高速撞上限位，可能損壞螺桿、滑軌或其他元件。

### 4. 韌體軟體限位（Software Limit）

這個就跟你最有關係了。

假設原點設為 0 mm，滑台行程為 20 mm。

STM32 韌體可以設定：

```cpp
#define SOFT_LIMIT_MIN_MM    0.0
#define SOFT_LIMIT_MAX_MM   20.0
```

當上位命令要求：

```text
Move to 25 mm
```

STM32 就應該拒絕這個命令，因為超出允許行程。

**重要：軟體限位、感測器限位及機械限位是不同層次的保護，不能互相取代。**

---

# 二、最高速度 20 mm/sec，移動 100 mm 都能使用這個速度嗎？

這裡要先分成兩個問題。

### 問題 A：速度是否可以設定為 20 mm/sec？

GMT CXS50 精密型的規格：

最高速度（全步級）＝20 mm/sec。([高明鐵 GMT][2])

這代表它標示的最高速度上限，不代表任何負載、任何加減速條件下都一定能達到。

例如：

| 設定速度      | 是否超過規格上限 |
| --------- | -------- |
| 5 mm/sec  | 沒有       |
| 10 mm/sec | 沒有       |
| 15 mm/sec | 沒有       |
| 20 mm/sec | 到達標示上限   |
| 25 mm/sec | 超過標示上限   |

實際還要考慮馬達扭力、負載、驅動器及加速條件。

### 問題 B：這台滑台能不能移動 100 mm？

不行。

因為 CXS50 最長只有 30 mm 行程。

這裡要區分：

* 速度：每秒移動多少距離。
* 行程：整個機械結構允許移動的總距離。

例如：

一台行程 30 mm、最高速度 20 mm/sec 的滑台，不代表它可以移動 100 mm。

即使速度只有 1 mm/sec，也不能突破機械行程限制。

我們接下來為了學習加減速，先假設有一台**行程至少 100 mm、最高速度 20 mm/sec 的滑台**，來討論運動控制。

---

# 三、為什麼馬達不能一開始就直接以 20 mm/sec 運轉？

這就是你提出的加減速問題。

假設馬達目前靜止。

如果控制器突然要求：

```text
速度 = 20 mm/sec
```

然後立刻輸出對應 20 mm/sec 的步進脈波頻率，會發生什麼事？

對步進馬達而言，這可能造成：

* 馬達無法跟上脈波。
* 馬達失步。
* 馬達振動。
* 負載較大時甚至完全無法啟動。

因此，通常需要加減速控制。

## 1. 加速（Acceleration）

讓馬達從靜止開始，逐漸提高速度。

例如：

|    時間 |      移動速度 |
| ----: | --------: |
|   0 秒 |  0 mm/sec |
| 0.1 秒 |  5 mm/sec |
| 0.2 秒 | 10 mm/sec |
| 0.3 秒 | 15 mm/sec |
| 0.4 秒 | 20 mm/sec |

這就是加速過程。

## 2. 等速（Constant Velocity）

當馬達達到 20 mm/sec 後，就維持這個速度。

例如：

|    時間 |      移動速度 |
| ----: | --------: |
| 0.4 秒 | 20 mm/sec |
| 1.0 秒 | 20 mm/sec |
| 2.0 秒 | 20 mm/sec |
| 3.0 秒 | 20 mm/sec |

## 3. 減速（Deceleration）

當平台快到達目標位置，就開始降低速度。

例如：

|    時間 |      移動速度 |
| ----: | --------: |
| 3.0 秒 | 20 mm/sec |
| 3.1 秒 | 15 mm/sec |
| 3.2 秒 | 10 mm/sec |
| 3.3 秒 |  5 mm/sec |
| 3.4 秒 |  0 mm/sec |

最後馬達停止。

這三個階段合在一起，就形成最常見的**梯形速度曲線（Trapezoidal Velocity Profile）**。

![Image](https://images.openai.com/static-rsc-4/QMzm8cHZcVV8iGwmhJbNicIsFVGi39cN_La4_nvnfgtoiMIbVEzVwVdQuR2FeKNKdmA4ce6k9Ggo_t0jvgWJFTnLUzpykgfRJPPib9zjPOUaUEM2c1R3G2F7hCnrfetdcZSQrueOpjpIM5EE6bM3vox3unHJppF7-KKe8n0EY6uS3Y75l9HxCAXTykk3Ur9K?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wLGM-3WOSUMiNL84-wPbBYX7nTO5OXdFNktijdoh_psBjUZNF831m7dIhVjNftAijaNuyY2WaouTxUzGPNvFGvQPihOy7i8RrlkWKTsV1SApOEo7opisPCKZJC_ekXneb6YkmJvDUDgPiUCtMh87cAJvngDcseZGu5SCgdo-_lHReZaHUpXAR0F8_KT_P_Ft?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RAa4OE9BKXS7h1gwAlsCpZF7N2LkeTwh-bb8M1Mp26D1XTmqc0pPZsADFP-qxhd3TpA6IcletSBcevWn2hKmbi33X2MQk8XB1Bw-hF_r9VY7dXCHp0tgzjleJVtoe7Iq5Ns4Bl9mESTZXdVKOiBYel5ubOXdWKWpOKj6e9EUwBimTlcnOuv7vtN_F5pKxkdv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/D5vFtTearjjfKgf1RBIJPKuUMc29furwxFr69dfQ6_xfsKc0Z_5uU3ixkymqjlthukte8SID2mXL3Aj0Ll3S9ogEeteZRpZhrPaT1aAl3sbp61r9TrOBmRvHqndzI85UhIbCQYcoI5aNL6K8nDA0CV482fG5UHylzw4EnmVsYGJXycb8wkwE9hzUZ9hlWMJt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lTGC16FAgjSlq1mazSgolfVrmlUOu0NilpldgPygLPhI2DzNaMiJDLZPG082uNkRlke_NXKYRZSbqOx4qAwUQO2SFpB7bECSrjboF0yihMRq1Ly1t_HaOxXxurXMsaPaVOb1Axehkn58XGl5e7Ij5ZLk_uRKyb9IewRMd7-IIllbKIl_RFrrTVVDNYVSbVLV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HPC7WF2Ak7nVchZHJmkRiqIfyLDxVcWQEkJsobqUPch7dHtF8CKQYvZbia23TuflW-ypEd871BlJ7aUydt4_Nx_-st7KxneboJTM1hzQBxymzf4ZQKzyKWt0iGi--VeWcNSo2HF_t3lxrObCx_jdonve-1qmnY-_P0jsjD_UcjnTfWbsRjw29tiHgri2Tg6h?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fn2fSgdagDS9_yCdmK-Kzgwn93-OBxuMCt6ZI60mIZ78P_WW49VMwxV9pP1E-yP2fFfoAWdFpUmh9tP8s9OvBrJkmaJVBTB_B2YvvwXu1CruV2p5-LZoBOQ6hi5yvs0Vs_2HzH8WguJskJhlUZnkLzC6ekdkD4PePUJiF4NH0zIwscZPoyE-gQPKCJhPzUO6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fFXIqpunjZvgM8O0L_-oZOO8Ju9l5jzJTOcKXrRYq0h3L7_akZegAi2ahZ_VTLNLQPY74m8NJ99bdrFkltQtAsD_m2eGt7W3AHcWUg5Jr9ikIaRF4CUZ73RuAaw6ZDlMomF3eJ8IH9wB2GZM_n2dPHO-lusRxDDyP2YKh1aXGAITLhKLrpnWnahAk_v9MMqu?purpose=fullsize)

圖形看起來像梯形：

```text
速度
 ^
 |             ┌─────────┐
 |            /           \
 |           /             \
 |          /               \
 |_________/_________________\______> 時間
       加速       等速       減速
```

這就是我們未來在 STM32 韌體中可能會遇到的運動控制概念。

---

# 四、加減速是 STM32H755 韌體工程師自己設計的嗎？

**不一定，要看系統架構與驅動器功能。**

這點很重要。

以步進馬達為例，通常有兩種常見架構。

### 架構 A：STM32 負責產生加減速脈波

```text
STM32H755
   |
   | STEP / DIR 脈波
   |
   v
步進馬達驅動器
   |
   | 控制馬達線圈電流
   |
   v
步進馬達
   |
   v
滾珠螺桿
   |
   v
移動平台
```

在這個架構：

**STM32 必須負責產生正確的脈波頻率變化。**

例如：

* 開始時，脈波頻率較低。
* 加速時，逐漸提高脈波頻率。
* 等速時，維持固定頻率。
* 減速時，逐漸降低脈波頻率。

驅動器主要負責依照 STEP / DIR 訊號控制馬達。

### 架構 B：驅動器內部具備運動控制功能

```text
STM32H755
   |
   | 下達目標位置、速度
   |
   v
智慧型步進驅動器
   |
   | 內部執行加減速
   |
   v
步進馬達
   |
   v
滾珠螺桿
   |
   v
移動平台
```

在這個架構：

STM32 只需要告訴驅動器：

```text
目標位置：100 mm
最高速度：20 mm/sec
加速度：100 mm/sec²
```

驅動器內部就可以自行產生加減速曲線。

所以，**不是所有加減速演算法都一定要由 STM32 韌體工程師撰寫。**

必須先確認驅動器是否具備內部定位及加減速功能。

---

# 五、假設移動 100 mm，加減速應該怎麼計算？

我們用一組簡單的數字來理解。

假設：

| 項目   |          設定 |
| ---- | ----------: |
| 移動距離 |      100 mm |
| 最高速度 |   20 mm/sec |
| 加速度  | 100 mm/sec² |
| 減速度  | 100 mm/sec² |
| 起始速度 |    0 mm/sec |
| 結束速度 |    0 mm/sec |

注意：這是為了教學而假設的數值，不是 GMT CXS50 的廠商設定。

## 1. 計算加速時間

公式：

$$
t=\frac{V}{a}
$$

其中：

* \(V\)：最高速度。
* \(a\)：加速度。

代入：

$$
t=\frac{20}{100}=0.2\ 秒
$$

所以從 0 加速到 20 mm/sec，需要 0.2 秒。

## 2. 計算加速距離

公式：

$$
d=\frac{V^2}{2a}
$$

代入：

$$
d=\frac{20^2}{2\times100}
$$

$$
d=2\ mm
$$

因此，加速階段會移動 2 mm。

減速階段也需要 2 mm。

## 3. 計算等速距離

總距離是 100 mm。

扣除加速和減速距離：

$$
100-2-2=96\ mm
$$

因此：

| 運動階段 |   移動距離 |
| ---- | -----: |
| 加速   |   2 mm |
| 等速   |  96 mm |
| 減速   |   2 mm |
| 合計   | 100 mm |

## 4. 計算總時間

等速階段需要：

$$
t=\frac{96}{20}=4.8\ 秒
$$

加速與減速各 0.2 秒。

所以：

$$
T=0.2+4.8+0.2
$$

$$
\boxed{T=5.2\ 秒}
$$

這代表在這組假設條件下，移動 100 mm 需要約 5.2 秒。

這就是加減速規劃的基本概念。

---

# 六、STM32H755 韌體工程師需要理解到什麼程度？

我希望你能建立以下觀念：

| 工作               | 可能負責的設備             |
| ---------------- | ------------------- |
| 接收上位機移動命令        | STM32               |
| 判斷目標位置是否超出行程     | STM32               |
| 規劃加速、等速、減速       | STM32 或運動控制器或智慧型驅動器 |
| 產生 STEP / DIR 脈波 | STM32 或驅動器          |
| 控制馬達線圈電流         | 步進驅動器               |
| 偵測原點及極限          | 感測器 + 控制器           |
| 回報移動狀態           | 驅動器或控制器             |

未來如果你使用 EtherCAT 伺服系統，架構可能又不一樣，因為運動控制器可能透過 EtherCAT 週期性傳送目標位置，而不是輸出 STEP / DIR 脈波。

所以不能把步進馬達的控制方式直接套用到伺服馬達。

---

## 七、今天先確認一個重要觀念

假設有一台行程 100 mm 的步進滑台：

* 最高速度：20 mm/sec。
* 加速度：100 mm/sec²。
* 目標位置：100 mm。

現在控制器已經讓平台以 20 mm/sec 等速移動。

請問，當平台距離目標位置只剩下 2 mm 時，應該怎麼做？

A. 繼續維持 20 mm/sec，直到抵達目標位置，再立即停止。

B. 開始減速，讓平台逐漸降低速度，最後在目標位置停止。

C. 立即切斷馬達電源，讓平台自然停止。

這題答完，我們下一步會進入**精密滑台的原點、極限感測器、回原點流程與韌體控制邏輯**。這會直接連結到你目前正在使用的 STM32H755 韌體開發工作。

[1]: https://www.gmt.tw/tw/cxs50-%E7%B2%BE%E5%AF%86%E5%9E%8B?utm_source=chatgpt.com "GMT GLOBAL INC-CXS50 - 精密型"
[2]: https://www.gmtglobalinc.com/en/x-axis-3/cxs50-precision?utm_source=chatgpt.com "CXS50 - Precision | X-axis | GMT GLOBAL"

---

答對了！✅ **B 正確。**

當平台距離目標位置只剩 2 mm 時，應該依照預先規劃的減速曲線降低速度，讓平台平順地停在目標位置，而不是突然停止。

接下來進入今天的課程。這一課非常重要，因為它會直接連結到你未來撰寫 STM32H755 韌體時會遇到的 GPIO 輸入、感測器判斷、座標初始化及安全保護。

# 第四單元：精密滑台

## 第二課：原點感測器、極限感測器與回原點流程

今天我們要回答四個問題：

1. 滑台開機時，怎麼知道自己目前的位置？
2. 原點感測器與極限感測器有什麼不同？
3. 滑台如何執行回原點（Homing）？
4. STM32 韌體應該如何控制這個流程？

---

# 一、為什麼滑台需要原點感測器？

先想像一個情境。

你有一台電動滑台，行程 20 mm。

滑台的移動範圍：

```text
       移動平台
          |
          v
  0 mm ---------------- 20 mm
  原點                    行程終點
```

現在你關閉設備電源，再重新開機。

請問 STM32 知道平台現在位於哪裡嗎？

**不一定知道。**

因為如果沒有絕對位置回授，STM32 可能只知道上一次下達了多少移動命令，卻不知道平台斷電期間是否被人手動移動過。

例如：

* 關機前平台位於 10 mm。
* 關機後有人用手推動平台。
* 重新開機後，平台實際上已經位於 15 mm。

如果 STM32 仍然認為平台在 10 mm，就會產生座標錯誤。

因此，設備通常需要執行：

**回原點（Homing）**

也就是讓平台移動到指定的機械參考位置，重新建立座標基準。

---

# 二、原點感測器（Origin Sensor）

![Image](https://images.openai.com/static-rsc-4/baFqVS4awmIGB55obPoVABz0CNhl1LUhxwGtHxmDaie9NTb_wNfY6BtlgUs_vGX8GBz6GGABNiwD2oNPK2pVgOVQZ79wGiXYq0DJdlMHvRjl79HkV6qUivZ33p7avRo2LR3QA_bZ4HatBmr-cCfmO1UWxI_-I_lTQBtbapJ3WE_qAwUNne_R35t_gnhAhKNL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pUlg3uVj-9YSA7LyTfmr9SzLzcV78lf4vpqOJyBk0YWwGGIuRKy0UWedpbcVgOhViE4OAo8ej4aAYXoou1RlnheDd4c-lVz4h88ipNMFqW7BXyHsDUmEzFDiy_tXT3fUbLIt121rx4dWeTltVKz2ejO_zJkP4EBaHZmskr1vJx7wg8Bmk-DvmT5vFOTuwdrj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Wi1wmqcGtu-EY5emHsIMt4CsjiwTxA7FlGMoqaXlJ0NK-Mq4jqXtxL0d8UvvUlQpDZ3Utyv3P5wqlBasX3jU30PQ89CICOK4HuPP6UH7pUVF8wFfsKcwewok927YEaqlzeE2GitGBOZcKXdIkMjDu0frLFplOFZyQkz7oOHZdUgYYs9DMA-cljZvYzlhPX6u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tgBADN8Xk92BYBV_4A4ygwjfZnrPM9lFueq4SmCpJFIgsxMOAhchq0ulnVH9mO_AoAhw1ebIDNoKJ7QhaSlrvukKzu2fyg8I2CIkYFHoUsqYYMV-b8r2qt_EvZodQvAPENf-ol5O7PeowjrstZW66uWShzUmy1NYhyCBZ0bDn5IInwf_amX_5cMadn62mLyg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/h6zE_8I348G4PcWsWUKMcfzPJunAFcw2W9sp5YsT8UZx6ym76DA-DL1adFfYJgzOBSH-jBVTSXiKmscJh_GH3BCXwukhykB7r6LXuSy1S4t7kJIanDOZ0kDnJ0SC9dYvDjejI72loowBGU6KvnV-7BHsW1oTsCz7xc-DnYyS0G3555f07b1qY2iCgu8uM5Le?purpose=fullsize)

原點感測器的工作是：

**偵測移動平台是否到達預先設定的原點位置。**

以 GMT CXS50 精密型為例，官方規格列出：

* 原點感測器：光電感測器（GMT Sensor）。
* 電源電壓：24V ±10%。
* 控制輸出：NPN 開放式集電極，24V、8mA 以下。

([GMT][1])

## 光電感測器如何工作？

你可以想像感測器有一個光線通道：

```text
       光電感測器
      ┌─────────┐
      │ 發射端  │
      │   --->  │ 接收端
      └─────────┘
             ^
             |
        遮光片經過
```

當平台移動時，固定在移動部件上的遮光片會經過感測器。

感測器偵測到遮光片，就會改變輸出訊號。

這個訊號傳送給控制器，告訴控制器：

「平台已經到達原點感測區域。」

要注意：感測器偵測到的位置不一定就是最終使用的座標零點。實際系統可以再透過減速接近、位置偏移等方式建立更精確的原點。

---

# 三、極限感測器（Limit Sensor）

![Image](https://images.openai.com/static-rsc-4/4su80fygWQc13YdjZwKmjbuXKZz1Nawe4uj22ZqqJTwPmLRvGWgfBrZpWhxYErP8oIpKLLeI1nExqrcZj3QAsnac6DVZ32yfvsUY44Rde2q1vrygZk8Gqq3TFGkiTZ5VB0rcqFX4kEAOxdW9MGx_Vx-2I10AvvZRe-eyFGHsqy6AOFyq7wzhQFqjR5DpRss_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W6jOg6n4zAbqhhmfzTS2WZe6zONDvRla6MlMZpGnNi0aaS7xut9rCBmNPHblzg3Q6bt1Zm_Lf8mTkTHoCsFrgb1NyvqR6vhFaEdhKv23zXf-y2yT-em6t_VPVu7PV7XpvR_ZSv-2kV1jDWpUlSo_kDXgXax6M-38exJeIt15KJXbhJIf5cuOYOMGskdB6e7Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sng1gtaFIKPpU0eFpuszy2QVySvDTtRQn8cgrDEuahU1jzyyh2LkIhLJ6o7Oaj5k-QSbrBFuSGH6DrUbxfv99dPxjsxhZpGsCgdUZTvyfUMIlYYr_pItDF6-GIDqeto3DBQqmA7i1O6AKge4SvZMaAU6yUl9GVqrH6R7r-svhJs7bZ4CNas9Xzn6s5BfVI6n?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LE3QYg9sxXLdedX2DePRoXOYtWl6DITA_8NXv6uBEuDvwo6wv-3ZX_XQAa1MG6rso_INX7U0svN8f2WwrFooT0obhO4fNRnmtYFidA95opU6e8IEzcREDU8fcgJemUUQkGUKJclug1Zy-nWw-0t66drT2AyVo04VLSEgNmVOSQWtCRHvuXM8uVyE7DuXAWXF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2V6ZQKj6QMHJcJ2eoOKnhQt8OtM3OLM9wkx2B4TjkhQSQdCmO8ffLy0XCqyKKfoZXV7VpWim5v0MR2I3dojwHWEAFEZEyeVzNjlxt4eo2n-CzRIG9gHrf7S3MsF1wyowl_F4holsJAUU9QiGFWy17Otb_kkJSsIHolqy_hvv5N9r1ILYJMCZk3C-mAdpUEyS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/baFqVS4awmIGB55obPoVABz0CNhl1LUhxwGtHxmDaie9NTb_wNfY6BtlgUs_vGX8GBz6GGABNiwD2oNPK2pVgOVQZ79wGiXYq0DJdlMHvRjl79HkV6qUivZ33p7avRo2LR3QA_bZ4HatBmr-cCfmO1UWxI_-I_lTQBtbapJ3WE_qAwUNne_R35t_gnhAhKNL?purpose=fullsize)

極限感測器與原點感測器的用途不同。

原點感測器：

> 告訴控制器平台到達參考位置。

極限感測器：

> 告訴控制器平台已經接近或到達行程邊界，需要阻止它繼續往危險方向移動。

例如：

```text
負方向                                         正方向

[負極限] -------- [原點] ---------------- [正極限]
   |                                         |
   |<----------- 允許移動範圍 -------------->|
```

假設平台目前正在往正方向移動。

當正極限感測器觸發時，控制器應該：

1. 停止往正方向移動。
2. 記錄極限異常或狀態。
3. 禁止繼續向正方向移動。
4. 視系統設計，允許往反方向退出極限區域。

這裡有一個重要細節：

**觸發正極限，不代表必須禁止所有方向的移動。**

因為有時候必須允許平台往反方向退回安全範圍。

另外，極限感測器不是唯一的安全措施。韌體軟體限位及機械限位也很重要。

---

# 四、原點與極限感測器的差異

整理一下：

| 比較項目     | 原點感測器                       | 極限感測器        |
| -------- | --------------------------- | ------------ |
| 英文       | Origin Sensor / Home Sensor | Limit Sensor |
| 主要用途     | 建立座標參考                      | 防止超出行程       |
| 觸發時機     | 回原點時                        | 接近行程邊界時      |
| 主要目的     | 確認位置基準                      | 保護機構         |
| 是否用於建立零點 | 是                           | 通常不是         |

這兩種感測器可以同時安裝在同一台滑台上。

---

# 五、STM32 如何讀取感測器訊號？

這是今天最重要的韌體知識。

我們先以 GMT CXS50 為例。

官方規格指出：

| 項目    | 規格                 |
| ----- | ------------------ |
| 感測器電源 | 24V ±10%           |
| 輸出方式  | NPN Open Collector |
| 輸出電流  | 8mA 以下             |
| 感測時輸出 | 電晶體 OFF（非導通）       |

([GMT][1])

這代表它不是一般直接輸出 3.3V 的 STM32 GPIO 感測器。

**不能把 24V 感測器訊號直接接到 STM32 GPIO！**

必須透過適當的介面電路，例如：

* 光耦合隔離電路。
* 電壓轉換電路。
* 適合該感測器輸出的輸入介面。

而且要根據實際接線圖確認訊號的有效電位。

## STM32 韌體看到什麼？

假設已經透過適當介面，把感測器訊號轉換成 STM32 可讀取的 3.3V GPIO 訊號。

韌體就可以讀取：

```cpp
HAL_GPIO_ReadPin(HOME_GPIO_Port, HOME_Pin);
```

假設我們把訊號定義成：

```text
GPIO HIGH = 沒有觸發
GPIO LOW  = 感測器觸發
```

注意：這只是教學用的假設邏輯，實際 HIGH/LOW 必須依照電路設計確認。

---

# 六、真正的重點：回原點流程（Homing Sequence）

現在我們把前面學過的東西組合起來。

假設滑台目前不知道自己在哪裡。

STM32 要怎麼讓它找到原點？

常見的回原點流程可以分成以下幾個階段。

![Image](https://images.openai.com/static-rsc-4/rfVvZ9fkurrHpPOJJRlepnOrUI7L0J-EGwe3W7jJbl3NtB5O08tC_O6Vz1hhwWiZFv_42HEsdVRkPmkShEGqIP38NsxZjWo5ArtP0ADQ8PtZAoKHYFYhBMiDgvlS5DQUJZTw2-3HntI6OrPvKhWHDbQLixtN26Wj5hD-R5QdB7Jr4TxFzt7TsenCTm6hEEnn?purpose=fullsize)

### Step 1：確認目前狀態

STM32 先檢查：

* 驅動器是否正常？
* 馬達是否可以啟動？
* 極限感測器是否已經觸發？
* 原點感測器目前是什麼狀態？

如果設備存在異常，就不應該直接開始回原點。

### Step 2：快速尋找原點

STM32 命令馬達朝原點方向移動。

例如：

```text
速度：10 mm/sec
方向：負方向
```

平台持續移動，直到偵測到原點感測器。

### Step 3：減速或停止

當感測器觸發時，控制器依照回原點策略減速或停止。

為什麼不能直接把感測器觸發的位置當作非常精確的零點？

因為感測器有偵測區域，而且觸發位置可能存在誤差。

### Step 4：離開感測區域

平台反方向移動一小段距離，讓感測器解除觸發。

例如：

```text
反方向移動 0.5 mm
```

### Step 5：低速重新接近原點

接著以較低速度重新接近感測器。

例如：

```text
速度：1 mm/sec
```

這樣可以降低高速移動時的停止誤差，讓原點判定更加穩定。

### Step 6：建立座標零點

當控制器再次偵測到原點位置，就建立座標參考。

例如：

```text
Home Position = 0 mm
```

之後，STM32 就能使用這個參考位置進行相對或絕對位置控制。

補充：以上是常見的回原點策略示例，並不是所有 GMT 滑台都必須使用完全相同的流程。實際順序、速度、方向及是否使用兩次接近，取決於驅動器和控制器的設計。

---

# 七、回原點完成後，STM32 如何保護行程？

假設滑台行程是 20 mm。

回原點成功後，STM32 建立：

```cpp
#define SOFT_LIMIT_MIN_MM  0.0f
#define SOFT_LIMIT_MAX_MM 20.0f
```

現在上位機傳送：

```text
MOVE 15.0
```

STM32 檢查：

```text
15.0 是否介於 0 和 20 之間？
```

是，所以允許移動。

但如果上位機傳送：

```text
MOVE 25.0
```

STM32 就應該拒絕。

因為：

```text
25.0 > 20.0
```

這就是軟體限位。

實際產品還需要預留安全距離，不能把軟體上限直接設在機械極限的位置。

---

# 八、今天從韌體角度整理整個流程

```text
           STM32H755
               |
               v
         讀取感測器狀態
               |
               v
          執行回原點
               |
               v
        尋找 Origin Sensor
               |
               v
        確認原點位置
               |
               v
        建立座標零點
               |
               v
          設定軟體限位
               |
               v
          接受移動命令
               |
               v
       持續監控 Limit Sensor
               |
               v
          正常運動控制
```

你可以看到，原點感測器不只是單純的 GPIO 輸入。

它是整個定位系統建立座標基準的重要元件。

---

# 九、今天的練習

我們先確認三個觀念。

**問題 1**

設備重新開機後，STM32 不知道平台目前的絕對位置。通常應該先執行什麼動作？

A. 直接移動到 10 mm。

B. 執行回原點（Homing）。

C. 直接把目前位置設定成 0 mm。

**問題 2**

假設平台正在往正方向移動，正極限感測器觸發了。正常情況下，控制器應該怎麼處理？

A. 繼續往正方向移動。

B. 停止往正方向移動，並依照安全策略允許往反方向退出。

C. 立即關閉整台設備的所有電源。

**問題 3**

GMT CXS50 的原點感測器輸出為 NPN Open Collector，電源是 24V。STM32H755 的 GPIO 是 3.3V 邏輯。

可以把感測器輸出直接接到 STM32 GPIO 嗎？

A. 可以。

B. 不可以，必須先經過適當的訊號介面電路。

答完後，我們下一課會介紹**精密滑台的多軸組合：XY、XZ、XYZ、旋轉軸及六軸平台**。

這會開始讓你理解，GMT 為什麼有單軸滑台、XY 滑台、XYZ 滑台，以及光纖耦合設備為什麼需要多軸精密定位。

[1]: https://www.gmt.tw/en/cxs50-precision?utm_source=chatgpt.com "GMT GLOBAL INC-CXS50 - Precision"

```
答案：
B. 執行回原點（Homing）。
B. 停止往正方向移動，並依照安全策略允許往反方向退出。
B. 不可以，必須先經過適當的訊號介面電路。
```

---

你問的這些問題非常重要，因為這正好連結到我們熟悉的 STM32 韌體：**感測器如何把機械位置轉換成電氣訊號，再讓 STM32 知道滑台目前的位置狀態。**

先確認你前面三題都答對了。現在我們先把感測器的硬體與訊號弄清楚，再進入多軸滑台課程。

# 一、原點感測器與極限感測器，輸出是類比訊號嗎？

一般來說，**不是類比訊號，而是數位開關訊號（Digital Signal）。**

它們主要告訴控制器兩種狀態：

* 尚未感測到目標（OFF）。
* 已經感測到目標（ON）。

例如：

| 感測器狀態     | 控制器收到的資訊 |
| --------- | -------- |
| 滑台尚未到達原點  | 未觸發      |
| 滑台到達原點位置  | 已觸發      |
| 滑台尚未到達正極限 | 未觸發      |
| 滑台到達正極限   | 已觸發      |

它不是告訴 STM32：「目前位置是 3.25 mm」。

它只告訴 STM32：「我現在有沒有偵測到指定的物體或位置」。

所以它通常不需要使用 STM32 的 ADC。

---

# 二、感測器輸出是電壓嗎？有正負電壓嗎？

這裡要先區分兩件事：

1. 感測器的電源。
2. 感測器的訊號輸出。

以 GMT CXS50 精密型為例，官方規格列出：

| 項目      | 規格                         |
| ------- | -------------------------- |
| 感測器類型   | 光電感測器 GMT-sensor           |
| 電源電壓    | 24V ±10%                   |
| 輸出形式    | NPN Open Collector         |
| 輸出電流    | 8mA 以下                     |
| 感測時輸出狀態 | Output Transistor OFF（非導通） |

資料來源：([高明鐵 GMT][1])

這代表它使用的是 **24V DC 電源**。

不是正負 24V 的雙電源，也不是輸出正負變化的類比訊號。

一般工業感測器的電源概念如下：

* +24V：電源正端。
* 0V：電源負端，也就是電路參考地。
* OUT：訊號輸出。

注意：0V 不代表一定要接到大地（Earth），兩者是不同概念。

## NPN Open Collector 是什麼？

這是你需要理解的重點。

NPN Open Collector 可以想成感測器內部有一個電子開關。

![Image](https://images.openai.com/static-rsc-4/r0D7vHr9ljOz7mWLTkqjfL_mi_7fMzWKSxSqd6R-2gllbvabqyWwGAiMKhRSmZluUd0_yiX6NRKLLGD6Bs8l72aWlHS6eDpG8VKvr-9o7xYxx1YliJSe8veqTPopZwQs-PMdyF-yx4FXGbVgqMeEVJDOyKyYpr8yZT2hwGAphsTRsNAEMpgSroNQo5zzXDxw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TjN4ZaVDFuf5iUxoxMuBH0lFYTzIDKxXgn-4ON4bBI90Al6C56f198o2JMY0umnO04c_KIWN8n8g7e_UKFvvT42ajE6L4LVMZC0MTa5s4xRneBWJ6aGj3D-FPExMJ70VNuGXHbz2UOOvGuNpBWQ9vZgnFc9RGny0O3rd20bne3ius93h1pnPpezYTvxaQrJe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sWPCAjbJjHK5vOliI-emLURuWazhTfKhypBefZr-PA-tVxgQ0lhV4R9-wDXO6IQuhqkREz0YIBQIFYD8_qAZ_TCzyHYeMcBT5fna02kipOMnwGgOtVdlzin7xlh4fPnduSkphQs2b3jkVxj8HG7JBo8zA-ilNUvg-pkBjtrl2NGY4Z6LG-fiiUqf2IgaYapq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/B7dkvc9qCA0HA0xVM26Ez8OB7z0JZfFwfqQJbcX-Z2HycAHXQu3TI0KVS1f3PnQi76nhDY32pg5uDdR4chOFrKlZ0cEBW0H4pPiWuBVnn-ho4jWFD_i8Dhg5ha6Ntvk1fKoYDXz1NRdORirRQzHVRsLNR6ffbDv1GC7dArrK6GioQDOW-ZlsODMYXrZLZuib?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/a38S3xgscMLRkXucVbSdYBmy1wI2sUkngC-PPbt-GyDE2IOFvY0_QLs24QRR1wP8g9YKB3xu-M6kdqHofh9EVgFNp6CfrkXpYcQy5gyYkoU7v2F7tMlf6LHLq29CjcwnZhch3H3lXZPT0sfXUcyz-uJgpmrUwEankTyWPJC-qkSefyaJk8jZ84z7csE7I14T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AygRdRJP_eSf3ppmFNCgwDyN6yKLgD0g-W5VOxZ7VPTYUpNiQQkDVaRh_V7mYbRu9fRNow7rvE3Oqz_UYUFiFo4PK3uchfAZkT_UiyqoF6O_XEVXR9TC_QRSGAfVBKqTd07mqtDCaBHLHm29CPEztBef5tsPLdzRqmnac4zwSWsr32uv59Eks506Un5jae9D?purpose=fullsize)

它的輸出不是主動提供固定的 24V，而是透過內部電晶體控制輸出端是否與 0V 導通。

假設外部有適當的上拉電阻：

| 感測器內部電晶體 | OUT 狀態  | 外部讀取結果 |
| -------- | ------- | ------ |
| ON（導通）   | 被拉向 0V  | LOW    |
| OFF（非導通） | 由上拉電阻拉高 | HIGH   |

這裡有一個特別值得注意的地方：

**GMT CXS50 規格指出，感測器進入感測狀態時，輸出電晶體是 OFF（非導通）。**

因此，不能直接套用一般感測器「感測到物體就輸出 LOW」的直覺。實際讀到 HIGH 或 LOW，還要看外部介面電路如何設計。([高明鐵 GMT][1])

---

# 三、感測器有幾條線？

常見的工業感測器有三線式：

![Image](https://images.openai.com/static-rsc-4/TjN4ZaVDFuf5iUxoxMuBH0lFYTzIDKxXgn-4ON4bBI90Al6C56f198o2JMY0umnO04c_KIWN8n8g7e_UKFvvT42ajE6L4LVMZC0MTa5s4xRneBWJ6aGj3D-FPExMJ70VNuGXHbz2UOOvGuNpBWQ9vZgnFc9RGny0O3rd20bne3ius93h1pnPpezYTvxaQrJe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jY1FZFOyCX8xGUL7gduby389exjdvcvGscVLKoJ8HToyyO2Vf1HUCv37mGnD3zf7z9mulT17lK9p7fR2dKGnfx4bcQA6NujTldj6TEAulyhpO5akleIlVApjOVnFwCc1nhhwR2K1HOC5P-s9Wda7_9INgPl28jEwGQf2NuuegtV21xOLl_jomrXcOxX3f-8j?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QM_Hbi8YANTXAcgGANiCYIp5r6Yyw0VhBJYbg0PschnYqdVs93kQBYaWh-xKFha7hnJsMH8Sjt9eONRtKwFp6mz4hu-cAFobVxoHqZv2zJlLOTJR-k4qutjHriZ_4l-oyFbAanpxsW8f59QPkUP3zGdH4d5yflKSKCbK7b5ZJ0JDx_b3c79Md89igsmVkjZ-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/54fGD9ZkLDASBo5Zs07SaffGxKjn8EMf86oxi5lO4Kl-ivZXzLE88n4Nd47H7SOIbKuxZOWCP14UMP1p1LHIkDuTbLyHH6DiBVfuzgd-Gvtfe-shz4X_wuWjFWYeSlyPP2iZKYtnYEb90QLyb-CgD9AgX9-DLKe_3i_Vdl9wxBzqLWzbNPZ27K-MgHxiWjlt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NTUdy1nNaX50QPJx-iFXCWpZt7GywSjH8iv-o0ak205s2TVeePtEL5ad2Zea0tPjN0pUhy2UFM_jGh_Obhq59Fvwq0YzvNmbD6LKybnE3kg45BOnYStMFg-4J9qMXJ8VWSqMJK2S2pGomVB84fbO5uU-Sf2BH7O0_0hcp3ZDBNnbWuJr5zMkgVVSupIAxeUd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OInI28mvbEAnd1KGgP8E9xfM6y7ZzqER1jV68QuWwb1XEAx3bHepRssdiF_rnSjC2RxliEIuM499s2Wi5dcLG25XOSoWZLYLHH0v-etpXwsivbnioVDfJW3eyVJ9GO6V5AfTlh86PmTJaVwNM3z4_9ewJZsShkEGTjJdg4IDzcaHrdd1a07r69KonZ7YVNFV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GJ3uqE5oOsCBhYoIM3JnE2nFci6ExQiUOkcVlb9_QxDvqZd_tbCjRBYDczJbzaDG3XwqL_pV1BXYHf1l7OMfvryywKtTJvJJoxTdKw2VfNH62mDVuDFJSM3azZer9PPjWWU5N-USRS-t5jn0exnqFnBtfTTxVBg3Wb9N7T23S9JDgYeGa26wZUESAC6aHkHD?purpose=fullsize)

通常是：

| 線色（常見慣例） | 功能       |
| -------- | -------- |
| 棕色 Brown | +24V     |
| 藍色 Blue  | 0V       |
| 黑色 Black | 訊號輸出 OUT |

這是常見的三線式配置，但**不能因此認定所有 GMT 感測器都採用這三種顏色或相同接腳**。

另外也有：

* 兩線式感測器。
* 三線式感測器。
* 四線式感測器（可能同時提供 NO、NC 輸出）。
* 使用多芯連接器的整合式感測器。

以 CXS50 精密型來說，GMT 官方列出的滑台側接頭是 15-pin D-SUB 或 12-pin HRS。也就是說，滑台可能透過多芯接頭，把馬達及感測器訊號整合在同一個連接器內。不能把它簡化成只有三條外露電線。([高明鐵 GMT][1])

---

# 四、感測器要直接接 STM32H755 ADC 嗎？

**不需要，也不應該直接接到 ADC。**

原因有兩個：

第一，這是數位開關訊號，不是需要 ADC 轉換的類比訊號。

第二，GMT CXS50 使用 24V 感測器，而 STM32H755 GPIO 的邏輯電壓是 3.3V。

如果把 24V 訊號直接接到 STM32 GPIO，可能損壞 MCU。

正確的概念是：

![Image](https://images.openai.com/static-rsc-4/bl38OtJbpaqfkkPcDcgY_uWgCencVpCHkOA6aa8amlK3ipRTPafTo7WcKto1U_IY6k1PqsdJaOJrjsN2rYVg2WXQELQXu6UeMa0dDqODzl83fXi3UbRvgBGYQ603E1ZVAeaMXSccf221lcKvtR4-4kT7iTNuV7HKBe4NXLorEml-X99IybqEi0v2JPc9nzCw?purpose=fullsize)

訊號傳遞流程：

```text
GMT 原點感測器
       │
       │ 24V NPN Open Collector
       ▼
訊號介面電路
       │
       │ 轉換、隔離或電平匹配
       ▼
STM32H755 GPIO
       │
       ▼
韌體讀取 HIGH / LOW
       │
       ▼
判斷是否觸發原點
```

訊號介面可能使用：

* 光耦合器（Optocoupler）。
* 電晶體電平轉換電路。
* 工業數位輸入介面。

實際選擇要考慮 NPN 輸出特性、上拉方式、電流、電壓及隔離需求。

這裡也可以連結到你熟悉的 STM32：

```cpp
if (HAL_GPIO_ReadPin(HOME_GPIO_Port, HOME_Pin) == GPIO_PIN_SET)
{
    // 偵測到某一種輸入狀態
}
```

但這段程式還不能直接認定 HIGH 就代表觸發，因為必須先確認硬體介面及訊號有效邏輯。

---

# 五、感測器只有光遮斷型式嗎？

不是。

常見的工業位置感測器有以下幾種：

![Image](https://images.openai.com/static-rsc-4/6o7VrXToGXKPza-Azfh3VxvN_Of3PfPlgmlo1MZv3B5FFl8NxWczPa94qH5kJ39pFIeXwaALqv2CiY1R2tpLjoJlOd82yk4ppIa9IH2kkMSm2c0rllZKRExD_R-dGxs32PLtSPDmTya1LF9amk8ovgOzxLED2_4x5coUbVJWx8BeDPVE4Tz5tpkfHBxegVRs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vOiGYdF5ggPWjmjiUJyxbPDHTSBdZLzwp7QhPWYRwZFkl10j_cPJ3uOewVF5NDKeT8ju3Zq2NWYKCrbRDsOd9uv5g2ouiVh-GrkUEnUd_zNALOrWTchrVpCb4VgrHQY9iCATShAf8TtaeiIAXxoUdKkORKFlRjeKK-sCDuVgHArhU28ODYsCv4Nepw3VP4IH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LJtnAfGxqkNg9cXTtk55hF3MPbVou2wo72Hk8mEaciuSQViSqYqOId4XdEdGXDIciRAhp1peu0CVfcfLsFy0gfscYDBpYTIIDZPTRvvL7AtOwgO7Cx3Pv6CU1VQBtd96g5lwxReKihe7wNSTnANwK4xZYL4j96GTlwMYWCUnpebom77lgahiAT4uSP4Lg0ey?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VRr3dv243_mNeHDqgeiXZRTOYXPHp68cgsUZv2LfzIqB4v2OWZoT-oqqnAkomWxmCao3jfkMfutZUoWktXDNpMkvk_c9x7AmK1XyW7Y_a9G9j8OJHmXmdUC3DyCHcqhqLag2GL9A6bukXwh11R2mbcJhDhn21XdeyMaiiYHyIyjLECIFlMEqaap4CEtlNwig?purpose=fullsize)

| 類型       | 偵測原理     | 常見用途      |
| -------- | -------- | --------- |
| 光電感測器    | 利用光線偵測物體 | 滑台原點、極限位置 |
| 電感式接近感測器 | 偵測金屬物體   | 金屬滑台的位置偵測 |
| 機械式限位開關  | 物體碰觸開關   | 行程極限保護    |
| 磁性感測器    | 偵測磁場     | 氣缸位置偵測    |

不同感測器有不同優缺點。

例如：

* 光電感測器：不一定需要接觸物體，適合精密機構位置偵測。
* 電感式：可以偵測金屬物體，不需要光線。
* 機械式：結構簡單，但需要實際接觸，會有機械磨耗。
* 磁性感測器：常見於氣壓缸，用磁場判斷活塞位置。

GMT CXS50 精密型官方列出的原點與極限感測器則是光電感測器 GMT-sensor。([GMT][2])

---

# 六、光遮斷時，是滑塊遮住光線嗎？

這是你今天最值得理解的機械問題。

答案是：**通常不是滑塊本身，而是由移動平台帶動一個遮光片（遮光板、Flag），讓它進入光電感測器的偵測區域。**

我們看一個常見的光遮斷式結構。

![Image](https://images.openai.com/static-rsc-4/tJjy7-YfZyfA30ixVn7OhQDrHhLL7j2R5-M5efOTG6oZw5XZm2rY3FHKcL5PfUEi5qUuzyek3skwR5d26yWJvUeT1u5vbQDix0tuukN1DdWqfAmb_GT_WzSt1-YjAotZVpCx2cK9vXmy4bma9j2LwXpzQWRFWVGWobV4PVAoMLXwmF2R79T9JwJTwt2FQtYz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_cPwh1U2eNl4xZWQBPVVRhKguvyK6KByoJSY-nMkV2hdhUq2ooQZR6pStdbd9JAuJeY3ROWBiDTZQOmP5--NkILh9X0tMILi4DXrkwzJXgAHfOaQxckeFDYDcNvcrU-F2nptlqMsPTrveq5KPhdTd963a4LNRX4NjcmC7gmbkasDr8KkaydyLpGkdjdG--ch?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1_KjzHMjR9wnqJPvk1wMJhz3UiQXOirggZ5kEqE0luYX-4_RuEM-30G8wID7H551FdbKPtpxdIb1jQvdn-PW2wJuTR2XQ3g1QY0liLRX38lg4ysNFH86r49yQ5UM1yBqNRujQYRi5R8IJnMsCGdjWFagkaC39kD0GMrB8j41STmMErrJuxy4ZC8S5orY20lR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Mjk_D6sB2_IYZ_UvWahgPJL6Etq9yQwIBtM7whbFRWJBfwfNRnP0feUJUl4NXeZPoRMUm9gHjAMSRA9QSXGc4tOOCNAZbqG0jAczUZ_PKMPssMqYiNyNcgICPvPhLavQxIoWaejfgGZ7Ql2ahPdgzreB_tSjfAF8GFHzb5hJE8EUKXfRu2lxYUkWcVLut3UP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MQ7iIZ4IRDqzJ07J8Rw-wpYleU0mtkT-ufzs3GbEE2OQjznbKo7F-aDjxEXDzluskKLkNJWhImDnH7ZiIspuHcHCBByw84AQTO7r__hyItvgfpLHtWxS7aC59iPwIDqLgrzqGQiAP1I441brqlL_oPVupFSUQbE3znJmrQEZ0alndXyIRnSuLQd7bNN5ifjs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W6jOg6n4zAbqhhmfzTS2WZe6zONDvRla6MlMZpGnNi0aaS7xut9rCBmNPHblzg3Q6bt1Zm_Lf8mTkTHoCsFrgb1NyvqR6vhFaEdhKv23zXf-y2yT-em6t_VPVu7PV7XpvR_ZSv-2kV1jDWpUlSo_kDXgXax6M-38exJeIt15KJXbhJIf5cuOYOMGskdB6e7Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LE3QYg9sxXLdedX2DePRoXOYtWl6DITA_8NXv6uBEuDvwo6wv-3ZX_XQAa1MG6rso_INX7U0svN8f2WwrFooT0obhO4fNRnmtYFidA95opU6e8IEzcREDU8fcgJemUUQkGUKJclug1Zy-nWw-0t66drT2AyVo04VLSEgNmVOSQWtCRHvuXM8uVyE7DuXAWXF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/K7M_bTb0RH5ICi4vBPYXVxGkgqg_XfNDCqnahUMTUhqEYhGiJgH7syeSf6a9DIxkSjXrA1I_spFDQcJzRCHeaP1Zz58YPhBStW7Y9Rg9U9Y7D2dsNWoTz5EiRQxeLEnvOVrM02o6dCCqq6Ly3Rn2_7dvlZdJfkftJZET5J4PnybV5GBsO2_Cd7msZvI1a5zB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sng1gtaFIKPpU0eFpuszy2QVySvDTtRQn8cgrDEuahU1jzyyh2LkIhLJ6o7Oaj5k-QSbrBFuSGH6DrUbxfv99dPxjsxhZpGsCgdUZTvyfUMIlYYr_pItDF6-GIDqeto3DBQqmA7i1O6AKge4SvZMaAU6yUl9GVqrH6R7r-svhJs7bZ4CNas9Xzn6s5BfVI6n?purpose=fullsize)

想像這個結構：

```text
       光電感測器
       ┌───────┐
       │ 發光端 │
       │       │
       │ 接收端 │
       └───┬───┘
           │
           │ 光線
           │
       ┌───┴───┐
       │ 遮光片 │  ← 固定在移動平台或其連接支架
       └───────┘
           │
           ▼
        移動平台
           │
          滑塊
           │
         滑軌
```

這只是常見的示意結構，不代表 CXS50 的內部安裝方式一定完全相同。

## 實際運作流程

假設滑台正在往原點方向移動：

**第一階段：尚未到達原點**

遮光片還沒有進入感測器的偵測區域。

光線可以正常通過。

**第二階段：接近原點**

移動平台帶著遮光片進入光電感測器的偵測區域。

**第三階段：遮光片遮斷光線**

感測器偵測到光線狀態改變，輸出訊號跟著改變。

**第四階段：STM32 收到訊號**

STM32 韌體判斷原點感測器已觸發，執行減速、停止或其他 Homing 流程。

這裡有一個重要觀念：

**光線被遮斷的位置，不一定就是最終設定的機械座標零點。**

例如，控制器可能先偵測到遮光片進入感測區，再反向離開，最後低速重新接近，依指定的觸發邊緣建立原點座標。

因此，原點是由機械安裝位置、感測器觸發位置及 Homing 程式共同決定的。

---

# 七、把今天的知識整理成 STM32 韌體工程師的觀念

你可以把整個系統記成以下流程：

```text
機械移動
   │
   ▼
遮光片進入感測器
   │
   ▼
光電感測器偵測到變化
   │
   ▼
NPN Open Collector 輸出狀態改變
   │
   ▼
24V → 3.3V 訊號介面
   │
   ▼
STM32H755 GPIO
   │
   ▼
HAL_GPIO_ReadPin()
   │
   ▼
Homing / Limit 判斷
   │
   ▼
停止、反向或建立座標
```

你可以先記住四件事：

1. 原點與極限感測器通常輸出數位訊號，不是類比訊號。
2. GMT CXS50 精密型使用 24V NPN Open Collector 光電感測器。
3. 感測器訊號不能直接接 STM32H755 的 3.3V GPIO，需要適當介面電路。
4. 常見光遮斷結構是移動平台帶動遮光片，遮斷感測器光線，而不是一定由滑塊本身遮光。

**下一步我建議先不要急著進入多軸滑台。** 我們可以先把 CXS50 的原點、正極限、負極限三個感測位置放在同一張機構示意圖中，並且追蹤滑台從開機、回原點到正常運動時，三個感測器分別如何動作。這樣你之後看 GMT 的 XY、XYZ 滑台時，就能直接理解每一軸的感測器配置。

[1]: https://www.gmtglobalinc.com/en/x-axis-3/cxs50-precision?utm_source=chatgpt.com "CXS50 - Precision | X-axis | GMT GLOBAL"
[2]: https://www.gmt.tw/tw/cxs50-%E7%B2%BE%E5%AF%86%E5%9E%8B?utm_source=chatgpt.com "GMT GLOBAL INC-CXS50 - 精密型"

---

你的理解**基本上正確，但要區分感測器接到哪一個控制設備，以及系統採用哪一種架構**。

原點感測器和極限感測器不一定都直接連接 STM32H755，也可以透過伺服驅動器或遠端 I/O 模組傳遞訊號。

主要有三種架構：

## 1. 感測器直接連接 STM32H755

```text
原點感測器 ──┐
             ├── 訊號介面電路 ── STM32H755 GPIO
極限感測器 ──┘
```

STM32 韌體直接讀取 GPIO，判斷：

* 是否觸發原點。
* 是否觸發正極限。
* 是否觸發負極限。

這種架構下，感測器訊號不需要經過 EtherCAT。

## 2. 感測器連接 Servo Drive

```text
原點感測器 ──┐
             ├── Servo Drive
極限感測器 ──┘       │
                     │ EtherCAT
                     ▼
                STM32H755
```

這種情況下：

1. 感測器訊號先進入伺服驅動器的數位輸入端（Digital Input）。
2. Servo Drive 讀取感測器狀態。
3. Servo Drive 可以透過 EtherCAT PDO 回報狀態。
4. STM32H755 透過 EtherCAT 讀取相關狀態。

例如，某些 CiA 402 驅動器會透過 Statusword 或其他廠商定義的 PDO 欄位提供數位輸入狀態。

## 3. 感測器連接 EtherCAT Remote I/O

```text
原點感測器 ──┐
             ├── EtherCAT Remote I/O
極限感測器 ──┘          │
                        │ EtherCAT
                        ▼
                   STM32H755
```

這種架構下，Remote I/O 負責讀取感測器訊號，再透過 EtherCAT 傳送給 STM32H755。

---

## 三種架構比較

| 架構            | 感測器接到哪裡    | STM32 如何取得訊號 |
| ------------- | ---------- | ------------ |
| 直接接 MCU       | STM32 GPIO | 直接讀取 GPIO    |
| 接 Servo Drive | 驅動器 DI     | EtherCAT PDO |
| 接 Remote I/O  | 遠端 I/O     | EtherCAT PDO |

## 那麼，哪一種比較好？

這要看設備設計。

例如，原點搜尋與極限保護如果由 Servo Drive 內部處理，驅動器可以直接對感測器訊號做反應，不必等待 EtherCAT Master 下一個通訊週期。

這對需要快速反應的保護功能很有幫助。

另外，**極限保護不應只依賴 STM32 經由 EtherCAT 收到訊號後才處理**，還需要依設備安全設計考慮驅動器本地保護、硬體限位或其他獨立安全措施。

---

### 我想特別提醒你一件事

目前我們知道你的 STM32H755 專案使用 EtherCAT 控制設備，但還不能確定你同事的 GMT 滑台感測器實際接在哪裡。

因此，不能直接認定你的專案一定採用第一種架構。

下一步若要確認實際架構，需要查看：

1. Servo Drive 的型號及 Digital Input 接線。
2. GMT 滑台的感測器接線圖。
3. EtherCAT PDO Mapping 是否包含數位輸入狀態。

**先記住核心觀念：感測器是產生位置狀態訊號的元件；STM32、Servo Drive 或 Remote I/O 都可能負責接收與處理，不是所有感測器都必須直接接 STM32，也不是所有感測器都必須經過 EtherCAT。**

---

# 第五單元：多軸滑台的組合與運作（第一課）

## 從單軸滑台到 XY、XZ、XYZ 多軸平台

上一單元我們學習了 CXS50 單軸電動滑台，了解它由底座、滑軌、螺桿、螺帽、馬達及移動平台組成。

今天要進一步理解：

**如果一台設備需要讓物體往左右、前後、上下移動，單軸滑台應該怎麼組合？**

這也是理解 GMT 精密對位平台、光纖耦合設備的重要基礎。

---

## 一、先理解 X、Y、Z 三個方向

我們先建立一個空間座標概念。

![Image](https://images.openai.com/static-rsc-4/zO0nRgmXmfHUK3c7JB96pxPjt9OtGxiLbvfWnY32OMH8AjildGFsGwBk19vwl9xjBSe1TddqIjOG-iyK6OGsitA4mYGr3NzXxHmqfbdtg43nll1dwXDUeUa2gMgNEvWuoTJZJ9jff5bmUmOLJ5OGcLv2-5-bQ0ygWFwv4uAvbEJEERZz2Y3GoJxlRY-abqcN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gNcCla3YSVMdfNJABZysSpcM1re6iAcZ5bBShyMaaBzxXvy_VLI9F9gQ-4NofjZts3bwwjBK5BMTlXxZ_zmISpTStmtYnYLta6_Oi3pcOn1A5kIfKDTOKs0IWtY3NzSUFLQf8OkhZh2doMKE4tOPdcP9-AllK4et-f4mk2y0gyLBN-1MaN7zQEbOjk5MbVZa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3VDSmdCUnknb5htT1WoAZY66370HZEyQ_VegLM8vBiODl3I9C9XMfB5LJxcHGiLamXaeXPL6xq9viSU88cJw2-7KPFnBy5_ZKibr-1p5SZOah1rdkMqcZEGaEMinEPq5ErnMGU0-XPHOmVw2ScEZ_YrlHDGv2sELMElTNJvnpFv2K-yMimZhwF11qzuRv_nV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uBCmJTVjc2OpgAMDe7uMIaB6TV9cJ3kMaUQhjs0qqw4HamR19V9AEmSVFv1zzlpszzbiEtEvjfvx_ooAczzTy_sjjKSsBz1uyGvTNlVLaiNbazqX-iI5-RZ1r8uYXKiTjAah4AiH4FG25mw7BYT6YX-lWwI60UfvuRK6egOMyjlOfLXrB4QWhajS0jKWGW4C?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2ebGASNL3hnk6KygDmIjeqByiNrA6JTWlMfAooEGPepHALmbNG4gk9cP9UtKIKgpCIZV3dtuwv9BWvfcNtABSgDn4yAylH9Bd4XV7FVGf0p-9xe_uRX9dIFLJ5ol5IND-ltYRi4XWBZJQuzz0OO2_eRe1LojJe2RGPNzr0Liqp2bkJ3AmB8aABtnpDubtpzs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/D--v8N12FQPNtAUKvkq3QgjgWqfMMSRu6XQAX8O1o3fYO7WkF-_uL1DoAdEI7ngBJcpsUoApMWbDgGPo_1ttlahcYw6T2DlqS5_RnLWBa24Ag8lPGYhts6XOrmN-g5K8L9KahBpmcKEWQiFBio3KXcEk7EjdgdxU_X539F4P2G1WOAPUivOfOL4Bn0Y6k8TL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/q4Fu4z2DCxtY7dX_TZ5OYG7x7jHGFcmDfMC6wz8lpGEWEyDeoJlv1ya7xGQA1X23sF3hkHDX39ul2CTyKUHp-rGU3F4w9c3d4bQWV2TfEMVWxR9L3NjlujdQFNOY4WPm8sQAY2eDjXO-UmDeCwN8XodWOS-LIHrgjpwXujhVAYm4vd3s40V-M9rOaPYYULLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7X5IhKzilPAKVfiXjZa5a2Z3YsB88uGe5fTHw3TUVBAMTwDV4_dmYgRBfCBDIjFOfjYe0VhqGtYMn_KnbbHkbiHof2NbgBstuZIcdJ3m_6Wc9SlBSYp0FgvkDStAhjIWWvYa-1DLKXsPOTzyRmLyzNGCSSmDFxq7IXL1FsR1UKnw2LvcbKNxCHkBHMEcU1Bi?purpose=fullsize)

在精密定位設備中，通常使用：

| 軸向  | 運動方向 | 想像情境     |
| --- | ---- | -------- |
| X 軸 | 左右移動 | 把物體往左或右移 |
| Y 軸 | 前後移動 | 把物體往前或後移 |
| Z 軸 | 上下移動 | 把物體往上或下移 |

例如，你有一個光纖接頭，需要讓它對準另一個光學元件。

可能需要：

* X 軸：調整左右位置。
* Y 軸：調整前後位置。
* Z 軸：調整上下位置。

這樣就需要三個方向的移動能力。

注意：X、Y、Z 是設備設計者定義的座標方向，實際設備的擺放方向可能不同。

---

# 二、XY 雙軸滑台是怎麼組合的？

我們先看最常見的 XY 組合。

![Image](https://images.openai.com/static-rsc-4/BR9aurL9YbdQpKkCG58xy7Odfn9S_Xuw5MCqfpubVK0zuYd0ORfFd0_Tt6xDerjHXUzMWDB31ZtY8S5Ait3t9D4_xLYQhlrQktZPr3h173EvBX8G-KfmqIp6yUkfqjQAtw3OReCwV_WNwQga8okB5TUCnguv54sF_WIGdg0mC_T3XaH3KgJ-rzThhZn4KVVc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RrV1NsyDtEWpa-c9PJ4O0fnXlQTr8JvL8Yeh_W0sKo7P52lYg78Jv2IULbXjV8Y3xq40e8utj8D5j4EG5Wt8MnLl5iHpGoUk8TajtUFLl3DCVKzLRDImT66eMOS04E0Auju40cbZkWRINiGh1q1SflFF_2nRZehvBuS9_CpbONq9zcO9Gj509nvFOJD0qSKy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7gtijK1efX2w4dHcBeruZMhg-bOmv5S66Aay7P3jrxdP-HRPUePLXBdn8avKZf5nVH5sfX2RQDRFKBftGpDy93r3Rb3d1ffhYh0WVEjPWqVe4APnMXjgSPxfaVYF_pvgN_LDooCxBX3WIxw3XJotmqpwZ-J6Ct9SRpJBygJ8-VOqd1guyb0qjdzFRBXL3enR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/e5tWrm5x00YfbTVfcsDbsmIVyCQ-gfY0jHv_kxVGa8GP5onbzOtwfJFp1HM4-a24hoBEMykNXHECS0k-jmkRqjbgHiJHGh8n0rwm4fLoctzw_vIt4c6_v7ewXzng84kc7QJ9SNW3ri64UELs0h3GCighVixFmyx0fhldPUq_rDdCNFNkdHyQc69fo6dRSN4G?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NGCCWbcPCedtBwcaVORw-_ymX7Bg32yv3FebNkv1REbD1TwBAO2mMACt660zQK_35sGpIu6Y1stuiyeB4-8sEVFewxP3LR497kPZHRUDYgIVRbJMLBwnmLieV05c-V5RiGV3zSEnFC24cYFw6tMfs1jOp34UuJ-kCoRox3Eahw_r0jqzvlukSeU-qVqZqHE_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WOL4TOlj2t6a5EbSLhkrWsRMW4k_avo2nMKgkYPzg9g3Sbku9Ol7MPuGFzygPXvZXz9kJgvl3znnNddOt6wcm57JiUYzjq-smkHA9fiUUJNcc5D6DkqXOTmcSrE4tAXzTqKrw1Lylg3hsupFE3hrO9BDuR4DEBe7XkAVzi_x6P_y_waJfPgfDSyzd9TFEoda?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ha2_Bspiu_SC83ybDm2sR-20UsIKU9iDw5X9pH3Pn2jvQ400RN2t0UnSV-qoJJIcElVklXvD3-F0Tj4KCq3XWghvQXG5ot6vwVO97p3ViTW-EsVGh7ln_8WnWCrkGcm1e5lLROPHz6GhW475GSgj3k0Xy68WjEL9acDgbC_6eDh__ES1IWF--3a7PDGEWuJk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gQXRw6NBGvSPFPGAc3VnkVGH8xTUDDYCEBZgS4yyGcJIDp3jxAnzE-XVa4ow1vvxNbYT8m71Reo75s1-4zqIpJOHPjna-O0G5NG3ekLQ-ObzELEEJUskuD6RoX94DDQl0q130uBHk83nY5D4IQdCbUkSdLIq17ov7-_Zfwxaoov9P9RBLALcJ9RVBU2r2V2k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TsQFThgKMsXF6ULO31d1AS-uUCWDu_RNmXLMdDQu5nqpxMPQqRDZagzBlTQiVXUF7RuYreEwn5ZUnOWG-rswia1WFvplRrfK3QOmBq8mXk5LBDkXFDxh25Pc44jkXhUv_6HQDN_ZrgSVGW4sSHOwbsA9StjfBJaoRRbJq3YkkGK6dashYiSY17ZV_iE--wtg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W1ALp8IrSi8XxTWvGqq39SB60tw4YKDq-AiTGgvd6GkCN7nOPDwbGcBlTeF0wv75HoTZvMBh19uPRdGsqAThJJAzoYGLuEXX9Uj6QOExPw1MjkPFc2B9jF1IboC8nvN0q5YiH4wuzXD4MNa5J5-6UUhfQ7F7TXe3w2oE_GBRCTqnRUi-64-WV8wFrMSW3kUW?purpose=fullsize)

它的基本概念是：

**將一個 X 軸滑台安裝在另一個 Y 軸滑台的移動平台上。**

例如：

```text
             被定位物體
                 │
          ┌────────────┐
          │  X軸移動平台 │
          └────────────┘
                 │
          ┌────────────┐
          │   X軸滑台   │
          └────────────┘
                 │
          ┌────────────┐
          │  Y軸移動平台 │
          └────────────┘
                 │
          ┌────────────┐
          │   Y軸滑台   │
          └────────────┘
                 │
               底座
```

這是一種常見的上下堆疊方式，實際設計也可能採用其他配置。

### 1. 當 Y 軸移動時

整個 X 軸滑台會跟著移動。

因為 X 軸滑台安裝在 Y 軸的移動平台上。

### 2. 當 X 軸移動時

只有 X 軸自己的移動平台移動。

因此，兩個軸可以分別控制。

### 3. 當 X、Y 同時移動時

物體就可以在平面上進行二維定位。

例如：

```text
       Y
       ↑
       │        ● 目標位置
       │       /
       │      /
       │     /
       │    /
       └────────────→ X
```

假設：

* X 軸移動 10 mm。
* Y 軸移動 5 mm。

那麼物體就可以從原本的位置移動到新的平面座標。

這就是 XY 雙軸平台的基本原理。

### 4. XY 軸的機械組成

這裡有一個很重要的觀念：

XY 平台並不是一個特殊的單一馬達機構。

它通常是兩組單軸機構組合而成。

每一軸仍然可能具有：

* Base（底座）
* Linear Guide（線性滑軌）
* Ball Screw（滾珠螺桿）
* Motor（馬達）
* Coupling（聯軸器）
* Stage/Table（移動平台）

也就是說，前面學習的單軸機械原理仍然適用。

---

# 三、為什麼 XY 軸的上下順序很重要？

這是機械組合時非常重要的觀念。

假設有兩種組合：

**組合 A：X 軸在下，Y 軸在上**

```text
        被定位物體
             │
          Y 軸滑台
             │
          X 軸滑台
             │
            底座
```

**組合 B：Y 軸在下，X 軸在上**

```text
        被定位物體
             │
          X 軸滑台
             │
          Y 軸滑台
             │
            底座
```

兩者都能實現 XY 平面運動，但機械特性不一定相同。

主要差異包括：

1. 整體高度。
2. 負載分布。
3. 移動質量。
4. 結構剛性。
5. 線材配置。
6. 定位精度與動態性能。

### 特別注意：移動質量

假設 X 軸安裝在 Y 軸上。

當 Y 軸移動時，它不只要移動被定位物體，還必須帶動整組 X 軸機構。

因此，Y 軸承受的移動負載會增加。

這會影響：

* 馬達需要提供的推力。
* 加速與減速能力。
* 滑軌承受的負載及力矩。
* 機構振動。

這也是為什麼不能只把兩個單軸滑台疊起來，就認定一定能得到相同的運動性能。

---

# 四、XZ 雙軸滑台又有什麼不同？

XY 是平面運動。

XZ 則是讓物體可以左右移動及上下移動。

![Image](https://images.openai.com/static-rsc-4/9cbSPnfg3GJUz_oC9lWIjv1uOSaCCp9Dqha4Wdo43Veas4iOdXB2CSyi5dOegeVU9VYBN_IRqbKAkd9lSgArKD6vKL3amic2VTtIeKosBSx2ktseb_MkA_i_7zn_XFCImcenrjpTfSI6Wo7lbtrvXa2vD61RYqGk-jph6NmQW6AUVGu-cXc5e_tcjoutIiOB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/maDdXJu-jKjVkjND9WbaOguTgj5-NuXZBQSARCflw9GGIipIXQu4-dAPGQ02w3cgEbebw_UJ329iF_-Zc0eOHI1weN5f-uiTJeGUmo9MTwEokF9hrxmZom07C2AlCBk8Y_d9gcpoeC_6bSZtg7FOFYPI8tYxixA1Ye1GXM5x7yrikJKUfT14K2KIPchDt6Be?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3VDSmdCUnknb5htT1WoAZY66370HZEyQ_VegLM8vBiODl3I9C9XMfB5LJxcHGiLamXaeXPL6xq9viSU88cJw2-7KPFnBy5_ZKibr-1p5SZOah1rdkMqcZEGaEMinEPq5ErnMGU0-XPHOmVw2ScEZ_YrlHDGv2sELMElTNJvnpFv2K-yMimZhwF11qzuRv_nV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WOL4TOlj2t6a5EbSLhkrWsRMW4k_avo2nMKgkYPzg9g3Sbku9Ol7MPuGFzygPXvZXz9kJgvl3znnNddOt6wcm57JiUYzjq-smkHA9fiUUJNcc5D6DkqXOTmcSrE4tAXzTqKrw1Lylg3hsupFE3hrO9BDuR4DEBe7XkAVzi_x6P_y_waJfPgfDSyzd9TFEoda?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/c1_zWvAjCJ96I_ma6UJXckpJ8_eM5FvX1Y1reYnsznOXYTSL_HAkgNhCmOnQvTq1ezsU10HZrDSxQziFGJDNXLqqgpCtx5J7nrvuXzIDX2cWJT5NwpoqYG71jIhQYHMYsKap6j6WymH79HMkL9TI4J9xM22OYHWECzveqry0FiH2I4UpRigZT8dkijY9rV2N?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Jghc8ui6yB31KU0ArFIoKTnrw111af69bZZO4jxBgBNOKLD2WNxfEetMClc-7-ieHJgeO2-jSmJb9a0NofoTg6qQs-mMOz2wDyOS0j3rm-maNPUew0ulinNLBEcgtFBghHTM-X_qiJ8uxyGj4skN5gqioq33YNKts2lZOoYrzRXbMYVZP0IU9lxDDBqiz6dt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qdG1JeqfxEmPC1_5dnUYcC3lGOq2JXAImHHHur4-Z8oT9SzlnztM5wq0DLNM6c4zlp858TC7giEcSq8sXZ9hfCYUuuApOsLhLFOCt-ztPrXR2WIVco8SCCkrO6_07HKYkJ66-HmnYen8diEXffq116J1mYfbME6aYAKX1llDxzFH_euvoXkD4c0U7SQsoEEW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GZohbyak_ImeERX6A7koFROovaRQmk_Bs5DWQl8YiFaacGxpny799W8bda_NrPovjSpVWmIlKM3CJ29rZYXHoVq1B3FNmYDqj-m5uLusJEA-H_IbBEQSmOBtujOjB2JvQdv6u_DTTpby5OnkrZVnk28CQlmYYu8Or97EJyZZ2eDPgeusJD7RxUG12g4sFhjl?purpose=fullsize)

例如：

```text
       Z
       ↑
       │
       │     ┌─────────┐
       │     │ 定位物體 │
       │     └─────────┘
       │          │
       │        Z軸滑台
       │          │
       └──────────┼────────→ X
                  │
                X軸滑台
                  │
                 底座
```

XZ 軸常見於：

* 相機高度調整。
* 雷射頭高度調整。
* 光學元件上下定位。
* 工件取放。
* 不同高度的設備對位。

## Z 軸有一個特別需要注意的問題：重力

X 軸或 Y 軸通常是水平移動。

但是 Z 軸可能需要垂直上下移動。

當 Z 軸承載物體時：

* 馬達往上移動，需要克服重力。
* 馬達往下移動，需要控制重力造成的下墜。
* 如果突然斷電，某些機構可能因重力而滑落。

因此，Z 軸可能需要：

* 馬達煞車（Brake）。
* 配重機構。
* 氣壓平衡機構。
* 特殊自鎖或防墜設計。

這些不是每一台 Z 軸都一定具備，而是依負載、結構及安全需求決定。

**對 STM32 韌體工程師而言，Z 軸尤其需要注意斷電、驅動器故障及煞車控制的時序。**

例如，不能不考慮機構狀態就直接解除煞車，否則負載可能下墜。

---

# 五、XYZ 三軸平台如何組合？

當設備需要左右、前後、上下三個方向移動，就會使用 XYZ 三軸組合。

![Image](https://images.openai.com/static-rsc-4/jqzYqP6f8LeS8wSg3NnF7sqH94xyis7p3lb4DQJ8W1rYYEW0UvEw7s6VqEghAH_jTtJwOGcafuxDhl_sjUHPlkjYMaiAuuh493pVHaKM2zY7MyFfcWqxTE6O3enEvhFYHoQH1EIW8SMnhg5McEWpE4KODEmP_hg0BOzakThnLrGS40K53ErS_ZD97wbSNDk8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gNcCla3YSVMdfNJABZysSpcM1re6iAcZ5bBShyMaaBzxXvy_VLI9F9gQ-4NofjZts3bwwjBK5BMTlXxZ_zmISpTStmtYnYLta6_Oi3pcOn1A5kIfKDTOKs0IWtY3NzSUFLQf8OkhZh2doMKE4tOPdcP9-AllK4et-f4mk2y0gyLBN-1MaN7zQEbOjk5MbVZa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/43Ewls4s_oQGz-pk2JxMwJAiL8yj_ih7YIm7jq7ThM45S3PCTG5BlTdCdpfC5ySTfACDZVDlx6W37RT2Qrk5YVRWmm28MLmJbTULan2krM4WgSH0i1ZlXlA_RRAoEVZH3aXlypwqVtTM62aL6sGqW7nrOA0IPGaM21FmIS4XBU9QW-IgI2__zE6xU1w5gzg5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3VDSmdCUnknb5htT1WoAZY66370HZEyQ_VegLM8vBiODl3I9C9XMfB5LJxcHGiLamXaeXPL6xq9viSU88cJw2-7KPFnBy5_ZKibr-1p5SZOah1rdkMqcZEGaEMinEPq5ErnMGU0-XPHOmVw2ScEZ_YrlHDGv2sELMElTNJvnpFv2K-yMimZhwF11qzuRv_nV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cD35AWc_GX2tflokI3jpwJAs21azwTk3pBuMOny5Zz19UdvISVdsgzt9XKpjuFjeg2eTOwinL70-raSbpNvGn8Jlal6u3KIYZY4rWsC662EbH1KqEQT5TvlHRbdy9YNO8JJDW7OKCXaDTUQNhvfesGlMmX8xTPHWKmzba0ogwiNERcZuWtMZzCGVlgi7Cc8t?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hIrCkc25LYga_Ov5pkKJMn6JpWMHXoguZSqkNGVx6E6vO-qtbc5VHUcbj_kQ8SV1gm_nEB1RQ5K3b4lb3Fa2unCyA0XdjwaYveom-8r2a6qGT71FRPbIo2_QOakJnogTTYDCZqFncdfH8tAy8xjnNVbr23pNBNNNw59jclimMU3ZCCPAKS-2arfQhNoCK3Ky?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uBCmJTVjc2OpgAMDe7uMIaB6TV9cJ3kMaUQhjs0qqw4HamR19V9AEmSVFv1zzlpszzbiEtEvjfvx_ooAczzTy_sjjKSsBz1uyGvTNlVLaiNbazqX-iI5-RZ1r8uYXKiTjAah4AiH4FG25mw7BYT6YX-lWwI60UfvuRK6egOMyjlOfLXrB4QWhajS0jKWGW4C?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rbb0lQwYyGpGlTvgHlMDr5nXZoqchNPXftcDsW5ZkmmTy4FX0Hx7eeE8697o0BJBzQHQh7Ov6OD6QQV7BsOGlC012k-6g_ZAI9GlUktods-2Stl4RfxwmPC2wFZq3btwmoQp4ayweCmgnAVouYTx2wJMfHFU203fo98tnT7Ti5yBzoOKxTAxhicANH18ylFy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ha2_Bspiu_SC83ybDm2sR-20UsIKU9iDw5X9pH3Pn2jvQ400RN2t0UnSV-qoJJIcElVklXvD3-F0Tj4KCq3XWghvQXG5ot6vwVO97p3ViTW-EsVGh7ln_8WnWCrkGcm1e5lLROPHz6GhW475GSgj3k0Xy68WjEL9acDgbC_6eDh__ES1IWF--3a7PDGEWuJk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/M8OJ_y3FMbd-MZKuEVg7-C40sY84_cgn0dWzLqqIESOx_-l5OvW6SwVbRMaxLttWyocNxQB6rQQ18KCSJvrl5VJGyL2jJPWnbiILvhb6RzpBaBvb1QrVXWt6JIJ_npd4LyOSE_J_wkPc-c_46v3TqGNto5nMsnvI_LI9IuYP0mtBsX-9CFEd0kS0Yp0gt6vl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OyyP-hE8wHyAiycwE8vikB76cmbYu_VwjxVqHmgcr1Gm7M159SXuqWOgKIDg21F2Si-zVVpn7b6Dewc9Z_g_rD_o3DU686sogzui3dIGsRion3YypSaRzt0vG8TwZYlth71TvrBK7hQI0ZlZDif4Lxwl0YLEoinUWVlkATmWntCh95wHffgDJ37vNdwm0zDy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2LwxvAbO_KIH2OHgio7yQnwys_8ev4yYbuqPIEXI_8UCS-skrbPiM9akDZGTVq5Jk8Gocn-UcGxc05X1kZDlnHqkZwjyb0-0x309nm99ndaPgUaIeeVoV338P94upXk8mqgDr-aOxGtkqgS4v4T2AyoaMSX5gDQi7oolAo-s3oU2fQh4pe3ifDi1DD-rPlhg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/c01FCY_orn8wDnep5eKJn7uhCPIMyAZ_qhg9AKwsPOoAziwbTwFcbnSW21-dAP1dNhMKVS3AhAOtNeas_ehCPZq9-o4wtVGufPVp3fV2418uiszCvcJstyC-BcBrv8eQtiKTdhiQM7k7oCoB5JpNZpVTuy_TApzXYk8XEeh7iyEPr9rlPD43zkC1lV8wIuyq?purpose=fullsize)

最常見的概念是三個單軸滑台組合：

```text
           被定位物體
                │
           ┌─────────┐
           │ X軸平台  │
           └─────────┘
                │
             X軸滑台
                │
           ┌─────────┐
           │ Y軸平台  │
           └─────────┘
                │
             Y軸滑台
                │
           ┌─────────┐
           │ Z軸平台  │
           └─────────┘
                │
             Z軸滑台
                │
               底座
```

這只是示意圖，實際 XYZ 組合的排列順序及方向要依設備需求決定。

XYZ 平台可以讓物體在三維空間內定位。

例如：

一個光纖接頭需要對準光學晶片上的光耦合位置。

可能需要：

1. X 軸調整左右位置。
2. Y 軸調整前後位置。
3. Z 軸調整高度。

這就是三軸定位的基本用途。

---

# 六、除了 XYZ，還有旋轉軸 θ

到這裡，你已經理解三個直線移動方向。

但是有些精密對位設備，光靠 XYZ 還不夠。

因為物體除了位置需要正確，角度也可能需要正確。

例如：

一個光纖接頭的位置已經對準，但它的角度稍微偏斜，仍可能造成耦合效率下降。

這時就需要旋轉軸。

![Image](https://images.openai.com/static-rsc-4/1Auqog4wYLud09dnU6DEh9qf6j9uqCnIoGoK3K0RI94Z1Xn5cqC3vaYkRQAUOrqtn5bq1Igu6KhL82hxj5dk9Ri-NeFCXNWLXrpRjjhSQJY4Kh2-Zsdd2K27-7dF8hNeaoPhtPMrYZvbZJ4bMRPUAR5JqyX6LLDwA8Vg0NOi0LZSkq7AmEuRFY-Em3aJ-ZEn?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uO9csXeu8_tLa_RsJ_FITeN5wFNUk1-FD69XmauYHxBpgnLk8PxjcEYB56nDpAA14_yjsaD_fzmr8_dexE8PQ8u4GZ7AxuyA5_ZStoWZBiQirZjDg50xG4Z3qAeJKP0h-gLUWMfCrSM5XogaFkbnCyJrBLO1-i4UGHoNeWoc2a5A_gJOkNCctRrE1zRvBb2F?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Ve1DwLW8CKJjZK7QDdtQDvSaq2LkRbRjf6g7OAFovuEnen9rHiXalEahUgM3PpUBfRSdbATY7EujUAPHrr-ySLP3fOlTM__iZDa4ajtSgKmzYogG06IAuq1QnIf7q_btoyKKgvBL6LJ1mQuo-1uWZFrEGTcru34wXauv6RUrJ-CX_kbPCN1IBj1RIj0bfLf0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RTYwEkxXMqY7Q_BveB_4eDgk--rC4SBPYes3-ViZzNF7_ySygEqz7zSGQgyPxh50RFFgjHUiLLHh4uwon_vlPYE8AErM2mzq4IP4Hyc4a623pnhUw47rdJ4b6UXifVT8NKC3LXL_5cWAwHhSH6ys9kAONIQYJgUdPkbf6VOz2JxwfxlzGpQEMfVaz77IkcjB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZbqpU1mYIImGkWuqjkSvFoY3SLXUsVjRoIErOgn5HNaNExbVeT_7o5lUbfcnZb50zAisU18Qstvl_A6Iy9Zt-Qfy9TZnMO46DytX348UKfp2MCzsfDI1dYTFXgHjSRxy82PUDk5fd_2VriFliNnHCv0EMYfuKG-q4Enu6c7hnyVay-r4MTuHGP_kKL6z46fe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ARo1xfJhQoNNZyb_Vs4DYFgr0Va2XfMWSGKQTIXpDCel42tzfnLqdFUx64L8eUm0ewHkBvBf8HzDt2WEx8w3MKyvmbvxstSujUSz5iXkEeDDTQXG_bW9SvROYpK_LooGuWwXWxc3Q_CVmA1m3TpR4kOmuwfNBoV1Rs9WwjgTMU1yebVif__ah9vKyH1Uqo-y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/blc8MPcfvkDRwh6Ir7hAeftv7Ij6B5xLNo2f8jFQYZdzip9MNZBZgX5H3pwTWdOtU3h7VeJ2vkALTZPwC4yQE93SFdI99uzeJcxrY_X2QUdO9eAns6cDNEco13snOxIIfdTsjO_WrhQXTgKghWFblmSmry3uuXlPlauzAWl1fCqkIxHbVaWwrjZ0DhV8690_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IiCENJ0ppKa5QokKsaGvkDYNhnMIwUMxJGcxwe3Y48446wBVWi4ZvfCFDGS7DPWJHsSdElsXDlMkmoSc9XfBuJVG2Bv1VPaDjXZj9WAAU7dVAnPDArhgeoAjS-ChPSNdi3LPAtqXTjWDvpzDLVz87cwJSHOKPXKRUDKSJ2rUBkE5QEABaGnQwW6XIUMKaLk6?purpose=fullsize)

常見符號：

| 軸向 | 動作     |
| -- | ------ |
| X  | 左右直線移動 |
| Y  | 前後直線移動 |
| Z  | 上下直線移動 |
| θ  | 平面旋轉   |

例如 GMT 的 AXYR60 精密多軸平台，就包含 X、Y 直線移動及 θ 旋轉軸。官方資料列出 X、Y 行程各 ±7.5 mm，θ 軸行程 ±8.5°。([高明鐵 GMT][1])

這裡可以看到一個重要觀念：

**多軸平台不一定全部都是直線滑台，也可能結合旋轉機構。**

---

# 七、GMT 產品中有哪些這類組合？

我們可以從 GMT 官方產品分類看到，確實有不同的組合形式：

| 類型     | 機構組合       |
| ------ | ---------- |
| X 軸    | 單軸直線移動     |
| XY 軸   | 兩個方向直線移動   |
| XZ 軸   | 水平與垂直移動    |
| XYZ 軸  | 三個方向直線移動   |
| XYθ 軸  | 平面移動加旋轉    |
| 多軸對位平台 | 多個直線與旋轉軸整合 |

GMT 官方產品分類也包含手動 XY、XZ、XYZ、多軸滑台及旋轉平台。([高明鐵 GMT][2])

另外，GMT 的多軸精密電動平台說明，這類設備可以由兩個以上的單軸電動滑台堆疊或連接，透過控制實現多軸聯動。([高明鐵 GMT][3])

你可以參考：

* [GMT 手動滑台產品分類](https://www.gmtglobalinc.com/en/manual-stage?utm_source=chatgpt.com)
* [GMT 多軸精密電動平台](https://www.gmtglobalinc.com/en/multi-axis-precision-coupled-electric-stage?utm_source=chatgpt.com)
* [GMT XYθ 精密平台 AXYR60](https://www.gmtglobalinc.com/en/multi-axis-precision-motorized-stage/axyr60-precision?utm_source=chatgpt.com)

---

# 八、從 STM32 韌體角度看多軸平台

現在把機械結構連結到你熟悉的韌體。

假設設備有 X、Y、Z 三個伺服軸。

STM32 可能透過 EtherCAT Master 控制三個 Servo Drive。

```text
                 STM32H755
                     │
                 EtherCAT
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
       X軸Drive   Y軸Drive   Z軸Drive
          │          │          │
          ▼          ▼          ▼
       X軸馬達    Y軸馬達    Z軸馬達
          │          │          │
          ▼          ▼          ▼
       X軸滑台    Y軸滑台    Z軸滑台
```

假設 STM32 要求物體移動到：

```text
X = 10 mm
Y = 20 mm
Z =  5 mm
```

STM32 就需要將目標位置傳送給對應的軸。

每個軸有自己的：

* 目標位置（Target Position）。
* 實際位置（Actual Position）。
* 目標速度。
* 狀態及錯誤資訊。
* 原點及極限狀態。

如果使用 EtherCAT CiA 402，可能會使用前面學過的 PDO，例如 Target Position `0x607A` 和 Position Actual Value `0x6064`。

不過，三軸能各自移動，不代表它們已經具備完整的多軸協調功能。

例如，若要求 XYZ 三軸同時走一條直線，就還需要軌跡規劃及多軸同步控制。

這是之後我們要深入學習的內容。

---

# 九、本課小結

今天先建立五個重要觀念：

1. XY：兩個單軸滑台組合，實現平面定位。
2. XZ：水平與垂直方向定位，Z 軸要特別考慮重力。
3. XYZ：三個方向定位，實現三維空間移動。
4. θ：旋轉軸，可以調整物體角度。
5. 多軸控制：每個軸可以獨立運動，也可以透過運動控制系統協調運動。

## 課後練習

請依照目前學到的知識回答三題，我會逐題批改。

**第 1 題：**

XY 雙軸滑台中，假設 X 軸安裝在 Y 軸的移動平台上。

當 Y 軸移動時，會發生什麼事？

A. 只有 Y 軸平台移動，X 軸保持不動。

B. X 軸整組機構會跟著 Y 軸一起移動。

C. X 軸馬達會自動停止供電。

**第 2 題：**

為什麼 Z 軸滑台通常比水平 X 軸更需要考慮煞車或防墜設計？

A. 因為 Z 軸使用的馬達一定比較大。

B. 因為 Z 軸需要克服重力，斷電時負載可能下墜。

C. 因為 Z 軸不需要使用滑軌。

**第 3 題：**

假設一個光纖接頭需要調整左右、前後、上下位置，還需要調整平面角度，至少需要哪幾個運動方向？

A. X、Y、Z。

B. X、Y、θ。

C. X、Y、Z、θ。

答完這三題，我們下一步會進一步學習**多軸平台的負載、剛性、移動質量及安裝順序**，理解為什麼看起來相同的 XY 或 XYZ 組合，實際定位性能可能有很大差異。

[1]: https://www.gmtglobalinc.com/en/multi-axis-precision-motorized-stage/axyr60-precision?utm_source=chatgpt.com "AXYR60 - Precision | Multi Axis Stage | GMT GLOBAL"
[2]: https://www.gmtglobalinc.com/en/manual-stage?utm_source=chatgpt.com "Precision Manual Linear Stages Manufacturer | GMT GLOBAL"
[3]: https://www.gmtglobalinc.com/en/multi-axis-precision-coupled-electric-stage?utm_source=chatgpt.com "Multi-axis Precision Coupled Stage | XYZ Stage Motorized | GMT GLOBAL"


---

可以，而且我認為你需要先看懂機械透視圖，再繼續學習多軸平台。

你提出的疑問是對的：**X 軸不一定要放在下層，Y 軸也不一定要放在上層。兩者可以交換位置。**

我先用實際機械圖來說明。

# 一、先看真正的 XY 滑台機械圖

下面這張是 XY 電動滑台的工程圖，包含俯視圖、側視圖及 3D 等角透視圖。

![Image](https://images.openai.com/static-rsc-4/V3LYnADDU2KVuZ_3GVcK6s-Ts_v1AJFwZTgAj51E9uW2YLJGf89mU5qJMaIU4UHksYeljXB0l4WUQTWmFzAHLl438kasV9moABUdLDoaoEqi1HA4xPi4lQf5LLIJPfBrLqMebyRe25W8v2qrvys4Hu9a_gWRz-J3yjr7QB43MV1r9qvM2tbXLPH1txzLScdB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QOa9B7sZN-Zq-9gJeZEPzf2hyQhQUPeUDJ-Xv0sMVxI79xtF5skLIBwEYGI6Ov3WKWaqfKWAsbm812Sl9g7-CHCzV3LnJE-d-EcYfv1DBUNZgH3xrT0Z1dHc-pe_ZouznHyQUB_7zAVumvdOA4donT-ET3dk90V3avih9lBSedsi9o_ouUPfpHL-hZBbrNFY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2DbHDNfBwF3GMA0CgeofYy6y9kTsq926C42QRM1Qs7_L_EEyL72BPTX5_pFJNadjEG5xnTGSpHVJzYWUKyFdV33Hh1wYLXtOcLJ6yHKYsMRJMs9ibJ83gLDE8iMc5XprDh1oKXuaFoZChFUNFrdyCDe1Yjd-SSUuaWSTOkVfcZlKyuXLEzpsrNS0Z-gH4yl5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J5HKcGSKLQH1Oed39U3ee_56ZLSb70Rr_Zv3ryqcH2ae35eQLpCR5Y-kcQYoYVfsZ2btqz8-EIykXQEav8Oui_lrSmd1Ltg6nIeS87jyEOKB6xuOoEiyS8VvRuMds8abV7AozHkE6XONn6SwNIZrOr8uTlNpx8wAWneHZH6m2qR09aU-U-WRMIc8GPHFkJ4m?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iVam1qi4ozEExR_avzRsMtbotuWAab0S0a15hroKe6KiWU5e-rI1EgwVWxYGMfbtWF9pczIGBDnvFpAR8NR3yxnJo6I39dzRzuyTAwOPatdw58g9617DOW5QZ7Yd_eF8qQoy13cUqhgTO1lhgI5m8YZ1bjLMn0lubwOUmXBS4YtmXjlN73YJuV3OVsbtceGq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KEfYN-dgODaaJLOQ-_KgLaAgq7ThIkIi449LMQGqoih6hJjEwc8mM6HEZmlw4tJhmFYzdSvrY3FC3zBrp7K0DiRU8pnFYaGVKb_RN3N_OCSpz4zua1YLYJxcWeiJGMOMVEf1wYEe0uS8jmR7hA4XNnDDwR3o_ITr2-o6L666rHvOdYKmltEQHrbZJd5lxY4p?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QxH-PI7Sx_mRLBD4mANo_04UcGE21Y2WC3EdQ2XufciQ3Rht4XopdTmNqIiHSypWmLf26MR5kGRfWGwBLxRIvyvz5JSpR5j-fiuyj44a8Euhg-PAwUlESUvJOTbiYDHTb8oIOPUXlJah4N8mQsq8rGQIeuML4p7CMi_oirrQBplzF8H0vS6k8G1ITrNnhceN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lFTOccpm066sUpaF-UiRoUa3T-VMKvNBhuYQqp9cIwU-gPgDK1TqMgDa0ru4fGlkzfZ3rs2GA4tll6FdvYtB4Y1kdBo-R8pTImvdCBjk8fLmai_vHuciYzZLDs-8Vw7abwTrMu18_U0keHKempHHELxKLvFi1g_qdImlfC_FhqCmf0folaVnmeGN2kCorKKg?purpose=fullsize)

先不要急著看尺寸，請注意這幾個地方：

1. 最下面有一個固定底座。
2. 底座上有一組直線滑台。
3. 第一組滑台的移動平台上，又固定了另一組滑台。
4. 最上面有一個工作平台，可以放置相機、光學元件或工件。

這就是我們所說的**上下堆疊式（Stacked XY Stage）**。

---

# 二、你問的「疊加」是不是這個意思？

答案是：**是的，就是把兩組單軸滑台上下安裝在一起。**

我們把它拆開來看。

## 第一層：只有 X 軸

![Image](https://images.openai.com/static-rsc-4/S_mGaCq3PzEcdO7gYcXFH69cYl_rA9wZep0-suyKbL5mR7T8M_F-uGiv05frn_Aad9MxvjAJVxOPIkn8UE_ewnuFwtT_n4337_fSz0ldZwUWS5iyVH-l65P_mmPSPBkN8JNMWcu0p7jtvUJcpawa9sro27ZOH4YL2a2Tz1X831UnsncMKgcqRxNuyAP-Atbq?purpose=fullsize)

先有一個 X 軸滑台：

* 底座固定在設備機架上。
* X 軸移動平台沿 X 方向移動。

這時候只能左右移動（假設 X 定義為左右）。

## 第二層：把 Y 軸裝上去

現在把另一組 Y 軸滑台的底座，固定在 X 軸的移動平台上。

![Image](https://images.openai.com/static-rsc-4/iTeZE_EYfOhXHfsUxMbudDuwDILb9oBnUYw5h6JWupLUIjnCTWRgWuPt4JuDaYmkkAs3xGorWmR4Ls6c8g2tfUvVJaV_beCaXP9-9P_VDMAR8okXb5aJTFeOa5nWOojn0evxWH5RJnAjs0qVRm1YDJq-48Mc_2Gfdc-pVJp7Kth6qOitAcl140yY6TkZ5ywi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/v2wYvXNiWboO2Vn22BmcM8Es1FQQzuAegB-I8Mt6joxgRZKu0j2zH2GfIaDNhnAfjqN8Kkpt7esJrcCSEicobIiNAMvolO24npAiXlA6KO07W3yUv5pqbJxQY8dRZ4D5cnjlBsdzg_mJVlUHKR0nG5JQVS0dv_XhssK_J5q1NYU-dZWpBF_VR6uES9UlPfCG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XzgVHfa4VluPf7Z9Kf-UFyt2ywPBT6luHfZqlvW0a8fQOueyYIH1OfGOVi1IA1zr5tXuT09k_DTBY8jvt4SdIfmjw2oI98d8fCHooEwMZJ9Fx3lGwspSZVWuNiMG8pJnGQ9fBpS2JQFZhubRbl88nghU8p95UNOTg9ix0oOfHlUkg-Tlmb2xMGBHZYxWXylC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BR9aurL9YbdQpKkCG58xy7Odfn9S_Xuw5MCqfpubVK0zuYd0ORfFd0_Tt6xDerjHXUzMWDB31ZtY8S5Ait3t9D4_xLYQhlrQktZPr3h173EvBX8G-KfmqIp6yUkfqjQAtw3OReCwV_WNwQga8okB5TUCnguv54sF_WIGdg0mC_T3XaH3KgJ-rzThhZn4KVVc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/frHR75FXnj5Ag5vQWnHCBERbB0JNm93ZKb1l9wcWUTZnnCyCAriZsOscbWpDxsZD7r8YKQ-iUHFLrFdSuHZp6g4--Lk0TP1kLX4Nb55kd1872tbgG3-1OyTISeh963l_eqOclHIZZErYreR9ceOhEtZEU6ZNeEn-UgQyz7lAHio4k_IG3gzTMh6EL2Qza6bZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2DbHDNfBwF3GMA0CgeofYy6y9kTsq926C42QRM1Qs7_L_EEyL72BPTX5_pFJNadjEG5xnTGSpHVJzYWUKyFdV33Hh1wYLXtOcLJ6yHKYsMRJMs9ibJ83gLDE8iMc5XprDh1oKXuaFoZChFUNFrdyCDe1Yjd-SSUuaWSTOkVfcZlKyuXLEzpsrNS0Z-gH4yl5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rO8OPYYEeaJpJE6ODEfHPiD_19P-ho1jo3ngOx29FAiNWzpRa6NOEc9AsAkiMW_83CrVrX56RcVPy9XZ6G0f079cV8F4CKbGlffUMy4pfe18P05GNsEwkrpRc457JHRRIjN_05vAeDE2oClfzcV2N-e-M2P868JgbppIOeTYShwW6Szg2IwB-qygNTHRi1-Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QxH-PI7Sx_mRLBD4mANo_04UcGE21Y2WC3EdQ2XufciQ3Rht4XopdTmNqIiHSypWmLf26MR5kGRfWGwBLxRIvyvz5JSpR5j-fiuyj44a8Euhg-PAwUlESUvJOTbiYDHTb8oIOPUXlJah4N8mQsq8rGQIeuML4p7CMi_oirrQBplzF8H0vS6k8G1ITrNnhceN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QOa9B7sZN-Zq-9gJeZEPzf2hyQhQUPeUDJ-Xv0sMVxI79xtF5skLIBwEYGI6Ov3WKWaqfKWAsbm812Sl9g7-CHCzV3LnJE-d-EcYfv1DBUNZgH3xrT0Z1dHc-pe_ZouznHyQUB_7zAVumvdOA4donT-ET3dk90V3avih9lBSedsi9o_ouUPfpHL-hZBbrNFY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X1RFDlzlwCKaF5R6fwjcu7F9ND34FfP-uZEIAf00zzkzybKaUXCE5_igBFoEWHrs42hF3hGWBtD5CnPDe5SJ9SPISlCSCG-OQXTeM8zJ5CEB7RXKbrQAJ2oZi0QX1zOmmIyNAd6QyLkyx4S7E2qkROvwr8Mz6zQAyhz1CUOiXN76QGdeIx3I9L9wTv_9fL1P?purpose=fullsize)

組合關係如下：

```text
           最上方工作平台
                  │
             Y 軸移動平台
                  │
             Y 軸滑台本體
                  │
        ──────────┴──────────
          固定在 X 軸移動平台
                  │
             X 軸移動平台
                  │
             X 軸滑台本體
                  │
               設備底座
```

這裡有一個非常重要的機械關係：

**Y 軸滑台的底座，固定在 X 軸的移動平台上。**

因此：

* X 軸移動：整組 Y 軸滑台跟著移動。
* Y 軸移動：只有 Y 軸自己的移動平台移動。

這就是兩軸可以分別控制的原因。

Parker 的精密定位技術手冊也有說明這種 Compound XY 組合：上層軸的底座固定在下層軸的移動滑座上。([Parker Hannifin Corporation][1])

---

# 三、你提出的另一種排列也可以！

你問：

> 有可能變成 Y 軸放在 X 軸滑台上面嗎？

完全可以。

例如：

### 配置 A：X 軸在下，Y 軸在上

```text
          工作平台
              │
          Y 軸滑台
              │
          X 軸移動平台
              │
          X 軸滑台
              │
            底座
```

### 配置 B：Y 軸在下，X 軸在上

```text
          工作平台
              │
          X 軸滑台
              │
          Y 軸移動平台
              │
          Y 軸滑台
              │
            底座
```

兩種配置都可以實現 XY 平面定位。

差別在於：

| 項目    | 配置 A       | 配置 B       |
| ----- | ---------- | ---------- |
| 下層軸   | X          | Y          |
| 上層軸   | Y          | X          |
| 下層移動時 | 整組 Y 軸移動   | 整組 X 軸移動   |
| 上層移動時 | 工作平台沿 Y 移動 | 工作平台沿 X 移動 |

所以你提出的想法完全正確。

實際採用哪一種，要考慮馬達位置、設備空間、負載、線材走向及精度需求。

---

# 四、還有一個重要觀念：X、Y 是方向，不是兩個平面

你剛才說：

> X 軸或是 Y 軸都是平面，擺放位置不同，前後或是左右可能就不同。

這裡我幫你修正一個觀念。

**X 和 Y 不是平面，而是兩個互相垂直的運動方向。**

例如：

![Image](https://images.openai.com/static-rsc-4/kN0llYRlN2UdogvVRJxrCNL9_J7ixq0uB1Dhsc2m4Sm9qJw8MIiUJK6uzdupyIacDl7wj2vPliNPgKoKQUT18tN_R15SF4H2FXH4jsT97oKuonWxd9VyUnuoe2RviYh19gAWp5JOsJwN0R-rOxCodL6lNbi2ixxBVGOrQW0OvWsaJco1I6jv6N7cs-9PpSBM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MxpGESFZpl4hLQTYbyT-ZlEG3nWYpCxCCqT9PrkIT14k4OSnu7aMvm8hk9_a4IAATwixYqxv4nsdzQVbZOrbL8O7NkmojGhwUNDe1NkPO3_D4mM-q7gyQ88vaFHDhCtRa10MzDmNhkTCbeLeeCFCaqShVe4I_X5U9Cr9L6P0lvwevggSrR0VfqLsxQinkGKO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wVsBw7FuwGjs6gQT7z1wZnh_tPdMb2gmF5zszR1e8CQ5-Y4qyYcykmsWPGxGyeqtYGSait4laemA5IVXLItgvCeE697TKV-TcSz3CCEBhsj9KJ8JFTuKhQ3CME_4LF-I7FTvaH57PMas_8I4n4EA6-vYSLC2uJPcWA8U0uLQccM53CjpijIHg5iXpzzKNxkj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8_qa6mgl8qmvDNcHVIpQK__vdsCnWQXuN1L1-YDsTLCpEObztefl-OmWNcpPZ9GADUv2cXH3aBVPaggOZAIhYGyRHwMg5DWgGQsCHpJA_OEy14F5X52_38ZcLoO_HYmvz1VRTzNiPqZDCb6ykkeuUaTaHGvBBOBlhksIkTpxtSgc1wZEvcM3cByCB2NwU-pk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xyFddf_HOLj4Kx0bmDXn1NQeFJb7nTMdFZJutFiO_-XPbKuECo4HlfIZOS5FLfbrtaFZaICLZkR3X3ulzuqI138YzRfP4lUnfOSVe5KGP3jtmACh6PgBiMiPICbSVE9PWBcIXKE4W2D9jmk3THLBoegMiwzngINHC2xKo5At6Jm48o2F523mIk9RkIaKWWju?purpose=fullsize)

假設我們從設備上方往下看：

* X 軸：左右方向。
* Y 軸：前後方向。

X 和 Y 兩個方向共同構成一個平面，這個平面就叫做 XY 平面。

也就是：

| 名稱    | 意義             |
| ----- | -------------- |
| X 軸   | 一個運動方向         |
| Y 軸   | 與 X 垂直的另一個運動方向 |
| XY 平面 | X、Y 兩個方向所構成的平面 |
| Z 軸   | 垂直於 XY 平面的方向   |

因此，不管 X 軸放在上面還是下面，只要整個設備的座標定義一致，仍然可以稱為 XY 平台。

---

# 五、還有其他組合方式嗎？不一定都要堆疊！

這一點也很重要。

除了上下堆疊，精密設備還有其他機械設計方式。

![Image](https://images.openai.com/static-rsc-4/9X1gaVV2jW1x0V6MF9oucg-AecoA9ODATviOrp7D2daSrc_EhWDEYUJnepqBXfNJYSgs6vJdYrJXkYn-Lp1fYWWmJgxNYbHidLV8tBhmFuINO4zHhZKjDCw75pI05_wDOEAa-xcoFkFAM2a6tuChU-_5M-6eTFcjQYicHmWImMO_KBbBJfSgQhytUrqAtQ5k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iTeZE_EYfOhXHfsUxMbudDuwDILb9oBnUYw5h6JWupLUIjnCTWRgWuPt4JuDaYmkkAs3xGorWmR4Ls6c8g2tfUvVJaV_beCaXP9-9P_VDMAR8okXb5aJTFeOa5nWOojn0evxWH5RJnAjs0qVRm1YDJq-48Mc_2Gfdc-pVJp7Kth6qOitAcl140yY6TkZ5ywi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ECkmtwgYBghq8qMLM4C2t0J7-RqW42KVRu9R0wa2GJqe2kOxwhiPwndnhG8fIxv12QX0EdmfmiM5bt4FxWzQ6o2GRdazMKYyfpOmYiPilEckzXOYw3cVRfzH37xxDf7xqFJkIFXECeQZ2JdC-0REohiw6NVd-GqZfY88bXaj5FPI3IDXnKD26kcyDUPLRL4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sD9wAIytS9RdwlblmQjNQ_usefLlfW_ZBX9TRaXPanDvXuXILPU3HvvEmIC08fNhCnHbwYbJ4HoK32a_K44wLRTWlWEFZO5yL5jqB9B--BR9dn68WnH4pbz9b2HqM1_0-7V5CDu6X4USqnyGPSisPLKiGvqOECu6wVWm3fN3XnC-fLrtUZgzgxeh2n9MYBWe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TL-uYhgu8DxvQoNyVfEkJfb4r8JjSe3PSjVgO-ochdRdz__pOTankhbgvuI8tt65M5owWKdIVc-wdI73HbZnsf13OB9-P7-ZZ7_lLwwIJANmQkKEeqsvZbyDUZJFsfIAw4ZhcpCyu983zxA_XUvdWwjxyZmhxoSVVa6uKf5FDnD_R65VLoHcfP3KgPRSwmFk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hrLOvKQtTNGit0Ry0wDxwEZznWzvFN-UZdQhED4K6y7o_CMqo3vRYYpx8w9iIBvI_c39z98BGcymTOSAqHyvAp5_nLp_uYBGw_jMlwJHMOWyUvMrjHPVB6MRn6D_Fi8BWHqcQeqdcvJ9OKASVMme8z87QdlMW1ymNOEAdszHrEBZU354u_tHHuNA4cRhCdBq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bGxHHtip3fDtnPOOZDHRlzbyrpwW6u1DMu0tgnKYRAXWd4K4HppsO2-Mc-KPtfPkmOys3YmSj3v5V3JCcAMRXPkaGEb6ThkW8XLb0HDQI4dc9mU5XNaVxeajeEfZnaUFO_pkzAByf6Q3tGoaNxxQzw7Eyl49m-Blf8BoC1ty6J-K6JTM170rDjtrd6EpryIl?purpose=fullsize)

主要可以先認識三種：

| 結構            | 機械特徵              |
| ------------- | ----------------- |
| Stacked XY    | 兩組單軸滑台上下堆疊        |
| Integrated XY | 將兩軸整合在同一套機械結構中    |
| Gantry XY     | 使用龍門架構，滑台沿橫樑及底座移動 |

例如：

**Stacked XY：** 結構容易理解，也方便使用標準單軸滑台組合。

**Integrated XY：** 兩軸機構整合，可能降低整體高度，並改善部分結構剛性。

**Gantry XY：** 常見於大型設備，例如雷射加工、檢測設備及自動化設備。

這些都是不同的 XY 運動架構，不應把所有 XY 平台都想成兩個滑台上下堆疊。

---

# 六、回到 GMT 的產品

GMT 官方產品中也有 XY 軸滑台，例如：

![Image](https://images.openai.com/static-rsc-4/sqgeyLJKrLt2a3TUws8mB_4_eP5JyHcxCMnBhoo40iC26uPAq39_12e2WLuZf57lN8w8DCQnhWVHa8QzGU3wzU8qFhAuz6zzBsL2v5Fukwxf7LbEWfdqmmwcxDP7qsejdGMf22bIExMLX1Ii3rtZPPAXlFaqD_3Qub9k2YpPqaWkWplOxVoTiJOhLEkvrWk9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6jnfdikX9k05yqlxsZdkY8aHmu7JKPPsN-968Pk72P_lSAaAiEioL5bBLuSVnxnyipnlHLhapKRJuWLzxmbKqzLHS9AJiFkhtUuHlmcI-V8_QZ6bYgxMyPxsXZjB0r7R5xBO00aAmyKbW5kzfLqCQuc15urIgVRsWvJmn649r6-_9vfz5rmurZ8v4CdVb8cU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xfq3wuQG_ZgIQLE9llGUdLZj8IH38ZiGt9NUZuH_wprLthyboM7n2rnZpX48h3VrlDdfIXRbEVKCE9r1poKFr2wBvIqnA7bfVgMAlkxrVqJJB6ovcGnUDPp1bLNSPeG5mSY0MJluyzL8IETDN3HYXzV8Fnb7YCP_-hkjHn5hX8hb4cdvajQB5TbEJocyX_FH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-SIPRBtK2tb7v_kzZf8fPdikxiOFyp7AQmlgswyMiAL7TrOrxyrP19sMYH2Q6k1gemNUkTAsluNeiAMj9sh2T0PKht8JbVjgvPsxwlpocQ5tQPxBVpwkKlSTn7UbWLrV0bhAnFyAkfMAWaiRIfMAkn0EOraJYrZxqBaZAkhp9PhuuxWDLzq0oRTuWWp3Fx2v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2ID6YGUcOomY4tDh_tKxrRssjqjFSelmEfFweU43fB5IYXi-IbnS-xH9hx49gpZEgnPNfUCw9J-qML7UvTuaUn_0PvhypcNBrtkWf8Fh_Fk4xboxzeE6w1RPSFlAIXuH-3KF4uULya5CGnHCQ-d5hiaKqtq-4XVlr82tsK3MLSLufmrKISv7KEdmFXbNMPZQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4-tYU3eCvTXQPm2Bh9dAbkXjfYZ1MeurVJo7Qdr_giOB2bTMP23KBzNWWunoe6ZXRYG7krBbK5vLmJSnL1M5v6jWC2WdJgv1ObiiEPKXod9Q-jVujI7ynclZx45CUufaiDTdSmnpOWNGbBL_-yRiTttqnAYdguA9JlY9544ck7rQnUYTk5XcGRlpvAhyF8vt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gQXRw6NBGvSPFPGAc3VnkVGH8xTUDDYCEBZgS4yyGcJIDp3jxAnzE-XVa4ow1vvxNbYT8m71Reo75s1-4zqIpJOHPjna-O0G5NG3ekLQ-ObzELEEJUskuD6RoX94DDQl0q130uBHk83nY5D4IQdCbUkSdLIq17ov7-_Zfwxaoov9P9RBLALcJ9RVBU2r2V2k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8hkpy3jB69x7yr0C1OncMl_QWMULdJci6NtSs_mmIUAk_8OfPNrU1kYoz8sShMzdIq48uA7vgCt28sLkXBla_DinxhJF20z4Eb2cOix3dN_brhJKXBxqeWaw82bMGuIYj-DZmatzSM28qfw47LosuB75JUpdZoUGugcv7ipx-Pv3g-0b1LlUq11OUZU_ooFo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2LwxvAbO_KIH2OHgio7yQnwys_8ev4yYbuqPIEXI_8UCS-skrbPiM9akDZGTVq5Jk8Gocn-UcGxc05X1kZDlnHqkZwjyb0-0x309nm99ndaPgUaIeeVoV338P94upXk8mqgDr-aOxGtkqgS4v4T2AyoaMSX5gDQi7oolAo-s3oU2fQh4pe3ifDi1DD-rPlhg?purpose=fullsize)

可以參考：

* [GMT CYS50 精密型 XY 滑台](https://www.gmtglobalinc.com/en/xy-axis-3/cys50-precision?utm_source=chatgpt.com)
* [GMT AY120 精密型 XY 滑台](https://www.gmtglobalinc.com/en/xy-axis-3/ay120-precision-type?utm_source=chatgpt.com)

這些產品頁面包含產品外觀、系統配置及技術資料。要注意，產品外觀照片不一定能完整顯示內部的固定方式；若要確認特定型號的安裝結構，仍需看該型號的技術圖面。

---

## 七、今天先確認你真正理解了沒有

我們先不急著進入 XYZ。

請你想像：

現在有兩組單軸滑台：

* 第一組：X 軸。
* 第二組：Y 軸。

Y 軸的底座固定在 X 軸的移動平台上。

假設：

1. X 軸向右移動 10 mm。
2. Y 軸維持不動。

請問：

**問題 1：** Y 軸的馬達、底座、滑軌及移動平台，會不會跟著 X 軸一起向右移動？

**問題 2：** 如果 X 軸維持不動，只有 Y 軸移動 5 mm，X 軸的底座會不會跟著移動？

這兩題答對，就代表你已經理解上下堆疊式 XY 滑台最重要的機械組合原理。

[1]: https://www.parker.com/content/dam/Parker-com/Literature/Electromechanical-Europe/Literature/192_591011_Precision-Technology.pdf "N3"


---

你的兩題都答對了！而且你現在開始注意到一個很重要的問題：**為什麼我們學習時看到的滑台有明顯的滑軌、螺桿，但 GMT 的實際產品照片卻像一個密封的金屬方塊？**

這不是因為導程太短，而是因為產品採用了緊湊式機械結構，許多零件被包在底座與移動平台之間。

我們用 GMT 的實際產品來看。

# 一、先看 GMT CYS50 XY 精密滑台

![Image](https://images.openai.com/static-rsc-4/jt89w9JShKQg5Mj8Vu-JYywBJWxK3sGQfxlzJX4lwn7m-_TnqVxHVfKKMDROSqp7HZjrxLeqKKDp87tXM30mYtLl9HV22VKH3cr_i2kIvpRIta4Y231hy9KHKwjGLJ-89JSPIA20bExuATwOpusWRAxFwLNoGOW86LHhYq8yDmQ9_pGm_j0TXW50IMrFkZjL?purpose=fullsize)

GMT 官方產品資料列出的機械結構是：

| 零件   | CYS50 精密型規格 |
| ---- | ----------- |
| 傳動機構 | 滾珠螺桿        |
| 螺桿直徑 | Ø8 mm       |
| 螺桿導程 | 1 mm        |
| 導引機構 | 線性滾珠滑軌      |
| 台面尺寸 | 50 × 50 mm  |
| 移動行程 | 20 或 30 mm  |

資料來源：[GMT CYS50 精密型官方產品頁](https://www.gmtglobalinc.com/en/xy-axis-3/cys50-precision?utm_source=chatgpt.com)

你可以看到，**它確實有滾珠螺桿及線性滾珠滑軌，只是從產品外觀不容易看到。**

---

# 二、為什麼看不到滑軌？

我們先把滑台想像成三層：

![Image](https://images.openai.com/static-rsc-4/IkfMBqoakNOnehhD0XR9Q4FXs3y9Zrz7IvlZUgG5FO1pqEwcraFAdD9mkN4NuQYg75N7CxhMvYglDBO612YJWvARK-Ee4AMX2G8S7GzhMEQy6pI23ovZT7Bju8c0NS_xJBg4XSJ8kmt8LroIgngHNypaOA3Orvp2vFImBndpnhlkGoggFjZbZZ0EeK-MhNfv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Cb0-4FBQFXdnwDnf52EheeLR0jByfT2Gy09cYoGoMDioUe2ftpaHu32Ziuwt_M50Nel4A7U8Q7HU8ml8VlGOZ6FmkwhUdZ0XhkLPnQqjI3dyCdidbPEWlOK0VKbWXxDkg3BOdoFdjKf3_LDfFQnQqNnwHs1r6esOJSeDttDVE2SiaqW4wtA2ugPkEXC3MYLA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J9dIzoeXWJr18N0ZnZo-_RzSvFp_T8kg8HVSyfKX09D8peKPHcHZrQApvUFFGLOvm4ddJRz_6xOxKjmUqZasiWE-lY3IjT7c2zEv_3SrMv7F0_Gl4s_szNUpxBONQFDLw0hvV7sTDDRmYHcRxNc9FKiNR4g1dTsR1HHfbUIvwWI1DtC3_XjakbdfhCTc_3Gq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZaOnw3nKJ1oFKZ76o9ap4jJIiI3AwWNVF5ejDFgGAtKlFLfp1-SJExhBpcM76zYX05dxNJ06y2YvnLRq166j7Ho8t4Gs5ONxpeztKSjduOGHVHkBx94v3lObaNgeo92Lexuc6dTo2iPhHMRvyBl0x1601WJqVae0AeWaqit2gnAwOKneEhMdWfyt57mF5bLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SHFjumwhqHtZaU8oICN7C1ia0a66_3H2h2RWHq_JSkzlfD3JnD2u-Jwt1Y8gMfkEvGqwvifAJ7sL-GoELli4qYiMgVjFeYpFnc33oOq0nC_AoTd2-bflB5ldvhenEczmh_Hx7tdU3gfw8aneop0FEk-qwKs1f8jsW9itaUOldSI73_Fz-htaDT8KbUd1HCqH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/r59oJVuhMvCY-m4qJXC_ZYE-KpGtXSlFLC_rdizUReR7bo9fmBlJoTwEwBmIIpNksunFDEmv3ihYV4dcL94oLYWkaTprr3U6lCTm9Fm6aXMvtlVQEH_ZjJuYQkCbdwn6rPZ7oDK55zrapi_xxexSbDazyJHoIzhuEvr8dhxvVK0cO_SqTHCjmOrBUZqgQ4vT?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tM1lQEgFXi4vMT4HfQeM-hN8KH3NUqZjEXqfqCcN8yR_-B7keb1zgHq7OmVKTquaj7uanmmZLOURsJUICK3CFfaTAZ1Ti_BBYUOzN3zykZVKltclPzllUueOB-IByK_1F1qe-ni5Ym1lHz8jGvvIGsQtH7iY4Vf5ayqyfloDK7xu5Y_3QFfqCVQMaE88Upkb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TRJvhoWdWfMT5yK0bt7jyJ0UryDmPnWZJzEhwBmU0ll_Aq2V0GjNY4KmPONJDYMLP62TW2Jx91Knw6wFMKkqokj4wTKp_yaOHwP35f08QHsf5_Ex6BbJGQGz2QD-9qZ6PHO-RkUw6b4nIC25FJa_d0Y_TU_1LQ8SRyH43aS1CMjYwPcTO35J6zJhyL49S2eA?purpose=fullsize)

由上往下看：

```text
       第一層：移動平台（Table）
       ┌─────────────────┐
       │                 │
       │    工作台面      │
       └─────────────────┘
                 │
       第二層：內部機構
       ┌─────────────────┐
       │  滑軌       滑軌 │
       │                 │
       │     滾珠螺桿    │
       │                 │
       └─────────────────┘
                 │
       第三層：固定底座（Base）
       ┌─────────────────┐
       │                 │
       │    固定底座      │
       └─────────────────┘
```

這是簡化的剖面示意，不是 CYS50 的實際尺寸圖。

重點是：

* 滑軌通常藏在移動平台與底座之間。
* 滑塊也被移動平台遮住。
* 滾珠螺桿通常安裝在機構內部。
* 螺帽與移動平台連接，因此外面不一定看得到。

所以你從產品正面看，只會看到一個金屬外殼和移動平台。

這種設計有幾個好處：

1. 節省空間。
2. 保護內部精密機構。
3. 降低灰塵或異物進入的機會。
4. 讓整體機構更加緊湊。

---

# 三、你提到導程太短，這裡要釐清一個概念

你把「導程」和「螺桿的長度」聯想在一起了。

其實兩者不同。

以 CYS50 為例：

* 螺桿直徑：8 mm。
* 導程：1 mm。
* 移動行程：20 或 30 mm。

**導程 1 mm 的意思是螺桿旋轉一圈，螺帽理論上移動 1 mm。**

它不是說螺桿只有 1 mm 長。

例如：

| 螺桿旋轉圈數 | 理論移動距離 |
| ------ | -----: |
| 1 圈    |   1 mm |
| 5 圈    |   5 mm |
| 10 圈   |  10 mm |
| 20 圈   |  20 mm |

螺桿實際長度需要足以配合滑台的行程、螺帽及兩端支撐結構。

因此，**你看不到螺桿，主要是因為它藏在機構內部，而不是因為導程只有 1 mm。**

---

# 四、還有一個有趣的地方：不是所有 GMT XY 滑台都有滾珠螺桿

這點值得你注意。

例如 GMT 的 MY100-SC 手動 XY 滑台：

![Image](https://images.openai.com/static-rsc-4/Tz8c14usQtVhVH4Lcld1qw6-F5V0QF-gXrDLSFjV-3o36m4_sDDXDOe77lMeeMbkZBlaq2bAunGs3xaKxGw2rlBy1M-oVv4Zq725cY-U9PM0JhLNeJhjTS2Ht7lB8Nj2k9N9KIqY710YYWSVFwEASCYkjWc13nXEhhHDMnwfgaOE4ZzoyjYFutSSU3bXx3wy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/p6jZmUg8Q-CDmC8XDP8i4bYuqW6FJ6lOcCLU-iMvL51jvOw3b0zINvHbtztyNTLCaoKhJPSsiLxlvIN6mT25-pxZDed3bYB1fKO1WyxMGMAVjzHaWo-fxjz1N7X9QtvyGEhNYEw2cdbXYcaGpDr3R_5rEURCzo-XLTqa40GwpcOHb9Aa7uKDsQAzoRCj_l66?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7O76rw7DEhcdYAWRWbqWc2lmznb4XHQvr7jC-9CmUML64Ik19gkkjMwKnm6EWOf-MiWIaK2EXarAcfJvludAopJsUTjT6Yik54W9VQ7bZMj3TMhu6EKwlt58CTntACnSdKAfGICPEdDJsFzlsKvyQl-5gs7bwEV4NtgdxN8kRDl6-NPvpylcI1xrsbMP7hvN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-SIPRBtK2tb7v_kzZf8fPdikxiOFyp7AQmlgswyMiAL7TrOrxyrP19sMYH2Q6k1gemNUkTAsluNeiAMj9sh2T0PKht8JbVjgvPsxwlpocQ5tQPxBVpwkKlSTn7UbWLrV0bhAnFyAkfMAWaiRIfMAkn0EOraJYrZxqBaZAkhp9PhuuxWDLzq0oRTuWWp3Fx2v?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/n347Fyk6Kw8nVSEeCb-5YDQr7zijLowmwykf9ipv-Ln0DerD169JmFBrwVU7A_iVE3qK6AhiSHCH9Xfrn-ljKsNJhxdmUkJ5V0ZfisNRPevBKhQ9cB40dW9lkNbpWVvUTknsumkNh6oQP5NSaoFFSwmtuNAVdEsjPE0Fd047jRw89HJVBWWTL8dQFrntl08u?purpose=fullsize)

官方資料列出：

* 導引：線性滾珠軸承。
* 傳動：測微頭（Micrometer Head）。
* 不是滾珠螺桿電動驅動架構。

資料來源：[GMT SUS Stage XY 軸產品資料](https://www.gmt.tw/en/linear-ball-stage-sus-stage-xy-axis?utm_source=chatgpt.com)

這代表：

即使都是 XY 滑台，內部機械結構也可能不同。

我們目前學到的滾珠螺桿式電動滑台，只是其中一種。

---

# 五、我找到 GMT 官方的 XY 機械圖面資料

這份資料值得你留著：

**[GMT 官方 XY 軸產品型錄 PDF](https://www.gmtglobalinc.com/Images/uploaded/Catalogue/EN/11-XY-TABLE-EN.pdf?utm_source=chatgpt.com)**

裡面有工程圖、產品尺寸及安裝注意事項，可以看到比一般產品照片更多的機械資訊。

另外，GMT 的 CYS50 官方產品頁也有 Image Download 區域，可以進一步查找產品圖面。

---

## 六、今天請你記住這三件事

1. XY 滑台內部仍然可能有滑軌、滑塊、滾珠螺桿及螺帽，只是被外殼和移動平台遮住。
2. 導程是螺桿每旋轉一圈所產生的理論直線位移，不是螺桿長度。
3. GMT 的 XY 滑台不一定全部採用相同傳動方式，電動型與手動型可能有不同機械結構。

**我建議下一步直接拿 GMT CYS50 的工程圖，練習辨識底座、移動平台、滑軌、螺桿和馬達的位置。** 這比繼續看抽象的方塊圖更有幫助。


---

很好，我建議把這部分分成**兩個階段**，而且不只是看產品照片，而是要學會從工程圖判斷機構是怎麼組合、哪個零件會動、哪個零件固定。

我們會以 GMT 的實際產品為主，再搭配必要的機械剖面示意圖。你不需要先具備機械工程背景，我會從「看得懂零件、看得懂運動」開始教。

# 課程安排：從 XYZ 到六軸定位平台

| 階段   | 學習內容                  | 主要目的                        |
| ---- | --------------------- | --------------------------- |
| 第一階段 | CYS50 XY 工程圖、XYZ 三軸組合 | 看懂底座、移動平台、滑軌、螺桿、馬達，以及三軸如何堆疊 |
| 第二階段 | RX、RY、RZ 三個旋轉自由度      | 看懂旋轉軸、傾斜軸，以及六軸平台如何實現精密對位    |

先提醒一個重要觀念：

**XYZ 是三個直線移動方向；RX、RY、RZ 是繞著 X、Y、Z 軸旋轉。**

這六個自由度合起來，就是常說的 6-DOF（Six Degrees of Freedom，六自由度）。

---

# 第一階段：先從 GMT CYS50 XY 工程圖開始

## 1. 先看實際產品外觀

![Image](https://images.openai.com/static-rsc-4/jt89w9JShKQg5Mj8Vu-JYywBJWxK3sGQfxlzJX4lwn7m-_TnqVxHVfKKMDROSqp7HZjrxLeqKKDp87tXM30mYtLl9HV22VKH3cr_i2kIvpRIta4Y231hy9KHKwjGLJ-89JSPIA20bExuATwOpusWRAxFwLNoGOW86LHhYq8yDmQ9_pGm_j0TXW50IMrFkZjL?purpose=fullsize)

這是我們要研究的 GMT CYS50 精密型 XY 滑台。

官方資料確認它使用：

* 線性滾珠導引（Linear Ball Guide）
* Ø8 mm 滾珠螺桿，導程 1 mm
* 兩個互相垂直的直線運動軸
* 50 × 50 mm 台面
* 20 mm 或 30 mm 行程版本
* 五相步進馬達

以上資料來自 GMT 官方產品頁。([高明鐵 GMT][1])

官方產品頁：[GMT CYS50 精密型](https://www.gmtglobalinc.com/en/xy-axis-3/cys50-precision?utm_source=chatgpt.com)

## 2. 先理解工程圖的三種視角

看機械工程圖時，最常遇到三種視角：

![Image](https://images.openai.com/static-rsc-4/Ha2_Bspiu_SC83ybDm2sR-20UsIKU9iDw5X9pH3Pn2jvQ400RN2t0UnSV-qoJJIcElVklXvD3-F0Tj4KCq3XWghvQXG5ot6vwVO97p3ViTW-EsVGh7ln_8WnWCrkGcm1e5lLROPHz6GhW475GSgj3k0Xy68WjEL9acDgbC_6eDh__ES1IWF--3a7PDGEWuJk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/j5XjD_VTqj2Wr965SkWROQiIrSQQzRYFDPQHyzAGTkMFfJaGNjMWgnf-cKTLlY7KZUR7gxXt5L-UVTeDIMxBRMmXQoFdVmN43N7FpCBwyd1wvPlTSCdiqhno7yvNaO2dSoIv436Fz_vPMEJYAsle4M6GKk5XK5L4HY2Oq279Vw94htNZjTQxQgZoxEFdYHZV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-HRXayU8Zbiw4AHb0IeRZVd7vnQ7yZXS0mx_BXm2pPDLSV8bsc5cy-lXiYUmz5QEPVRQUP3YXL7gg4YSW73XHZk3k7FmAFOIopTRAIm1L56JdreY-UdJG84W_itBdFkVqFIP5DoXnEzYRrrXZLq9wSkndFj9VN_YDEj1Trp1fsjOzR9z39m4318itP1X08Is?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XBY_XFtnEkFE1dW6S5ID0f94g4csjwvZRJESLGrYLk5ws2WObfQJRItK6WN2J_6-TxJjUYCs093o_NCrDS4noqwHg0Yfrb_YUyC8wGhdxqQzuEGpr4oc-5n7RSRYaZBFBqSlcrWkSBdHxrp0u4rx_V0XVbE2mNJkYGMDNKcC24-1G0FKq-iTk3A_VNS3L4ns?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2DbHDNfBwF3GMA0CgeofYy6y9kTsq926C42QRM1Qs7_L_EEyL72BPTX5_pFJNadjEG5xnTGSpHVJzYWUKyFdV33Hh1wYLXtOcLJ6yHKYsMRJMs9ibJ83gLDE8iMc5XprDh1oKXuaFoZChFUNFrdyCDe1Yjd-SSUuaWSTOkVfcZlKyuXLEzpsrNS0Z-gH4yl5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SHFjumwhqHtZaU8oICN7C1ia0a66_3H2h2RWHq_JSkzlfD3JnD2u-Jwt1Y8gMfkEvGqwvifAJ7sL-GoELli4qYiMgVjFeYpFnc33oOq0nC_AoTd2-bflB5ldvhenEczmh_Hx7tdU3gfw8aneop0FEk-qwKs1f8jsW9itaUOldSI73_Fz-htaDT8KbUd1HCqH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VjGMy45rVskKoiPIXxW8y0OlhsQ2s-dSsyn0BYC_IuBtE-ujSeGRiSuY8F7XfinOPxc4te3i9JPuHqsTqHpUYoxlt9eTlFDetieHYgYyymPWJ_ZmzAk63pv8lgGcvpmGynxHnLKqXOZBORsPYBK2uAYCMMd6pmjYtkGZd9P51ZbuuYqGT7zz6ZdUDZNqLTwo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5KeEsS-_QvMLw8FqYIXEt1rFvE2icFWW6lgPKjUSulBGHmj61FteLkcGXZZbA51xg6ns__rRs0LrKBUd7_foaO6Sw7Hc_drlzVdFHEdRZ1HzJWFwpKgUQcD-qwPgqP9-7yuTkmCSlKgsR0rZFEtsPlcqHFAyBkkoFc04pKVl5JmRgNw1FcsWJAUXDPPOn4_f?purpose=fullsize)

| 視角                    | 主要用途  | 我們要看什麼          |
| --------------------- | ----- | --------------- |
| 俯視圖（Top View）         | 從上往下看 | 台面形狀、螺絲孔、X/Y 方向 |
| 側視圖（Side View）        | 從側面看  | 底座厚度、平台高度、馬達位置  |
| 等角透視圖（Isometric View） | 立體角度看 | 零件如何組合、上下層關係    |

另外還有一種非常重要的圖：

**剖面圖（Section View）**。

它就像把滑台切開一部分，讓我們看見外殼裡面的螺桿、螺帽、滑軌和滑塊。

CYS50 的外觀照片不會把所有內部零件都展示出來，因此我們會把官方尺寸圖與機械原理剖面示意圖搭配使用，不會把示意圖誤當成 GMT 的實際內部結構。

## 3. 我們要在 CYS50 上辨認的五個主要部分

接下來先建立辨識目標。

### A. 底座（Base）

底座是整個滑台的固定基礎。

它通常固定在設備的機架、光學平台或另一個滑台上。

你可以把它理解成整個機構的地基。

### B. 移動平台（Moving Table / Stage）

這是上方可以承載相機、光學元件、光纖夾具等物品的部分。

當螺桿推動螺帽時，螺帽帶動移動平台，平台就沿著滑軌移動。

### C. 線性滑軌（Linear Guide Rail）

滑軌負責支撐和導引平台。

它的工作不是主動推動平台，而是讓平台只能沿著指定方向平順移動，並承受負載。

### D. 滑塊（Linear Guide Block）

滑塊與移動平台連接，沿著滑軌移動。

它與滾珠螺帽是兩種不同零件：

* 滑塊：支撐、導引平台。
* 滾珠螺帽：接受螺桿的旋轉運動，轉換成直線推力。

### E. 滾珠螺桿與馬達

這兩個零件負責提供運動：

* 馬達：產生旋轉動力。
* 聯軸器：連接馬達軸與螺桿。
* 滾珠螺桿：把旋轉轉成直線運動。
* 滾珠螺帽：將推力傳遞給移動平台。

這五個部分就是我們要從 CYS50 工程圖中找出來的重點。

---

# 4. 從 XY 延伸到 XYZ 三軸

先看三軸組合的概念：

![Image](https://images.openai.com/static-rsc-4/vufQ_LbV5_GjFPKQWTlCwnOJJfgXsBdf8QHYs20vGTN9KXCbMgKe717WFF38yzhDBtDH5L8VkjaM8-586iRusu2mxGNZ2F3DnivikZJkXxKOmCmfVOhlcMCBF5CzZvjl-yVnmnadAaiODre0Zh3H4xkiG-K0vgprsHOhs6xZonaKLGSKugH8xd1WJbqtGERf?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yACzXO-cgGAZKeuv-RLTtvJ1rs5QSzjRooRlM3zkoLlhPM7kyReGsQDoNf_KNjeQgqQKjVOQGmpGBc-Za6pnixD2tHgtn-Eksk_tlqkPGe8Nn54csh18MRdIp-98PnncMPnrQrJFfWeRq0cFT_m3rdcskf7gUr-cqlpsWhLccUsBpgB_an2OugJDr5OYp63h?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gNcCla3YSVMdfNJABZysSpcM1re6iAcZ5bBShyMaaBzxXvy_VLI9F9gQ-4NofjZts3bwwjBK5BMTlXxZ_zmISpTStmtYnYLta6_Oi3pcOn1A5kIfKDTOKs0IWtY3NzSUFLQf8OkhZh2doMKE4tOPdcP9-AllK4et-f4mk2y0gyLBN-1MaN7zQEbOjk5MbVZa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iTeZE_EYfOhXHfsUxMbudDuwDILb9oBnUYw5h6JWupLUIjnCTWRgWuPt4JuDaYmkkAs3xGorWmR4Ls6c8g2tfUvVJaV_beCaXP9-9P_VDMAR8okXb5aJTFeOa5nWOojn0evxWH5RJnAjs0qVRm1YDJq-48Mc_2Gfdc-pVJp7Kth6qOitAcl140yY6TkZ5ywi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LCY42MQT7JlqB6bQDUrCDIg8EeLJjC3ICxG-JII1XgmUBbsVyv4u20vPo2hfWciIAVbXvSQA3xu1taTvoeT1rLLiLAnd1mDkv2u1v204p8kpCsu0Z-RVxgmzDo5vCgvLTBZK0tSB4Z4oOTeHeF5woOC9CRZxOhQjx2Hbv5WdfHAkVJXOYwyIWQw_mtVp8gbU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Tmf4QDAymxywb_CKdpbuLhzd93TEVPB0eZ2mgYvUlcngv9Kb7vu08ZzdGiGjBmi7ypC_nGQNw2j9T9v59ATIqznejO78wZs27lBxns5eYGrlfsESUU8zr3baX8J2tE3c-sipplkmOrM5L4aaCUXDDOQZBl72KV0QJaVcPazKtqICjojLZqUC0ZoNFs_Jjap8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3U3SG5O--FGiaKW4Xpne_acP3G-facns0CHvTrHLOuh9qHZQw7JDlA1fv_Fg-B2XYND0N8w1uL7GFuH-HTK6FNIowi4qCEIAn6DRt9KyKW8VVwYe4tFMvTQP6-1xU3DBqYEOp8eyvQW7SOAry0w9RXKp-NmSW7yKxqBQUY7TwzDWNysKgDqOU6F2pGkkzKne?purpose=fullsize)

XYZ 三軸通常可以想成：

1. X 軸：提供一個水平方向的移動。
2. Y 軸：提供另一個水平方向的移動。
3. Z 軸：提供垂直方向的升降。

其中，常見的堆疊方式是：

* Y 軸底座固定在 X 軸移動平台上。
* Z 軸底座固定在 Y 軸移動平台上。

因此：

* X 軸移動時，Y 軸與 Z 軸整組跟著移動。
* Y 軸移動時，Z 軸跟著移動。
* Z 軸移動時，只有 Z 軸的移動平台及其負載移動。

這個觀念對你之後理解 STM32 控制多軸非常重要。

例如 STM32 同時要求 X、Y、Z 移動，實際上就是三個不同的機構，各自執行指定方向的運動。

不過，如果要求末端沿著一條斜線移動，就不能只是讓三軸各自隨意移動，還必須讓控制器協調三軸的運動時間與速度。

---

# 第二階段預告：RX、RY、RZ 三個旋轉軸

這部分會是我們進入光纖耦合、矽光子對位前非常重要的基礎。

![Image](https://images.openai.com/static-rsc-4/2wP9Lq9XiOfDxyxTfHIytraLvwPwUSj6soMKGMczdIsCowZjc43nybNIWVdYXtPGX04yrDKripoT241l7RcIbgULV3m2ngj4q3rYGqYjc0BJwNyRZLamlROQLFWpl4isqQgdkMesvbq8-a3QSvlTMIbj1MsLgw5HNM_2xEBnKuWQUp5JAIpnZOiPUsYUm19r?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TfWq7L8Hfo1NtxDDTPd6kL3ypwFgG1JzdghNRMo0S96JVYovfM9yXoQEHVIGynCRcLVaaehCj8JKcntmBkp35xvl45HSLIB5o9Kuxsj44FQIj6j8hk8DZzRs_XRlcxM5MwMXQvA_J07I3-OF6_QOcnJYu5HXqW8S3MmTagj4QSi2nzGSByNRloibYK-nMuW4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vIRr1iWOOgqVawqimbhLFrkhT7irpwp2n-1_2X_49LDPF5sLfLUH-OZafGPtImllRgusPNBKEIlv6KTVmw83eNw4e9euYheHY2XXyN7ebyA-_tilI0ESfB1DC1PHUCt2X6Mbn9t0hfeLGtfWo3UWJmO5x_JlTrpii_aetZ2DnsFWL9h_QYiZIkR1k1ljLJd9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VoDtkGa0Cycxu9HKS74vkaP_XO2EY7xJvnvAei20U9YHePIcvOeertNxFgFmO696YM-B-40Uh6elllAfcNf4YGfSR79CNAJSiMV0p7X_Cb5EycSyrj2MPUyKVkdH-1kqkb6PfAIhNi93_UXkVfI_M9Y2nOK_s4Pa7nfpIKzTHavz7Ach_1CD0MoXUn7B30Cq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W_2vBJ7o5vxzq_iFwvR_wPCROdvz_GgHiZniHLyCRXijbkTXbaqW_DlP1KeS29EJxcUiEFJp11ihvoYZ5BZ9oi9UoeJ_XAUikMyhpbgGwqk-K3YkS4bQMjVREPoRSqm6fAK3gZlWosExyNsH60iD8i0APvNv4ISKshogv0Dy_arKt9zVIQLSa5StIgnfNJxd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/q2U4MclHn5yVMCQaPCqzAdq4EmRzdfP_oY8SZtpedQSW3XqKSs820iJMLZkB5IES1acNp7MPB1_FpUluPDsBal5SK2SotgWBUmvqfOkCxkUvw9TLouomQ4jRl2_2G1ELehmFb9mzfZEh1u2MkRcN0ExzIiSdrynZsSK3gVySRUY3Nqp7LuEpSJvfJSBh6wJZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qi4BMXFjm1nW9hA-BI2DqHwOtztRPY2TIoTkiHZRz8nIazIq6gIPyL1vCO3cwaQwd5F1gW4dPMv7mY4blihvcndJGQg_4llJrFjCLyo62qOnv4zRXjWYhDheDPZ6HwmCoOzPYZMrNzfcpkC1feyGWGjXh7Nv-3_bZ6LjadAomcZ0VBVCwyFPQH8CRuMQhqRO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kxHH7N7KPnzIEoSXz0Oyje6k5GODfec5jc5yE1wEegJkdVRbjVqIARxnQ9L55uXRcbUE-GS5GN7CQ2rADjzzykT6_nJFvOTWAC3l5mmXafM9MRrGPLpTxd14SWoisLqOHqEaw3rsQVNydTio7fDqPgbG4DTbVg1cTPpeCxPZlrp1B2jlnlmsovoYGmVLxpSG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nbciy4c39TespXjxBepEQ0lr9xLoLAkI_qKxf1BvIWEcMKmkuJ49CU98Cm4QoZ79svgT8ZgZ6AjH5eChnigzg1FvJW1DkqMil3KteT-EVwgIt8L41dcp6ElaxUVYQUvWccQpoD1aHTemIMBzEBz87sxz0GXwiWDAZ8w5XLGTUKWM0dMFlQqoEyFouQV6XlVk?purpose=fullsize)

六自由度可以這樣理解：

| 軸  | 運動方式    | 簡單理解    |
| -- | ------- | ------- |
| X  | 左右直線移動  | 左右調整    |
| Y  | 前後直線移動  | 前後調整    |
| Z  | 上下直線移動  | 高度調整    |
| RX | 繞 X 軸旋轉 | 繞 X 軸傾斜 |
| RY | 繞 Y 軸旋轉 | 繞 Y 軸傾斜 |
| RZ | 繞 Z 軸旋轉 | 水平面內旋轉  |

這裡還有一個值得注意的 GMT 特點：

GMT 官方軸向定義中，直線運動使用 X、Y、Z；繞 X、Y、Z 的旋轉則分別使用 α、β、θ。也就是說，我們在一般六自由度資料中看到的 RX、RY、RZ，與 GMT 的 α、β、θ 是對應的旋轉概念。([高明鐵 GMT][2])

六軸平台有不同的機械架構。其中一種是 **Hexapod（六足並聯平台）**，它不是把六個單軸滑台簡單疊在一起，而是透過六支可改變長度的支柱共同控制上平台的位置和姿態。

這種架構很適合用來理解六自由度精密對位。

---

## 我們接下來正式開始的方式

我建議先不要急著一次把六軸全部學完。

**第一堂先專心看懂 CYS50 XY 工程圖。**

我會帶你依序辨認：

1. 哪個是固定底座？
2. 哪個是移動平台？
3. 哪裡是 X 軸的滑軌？
4. 哪裡是 Y 軸的滑軌？
5. 滾珠螺桿藏在哪裡？
6. 馬達和聯軸器在哪個位置？
7. X 軸移動時，Y 軸的哪些零件會跟著移動？

我會以 GMT 官方工程圖為依據，並在必要時用剖面示意圖補充看不到的內部構造。

**這堂課最重要的目標不是背零件名稱，而是你看到一張滑台照片或工程圖時，能夠在腦中想像它內部的機構，以及馬達轉動後究竟是哪個零件在移動。**

[1]: https://www.gmtglobalinc.com/en/xy-axis-3/cys50-precision?utm_source=chatgpt.com "CYS50 - Precision | XY-axis | GMT GLOBAL"
[2]: https://www.gmtglobalinc.com/en/xy-axis-3/cys50-precision "CYS50 - Precision | XY-axis | GMT GLOBAL"


---

這個問題問得很好，而且你已經開始從「機械結構」進入「三維空間定位」的概念了。

你注意到一個很重要的現象：

**X 軸在最底層，Y、Z 軸都安裝在 X 軸上面。因此 X 軸移動時，Y、Z 整組也會跟著移動。那麼，控制器怎麼知道最後的工具或平台會到達三維空間中的哪一個點？**

答案是：**控制器不是分別考慮每個滑台的外觀位置，而是利用三個軸的座標值，計算末端平台在共同座標系中的位置。**

我們一步一步看。

# 一、先理解 XYZ 三軸的堆疊關係

假設機構是：

* X 軸在最底層。
* Y 軸固定在 X 軸的移動平台上。
* Z 軸固定在 Y 軸的移動平台上。
* 最上方的 Z 軸平台承載相機或光學元件。

![Image](https://images.openai.com/static-rsc-4/K6GwSHGHF9h4NIKXPhRSiWkptSPS7PzemOROuQ_su6LYucQ-5qox9D0FneIKLtiaMUlYBujH-wvC7O4GanvxYoHULIl5d1JcnVbNgmjCflQlaq4SjkXWPFKEaKyFYx9PDpVNXtCN3ieE3wvu7HGLYQHlPxW90xUyo3Yqh4BoITVRSObGYJZAbOxOtgOi-F53?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gNcCla3YSVMdfNJABZysSpcM1re6iAcZ5bBShyMaaBzxXvy_VLI9F9gQ-4NofjZts3bwwjBK5BMTlXxZ_zmISpTStmtYnYLta6_Oi3pcOn1A5kIfKDTOKs0IWtY3NzSUFLQf8OkhZh2doMKE4tOPdcP9-AllK4et-f4mk2y0gyLBN-1MaN7zQEbOjk5MbVZa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2ebGASNL3hnk6KygDmIjeqByiNrA6JTWlMfAooEGPepHALmbNG4gk9cP9UtKIKgpCIZV3dtuwv9BWvfcNtABSgDn4yAylH9Bd4XV7FVGf0p-9xe_uRX9dIFLJ5ol5IND-ltYRi4XWBZJQuzz0OO2_eRe1LojJe2RGPNzr0Liqp2bkJ3AmB8aABtnpDubtpzs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IgecYj_A0NA2W0nkTbPewKASJJnDK6r2H8qeBxX0FaeqN6-BsisZh0io9brw4TEFZFDdgJJoS-efcCAnkJuljB9XMYnpwYOnTytm2ke0A4eDL4HeHTszwowPUxtkIc1-qxvMAVG9897XXlQifcm_weFeDJIDWP_-0zJ2-5IuyR5GjMTK8r0agdnea_2yatJ0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5KeEsS-_QvMLw8FqYIXEt1rFvE2icFWW6lgPKjUSulBGHmj61FteLkcGXZZbA51xg6ns__rRs0LrKBUd7_foaO6Sw7Hc_drlzVdFHEdRZ1HzJWFwpKgUQcD-qwPgqP9-7yuTkmCSlKgsR0rZFEtsPlcqHFAyBkkoFc04pKVl5JmRgNw1FcsWJAUXDPPOn4_f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/y9RYTUb-sgBZYJ9sxPJ-NA_zTVwh7jvb0FSuZOX9IWH8UT1_DYJjyMtiZeIAeMTbxQhqUW0F31R6YQHYpQoXT6UZZ3ZXOfRUQKPJWbvbO-xYelyA1YnNB9dfRlQfvSxVfoWRUOqzUJF6OJL8DFldhUXe8d6MAtI6g3WOOs7HpNc9nRESv3tSo9Li4ps_DZz0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CKr4cGKxnltNSAHaPdVzWjfKAw7vByhcDvPxsvfXpidY8xX768tKfVNatqan4UXWCC7LbUSQNQk8IuOT8uwSm4wA0vpvGtxQK5RXgwS7-IfkenARhvNElg62qBtVC_aD9K_K8lIUX_7tdCmDNXOo27bQ5fmDCZ-xnB5qHBFQyMkaQ0Hg_lLjYrqQKVtIf1cI?purpose=fullsize)

你可以這樣想：

當 X 軸移動 10 mm：

* X 軸自己的平台移動 10 mm。
* Y 軸整組跟著移動 10 mm。
* Z 軸整組也跟著移動 10 mm。
* 最上方的負載也跟著移動 10 mm。

所以，X 軸雖然只直接推動自己的平台，但它會間接帶動上面所有機構。

同樣地：

當 Y 軸移動 5 mm：

* Y 軸平台移動 5 mm。
* Z 軸整組跟著移動 5 mm。
* X 軸不動。

當 Z 軸移動 3 mm：

* 只有 Z 軸平台和負載移動 3 mm。
* X、Y 軸不動。

# 二、三軸如何對準三維空間中的一個點？

我們假設設備有一個共同的座標原點：

**O = (0, 0, 0)**

這個原點是控制系統定義的參考位置。

例如，現在希望最上方的相機移動到：

**目標位置 P = (20, 15, 8) mm**

代表：

| 軸 |  目標位置 | 意義              |
| - | ----: | --------------- |
| X | 20 mm | 向 X 正方向移動 20 mm |
| Y | 15 mm | 向 Y 正方向移動 15 mm |
| Z |  8 mm | 向 Z 正方向移動 8 mm  |

控制器會讓三個軸分別到達：

* X = 20 mm
* Y = 15 mm
* Z = 8 mm

最後，相機就會位於共同座標系中的目標位置。

![Image](https://images.openai.com/static-rsc-4/33nlnO03qODD5oVex5VLprNBRhDk0oodF72WvpOSeqckRf81Cag9XxynPBypc_sf6F3yT-1Js_45Dy-0NSHpc6ZQbfxN_1lsVus82BL9CQSzxLCF4CgLiO9ml0o13KH4wm-9xkvTVJXzTv5C0uC0YNs1lfj46MGOaX7FtcIDkA-KL5LPacCicaN4Tuxt47jm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UjB3E09U69HENCBePdyD46l1xYxIhsRdsLH05lT8Vq1bi_9cazDu0IbxaeAf5nUOsT-wr1TMoVDJki-ZMbfark3fZZuxYyW_I6oZUYsmLwJqpDK8qC6hhdyhK1tll2f2eJYFr1AO3kMVt7u28U1X_tbbzxDDaQT5Un7cdb0htE_1W6qocB61PjzcNRLLdDUr?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RM0OAmhjxbQU43xLsFql1tIftgs4fvR5NdWRxyfMRa2LMGd8KVTzD4MBmroVWsGhWDFHTjxpl9_fw1MhFufU455Os1emEl3CbRcnz1nQKh3J-SELJd3BMlo_mOI-tx30l9UYi3_0moVzY0VwOCuT-x-J-NefhS140jpuf--3XjW_GGJaPxvsThzWnw62lJiW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NVaJDnYmXmlQ8l5Au_7rTSqI4ulWJNuEO39ubXRem_R2A4TkDyJ6eAbTKO607HqF5LWHFjl4rjFqgzHOZe534eXV7VcG2YnrFvHZ2po2ck5tT-8--HkpV1zPtaqIX_UCVLjkf5T88QPnJ36kpBBdZ42Iifsa65OL_VRDI0cd8o9Ia3pIdRVwJgR9YPgQs5DS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9hyt9mnGQ4ZJMMb73i9C9INFUGBVw7RYUepDNJB7qqgVQn3kgoS4QXWnqEZop9mpODIYZQdEL6gGC_QYvc-PgbIooI8OHyCOv9CIGYWF2Jqmgt0bUY7sbLCpwSFZuCmoezd7CzG--PvG7-nDuIuujWsx6hKBhslY9trVFHqZKuSlHh3GFxfa7BOh8ffBUxXz?purpose=fullsize)

注意：這裡的 X、Y、Z 數值，是以共同的機械座標系定義的，不是三個滑台各自隨意選擇的座標。

# 三、為什麼 X 軸移動，Y 和 Z 的座標不需要跟著改變？

這是最重要的地方。

假設目前位置：

```text
X = 0 mm
Y = 0 mm
Z = 0 mm
```

現在只命令 X 軸移動 20 mm：

```text
X = 20 mm
Y = 0 mm
Z = 0 mm
```

雖然 Y、Z 軸整組機構跟著 X 軸移動了 20 mm，但是 Y、Z 軸自己的移動量仍然是 0 mm。

為什麼？

因為控制器定義的座標是相對於整台設備的共同座標系。

換句話說：

* X 軸負責整組機構的 X 方向位置。
* Y 軸負責在 X 軸所帶動的基礎上，再增加 Y 方向的位置。
* Z 軸負責在 X、Y 軸所帶動的基礎上，再增加 Z 方向的位置。

因此，三個軸的位移可以直接組合成最上方負載的三維位置。

這也是為什麼我們稱它為 Cartesian XYZ Stage（直角座標平台）。

# 四、實際上 STM32 如何控制？

現在把這件事連回你的 STM32H755 和 EtherCAT 架構。

假設 STM32 要求相機移動到：

```text
Target Position:
X = 20 mm
Y = 15 mm
Z = 8 mm
```

控制流程可以是：

1. STM32 設定三軸目標位置。
2. STM32 透過 EtherCAT 傳送 X、Y、Z 的目標位置。
3. 三個伺服驅動器分別控制自己的馬達。
4. 馬達帶動各自的螺桿。
5. 三個滑台依照命令移動。
6. 驅動器回報各軸實際位置。
7. STM32 讀取回授，確認三軸是否到達目標。

這裡要特別注意：如果設備要求相機沿著一條斜線移動，而不是三軸先後到位，就需要運動控制器或 STM32 的運動控制程式協調三軸的速度和時間。

---

# 五、確認你是否真正理解

我們做一個小練習。

目前三軸的位置是：

```text
X = 10 mm
Y = 20 mm
Z = 5 mm
```

現在 STM32 只下達一個命令：

```text
X 軸再移動 +5 mm
```

請問移動完成後：

A. X = 15、Y = 25、Z = 10

B. X = 15、Y = 20、Z = 5

C. X = 10、Y = 20、Z = 5

**你認為是哪一個？**

這題如果理解了，我們下一步就可以進一步學習一個很重要的概念：**機械座標系（Machine Coordinate System）與工作座標系（Work Coordinate System）**。這會直接連接到你未來要理解的光纖、Camera、雷射與矽光子精密對位。

---

```
B. X = 15、Y = 20、Z = 5  我有個問題，你說的這句話我不懂：如果設備要求相機沿著一條斜線移動，而不是三軸先後到位，就需要運動控制器或 STM32 的運動控制程式協調三軸的速度和時間。  
```

你的答案 **B 完全正確！**

X 軸再移動 5 mm，只有 X 軸的座標改變，Y、Z 軸維持原來的位置。

接下來解釋你不理解的那句話。這個概念很重要，因為它涉及多軸同步運動，也是 STM32 運動控制程式需要處理的事情。

# 一、先想像一個實際情況

假設相機目前位於：

```text
起點 A：
X = 0 mm
Y = 0 mm
Z = 0 mm
```

我們希望相機移動到：

```text
終點 B：
X = 20 mm
Y = 20 mm
Z = 0 mm
```

也就是：

* X 軸移動 20 mm。
* Y 軸移動 20 mm。
* Z 軸不動。

從三維空間來看，相機應該從 A 點移動到 B 點。

![Image](https://images.openai.com/static-rsc-4/HMWxoNYgBfWfkv4ThoHuniUrvai3SlvtWvqL-kMVHIYiNqW5Id3DNSFmrgzT71Ced6lLIXFL8RjjbCpthPBhu-NN8lyy6FC2YHWXU7FLFJel4H1ZVQxdXSoTJFEUp3iijpwvA3VaeX4HYQKfI9qa4efg8kPRr1WSuD17baPzQwLns0BMXdp1RPzCtmu8B-OV?purpose=fullsize)

這裡有兩種不同的移動方式。

# 二、方式一：X 軸先移動，Y 軸再移動

假設控制器這樣下命令：

**第一步：X 軸移動 20 mm**

```text
X = 20
Y = 0
Z = 0
```

相機從 A 點移動到 C 點。

**第二步：Y 軸移動 20 mm**

```text
X = 20
Y = 20
Z = 0
```

相機從 C 點移動到 B 點。

實際路徑會是：

```text
A (0,0)
    |
    |
    |
    C (20,0) ───────── B (20,20)
```

這樣雖然最後也到達 B 點，但是相機不是沿著斜線移動，而是先沿 X 方向，再沿 Y 方向移動。

這就是我說的「三軸先後到位」。

# 三、方式二：X 軸與 Y 軸同時移動

現在換一種方式。

我們讓 X 軸與 Y 軸同時開始移動：

* X 軸從 0 移動到 20 mm。
* Y 軸從 0 移動到 20 mm。
* 兩個軸同時開始、同時結束。

這樣相機就會沿著 A 到 B 的斜線移動。

例如：

| 經過時間   | X 軸位置 | Y 軸位置 |
| ------ | ----: | ----: |
| 0 秒    |  0 mm |  0 mm |
| 0.25 秒 |  5 mm |  5 mm |
| 0.50 秒 | 10 mm | 10 mm |
| 0.75 秒 | 15 mm | 15 mm |
| 1.00 秒 | 20 mm | 20 mm |

你可以看到：

每當 X 軸移動 5 mm，Y 軸也移動 5 mm。

因此，相機會沿著同一條斜線前進。

**這就是多軸同步運動（Multi-axis Coordinated Motion）。**

# 四、為什麼不能只讓兩個馬達同時轉動？

這是最容易誤解的地方。

假設：

* X 軸需要移動 20 mm。
* Y 軸需要移動 20 mm。

如果兩個馬達同時啟動，但是：

* X 軸速度是 20 mm/s。
* Y 軸速度是 10 mm/s。

那麼 X 軸 1 秒就到達終點，但 Y 軸還需要 2 秒。

結果就是：

1. 一開始兩軸一起移動。
2. X 軸先到達目標。
3. Y 軸繼續移動。
4. 最後才完成整個動作。

相機就不會沿著預定的直線移動。

所以，**多軸同步不只是同時啟動馬達，還必須協調每個軸的速度、加速度與停止時間。**

# 五、STM32 在這裡扮演什麼角色？

連回你的 STM32H755 + EtherCAT 架構。

假設 STM32 每 1 ms 傳送一次 EtherCAT PDO。

如果要讓相機走直線，運動控制程式就必須計算每個時間點的目標位置。

例如：

| 時間      | X 目標位置 | Y 目標位置 |
| ------- | -----: | -----: |
| 0 ms    |   0 mm |   0 mm |
| 250 ms  |   5 mm |   5 mm |
| 500 ms  |  10 mm |  10 mm |
| 750 ms  |  15 mm |  15 mm |
| 1000 ms |  20 mm |  20 mm |

STM32 依照運動規劃結果，週期性傳送兩軸的目標位置。

X、Y 軸的 Servo Drive 接收命令，控制各自的馬達追蹤目標位置。

這裡要區分兩件事：

* **運動軌跡規劃：** 決定每個時間點 X、Y 應該移動到哪裡。
* **伺服控制：** 驅動器控制馬達，使實際位置追蹤目標位置。

實際設備也可能由獨立 Motion Controller 負責第一件事，而 STM32 負責設備流程及通訊。

---

# 六、我想確認你是否理解這個差異

假設相機要從：

```text
A = (0, 0, 0)
```

移動到：

```text
B = (30, 10, 0)
```

如果希望相機沿著 A 到 B 的直線移動，那麼在移動過程中：

**X 軸與 Y 軸應該如何配合？**

A. X 軸先移動 30 mm，完成後 Y 軸再移動 10 mm。

B. X 軸與 Y 軸同時移動，但兩軸速度可以完全不管。

C. X 軸與 Y 軸依照規劃好的比例，同步改變位置，讓相機沿著直線移動。

這題理解之後，我們就能進一步認識運動控制中很重要的概念：**直線插補（Linear Interpolation）**。它正是 CNC、精密定位平台及多軸自動化設備常用的技術。


---

完全正確！你已經掌握多軸運動的核心概念了。

你的理解可以再整理成一句話：

**同樣的起點和終點，可以透過不同的運動方式到達，但移動過程中的路徑可能完全不同。**

而你接著問的問題非常好：

> 既然最後都能到達定位，為什麼還要在意移動軌跡？

答案是：**因為精密設備不只要求最後的位置正確，也可能要求移動過程安全、平順、穩定，而且不能碰撞或破壞正在處理的物件。**

我們用 GMT 精密對位設備的情境來理解。

# 一、同樣的終點，不同的移動路徑

假設相機要從 A 點移動到 B 點：

* A = (0, 0, 0)
* B = (30, 20, 10)

這裡有三種移動方式。

### 方式 A：XYZ 依序移動

1. X 軸先移動 30 mm。
2. Y 軸再移動 20 mm。
3. Z 軸最後移動 10 mm。

![Image](https://images.openai.com/static-rsc-4/pv7Rbls_lHjQIkaYAjadWYr_lel8KRw7acOd9raVBUoY7oSQlRYTygrM9vDD0VsN6pIdTUmiATN9ki2CtfyESD7UWIrJUdOEZHgvnQc2sz4s_LWf2iZh-BFfN98XdcTm_mhzHs0Q28NO9d64es4oSdv9PTbN8Oa8IVNo1DmVX4YqB6R22SKIuBZ9LaIR7IDL?purpose=fullsize)

這種方式比較容易理解，也比較容易控制。

但是移動距離比較長，完成時間可能比較久。

### 方式 B：XY 同時移動，Z 最後移動

1. X、Y 同時移動到指定位置。
2. Z 軸再移動到指定高度。

![Image](https://images.openai.com/static-rsc-4/CIv0nRdwGBmNvfG5FOHRx10wCkLcEvoO8wwyb8ZuZL8Ni0sKB1M4aYO7nCIH3tBxbefb3-7H1pvY0N8vzMbNCvai2RIqFAKeoWhPwx6DMibEXnDBPLe3Qbtkej_3fDTier4k6jIe62hL8zX7bgQPQv9o05W12MJ2vGXcQY_j9SYM_k8oNZh33f8n_HIXwYhx?purpose=fullsize)

這種方式可以減少部分移動時間。

例如相機先在水平面移動到目標上方，再下降到指定高度。

### 方式 C：XYZ 三軸同步移動

1. X 軸移動 30 mm。
2. Y 軸同時移動 20 mm。
3. Z 軸也同時移動 10 mm。

三個軸依照規劃好的比例一起改變位置，讓相機沿著三維空間中的直線移動。

![Image](https://images.openai.com/static-rsc-4/HMWxoNYgBfWfkv4ThoHuniUrvai3SlvtWvqL-kMVHIYiNqW5Id3DNSFmrgzT71Ced6lLIXFL8RjjbCpthPBhu-NN8lyy6FC2YHWXU7FLFJel4H1ZVQxdXSoTJFEUp3iijpwvA3VaeX4HYQKfI9qa4efg8kPRr1WSuD17baPzQwLns0BMXdp1RPzCtmu8B-OV?purpose=fullsize)

這就是三軸直線插補（3D Linear Interpolation）。

---

# 二、為什麼要在意移動軌跡？五個重要原因

## 1. 避免機械碰撞

假設相機要移動到一個光學元件旁邊。

中間有一個固定支架。

如果直接走直線，可能撞到支架。

但是如果先往上移動，再水平移動，最後下降，就可以避開障礙物。

![Image](https://images.openai.com/static-rsc-4/b4aprEZwajstCvki9-HyjAs8VLOt1jtYaGGInPD1ztePUHArktI1yiiEO4jbi0UftvKRh5L9yi5_j-gAFuDI793SxGHcWY7NqtsB4EXBds7pJJhzJl7FcEcAQ4Spaezh1UaJhz2HR5kP1ajGnlVLndcmmhR41iL6ArUUdPZpYkBz0p5SjObruSfTG-bybG3Z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vOycSglF50PF8sbJx3hp36oUA4d_idKeJtjkeFmrymWB-FZfZ-UQmhNWmjUgeK184crE__GLeudw7tEAR7jDvo_0hINXe0rAtSNADstxRcpYwg2wV-WYGFqmlKJBClmwLaEygWZU-SuuhC-ag2kMz3DwB3xJ3lyPu2jaL02-ZYyu7nZG_-HsLxVf-xZfanCj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FPNvhegscnK79d37Aq5X-IHvJIELAetQ8YzuPpljZ-Oqaxjd2gKyEYsZlM4TfzLAk0aiRDowapTDJ-HZ2E36rU78SSpsr28U8OkE788C7iET7jluHwDxeMoqPyREL3VlH9rTDmi1FtO1i1r3sdtIeFxhJ9DcC17c7GMJYdyjqcD6y3citaaO-iH2D83x3hJc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/n-7F_yhEaidUwUUGTjbiCkya41cM_IAZPFiPJvym-_2RqcNNbt8EZ7-_D4A1jUOXDBVE-H0-ePXDcGNem9_AF4UD5a4PjZdaLeUv5b-j5nFfMzChZFpzS2XxgJD7uZSRsBiwzBi3cFnV2yae_4wBjSFrz5WyO15z9gbMKoTbB1-bUGG3mLP5vAx7YJqWXvOP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sHz0781Z-_OKdcHa4qXESU2gt4IXtur1GAeyGGhNC6wIsx91j42-1t6Pd9cH7nKBf1DSo5UiMU0Ct0wI1cTHFCAc-w3x2np-NYLWE_906fkNHQX054NhmnzQlhT3JEkAkNlnui5_oJ9PeMFQ8683tj24EP11CxEV5dIUTmtEkwVOWmqZKQV6dV3fkfVwJf56?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FxewPRPHx1duZWFsKFgD6PJ3kw26G4WrRSnXyX2SqOiu67O-wTTkdT7Lj5dzZr-BzbjbosVxEfjBiKvb4tfMOWrpV3dyjbs1Kabr6ZOIHTBvPCmqawZMZQ8C7kzO3ndeUEhQ3ub6fUGYQuyfZ2IaAUXTBM7Kh7N3Hh71dA6N5zQ8oFRISmNRKHL7Y659kE_h?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LAzneMVtvlPb6tPU7tSCt5D0JThB6VlehyYHh3i-8IRzhWMih4HvppKm0mMFluCxQ64njVHEUyZuBeM9YteZbHEV6nXgY4NowxUGbHYLW4xNae9QvCjpCM8-AgRpwj4OTydHG6MNhfLxkl53c5VO9pSQ7zJuMrtTGitQgRfJRAqHGqZ23eK7cHsv6rK68Exe?purpose=fullsize)

所以有時候：

**直線路徑不一定是最安全的路徑。**

這也是自動化設備需要考慮移動路徑的原因。

## 2. 避免損壞精密光學元件

這個例子和 GMT 的應用非常接近。

假設相機正在接近一個光纖接頭。

如果 Z 軸下降時，X、Y 軸還在快速移動，可能造成：

* 光纖接頭偏移。
* 接觸位置不正確。
* 光纖受到側向力量。
* 精密元件受到撞擊。

因此，設備可能採用：

1. 先完成 X、Y 定位。
2. 確認位置正確。
3. 再讓 Z 軸緩慢接近。

這種方式不一定最快，但可能更安全。

## 3. 控制移動過程中的速度與加速度

精密設備不希望馬達突然啟動或突然停止。

因為加速度太大，可能造成：

* 機械震動。
* 平台晃動。
* 光學元件偏移。
* 到位後需要更長時間等待穩定。

因此，控制器除了規劃位置，也要規劃速度與加速度。

這就是我們前面學過的梯形速度曲線，以及後續會學到的 S 曲線（S-curve）。

## 4. 確保多軸之間的相對位置

例如相機需要沿著斜線掃描一個光學元件。

如果 X 軸移動速度和 Y 軸移動速度沒有依照比例配合，相機就會偏離預定路徑。

即使最後到達終點，中間的掃描資料也可能不正確。

所以有些設備不只在意終點，更在意每個時間點的位置。

## 5. 提升設備效率

如果每次都讓 X、Y、Z 依序移動，可能需要等待每個軸完成後才能執行下一個動作。

如果允許安全地同步運動，就可以縮短整體動作時間。

但要注意：

**同步運動不代表一定比較快。**

還是要考慮各軸的速度、加速度、負載、行程和機械限制。

---

# 三、連回你熟悉的 STM32 + EtherCAT

這部分是你未來看韌體時很重要的觀念。

假設 STM32 要讓 XYZ 三軸從：

```text
起點：
X = 0
Y = 0
Z = 0
```

移動到：

```text
終點：
X = 30
Y = 20
Z = 10
```

STM32 可能有兩種控制策略。

| 控制策略   | STM32 的工作                     |
| ------ | ----------------------------- |
| 各軸依序定位 | 分別命令 X、Y、Z 到目標位置              |
| 三軸同步插補 | 計算每個時間點 X、Y、Z 的目標位置，週期性傳送給驅動器 |

第二種方式需要運動軌跡規劃。

例如你的 STM32H755 每 1 ms 透過 EtherCAT 傳送 PDO，就可能每 1 ms 更新三軸的目標位置。

不過，這裡要記住：

* STM32 可以負責計算三軸的運動軌跡。
* Motion Controller 也可以負責計算。
* Servo Drive 負責追蹤收到的目標位置，執行馬達控制。

實際分工要看設備架構，不能只因為使用 EtherCAT 就認定 STM32 已經具備完整的多軸插補功能。

---

# 四、這裡還有一個重要觀念：不是所有設備都需要直線插補

例如：

| 設備工作      | 可能採用的運動方式        |
| --------- | ---------------- |
| 相機移動到拍照位置 | XYZ 依序或同步移動      |
| 光纖接頭精密接近  | 先 XY 定位，再低速 Z 接近 |
| 雷射掃描      | 依照掃描路徑進行多軸同步     |
| 工件搬運      | 先升高避障，再水平移動，最後下降 |
| 多軸精密對位    | XYZ 與旋轉軸協調運動     |

所以，運動軌跡不是越複雜越好，而是要符合設備的工作目的。

---

# 五、今天的小結

你現在已經理解三個重要概念：

1. **單軸定位：** 一個軸移動到指定位置。
2. **多軸同步：** 多個軸依照規劃同時運動。
3. **直線插補：** 多個軸按照指定比例改變位置，使末端沿著直線移動。

我想再帶你往 GMT 的實際應用靠近一步：

假設我們要把光纖接頭對準另一個光學元件，接近時必須避免碰撞，而且最後要非常精準。

你認為比較合理的流程是哪一個？

A. XYZ 三軸全速同時移動，一到目標位置就停止。

B. 先移動 XYZ 到接近目標的位置，再降低速度，進行最後的精密對位。

C. 只移動 X 軸，因為 XY 和 Z 軸會互相影響。

這個問題會引導我們進入下一個重要主題：**粗定位（Coarse Positioning）與精密定位（Fine Positioning）**。這是光纖耦合、矽光子對位及 Camera Alignment 非常重要的概念。


---

正確！答案是 **B**。這也是精密定位設備非常重要的控制策略。

你已經開始理解為什麼精密設備不能只追求「快速到達目標」，還必須控制接近目標時的運動方式。

今天我們就進一步認識這兩個概念。

# 一、粗定位（Coarse Positioning）與精密定位（Fine Positioning）

想像相機要對準一個光纖接頭。

假設目標位置是：

```text
X = 20.000 mm
Y = 15.000 mm
Z = 8.000 mm
```

但是目前相機位於：

```text
X = 0 mm
Y = 0 mm
Z = 0 mm
```

如果一開始就使用非常慢的速度移動，雖然精準，但會花很多時間。

因此可以分成兩個階段。

## 第一階段：粗定位

先讓 XYZ 三軸快速移動到接近目標的位置。

例如：

```text
目標位置：
X = 20.000 mm
Y = 15.000 mm
Z = 8.000 mm

粗定位完成：
X = 19.000 mm
Y = 14.000 mm
Z = 7.000 mm
```

此時距離目標還有一段距離，但已經接近了。

![Image](https://images.openai.com/static-rsc-4/6XeqPAptfofO6qFWh9nVUS8yeB6ZejBRhYEzMiDSqmZyMmmjuJO0QGFCaPuCQZOCPUkPoanmZaowNcTJ2fjXiG2hqoQJ1EJO8ZBBRNWggkyWiDYyZbEWXvAH1BQAhHI0nKnZha7x_8x9DFNmAqWN5EByicdW0yQiHEyB2R0fyjuojpkxourm6HfT68SweICK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/hDMtlrNkmXquFM4XW6OD2xtUjEO0zJHiKs-2n_j0QAOx1rXPdPynICoWy9Ch6GBmeDUcNwnBU97QmmL9EtJaaEGvQ1iwuBkPWL_rgsRtlbHDkgTM1CiwTEL0wzdwykXSw2nCtsd8Tp5LdwIIb7IqgQYhv-vBCCl_RD4_gkfl-9zTKKsW0OkcjWhhMvQePKGo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zpZr5NfOd7TBM_w3H1T2Qs7X60fkigf05UZ5ZRiTAZ9W7qt-NhH9jLVTcJVZo3CCmOXDyTCyCF5McHo07WRJ1kJUxx6-z0EoUFKvFgLFkTbrO0X0qzK17746HAx1Ycldkuyr_BNtraiW4Zd97zTlNjtlXI9nvobMDM4xnMNQ3F6gHGHsT-qqU69V7-v1K5k9?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Cno8f3-9ykKUqE5OsOO0OTCSPBny3QTwYp0eAZLoU7LEWlISYsxkAFyRoB5evJgb50pJZ3k_IqlU94mMpxQ846_Y8pBsBOBXYkooM-n4iJQLI9_2MTjrrFaokorceKIxyHhwyowSs5nFm8hFS6kMTR2EqKrxI9adCHYSMZinr0T3iyUrQwMYop6UtvRG4ViQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xO0qITPi7sbsROANTcnHO1ciG2qENRvTzr72tdE0J5tC0Dn2ezoMEGmgJVPzJV_QKBM3O_-0vTNJhk4zJDLEH3IcrE_V9BH1oETfZK2tfmexJ0E6EfAaNhG9lh50MPkNPZ4T_HE30qfynj6ZOQsA6efcHYcx99t0VLF_OMOkhnn6vANQ1TbohKEtPxLwNj-z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9QGpPs63l7Uwmjl9KP7vpb9LWYrB6dxCZY7bq-vb_4yYZmqcoKc50E0J_62gbe9QmDNMtnOGaYR3G7yFmcER8l_o5JJQ1G0QsG6776I1l2-trvQQNygxizrV0dS7CFwtaZ_g52CxweymWH6mhK8_FsAcOrti_7On9XrjAyfamjl3VvcgDWyni-Y02lshXov3?purpose=fullsize)

## 第二階段：精密定位

接近目標後，降低移動速度，進行更細微的調整。

例如：

```text
最終目標：
X = 20.000 mm
Y = 15.000 mm
Z = 8.000 mm
```

此時可能需要非常小的位移，甚至以微米（µm）為單位調整。

這樣可以降低接近時的衝擊，也能讓設備更容易找到最佳對位位置。

---

# 二、為什麼光纖耦合特別需要精密定位？

這裡舉一個與你未來學習矽光子有關的例子。

![Image](https://images.openai.com/static-rsc-4/Gb-zmcaJ9W6uDCD13HJPmZP1UAYMFymjIOromN2GpGMHxysbFPWTDHHQCyHsIrOclX_hXp9ZpeHots79lnjXewA7NAynJ9so2fi0mWHlp64_6DctGx3aqzbeM2TA238pY9ngz_kzzAySK9QXj29N4g57UkoebeCwW474iFB86FNW17FVhDamd9Bq27hi4WcV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/913_LGROsgwWk0C1H_PnLQv6m_qvxM296uMpL_NlX4cIp_c0oPUeQoJVgwHxVC3lybcsuJFwU2xhE5mWgY0BA9MfwTjSfFYmbSN3MIkJTzcStYzM_0xHBwXWP9c4I5HJD0_eXXoFcpQ6nH3Y8c37asoYEe-CtuxlCi4nZZRA-Q91ZbhvBPICchg0_AMY9x99?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nSpg1lW1BThy47sBsvsbfIUU0VX6aYrMobdHajX4N5GN1UsYUbzmPvehJpymyLRxqu6OdfRwlANpfKWE5HngIV61BEiu9jcUvyPnoy7uR7Qy8nqOTVa3VsdjNK1h6Szn7TyHX2W9yx152CUOtFAVOwbnutq82FSHf3o1zhWiXcV9OAOioqQU37mZQrFE-bZD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZgGFvU4g8tWaMaERt6-1Adumd03noSQOZUBENN60fjuhiVMs7h0RldTqTZak9ogZxhfsSEhz58_eNkQfMIiDvezChD0KmMe0M8TMOZA6JHTPTi1_IVqovEWn9Tx9tHR1HNWGxnPJWalE-OPZFtqc3caR5uU2DXYVsUscnEwj67gCt7zIDRJSuHH3JlINzYo3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Dm2UajGG_dEypiAtFuBTeEohJPVypYVewTuXqa9EFmGwPDg0BeRev7WOBlJrZZmCivGxTkRzg4LTRchhUJinqxHn8A8-JckiYOzJipGxmqxlYhHYNpYqgkSv7JVcSzGglkVcpKBHxvSKl3mlm9mOSnFqfetQTY_dLYe8x7tDOR_RzZ1ZfEeaSQuWy5B-gBfs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1gCY56SHk-6_pBzMF-WYXlHx5O6BDxmTlMLLIt6iDS0LGePeVQfbgKhmt5UtRPypvJ8te3Kpg-neoNMuw3rThDpcrGn9UkWHJTCXmm6-ac3M82oBz8QohxGTbmWZP389Aq6oZ-t2oeqgmlJa4JZ9LbSAGT4UirN-MIjcqTeUfUJUu3acJIeZovZHIAVBGYS5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8QtjfqvEoYcwKaMMP3EScJ95meD5PFeF2ug2cYlkPu-ik21zM7ZNYD2Z52O_Im3SU-iNzE7PPdPJFSnEZ2LgwDXE08UlrFd1y2phmLHy8UsmuOxGUXLxqUVJmkwbvd52pA4Y_ng_S6FezP2_9J6d7B0689-dbPWpKv1-V_BB6v5T2eQTnYuWrtoHiXSuD3TL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/E8r1HdLe5GgXXPltO931wxShHWkagVSmdAWDENonZ0V5KPculCil8z3OaVVdxQR4E-zqc_dfi3o6Ae0dooWhr026SABwGGakOsv8u3xPaKVPt6DqH2KAeRWi5oZD1nl7_dtFFv8Y4dwYDNUIBOCBDpu1T_cmFGsJmf_dxu3ctvML1ivnAECQ3omMcRGGRerl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CDpLYp7t2wHPGmzFbxbK1EfMbm57FDDBLz6loBocHXxqAqxHjrExqjj9J3cY9tPXLuE74ecSGhkYq9j6sSE5ScJ9C3lszPAeIclZwRXd_fk6B3v-Ur6vJyWxkAZS67nZp-T7AmKVrOwdBCMn_oUcmFHrCdIvmrsrGoxYrSJ8FChDHDn1ZonCW2S71AQKvn6z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GdenXCcO2MObiXURQINwSeaclSQrS3cplg5X12jPA4Dhrd0jULf_AfAOnjWdZE7j7KKq6eejWhazcHbJ-sV397fT9j7vEYDqI8_Z1lMtSL1QNqGyeus3SwEg9LnMxNG8CIriMnF2J8qiqk1Uu-DjOQGsoz_MI9gseZbHqvLdkdTPrHFgMhzCCExEtv4HKIfa?purpose=fullsize)

假設一條光纖要把光傳進矽光子晶片上的光波導。

光纖的出光位置必須與晶片上的光學入口對準。

如果位置偏差太大，可能造成：

* 光纖與光波導中心不一致。
* 光線無法有效進入波導。
* 光耦合效率下降。
* 光功率損失增加。

因此，設備可能先用 XYZ 平台將光纖移動到接近的位置，再透過精密調整找到最佳耦合位置。

這裡要特別注意：**最佳光耦合位置不一定只靠幾何中心決定。** 實際設備通常還需要量測光功率，透過回授尋找最佳位置。

---

# 三、從韌體角度理解：位置控制與量測回授

這個流程就能連接到你的 STM32H755 專案。

假設設備具有：

* STM32H755：設備控制。
* EtherCAT：通訊。
* Servo Drive：馬達控制。
* XYZ 滑台：機械定位。
* 光功率計：量測光耦合效果。

可能的流程如下：

1. STM32 命令 XYZ 移動到粗定位位置。
2. 驅動器控制馬達完成移動。
3. STM32 確認各軸到位。
4. STM32 開始精密搜尋。
5. 每次微調 XYZ 位置。
6. 讀取光功率計回報的光功率。
7. 比較不同位置的光功率。
8. 找到光耦合效果最佳的位置。

這裡會出現一個新的概念：

**閉迴路不一定只有馬達編碼器的位置回授，也可以有設備工作結果的量測回授。**

例如：

* Encoder 回授：馬達或平台實際移動多少。
* 光功率回授：目前光耦合效果如何。

兩者用途不同，但都能幫助設備完成精密工作。

---

# 四、今天先記住這三件事

| 概念   | 目的                 |
| ---- | ------------------ |
| 粗定位  | 快速接近目標             |
| 精密定位 | 小幅度調整到理想位置         |
| 量測回授 | 確認實際工作效果，例如光功率是否最佳 |

這三件事結合起來，就是許多精密光學對位設備的基本工作方式。

**下一步，我建議開始學習六軸平台中的 RX、RY、RZ。** 因為光纖對準矽光子晶片時，有時候即使 XYZ 位置正確，只要角度稍微偏差，光耦合效果仍然可能很差。這正是旋轉軸存在的原因。


---

很好！我們正式進入六軸平台的第二階段。

這一階段會學習 **RX、RY、RZ 三個旋轉自由度**。它們對你未來理解 GMT 的光纖耦合、矽光子對位、Camera Alignment 特別重要。

先建立一個觀念：

**XYZ 負責把物體移動到正確的位置；RX、RY、RZ 負責把物體調整到正確的角度。**

即使 XYZ 三個位置完全正確，如果角度不對，光纖也可能無法與晶片上的光學入口正確對準。

# 第一課：認識 RX、RY、RZ

## 一、先看三個旋轉軸的空間關係

![Image](https://images.openai.com/static-rsc-4/TfWq7L8Hfo1NtxDDTPd6kL3ypwFgG1JzdghNRMo0S96JVYovfM9yXoQEHVIGynCRcLVaaehCj8JKcntmBkp35xvl45HSLIB5o9Kuxsj44FQIj6j8hk8DZzRs_XRlcxM5MwMXQvA_J07I3-OF6_QOcnJYu5HXqW8S3MmTagj4QSi2nzGSByNRloibYK-nMuW4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Z3uLskDzlFWVs3MxLa0T0hNBNm0R9_quPx1PUlSY3yxLzotDCwuHHyyxXzuSN5pFDRoblAzotME66Fuv5casH3xzcsQk02RDzWs1GduhuRv3Fa1zMmizLJj8Q5E4LySn_zotUeg5c5sVCl2psaClPKgFJSD3-oVjOOUrBlkA0nuOofTyxQny_qdaiCA0MC8M?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ELuIPPeOMfnGKGywSnLQgEXs2CQZ-LkOQB8ky3YrO7VRvqWtRnfnPc7UtdX36Hhvgkga_k1Z-_u-sXUBtSZ7S210M2aL8ngKxk4VX0mPas_utnjD3bpaRSjw3OQ0BL2UnlvS6_18I1ncRL45KUHWpgiAgE2dkYuzzoBLVDOYnLuf6_9Y1FETSCxQpomzXCLb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZhAcOmRW5jXBeW1lq2BmkN4KhM6737z1Ds858j7ICjCIzrafvN8mVM-qHl-898qHinm9wDEexBj_nW-0AWNiB8OzwcasOR2c4_nd4QQK8BqxDOnyeOaIzqYcBwbzzA_mQdxA4OYl6NTCWVby1Avo5-H0cMovc3xFBUvz1nhuUHzzx2wDFEd8o2DW3bCKonP4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/FrQMzOFRtq3Wrz0Hp_Hb6RTuDth3xRVzpxFDuXvFlb8ACFnoYmDt-OIe5aLV1w4uOYENMPi1EuFftbiSbRLDodUNPLRvy5dPf1TZ4t0BQQgwS82H1X8JSQpNUHBxSmZ2m08hQ_nM5pzO4OOLV9gjxVZvjsFLlONwuGEEsIPqVzeMZQKF3ELG5Oj7Xl7MO1Zp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zuF7CZ5L_fkLvUJHnIEhOax67a3BLcrYxn21I7TjYg5sfRfvya1S7O4R3gNciDmDwtICk0h5qBk1VMXPJhwOZJRIMAa2EG5mINV46KU8IfTKfVcNwxFlFCYxQjYQYZVjTZVREOYUwc_sswhl3eFxyLPkBYSM7CCjDODln1FZMTyj5xPUykrvAf7CQVmIFsf0?purpose=fullsize)

先看這張概念圖。

我們有三個直線方向：

* X：左右移動
* Y：前後移動
* Z：上下移動

現在假設有一個物體放在 XYZ 平台上。

我們不移動它的位置，而是讓它繞著某個軸旋轉。

這就是 RX、RY、RZ。

| 名稱 | 意義         | 旋轉方式    |
| -- | ---------- | ------- |
| RX | Rotation X | 繞 X 軸旋轉 |
| RY | Rotation Y | 繞 Y 軸旋轉 |
| RZ | Rotation Z | 繞 Z 軸旋轉 |

注意：這裡的 X、Y、Z 是旋轉所依據的軸，不是物體移動的方向。

---

# 二、RX：繞 X 軸旋轉

假設我們有一個相機，放在平台上。

現在讓相機繞著 X 軸旋轉。

![Image](https://images.openai.com/static-rsc-4/eIQgGWjinSmR0P7knr1TA0lnZ1PCV16GFjRqGYitANGSRcijFdwhTAeWETL4bjB37gdpOe7b-6nAEQlioS3Ac5CHEPL9EnF0yrRTX41cO42NKGnfW7lnLvvT5Xk1dRtQcMb9JU3HVUb49jqfYKtxhkdRYjkzXU_V0m2deJvJUd7MrYRm5liM0GA2Wnz7K7mE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/S73E1fBlu_XoBTXW_-lJgf3L00ktxoHRTFe2shw_4zNS3LcmkcBRv-cFheqSRF93CqNVzeOnoMwZyG9nYmjUDahSp0qlOUr8WymqxDfIfJWhbQwmhZ0BKWCUqcCHbgU4Aml6qfjRggyVCGmOPgYGlOUROHSvvYRTNEhZHIamZl-TixpDia3A8PIfp7kZ8ZmH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zS335BbMzdFVeHx3hrHnlkYCcFKJwaivH0ykcKjWOytzv5v77SJE5V6j8cPO6lyaE8LERORL5poHfNl-ewYP-NmZQKy2nH7Uf5qG01byKs13lFTis3ZS-WafllxAXtK2ySqlRvpcauOz-AH9FltTjdZ7Hu1iWebYJlRacxSKhibT4Gtw4_0D_hKNKr-EV3Wc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/zi_tI00q_q4pToBkgy9yaJDPdRnwC6nHfQ_dXjshnVVF7wHSLM2_lSsBP4Xb-qOwmT3yqgGm0ifEAwl_spywZ_B2xeAL-no5yRMB0i-2TVjdwMa9kZ5OTqy9hpDmnEmMe2hx9kGl74nrnpgkdk-Cdb5YHfbtLGSXbo5vp9xUcPEJ_bYBeI8UBTWJ6qz2dpCU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wi-YDRg-9odxeJWybBz7Iv48kGw2OwepY2I6YMnC7wcXjWR65p5aRF9N7iaHPz_beCIaBp6YPR8UyxuxvJv2Y15JfstBHLPuTrwmw4osjLDIq25FNzoVUdn4LGENhQ3eIhh1M_Tz8rMp0o9Qau_i-p-iEg_pOZIv1tj9dEB__l_Tkxk-wW2JnunAP70k6mfg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7_Og9s7uwoOroFKbZh4f_lMpcnxfzh96QOlqxHC1rgaBjpRtaq7epSybLYHXBDwS1n-PUyQdyH-ibEomB9ul8znw64MtxfUDbDu60CNq7rOjTz5Lr67wE-DMU9eFiS_VQDT0saeNR1JNmje7oCmEBxRWXn_pYixJAv0L3apjRu6xV8CQp37tp8kQzEuJpfdD?purpose=fullsize)

你可以把它想像成：

相機沿著 X 軸方向當作旋轉軸，讓相機產生翻滾或側傾的動作。

例如：

* RX = 0°：原始角度。
* RX = +1°：朝某個方向旋轉 1°。
* RX = -1°：朝相反方向旋轉 1°。

這裡的正負方向依照設備定義的座標系及右手定則決定。

**RX 的作用：修正物體繞 X 方向的角度偏差。**

# 三、RY：繞 Y 軸旋轉

接著讓相機繞 Y 軸旋轉。

![Image](https://images.openai.com/static-rsc-4/h0EePLdv0ol2bWNkscBvGAStTwmLmd69meGpBdKUiprYzTQNdQulvPRyyTPM5ClVdd4RWIKCku5rOiNso1s62hcF1VN1dk3YMmb_f7IQJnJGVgFo9lW2ZtB82F2C4vXFQ9aC2Z2FCfFRTg8IUjjUSof4-LT5C9Ed9SSQoLdmYnsdxl9ym2RF_22Z9bqhrXWE?purpose=fullsize)

這次旋轉軸換成 Y。

你可以想像相機的前端往上或往下傾斜。

例如：

* RY = 0°：原始角度。
* RY = +1°：朝某個方向傾斜 1°。
* RY = -1°：朝相反方向傾斜 1°。

**RY 的作用：修正物體繞 Y 方向的角度偏差。**

# 四、RZ：繞 Z 軸旋轉

最後是 RZ。

![Image](https://images.openai.com/static-rsc-4/1EqF4meH3arDNh8unkcLrnqenCKDeLms5ZPuleedDVEmrcGLcAcdXKqMzAKxOJLwpKIXycVLdU-klkm-08kWoKjK0-r7wM62JdtN_Via6X1Ie-sb7tDqhTjAZeGQxJaon4Ydr90qKxIM8KYFO5aPyvQD2soQDTp8x9HQV7wMqI0B1_Pyd21f22RCIH7KaIxX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SlwoMtIkIDTIROFTAVGGULLPslYkwRVjAVONDsMpbzv0K08fkYlog6BggVsTxrJv8ArQC8Wu0IVteF0FjA54N735hWODjNOdNiG7ttOzFsDCbw2lRoyQo7fI7At5RIG_wX3HEI9IitockI1lNvln7vArlF90AoAKl50oB4lwMrwk8LQYvY7D_HIVcSsetyqL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uRH11xnG0JbuGwg0Pkpx_MfYEDxRnsznp-lMGHgm7wCmAqhHYB1CnCDYf_pTiEcebn-mlKZIFi2kgUtvImBAT0kRl2VKzfEVXnRm3xW0uXGAHpiFKFnhR1wq6biQoW8VJPkil3FnMkSh4mqdpQC4ZO6SjhQ62p9muApB1nV8iiSmJsw54MdVKlh9t7hFYNUX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_DnUVBIffHTmD6972npfYzCV7atAihj5WcOZBvMFyLDsBHBkn1sDZJoBBM4-PynrkLJWFQHYnSXN-yNmZLQ8CrnzJQzqWX5B7lQSsS63yR580SeJKjX4Nf27mw3nUXAVmlzmGllSXyaI-DkM_cGteYjrKu0JF7iMkNx5s-qtfxjqVhu9sFaTTwWV4HSHzA7i?purpose=fullsize)

這個比較容易理解。

假設平台是水平的，Z 軸垂直向上。

讓物體繞 Z 軸旋轉，就像轉動桌面上的一個圓盤。

例如：

* RZ = 0°：原始方向。
* RZ = +5°：從上方看，依照約定的正方向旋轉 5°。
* RZ = -5°：反方向旋轉 5°。

**RZ 的作用：修正物體在水平面內的旋轉角度。**

這種旋轉在光纖接頭、相機模組和晶片方向對準時很常見。

---

# 五、三個旋轉軸放在一起比較

| 自由度 | 旋轉軸 | 可以想像成      |
| --- | --- | ---------- |
| RX  | X 軸 | 相機側傾、翻滾    |
| RY  | Y 軸 | 相機前後俯仰     |
| RZ  | Z 軸 | 相機在水平面內轉方向 |

這裡有一個容易混淆的地方：

RX、RY、RZ 並不是三個固定的「左、右、上、下」方向。

它們描述的是**繞哪一條軸旋轉**。

而且，當多個旋轉軸連續旋轉時，旋轉順序及座標系定義可能影響最後姿態。因此在精密設備中，控制器必須清楚知道旋轉軸的定義。

---

# 六、GMT 實際產品中有這些旋轉機構嗎？

有，而且 GMT 有不同的機械方式實現旋轉與角度調整。

### 1. Rotary Stage（旋轉平台）

![Image](https://images.openai.com/static-rsc-4/B5vVLZFtXt1vdB_Z7mxKa8fEVXL6EyLtZ2IBlonBfaTqp91KlH2_XNAFmc3b3j19LUAY5os2F10nI7Sd-AwQUy27Ywl7CGOXB7n-4VJRWS2pcKdk0C8YAPK9wN-JwiZlhLN7WijgBnsP7f3TmIha_e_J1uEb_sZFNw-OZKG7Za0od4rTitAsf0w65rurHZTb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gsyJ6yTStDzqSd-hyHEttVcNpc8Sa1cqj8s4YhVFGSmgPiOej9umOzMdfy75m1o7O_h7u7kZBPyiP0kks9gmiRR8KiUVUejAAqYEMGLiipam4I1YZARwd-cP1V9uCwqbJkfOp1HEpGw7ncaYhpcPIbr-KJjQnr7kSriW3oAVK-neXrN6ecQonAXkvY2T8T2n?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3qwY6T3kehh7pSCNCTBNCBiY1d7wEFzPhkZFa2Jq_RUxWdmcSGXLOokJAFmPmJodmDMDmKYxbiaU1pRkWSHE7FtZ8SqjSJ0K8fAGbnsXKt0eScvhb5NkhEbPKAWF3mkct3B5YGuFFwQDMVsjFTIsVI_k7y5ESsldA_wPZTITTxP9Np6WQvvqdA0icDdmUFvA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TlNK9ugcBJbVHdd0Ar3kKhj9N6oJif5IZqGvbcs8Pkato5Xozi1V1nBjDYNf05yHAS0WYadV7voMgQPoBMRGaJpralWdBgatWGucHAcfGH-0ZuJXGkGix6-0D8K6jAVDF91z8b2uGGsrQHQh0OfcCFWkXGtByBLoGGuJBYAFqnbq0T6RNc9tV4cBMMrIvS2G?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vVqfe7iMNzuWQs4H-wpXDabg8otw9X0isfmMulS9zWsi2wwcJhyEY4Ht7YGM3x5siVQTZd58HL42kp_6IzJRk8oeyqBLiO_ywR7Gm7KzXs3rmkU7Nl_lqlG-qCPE3mC4Yw6EX3dEjgu5o6aP8FMXiZd9DsbScndoPwpT-Ky7gQWs8zv4ByCXkaIYpM65fEHh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u6B43E0mgdQ5IyzOzdjmYYfBK7_GbjeYg4UuvczEwgyCOH9mR5X89kbudKHWFBL-tsihGY1ah1SaYvNiV3P004nqQsfURTrki7JOS363ltiTkGPfilpP5B6NH5qq2yKVst9983FUWYYWnlOLb9qWL8XQlDmvnHlXqDYr52cxEyXchBdaG7e-zC2pKTaxPo5R?purpose=fullsize)

GMT 的 Rotary Stage 主要用於 θ 方向的旋轉。

例如官方的 MR38-AR 精密旋轉平台，使用測微頭調整旋轉角度。

這種平台可以讓上方的物體在平面內旋轉。

官方產品資料：[GMT Rotary Stage](https://www.gmt.tw/en/rotation-stage?utm_source=chatgpt.com)

### 2. Goniometer Stage（測角平台）

![Image](https://images.openai.com/static-rsc-4/KInpkinKscYyEnSHwrJxAUKww_mL-epZP0bfgWTcXgiuE-_uX-3W6eCMsoLUTROWAh0p8BRXHrxyOk9t7mxS9HUu6nBecjYLwe5hUZl6nkamYx3-gPrluNJq2nsI8QkX3CxLn2ZoeIdBFdMz6_9zOtjaeLsYwqARVJMWi6ZY8Ye9qHLBNEYYmrrZ7rFlLyx8?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uyJbMerXbOumFUnb0yi4moF2jJz0bKcZEfjm9Y1GRX2JOrzPKkDNfz5FAYuyJnlYcXsAK7A5iuyHUdctX_4OcI0UR8sL50TnaNsrnGIMJcVyxw-FeP0XQ0gg3DjOujoOnZpuG-Xmf5frE1ZSr5Hl7orumYYQlCGRsmKgSZ9hZbPIiyYn5p9z229EKA-vAApw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KwDcbg60_kB7JmAhp7lcZVVL76K6LrhzPD7CPcZftiyYx_bi0lBOz5M-Pqx1APrby_979jhbi3xTEVwTn934C-mEcKtGDeSS_nZou0YQTADnpfL1OKQECh2F49wEZyYVIe49_vrHXUhSU8q1ir67mn11GQ4qp6ITsyMTzL910omFvTSuyISNvlersVoHEfP7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qaO3KeluG2hibfMF-LRl0itWeSYyXNCshyodLXMq8i96MbA1Sr8I35CX0IGmz6h7EUb6sSAXP5fQxSl96r1viGNl7Z5xwDWlKWmbll3dCN0x3aG5mBVfA2kzqUhG4D1zTyt4C18l6kVz0mbO0DOBEsA1Vpud-9mBoS6nCFqMW-KnkQzoVQgCcXGPuWu9G4X2?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/DitBBHSn0XkQDzPshWapu04pONWA53YAb9h_M_0K9cl0RZQyDRsioaOPNRRv8z20Z5MYWjWl5rQQYmgeOg5BcAIAvqeMUZUrjlQW-S6cs3vy0g7QBPbdmE8C98VThMdMFpo3f7mlDIXW9U_zxbKzLNxnLIhZhITCUwr561bGG5GgCKo_S9gfjG90dcW18H4M?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/42YlHVJMCE-4GScAMD_RdTVpRefOA3PNcRc5ePVsWiwe84IZJanG1imVCGUQUgFkO0gHDo0NkJoq3BGGaSqHSVr5XUVXKgtFJsqXrZEz0EQdSA00DEDtzsb24yuGhigOGt7D8N0Akzt04TQ_YxE5KJ9Av0v9Glws3KnJrtD11YlsT0vXRs39dTN88eX058fW?purpose=fullsize)

這類平台可以讓物體產生角度傾斜。

例如：

* α 軸：一個方向的角度調整。
* β 軸：另一個方向的角度調整。

GMT 官方產品系列就有 α 軸與 αβ 雙軸 Goniometer Stage。

官方資料：[GMT Goniometer Stage](https://www.gmtglobalinc.com/en/manual-stage?utm_source=chatgpt.com)

### 3. Hexapod（六足並聯平台）

![Image](https://images.openai.com/static-rsc-4/pgh0o9BWDoqKOc5TU79kNCXxcUJyUmcl2JQFEOVLwX5NKcF9v9DFDPjMd2HgMNKqpJUPwQw1fxxtzM0eOjx5uBR3LSO92yrKk7S7EWLl0K8wxEBXYkq6HHJk7uULS0gx4GILKyA9GX0Wbxh46M_4RFi-ZPiFs7mzi2-9IRp2Xfrf9iObhwRh20dbHVQpPfNw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-LbpTHVW2TJoi04ARanEcVx1ETiUn8pEMlCn86pg-QDcl_nKB1R5egZNK_ZKVhCl_szqHDZ0Qsyslm83x60L7i69LMtNoJuvApymrh2TXdgzxuGk3qArKPMvrA950wRuPg5t-2w2-rsKEaIh2qranE4zv4Jlbf_2no7YjiJG2cK612HTrkHJ3zql0vPljKHu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iYY6eim7_2g7esRnd9jdYfzDF3EyVmWYTUFcELP7SdZFkEopcHJ9fcSXwRazaGohpk8JFosZFKii6-TnrM_GdRU7EXVHPPXpO-brBOWzpp4xbIKYQB_JiQv6iShswsFdqpKuXnShsjk3onzftE8PW2qTKpi4qOXaCiu7f1y-_hDbRJTqPI8E3wQ7pQDxc5qs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4p9ZqBxXWub5NJ8l-llSBQALCSgSb6aaD-yjYCzWiFJHgrmwcIhebmUqqcwk3svB59ofByfzAoW-zHKeflboxeb1LW6rrv5Ij9pQ5n3pEC-SFO_WDpv33GGWqQCgOmqZ8VbdFBcpW4XCSCiWvnrETqusUyhyc6_EWU48J7Cyg1EskHQRUllyZNfCPPb9TC50?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XPAc45b9Wh2QcygudHaCcGahANqoGoNCY-pyeERar7-crgta-boLKeY0JUwlJRTC1svANhJu56iQjgK3qO4BCZ3BJ1hewWXuNxd1gTdZIx2BXQzAk8-3y8_tBPNKc3Y3V4lPq9o8-iPPetUVmFCZcUR_Ow6QLTWZ0OPk3rirJ7V0NDFRUQcj4gl9rLyrqfDn?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X2wZjiCupSf_HWKNdqygJDMl1DA5BS2zm8QHXyKmIuHOSROczDV-dryIYjt8acOlVmesUc77BFT_F9NcsOVFKVW_DdM8s3VkgjwEbKbMHu49mHDqZc0D9o5-IbugmitIqRweKKqFUATZsc8APtksWtzUUJjhSuRzVjC8AqjOJm4F1Qu0zXcsE9W9rd8GrSqD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ET54TEhNftgu8SVyW9FwXUrxMvsKa9NKbxXpJq0PV3w0h6JiQHh7Ie_gDoRg-_2pWTqONC_TUQ1vFSs1ePuSFplSWqifs52FaqLue4sEsyUCYIg69907An7yDupCVIiC0sUtRGLKuPaNjW3EN-n_AG1NDQDZkHgu0wVH1O1q8hFWUm57HZNkbzAPYR--GfTQ?purpose=fullsize)

這是更進階的六軸機構。

GMT 官方的 Hexapod Platform 是六自由度並聯平台，可以同時實現：

* X、Y、Z 直線移動。
* RX、RY、RZ 姿態旋轉。

它不是把六個單軸滑台簡單疊在一起，而是透過六支可改變長度的支柱，共同控制上方平台的位置和角度。

這類機構應用於光學耦合、矽光子波導對位等工作。

官方型錄：[GMT GSS-100-D Hexapod Platform 型錄](https://www.gmtglobalinc.com/upload-files/catalogue/display-information/EN/GMT-Hexapod_Platform_GSS-100-d_-EN_P.pdf?utm_source=chatgpt.com)

---

# 七、最重要的實際案例：光纖對準矽光子晶片

![Image](https://images.openai.com/static-rsc-4/Gb-zmcaJ9W6uDCD13HJPmZP1UAYMFymjIOromN2GpGMHxysbFPWTDHHQCyHsIrOclX_hXp9ZpeHots79lnjXewA7NAynJ9so2fi0mWHlp64_6DctGx3aqzbeM2TA238pY9ngz_kzzAySK9QXj29N4g57UkoebeCwW474iFB86FNW17FVhDamd9Bq27hi4WcV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bsgyV2xLQ5pBx8GkabVefY6WJZoAsABTLcRAjzbB1sdcwIEymaXVlDfv5-HjrRSgwnt6OVMNt927JOLiOYJ_f6vclvDvKFhJ4O4UN09vwz6EcoaF_vHXZwyK3D5U8bBDyjOEviX4uJvPOAnUbzomFyegHdiTmoIU8DjG-aRaW9QQIQ2Xd1hSfXDxyIsHvHTE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CDpLYp7t2wHPGmzFbxbK1EfMbm57FDDBLz6loBocHXxqAqxHjrExqjj9J3cY9tPXLuE74ecSGhkYq9j6sSE5ScJ9C3lszPAeIclZwRXd_fk6B3v-Ur6vJyWxkAZS67nZp-T7AmKVrOwdBCMn_oUcmFHrCdIvmrsrGoxYrSJ8FChDHDn1ZonCW2S71AQKvn6z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZgGFvU4g8tWaMaERt6-1Adumd03noSQOZUBENN60fjuhiVMs7h0RldTqTZak9ogZxhfsSEhz58_eNkQfMIiDvezChD0KmMe0M8TMOZA6JHTPTi1_IVqovEWn9Tx9tHR1HNWGxnPJWalE-OPZFtqc3caR5uU2DXYVsUscnEwj67gCt7zIDRJSuHH3JlINzYo3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Cno8f3-9ykKUqE5OsOO0OTCSPBny3QTwYp0eAZLoU7LEWlISYsxkAFyRoB5evJgb50pJZ3k_IqlU94mMpxQ846_Y8pBsBOBXYkooM-n4iJQLI9_2MTjrrFaokorceKIxyHhwyowSs5nFm8hFS6kMTR2EqKrxI9adCHYSMZinr0T3iyUrQwMYop6UtvRG4ViQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/78Jp-V4KGjieEQki3UPLeug2EX1w3MSDBW4Oskv4DYKEJM5v3ZMS6zWXgSAYxNM1OltF14G2lZnq3KT7Y5gRIECgWJBZH55rUPMXMgLD39Tarv-vMcKXspH0ISDf2XAWK-4962aCwmGG_TwgCsD_IRbOMMfArUa-JzrOmbNZrQB_vcfduA3J8sv_Yr2P-wvl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/28gOY-grMqZ0SeDNVb_h2dwOZs7LYNE5yRu_CP_ZVtx_zQMvfWj2gpvOMUyBB9CmehO_SfnVbB_hevKjSZBKpUXxje5AkSrMPTWzjsDUQxE98oq77gRDcnuivu_hBsxklwny5lDjMPe7lqV-4tMFGhBk7sArPL32raHG9Pv1pgoOkhR9AIHMcI3cJyc7WSyV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/913_LGROsgwWk0C1H_PnLQv6m_qvxM296uMpL_NlX4cIp_c0oPUeQoJVgwHxVC3lybcsuJFwU2xhE5mWgY0BA9MfwTjSfFYmbSN3MIkJTzcStYzM_0xHBwXWP9c4I5HJD0_eXXoFcpQ6nH3Y8c37asoYEe-CtuxlCi4nZZRA-Q91ZbhvBPICchg0_AMY9x99?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/J2VoQn9shDFEyk7-buCWJyXnIRtAKvoqCaphlEh0Wrz8Dg6CsHVNTvIijVmOusTucaZXbwlXMKEJna3rDTrFDNeWd_Xs0Wa5xT7e4bVH1zqHh6tza0pYH2xT0oGiDZ4MATgNqtYHCbW1hammhytrMtDNK23kHdayHDEpl3X08y0Euhg3JAgjdlk9HYEgybGR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vFIZNyB1HdqZj5DyArbyWaSzy3Nbl-qEpr-H0riFY8FTea-_2Z2y6QUaSpy5olzdwsWTtbaQIoAz5lhxyZJ6NZqTbw3G10KGwJV34718hM2mRfkPHdrPCemf8aUowRrblwhRw3vDCOX-Qu8CDnFtsS3JC38a1kMpGDJVFmJRBZun83w7s9vqBns-nhs5Jq4Z?purpose=fullsize)

假設有一條光纖，要把光送進矽光子晶片。

這時候需要考慮六個方向：

| 軸  | 對位工作         |
| -- | ------------ |
| X  | 調整光纖左右位置     |
| Y  | 調整光纖前後位置     |
| Z  | 調整光纖高度       |
| RX | 修正光纖繞 X 軸的傾斜 |
| RY | 修正光纖繞 Y 軸的傾斜 |
| RZ | 修正光纖在水平面內的角度 |

為什麼角度也重要？

假設光纖的中心已經對準晶片入口，但光纖方向稍微歪斜。

這時候，即使 XYZ 座標正確，光仍可能無法有效進入波導。

因此需要利用旋轉自由度修正角度。

這就是六自由度精密對位的價值。

---

# 八、今天先記住一件事

**XYZ 解決位置問題，RX、RY、RZ 解決姿態問題。**

而且這六個自由度必須一起考慮，才能完整描述一個物體在三維空間中的位置與方向。

下一課我們會進一步研究一個非常重要的問題：

**RX、RY、RZ 到底是怎麼透過機械結構實現的？**

例如，馬達如何讓平台傾斜 0.01°？旋轉中心在哪裡？為什麼有些平台會使用交叉滾柱、弧形導軌或多支電動推桿？

這會讓你從「知道有六個自由度」進一步變成「看得懂六軸平台內部的機械構造」。


---

這一課正好可以把你已經學過的 XYZ 直線運動，延伸到 RX、RY、RZ 旋轉運動。

你已經知道直線滑台的運動鏈：

**馬達 → 聯軸器 → 滾珠螺桿 → 螺帽 → 滑台沿導軌移動**

那麼旋轉平台呢？

答案是：**旋轉軸也有馬達、編碼器和傳動機構，但不一定需要滾珠螺桿、螺帽、直線導軌。**

我們先認識最常見的三種旋轉驅動方式。

# 第一課：旋轉平台是如何被驅動的？

## 一、第一種：馬達直接驅動旋轉平台

![Image](https://images.openai.com/static-rsc-4/RrBGm03QneOu2FCxD5OW3yBIuGaFb-syGYNXWfIdtrwwT6LSRlZDxg1bVpIyfToeoljK-bG8Ao_Ab0FA62OePETU4plhWFo73mRXDNXFaGXYEApSfQPIDR1_BnaQxWBs7gSlgOpqRUNXikwrtPymW1nMyG06pHfivb25BMkuCQeKqqqLIe0XOffGExWY4G7k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Y4UJY2M_sj8G44BT9UROiFU-wPNKS790_RYzgMnK2ac-dHexcbeW6ViMb5fLrloO7VGrkyYcbTKhItdbHAcvYPyTBfSZFHUcW3B8UEz-eMi2zGO1bBIWpPjfBrDnHiXFaQ64fYMrb7csHAoJWZ7JXFJXXIIyioWZhTkfmjvxZTCcoAhI8XvKCDw_NpsDX2_f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QPgcvadAv9sTeQT1Ao5iMH4qTBLQqHiiAmUxKxsCxpg5F6KBceoHHK5d3jrPc23KoOj8CbW62LSdzshw271wmIkGajvpwKZk4UaA7m9ZcfXwhBLrFoKVEI02dkh4MmtJ6b9GOawJ2AZWkYxNtjuZjitr7b1y5D_XnFsuzeyOnDm5lHdrjW7IvHVIP0eeew7_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nsuUu8qM3t5IkmxzZZFJntN6_FXefKV4U5wJlO1sAFACMU5nGsLHT1CZ8Xb30Ka8J_DZmHfRfSS-ztuQ0iXxkiU7UhWLiId3GOLqL4dZ4VgcM2bdz2wGT7p23I1N8HLrqMGzgyJCty6_UJ7QFBuqjJ4odkUqyESUk1a6AeRpeBQ9JXAlYUoa-VjT3BGPEBNN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1TMvYRtT0fdQ-xE-DNZm6SWNNcA2DlB4pkALGWmXsjg6Jud3o7cvqVhZ2lQ_nmB1RphXlwu80rMsNkgnXTJImHgelfaYDCXtAEyQv61qj-zfDpidk066HxNIFpT_Ih-MDiQyZt3jYR0TGi7B6UHwemkX27DQb5GVh-bVLX1q-FJ1k9DKM4dTG74p7SX-CJu1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VRZd2AkpYhXAk55hRbLGuuh4W7PT11gscWIbBba9WSfV9i4ABFmXrr_mbAUUpe5eDPZOpqlZYUdQ8wIOcGq7JRLKjMHJkcUOH_bVZti3ROU75z53rmDJxaaU27v-bOwKjZxxlehLN3LSl2w3MpaYq4xpR0j64mstS1SY_fjJjnt5Wi0VpjhlDChwjvMTkyIm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TgxnsyeKR1VN6M2fFSVBdTLYf2BvbITcnNGA_cGYs4orhb__Y1SjP0UDT2Ork1xhOZHdqq7PmsnOzPI7KLQ3Tf6SbxtSphJ-xarM2LAjZQNefkuuKJXQhFA1Y32f8BvYoWpfwTNea2g-KKLA7sA1ev1ve33pFNSks-qkZZFYtuZ3s7HihGVw4TaIwFDGm7I5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dpVBgW-Y5JJPpiqHAnuLVoYj1KsGZwN1N7bQ4XTEu2jdWhjx8a9aAEvrvyl8NX_74V1ii9_P83bCR0cCLTyuB8eLYmme1v1PHB-q_L4RK2gMhv7uz69WqQkIDYb8p6KXJY5_RwxOX7s7IKUZKYWui0N9baG_o7hUUFu3rS7rbD2Iq-TQ6xTx4ZnexzQ-Nzei?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oVhdF2NQGSeUbueis35voXZlJ4HJrhBE2U82ebNEhTqUsiF414vJsuq76LM_bcn7sSAMgJniUbxVSWrdAdfAa6Kb_Jr3kt-zcRDye1b8DidKnX6zugHB8nP0FdUxBTswvubVoCz6iDKI45txSQkaD7b8DSfTXGxUVENcK6xSosN9XQVRu4yGcqItckC6lVX4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CYI0ncCFh1Xu6Hb5hsTjugSHg4DtYoti5EQjikaJVMfrDl-3CLZoeKGVoCoHEnpiu7q0ktw62SwfryF9kyEOHOB7wQwceO0NLCbM0pdWQLWh7sN-HuLDQDGTA3ccz3XE15_rU9m7JYHSx7m55UAsPUiuUd3bk5KmDbrUxCutHe0jJLFJBWOx_4dckaTRjRAX?purpose=fullsize)

這種方式稱為 **Direct Drive（直接驅動）**。

它的結構概念是：

1. 馬達定子固定在底座。
2. 馬達轉子與旋轉平台連接。
3. 馬達通電後，轉子直接帶動平台旋轉。
4. 軸承支撐旋轉平台，讓平台能平順轉動。
5. 編碼器量測旋轉角度，提供位置回授。

你可以把它想成：

**馬達轉一圈，平台也跟著轉一圈。**

它不需要聯軸器、滾珠螺桿或螺帽。

### 這裡出現了一個新元件：軸承（Bearing）

你之前認識的直線滑台有：

* Linear Rail（直線滑軌）
* Linear Block（滑塊）

旋轉平台則通常需要旋轉軸承，例如：

* Crossed Roller Bearing（交叉滾柱軸承）
* Angular Contact Bearing（角接觸軸承）
* Precision Rotary Bearing（精密旋轉軸承）

它們的主要工作是：

**支撐旋轉平台的重量，同時允許平台繞著指定軸心旋轉。**

這跟直線滑軌有點類似：滑軌負責支撐與導引直線運動，旋轉軸承則負責支撐與導引旋轉運動。

---

## 二、第二種：馬達透過齒輪減速，驅動旋轉平台

![Image](https://images.openai.com/static-rsc-4/QPgcvadAv9sTeQT1Ao5iMH4qTBLQqHiiAmUxKxsCxpg5F6KBceoHHK5d3jrPc23KoOj8CbW62LSdzshw271wmIkGajvpwKZk4UaA7m9ZcfXwhBLrFoKVEI02dkh4MmtJ6b9GOawJ2AZWkYxNtjuZjitr7b1y5D_XnFsuzeyOnDm5lHdrjW7IvHVIP0eeew7_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/BET-DNCLXGecn6aw9mTK4CdDK-LeGm5nI7BbLDAe0doIEMyLOTYUYPpKTnw4DZrqX6FOMC8cmFmTXEw8cR3bHKdMzH_n4hrMS_jI-7THthPbORZ3gqaK7p5m6YhNgEe3PNd5W9Hige3umbeuBhDksIg1qj5-SWc8HYSE6RELnd6aSrP6E3ZDZfAaeNGHmrHS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Q5enZ__zeWK86go_LHHNWeLHvkDaOSA4HvK5oOF7wHW-0mFaM_EqF0H0setwz0l2Z-zFclJAbPaGCL-3hbDeSDwUNTxBi3uuxFUXAnjlDZQlQcp4dPcCOqMJ6P8T5lL3XiqU1xJXPSRanh_g60Q9NTdCYLmaz559z1q_bdSy6dn0QaNH6BACa-58h0PpImsh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0I2wHMAU4SoLpBylyd38irbOBFQeXhuJSazHYCUyQmyIYaOz9uWpY1aBuMY0M92oJDx1FFwKwrCDRwcqzAofyKpbna-lEJCmcFHsrsnwB0ffSNBz8_KxnCtpFHuOdP91xvw9iFgDG5GkiKOhwsnMHto2zgl1vEuKrXh06GSLZ5A1cQ2Pin9ieAkYx-Rk1dBc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iW1TnsR99gxG5Ty7x-ODYY7NTSvkCftWysl7qqiwgu4nKIYEMZ40BdJxhhLxHB-auQ7GHkm_A2cxpwbVJowkZ4oSHLxlPq46kfHxpFdHCNQTEE5sK110qhdCC5fvaQJddTL0a1s1uU7hWT5smkZws7Mr1-hwREzNui2FNpa0Mbh5P0YD3-YXWZ7MVlsb-51y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2kU9eW7o4vPpGfvwW4Y4SQrWxVqe-8ya9QC1ABWeX6KLL0whlPVaj_ZLegKnz6MXkD-jDHATjXItgl4AERuzIV_YR2MdvDoyr-7ld_QRRz7b0iEvcIvFFXnvKPxuwOl35zSrXw5gQal4xbakC2K4W9L1BI88eiU3QkiNJZrSwJQo9JfFWqlKHh6INYSp7ioZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KtWiML8_FlE5vyG31WgSAdTjvm01j9DKI8Muby8NuQaVlTrxLHrUu6fGfmgcckazZjo9caABA33nLYOQ4irj686TX0ESY17rbWtC6liYZQykDCsbP_rD4jbJULMOZdG3s-hrzc3fSj8o9TXW-zGyvJqj9pizHLFbFdsYv4F4lfStkOZd3c1e5F7MDftw5U-u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZR5iYYfNnDS-Myw_Hf9WP-AkZb8Vpre3nQgZ3rcyXjv2_fNTyKiKJ4h1JT2J5MvpQ46lIQ3xdNIV9Aojyd1c0CMDFcxKWvfT0qGrxlwi-2PPDpNlN3VKKvtZv_Zzoe5l8hCcRZzZqHh5F-acjz1aiBYOYS9ooILVDhCTIIqU2fvcw-t5TDST3_QCcuRubzFC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ebMAm8T62Zq9Tl2d3xlPnU4yoknp7zfbH-0hgPbmbw-WkRJmSei23Xhtsj3jxluC4QLUuA635tNLBAWcDPx-C8fRn7uSEGY1hS92vuGsOfCvpsuZln8r1m-x7eRVqqA-66W-AMgGQVIShV2FYeyCPmI_Sdm6B3oPxOIHcY1XPlAs7g4aZPhLrp5d9w05zqV5?purpose=fullsize)

這種方式稱為 **Worm Gear（蝸桿與蝸輪）驅動**。

這是很值得你認識的一種機構，因為 GMT 的旋轉平台產品中就有這種設計。

它的傳動方式：

1. 馬達旋轉。
2. 馬達帶動蝸桿。
3. 蝸桿推動蝸輪旋轉。
4. 蝸輪帶動上方旋轉平台。
5. 編碼器或其他位置回授元件提供角度資訊。

### 為什麼要使用齒輪減速？

假設：

* 馬達轉 150 圈。
* 蝸桿與蝸輪的減速比是 1:150。

那麼：

**馬達轉 150 圈，旋轉平台才轉 1 圈。**

這樣可以讓平台的角度調整變得更細緻，也能提高輸出扭矩。

GMT 官方型錄中的 ARS 系列，就是一個實際例子：

* 使用蝸桿與蝸輪。
* 減速比 1:150。
* 使用交叉滾柱導引。
* 搭配五相步進馬達。

資料來源：[GMT ARS 旋轉平台型錄](https://www.gmt.tw/Images/uploaded/Catalogue/EN/Precision-Rotary-Stages-EN.pdf?utm_source=chatgpt.com)

### 這裡有一個重要觀念

旋轉平台的齒輪減速，某種程度上扮演了直線滑台中「螺桿導程」的角色。

但兩者不是同一種東西：

| 直線滑台        | 旋轉平台                |
| ----------- | ------------------- |
| 螺桿導程        | 齒輪減速比               |
| 控制直線位移      | 控制旋轉角度              |
| 例如每圈移動 1 mm | 例如馬達轉 150 圈，平台轉 1 圈 |

---

## 三、第三種：馬達透過皮帶或齒輪驅動

![Image](https://images.openai.com/static-rsc-4/nsuUu8qM3t5IkmxzZZFJntN6_FXefKV4U5wJlO1sAFACMU5nGsLHT1CZ8Xb30Ka8J_DZmHfRfSS-ztuQ0iXxkiU7UhWLiId3GOLqL4dZ4VgcM2bdz2wGT7p23I1N8HLrqMGzgyJCty6_UJ7QFBuqjJ4odkUqyESUk1a6AeRpeBQ9JXAlYUoa-VjT3BGPEBNN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kCQoPKvkajMK7274_lWuurXL_NzJ_sQlhIqC1qD9_mySlZNp8zIcWV04RUWsCgdCiy8ab4wrV4FGVMCju4R6mRgGN56Uu1lPzu55IMQr1mlabgMM8YmOObFZ6HqzGYXYhRdSxjiRA4N7nmzFU3msMLYA_sNpKlgc7uOLxvZ5w1TCCm5R6HszbjE4N9z5rzyl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-SWMS4MLh64LTfymL5JwNSrqDrd4C0R1AsLCx8MzQbuzcPsbhm32v_pEeBcDOnhaf4H4lQt8xktHmhmsXOQamI87XeL22A0tvnBa7JX2JD2Hwjyu65TF5T8QzE8FEJ2Ux8as_TC0IWOy1yuWm0efBv32pe-xsWZscshJJoVMct98vIUBfayQgi-y8PP4d_oc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X00szICgRW34fbtyltw3rK6YgElOAq0OU7yy_kEO25qrAQftgnxamki6uKy3qt-46YUeS0DXJ_14Atc1q0PyW6qcW8yVeWIPXCWgCGWH2pYR1kqaiATpqb7B40o7Gig9J9mMQqreZeK95UtsdVSMlxfN-AJx5sx0ODpnwAwqRBe_wNopQEAdUftTBlswsQTm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/P_SFsF72b_tGemRsRbF3l4cO3e55Z6pSfMmL5yJxzX1Fmwk5urZyClnuS4KKtefgM07sDDO5eqCH0eqIjNVpclGgYJiHGW0-egDd2-IWpLTPWTfBiiIhu77ymlfIDwdGhnJ5U1kHXjB2JJ02zHElggDqg2SZnO9aScq0CAB5bdKHlLB6xwKRbnBqkj38IJ3Q?purpose=fullsize)

這種方式是：

1. 馬達轉動。
2. 馬達軸上的皮帶輪轉動。
3. 皮帶帶動另一個皮帶輪。
4. 另一個皮帶輪帶動旋轉平台。

這樣也可以讓馬達安裝在平台側邊，不一定要放在旋轉中心下方。

GMT 官方也有皮帶與齒輪驅動的旋轉平台產品。([GMT][1])

---

# 四、旋轉平台和 XYZ 滑台，元件有什麼不同？

我們先整理一下。

| 元件    | XYZ 直線滑台 | 旋轉平台    |
| ----- | -------- | ------- |
| 馬達    | 有        | 有       |
| 編碼器   | 視架構而定    | 視架構而定   |
| 聯軸器   | 常見       | 視傳動方式而定 |
| 滾珠螺桿  | 常見       | 不一定需要   |
| 螺帽    | 常見       | 不需要     |
| 直線滑軌  | 常見       | 不一定需要   |
| 滑塊    | 常見       | 不一定需要   |
| 旋轉軸承  | 不一定需要    | 常見      |
| 齒輪    | 不一定需要    | 視傳動方式而定 |
| 原點感測器 | 視產品而定    | 視產品而定   |
| 極限感測器 | 視產品而定    | 視產品而定   |

所以你可以先建立一個概念：

**XYZ 是把馬達的旋轉轉換成直線位移；旋轉軸則是直接讓平台繞著某個軸心轉動，或透過齒輪、皮帶等機構傳遞旋轉動力。**

---

# 五、那 RX、RY 是怎麼讓平台傾斜的？

這是下一個很重要的機械結構。

RZ 比較容易理解：讓平台在水平面上旋轉。

但是 RX、RY 不一樣。

例如你要讓平台向前傾斜 5°，機構就必須讓平台的一側升高、另一側降低，或繞著指定的旋轉中心傾斜。

![Image](https://images.openai.com/static-rsc-4/-J4_Ects6zh5uZjIihitgVuOdDXJoClMOzfihhYTx5u953gnwRmbW6KWoKG0xqlIAKD-UM8Zk4DOTadX3dpXyo7aYB5pfbe8bfSJ1_FUWPi-iSL2ejRyAVWHoB1EFWKNkO7by92YCp69gB02AgblqR9F5-3IB-E4Up6GsBejy06SNMptSUGfQeZJMuh5mY3e?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jB-fsysF7-G4Aflzqo9GY_xaZUXX9SQ1I8aA6oBfECT3cGydv6iZBkY5Z8kQin1CkrEaR-tuThslEg82ZJfib2_RAgCIjEYp8rFo6T6IkESy-YWdt4Em8skLeeB06njp7ZnMDq7I06_W2ve4iVyFBG89uuiNu0Lx_yyKp_-HYp90mi6ldpalefX04B4qPKC4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QUKm2yd6cK5xI6WrX1p4JcuhIztBEtCABtNEf-g0PxnBjwRJgbTOE8dOQyr-yOYYqu86BcxF-mH1ZJt8Jt15ZdVlUZS8qui54Q4noU_Th_uegYqWfNjn426VFezTz3mCqtIuq53HN0dudWBrc_T-skrNmKGnrcOqcseB_Bec7lX0U31c1PtURtDOsu9STP1W?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IyeFtYsfav18ktOlfRYeB0yKEhoT8XaJRYUAMnOKcJkKT1Rfq_pWVgHThxuf3dEZubSRMA3weN12MVAYWCL65Yj1OCvLE3H1eVvpspu01kHjUBxJnokuE_yEXi4SnSi8lB7fl8tJmFHRM0PnTwSOoDyiROWBKgpDqSKWXSJ-6UyBaHsaaPKN-G8acyML8Ao1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4peCL7kFrFtWKmnpOlNRNoVxwJswlHcpojc-Q1ZR2qnedzayBef-ESAP8ubAWe4seRt0uf74SGkM87PKz1j712rRw1DqQx1UGVJGO1KqODc3s4DuaUY8aGL7nhLPhI3pUEMGSkHAM4KlziuyjDNI8aLg7191hicCtCTpfB8taJWqdF6RCt6qvmpNWrq0uA98?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oOUF8siXTs5ICUXUQIlb4Rb0espsUHFSKy0IkQ_7NdQcoBGbNQT5KMaR_Y5uUmQsEYIpxvZ_FV3WoaDrve2Fgw_LBBAFIVxHhnzRRAFXMuSSXpIahchq2SfkMSq5_UIVNfUymsivFWnbwrV1xKFvPcCEPhh6yaAPPGBwOH5v-Z902LnxYQiCRIXJEqArLsjl?purpose=fullsize)

這類機構常見的名稱是：

**Goniometer Stage（測角平台）**

它通常具有：

* 固定底座
* 弧形導引機構
* 移動平台
* 馬達或螺桿驅動機構
* 角度位置回授（依產品設計）

### 為什麼要用弧形導引？

因為平台不是沿著直線移動，而是繞著一個旋轉中心轉動。

想像一下：

你用手拿著一本書，讓書本繞著某個位置傾斜。

書本上的每個點都會沿著圓弧移動，而不是沿直線移動。

弧形導引機構就是用來約束這種圓弧運動的。

---

# 六、還有一種特殊結構：六足平台（Hexapod）

這個非常值得你認識，因為它跟 GMT 的光纖耦合、矽光子對位應用有關。

![Image](https://images.openai.com/static-rsc-4/HnW3Gbzk26ql0ciSiXPceuz1YJbIw7352B7vfZ6ZcOa9braQw0ckQ2oBxykpYwxVUsfgMWFuZRbx9LZJNlX6Mv8NFSmdPcv7gw7XgRJF5m3FN8-0cmeoUi3wYljwd0rkH5Zu4UUk71hMAz-zTnDeq68-qDjQKCSfqKcKCQ8ndJHSLXkj_CgmpfbmEjN5IAr_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SCE6iqq_SmanUaGKvj_oDE8ArqtYBgmQ-a9e2GwW8uXsFZIOHh00QZ2XPdnhV6tLqg4Nkx_axyz_e3QPUOji3UC6KZSAZuRqOdXPNRt7DFJldSU9ZpzSlpBiJ312xRWuA5RJMYgslsQvFmP_YleVnjw6hbifljKHtADNG6e38S6p_1HB_hlDd3jxlJgxmw0O?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pgh0o9BWDoqKOc5TU79kNCXxcUJyUmcl2JQFEOVLwX5NKcF9v9DFDPjMd2HgMNKqpJUPwQw1fxxtzM0eOjx5uBR3LSO92yrKk7S7EWLl0K8wxEBXYkq6HHJk7uULS0gx4GILKyA9GX0Wbxh46M_4RFi-ZPiFs7mzi2-9IRp2Xfrf9iObhwRh20dbHVQpPfNw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Di8yfdeNDuOIF1_zO5kXvTe9SsWQgXgA5ng7Jmj4cqHMXlHZMTxrQgPZMql055DKsOmH9Ai9m_G7nCOpy5ltP1OFCDTc3Zdzrzex24TciXLyrXrJVZt2DzTHSJlhe3kU-z5R5-Gk876oExn7rLugDoTjIDfqf0u9JO_FLta2e3M-wxOl2qN9Oqz_ZXOntoq-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VEz9cDVeNb5gajyjfY0KeK1m8tRMJwBwT-kYUSnqFvDrlej-GVPH8oPWHPdlZrxc1nzNo1hwywfrYvGHe_C7FSPXaN3FJgJ16rAvf2Rf1bt_AJbOYCFPHLv_8vGHDOlzIBAf6Xw1cdkuXkEBxoirD3Tuk2wh5d7tA2DOnJ5bhWlWle4hyQDfKM0GqCj5skHK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XPAc45b9Wh2QcygudHaCcGahANqoGoNCY-pyeERar7-crgta-boLKeY0JUwlJRTC1svANhJu56iQjgK3qO4BCZ3BJ1hewWXuNxd1gTdZIx2BXQzAk8-3y8_tBPNKc3Y3V4lPq9o8-iPPetUVmFCZcUR_Ow6QLTWZ0OPk3rirJ7V0NDFRUQcj4gl9rLyrqfDn?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9mdtzzwZuDgsLnNU1jzZWZgWgGDrSVPhVLViCo5DWy-u5JZO3etmuerjrFCZQBWLjPRs6PlfjrdUnlJjNXW486Jxk9vq09Jg25uCqOA1AZ8ZTk6EErBEvY3K1PQ1Tv7KrDaTCYzCC7tTRJn88Nlg9TGdDYDjazKWCz2cpWH-bl0xy4F5EouvLLgUo-01yDAh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0emfItX-YY7I_cBO9WbKkzSEDgMhbaBiFlLZiQezHMyBJuRaZowA7EEnxQP_R6vrFe3t7zzP6AWRc5gXCnhvdqPqVAwEbXkMBavuA3aOMf2uQAsImtZRqJXopWkz1EGd_jn-0uJ8uvkoivN6DUZNfEZBSdsQsC64MW0ZgYjg2hOrCucYHrejqKYsY0LAYMIF?purpose=fullsize)

GMT 的 GSS100D 就是這類產品。

它跟前面介紹的 XYZ 堆疊式平台完全不同。

它不是：

X 軸上面裝 Y 軸，Y 軸上面再裝 Z 軸。

而是：

**底座與上平台之間，使用六支可改變長度的致動器連接。**

六支致動器分別伸長或縮短，六支一起協調，就能讓上平台產生：

* X：左右移動
* Y：前後移動
* Z：上下移動
* RX：繞 X 軸旋轉
* RY：繞 Y 軸旋轉
* RZ：繞 Z 軸旋轉

這代表一個很有趣的事情：

**六足平台並不是使用三個直線軸加三個旋轉軸，而是透過六支直線致動器的協調運動，同時產生六個自由度。**

GMT 官方 GSS100D 資料也明確列出六個運動軸，以及旋轉中心可以透過軟體設定。([高明鐵 GMT][2])

---

# 七、今天先記住這三個重點

1. **RZ 旋轉：**可以透過馬達直接驅動、蝸桿蝸輪或皮帶齒輪等機構實現。
2. **RX、RY 傾斜：**可以透過弧形導引、測角平台等機構實現。
3. **六足平台：**六支直線致動器協調伸縮，就能產生 XYZ + RX/RY/RZ 六個自由度。

## 小練習

假設你現在要設計一個光纖接頭對位平台：

**題目 1：** 如果只需要讓光纖接頭在水平面上旋轉角度，例如 RZ 旋轉 10°，以下哪一種機構最直接？

A. 滾珠螺桿與螺帽
B. 旋轉平台
C. 直線滑軌與滑塊

**題目 2：** 如果要讓平台向前傾斜 5°，哪一種機構比較符合這個需求？

A. 一般直線滑台
B. 測角平台（Goniometer Stage）
C. 一般滾珠螺桿

**題目 3：** 六足平台如何產生 RX、RY、RZ？

A. 只靠一支馬達旋轉
B. 六支致動器協調伸長與縮短
C. 靠一條直線滑軌移動

下一課我們再深入看一個關鍵問題：**旋轉平台的編碼器裝在哪裡？它怎麼知道平台已經旋轉 10°，而不是只有馬達轉了 10°？** 這會直接連結到你已經學過的 Encoder、Feedback 和閉迴路控制。

[1]: https://www.gmt.tw/Images/uploaded/Catalogue/DM-EN/Motorized-Precision-Positioning-Stage.pdf?utm_source=chatgpt.com "Motorized-Precision-Positioning-Stage"
[2]: https://www.gmtglobalinc.com/%E5%8F%B2%E9%83%BD%E8%8F%AF%E5%B9%B3%E5%8F%B0/gmt-gss100d-high-precision-hexapod-platform?utm_source=chatgpt.com "GMT GSS100D 六自由度精密史都華平台 | 高明鐵企業股份有限公司"

---

三題全部答對！而且你提出的兩個問題非常重要，尤其是六足平台。因為你已經理解一般 XYZ 滑台的機械結構，現在要做的是把原本熟悉的「直線運動」概念，轉換成「六支直線機構共同控制一個平台」的概念。

我們今天分成兩部分。

# 第一部分：旋轉平台的軸承，內環和外環誰在轉？

先回答你的問題：**兩種設計都有可能，但常見的旋轉平台設計，是軸承外環固定在底座，內環連接旋轉平台。**

先看看軸承的基本構造。

![Image](https://images.openai.com/static-rsc-4/yQumGGpBH9ESjkAV5DVrNAfoL5QYhNPehAKNDrvXreWnTR9TbhuMjjPqwUpZ84bSdICwk12el8VrVD5B4IpT5vB08AjdcYPJNaoZMboKq-4bQa-Iet_mRd18KRwJ8FVwzzyrSNheMlQkxa3PYuWE08Ob8WzFgzP8dd_6hC9DG7MuaB4MarL7Q4jBbKJD3J_u?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HKzeaJduI9ss_xV0wEOjk2w5AKS0ILvhawJ5MZPG_ichiSNNidlMS40ZtLcFx3BoOYTcQXzOYivlM-bltt9gaFU7KNzCO2NxkaJFoy05fijAeKwe-9g4q79FT-Ns49jKQMq1qTyVMyf8Uq7iBEw1PFhRniJ2QEbjQLtIgwcv8YpVKSDhyWZxnMYJr94o-uLy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3NJpa5O_hPE42P4PTZzYejsDoDvJXVEdNjtzCYSVfHQrdGBcUG7Dk9zqbJoVcdlhbVWhbSTF41vKBAVFGvNS48L9ogsjbJo3MUFQwWp2gOrShUoDlbr9CY0evxb5XRO_IKzsrJsuh74UQso0YFnGywTSHX2iY25p0ZiV2h0n-zaxpMHnJKtgYjsV0-A1sQQb?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QPgcvadAv9sTeQT1Ao5iMH4qTBLQqHiiAmUxKxsCxpg5F6KBceoHHK5d3jrPc23KoOj8CbW62LSdzshw271wmIkGajvpwKZk4UaA7m9ZcfXwhBLrFoKVEI02dkh4MmtJ6b9GOawJ2AZWkYxNtjuZjitr7b1y5D_XnFsuzeyOnDm5lHdrjW7IvHVIP0eeew7_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cZzywxeLTr-7ROFFowa4s_VC8xLbbqbwAsD1aJPJfNGHregDnCdFzqdRKfLDqRnwk5VQQtm1AJramNCTAgvJVcboR-1k5JoyMB8EGxEx64xM_xp4APuARWxCIBse3w7Ppef9mAKgxRJeuSftC6hxHJvnyzMtvZjaiijyB-wd5HYn1AiSkp7RI8KvT-nVJkIQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tSYTe8XqTzmAAwsjCa3pT43ChUneAeMlczpQG4jLg98k1DZfAg-O4MJj8mxXVZ6YoYLxbPMGGbJUrN--8w25ly8X7FhF0L1MkI68R609SJC-0A5KfdBQvu-lGd5xct0Xm-ugElyq-D6_ySDCUHBrYw7ZdLpYTeJ29Jfym0nHn1PSHeXCuZvN17qP5xZmrkCK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uc5eBq0-ejw4FtFwWEMMRt7dhM5v-G3aaXPgkTCDwOEj621ckkWanOypqYR-oPAf5VuKN1b0tuCZE4_kX6JhIK76M6MxnNkljUp-XoJ1W2mtyn_etLnSgmF4lL89rBlA_c6iJ33nGK71roMORkcnMW_s8K1wiHhsPOb3Mv_Qkjv5FrDW3gZHCY3MH7w_YqTV?purpose=fullsize)

## 1. 先認識軸承的四個部分

以常見的滾珠軸承為例：

| 元件             | 功能           |
| -------------- | ------------ |
| 內環（Inner Race） | 支撐並固定內側的機械結構 |
| 外環（Outer Race） | 支撐並固定外側的機械結構 |
| 滾珠（Ball）       | 在內環與外環之間滾動   |
| 保持架（Cage）      | 讓滾珠保持適當間距    |

軸承的主要功能是：

* 支撐平台重量。
* 允許內環與外環相對旋轉。
* 降低旋轉時的摩擦。
* 限制不需要的偏移。

## 2. 旋轉平台通常怎麼安裝？

假設我們有一個旋轉平台：

![Image](https://images.openai.com/static-rsc-4/RrBGm03QneOu2FCxD5OW3yBIuGaFb-syGYNXWfIdtrwwT6LSRlZDxg1bVpIyfToeoljK-bG8Ao_Ab0FA62OePETU4plhWFo73mRXDNXFaGXYEApSfQPIDR1_BnaQxWBs7gSlgOpqRUNXikwrtPymW1nMyG06pHfivb25BMkuCQeKqqqLIe0XOffGExWY4G7k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Y4UJY2M_sj8G44BT9UROiFU-wPNKS790_RYzgMnK2ac-dHexcbeW6ViMb5fLrloO7VGrkyYcbTKhItdbHAcvYPyTBfSZFHUcW3B8UEz-eMi2zGO1bBIWpPjfBrDnHiXFaQ64fYMrb7csHAoJWZ7JXFJXXIIyioWZhTkfmjvxZTCcoAhI8XvKCDw_NpsDX2_f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QPgcvadAv9sTeQT1Ao5iMH4qTBLQqHiiAmUxKxsCxpg5F6KBceoHHK5d3jrPc23KoOj8CbW62LSdzshw271wmIkGajvpwKZk4UaA7m9ZcfXwhBLrFoKVEI02dkh4MmtJ6b9GOawJ2AZWkYxNtjuZjitr7b1y5D_XnFsuzeyOnDm5lHdrjW7IvHVIP0eeew7_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/c159ALPUt-1lydzSuTOT-yMIjd6isELrS4gss9NVWCjrS9IurcB9bSj_fKgWu_GXEC0TnEEz657B0JOYyGCO1KqpYA-jMHHXGYNsPnCfISmsbsEIH1HFhIceIUYq9qTdJtW2BRSEkliiYNBE_yYXBg4BvduNbb6bvmuF1AkhAiTjOSKUn_3o2u6P8KVQ3JkZ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/TgxnsyeKR1VN6M2fFSVBdTLYf2BvbITcnNGA_cGYs4orhb__Y1SjP0UDT2Ork1xhOZHdqq7PmsnOzPI7KLQ3Tf6SbxtSphJ-xarM2LAjZQNefkuuKJXQhFA1Y32f8BvYoWpfwTNea2g-KKLA7sA1ev1ve33pFNSks-qkZZFYtuZ3s7HihGVw4TaIwFDGm7I5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/I7ej8vJ0WGbIxS8JlxO0i-IcTMTzFIMkf_bACi_RsJEYHmyx6IJmfnjZMoUotax4ciqx_Lcg9rJJZ1pMgAqMziWN8i9EmifCY3VQpuRd6Ko8_4Lu7vwdSGYYKGUDKL2gs4iXrP5ejvWETi0w6PhgCc_5G57UOnEU-NSD-CxqNnlYTtXcJ9OE36jsXMdXEIRx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nrYNTmagimjTK_EpyNgbbkLGp1pNGS4kUrwYcr6B1hpt6uZQyAGJ6j7wxurklL4i0nvUL-WpyQl_9jcY1M-weKR9_5cI5pthfGrxa7CQ0bfLp2o5gwz1izSp62yIwaj6gUsMnq4YfhUWCCaptn-AzS0goYKMoiXb030ZiBPLej36Feqgalr1Vl2GF0DFBifJ?purpose=fullsize)

可以這樣理解：

| 零件   | 固定方式    | 是否旋轉 |
| ---- | ------- | ---- |
| 底座   | 固定在設備機架 | 否    |
| 軸承外環 | 固定在底座   | 否    |
| 軸承內環 | 連接旋轉平台  | 是    |
| 旋轉平台 | 固定在軸承內環 | 是    |

因此：

**外環固定，內環帶著旋轉平台轉動。**

不過，這不是所有旋轉平台的固定規則。有些結構會反過來設計，必須看實際機構圖。

## 3. 那馬達的轉子與定子又在哪裡？

這裡要特別注意：**軸承的內環、外環，與馬達的轉子、定子是兩組不同的元件。**

以直接驅動旋轉平台為例：

* 馬達定子（Stator）：固定在底座。
* 馬達轉子（Rotor）：連接旋轉平台。
* 軸承內環：連接旋轉平台。
* 軸承外環：固定在底座。

所以在這種設計中：

| 元件   | 位置   |
| ---- | ---- |
| 定子   | 底座   |
| 轉子   | 旋轉平台 |
| 軸承外環 | 底座   |
| 軸承內環 | 旋轉平台 |

馬達轉子帶動平台旋轉，而軸承負責支撐平台。

**馬達負責提供旋轉動力，軸承負責支撐與導引旋轉。**

這兩者不能混為一談。

---

# 第二部分：深入理解六足平台（Hexapod）

這部分我們慢慢來。

我知道你現在最大的困難是：你已經知道 XYZ 滑台是怎麼組合的，但是六足平台不像一般滑台，沒有明顯的 X 軸、Y 軸、Z 軸可以讓你辨認。

所以我們先不要從六軸控制開始。

先從你已經熟悉的單軸滑台開始。

## 1. 先把一般 XYZ 滑台想起來

你已經知道這種堆疊結構：

![Image](https://images.openai.com/static-rsc-4/ZA69ySRaRxlZieQwZ689NEFoLmnDLBA4K0b6ALWc7x64m7JZuMIo-ifEqt7xVvOo-ERwgDR-FJdMeFzXv6n4O8O2O3aL3RQ9j3fJBCqa4Ofl0wprcFhz4GuBbizGWrBVgePnR10vBP6Y2gIR3yL-QJ84ErRMMvkhns4GbOeiEVNH3TVW3vmgOeJMApB98sSz?purpose=fullsize)

假設：

* X 軸在最底下。
* Y 軸固定在 X 軸移動平台上。
* Z 軸固定在 Y 軸移動平台上。

當 X 軸移動時：

Y、Z 整組跟著移動。

當 Y 軸移動時：

Z 軸跟著移動。

當 Z 軸移動時：

只有 Z 軸自己的平台移動。

這是你已經理解的三軸結構。

## 2. 六足平台不是把六個滑台疊在一起

先看六足平台的外觀。

![Image](https://images.openai.com/static-rsc-4/-LbpTHVW2TJoi04ARanEcVx1ETiUn8pEMlCn86pg-QDcl_nKB1R5egZNK_ZKVhCl_szqHDZ0Qsyslm83x60L7i69LMtNoJuvApymrh2TXdgzxuGk3qArKPMvrA950wRuPg5t-2w2-rsKEaIh2qranE4zv4Jlbf_2no7YjiJG2cK612HTrkHJ3zql0vPljKHu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yokVD4tUDVARBSxdBpfHOqXpOeh-0a7lg0y0PubxkwDvprcV-CwflnMeR3kQpAsWX9FaTXKS0c6uWAqgxnr8guEfP2e_bc19LV5D2JEuX_TH_RbT3yg9RM8EiaKAi3_dEAvtYppd5EsDzZ8YHt2i00y8Zj_TbJQc9w4fas-f-a0-3XF-06Gc_ADeJhRpHMwy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5k3GdQRz-Ytuy83KCRBOkVxBODdSJszOZZ6WsjLLXYsGtEVAyvXl3HzBUZ6quueJWHNmVvsO4fxvURwbfeGjPGLEL1-sJDt_OrX3MiJiyuVhoA3ox__TY9zzxuiO46pqZ89mb7OYUmveR9cuGzn3LT7dS_eXDkZXk5H9SC2Vahy0Al7KQV8n1r_Zk9M3lG5l?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oTZr0mpVOnwrbxhy1fEim4yK3wBWhPmCNbDvjYig1O_Ks_P1h0uwgKr88MNaXguh7EuQ4emQrm2ie7_hbjle_bfrK7uqmFQHYuG4CbgTmnHjrjhxwG8qVypi7lRxF0s8sddJyXoVWnJ-5Hr-Q_OrDZGz-igre3K6uBoEsTQoOyfdvOFW_rAhhz1uw_NAvshH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2QeO2M5AzItEoHYUK_evGQu4A19dl95skwLf4VvijqLlwjxjV6IJKYYcMwncVMjspbMnsMGIzaL8pkFTAymOwxsBABiW1bkAHj5DnrtfFu1dXNwYuvxiUnOy7zm8p3qzGOCZvOzeCv2P-Diyvu_FEIZ2wrJ8Bxv2CrGG4-HXNbBp3cPhQJNuUGWjAwfXIlOj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Eari3CpAxGEYYoz9sfAKmiQawAIMm2SlCJaI1MoPotmW-cSz7FCFCoJh5Gl6QaHoIAyy80cD9UJtlnDlDQKHmHkaGjNRy4O8TJnmt44lr4_1lRbzl3xB1Ar44CuN4hK2L5EVz7Kz2zCA_52iPfYAAxU8nQgGw3YD6MNpFq0N7GsUH3uLJ185YlGSdmDb3Usc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Cj3Jj2nk5zuG9AeTtFWur7t_AGYvBPvjy__Do7XwtwRhDYGTQPqgVAewTak7NS-EEXao_4EM5DW2QMsgWG0wLgLWmmqp-tZTaI7a3bnrJdVvGp9A4AXF3Z0BEVh0OOh1GDXx9SZ9waK6hxkoL7Njbi6tQvbIqRAKktkrxb1sv4jw3xK1xGiNyKRkPf8EXCr-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cMNlzC5AmoNfMJ8xVJjsrz_yLp4OZnXcuY3yAiuGHyedtSJLgE8DWEBk3VYcnclzQfd4rUS5rQqfa71Ge0mhfkFnERYDA1kfzQXqhCqxiEYMLIQ1mK7O0Ee8m5qMl22Ko-rGuELlW6yJqc0tuIBKrY7rlfhM0KWi04pIS8OF-e2qg2XY3nasycww-RH-4q5e?purpose=fullsize)

你會看到：

* 下方有一個固定底座。
* 上方有一個可移動平台。
* 中間有六支可以改變長度的致動器（Actuators）。
* 每支致動器的上下兩端都連接到平台。

這六支致動器就像六根可以伸長、縮短的機械支柱。

請先記住：

**六足平台的六支致動器，每一支都能獨立改變長度，但六支必須協調運動。**

這就是它最重要的特性。

## 3. 每一支致動器裡面有什麼？

這裡就可以使用你已經學過的滾珠螺桿知識了。

很多精密六足平台的致動器，可以採用電動線性致動器結構。

例如：

![Image](https://images.openai.com/static-rsc-4/THqyQ20K37XLCgML3_eezp825QX-pnWoLTcX_7ZQFakvIk3sF0LNT2UVlL8uW5C63Hw3gMEXLsu3p6LMbnhJPML0x0HZJmoHsufftRBahMCLqCbBBgRH9fyYKAqyenwKGGwIFEUwWxZmBS8kvXMMJbOP8aLpOANynvv9-TwYH0DGo406yyynGXKJCfzl1QFX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1tpilKJ11JdOw2QrI_r2aa5ktUwySK7_czbMziuBu0BqXGBR1FnNgxoKID7NKA1xvn-lW98LDTr1WRsvJYZvhMEP91m2mAsX09AjHzzzr6S9ifU3U2wryBy-WvY1nZdwaAcyhqTO4DgB-0pzNhSC0AbmZfbhN8lc9G3SMkn7uacDc7h-_qcwBfq6s5LGBYZ5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/apRq5nxoPt-XJY914YCLanwZ0wTAob-ZRzbL4k_xEIll8jsZn0rN122mx42V_vpXbYu7A6W5uAB98M5i4pFX5EhwZ9js6cVCyIugP0NLfrGIlygsBQYTFak6f-_r3w1KxPTew-SBhf4Uqh4piLKjYgQhzIdKneI6BkPfzkcuaeT2BFLecJ1jeABJtEfoKF0T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0emfItX-YY7I_cBO9WbKkzSEDgMhbaBiFlLZiQezHMyBJuRaZowA7EEnxQP_R6vrFe3t7zzP6AWRc5gXCnhvdqPqVAwEbXkMBavuA3aOMf2uQAsImtZRqJXopWkz1EGd_jn-0uJ8uvkoivN6DUZNfEZBSdsQsC64MW0ZgYjg2hOrCucYHrejqKYsY0LAYMIF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nlYCf7mmuMMvyLdZlA7wdSnIJavnKEe70XW_fN0LZqu-NLTFxB0N0SoH7Ksi2R9g4kNsfcO3VP3wBUODaxd_lJHPDio8GUvgMTNnI6xpRCKgbdYkvJG6QfDHkD2dC62mvhp-GctlctW2HpTgSesYLUVJvJ9Y0Ta6gCGWpt_BpDpZbmQ7CIBsQ9lf0E6QtsQ6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QUOS4pU_rFjtscs7gLrsdGyWTSrR_8_vvu5-u5TX6nTzuz6TIIn9SfJNOs-nhfFSTos6i4_4Ox8iKd7Bpj51z3hvCjd1bhY7YBNPY-PGQBYAwUmYotIP9xLYswgUoQU2LropwEJvJHePkCV012Tx-dtBn4mxuXEcQk1kn9pIe-4IcD6SWtameu3uLarvSTqw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wn9yXx9R7pQQiH9AxgD7-ILzKsiIzGU9wTkH9fm7Hta6KXxn4wR6VjpvVXYhPhQN8bGjidg7Irsb928G0mWP5dc6IOucboTawV6YCEM-WOxHdFeEx8DNvV9qmYFha9-eB-I8VTTilAkBrno0cMuMtetH70sptncMYFcS7BunPu2RjuIWrnrWcVyMpskhkARA?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H3HwX4kJgSYzp5qmFcdVT1Q_epEPe1fXy5AI8M26mzWQJ7VlFOjhC124vRghCbv7DCcZHNDthiODCmNa_62Cm_YPe-20g2ut5ai-O-rqrN0mh9admdnkPtP_nDOS-i9VjnUaaioiwboM4P7t1rWo2iCMbGp33WzqePcNhCnwpmwImlOdQUJiKOOCnh56o1Uu?purpose=fullsize)

一支致動器可能包含：

1. 馬達（Motor）
2. 編碼器（Encoder）
3. 聯軸器或其他傳動機構（視設計而定）
4. 滾珠螺桿（Ball Screw）
5. 螺帽（Ball Nut）
6. 直線導引機構（Linear Guide）
7. 伸縮桿或連接桿（Strut）

它的運作方式與你熟悉的滑台非常相似：

馬達旋轉 → 螺桿旋轉 → 螺帽直線移動 → 致動器長度改變。

但有一個重要差異：

**一般滑台的螺帽帶動平台沿固定方向移動；六足平台的致動器則改變上下兩個連接點之間的距離。**

而且，致動器兩端通常需要可以改變角度的關節，例如球面接頭（Spherical Joint）或萬向接頭（Universal Joint）。

這樣致動器才不會因為平台傾斜而被卡住。

注意：這是常見的電動致動器實現方式，不能直接認定 GMT GSS-100-D 每支致動器內部都使用完全相同的滾珠螺桿、導軌與馬達配置。這需要看該產品的詳細工程資料。

## 4. 六支致動器如何讓平台上下移動？

這是第一個最容易理解的動作。

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sfd8zTyvgLsviuZ_DS8fMhx1pWIAu3mxzNGrlOmaVr836qlZFjSDnIp_5if4dwoXRKm2ib6h1tr-moe0nHWQtAJ_tGD_y54PFJOEPnsPVECHy-MOLlsFxMoshsQDKnK0kaDISylnJ5WjmO86A6WQzUUITexKUig4wIxA9ws4Cw9HRBD28IyuF4DaJCOqPTAj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mzmCneR7eetNptGcore6VarPRoGpOgau__V6lcVgdXpFhmLZHOxdMkKXJuQAbxa1tNm2KQMBwh1Am2GLLJDqQ883YlXXaKG7buLuvSR_TwRIh2chZokoVEw46TwHxHlHzdhC3fvr-SFzatrk5yQgGKD2l0vpPzCA3IO6AGxhk2fnoEkT1N1DP80oJemiEYLI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2wP9Lq9XiOfDxyxTfHIytraLvwPwUSj6soMKGMczdIsCowZjc43nybNIWVdYXtPGX04yrDKripoT241l7RcIbgULV3m2ngj4q3rYGqYjc0BJwNyRZLamlROQLFWpl4isqQgdkMesvbq8-a3QSvlTMIbj1MsLgw5HNM_2xEBnKuWQUp5JAIpnZOiPUsYUm19r?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Q5wTBfcYA9puxOIFYnVo9Vdd1oZEBZxJpHKZ63JSfiRN-fua5dINejj8lQNuL0xe1t8mzFPyJ8HnZ3tec33NU7jjffMEXrtx32Syi8oXPhMWoioKlApMM1Bh-VdNiXcTPl6gondc9zygQw619xja3M2aMS34PWJxpxGgL_0T2BOFKBerhSmMwUhPGaNg-pLl?purpose=fullsize)

假設原本六支致動器長度都是 100 mm。

現在控制器命令：

六支致動器全部伸長 2 mm。

結果：

* 第 1 支：102 mm
* 第 2 支：102 mm
* 第 3 支：102 mm
* 第 4 支：102 mm
* 第 5 支：102 mm
* 第 6 支：102 mm

如果幾何配置與運動條件適當，上方平台就會整體升高。

這就是 Z 軸方向的運動。

你可以想像六個人一起抬起一張桌子，每個人都把手臂伸長相同距離，桌面就會升高。

## 5. 六支致動器如何讓平台傾斜？

現在進入比較有趣的地方。

假設我們希望平台向前傾斜。

![Image](https://images.openai.com/static-rsc-4/Sfd8zTyvgLsviuZ_DS8fMhx1pWIAu3mxzNGrlOmaVr836qlZFjSDnIp_5if4dwoXRKm2ib6h1tr-moe0nHWQtAJ_tGD_y54PFJOEPnsPVECHy-MOLlsFxMoshsQDKnK0kaDISylnJ5WjmO86A6WQzUUITexKUig4wIxA9ws4Cw9HRBD28IyuF4DaJCOqPTAj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cjz1KqPzp-sH3W91XICDN9o5q7pcbG0RuWGMQVQ3vhZm7zqlgEBpyLNC9KuayAxINEXcQ0iSvGkq-gHfJnIeD_TkHP9jIfwCvV1LR-cie2x1zJumtCX0OtmcHQye9_Edsm1QORb043t-GhmFG6qXNCP0kQj-deQnnSSvZ7Le1MC3o1IBenRyZ6vrjuILooCo?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/agZzfWVtMpYn48OSSHwg95DWKohkAhyc1LUeuadOV36o_nd0xLFMiXMWLdY74LvzQ0sUlQ8HfAjm1BuLnlbec20yVl9I3JS3vDtMfsd_gGwCtYIqEN8c1r6e-7lBLsw2m1-rml1JnYxvgzrTeo5dn2V9dtZYG9znHkSb-e5_e-dXzgs0PPZ8RVkpDHICNBwa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IEfpd-9eFLYy0NOFE29Xqj9bJ5v74ShJZmyj38aIkvkt_gN_TeweTVvbE5en5pmXEcybND-Ma5q5Z2SzFpmFit0g7mTUv3LR3IB6Zj7sTXalZVSy5CKvRIufdYE4rkKxrswkuTOp5IK5VjwwlzLjJQrCmL8Nnw9LabrmJeYeYS75pU6fTHKcn0veKG3n-28I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mzmCneR7eetNptGcore6VarPRoGpOgau__V6lcVgdXpFhmLZHOxdMkKXJuQAbxa1tNm2KQMBwh1Am2GLLJDqQ883YlXXaKG7buLuvSR_TwRIh2chZokoVEw46TwHxHlHzdhC3fvr-SFzatrk5yQgGKD2l0vpPzCA3IO6AGxhk2fnoEkT1N1DP80oJemiEYLI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0hvJb5Ch-wDo8DjIvmrywk5nEOfAo2WK-WpHnoPg2Q0Nf7wXX-e4Erh6XvREKIqEMDC4RzDukHLwEXCZmtwpWt5ro7zPOMM10-kDya5LTnimuv44-63komQWfctc95oABrfCCY7dx_sfzv6spxyhqpHKJicIcthmbM8g8SzkimB8G4ShTunMqEAAs4SIsZUW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_q7vo-_MKYQBIahHcoNgn1nyaFvzte4VrKoDy1U8qGU8gaEnEDSK4FfIf0y0IZemT1W6c6Xy0C1IppFIhUOBKF9lXyrv4dVcM1CWFC7F9IpU6fdcOJ9T0iFRMCnwZpXWJgcCo1kgnmH9RC36F94a7mn3Xv3sxHqG8tU9kEH2leMzw4LeJkKkV8S3EUzYT4Ef?purpose=fullsize)

這次六支致動器就不能全部伸長相同距離。

例如，概念上可能出現：

* 靠近平台後側的幾支致動器伸長。
* 靠近平台前側的幾支致動器縮短。

這樣上平台就會產生傾斜。

這就是 RY 或 RX 方向旋轉的基本概念。

實際上六支致動器的伸縮量不會只由前後兩組決定，而是取決於六支致動器各自的安裝位置、上下連接點及旋轉中心。

同樣地，如果要產生 RZ：

某些致動器伸長，另一些致動器縮短，透過六支致動器的協調，讓平台在水平面上旋轉。

所以你可以先建立這個理解：

| 目標運動   | 六支致動器的協調結果   |
| ------ | ------------ |
| Z 上升   | 整體伸長         |
| Z 下降   | 整體縮短         |
| RX 旋轉  | 依幾何位置分配不同伸縮量 |
| RY 旋轉  | 依幾何位置分配不同伸縮量 |
| RZ 旋轉  | 依幾何位置分配不同伸縮量 |
| XYZ 平移 | 六支致動器共同改變長度  |

這裡最重要的是：**同一支致動器可能同時參與 Z、RX、RY 等多種運動，不是每一支致動器只負責一個軸。**

---

# 6. 六足平台到底怎麼知道六支致動器該伸縮多少？

這就進入運動控制的核心了。

假設 STM32 或 Motion Controller 收到一個目標：

```text
X  = 0.1 mm
Y  = 0.2 mm
Z  = 0.5 mm

RX = 0.01°
RY = 0.02°
RZ = 0.00°
```

它不能直接把：

* 第 1 支致動器設為 X。
* 第 2 支致動器設為 Y。
* 第 3 支致動器設為 Z。

因為六支致動器都參與整個平台的運動。

控制系統必須進行一種計算，稱為：

**Inverse Kinematics（逆向運動學）**

意思是：

已知上平台想要到達的位置與角度，反過來計算六支致動器各自需要多長。

例如：

| 致動器 |   原始長度 |     新目標長度 |
| --- | -----: | --------: |
| L1  | 100 mm | 100.12 mm |
| L2  | 100 mm | 100.08 mm |
| L3  | 100 mm | 100.15 mm |
| L4  | 100 mm |  99.94 mm |
| L5  | 100 mm |  99.89 mm |
| L6  | 100 mm |  99.96 mm |

以上數值只是示意，並非 GMT GSS-100-D 的實際運動數據。

接著：

1. 控制器計算六支致動器的目標長度。
2. 六支致動器各自執行伸縮。
3. 各致動器的位置回授提供實際長度資訊。
4. 控制系統持續監控六支致動器的運動。
5. 六支致動器共同讓上平台到達指定姿態。

這跟你原本熟悉的 XYZ 三軸控制不同。

XYZ 堆疊式平台比較容易把每個軸的位移直接對應到 X、Y、Z；六足平台則必須先做幾何計算，才能得到六支致動器的目標長度。

---

# 7. GMT 為什麼要使用六足平台？

這就連結到我們這套課程最終要學習的光纖耦合與矽光子。

![Image](https://images.openai.com/static-rsc-4/1facA9U_MKkjVDNZKRCokoM2LkklZHzvNGtrpiF9XFLSHPuTc1yl_9upJfM9UjzSO54zgRvJ4JnwdRHhM0MComY71Oj63UeOopSLjj5DUlgqNDK0HSN0-ylEY8inyMatmYqZIyf3Gv39GrTSQC7e0NsflHa_qjvkc76o_oalGgr0NoE9OZiC05Rynkyt0E8a?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iAH0EV-Rbni1e5DaK1iPMDzUHQJPZZpPFzOCqEJ9ZmtgHx2NWPeijdR0v7Fpy6P40eLBpt8ckUGKUIPpPHiIPK-CR2sFS3sKU4_GUabg-UR9PTLbY_2xcVPveHUr8QhFQMkcF-zl-9SlZBx1Hlm_zTX1fQDEnWoq4lum4dKoqSZ8L5fOwZqHKrI1JLfi2nZ1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/jIVngAXbYQ9PxRyRU_IdL6c9HGfgeFvMe6Y2I8LphsIyqk_ovEbLIQKcrOaM2Yw3y32jp8j9VO-U-1Npv39_6nnhpy0y2k7trIQDb2HQXy3_kEesWBN4af_Ut41_EoGNuK0JsBBPF-xy8qM1REQ1Fpa6P_IInpwN1dgd5bnvcl9kgxn9gVutD5_n3eP1TU_q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/rYIZ7sH7lcRKTUtP8DpKZhk7leUlR8SyjmiFuO9bN9keF7B2QW71k5eFDYNbmF863PHT6A22FUXzsbeelUplytujaCz3YZktZw8rN2AzrUyVSGUHLoaKyHRMfyKKLva6cXc4t7o_4zuzHZ3IXbDXJUtqlKe-8HboT12MXTgBYn8iN-9WzyXamZ5cO7es4oWm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Gb-zmcaJ9W6uDCD13HJPmZP1UAYMFymjIOromN2GpGMHxysbFPWTDHHQCyHsIrOclX_hXp9ZpeHots79lnjXewA7NAynJ9so2fi0mWHlp64_6DctGx3aqzbeM2TA238pY9ngz_kzzAySK9QXj29N4g57UkoebeCwW474iFB86FNW17FVhDamd9Bq27hi4WcV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8MynDX_KiqFC777yybxxiLN6euCEzuNHBIq5S6CEnGEc9cYt0eAzONTFhT4ILWObg1F0D5RQVU6hwYz0g24xUJcFqHwffgX-NdaKGSJdxwhOZ6CMMcMEJ8temVZ7QdnU4nLK5Y3Ozp9rhR4nzz7CS9wL5yiZEfohJPlUXKRgW5ousbtKl7LkiP0f4vpRUNBR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4EaJU-fQDCXDBCnK5eLZ6alfxeY5h-R78iuaQL4z1VJ00Ml9yV8hKoPJURISfn1uZvAQZaxZ1M8VumW3hfgHrl5xd1kWiDnG03trZHgg-2pX4XG6brex3G0cf_BVAkTNqHvnRu2PJv3NZFEoUNCEtnjddB6etNGdpFzvTnqPqkhvH80aPGrq7d87DzlLS8f7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/f1CM4ujIniZUkwf3xjTv_s7SRlzS6NPkkH48yguYOBuMy_uq53nHOZClMACYSuoy-SI61eXAoBE56NpVdFj-sDZa93K7xkd006DE9gcxwBQQi7TcZUweDmryHCd7mGjtxgyUfJkX2OqG4-dxOlfrUVMbIqajLNNz8YkEGO9sJQPDsujjV0UXDol81yi926DV?purpose=fullsize)

假設我們要把一條光纖對準矽光子晶片上的光波導。

光纖的位置與角度都非常重要。

如果只有 XYZ：

* 可以調整光纖的位置。
* 但無法完整調整光纖的三個旋轉角度。

如果使用 XYZ + RX/RY/RZ：

* 可以調整光纖的左右位置。
* 可以調整光纖的前後位置。
* 可以調整光纖的高度。
* 可以調整光纖的傾斜角度。
* 可以調整光纖的水平旋轉角度。

六足平台就能提供這六種自由度。

而且它採用並聯結構，六支致動器共同支撐上平台，與傳統六軸堆疊式平台相比，在某些設計中可以具有較高的結構剛性、較低的移動質量及較好的動態性能；實際優劣仍取決於設計和規格。

GMT 的 [GSS-100-D 六足平台官方資料](https://www.gmtglobalinc.com/?utm_source=chatgpt.com) 將光纖光學耦合及矽光子波導對位列為應用方向。

---

# 8. 今天先建立一個重要觀念

把你已經學過的 XYZ 滑台與六足平台放在一起比較：

| 比較項目 | XYZ 堆疊式平台   | 六足平台          |
| ---- | ----------- | ------------- |
| 結構   | 三個直線軸逐層堆疊   | 六支致動器連接上下平台   |
| 致動器  | 通常每軸各自控制    | 六支共同控制        |
| 直線運動 | 各軸沿固定方向移動   | 六支致動器協調伸縮     |
| 旋轉運動 | 通常需要額外旋轉軸   | 六支致動器共同實現     |
| 控制方式 | 各軸座標控制與多軸協調 | 需要逆向運動學計算     |
| 機械結構 | 串聯式（Serial） | 並聯式（Parallel） |

**最重要的差異：**

XYZ 堆疊平台是「一個軸帶著另一個軸移動」。

六足平台是「六支致動器一起決定上平台的位置與角度」。

---

## 課後練習：確認你是否真的理解六足平台

這次先不考複雜的數學，只確認機械運作。

**問題 1：**

六足平台的六支致動器，如果全部伸長相同距離，在適當的對稱結構及姿態下，上平台最可能產生什麼運動？

A. 向左移動
B. 整體向上移動
C. 向前傾斜

**問題 2：**

如果要讓六足平台向前傾斜，為什麼不能讓六支致動器全部伸長相同距離？

A. 因為全部伸長相同距離，通常只會讓平台整體升高，而不會產生傾斜。
B. 因為六足平台只能上下移動。
C. 因為六支致動器不能同時運作。

**問題 3：**

六足平台的第 1 支致動器，是不是專門負責 X 軸？

A. 是，每支致動器各自負責一個軸。
B. 不是，六支致動器必須協調運動，共同產生 XYZ 和 RX/RY/RZ。

你先回答這三題。下一步我會帶你從**六足平台的實際機械結構圖**出發，逐一辨認底座、上平台、六支致動器、上下球面接頭，以及每支致動器內部可能使用的傳動機構。這樣你就能把它和熟悉的 XYZ 滑台真正連結起來。


---

這一課我們先不談控制程式，也不急著談逆向運動學。我想先讓你真正看懂六足平台的機械結構。

我們以 GMT 的 **GSS100D 六自由度史都華平台（Stewart Platform）**作為主要案例。

先建立一個目標：上完這一課，你應該能夠看著六足平台的照片，指出哪裡是底座、哪裡是上平台、六支致動器在哪裡，以及它們如何讓上平台移動和傾斜。

# 一、先看六足平台的整體結構

![Image](https://images.openai.com/static-rsc-4/vTD6Qn7O8uaoVd6BYqZTkoIhflf0HXA1CgL8P7g0wMljaFxBZyjGfPFIxDIGzb8_n3DgYlGWmAbs_2rdQmA5GFg7ajkB_VbcjUNh4q6mm5yTjHZ5rW6XKQPj1I-aprHTeS_f3Oyw-5M4a7z53bvZALvysaJP3OafQMBtBDWOQsiLClbpFiwZL1DJX9YKQO-T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-LbpTHVW2TJoi04ARanEcVx1ETiUn8pEMlCn86pg-QDcl_nKB1R5egZNK_ZKVhCl_szqHDZ0Qsyslm83x60L7i69LMtNoJuvApymrh2TXdgzxuGk3qArKPMvrA950wRuPg5t-2w2-rsKEaIh2qranE4zv4Jlbf_2no7YjiJG2cK612HTrkHJ3zql0vPljKHu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/1t_HTgb3vmVMprNm5gZj1R1KaN3SJMaxOkhdlYQ1myF_MEVdOhTvQKjyaPvcI22vCZ0h8doofMvk7pv_Q6SYtroxWriIvPvd8IJS4VqW15sEMqWOMjGloi8tDbb77_v6d7EGMFH6lchCsJz41MPZtiKlyab82-pZiOb7SMMAClpPgcDEwbjyFYASVb8evzvO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8_mlHQtIdMJdLb9QT2hSYrVwDdY0qsdMsTBoWZ4fu2dm1Wevljv00Sd7gbNw5FQwFFgV_ZoXeuqzjbo_VUaU0aJUNTrSUznFCby-DLK9J_HPtTXM1boZ8UyFS3PMwI_iVx30smGpBNX-jbZtFnTYj0eYo8UiwyNG3YMSS9rz3RyBx3FUOg93Nnbu9MT5nsGk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8GpQsLVXSsdbyWp8Qe4nmY07PyU-Gq_lXJ10pK2bxecmNZXD1BJIp4jahnAoWBzLroqBD_EudNkLdpL4RmPPXJJ24U5NoBp21qiLwSC9ALCZObbPpBOFXPM4550zlJ5Gzwq-jWfzBPAA_VN3CiNaTZWd8ZmpqzMY3I3fJSo-PbrSVpPxy_q9qvXoq8CNhH0q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/E1LMGoBh-jfGvSZZvBAn5Y0b6ZwoZFLwT7Vo4ygF06fqhlyF0ubdY2BL8dTYQPiMvIbP87vdzgtzHnH8-Y4Z5wZR9Q2PjoIwbBQDneuc72MJu2kzOf4YO5sfcTQblzvVlVfkeKwuQ2EBCZ5PBeSKo2WFU1plsdrzbJMifmQccd9OL5On5qEBB5XAiWFYzGh6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7kPd4w3GIV9tEeUnj3fEG95bnwHMtEsxKE8NPFVMb_8E_JCyoaaSvq2t_kSQRrhZ8XQGMdzVKcV20883e2CtLgXLPUWGykuJg3upFv2OiXMkxg4zn6k-RjyonROLJ7PXxAs_QYhMJf5Q_eJCozAvxSks9uK1TE8E6dcQ4fWlJkXqaTPEIK7Yh-aes4be6G4z?purpose=fullsize)

先不要管它裡面有什麼馬達、螺桿。

請先把它想成三個部分：

1. **底座（Base Platform）**：固定在設備機架上，不會跟著運動。
2. **上平台（Top Platform）**：安裝光纖夾具、晶片載台或其他工作物，會移動。
3. **六支致動器（Actuators）**：連接底座與上平台，負責改變兩個平台之間的距離。

這裡有一個很重要的觀念：

**一般 XYZ 滑台是把一個移動平台放在滑軌上；六足平台則是讓上平台懸置在六支可伸縮的致動器上。**

換句話說，上平台不是放在底座上面滑動，而是由六支致動器共同支撐。

---

# 二、六支致動器究竟安裝在哪裡？

這是理解六足平台最重要的一步。

請看下面這種結構示意：

![Image](https://images.openai.com/static-rsc-4/RePzWWHAL7btWLOkarUJmGlb68rqokeGsXR4znxD4VJvX2bmuq-JDlDVP8IM2I-Hb9hHfQQIWX7fhtqH_jS3G0OnLHhqLaqulsAd-KGp3igJeC4Y1ATOx0oxmMr9fKw_XuDBVY9jGYWtD_1PQTIj1w-0SAPMQ--F_Nz33JLfldif1BjErbOVwmFM-X8kKvz4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GY-48XPa85IoAltnZX0tokMSKkWmZvCSyvRbSK_4KMfAcwJ_2KyHntdgWZubzrqOQdUIbLdd5KUfmcAWaFrUfEJTHEu5Vm_kYqbhU2cS_Fwo0GtckHkLZlJSX6bQ4vNbjekQLF1fB5bBq0IcbZQZ3y0AMvCoYG1HB4HdtglqkJaTuEdFq-yXtxefEPajphPy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9cHOVKoMb7KzeqK4iFKfmq2GEuiRDCtOy810-6pOCwd4dW9vK4RAs5yM7FU4ZecTQBezmMIdNKttCp3V8lMBUu7I79ChI6BL1wz2xO3du_KpSU1uao0p6cej6oXFPEhIDCDG5mzhVyE1cTB2xzF9liGsib2SxwnyuTe4Ockeu7K1EGK8WtPWV0WGsaCTHthh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4Os3Mgoa68JB29yG7Bl_Zj6ny9-FFr2iAwu5DNZLR9xymydAAEluJOFORXs1YdNf2RHyFKZ3xFi5l4eC1kycOYf_7-xyjn-cIeHDEJSYvi0cGFUdI_H7e6cy_Eheot6gxyqhjueS-AGeiLGmbS_oqkSa0MUstZX91hUHo7cnlPpXerx_MeGWQOvORc05hm-l?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VmwPFOEGyEj7IyT5r3oIRisu2QcsIcJ9CNmZrCXSxABdPkcmEBXK3uwQn0yDUK6NG20VIbx4QcWkUMLQE-MvpSFPfcgY2VG3zfio3ET3nQWP920fF_2XcuvWlfj1i5jaGLqdtLui1LgW0RDFF7kPpYnqmc8pmLP0D9ybfGQL-11J6pB25XKQHfLHeVSubMXL?purpose=fullsize)

我們把每一支致動器想成一根可以伸縮的金屬桿。

每支致動器有兩個連接端：

* 下端：連接底座。
* 上端：連接上平台。

例如：

| 致動器 | 下端       | 上端        |
| --- | -------- | --------- |
| L1  | 底座連接點 B1 | 上平台連接點 P1 |
| L2  | 底座連接點 B2 | 上平台連接點 P2 |
| L3  | 底座連接點 B3 | 上平台連接點 P3 |
| L4  | 底座連接點 B4 | 上平台連接點 P4 |
| L5  | 底座連接點 B5 | 上平台連接點 P5 |
| L6  | 底座連接點 B6 | 上平台連接點 P6 |

這六支致動器並不是全部垂直排列。

它們通常會以傾斜角度安裝，並且分布在平台周圍。

為什麼？

因為如果六支致動器全部垂直排列，就很難透過它們的長度變化產生所需要的水平位移與旋轉。

透過不同的安裝位置與傾斜角度，六支致動器才能共同控制上平台的六個自由度。

### 你可以先想像一張桌子

一般桌子有四支固定長度的桌腳。

現在假設：

* 桌腳可以伸長。
* 桌腳可以縮短。
* 桌腳與桌面之間的連接處可以改變角度。

如果六支桌腳能夠獨立伸縮，就能改變桌面的高度與姿態。

六足平台就是利用這個概念。

---

# 三、每支致動器內部究竟有什麼？

現在我們把其中一支致動器單獨拿出來看。

![Image](https://images.openai.com/static-rsc-4/fIKUcS13oiyXTL8XjX892cstkkGgUY_5HjetT2P71anzc3wTiUQnEvl1G0sO1N3DCO2imJrA3p0mGVPtqQnuB-PdfWuzNaG-ggWE10J-wMGMHGWLBXduPMGnRQHSeefCSG7Pj7LITA_hR_mF3P1pRzc16obLYFfScNd5LJt4_aMlRe4RS6Bk89tttwJDEkix?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HC9-CiJ_RPNhc1zwIqh9TwmYM8Mj9cAgJJVqobPr9CCuzpkIVjcQEh6IzHe3B_5MV3QfmAMIBbU73ZiVlQVeoMPBaYxLV0nfjpb9nHeT0WH7bEw9sU5r7kP1tmn6ubps6rssW4ryM3shtmpdP8L_m_3fNu3IRsyGYalPkIHw_Op45gg-CLFw8BuGFAoCCuXl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/THqyQ20K37XLCgML3_eezp825QX-pnWoLTcX_7ZQFakvIk3sF0LNT2UVlL8uW5C63Hw3gMEXLsu3p6LMbnhJPML0x0HZJmoHsufftRBahMCLqCbBBgRH9fyYKAqyenwKGGwIFEUwWxZmBS8kvXMMJbOP8aLpOANynvv9-TwYH0DGo406yyynGXKJCfzl1QFX?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/apRq5nxoPt-XJY914YCLanwZ0wTAob-ZRzbL4k_xEIll8jsZn0rN122mx42V_vpXbYu7A6W5uAB98M5i4pFX5EhwZ9js6cVCyIugP0NLfrGIlygsBQYTFak6f-_r3w1KxPTew-SBhf4Uqh4piLKjYgQhzIdKneI6BkPfzkcuaeT2BFLecJ1jeABJtEfoKF0T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-HRXayU8Zbiw4AHb0IeRZVd7vnQ7yZXS0mx_BXm2pPDLSV8bsc5cy-lXiYUmz5QEPVRQUP3YXL7gg4YSW73XHZk3k7FmAFOIopTRAIm1L56JdreY-UdJG84W_itBdFkVqFIP5DoXnEzYRrrXZLq9wSkndFj9VN_YDEj1Trp1fsjOzR9z39m4318itP1X08Is?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/y8VvJyHqsWVVbEZOc9EVDk1Nmk1iFcqA_dQioCLJh3F1KY_EP96F9t2MsFGnz_g9eP9_Wv9rs18HtZwHj1d5D5t3Ovsy1xmIEejUEk6tTlIBjPcJ3HbgXTOD6UNbFOl2Z2V3GyEHGSMj5z6PLbKKR_Nx5gQfDrAGgsje6JcS9osztT72YszUwGwaq5iTMeCC?purpose=fullsize)

你會發現，它其實跟你已經學過的 XYZ 滑台很接近。

一種常見的電動線性致動器可能包含：

| 元件              | 功能                |
| --------------- | ----------------- |
| 馬達 Motor        | 提供旋轉動力            |
| 編碼器 Encoder     | 量測馬達或致動器的位置       |
| 聯軸器 Coupling    | 傳遞馬達扭力（若採用此結構）    |
| 滾珠螺桿 Ball Screw | 把旋轉運動轉成直線運動       |
| 螺帽 Ball Nut     | 隨螺桿旋轉而產生直線位移      |
| 導引機構 Guide      | 約束內部移動件的運動方向      |
| 伸縮桿 Rod         | 將致動器的伸縮動作傳遞到外部連接點 |

例如，馬達旋轉：

1. 馬達帶動螺桿旋轉。
2. 螺帽沿著螺桿移動。
3. 螺帽帶動伸縮桿。
4. 致動器整體有效長度改變。

這跟你熟悉的滑台很像。

**最大的不同是：滑台的螺帽推動平台沿固定導軌移動；六足平台的螺帽改變致動器長度，讓整個上平台改變位置。**

這裡要先特別說明：這是六足平台常見的電動致動器構造，並不是已經確認 GSS100D 每支致動器內部都採用這一整套零件。GMT 公開的 GSS100D 產品資料目前主要說明平台運動性能及控制架構，沒有完整公開每支致動器的內部剖面。

---

# 四、為什麼致動器兩端需要特殊接頭？

這個問題非常重要。

假設致動器下端固定在底座，上端固定在上平台。

當上平台傾斜時，致動器的角度也會改變。

如果致動器兩端都用一般螺絲剛性鎖死，就可能造成：

* 致動器彎曲。
* 機構卡住。
* 產生額外側向力量。
* 機械結構受到不必要的應力。

因此，六足平台通常需要能夠允許角度變化的關節。

![Image](https://images.openai.com/static-rsc-4/GY-48XPa85IoAltnZX0tokMSKkWmZvCSyvRbSK_4KMfAcwJ_2KyHntdgWZubzrqOQdUIbLdd5KUfmcAWaFrUfEJTHEu5Vm_kYqbhU2cS_Fwo0GtckHkLZlJSX6bQ4vNbjekQLF1fB5bBq0IcbZQZ3y0AMvCoYG1HB4HdtglqkJaTuEdFq-yXtxefEPajphPy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/3S-j30rGWrCkPHTjYmamg01mc7N5zntIAW_y3WBClU7vfB8LAlZa0IscUT_3L_3SAjbwhmm9z20XUDOThr-iY8pYG5YUDmoauU3n1wR4lkjHqxDIR7GRKRK4p0MEXwfHZCR3avqM_bP_Qo0hDg4ea6ZFPHWiMgqbz7-Cli-Wh5J_5KOY1TeKFddeIDaKPErN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0tb6GpbLBl7PHCP6PvreN5zVsAD33IPy6hg6A8mjb4Qe1Zvn-2ZriWTshvpLOSm-ffy1N-V72vu1F3qZJMGRU7E52yM2ICMi8ESVy_EELr7fftVwnMmxmg3ZyBlVz_bRD2Mbx7WPrCtfu83HN2zigiFoEX_8dV66lytYBuYpCE9aiTUmiuPwx_fArjIR5IxV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/269QOHSjNN9T5xejAzHx0NTFsJvZUI7FhIF7Tv50B3IS3E-2uJn53BjpMkEl8ibApSsMwnJ6p-Fek5QuLFq-ek_Ut3bVPOXwKA-ySv2DH7jM-wf9SZ4vmzCQJIBEP5r8VDfhABavwPGthIccMTAmhvXFklst_FGPRBVHMKQCRfnmyxzrUkyqL07rNXz6-CoK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tN3tyb5utrg3eemtifGbZkJPYAAN02KiC6XSVNrjh2oXoDgii6IXpMLRfw5W10JE5P8swsQ6JjEfv-AzMODY6pfsqWXsjxyCSH0iZUTBjKt14bwMm-93EQ5yw_FJ0ZgVe25gAquANhN_sXOJBIwV3F3QccdMZoB7qQM-sQH54wYQcuKKE9_8HCyjzXCznhbI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ROQQTBedaOmWLb2oNfh1Gfkyl29GipZkRrZF9rWwgfB7NuEdX71haNipfPdN_lr0TB0NLlep_NNflr9r73Kic4yf3zCKcCbUvbn8l9reS9CbNKTpBW6jPjYFgGg4vYXGYtZskNTf8XwMZXGe83RPNwMG8fJKYT6LKwcct7-KlRo0MInWFVcFNSWIYnHOGzGq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qok6gWQyZtcGVmvwL1Rmjn39muiyiPIQdf-Rb62s3v0LgwA4GmeXXHPePqiNVOPlKFoORtysvAA6ntFbAx3P0aXt2DCYrf41LT3MQkqi2I8NYQQlrFSjPp9WCi_YZAi4o4rtkXBGVSCANnfalhhJHWaV-2OkQ77ia5td-EpyUOrPsk7GgKASqBYbgykPPxLr?purpose=fullsize)

常見的關節形式包括：

**1. 球面接頭（Spherical Joint）**

允許連接桿在一定角度範圍內改變方向。

**2. 萬向接頭（Universal Joint）**

允許連接桿繞兩個方向改變角度。

它們的共同目的就是：

讓致動器可以改變長度，也可以隨著上平台運動改變方向。

你可以把它想像成自己的手臂：

* 手臂代表致動器。
* 肩膀和手腕的關節代表兩端接頭。
* 手臂可以伸縮，關節可以改變角度。

這樣才能讓六支致動器在不同姿態下共同支撐上平台。

---

# 五、現在來看六足平台真正的運動方式

這部分我分成三個動作。

## 動作一：上平台整體上升

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VcpAFMTck13nMChmQLHaoYc_eaJdXZum4OuydZmKzAH-72x1gKnAB_ChOVRjOtN7cM3b5LCoNtDAS4FXGvl4WA2JA10n6JSHvfYraNQvQ4fbc2BkP8aOC-v1joRP8qhUHNISXiSWzBc6bYEnkdIspkOEaIrFtYEiC1O1gHBwlNPsXQyBVdfs3B8oX-Se69bl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2wP9Lq9XiOfDxyxTfHIytraLvwPwUSj6soMKGMczdIsCowZjc43nybNIWVdYXtPGX04yrDKripoT241l7RcIbgULV3m2ngj4q3rYGqYjc0BJwNyRZLamlROQLFWpl4isqQgdkMesvbq8-a3QSvlTMIbj1MsLgw5HNM_2xEBnKuWQUp5JAIpnZOiPUsYUm19r?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Q5wTBfcYA9puxOIFYnVo9Vdd1oZEBZxJpHKZ63JSfiRN-fua5dINejj8lQNuL0xe1t8mzFPyJ8HnZ3tec33NU7jjffMEXrtx32Syi8oXPhMWoioKlApMM1Bh-VdNiXcTPl6gondc9zygQw619xja3M2aMS34PWJxpxGgL_0T2BOFKBerhSmMwUhPGaNg-pLl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sfd8zTyvgLsviuZ_DS8fMhx1pWIAu3mxzNGrlOmaVr836qlZFjSDnIp_5if4dwoXRKm2ib6h1tr-moe0nHWQtAJ_tGD_y54PFJOEPnsPVECHy-MOLlsFxMoshsQDKnK0kaDISylnJ5WjmO86A6WQzUUITexKUig4wIxA9ws4Cw9HRBD28IyuF4DaJCOqPTAj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IEfpd-9eFLYy0NOFE29Xqj9bJ5v74ShJZmyj38aIkvkt_gN_TeweTVvbE5en5pmXEcybND-Ma5q5Z2SzFpmFit0g7mTUv3LR3IB6Zj7sTXalZVSy5CKvRIufdYE4rkKxrswkuTOp5IK5VjwwlzLjJQrCmL8Nnw9LabrmJeYeYS75pU6fTHKcn0veKG3n-28I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

假設六支致動器原本都處於某個基準長度。

控制器命令它們配合伸長，使上平台保持水平並向上移動。

結果：

* 上平台升高。
* 上平台仍然保持水平。
* 不會刻意產生傾斜。

注意：在對稱且適當配置的結構中，純 Z 運動可以由六支致動器以對稱方式協調完成；但不能把所有六支致動器的伸長量一律視為完全相同，實際數值取決於安裝幾何。

## 動作二：上平台向前傾斜

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4Os3Mgoa68JB29yG7Bl_Zj6ny9-FFr2iAwu5DNZLR9xymydAAEluJOFORXs1YdNf2RHyFKZ3xFi5l4eC1kycOYf_7-xyjn-cIeHDEJSYvi0cGFUdI_H7e6cy_Eheot6gxyqhjueS-AGeiLGmbS_oqkSa0MUstZX91hUHo7cnlPpXerx_MeGWQOvORc05hm-l?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0hvJb5Ch-wDo8DjIvmrywk5nEOfAo2WK-WpHnoPg2Q0Nf7wXX-e4Erh6XvREKIqEMDC4RzDukHLwEXCZmtwpWt5ro7zPOMM10-kDya5LTnimuv44-63komQWfctc95oABrfCCY7dx_sfzv6spxyhqpHKJicIcthmbM8g8SzkimB8G4ShTunMqEAAs4SIsZUW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/P45d50lRkBDM86gfX_hog2dAQTZ25W4fSoNGkbo4idT0sVL3kfmSJmi7e7FkUaIeZnHQwptP-W2bnGlKu3SGgbzqSpr1YXZcFGhsBumq7DTTn4wLQjPLGRuVBIRQdBD1n_N4AKlysZTlyXMYLXRnASUC0kHskgavQHpKW1hlnYKKwQqI0TxNre2xXEX2tGyL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sfd8zTyvgLsviuZ_DS8fMhx1pWIAu3mxzNGrlOmaVr836qlZFjSDnIp_5if4dwoXRKm2ib6h1tr-moe0nHWQtAJ_tGD_y54PFJOEPnsPVECHy-MOLlsFxMoshsQDKnK0kaDISylnJ5WjmO86A6WQzUUITexKUig4wIxA9ws4Cw9HRBD28IyuF4DaJCOqPTAj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/agZzfWVtMpYn48OSSHwg95DWKohkAhyc1LUeuadOV36o_nd0xLFMiXMWLdY74LvzQ0sUlQ8HfAjm1BuLnlbec20yVl9I3JS3vDtMfsd_gGwCtYIqEN8c1r6e-7lBLsw2m1-rml1JnYxvgzrTeo5dn2V9dtZYG9znHkSb-e5_e-dXzgs0PPZ8RVkpDHICNBwa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dPswECGhd2uvYLA0DIMTqvbOcp-3zewnRdjjy_mV2T7CZ-HVegyBvX4RWIIZQvR-e3B0CLUBJCN1wuxP4TeZmuglcAI19bdH94CkfsMgJ7xf0zf7kuU1tz4ot_cWn6QogA8clwsrOSeJRPQz_RumMsZx0TjxbCbZOs8dqth87S9LXoQqHv5NP1Leduzn1z9Y?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ETD0wAY933Nt7pXTJuIyrkxUgo5fYR2Wlul094V526TgdVmIQNHKIoH85Qyr8S5fTHZNtCMQTpsCBx4hP6x1MEi3FSvaYfv0Wy5pCNCtTc6A_Xtw7jHaIoCulxSVjgImGJqSOxe6sF5ZmodHwnYM8qM5nkkeBzOWMMzD_KiG1YOFF2OwWs0fpKRo-XzIVgf9?purpose=fullsize)

這次我們不要求上平台升高，而是希望它向前傾斜。

因此：

* 某些致動器伸長。
* 某些致動器縮短。
* 另外一些致動器可能只改變少量長度。

六支致動器的長度變化組合，讓上平台繞指定軸旋轉。

這就是 RY 或 RX 旋轉的基本原理。

## 動作三：上平台向右平移

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Sfd8zTyvgLsviuZ_DS8fMhx1pWIAu3mxzNGrlOmaVr836qlZFjSDnIp_5if4dwoXRKm2ib6h1tr-moe0nHWQtAJ_tGD_y54PFJOEPnsPVECHy-MOLlsFxMoshsQDKnK0kaDISylnJ5WjmO86A6WQzUUITexKUig4wIxA9ws4Cw9HRBD28IyuF4DaJCOqPTAj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7kPd4w3GIV9tEeUnj3fEG95bnwHMtEsxKE8NPFVMb_8E_JCyoaaSvq2t_kSQRrhZ8XQGMdzVKcV20883e2CtLgXLPUWGykuJg3upFv2OiXMkxg4zn6k-RjyonROLJ7PXxAs_QYhMJf5Q_eJCozAvxSks9uK1TE8E6dcQ4fWlJkXqaTPEIK7Yh-aes4be6G4z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9cHOVKoMb7KzeqK4iFKfmq2GEuiRDCtOy810-6pOCwd4dW9vK4RAs5yM7FU4ZecTQBezmMIdNKttCp3V8lMBUu7I79ChI6BL1wz2xO3du_KpSU1uao0p6cej6oXFPEhIDCDG5mzhVyE1cTB2xzF9liGsib2SxwnyuTe4Ockeu7K1EGK8WtPWV0WGsaCTHthh?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IEfpd-9eFLYy0NOFE29Xqj9bJ5v74ShJZmyj38aIkvkt_gN_TeweTVvbE5en5pmXEcybND-Ma5q5Z2SzFpmFit0g7mTUv3LR3IB6Zj7sTXalZVSy5CKvRIufdYE4rkKxrswkuTOp5IK5VjwwlzLjJQrCmL8Nnw9LabrmJeYeYS75pU6fTHKcn0veKG3n-28I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Q5wTBfcYA9puxOIFYnVo9Vdd1oZEBZxJpHKZ63JSfiRN-fua5dINejj8lQNuL0xe1t8mzFPyJ8HnZ3tec33NU7jjffMEXrtx32Syi8oXPhMWoioKlApMM1Bh-VdNiXcTPl6gondc9zygQw619xja3M2aMS34PWJxpxGgL_0T2BOFKBerhSmMwUhPGaNg-pLl?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/P45d50lRkBDM86gfX_hog2dAQTZ25W4fSoNGkbo4idT0sVL3kfmSJmi7e7FkUaIeZnHQwptP-W2bnGlKu3SGgbzqSpr1YXZcFGhsBumq7DTTn4wLQjPLGRuVBIRQdBD1n_N4AKlysZTlyXMYLXRnASUC0kHskgavQHpKW1hlnYKKwQqI0TxNre2xXEX2tGyL?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oTZr0mpVOnwrbxhy1fEim4yK3wBWhPmCNbDvjYig1O_Ks_P1h0uwgKr88MNaXguh7EuQ4emQrm2ie7_hbjle_bfrK7uqmFQHYuG4CbgTmnHjrjhxwG8qVypi7lRxF0s8sddJyXoVWnJ-5Hr-Q_OrDZGz-igre3K6uBoEsTQoOyfdvOFW_rAhhz1uw_NAvshH?purpose=fullsize)

現在我們希望上平台保持原本水平姿態，但是整體向右移動。

這時候：

* 有些致動器需要伸長。
* 有些致動器需要縮短。
* 六支致動器共同配合。

最後，上平台向右移動，但不需要產生旋轉。

這個動作很值得注意：

**六支致動器不只負責上下移動，也能共同產生水平位移。**

---

# 六、六足平台和 XYZ 滑台最根本的差異

我們把兩者放在一起比較。

![Image](https://images.openai.com/static-rsc-4/A-8IeFKHKOFp0CC_9jmeCXy8fD0Ux0E4deyFAvHEb6dIjW8OiO-PZu4p6NYdUvRpBXS0hxV7BUKxoqgLI-vqSxW6mGJMzS8eLt2jq3lOZNL9pC9douoZNN-VQ64xzXNw3ZWIUvLRmCBukLh2Gbu0riK9CyKr-_3ppsbLPg7li2juTitbo5V4jSd9914G5t3H?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0q3-VtoKHWBoqjUo9gG1Vs-oJfP2_BEcDadTRf4jM2CT0uRONFqPesIpgtKYttIUxRuwWTFN7BAeJX2R40fSrDgLpTSF-agRjp8ZbKVAk34nU6TVKS2TNLmcmnRWGD2UxWkh1VUHEkhII5LloTOE_UTUP1zXOdTrSmaM6ibbjH1ex-rmG1PVIUOmWGUnHMhJ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mmiAANzfhw2Nrjcif6exPHrGJ4ffnocyOxpdXxSveVE-9Tkwh_aucrP2lsMR9eaVtRuUpe55XtA5pfABtwxURsDdYhRMgnBjmXcOq7D3pGqAbWRB3Nf_Xxg2eQjWFRNTv_uFVMO7L-yDTwuJAirUncSogwmOT7yE5X78pkQCyyRRMWcQLojOgD7nlNplRyP3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Go6B4tIq-wT8k505OgumfUvZwvs36KWKE4RqPFmtNEgBkWlsD_jitsmkTIq0I-CXBHwKXmV-b44AVLdQCzRnv7vW9tlc1nt4g-g0ph6zzLBFWYSWr-SU0hbPrrvso8jG_cQ0pOWZv5mu1Jp2Lv8UZ-xfDCDDW-I3wQI_adwVH6GuOUGMVQRNAz8STAa4DupH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vZkOuBjNEk8vPzAwq_Wj4MKORSLYLwDLIm7YHu1PaqpQatevOovrkDf0yRVYRVu7Y1RshFWwxXwA8WVV2zFYwn22HHr5e0_atcPmNHgMfQkibGMWxVymYFzWXkSdep9BtYjTe3GtW1u6i-UZGWEb8ggKTgg9XK0hmK1K8ADf3NX-4IJn7FJNWsfg3UYUHnk0?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5KeEsS-_QvMLw8FqYIXEt1rFvE2icFWW6lgPKjUSulBGHmj61FteLkcGXZZbA51xg6ns__rRs0LrKBUd7_foaO6Sw7Hc_drlzVdFHEdRZ1HzJWFwpKgUQcD-qwPgqP9-7yuTkmCSlKgsR0rZFEtsPlcqHFAyBkkoFc04pKVl5JmRgNw1FcsWJAUXDPPOn4_f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/aI5uqTa3_UjS6LncPZ3IGO2UMfVlVo-LlQtya-FkHeR1g9y_HGkx5uXwdZEI4lJXBwp1Uqf5kmunvOXpc5lNsXFim0LueJng5t6ueuIxVCZf7PVs8EHeHs64Ptmlk3syQxr8W029FfOCdw_mxgAcnWrg4zhIYsPV1STTg7QpASvykePMQza5xC2BEXULcSs4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SXKeLhLyORa6-J9OvQAb_nCvYm_JIOtBM7WzOP2GQVs-zV5OU2Ots9oxHq_uil6b5b-c-Pw-tf5jFP415xuXWj3PPGNgyFDbQ-JochyJshbPOa9FZnfR6GryTliBb2AMFSdm1UccdmnSN18mhUZrxzHCA52scPYYDLsAG1rkT3_Y2OlYReY5rbekHgERvo2c?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XPAc45b9Wh2QcygudHaCcGahANqoGoNCY-pyeERar7-crgta-boLKeY0JUwlJRTC1svANhJu56iQjgK3qO4BCZ3BJ1hewWXuNxd1gTdZIx2BXQzAk8-3y8_tBPNKc3Y3V4lPq9o8-iPPetUVmFCZcUR_Ow6QLTWZ0OPk3rirJ7V0NDFRUQcj4gl9rLyrqfDn?purpose=fullsize)

| 比較項目    | XYZ 堆疊滑台  | 六足平台         |
| ------- | --------- | ------------ |
| 機械結構    | 三個軸依序堆疊   | 六支致動器連接上下平台  |
| 馬達數量    | 通常三組      | 六組致動器        |
| 每個馬達的工作 | 負責一個直線軸   | 共同決定上平台位置與角度 |
| 水平移動    | X、Y 軸直接移動 | 六支致動器協調伸縮    |
| 垂直移動    | Z 軸直接移動   | 六支致動器協調伸縮    |
| 旋轉      | 需要額外旋轉機構  | 六支致動器協調伸縮    |
| 控制計算    | 軸座標與多軸協調  | 需要逆向運動學      |

你應該已經發現：

XYZ 滑台的 X 馬達很容易對應到 X 方向。

但是六足平台的 L1 並不是 X 軸，L2 也不是 Y 軸。

**L1～L6 是六支致動器的編號，不是六個運動方向。**

這是六足平台最重要的觀念。

---

# 七、實際 GMT GSS100D 的資料

我們來看官方公布的規格，讓你知道這不是只有理論上的機構。

![Image](https://images.openai.com/static-rsc-4/-LbpTHVW2TJoi04ARanEcVx1ETiUn8pEMlCn86pg-QDcl_nKB1R5egZNK_ZKVhCl_szqHDZ0Qsyslm83x60L7i69LMtNoJuvApymrh2TXdgzxuGk3qArKPMvrA950wRuPg5t-2w2-rsKEaIh2qranE4zv4Jlbf_2no7YjiJG2cK612HTrkHJ3zql0vPljKHu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4p9ZqBxXWub5NJ8l-llSBQALCSgSb6aaD-yjYCzWiFJHgrmwcIhebmUqqcwk3svB59ofByfzAoW-zHKeflboxeb1LW6rrv5Ij9pQ5n3pEC-SFO_WDpv33GGWqQCgOmqZ8VbdFBcpW4XCSCiWvnrETqusUyhyc6_EWU48J7Cyg1EskHQRUllyZNfCPPb9TC50?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RL8S1C0ntdV3Hd69xNudZOWAyMJKcu5ch7HqelU6PFZ79kYFVNj4rFiMd2mo9kAJseJvh5HW9WJqt08D-OS-5_VhIF84pxvdST3IDLn4ApySnzf1I6Bbn7G0FHd7ObOu8z4no5UPJONi2NQLpacuOBFzG5GHNffTpU8FRMmpZENzLuuFNGH1PaTTygaKiMtd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/4EEzIx7nieo_sr4tGrkkf4lcgk4aMd9WIU1TA4F6ns8FbFnkWhoWhgvO4U9NnN3bykqW0HEy5zTzjEPrbH4RZTQ4DOKmBfPMZ1fDR9jyQ2knJniXP5VhGy2SPHFuBJxVIEFfz2ADDZoxIPErpKmUnrtx60ruoZ2gTsjsSeM9mo0eOIEjMq7CLKoN4rBZ6mca?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0jv8R3DC8OBm83PHrgMP8ONG18y9iYnloelP5dJR-HnsV5ggOZ6a9khfFHmXB6xF3XxGywPmg_WiwZLangc46AHmndO2R_Q_mOfszpKVOmJjz4Qznt-Ixq0OqnIug0e0Am-CiLpJEJ59_fp3ZoyT63LGahoghk8hA_LDkoKsAl70-_8ujX3ooPe__akdM6bg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UgrlS7JVcWXSuHoxp6_7PC92reGgp9Fo_2GR3R0a2W4MohgQiDK-PlhsI91CcsWGnh0_IaxzTN0AQ5U3PySLWeIvrdKeleILpwlEhwSOjk_a2VQMCjjfHJlQPCn8__Al8Dcuk2au6litbbjYbmgDg3ogmJPQNYJUYlTMLSFAQh1M0LzCAGcJ3etikqgx0KLo?purpose=fullsize)

GMT 官方資料列出的 GSS100D 特性包括：

| 項目    |         官方規格 |
| ----- | -----------: |
| 控制自由度 |            6 |
| X 行程  |       ±16 mm |
| Y 行程  |       ±16 mm |
| Z 行程  |      ±6.5 mm |
| θX 角度 |         ±10° |
| θY 角度 |         ±10° |
| θZ 角度 |         ±20° |
| 控制方式  | EtherCAT 全閉環 |
| 控制軸數  |            6 |
| 上平台直徑 |       100 mm |
| 底座直徑  |       136 mm |

資料來源：[GMT GSS100D 官方產品頁](https://www.gmtglobalinc.com/%E5%8F%B2%E9%83%BD%E8%8F%AF%E5%B9%B3%E5%8F%B0/gmt-gss100d-high-precision-hexapod-platform?utm_source=chatgpt.com)、[GMT GSS100D 英文產品頁](https://www.gmtglobalinc.com/en/stewart-platform/gmt-gss100d-high-precision-hexapod-platform?utm_source=chatgpt.com)。

這裡有一個值得你注意的地方：

GMT GSS100D 使用 EtherCAT 全閉環控制，官方資料列出專用控制器、六軸控制，以及編碼器介面。

這表示它不只是六支機械致動器，還包含完整的運動控制系統。

我們後續會把這部分與你熟悉的 STM32、EtherCAT、Encoder、Feedback 連結起來。

---

# 八、這一課最重要的觀念整理

請先記住以下四件事：

1. 六足平台由固定底座、移動上平台及六支可伸縮致動器組成。
2. 每支致動器的兩端都必須有能適應角度變化的連接結構。
3. 六支致動器各自改變長度，透過協調運動決定上平台的 XYZ 位置及 RX/RY/RZ 姿態。
4. 六支致動器並不是各自對應一個運動軸，而是共同控制六個自由度。

---

## 課後練習：這次確認你是否看懂機械結構

**問題 1**

六足平台的六支致動器，下端通常連接在哪裡？

A. 上平台
B. 固定底座
C. 馬達編碼器

**問題 2**

上平台向前傾斜時，為什麼六支致動器需要特殊關節？

A. 因為致動器除了改變長度，也必須允許連接角度改變。
B. 因為特殊關節可以讓馬達轉速提高。
C. 因為特殊關節可以取代編碼器。

**問題 3**

六足平台的 L1 致動器是不是專門負責 X 軸？

A. 是，L1 就是 X 軸。
B. 不是，六支致動器共同決定上平台的六個自由度。

你先回答這三題。

下一課我們再做一件很重要的事：**把六足平台的一支致動器拆解成馬達、編碼器、螺桿、螺帽、伸縮桿和接頭，逐一比較它與你已經熟悉的 XYZ 滑台有什麼相同與不同。**


---

很好，這三題全部答對。

今天我們就進一步拆解六足平台的其中一支致動器。這一課很重要，因為你會發現：**六足平台並不是完全陌生的新機構，它其實是把你已經學過的直線運動機構，改造成六支可以改變長度、角度也可以變化的機械支柱。**

今天分成五個部分：

1. 一支致動器的外觀與組成。
2. 致動器內部如何伸長、縮短。
3. 致動器兩端如何連接底座與上平台。
4. 六支致動器如何共同支撐上平台。
5. 把六足平台與你熟悉的 XYZ 滑台做一次完整比較。

# 第一部分：先把六足平台的一支腳拆出來

先看一般六足平台的機械結構。

![Image](https://images.openai.com/static-rsc-4/6hwkbFBBS7hKwHtRGz9tY4GdTPQIvu3woWJLUu4vTmMaDoJLISRZmJLSCOZKiY8x2KXVPOxCYwmkEoG4dB-iTGirHmumG64t3SVHS-ZQApgvP0vDu9lGO8DHYU4pwBx7TflCZkc1G3gJ3tY7UriyG4fXujtuMieZCknABR_km4oen7xBgYmEgQJVR3MsoXYz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qBU81wCFyTwxa16rMj-0gcTDhNCG6bqqGh1Uk9mU-7SgaaDdLonkWftUuJNbcy9YyO3HqwdwnwyrS5ilxV-2MO7vUv-nk-udWmDvBST3xp279KmwAUbu-tVHgWUtcu8YYN97i6mMYncYxNqpdvBjRKb1CweyQwYdsYYvqH6RrXpsLYlY5JPIRBfhxFWxeQ5R?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vTD6Qn7O8uaoVd6BYqZTkoIhflf0HXA1CgL8P7g0wMljaFxBZyjGfPFIxDIGzb8_n3DgYlGWmAbs_2rdQmA5GFg7ajkB_VbcjUNh4q6mm5yTjHZ5rW6XKQPj1I-aprHTeS_f3Oyw-5M4a7z53bvZALvysaJP3OafQMBtBDWOQsiLClbpFiwZL1DJX9YKQO-T?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SQpuFzIvbzm53qGhSfsRCTHoXCa2b7I54rfTzXk2N1j6lTsG2QvvvC2tDT9LAC0TCd8d26Xclldr4dtXq4e3TXVDYt_9EIhCCbq7HDNqXjdRj-gJ3b50U4LiSCTS2GrDqFOvS2sunOJ4Ex7f6R7P5MiuUMRHFOkBepztim0lWQi6TlpjSZiyHY7CPzj2hOmq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Eari3CpAxGEYYoz9sfAKmiQawAIMm2SlCJaI1MoPotmW-cSz7FCFCoJh5Gl6QaHoIAyy80cD9UJtlnDlDQKHmHkaGjNRy4O8TJnmt44lr4_1lRbzl3xB1Ar44CuN4hK2L5EVz7Kz2zCA_52iPfYAAxU8nQgGw3YD6MNpFq0N7GsUH3uLJ185YlGSdmDb3Usc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yokVD4tUDVARBSxdBpfHOqXpOeh-0a7lg0y0PubxkwDvprcV-CwflnMeR3kQpAsWX9FaTXKS0c6uWAqgxnr8guEfP2e_bc19LV5D2JEuX_TH_RbT3yg9RM8EiaKAi3_dEAvtYppd5EsDzZ8YHt2i00y8Zj_TbJQc9w4fas-f-a0-3XF-06Gc_ADeJhRpHMwy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-v7g0R03pIngQ5Gf4WgUJUCYbyTHex7Lsk_rNcAaxjZMJF6z8jbbFvc2gM8H4GHrmCZ2g6oPUTY6GbefcJ8d7Rjmeg5EVY6KQZfDcOTjlyaN8zBStKssVO1A1Hvi30J-Du0sR2RffixSGFYZ1y5S8fQnFgBkArqVJJGyy7iPzKA4fPEecxoyIisvh52U8-87?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/y1Krk891ixp1D_rtrfgVjAWid-nmQitZVXBiWRivrkUqolgvsHAB0MiN1Ih6SVDEwwK7mdMLihg9bu0IfLLxZNviUWu5Noz2oDPR_KonMIsz_xFeR3Ou4Rwq_ap08DTicKO9KI1tegA1IHsHXs7kppkFlSiJvWHBx_3IPMwY4FQAMR_AM5qJWlY7M0zZCi6R?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oQz3mlTE2YhVVrk-dlrWyvfvQTas_tjNMZ05zKtiDfdJEZr-0qCOIpG0yIFtf0YgxGfF8d_CY8vTcUGlh0P_15X1Upedyr5Xi09jF2bMNOynXh41TSEh9N382I-GgR1V3PSZP8XxpTFbOWsB_lbjOXNrcQzsMIAzU7jN7G7irJKsJl2QTP64c0xGhPKdpdtU?purpose=fullsize)

請先把一支致動器想像成一根可以伸縮的支柱。

它有三個主要部分：

* 上端關節：連接上平台。
* 致動器本體：內部具有產生伸縮運動的機構。
* 下端關節：連接固定底座。

這裡要注意：

**致動器不是單純一根金屬桿，而是一個可以主動改變長度的機械裝置。**

它與一般 XYZ 滑台最大的不同，就是它不需要固定在一條水平或垂直的直線滑軌上，而是可以隨著上平台的姿態改變安裝角度。

---

# 第二部分：致動器內部到底有哪些元件？

現在我們把致動器剖開來看。

以下是一種常見的電動滾珠螺桿式線性致動器，不代表 GMT GSS100D 的實際內部剖面。

![Image](https://images.openai.com/static-rsc-4/DmPLGQ4oFG7RxP72Z5ETQ8buKir8-LWADhraovXqshmvgOUYGBCZELR0f1qJ2qz47Slkvx1dn3LRJfUnOUBTpDmr0_iEiOO4gUWu0LAkcWoujcgz4IG3HgJ2Eswtu0-fIPRV7kH0m05COckKPws0Dvbct_5SHCQp8yyK0YO_ft_Vo7zS32zIwLzseT-n4ghO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/y8VvJyHqsWVVbEZOc9EVDk1Nmk1iFcqA_dQioCLJh3F1KY_EP96F9t2MsFGnz_g9eP9_Wv9rs18HtZwHj1d5D5t3Ovsy1xmIEejUEk6tTlIBjPcJ3HbgXTOD6UNbFOl2Z2V3GyEHGSMj5z6PLbKKR_Nx5gQfDrAGgsje6JcS9osztT72YszUwGwaq5iTMeCC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LuWR829YUgaave7u_qgmK4scxPEX_LJ35RLBfMZVuJts6m9RaOKqk0PNqM8Fj7vUATCdl7_cC3KkuqWUJ2kHvxhy6xHJIaKzABdhw50KKju7qDxD-vlWoJtCvmj-nz-TlOgvgh9zZPKHFYI-73qTBzlv4ZRE2c5U4V1F7nNI7gKinqOrOq07LJmhwCoI2Qz7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LXUy83v7lW3AbA4YjDo5xKFhJNtHn6NK9dSr-b2KwOYwl7aB0G8p0FTfB9aOBr4mBnDTY8n1LzP3OJarI1mfAmf-55gkJnJgmr7DISg6JSQfptQtVUgYaLZA6KruydVp_BK9TLoTq2zOTwUoZkD-vwmLPU0cuUfG6oKgyJu4ClSRCgbHxxFXBL3VHZkNjdGy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/QUOS4pU_rFjtscs7gLrsdGyWTSrR_8_vvu5-u5TX6nTzuz6TIIn9SfJNOs-nhfFSTos6i4_4Ox8iKd7Bpj51z3hvCjd1bhY7YBNPY-PGQBYAwUmYotIP9xLYswgUoQU2LropwEJvJHePkCV012Tx-dtBn4mxuXEcQk1kn9pIe-4IcD6SWtameu3uLarvSTqw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GucUVyx6Se2xBIkCJb3NWLRCuWDvcnKQY5D_1_6UxjdzsgTpWC9Yw_U3ytydOflzIQaT0yxgKoEW9cG6NqWLVQal1yr30jL7sp2EV61qorN4m9wdZKQzi6FTiSiC54bfrugMsvUTQCMvB2ONZFf8Oqo8m7xWayjjkqKmHJY7mRUa9JTMTdc2-12UNYGxVh0Q?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/UzxnQ1B61VCOhBSv6BoXkWiD5ugpCInOsbPbxwyHY8VQa374RGkdsTFbKhM8LEoHmT1XBSRUXpqVFj13D1BF4atk4dS2PIsFDVihwPaX4ug5bzh0LEng6vVTsodhmy60GQQJwudw8eavyfAVnXQdpT-1X-AHGWF1ziEOCjpKR62AUGrpK2k2aRb2byuAaUNm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/sX3lfn8TvPx6md-Yltof8dZb-iCmC0Jhx-N4PF5_vARvf5mOht26Rhj5EOvKM00_TgBiKboLLNebwmFGZnrHBWavwQRwTwzWMAZeIeN35oCD5UgdswgFQpwrkDC-bQi3RZNOAgPPWrU6uyu0lkQwlYdODoIjH_CtLgaO1OKIhG-L9lILFVNKSFpccBQYBToK?purpose=fullsize)

我們從馬達開始，一個元件一個元件認識。

### 1. 馬達（Motor）

提供旋轉動力。

可能使用伺服馬達或其他適合的驅動馬達，依產品設計而定。

### 2. 編碼器（Encoder）

量測馬達或致動器的運動位置。

例如馬達旋轉了多少角度、轉了多少圈。

如果系統另外有直接量測致動器伸縮位置的感測器，也可以取得更直接的長度資訊。

### 3. 聯軸器（Coupling）

如果馬達與螺桿是分開的，可能使用聯軸器傳遞扭力。

但有些線性致動器會將馬達與螺桿整合設計，不一定有獨立聯軸器。

### 4. 滾珠螺桿（Ball Screw）

把馬達的旋轉運動轉換成直線運動。

例如導程 1 mm：

螺桿旋轉一圈，螺帽理論上移動 1 mm。

### 5. 螺帽（Ball Nut）

隨著螺桿旋轉而沿軸向移動。

它會帶動連接的伸縮桿或移動構件。

### 6. 伸縮桿（Rod）

將內部螺帽的直線運動傳遞到外部連接端。

它是致動器對外輸出直線運動的機械部分。

### 7. 外殼與導引機構

外殼保護內部零件。

導引機構則限制內部移動件的方向，避免伸縮桿受到不必要的側向偏移。

## 這裡做一次比較

| 你熟悉的 XYZ 滑台 | 線性致動器     |
| ----------- | --------- |
| 馬達          | 馬達        |
| 編碼器         | 編碼器       |
| 聯軸器（視設計）    | 聯軸器（視設計）  |
| 滾珠螺桿        | 滾珠螺桿（若採用） |
| 螺帽          | 螺帽（若採用）   |
| 直線滑軌        | 內部導引機構    |
| 移動平台        | 伸縮桿       |
| 底座          | 致動器外殼     |

你會發現，它們其實有很多共同點。

但是，兩者的用途不同：

* XYZ 滑台：讓工作平台沿固定方向移動。
* 線性致動器：改變兩個機械連接點之間的距離。

---

# 第三部分：致動器兩端的關節，究竟有什麼作用？

這是六足平台很特殊的地方。

先看球面接頭。

![Image](https://images.openai.com/static-rsc-4/fArR89pSioFORUdOGQ2UekkfK8qjIq1tcEAFTjsizTZtR0V42vlhsU64xebAeUt7kfoUXk2cW3GL1nQx1V6Gpne7PcttJW4RbR4pSnEdbsbxUVI1shr_rVAWHlsuxD4-p4SOKWnp2CNaUx62cn7tTjriStwJcXm5pz1SVkdftcxNJ7viuxt7PnEFA2onz6hy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/269QOHSjNN9T5xejAzHx0NTFsJvZUI7FhIF7Tv50B3IS3E-2uJn53BjpMkEl8ibApSsMwnJ6p-Fek5QuLFq-ek_Ut3bVPOXwKA-ySv2DH7jM-wf9SZ4vmzCQJIBEP5r8VDfhABavwPGthIccMTAmhvXFklst_FGPRBVHMKQCRfnmyxzrUkyqL07rNXz6-CoK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/GY-48XPa85IoAltnZX0tokMSKkWmZvCSyvRbSK_4KMfAcwJ_2KyHntdgWZubzrqOQdUIbLdd5KUfmcAWaFrUfEJTHEu5Vm_kYqbhU2cS_Fwo0GtckHkLZlJSX6bQ4vNbjekQLF1fB5bBq0IcbZQZ3y0AMvCoYG1HB4HdtglqkJaTuEdFq-yXtxefEPajphPy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HO8Pk6rUdLPKOqjSr1CVYvrQMnMHckAHgXLyHUx9KBoC0mq9jz6RXzStReqrhT40iZMnJ_Lc1xHZEq0msoMjJZL7M7-eryti8Xkmwj6C5uBrC5KB0mXHTrFK7x2Ey1JAejlB22XMT39WTO7PwxpVZM5JiGx23uR77Z__fSwT2FALSzL38pDXCv4r-aTl5xuI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8TRzgIJ8gMn5MvskD7s1mDLSltVGGRURSI6GgO1aFUTZLqDIuy10QtOQWJa9ovMujVkI1aCAXZj9uJQXBt85vPiecktV3V0y2Ea8d1nr4qiTkao2F3YykQnmuu9lY5xUTHdUEKZObW1BPZd1ryvek80JrownXp5bUfGs0Weoz46MpRQH3I5gVPLZh8FJQUDQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qok6gWQyZtcGVmvwL1Rmjn39muiyiPIQdf-Rb62s3v0LgwA4GmeXXHPePqiNVOPlKFoORtysvAA6ntFbAx3P0aXt2DCYrf41LT3MQkqi2I8NYQQlrFSjPp9WCi_YZAi4o4rtkXBGVSCANnfalhhJHWaV-2OkQ77ia5td-EpyUOrPsk7GgKASqBYbgykPPxLr?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9gtga0N9byEpnFLZ61pRZAZsn4l6OsP4P053dgfEyDBB9gIICco48W0Oj0Q8X9EsahRb6UoFukFVo7RrJswWJN2kwvaDH-gNoZMnDg_mDaGRnbjXc6vg0-s86cby3tSxWTAe7Wsduk3hRePkq7BfrTqtZ9Ev_fY340ZnZt9-tkCl3ksJXRPc-B_mkfRPQQM_?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ROQQTBedaOmWLb2oNfh1Gfkyl29GipZkRrZF9rWwgfB7NuEdX71haNipfPdN_lr0TB0NLlep_NNflr9r73Kic4yf3zCKcCbUvbn8l9reS9CbNKTpBW6jPjYFgGg4vYXGYtZskNTf8XwMZXGe83RPNwMG8fJKYT6LKwcct7-KlRo0MInWFVcFNSWIYnHOGzGq?purpose=fullsize)

你可以把球面接頭想像成一個可以多方向轉動的關節。

假設：

* 致動器下端固定在底座。
* 致動器上端固定在上平台。

當上平台傾斜時，致動器也會跟著改變方向。

例如原本致動器是：

```text
       上平台
    ───────────
         |
         |
         |
    ───────────
       固定底座
```

上平台傾斜後，致動器可能變成：

```text
       上平台
    ───────────
       /
      /
     /
    ───────────
       固定底座
```

這時候，兩端的關節就必須允許致動器改變角度。

否則，致動器會受到額外的彎曲力量。

這也是為什麼六足平台的連接關節非常重要。

**致動器負責改變長度，關節負責允許方向改變。**

兩者的工作不同。

另外，實際六足平台可能使用球面接頭、萬向接頭或其他多自由度關節設計，不一定上下兩端都使用相同類型。

---

# 第四部分：六支致動器如何共同支撐上平台？

現在我們把六支致動器裝回去。

![Image](https://images.openai.com/static-rsc-4/EibQ9qmAq2hSv7PR3ANQo6JVVWzRxkeMPWc-q51u68pYiUTXLff4XSzHU7HHHlgSfbVPi7K-S_2a-zQQKHLywxkgfGG64JaSjVJjNXre0GQB8C1RhYrcmlziFp2iN2dJx-BZ1qA5BDEi_teiXIvCNYa_ZHgR8mlgif_O0zUovhUNFon2Gca2DgZPAkcRNq8X?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tPQawAXEpGV5Dikm3Qv7CAoAdxMB271D7g2k0CwShrEyIS4yWbIVybEpX6JA6vS9LP1RQ2vyP6CROT_Aq3jXZs8guU2lE-T1ErbMncbRe5JnpjPJrhRpSAP6GN5yo5LeH4-RSACEuz198J13MzqDJ8Zpkn4M6DO62fNDh-1n1hgq9vsttwlImEFtqkss-WAi?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/7kPd4w3GIV9tEeUnj3fEG95bnwHMtEsxKE8NPFVMb_8E_JCyoaaSvq2t_kSQRrhZ8XQGMdzVKcV20883e2CtLgXLPUWGykuJg3upFv2OiXMkxg4zn6k-RjyonROLJ7PXxAs_QYhMJf5Q_eJCozAvxSks9uK1TE8E6dcQ4fWlJkXqaTPEIK7Yh-aes4be6G4z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/agZzfWVtMpYn48OSSHwg95DWKohkAhyc1LUeuadOV36o_nd0xLFMiXMWLdY74LvzQ0sUlQ8HfAjm1BuLnlbec20yVl9I3JS3vDtMfsd_gGwCtYIqEN8c1r6e-7lBLsw2m1-rml1JnYxvgzrTeo5dn2V9dtZYG9znHkSb-e5_e-dXzgs0PPZ8RVkpDHICNBwa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mzmCneR7eetNptGcore6VarPRoGpOgau__V6lcVgdXpFhmLZHOxdMkKXJuQAbxa1tNm2KQMBwh1Am2GLLJDqQ883YlXXaKG7buLuvSR_TwRIh2chZokoVEw46TwHxHlHzdhC3fvr-SFzatrk5yQgGKD2l0vpPzCA3IO6AGxhk2fnoEkT1N1DP80oJemiEYLI?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8ENM21GHG8T159_YmnhHbnIdLKXHOLuuRno3ewI_q2IGdVZGL3KcvPZnocS3qa_3fgLy4Ywpba5PRnnj-i5qs3VadsTFyMKxZxP4i_nzt0aqe2n6QjkqB3wfTcGSAFhEEJTUjlvw2TU4DFigKKHpzpTiXrlSULlZP9cEJi_EInYKyEdxaiHq7vAKsCerebEt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OZE9qVUk4BNgOMC0IVMxde0GwN32DOz_UpVBZaQh2TsTp1oBFhvWGIDNlJHudVzt7_FnEQOnaEpj034d3wwAp9-WJb-msqKTB4TzT93F8oHNQV1iVSzLv0Nu3QiPo147lUxYpICjc6eq9IvdC5U9OkS_QGeX7vVey-k5J4OGKLJfUpPyw8n58n9OgAc_MIL3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nI_HgZxvOwPxFjR_tjiJkH1sW1T4cDt0HbHGDD1pJe0c3J7eVmyjU-1ug4buao-NwTJVSp-4iITuUw-iecDPYMZTjaDmA8-7SvaajSfoxFnohtj83dxULdrI_IM7MymeF5lY1iAI_tIi-u8La8dpkJyP8WINgnPUJPoLTRGLT9QMmQQC1lHvd_OUeOy0Z18l?purpose=fullsize)

這次請你特別注意兩件事。

**第一：六支致動器不是垂直排列。**

它們通常以不同方向傾斜，分布在底座與上平台周圍。

**第二：每支致動器的兩端連接位置不同。**

例如：

* L1 連接底座 B1 與上平台 P1。
* L2 連接底座 B2 與上平台 P2。
* L3 連接底座 B3 與上平台 P3。
* 依此類推。

這六組連接點的位置，是設計六足平台時非常重要的幾何條件。

為什麼？

因為它們決定了：

* 上平台能移動多少距離。
* 能夠傾斜多少角度。
* 每支致動器需要伸縮多少。
* 致動器會承受多少力量。

## 一個非常重要的觀念：致動器不只負責推，也可能負責拉。

例如上平台向上移動時：

某些致動器可能需要推動上平台，另一些則需要拉住上平台。

它們共同支撐上平台，而不是六支全部只負責向上推。

因此，六足平台的致動器、接頭及固定結構都必須能承受設計所需的拉力與壓力。

---

# 第五部分：實際看看 GMT GSS100D

![Image](https://images.openai.com/static-rsc-4/eQptT9jGDmtKku913SdLPm1zPvi31_K2uow_0xoNrce0YPnLlBk96ZLRTA1sk7jTS6L_lHh8N-e5-kCanjoInC00is5mx67sTXQBnKFIbmtAHajzEd3H6-mw_HslsGbKzv7EnhsJv_40c8MQvaSUVAqUmHAl8RksZzRpHG4253X2R5AYy8rj_DeUV4kmifC3?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/RL8S1C0ntdV3Hd69xNudZOWAyMJKcu5ch7HqelU6PFZ79kYFVNj4rFiMd2mo9kAJseJvh5HW9WJqt08D-OS-5_VhIF84pxvdST3IDLn4ApySnzf1I6Bbn7G0FHd7ObOu8z4no5UPJONi2NQLpacuOBFzG5GHNffTpU8FRMmpZENzLuuFNGH1PaTTygaKiMtd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0jv8R3DC8OBm83PHrgMP8ONG18y9iYnloelP5dJR-HnsV5ggOZ6a9khfFHmXB6xF3XxGywPmg_WiwZLangc46AHmndO2R_Q_mOfszpKVOmJjz4Qznt-Ixq0OqnIug0e0Am-CiLpJEJ59_fp3ZoyT63LGahoghk8hA_LDkoKsAl70-_8ujX3ooPe__akdM6bg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HcdqhT-8oEf0W7wYQmfnAX2U9SkQHnkRdQW4adYqlSbqYHpR8CpQhRvgLUo72n-Lvzr-fxSj2irlfXnhWU0sRyn5klRK4Y_WnPGQx2gqjYoF5txde4FRmmbwg_HoONuy7wpnp85I2fB6vKZPfYwpb9fs2csVj65Dp9aEXpwqSBnjDtFKXz3SPVtbl7Cnl5Bm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8KQa7aurKccH2b5_MhG0nyazvPNeViZW97eeZdqBBUMCegN_DINSRBhF5AVELCrMKMPty3lIxnrVw7drmue9qAR8x0yvohtuUrBOGS2SEzWowHMdRi24S3_ksifJvK6_Th1f51dx0x0MAi0YiMMcJJgePIPohYaOV3kfOBBfoxdlQ4_a9Fc0mnOaick9Tl6L?purpose=fullsize)

我們把剛才學到的結構套用到 GMT 產品上。

GMT 官方產品頁確認 GSS100D 是六自由度並聯式運動平台，採用 EtherCAT 全閉環控制，應用包括矽光子、FAU 主動對位及精密組裝。([高明鐵 GMT][1])

官方公布的運動範圍：

| 運動方向 |      範圍 |
| ---- | ------: |
| X    |  ±16 mm |
| Y    |  ±16 mm |
| Z    | ±6.5 mm |
| θX   |    ±10° |
| θY   |    ±10° |
| θZ   |    ±20° |

這些是整個上平台的運動範圍，不是單支致動器的伸縮行程。

這一點非常重要。

例如：

上平台要向 X 方向移動 1 mm，並不代表某一支致動器就伸長 1 mm。

而是控制系統根據六支致動器的安裝位置，計算出每支致動器應該改變多少長度。

另外，我們目前沒有足夠的官方剖面資料，能確認 GSS100D 內部究竟使用哪一種馬達、螺桿及關節配置。因此，前面介紹的是一般電動六足平台的典型機械原理，不能直接當作 GSS100D 的內部 BOM。

官方資料：[GMT GSS100D 英文產品頁](https://www.gmtglobalinc.com/en/stewart-platform/gmt-gss100d-high-precision-hexapod-platform?utm_source=chatgpt.com)

---

# 第六部分：從韌體角度看，這六支致動器怎麼運作？

這裡開始連結你熟悉的 STM32、EtherCAT 與編碼器。

假設使用者希望上平台移動到：

```text
X  = 1.0 mm
Y  = 0.5 mm
Z  = 0.2 mm

θX = 0.01°
θY = 0.02°
θZ = 0.00°
```

控制系統不能直接把這些數值分別交給 L1～L6。

它必須先計算六支致動器的目標長度。

這個過程稱為：

**Inverse Kinematics（逆向運動學）**

計算結果可能像這樣：

| 致動器 |       原始長度 |      新目標長度 |
| --- | ---------: | ---------: |
| L1  | 100.000 mm | 100.120 mm |
| L2  | 100.000 mm | 100.085 mm |
| L3  | 100.000 mm | 100.145 mm |
| L4  | 100.000 mm |  99.940 mm |
| L5  | 100.000 mm |  99.890 mm |
| L6  | 100.000 mm |  99.965 mm |

以上數值純粹是示意，不是 GSS100D 的實際數據。

接著控制系統就會：

1. 計算六支致動器的目標長度。
2. 將六組目標命令送給對應的驅動控制通道。
3. 各致動器執行伸縮。
4. 讀取位置回授。
5. 持續監控實際位置與目標位置的差異。

這裡你應該能連結到之前學過的 Encoder 與 Feedback：

**六足平台的六個致動器，各自有自己的運動位置資訊；控制系統再把六個致動器的狀態與整個上平台的目標姿態連結起來。**

至於 GSS100D 的逆向運動學究竟由專用控制器哪個軟體模組執行，以及各致動器回授的實際實作方式，還需要更深入的控制器資料才能確認。

---

# 今天最重要的總結

你可以把六足平台記成：

```text
固定底座
   │
   ├── 致動器 L1
   ├── 致動器 L2
   ├── 致動器 L3
   ├── 致動器 L4
   ├── 致動器 L5
   └── 致動器 L6
          │
       移動上平台
          │
      光纖 / 晶片 / 夾具
```

這只是結構簡圖，實際上六支致動器會分布在底座與上平台周圍，並以傾斜角度連接。

最重要的是：

* 馬達提供動力。
* 螺桿或其他傳動機構產生直線伸縮。
* 伸縮桿改變致動器長度。
* 球面或萬向接頭允許角度變化。
* 六支致動器共同決定上平台的位置與姿態。

---

## 課後練習

這次我想確認你是否真正理解致動器的機械作用。

**問題 1：**

一支電動線性致動器使用滾珠螺桿，馬達旋轉後，最直接的結果是什麼？

A. 整個上平台立即旋轉。
B. 螺帽產生直線位移，帶動伸縮桿改變長度。
C. 球面接頭開始旋轉。

**問題 2：**

六足平台上端的球面接頭主要負責什麼？

A. 提供馬達動力。
B. 量測致動器長度。
C. 允許致動器與上平台之間改變連接角度。

**問題 3：**

假設六支致動器中，L1 伸長 0.1 mm，是否代表上平台一定向 X 方向移動 0.1 mm？

A. 是，L1 就是 X 軸。
B. 不是，必須考慮其他五支致動器的運動，以及整體機械幾何配置。

你回答完，我們下一課就進入六足平台最有趣的部分：**逆向運動學究竟如何計算六支致動器的伸縮量？**

我會用實際幾何位置和簡單數字帶你計算，不會一開始就丟複雜數學公式。

[1]: https://www.gmtglobalinc.com/%E5%8F%B2%E9%83%BD%E8%8F%AF%E5%B9%B3%E5%8F%B0/gmt-gss100d-high-precision-hexapod-platform?utm_source=chatgpt.com "GMT GSS100D 六自由度精密史都華平台 | 高明鐵企業股份有限公司"


---

三題全部答對！六足平台的基本機械原理你已經掌握了。

我同意你的想法。六足平台的逆向運動學、致動器內部細節及複雜幾何計算，已經超出目前課程最主要的學習目標。我們先停在這裡，不再深入數學與控制演算法。

接下來回到原本規劃的課程進度。

## 目前課程進度

| 單元 | 學習內容             | 進度        |
| -- | ---------------- | --------- |
| 1  | 精密機械基礎元件         | 完成        |
| 2  | 傳動機構：滾珠螺桿、滑軌、聯軸器 | 完成        |
| 3  | 馬達、驅動器、編碼器與控制    | 已學習主要概念   |
| 4  | GMT 電動與手動滑台      | 已學習 CXS50 |
| 5  | 直驅、壓電及自動化模組      | 進行中       |
| 6  | 光學元件、雷射與 Camera  | 尚未開始      |
| 7  | 矽光子與光纖耦合         | 尚未開始      |
| 8  | GMT 精密對位設備整合     | 尚未開始      |

## 下一階段：直驅與壓電模組

接下來我們會認識兩種不同於傳統滾珠螺桿滑台的運動方式。

### 第一部分：直驅馬達（Direct Drive Motor）

主要學習：

* 直驅馬達與一般伺服馬達有什麼不同？
* 為什麼直驅馬達不需要滾珠螺桿？
* Rotary Direct Drive 與 Linear Direct Drive 有什麼差異？
* 編碼器如何量測直驅平台的位置？
* GMT 直驅產品通常用在哪些設備？

### 第二部分：壓電致動器（Piezo Actuator）

主要學習：

* 壓電材料如何產生微小位移？
* 為什麼不需要傳統馬達與滾珠螺桿？
* Nanometer 級定位是什麼概念？
* 壓電平台如何搭配位置感測器？
* GMT 壓電平台的實際應用。

### 第三部分：自動化模組

了解如何將滑台、馬達、感測器及控制器整合成自動化設備。

---

**我們的最終目標不變：**

讓你能看懂 GMT 的設備機構，理解它如何運動，並且從 STM32、C++、EtherCAT 韌體的角度，理解設備如何控制這些機構。

六足平台先告一段落。下一課正式開始：**直驅馬達（Direct Drive）——為什麼有些精密滑台不需要滾珠螺桿？**


---

# 第 5 單元：直驅馬達（Direct Drive）

今天先學一個核心問題：

**為什麼有些精密定位平台，不需要使用滾珠螺桿，也能讓平台精準移動？**

你已經熟悉傳統電動滑台的運動方式：

馬達 → 聯軸器 → 滾珠螺桿 → 螺帽 → 移動平台。

直驅馬達的概念，就是盡可能縮短這條傳動鏈，讓馬達直接對負載產生旋轉力或直線推力。

---

## 一、先比較傳統滑台與直驅滑台

### 1. 傳統滾珠螺桿滑台

![Image](https://images.openai.com/static-rsc-4/mEClov8EaUUV9UYP5OAKUeavyJj4Zhitk1iwtE43zyKdAWgKS2qmbrgTNHNGee_4dy3xHXlofvupDZ12hNx8U3mYWAh_8kvubLjH2XPUVzX91bBOUjN-GoeUV5NqwMr46UNiTgel6wFenB9Yi1Y5kh_CMUOVpV9b_5BD810AZmNR0aQ2YuXsSvW05V7z0Hup?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/H0EHImYuQMSPz2w5yK1WzudUttEUJQrPnhZHGm-3mBwHkV7W0Sk68fM029q6hO8k44hq9sdjR-pl3mEpm-q9T4WBzCndZQXYl1cTuErvsvaOcFkg2TH72sR9lv5gDeMQPNP4xOuBEksFztbnhBlewyrmsQG6wlcR5R_gcc-NUbRH4lEhthagz5U8exNrr6lK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EibxTVBuTsiE5-gSzQv1d59ZRDzYrtJnIVirL2NXGyU8-Su5BgLpuoNLsi0tnnn1mJ_RBSrPBPZu9V8ZUWsA70QCXvwMMDLeBArWWj8-BrljUqyqkeqipJuT3yYvS9stf2AxZtnsk_hHdlqOBMxW4naoGKy85lnoYYYMRrxQ7sRF58qoGJknxHtBVFIk_eYD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZaOnw3nKJ1oFKZ76o9ap4jJIiI3AwWNVF5ejDFgGAtKlFLfp1-SJExhBpcM76zYX05dxNJ06y2YvnLRq166j7Ho8t4Gs5ONxpeztKSjduOGHVHkBx94v3lObaNgeo92Lexuc6dTo2iPhHMRvyBl0x1601WJqVae0AeWaqit2gnAwOKneEhMdWfyt57mF5bLP?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kb4252ETSg5LEkuO1IbHGAcv6NUpKfSmueRvtB_dUyG6JqxGKre_zEYMQFR3MAzKSXCOACgY2Z0SRbq-QOINuzamSCykD5k4NtoN8RNRc9TJuDoy4zIMcee_oK4vEazRI8ispvhttLc78A_pZwQNPSt3JYwZUVlkLOzxmve4rrqkJiovlIA05Dvfcrr8j_qc?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fhR2CgYnYdz_dP0ot31uWhlQacq81fRbe4waM36d6hlQgV9BL8H44v-JAhS3BEWNwdWbH1h9NXKNB_cmcF3QPxjXWxEcxLCfQAlENglL8tcNADYcHm_gHx5m_RhyBj37THE5S3YgoLmvfnsT2NrkFlkHNPerAUgM9ZG4xXYwPXnbDESmUvJ-DeGlFdiNgDJW?purpose=fullsize)

傳統滑台的運動流程：

1. 馬達開始旋轉。
2. 聯軸器把馬達扭力傳給滾珠螺桿。
3. 螺桿旋轉，螺帽沿著螺桿移動。
4. 螺帽帶動平台。
5. 平台沿著線性滑軌移動。

這個架構有一個特點：

**馬達的旋轉運動，必須經過機械傳動機構，才能轉換成平台的直線運動。**

### 2. 直線直驅馬達（Linear Direct Drive）

![Image](https://images.openai.com/static-rsc-4/uouXmPwyb_qarR9uOIKHrMlsx6bver0MZEXAMEd98lZJStIV5muTc6PwSSim4gYFo_pqI5p9z-Yt-ZbLNV2PDJdhi564VAILofklNoyegdkj1UOcpx95Z-b73vvPdkbMqVfrhcbR3PbIPq13qt78xq5szrpqMfUDzIQFmDXPVzcskxTvEhH5Z6Rq2DSTGtXN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5Z_yg5EHz5Nbr2drB9eEcnyOmDLD7Bv1XhBroyqC2vVeYC5WVi6d6vehpDX8CES9XUkuotSEG0HQ-H3pU6b7W0KHd3Ej6fSNZSvmzA3H-5-KVc_aYaqosEPC5_aolHiuA1RYi1uziFGE4UI-Qlz03Ppx-LSgg0HpRGbsxqWLscO2p0Gs6_b86VUHNZhZvMec?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Lm1f6eMZf1eO4HR5e01GtS_46Hr1JmAodawLBCx5OqRjYTAgoo-GMhWKpzg_KaL74aBOMdOrHZhZx5pL8AuBLWPV_A2aTXZvWOSzUkGNX7XH11MynrlXArGge_AB7mKeY5NAgI9jeT1bHFixyEUzIfuPGg95gJZbh91X1iyM8HPDM1emhQERrCmSVjpt9BEV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lipHUSwEM973CO0luvJhj53fseV68e6XFUfESN1X54OdpBuJb5zQYowygHrM6mthJbF6SAMUamQaLIxE95_BtU_Y_SJyk6dUOWIi9YlVyT1Jcbcd4P1mC-_T7W32SEHqWNGBl5cM2fwMpYyCgI692HMghBuW5Pe7mXSA74c-4mzGdIZCHcD1tdo0HhClw68o?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tKjA9a8fxhIOmTcw0byezSa1MiJ6tQ5f4_9BJb7mULJF9fCB1DfKDUsNvXqcB7-PRF7msEkavS2AUalbWuXa4Io7kKSk4yoDVMu17_ih0fTpsu1zp1pyWhPOETOo4OPl-4q-Bg9ex2POy1Mn-NK4-wNQdAjaHAkBCogxBrGIFz-CMGiu0ALp2VVtuwUd9P84?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/F9c-WuKAsn2wiI0kMJgVxRjwS7WiRivz-D_tXihK53ul_GUbW-Xrf5iGZWbSY2LgSC_s9qtdYwcdxcT0A0F6BHvLhMkg-vcp5N5yNhjIc27H_Fl8Q1Nntqs_5rM-BQeWbAfA-sElOmoRO6Wdd5_j-QGaURMbhiI_fivQKblDN_AKuQDqc9ZRwyE9BQpiXVdx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0YcCAEI-4bYK3qmneDqqHIXvJkT460sHT9n_u8Uzb0ETUHlDwflnEJyIrJTQj3SN5haRK59uDx3QVpMg2x0S87K1s9bK_6JhnbLqXlAUFj_5kuBK3E37wi2-fGrA3_9lrtUldzeV4qNYTxVovsrCglkFKDYwOneuOimTq6hO998giHNwjERGEM-fKQDWn__I?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/6olGKZru5WdpSwVX32aUzDcyoQTx69PBt6LDH3B8AdHGpVrGGhAA6Tcm5ZiDt-pKqj1Pzv-dZOaIHJYhtkxBS76kZjj5_AMgye7Nxg2VKVcDpvo7uObjcwB38mVg6j0twI0dOiKxcF3iUuAeyyr01x7y-Fvdcc5WmfhjuO3ed9wnREU7WgT7J8usAz3rj6Pv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/xHz5B5TK6Al1FUiegGReg6ECH6q0hLzT-lcS9753_jvRIV1jXYBpGbv7PTeEKyx_-QIc-94YbzZ1MiMyb91BvqKvfb3l_cfcOBC32mmfSglkH6hTllxpxR1EpkDLpCOrBZyVAO7FeDfL1wbABy4yRh9XYfAeQYiacL_d_FBByCj8WwHNmClL6iDhr6qzjBNA?purpose=fullsize)

直線直驅馬達則不同：

1. 馬達的線圈與磁鐵產生電磁作用力。
2. 這個作用力直接推動移動部件。
3. 移動部件帶動平台沿著滑軌移動。

因此，它不需要：

* 滾珠螺桿
* 滾珠螺帽
* 馬達與螺桿之間的聯軸器

不過請注意：

**直驅不代表整個平台沒有機械零件。**

它仍然可能需要線性滑軌、軸承、編碼器、底座等零件。

---

## 二、線性直驅馬達的基本構造

先認識四個重要部分：

![Image](https://images.openai.com/static-rsc-4/5Z_yg5EHz5Nbr2drB9eEcnyOmDLD7Bv1XhBroyqC2vVeYC5WVi6d6vehpDX8CES9XUkuotSEG0HQ-H3pU6b7W0KHd3Ej6fSNZSvmzA3H-5-KVc_aYaqosEPC5_aolHiuA1RYi1uziFGE4UI-Qlz03Ppx-LSgg0HpRGbsxqWLscO2p0Gs6_b86VUHNZhZvMec?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/uouXmPwyb_qarR9uOIKHrMlsx6bver0MZEXAMEd98lZJStIV5muTc6PwSSim4gYFo_pqI5p9z-Yt-ZbLNV2PDJdhi564VAILofklNoyegdkj1UOcpx95Z-b73vvPdkbMqVfrhcbR3PbIPq13qt78xq5szrpqMfUDzIQFmDXPVzcskxTvEhH5Z6Rq2DSTGtXN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Lm1f6eMZf1eO4HR5e01GtS_46Hr1JmAodawLBCx5OqRjYTAgoo-GMhWKpzg_KaL74aBOMdOrHZhZx5pL8AuBLWPV_A2aTXZvWOSzUkGNX7XH11MynrlXArGge_AB7mKeY5NAgI9jeT1bHFixyEUzIfuPGg95gJZbh91X1iyM8HPDM1emhQERrCmSVjpt9BEV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/raGwbWdG66-rhL7_ovSARVGV1vxCSiwbE3XPASrm9vvahml2BpEDazFe5g0YXPjMRmieOIiq_HDp2JzeoKZqW3MArwRiVrXDpCbshPO0RTgeRvAibx50XZy7qqv3zk8Ii0E1Ui9ftagof0Xo7zDGKD9T7GA0OERDLAsizMC4VEkRBVqtnNItOm19UWEqfbhr?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/kjkbTsUgY_9lfc0OnJRIO9VB73DKCxO2-pEAo5X9xobg_Ezsvy9EPNEtwjIIY00mOByCOTkpBy02iwS7iYoptJ7CPnoOSbYk8ho9Bc01KDimHM7wU7LV3NzgpTceI_YMzDufjt9ByFb_34CoegDJXOeQIQ9EUIokPCoa-X-PqMBE5exFWVf66FXHHM5iDEeY?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/OvHNggLDw0QQ1xEbKVmvc8WfGPW6ytaa5ytxVORIOg-v7EucVCoLLBaTSXCx9Qo7qnC9c8nBVrOwwM0OGG1QyPR_CjsRhrckgpAwsyoZnpV7DZcdRYCgnsIA1LWG7I7PSRjLLEWDmRj7xBdNQxW1lRLEebXShauNT57IFSycD4XKEteLhT7e4MJKz1WYAlFR?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qi_jzwYmQKvd8gJH5unh6XXlzSJFI6uk12jRukum6PAPtGKmVwhf23QhR1Pf5GtGMOX49o_GRvLBQoEn9oVG-DG1G_96Rv_s7wS-utttuQta0kI13W2YMy2dW6DIV6eD-1cZqIJu7JkDhdHKkD9v4rb2023SrY_tVHbmxtXijNVpSe7r5tXXzctWImEJ9vId?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u2p0r7MIzHhmy8RnOKbEkbYKiusSZWsI6utVXTkUcaIt2sWx0gcRBafLY8FSCVIUKU3ZZAi8dKxaOub44yLTLMOIU1P9SEIdV5eg2btUTYqbBYmpHTczchKzNqnBhgkSIZTE2GiOZJ23zp3Bx8wLQzFCTEkQM4S9TQcdm81QT0ZiPxSfr2qz1mTWRYNMuEw1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lqvgxHDMMVL4N8azcKBlJD7mPWPFIH8GdQkaqkyMxN055cBwmPKsUg3ZDpvg56aRUfY6-H4FzUZ4sGJvu-z__BwJjzHh_sYS3mMC2wN_pX6VSH0JtGA1QAAbxXWYNYDi2PptvZ3ih26euCN9VXbM4kVGHTK_4vrbKevjh-eVNvHA-c5oeaQu39gLNlxkERL5?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Em88LZ9kz3EaXaNaHU0mg12IoEP3iylqTsxrQ3f8DgqQlLj1VMEhiciKNtMs0MEx3bqE3s8fSZ340OzdRzrGSBW_TqlVZr-mHi3z7nTC7r0mkXxbW6cPPXam5k3uXpNmkU6IF90o4X8NrvCnUmZDxw5Fc4jjIcB0-UX4pAUNseih8IPRIPzUy-AlyOqT19iv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/21ha-ECSDwIlC_7UGHKQLkv-vbxY1daMED9laa5JcMBLeJO1byDVxR2LzlYEFnix16MiaQ3XCnu2sOOSUK8X_K9D3WVE7vcVhIVjxXXAaXEUH5AIDtGgEh0uuEpUrg-NlL23Ipgs-w1X9H_o1AlL07pjpCU19umyIP4Ph9hsJyhYoxv149nnE8jxR64tcy-S?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oxrfGy2ihy0ur0YeogyP6YO7k3JFtp9SgSeUPuFDhpmZKF6zCiQux618DrEmEVwGmXOSfVCtrq9yoTgHULg2IE8gFddXTmPJq_DryopY6RYaxxWLlzfJaluVeYr1RMMGgrjdX_fQgFS4o8lngZ6RbvT0R_GNQF9wQPh9tMyKkD3B7WmszaiwusZ6KWzKzNWV?purpose=fullsize)

| 零件    | 英文             | 主要功能        |
| ----- | -------------- | ----------- |
| 磁軌    | Magnet Track   | 提供磁場        |
| 線圈動子  | Coil / Forcer  | 通電後產生電磁推力   |
| 線性滑軌  | Linear Guide   | 支撐並導引平台移動   |
| 線性編碼器 | Linear Encoder | 直接量測平台的直線位置 |

其中最重要的是前兩項。

### 磁軌（Magnet Track）

通常由排列好的永久磁鐵組成。

它提供磁場，讓線圈通電後可以產生電磁作用力。

### 線圈動子（Coil / Forcer）

線圈通過電流後，會在磁場中產生力量。

透過控制電流的大小與方向，可以控制推力的大小與方向。

你可以先把它理解成：

**線圈通電 → 產生推力 → 推動平台。**

這就是直線馬達最核心的原理。

### 線性編碼器（Linear Encoder）

這個元件你之前已經學過 Encoder。

不過這裡有一個重要差異：

* 傳統滾珠螺桿滑台：可以使用馬達端編碼器，再依螺桿導程推算平台位置。
* 線性直驅滑台：可以使用 Linear Encoder，直接量測平台實際移動位置。

例如：

STM32 下達目標位置 20 mm，線性編碼器回報平台目前位於 19.8 mm，控制系統就能知道還有 0.2 mm 的位置誤差。

這裡的編碼器仍然只是負責量測，真正的控制修正由運動控制系統與驅動器依照架構分工完成。

---

## 三、直驅有兩種常見形式，不要混淆

這裡要特別區分：

### A. Rotary Direct Drive（旋轉直驅）

![Image](https://images.openai.com/static-rsc-4/RrBGm03QneOu2FCxD5OW3yBIuGaFb-syGYNXWfIdtrwwT6LSRlZDxg1bVpIyfToeoljK-bG8Ao_Ab0FA62OePETU4plhWFo73mRXDNXFaGXYEApSfQPIDR1_BnaQxWBs7gSlgOpqRUNXikwrtPymW1nMyG06pHfivb25BMkuCQeKqqqLIe0XOffGExWY4G7k?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-vsNjFQHrjsVYaNafgRhr-85YCKN-CynnKmF5RFqQkOalwk3p9p2ArXOwBbj-_5wonnRsB9tkQ0iFUzWHxMJwa_4D48ZZDTFRV6e0shd6iNJIqy5eDihZrkEmn2xGKc97kvnw48w0KW1PYKM82iHf0Wqq8oR6TrcJrZXdFnIqCxrNlzoJOse8jX7JrvFeONx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Y4UJY2M_sj8G44BT9UROiFU-wPNKS790_RYzgMnK2ac-dHexcbeW6ViMb5fLrloO7VGrkyYcbTKhItdbHAcvYPyTBfSZFHUcW3B8UEz-eMi2zGO1bBIWpPjfBrDnHiXFaQ64fYMrb7csHAoJWZ7JXFJXXIIyioWZhTkfmjvxZTCcoAhI8XvKCDw_NpsDX2_f?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/VLsvf3A8Fx1e6m0ujJCpPaBwcm5juX20tafV57vRkJ8OL3vY_1ARev_T_JRP5KFo351v6o4LAJQcI_gMX6t_PGL1rB4ybSdFNQvtg0YO_PWuvX7qQV1fzW39bxCzqLbWkQx1JaCXcrggX3xRsw8J3WpN0BfJtQU8LLS6ukWX5cwkgCSWAzZm27QqzolEOpyG?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/gwZHIFCyaLx6agl_TvgfuMoLxRkbLWCBv5FZUbZwMNSv5zlcZQjcDUjevV7G6gC8-L7d47dxrwTydoIo18gOdUP6L3sdu1bsuxXmYXIH4tKZ2lbZbyOzXDC0VHJE7SKEDxkESqguQ5W-g2eno8ojNt7ulN1huw-EFhHkB4tUEstzV-brwxncwYLKW2Z2_sEy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oVhdF2NQGSeUbueis35voXZlJ4HJrhBE2U82ebNEhTqUsiF414vJsuq76LM_bcn7sSAMgJniUbxVSWrdAdfAa6Kb_Jr3kt-zcRDye1b8DidKnX6zugHB8nP0FdUxBTswvubVoCz6iDKI45txSQkaD7b8DSfTXGxUVENcK6xSosN9XQVRu4yGcqItckC6lVX4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/dpVBgW-Y5JJPpiqHAnuLVoYj1KsGZwN1N7bQ4XTEu2jdWhjx8a9aAEvrvyl8NX_74V1ii9_P83bCR0cCLTyuB8eLYmme1v1PHB-q_L4RK2gMhv7uz69WqQkIDYb8p6KXJY5_RwxOX7s7IKUZKYWui0N9baG_o7hUUFu3rS7rbD2Iq-TQ6xTx4ZnexzQ-Nzei?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Oo8wh0GiRDlm5SvOyv-ltHc4kMp3mm2z7Bnxtq4T9XGLyvp7MvZEFbCouiYbQ0rCMDw6cBZWx56jwcFIe8AkULNndTx8FRymLwGGakeoqOiE-ZmXfbnwWgRZv3_-uX2o_xHpF5SwFGhJmJPVSOUYzXA-wbPva6rpxBRRISRRd6-6Ve5VGOLsTN2sfDpMvCJj?purpose=fullsize)

這種馬達直接帶動旋轉平台。

例如：

傳統旋轉平台可能使用：

馬達 → 聯軸器 → 減速機構 → 旋轉平台。

旋轉直驅則可以直接讓馬達帶動旋轉平台，不需要中間的齒輪或減速機構。

適合需要高精度旋轉定位的應用。

### B. Linear Direct Drive（直線直驅）

這種馬達直接產生直線推力。

它不需要透過滾珠螺桿，把旋轉運動轉換成直線運動。

適合精密直線定位、高速移動及高動態響應的應用。

**簡單記住：**

| 類型                  | 馬達直接帶動什麼？ |
| ------------------- | --------- |
| Rotary Direct Drive | 旋轉平台      |
| Linear Direct Drive | 直線移動平台    |

---

## 四、直驅馬達有什麼優點與限制？

| 比較項目   | 滾珠螺桿滑台   | 直線直驅滑台       |
| ------ | -------- | ------------ |
| 動力傳遞   | 經過螺桿與螺帽  | 直接產生直線推力     |
| 機械傳動零件 | 較多       | 較少           |
| 傳動背隙   | 可能存在     | 沒有螺桿傳動背隙     |
| 高速運動   | 受螺桿等機構限制 | 通常較有優勢       |
| 加減速響應  | 受傳動機構影響  | 通常較有優勢       |
| 散熱需求   | 依馬達與負載而定 | 線圈可能需要特別注意散熱 |
| 成本     | 依規格而定    | 通常較高         |

直驅的優點：

1. 減少螺桿、齒輪等傳動零件。
2. 沒有這些傳動機構產生的背隙。
3. 傳動鏈較短，動態響應通常較好。
4. 適合高速、高加速度及精密定位。

但也有一些限制：

1. 成本通常較高。
2. 線圈發熱需要考慮。
3. 需要適當的編碼器與控制系統。
4. 沒有滾珠螺桿的機械自鎖效果，垂直軸要特別考慮煞車或防墜設計。

---

## 五、GMT 的直驅產品

我們回到你真正要認識的 GMT 產品。

GMT 官方網站將相關產品歸在「直驅模組系列」，目前列出的產品包括：

* ZR 馬達系列
* GLM 微型對位模組
* DD 馬達系列
* XYZ 三軸運動模組
* XYZT 四軸運動模組
* GLM-Z 軸微型平台系列

其中，GMT 官方直驅模組系列頁面說明，部分產品採用**音圈馬達（Voice Coil Motor）**，搭配阻尼器，沒有減速機構；該頁列出的系列行程為 8 mm，依型號負載為 1 kg 或 1.5 kg。這是該系列頁面所列的資訊，不代表所有 GMT 直驅產品都採用相同規格。([高明鐵 GMT][1])

另外，GMT 的 DD 馬達型錄說明，DD 馬達採直接連接負載的方式，不需要齒輪箱，並整合高解析度編碼器。這是**旋轉直驅**的例子，與前面介紹的線性直驅不同。([高明鐵 GMT][2])

你可以從以下官方頁面查看產品：

* [GMT 直驅模組系列](https://www.gmtglobalinc.com/%E7%9B%B4%E9%A9%85%E6%A8%A1%E7%B5%84%E7%B3%BB%E5%88%97?utm_source=chatgpt.com)
* [GMT 產品型錄下載](https://www.gmtglobalinc.com/download?utm_source=chatgpt.com)

---

## 六、本課練習

今天先確認你是否掌握最重要的概念，不急著深入電磁學。

**問題 1**

傳統滾珠螺桿滑台需要透過螺桿與螺帽，把馬達旋轉轉換成平台直線運動。

那麼，線性直驅馬達是如何讓平台移動的？

A. 馬達旋轉，再透過齒輪推動平台
B. 線圈與磁場產生電磁推力，直接推動平台
C. 編碼器產生力量，推動平台

**問題 2**

線性直驅平台不使用滾珠螺桿，是不是代表它也不需要線性滑軌？

A. 是，不需要任何滑軌
B. 不是，仍可能需要線性滑軌支撐與導引平台

**問題 3**

如果一個平台需要旋轉 360 度，應該優先認識哪一種直驅馬達？

A. Linear Direct Drive
B. Rotary Direct Drive

你先回答這三題，我會依照你的答案確認觀念。下一個小節再介紹 **音圈馬達（Voice Coil Motor）**，因為它正是 GMT 部分直驅模組採用的技術。

[1]: https://www.gmtglobalinc.com/%E7%9B%B4%E9%A9%85%E6%A8%A1%E7%B5%84%E7%B3%BB%E5%88%97?utm_source=chatgpt.com "直驅模組系列：馬達模組與多軸運動模組｜高明鐵 GMT"
[2]: https://www.gmtglobalinc.com/images/uploaded/Catalogue/DM-TW/DD%20Motor%20Series-1141104-en-p.pdf?utm_source=chatgpt.com "Direct Driver Motor"


---

三題全部正確！🎯

1. **B 正確**：線圈與磁場產生電磁推力，直接推動平台。
2. **B 正確**：直驅平台仍然可能需要線性滑軌支撐及導引。
3. **B 正確**：旋轉平台應認識 Rotary Direct Drive。

接下來我們進入直驅模組的下一課。這次會認識你指定的三個元件，而且會以 GMT 的 GLM-Z 微型平台作為實際案例。

# 第 5 單元（二）：音圈馬達、阻尼器與減速機構

先看 GMT 官方的 GLM-Z 微型平台。

![Image](https://images.openai.com/static-rsc-4/pUi9CA4bvrkCxR_xAkfTxDD61JwcqIXfpPAkg9qdj-YgU-_uDap9_iIYATOQnqwELeAq5_ws9DTN2gcwQHhAcxM4XZteOvbxXW1U0onAbYTFZ1KiJ2BsNdJVjTxmymq9j2pntCMVphnYxAzYKQ54JFMg0OYVAFwc7JvjDsA7nUFplEPVveTMerzFZHod3Lgj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/HnBnqK4d7RokvvOG8F14ly7PCCvQaybNUf3X7WfUgj6SSPwiZlV1DlDtAUj5mTk3QX8_dSfOhvye3yeTO23ZRniaVkC8CBVSxLde990pyCLXrsQzxc-_wI9kkWuGnP_oPYPz-13gavsNUv4KfdK7CTeZiENnWnHfmd-F3YPlcqZtVpqVnNz_leRxnhy5VwLz?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CxDjBNUZDHcPrpH1yLYsv1WZGd6jyStHK6Og-vAhYHgjbFkt8ePUMXeFuCTYb-qru3XERzByje5oBxgq03AW-DuFR4bZdqQbZQdekvQJvvHt08yYM6qY9u3OCaOewCGoEtRjAZ6RVtbywE4p95f9we6n_fWuhAbfYehFTMzP3C5whI2kFbEYe2Ka1YwdC4cw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X_P8M-nl2PHQQYzKNhm9r2N7b0WR5CtGpAT108tAs3giCuZWGye32OFmctnOfTOMjUlo6Ra6BtAialDvEDsgIbw7OBeVcAKf1eyKTWyY8gzV4t2N-isCc8ZUwdpsaEp6nZ2II_Kx5ajfhJujbwvtxWglpuSRzr770pNaDY3d5_EGQd2aTdxdvgPfxQDcD_zV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/49nNq8g6nn0VqgNksbCZorIFJJKYd-1dh69UqLDDYlR663pBHGJI1FBUKJhyOuP_8lqZWqm7_9-H0JLABr9uxYzq2B_vgCOCCDN5RieixcRbakVumIjKpotljLp0-6_LcDUA7t0bszHhtx8zfnVrtYHU8DiV_oza9WcGvts1p6aag_8eEA_e-mruc6OFZVyt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/W752UR5-H88yuI1n5BwqW6Z7O5v7Iw779elbjCL92ysOIhTJdgbxvGP6stoPSGzeBD4awOcL8i_StEwKZq9VWytSBBJzHChkrcQ45a1L71JtbKO0v8cx4NNujo58Q9b1JZXZUviK_D2eMOSYqoK8lalLJ9wF_CIsINBX7rFanKaqIK9p_QIJJLK59JQaUidx?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WS5LtGYQZHWACsfoGQYUQCTNCnZh-saH6FGhJqXuqd-cUqukmJ7LW6l8cN9QPPIxBsm0MLcJ-i2vaV1MnpiQbxkB6d4y_g9y-hkIYEdJVyfMQ1IIFHSyxD8Cane9dQyOzTjAJoup6bmSI7BerK5SDYwBh6kn7Fatiokl4zLg-8vPj-Ylmx4dQk7U0kjqFClt?purpose=fullsize)

GMT 官方說明，GLM-Z 使用音圈馬達（Voice Coil Motor），搭配阻尼器提升穩定性，而且沒有減速機構。這三個概念剛好可以連在一起學習。([高明鐵 GMT][1])

---

## 一、音圈馬達（Voice Coil Motor，VCM）

### 1. 它是什麼？

音圈馬達是一種利用電磁作用力產生直線運動的馬達。

它的基本結構可以先簡化成兩個部分：

* **永久磁鐵（Permanent Magnet）**：提供磁場。
* **線圈（Coil）**：通電後在磁場中產生力量。

![Image](https://images.openai.com/static-rsc-4/SaxRTGWR5QSu9ef1DdZM21thu8Z7OQ5psCsT8BuKFuTzmJjLKiQF7BNjzEyLqwgfVyoHY3rQz89qg_lkLh8w4bbyIapPlbanMq92bnFmvHgtzU4CzQrXr3VV3XNC_agxJWLb1V4RxYOhpOgYshqZ8WLefLZg2CU2BkQhtHYY7Rd1q0ZiJNSgtKX7_26e5_sW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/X_P8M-nl2PHQQYzKNhm9r2N7b0WR5CtGpAT108tAs3giCuZWGye32OFmctnOfTOMjUlo6Ra6BtAialDvEDsgIbw7OBeVcAKf1eyKTWyY8gzV4t2N-isCc8ZUwdpsaEp6nZ2II_Kx5ajfhJujbwvtxWglpuSRzr770pNaDY3d5_EGQd2aTdxdvgPfxQDcD_zV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NYozoPkxUAHuc4HIzB9aI0jydeiYX_MZhIJ6wkgQhpBTX8VmqKJAYkpIVaSbMlaHNzs-tiRm68Sf_Ix42RmKG5Sj5Y57kKtNnnDEd-2ouIY8UQlQe6vmoiPyKWDJRCHFjvMiA459GOzuWOe0LsNaiskGqg8A7bhOAnCeNkkv2fXnz2YUoZikrmsQdizrmYL4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/g3ZFcILv2ljTuuYoQcp_FX3gEDP0RztkKgWChfF0CbvhInG6BoUo3W8esa6j9pCzHpETNrDoRcdJCEJtOu1u_6LaoYGuKmtFnO6hsPhiLMuGGfMkmgBPbXxS4mFeOmZcUErSN-n3dadsBT9W1bWDllY7FQ5OLwFshbo18aupwsuHurV3J0FuKzaw8RZgnHLE?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/f8RhysYsjpBKci0oIcz0YxxfJ4uYMfNu0IfV1ULeWQpunwiKeohm0NiGWM2wEhcAeVgUt0HPB05Y3TvNBVo5tK-Gw6plgWz4H7wgNw1vMMIR3rwDX-1K7i_m2MK7fFRgg-GvQvajHMEG5cLm_plt5spcjeV6krHEZNw4W13jWmUt85kNza_mKme65-rXLlCd?purpose=fullsize)

### 2. 它如何運作？

我們用四個步驟理解：

**第一步：線圈通電**

控制器透過驅動器讓電流流過線圈。

**第二步：產生電磁力**

線圈位於磁場中，通電後會產生電磁作用力。

**第三步：線圈開始移動**

這個力量會讓線圈及與它連接的移動部件產生直線運動。

**第四步：控制移動方向**

改變電流方向，可以改變推力方向，因此平台可以向上或向下移動（實際方向取決於機構安裝方式）。

### 3. 音圈馬達有什麼特點？

它有一個很重要的特性：

**推力大小與線圈電流大小大致成正比。**

例如，假設：

* 電流 1 A，產生 5 N 推力。
* 電流 2 A，理想情況下產生約 10 N 推力。

這只是用來理解原理的假設數字，不是 GMT 的產品規格。

因此，控制器可以透過控制電流，調整馬達推力。

### 4. 音圈馬達與滾珠螺桿有什麼不同？

| 比較   | 滾珠螺桿       | 音圈馬達        |
| ---- | ---------- | ----------- |
| 動力原理 | 旋轉轉換成直線運動  | 電磁力直接產生直線推力 |
| 傳動機構 | 螺桿、螺帽      | 不需要螺桿、螺帽    |
| 運動方向 | 可依螺桿旋轉方向改變 | 可依電流方向改變    |
| 背隙   | 可能存在機械背隙   | 沒有螺桿傳動背隙    |
| 行程   | 可設計較長行程    | 常用於短行程精密運動  |

**特別注意：音圈馬達不是只能向上推，不能向下拉。**

只要電流方向改變，它就可以改變推力方向。

---

## 二、阻尼器（Damper）

現在來看第二個元件。

### 1. 為什麼直驅平台需要阻尼器？

假設音圈馬達推動平台向上移動。

平台到達目標位置後，可能因為慣性而產生微小振動。

例如：

目標位置：5 mm

平台實際移動時可能出現：

5.02 mm → 4.99 mm → 5.01 mm → 5.00 mm

這種來回振動可能影響精密定位。

所以我們需要一個元件，協助抑制振動。

這就是阻尼器的用途。

![Image](https://images.openai.com/static-rsc-4/RXcB1ls5g8iEyl4QPva2u4AfbDAiSQ-or9XaI16fvfpNymZm9uiOPdms1IGaG9EyILAYlLdFR1iyAgpE8k567OoEZhPxYk5vgl7vaQh8L2koPJBDPIwR8hOItU2RCOwqdHUn1dwBRxSy7YysQGK_ebma25Zw9PicYymbZ3DqK5GQjLBtMsN6ZLuRgF6JKtxH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/whjpqb2-a2NPvLFgMebuQ5sfepWqok1OVl15cx3oKiigOTBcDzHOMLRn3fybles_PauQWAa-gAW77XmMMu2OP-Yjysemp-SaFNg0g5iQTIzZa4gWP-ao1XUop_blohrbvvi3QETXcoZfSlGXow48UZ0sU6qMPgHjUakTt2cLynSBFma5zDVx_iu0w5rynxfs?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ADLV5_qfiaGQ5ozdAqQDk5nSC1a0HpOsVkvDAZJLUQkdtQJ0ovYy9kqXGq3PRD-Q0RC5IWGFLT488rVg63LlfZ4N0RW9Vk9KsESQaFjVrfGgJv6cJjZZvzAlfixYp-Y_Cko5IVtHiO-DuByPZzZlCnw3AXTDxbM-1jAgULc8fGUwLcc2V91KLmO5fVSCCPae?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/PxjGNveQHWhFWbGd4BjPfpSqepHdusnKcVcWoR3L8xyBvqBkZhTmMkyyL1tUctHtpyG8fTIOm8FUFPuWkIjB90-r6FhX9hrUFBTRffLuCIFeaUNdjyGh0aqWbGXpcUAMRvCmYyvVPo4F0OkrPb9TrhsQQ_yHvLDydimT5-NQNKiwAaJSNdKZdMHqecXR144k?purpose=fullsize)

### 2. 阻尼器的主要功能

阻尼器（Damper）主要是：

**吸收或耗散運動過程中的能量，降低振動及過度擺動。**

你可以把它想像成汽車避震系統的一部分。

汽車經過坑洞後，彈簧會讓車身上下彈跳；阻尼器則協助抑制這種反覆彈跳。

精密平台也是類似的概念。

| 元件   | 主要作用        |
| ---- | ----------- |
| 音圈馬達 | 提供推力，讓平台移動  |
| 阻尼器  | 抑制振動，協助平台穩定 |

這兩個元件的工作完全不同。

### 3. 阻尼器是不是煞車？

不是。

這一點很重要：

* **阻尼器**：抑制運動過程中的振動。
* **煞車器（Brake）**：用來制動或保持機構位置。

阻尼器不等於煞車器，也不能直接假設它可以在斷電時固定住平台。

另外，GMT 官方目前說明 GLM-Z 搭配阻尼器提升穩定性，但沒有在該產品頁面明確公開阻尼器的內部結構，因此我們先不推測它使用油壓、摩擦或其他哪一種阻尼方式。

---

## 三、減速機構（Reducer / Gearbox）

第三個元件是減速機構。

你之前已經學過馬達、聯軸器及滾珠螺桿。

現在我們來理解為什麼有些馬達需要減速機構。

### 1. 減速機構是什麼？

減速機構通常利用齒輪組，讓馬達的輸出轉速降低，同時提高輸出扭力。

例如：

假設馬達：

* 轉速：3000 RPM
* 輸出扭力：1 N·m

加入一個理想的 10:1 減速機構後：

* 輸出轉速：約 300 RPM
* 輸出扭力：約 10 N·m

這是忽略效率損失的理想示意數字。

也就是說：

**轉速降低，扭力增加。**

![Image](https://images.openai.com/static-rsc-4/Ek8a6EwNxq8Oz8mD-tLSVoTkMrgSDmwZfO4H6iI1iThzl8U4YUCzXbAaWLXan_Sp4k35hjYBEctvPM7izRT8AWtJwtdSU6WmxlZ_iAc_FXOr_3EmOT2Tme-_sy9mTajuQRp12B72TjlFOgnMIl9G9bVSpAj8WWFTdy-olnS_HnO692Pao8BQo9n7kGGZzRfp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/yok3l95jv0ibQAjWFCIbGGE1zCO93LL1qjgtj6UYPdbzn28EqT6R8E8eM2_pFdyq13TJP7YKsRpv1iet8I88qAMfl9Bx1Uw0BEWf2u6YU9uFKIGAKPdBXDSXUm17F_cnH73uvEbb1xVwz7z5VI5fmz-2_RbSiLnWw7J59n4L2IEMAu9buWi5rIZ1vtQzI4Db?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CcvaiSwCC5Tfz9vrVpDqxSFWtuzoUSxcyjQPe3v7BJS7Nqhm8W618l4gI9ci-9-re_8uleQhl9yS524pRPya1002FG0Fu76PbYHCUzj5CvtNACZ7V9SUOwFJRuDrdoIzNsIVvaquMZVjjcnUsm9mgdXrGpmvt2gdfRBWioBLhmVkaWwo__cUQZHZyoeGuZl4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Qx1W9rbXCMf1ry0z2UDUYxaShZWgfo2UeRpWPo-1N9WJUDHLcYDTBoNHbsxcUU4rn5Uf9hReZVO4Jc5yb-Kh3bBI1cUMDJ11iE2WmXELSFqSVP-1eA9nZbWYHnxGgV9qoU4mnIpaFsw71GfbnZpz-JTjcsD7gRPwLrTRC_kwoqqkJ6XtFdVn_G-G3niWyBfO?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oRhjR_3J-ZLW_a-xS_2pbn1_ke6IdLFB6v_5lSJmP2iKwVu2DCoeG3ZNVJnm2k-c3Wdxo9ICc7_7vFI5uhLsdn1-ATvsJCZRRlkoO2PKb7GfHXPVGPLDndHY2YGEWHWPiZ4lWjOmaZ2lHv6fduYknG1tDQVmVZl2blL93406Uh43V89r5eOncbouZF2KVPxg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/SjHHtBMhx5H4oCA6Taht9JDyGCt-QDxhteS5oI8WqnDONLOYPq-w1itj8dA3kREPCkZhX_QaHpkk5W1sM6QC25rU1Y7zPvM1J1eGyAkqK-qMJ-1hvOEmkSgQ085hRyXC70YOF8tWpliP1AkZm06HKrwYq7esESsVYCgZUm1o4nICC5yJop7S8ThydLGV8MkT?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5daESudmEOdzYZZgHhkYwc6EUlHVDkfFj7VMg1ehDx2YXWoiPHdqHZoAQ9XRBxo8LOK4sPdWaghjz4VaggGRuCIMmESQB-HugePfR43lAx-j2qeJxZED4Pk1GD2a8p3537b_gdZQnna7cD9nlKWuBkjKpWQvy7ub7WK_HC0z3PdJKW-dU2Fh0TaeKERMsbs4?purpose=fullsize)

### 2. 為什麼需要減速機構？

假設有一顆馬達：

* 轉速很高。
* 但直接輸出的扭力不足。

如果要帶動比較重的旋轉平台，就可以加入減速機構。

減速機構可以：

1. 降低輸出轉速。
2. 增加輸出扭力。
3. 讓馬達較容易帶動較大的負載。

但是也會增加機械傳動零件。

部分減速機構還可能帶來：

* 齒輪背隙。
* 摩擦與磨損。
* 傳動慣量。
* 噪音。

不同減速機構的特性不同，不能認為每一種減速機構都有相同程度的背隙。

### 3. 為什麼直驅馬達不需要減速機構？

因為直驅馬達的設計，就是讓馬達直接對負載產生所需要的推力或扭力。

例如：

傳統旋轉平台：

馬達 → 減速機構 → 旋轉平台

旋轉直驅平台：

馬達 → 旋轉平台

線性音圈馬達：

音圈馬達 → 直線移動平台

少了中間的減速傳動機構，就能減少機械傳動鏈及相關誤差來源。

但這不代表直驅馬達一定比所有傳統馬達有更大的輸出力量；仍然要看馬達設計、尺寸、電流及負載需求。

---

## 四、把三個元件放在一起理解

回到 GMT GLM-Z 微型平台：

![Image](https://images.openai.com/static-rsc-4/pUi9CA4bvrkCxR_xAkfTxDD61JwcqIXfpPAkg9qdj-YgU-_uDap9_iIYATOQnqwELeAq5_ws9DTN2gcwQHhAcxM4XZteOvbxXW1U0onAbYTFZ1KiJ2BsNdJVjTxmymq9j2pntCMVphnYxAzYKQ54JFMg0OYVAFwc7JvjDsA7nUFplEPVveTMerzFZHod3Lgj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/49nNq8g6nn0VqgNksbCZorIFJJKYd-1dh69UqLDDYlR663pBHGJI1FBUKJhyOuP_8lqZWqm7_9-H0JLABr9uxYzq2B_vgCOCCDN5RieixcRbakVumIjKpotljLp0-6_LcDUA7t0bszHhtx8zfnVrtYHU8DiV_oza9WcGvts1p6aag_8eEA_e-mruc6OFZVyt?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/bCALb48OEJW5SsrF38HB0J0PuoPVWbSGtCBQcBj8C1Ji8c7-kZHTP2YDRN-4_RnsoZqjsBw1Z_9wt-4qnbsxbkEv4tIzeWZz2aXAlyitv2wk3U5hPVoAUTm6QXaDOgSZ2QsU8yPU0wDGVWsI4x2mp3J1YB8WbTKTyav5pwVMJOiS8Uu2O8Yc-9j3XzxwZwcM?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/_ahEjBLbt4EifJF7RvMivhJ5RYPaBq_STLIG9adSLBG3uJ-F-aIoy8bBcioDXDUfWUS6VjVoEU_2wiSv269sFJW6FwQf3PZb8O8RHrfDIpQ5UgWEcZ_Gq21eXU2Nvmgah6aLm6eoThYgmHvr0nCfnChi3XIirls3jEVCMK2eZaoRhF7c7q5AhZf0JVWbViI-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oeX3G_uY1MrOMkKGf0KDUTAwVJza6ZGh3rxaQ0yOYH-uWix_R7Q31Fcw3chR_iCH8fYYUiY4JDn6JmloQgcYZ6LV3h9vz2PtWV7-qWUzGAtdEvu4WWwftzMlOjvhfluoXgnx8qyh7CJ9iRjATMxi-SkECQeU-5cYc9n_atzuzBg9eyWSYguWpn1bSpJy9HCK?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/lt5o4cJcKednE1EN3oxu6HKsf58Ry6FL0kc_xuxBXriLgbVNTJaKFhWCwzX65-Ywvr2Ix6J3hW5CqXaYQXSwje7cRyEksSvACNV1FNVhgdSld9i46T1uotnakYNyr85eNOSqQ9AuxYCGeOHsxRYysuiLuDMaE3RKxezHAnglBvWAb9000xLfDtEzOn3ItZqu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/qN-Y_EIpv-EXMJG1Bag6HlzF1oDdF28ZlLx8PlENyvB0biyRpyK3bEW4zpPTIsEphC1HDO0N7c_XvZi6t0dx-5rxAOnrqGBQoRPerChHJEyiWrFQXpYVb2Iri9X62in45J3ycZnbw34RMuUdsmHZ1Dn3eQ41pjtB6wc4tz0kxqPNh0C2ok0z9ey_-Z3EvPPy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u-aqOeUNGj6w2WUYnTQPmCBRugcWhftpAgD5HlzK1JyADyHtF3RiLfnxQrI_DHRqq1VsEWpM4QRJ3w3tYRPI1Ghr7UDNvEBLeWGsz3ARcow_9AvJzOgiCVXyPEGdMpLkPeaqQ0bJ24Cz-SdRGlkQK4gF6zxklB7LrUX75AWZ-e0p50E72P7vqJVvfkFyzonW?purpose=fullsize)

我們可以先用以下方式理解它的功能分工：

| 元件   | 在平台中的工作           |
| ---- | ----------------- |
| 音圈馬達 | 產生直線推力，帶動平台       |
| 阻尼器  | 協助抑制振動，提高穩定性      |
| 減速機構 | GLM-Z 官方說明沒有使用此機構 |

這裡有個重要觀念：

**減速機構不是阻尼器，阻尼器也不是馬達。**

三者的功能完全不同。

GMT 官方 GLM-Z 資料還列出以下規格：

| 項目   | GLM-Z 系列官方資料 |
| ---- | -----------: |
| 行程   |         8 mm |
| 連續推力 |         15 N |
| 最大推力 |         45 N |
| 絕對精度 |      ±1.5 µm |
| 重複精度 |        ±1 µm |
| 解析度  |      0.02 µm |

這些是官方產品頁目前列出的系列規格，實際選型仍要依 GLM80-Z 或 GLM100-Z 的個別資料核對。([高明鐵 GMT][1])

官方資料：[GMT GLM-Z 微型平台系列](https://www.gmtglobalinc.com/%E7%9B%B4%E9%A9%85%E6%A8%A1%E7%B5%84%E7%B3%BB%E5%88%97/GLM-Z%E8%BB%B8%E5%BE%AE%E5%9E%8B%E5%B9%B3%E5%8F%B0%E7%B3%BB%E5%88%97?utm_source=chatgpt.com)

---

## 五、本課練習

請回答以下四題：

**問題 1**

音圈馬達的主要動力來源是什麼？

A. 滾珠螺桿旋轉
B. 線圈通電後，在磁場中產生電磁力
C. 齒輪互相咬合

**問題 2**

音圈馬達要改變平台的推動方向，可以怎麼做？

A. 改變線圈電流方向
B. 改變線性編碼器方向
C. 增加阻尼器

**問題 3**

如果平台移動到目標位置後仍然持續微幅振動，哪一個元件可以協助抑制振動？

A. 減速機構
B. 阻尼器
C. 聯軸器

**問題 4**

一顆馬達轉速很高，但輸出扭力不足，想降低轉速並增加輸出扭力，應該考慮哪個元件？

A. 阻尼器
B. 線性編碼器
C. 減速機構

---

你回答完，我們就可以繼續下一個主題：**壓電模組（Piezo Stage）**。這是精密光纖耦合、矽光子晶片對位中非常值得認識的技術。

[1]: https://www.gmtglobalinc.com/%E7%9B%B4%E9%A9%85%E6%A8%A1%E7%B5%84%E7%B3%BB%E5%88%97/GLM-Z%E8%BB%B8%E5%BE%AE%E5%9E%8B%E5%B9%B3%E5%8F%B0%E7%B3%BB%E5%88%97?utm_source=chatgpt.com "GLM-Z軸微型平台系列 - 直驅模組｜高明鐵 GMT"


---

四題全部答對！不過你提出的問題非常重要，而且正好是理解音圈馬達的關鍵。

你會覺得奇怪，是因為我們之前學的馬達，幾乎都是「旋轉馬達」。你熟悉的概念是：

* 定子（Stator）：固定不動。
* 轉子（Rotor）：在定子內旋轉。
* 馬達軸：跟著轉子旋轉。

但是，**音圈馬達（Voice Coil Motor，VCM）與一般旋轉馬達的運動方式不同。它可以設計成直線運動，不一定有旋轉軸。**

我們一步一步來。

# 一、先回答你的問題：移動的是線圈嗎？

**不一定。常見音圈馬達確實有一種設計是線圈移動，但也有磁鐵移動的設計。**

先看這張簡化結構圖：

![Image](https://images.openai.com/static-rsc-4/ZLyumvROmmqP6ph981rlfQc9Hhy1aSkyfSI6NbYZ3KRAH7xKEx0YGiucWSX_xcAPb29RVrP6Slg8a-Eoz6T1CUL1mYw_GRY3p9GVGfPZPBSHtZ2p_8seK4cTeXCaRlOhqVu5sK3h6nvomAheeFi3-IDj7vBK08gDtNYJtMgR9RCV9KncBOIY0d-47syTpoVD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NYozoPkxUAHuc4HIzB9aI0jydeiYX_MZhIJ6wkgQhpBTX8VmqKJAYkpIVaSbMlaHNzs-tiRm68Sf_Ix42RmKG5Sj5Y57kKtNnnDEd-2ouIY8UQlQe6vmoiPyKWDJRCHFjvMiA459GOzuWOe0LsNaiskGqg8A7bhOAnCeNkkv2fXnz2YUoZikrmsQdizrmYL4?purpose=fullsize)

音圈馬達主要有兩個部分：

1. 永久磁鐵：提供磁場。
2. 線圈：通電後產生電磁作用力。

這兩個部分中：

* 一個固定在底座。
* 另一個連接移動平台。

因此，常見的配置有兩種：

| 設計    | 固定不動 | 跟著平台移動 |
| ----- | ---- | ------ |
| 移動線圈型 | 永久磁鐵 | 線圈     |
| 移動磁鐵型 | 線圈   | 永久磁鐵   |

所以，不能認為所有音圈馬達都是線圈移動。

---

# 二、為什麼線圈可以上下移動？

這裡先不談複雜的電磁學，只理解一個原理。

**通電的線圈放在磁場中，會受到電磁力作用。**

你可以想像：

* 永久磁鐵固定在底座。
* 線圈放在磁鐵產生的磁場中。
* 線圈通電。
* 線圈受到向上的力量。

![Image](https://images.openai.com/static-rsc-4/X_P8M-nl2PHQQYzKNhm9r2N7b0WR5CtGpAT108tAs3giCuZWGye32OFmctnOfTOMjUlo6Ra6BtAialDvEDsgIbw7OBeVcAKf1eyKTWyY8gzV4t2N-isCc8ZUwdpsaEp6nZ2II_Kx5ajfhJujbwvtxWglpuSRzr770pNaDY3d5_EGQd2aTdxdvgPfxQDcD_zV?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/f8RhysYsjpBKci0oIcz0YxxfJ4uYMfNu0IfV1ULeWQpunwiKeohm0NiGWM2wEhcAeVgUt0HPB05Y3TvNBVo5tK-Gw6plgWz4H7wgNw1vMMIR3rwDX-1K7i_m2MK7fFRgg-GvQvajHMEG5cLm_plt5spcjeV6krHEZNw4W13jWmUt85kNza_mKme65-rXLlCd?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/COD2yPk1ZtgYjak8HaRFNzoRnPLppray5PS-QZIkx8tVZ-1w56lOaAw9rf8RVIDvEA5mSlAbrDU7IAnqxOI-_3HqPToMAYXTpfBPwVI6KrH0UxVm1B80XA5Zy4C3dEV246XPjIhjYiBZJZqYLq18A1MfSintK69GI2V0XfmKxwb56qB_vUppuG5BbSnkUHnu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Lg4iB3jaW8h-8HyIntLPMuHx9H0xRgwRHG6fJ4gXJBYxkt8ckDgrA5qhL9SaYo10aawr8lHCnH4bbd-YlMeFVhm9hEkg-tR4WXVzXwHVp9Qe8I9OSq1OLygoR9Dhlp2-s6SP_a8lzpJP1aMrK-bleeEbGQZwkqrlHiYK6HSq2ZobhUqj3qw1Z8cJqAp8OUFp?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0UWadWgyWOHsJm9CU6qcODs7Px4Bwham0--B5PuryGnLeNctsfyXg4z7AUpsoqMkJFSXMuDXJe6b0bvVORZAVgFc_80c0EuTruxGl6LoyiTZnDvR_A19vxbVEkC48Am673Z_zwKyXcMX14_arNbhCpXqbGhNCqSg8gTaBhAGpGqY6olPs5FuB_LwlR1PHs9B?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/Fg-X0_a2tRbq5Hkaty-Q9yrwAk5Di3jwkr3N4bbFcvV7OeXgUhEpD3Tv9UKKfpk10ol7qSaoWUzIc-qj-6JV5wFFYNzb8uhTfM3iSE9ZnDc3SKIIETHD1h9KceBZ9ajpcfTNJpVTSBjTmI-Y02DblNNNFXkaJuiotJJcdAhCbE9dBoNjCrv_jAfKEtdkUDYe?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/9u6nj8HY6xKlpTxH9mkr-OwemHCUr5q6mhRJrcG1vtQImiFyf3NB5xV-Nmdc-wyHACOHUNwXFTZRFhN0g6MR8jAn3mZsEJYItOmvBxgQ1rWn6cT06haOENZC_7C4bDFWP-ozdSLrdcZRP4X8QOGhkmidyNQbqO1V6XwGOI_hoSaQM-7c3gX0NyYU145hSp0F?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/NYozoPkxUAHuc4HIzB9aI0jydeiYX_MZhIJ6wkgQhpBTX8VmqKJAYkpIVaSbMlaHNzs-tiRm68Sf_Ix42RmKG5Sj5Y57kKtNnnDEd-2ouIY8UQlQe6vmoiPyKWDJRCHFjvMiA459GOzuWOe0LsNaiskGqg8A7bhOAnCeNkkv2fXnz2YUoZikrmsQdizrmYL4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/g3ZFcILv2ljTuuYoQcp_FX3gEDP0RztkKgWChfF0CbvhInG6BoUo3W8esa6j9pCzHpETNrDoRcdJCEJtOu1u_6LaoYGuKmtFnO6hsPhiLMuGGfMkmgBPbXxS4mFeOmZcUErSN-n3dadsBT9W1bWDllY7FQ5OLwFshbo18aupwsuHurV3J0FuKzaw8RZgnHLE?purpose=fullsize)

如果線圈連接著移動平台，那麼：

線圈受到向上的力量
↓
線圈帶動平台向上
↓
平台移動

如果改變線圈電流方向，電磁力方向也會改變：

線圈受到向下的力量
↓
線圈帶動平台向下

因此，音圈馬達可以控制平台上下移動。

這裡有一個非常重要的觀念：

**音圈馬達不需要先旋轉，再透過螺桿把旋轉轉換成直線運動。它本身就能直接產生直線推力。**

---

# 三、音圈馬達也有轉子和定子嗎？

這個問題問得很好。

一般旋轉馬達的轉子、定子，是按照旋轉運動來定義的。

但音圈馬達是直線運動，因此通常不會使用傳統旋轉馬達的轉子、定子概念來描述，而是使用：

* Coil（線圈）
* Magnet（磁鐵）
* Moving Part（移動部件）
* Fixed Part（固定部件）

如果硬要類比：

| 旋轉馬達   | 音圈馬達       |
| ------ | ---------- |
| 定子固定   | 固定的磁鐵或線圈組件 |
| 轉子旋轉   | 移動的線圈或磁鐵組件 |
| 產生旋轉扭力 | 產生直線推力     |
| 旋轉軸輸出  | 直線移動部件輸出   |

注意：這只是功能上的類比，不代表音圈馬達一定有真正的轉子和定子。

---

# 四、音圈馬達可以旋轉嗎？

這裡要區分兩件事。

### 1. 一般音圈馬達

一般音圈馬達設計成直線運動。

例如 GMT GLM-Z 微型平台：

![Image](https://images.openai.com/static-rsc-4/pUi9CA4bvrkCxR_xAkfTxDD61JwcqIXfpPAkg9qdj-YgU-_uDap9_iIYATOQnqwELeAq5_ws9DTN2gcwQHhAcxM4XZteOvbxXW1U0onAbYTFZ1KiJ2BsNdJVjTxmymq9j2pntCMVphnYxAzYKQ54JFMg0OYVAFwc7JvjDsA7nUFplEPVveTMerzFZHod3Lgj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/cckKSiiaPnl_fhNRtHjkajcjIzqCa2tACfq5nF-TGKTAEP8WRXiz2EN1oc64t8wT_of_kxgQHzfT12HKHYINw60ANzFonNWxG1AmsD73SJebbx6m7Cio_LFM3BfPrw260UaQfywGLoV8IwlFcWxZbEy8lhe_eMpx1w1AhsFW4mxNWR3DdRQXsHr-94xSpUUa?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/CxDjBNUZDHcPrpH1yLYsv1WZGd6jyStHK6Og-vAhYHgjbFkt8ePUMXeFuCTYb-qru3XERzByje5oBxgq03AW-DuFR4bZdqQbZQdekvQJvvHt08yYM6qY9u3OCaOewCGoEtRjAZ6RVtbywE4p95f9we6n_fWuhAbfYehFTMzP3C5whI2kFbEYe2Ka1YwdC4cw?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LWMT1TnubjD1fU-7azIh_as2Mcfk9iSg6em774B3p9jfiag7LOCqZYVS47OhbzYNyO6ubOCLIb8ipq6rYRqla4bHmTL8xHD-Iuk5_gHhC7jb54jpvRDELL6rsiwaxrjJTuxLKlb0M8tICm3Vws3E4rVp2rHeWhWSSmrhqRT3rQlTCCr0HI2HmrNNVfNakNV-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/u-aqOeUNGj6w2WUYnTQPmCBRugcWhftpAgD5HlzK1JyADyHtF3RiLfnxQrI_DHRqq1VsEWpM4QRJ3w3tYRPI1Ghr7UDNvEBLeWGsz3ARcow_9AvJzOgiCVXyPEGdMpLkPeaqQ0bJ24Cz-SdRGlkQK4gF6zxklB7LrUX75AWZ-e0p50E72P7vqJVvfkFyzonW?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/EkmJ3ZNkdmL4XDeuan23JrdNC0tBzY-5DQC4m0r-eS5zZR_g0-VqrNz6Vpl_xTxJ457t-noMvez8LuZqtKEznENjNAG-MDNqsOeLzMU_BWLIdrjHYikG02Nbi8T5NJpZa4M-GHm8J_CVMkA4OmQ15kAB5GfzTemLXRtGdxk0Y_Riz1a3UW7XEJHCnlbCHqcm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iVMG_K1CpacHZ0SGpWaAEDShOthx5Wi_UvQYigg1rwSeBDNkF1YxqGEpfIji9OLyb5r8c-V2a0eR1aRlemiOw02XQ0bNF7hSRq68925G9fXgi6TwH7bviDEVq45ZI_b4_8rwTZEM7IFRRWLN46J8-Kh7wA4uNN2WbQmfswNIdcYJZLrMyOKKvmsKnHy4enyj?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/WS5LtGYQZHWACsfoGQYUQCTNCnZh-saH6FGhJqXuqd-cUqukmJ7LW6l8cN9QPPIxBsm0MLcJ-i2vaV1MnpiQbxkB6d4y_g9y-hkIYEdJVyfMQ1IIFHSyxD8Cane9dQyOzTjAJoup6bmSI7BerK5SDYwBh6kn7Fatiokl4zLg-8vPj-Ylmx4dQk7U0kjqFClt?purpose=fullsize)

它的用途是讓平台沿 Z 軸上下移動。

它不是用來帶動平台連續旋轉的。

### 2. 旋轉音圈致動器

音圈原理也可以用在旋轉致動器上。

例如，讓線圈或磁鐵受到切向電磁力，產生旋轉力矩，使機構在一定角度範圍內擺動。

![Image](https://images.openai.com/static-rsc-4/dVJQgjQqa-sglUXgyJ6zOxrInsN_U0kot0QS5JdgiD0u0tLTrUBdrgrr6qOqgKwFrgOZKB4I0XcPuB1uFfh3NXQ7rvtr3zeImamlh3a8JFJabjnypQy96NyN5SCbfQuFBlK9GkYNZlnHuhiMDmGELmxWhs--sG1eq_7gGYXueAXCr9J7mD2od1xXNufaJAm7?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/j_c6weEv1Uk7aOx0L6peA9BuRZsWKSEp2XDasdgfzsPYXrzPCylNlUo7YhTKCPlpHjodCtowV4E6kS52lAw1g6kxdqK2IWLEtxQ0Ip-s_ZsjaDVGTv-D5zrVbMre890ztX3OrvPUtQ2Lk9WaPwukLBvEBx_AXYoTBBdTug960_TQLXMj7Byz4d8KK_ZOSTsQ?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0RJzH0rPiQ6_Dnjqya333WPIj4mOlZBwjxA81GJcDdTv-1zALGej6bf8rUQ6a8KV8wY9U48Tkci8ApA9jYBbBLHypNm15dlFk4Tt6en48snHQ-aawmCzTWInHd61WHAgxDqXpTOLQJfCONWzQz30LM9fJ_n6K0A1yJzfTfBwFXFq1VJQ0E2SDYbbtWJaxOPy?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/ZLyumvROmmqP6ph981rlfQc9Hhy1aSkyfSI6NbYZ3KRAH7xKEx0YGiucWSX_xcAPb29RVrP6Slg8a-Eoz6T1CUL1mYw_GRY3p9GVGfPZPBSHtZ2p_8seK4cTeXCaRlOhqVu5sK3h6nvomAheeFi3-IDj7vBK08gDtNYJtMgR9RCV9KncBOIY0d-47syTpoVD?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/LUW0UceJ4LN3_NxI-3Uwr_GM6R0CT25FnL6JEIMQygiZwcxI7_PmAM2cED8wn38YcYXLS2nECyAKW8O1xjNAWqn3NN-7Yz3vwdLMdqW2JcIv0sjAdRR9ELN4ZQJDjxEEZlDTs788wpei8P5IS4Q0OtjQpKG6WxFaTTQSeoJKPNXQ2cpV1d5IrZgDTr8G18gO?purpose=fullsize)

這種裝置可以產生旋轉運動，但常見用途是有限角度的旋轉或擺動，不一定適合像一般旋轉馬達那樣連續旋轉 360 度。

所以：

* 直線音圈馬達：產生直線推力。
* 旋轉音圈致動器：產生旋轉力矩。
* Rotary Direct Drive：則是另一類旋轉直驅馬達技術。

---

# 五、把它與你熟悉的 STM32 控制連起來

假設 STM32 要讓 GMT GLM-Z 平台向上移動 2 mm。

控制流程可以概念化為：

1. STM32 下達目標位置。
2. 運動控制器或驅動器計算需要的控制輸出。
3. 驅動器控制音圈馬達的線圈電流。
4. 線圈與磁場產生電磁推力。
5. 平台向上移動。
6. 位置感測器回報平台位置。
7. 控制系統根據位置誤差繼續修正。

這與你之前學習的伺服馬達控制有相似之處，但動力來源和機械傳動方式不同。

尤其要記住：

**音圈馬達的推力主要由電流控制，而不是由馬達旋轉圈數和螺桿導程決定。**

---

## 六、確認你真正理解了沒有

這次不急著進入壓電模組，我想確認你對音圈馬達的理解。

**問題 1**

如果採用「固定磁鐵、移動線圈」的音圈馬達設計，當線圈通電產生向上的電磁力時，哪個零件會移動？

A. 永久磁鐵
B. 線圈及與它連接的平台
C. 整個底座

**問題 2**

音圈馬達為什麼不需要滾珠螺桿，就能讓平台上下移動？

A. 因為它先讓馬達旋轉，再透過齒輪轉換
B. 因為線圈與磁場直接產生直線推力
C. 因為編碼器可以推動平台

**問題 3**

一般直線音圈馬達與一般旋轉伺服馬達，最大的差異是什麼？

A. 一個直接產生直線推力，一個主要產生旋轉扭力
B. 一個有電，一個沒有電
C. 一個需要編碼器，一個完全不需要編碼器

你先回答這三題。這部分弄懂之後，再進入壓電模組會容易很多。


三題全部正確！🎯

你現在已經掌握音圈馬達最重要的概念：

1. **固定磁鐵、移動線圈**：線圈通電後受到電磁力，帶動平台移動。
2. **不需要滾珠螺桿**：因為音圈馬達直接產生直線推力。
3. **與旋轉伺服馬達的差異**：旋轉伺服馬達主要輸出旋轉扭力，直線音圈馬達直接輸出直線推力。

我再幫你整理一個重要觀念，方便你日後看 GMT 設備時判斷：

| 項目     | 旋轉伺服馬達       | 直線音圈馬達   |
| ------ | ------------ | -------- |
| 主要運動   | 旋轉           | 直線       |
| 主要輸出   | 扭力           | 推力       |
| 動力來源   | 電磁作用         | 電磁作用     |
| 常見回授   | 旋轉編碼器        | 線性編碼器    |
| 是否需要螺桿 | 直線滑台通常需要傳動機構 | 不需要螺桿    |
| 控制重點   | 電流、速度、位置     | 電流、推力、位置 |

還有一點：音圈馬達通常沒有滾珠螺桿的機械自鎖效果，因此垂直 Z 軸設計必須考慮斷電後平台如何保持或防止下墜。

**音圈馬達這個主題可以先告一段落。**

下一課我們進入原定課程：

**壓電模組（Piezo Stage）——如何讓平台進行微米甚至奈米等級的精密位移？**

這會直接連結到你未來要學習的矽光子晶片、光纖耦合與光學對位。


---





















































































































