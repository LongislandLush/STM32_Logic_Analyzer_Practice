# ZeroPlus LAP-C 16032｜GPIO / UART 擷取與解碼操作紀錄

> Lab 01｜2026-10-09｜ZeroPlus Standard V3.14.14 (CN01)｜NUCLEO-F446RE

## 1. 實驗目的

先用 PA8 GPIO 驗證 Logic Analyzer 的輸入、取樣與觸發，再使用 PA9 USART1 TX 擷取 UART 波形，最後將 UART Decode 為 ASCII。此文件依實際操作過的繁體中文介面整理。

## 2. 硬體接線

| STM32 | ZeroPlus | 功能 |
| --- | --- | --- |
| PA8 / Arduino D7 | A0 | GPIO 方波 |
| PA9 / Arduino D8 | A1 | UART TX |
| GND | GND | 共地 |

- STM32 GPIO/UART 訊號為 3.3 V 邏輯；Port A 使用 TTL / 1.5 V Threshold。
- 只量測 TX，不接 UART RX，也不需要將 3.3 V 電源線接至邏輯分析儀輸入。
- 修改接線時避免探棒滑動造成相鄰腳位短路。

## 3. STM32CubeMX / Firmware

1. PA8：`GPIO_Output`、Push-Pull、No Pull。
2. USART1：Asynchronous；PA9 = USART1_TX；115200、8 data bits、None parity、1 stop bit。
3. 使用 CubeMX 產生初始化程式，保留原本 USER CODE 區塊。
4. 在主迴圈每次 Toggle PA8、`HAL_Delay(1)`；每約 100 ms 以 blocking UART 送出一段 ASCII 字串。
5. Build / Flash 後再接上分析儀測試。

範例程式請見專案根目錄 [`README.md`](../README.md)。

## 4. ZeroPlus 基本設定

### 4.1 取樣設定

進入「取樣設定」視窗：

- **取樣信號來源**：選「內部取樣信號」（Timing / Internal Clock），不要選外部取樣信號。
- **GPIO 測試**：1 MHz、16K Samples。
- **UART 測試**：10 MHz、32K Samples（此版本沒有使用者想選的 5 MHz 檔位）。
- **啟動壓縮**：先不勾選。
- **信號濾波**：初次測試先不用。

擷取時間約為 `Memory Depth / Sample Rate`：

| 設定 | 擷取時間 |
| --- | --- |
| 2K / 100 MHz | 20.48 µs |
| 16K / 1 MHz | 16.384 ms |
| 32K / 10 MHz | 3.2768 ms |

**重要**：如果用 2K / 100 MHz 擷取約 250 Hz GPIO，極容易只看到平坦 High 或 Low，因為取樣視窗太短。

### 4.2 Threshold

開啟「觸發屬性」，Port A 設 `TTL / 1.5 V`。此設定已用於本次 3.3 V STM32 訊號；測量其他電壓（尤其 1.8 V I²C）時須重新核對儀器輸入門檻與相容性。

### 4.3 Trigger

在左側通道「觸發條件」選擇：

- GPIO：A0 上升緣或下降緣。
- UART：A1 下降緣（UART Idle High → Start Bit Low）。
- 其他通道：不設定觸發。

UART 可先使用 10% Trigger Position，讓觸發後保留更多資料。按 `F5` 進行單次擷取；`F6` 重複擷取；`F7` 停止。重複擷取不等於無限長時間錄製。

## 5. GPIO 方波驗證

1. A0 接 PA8，並共地。
2. 設 1 MHz / 16K、A0 Edge Trigger。
3. 執行單次擷取，確認 A0 交替 High/Low。
4. 利用 A/B/T 游標量測邊緣間隔；注意 `A-T` 是兩個游標間的時間差，**只有對準相鄰同方向邊緣才是一個完整週期**。
5. 實測 High/Low 約 1.99 ms、頻率約 251 Hz。

### 顯示時間、頻率與滑鼠提示

- 主工具列有「波形寬度資訊」顯示選單，可選「時間模式顯示」、「取樣點模式顯示」、「頻率模式顯示」或不顯示。
- 「自定義設定 → 常用」可選「時間模式」及勾選「顯示波形區提示訊息」。
- 這套 UI 不一定像 Saleae 一樣，滑鼠移到每個邊緣就直接顯示全部脈波資訊；直接在波形上標註時間較實用。

## 6. UART 波形擷取

1. A1 接 PA9 / USART1_TX；A0 可以繼續接 PA8。
2. 取樣設定：10 MHz、32K、Port A TTL / 1.5 V。
3. A1 設 Falling Edge Trigger；Trigger Position 10%。
4. 按 `F5` 擷取。
5. A1 應出現密集的 High/Low UART 位元波形。

115200 baud 時，每個 bit 約 `8.68 µs`；10 MHz 下約有 87 個 samples/bit，足以進行 UART 解碼。

## 7. UART Protocol Decoder：最重要的 ZeroPlus 操作

ZeroPlus **必須先將訊號通道 Group into Bus**，即使 UART 只有一條 TX 訊號也一樣。

1. 在左側只選取 **A1**，不要連 A0 GPIO 一起選。
2. 右鍵選「**歸納信號線為匯流排**」（`Ctrl+G` / Group into Bus）。
3. 出現 `Bus1` 後，在 Bus1 上按右鍵，開啟「**匯流排屬性**」（Bus Property）。
4. 選「**匯流排協定**」，在模組清單選 `ZEROPLUS LA UART MODULE`。
5. 按「**參數配置**」，設定下表。
6. 按確定，Bus1 將顯示解碼資料。
7. 在 UART 模組的「**進制顯示**」分頁勾選「啟動」，選 `ASCII`。

| UART Decoder 參數 | 值 |
| --- | --- |
| 訊號通道 | A1 |
| 同位檢查 | None Parity |
| 資料長度 | 8 |
| 鮑率 | 115200（不使用 Auto） |
| 停止位元 | 1 |
| 傳送方向 | LSB → MSB |
| 取樣位置 | 70%（本次成功設定） |
| 波形反相解碼 | 不勾選 |
| 數據反相解碼 | 不勾選 |
| 進制顯示 | 啟動 → ASCII |

成功時可看到 `Hello ZeroPlus!` 或其他實際送出的 ASCII 字串。波形前後沒有有效封包的區段可能標為 `Unknown`，不代表中間字元解碼失敗。

## 8. 常見問題與排查紀錄

| 現象 | 可能原因／處理 |
| --- | --- |
| 全部顯示 `Unknown` | 確認是否已完成擷取、內部取樣模式、輸入通道與實體接線；不要單憑 `Unknown` 判定 MCU 沒輸出 |
| 一直顯示「等待」 | Trigger 未被滿足；檢查所選通道、下降緣、訊號活動與 Threshold |
| Signal Statuses 全紅 | 代表當下判定 Low；未接通道也可能如此，不能直接認定 Firmware 錯誤 |
| 只看到固定 High | 可能是擷取視窗遠短於方波週期；降低取樣率或增加記憶體深度 |
| UART 原始波形有，但沒有字元 | 先建立單通道 Bus，再選 UART Protocol Analyzer，設定 115200/8N1 與 ASCII |
| 「分析 → 資料格式」為灰色 | 這不是 UART Decoder 入口；不要在這裡找 Protocol Analyzer |
| PA2 / ST-LINK VCP 有字串，ZeroPlus 卻抓不到 | VCP 正常不代表外部探針位置已驗證；需核對 header、接觸、觸發；本實驗改用 PA9 USART1 成功 |
| GPIO 實測約 251 Hz 而非預估 500 Hz | 先以量測為準；後續用 Timer PWM、Tick 設定與執行路徑進一步驗證 |

## 9. 後續練習：I²C

1. 使用 F446RE 與外接 I²C 裝置（例如 MPU6050）。
2. 同時擷取 SCL / SDA，確認正確的上拉電阻與邏輯電壓。
3. 選取兩個訊號通道，Group into Bus，設定 I²C Protocol Analyzer。
4. 練習觀察 `START → Address+W → ACK → Register → ACK → Repeated START → Address+R → ACK → Data → NACK → STOP`。
5. 練習保存正常／異常擷取紀錄，並區分數位解碼結果與實際類比電氣品質。

## 10. 參考

- ZeroPlus LAP-C User Guide V3.14 系列（取樣、Trigger、波形顯示、Bus / Protocol Analyzer）。
- ZeroPlus 官方 UART Protocol Analyzer How It Works：單通道 UART 也需 Group into Bus，再從 Bus Property 設定解碼器。
- 本次 NUCLEO-F446RE / LAP-C 16032 實機操作截圖與量測紀錄。
