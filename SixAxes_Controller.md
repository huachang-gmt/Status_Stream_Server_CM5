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
          Lens / PIC 對位
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
































































































