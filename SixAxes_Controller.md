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


































































