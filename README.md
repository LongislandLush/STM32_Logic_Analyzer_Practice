# STM32 Logic Analyzer Lab

使用 **NUCLEO-F446RE** 與 **ZeroPlus LAP-C 16032** 練習數位訊號擷取、GPIO 時序分析及 UART Protocol Decode，後續延伸至 I²C / SPI 除錯。

## Lab 01｜GPIO + UART（2026-10-09）

### 硬體與軟體

| 項目 | 使用環境 |
| --- | --- |
| MCU Board | ST NUCLEO-F446RE |
| Logic Analyzer | ZeroPlus LAP-C 16032（16 CH） |
| Analyzer Software | ZeroPlus Standard V3.14.14 (CN01) |
| Firmware IDE | STM32CubeIDE / STM32CubeMX |
| GPIO | PA8 / Arduino D7 |
| UART | USART1 TX：PA9 / Arduino D8 |
| UART Format | 115200 baud / 8 data bits / no parity / 1 stop bit (8N1) |
| MCU System Clock | HSI → PLL → SYSCLK 84 MHz（依提供的 `SystemClock_Config()` 計算） |

### 實體接線

| NUCLEO-F446RE | LAP-C 16032 | 用途 |
| --- | --- | --- |
| PA8（D7） | A0 | GPIO 方波 |
| PA9（D8） | A1 | USART1 TX |
| GND | GND | 共地 |

> Logic Analyzer 僅用於量測，需確認輸入電壓與 Threshold 相容；不用將 3.3 V 電源接到分析儀訊號輸入。

### CubeMX / Firmware 設定

- **PA8**：GPIO Output、Push-Pull、No Pull、Low Speed，初始 Low。
- **USART1**：Asynchronous，115200 / 8N1、TX/RX Mode、無硬體流控、16x Oversampling；本次實驗只量測 **PA9 TX**。
- **USART2**：專案仍有初始化（115200 / 8N1），但 `while (1)` 中原先的 `Hello ZeroPlus!` USART2 傳送程式已註解，不會執行。
- **Clock**：HSI 16 MHz；PLLM=16、PLLN=336、PLLP=4，SYSCLK 84 MHz；APB1 分頻 2、APB2 分頻 1。

#### 實際執行的 `while (1)` 邏輯

以下依 2026-10-09 提供的 `main.c` 摘錄，保留實際 UART 測試字串：

```c
uint32_t last_uart_tick = HAL_GetTick();

while (1)
{
    /* GPIO PA8: continuous toggle */
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_8);
    HAL_Delay(1);

    /* USART1 PA9: transmit every 100 ms */
    if ((HAL_GetTick() - last_uart_tick) >= 100U)
    {
        const uint8_t msg[] = "Gay Roger & Suck Bear love Bonbon!\r\n";

        HAL_UART_Transmit(
            &huart1,
            (uint8_t *)msg,
            sizeof(msg) - 1U,
            100U
        );

        last_uart_tick = HAL_GetTick();
    }
}
```

> `HAL_Delay(1)` 不保證每次 GPIO 翻轉間隔精準 1 ms；實際週期應以量測為準。`HAL_UART_Transmit()` 為 Blocking API，傳送時主迴圈暫停執行 GPIO Toggle。每約 100 ms 的排程以 HAL Tick 為基礎，實際發送間隔亦可能受執行時間影響。

### ZeroPlus 擷取設定

| 參數 | GPIO 測試 | UART 測試 |
| --- | --- | --- |
| Sampling Rate | 1 MHz | 10 MHz |
| Memory Depth | 16K | 32K |
| Logic Threshold | TTL / 1.5 V | TTL / 1.5 V |
| Trigger | A0 Edge | A1 Falling Edge |
| Trigger Position | 50% | 10% |
| Protocol Decoder | — | UART：A1 / 115200 / 8N1 / LSB → MSB / ASCII |

### 實驗結果

1. **GPIO 擷取成功**：PA8 波形的 High/Low 約 **1.99 ms**，週期約 **3.99 ms**，頻率約 **251 Hz**（本次截圖實測）。這與單純假設 `HAL_Delay(1)` 對應約 500 Hz 不同，原因尚未獨立驗證。
2. **UART 擷取成功**：ZeroPlus A1 量測到 PA9 的 USART1 TX 數位波形。
3. **UART Decode 成功**：將 A1 建立為單通道 Bus，再選擇 UART Protocol Analyzer，成功以 ASCII 顯示測試字串（包含後續自訂內容）。
4. **跨通道時序觀察**：UART Blocking Transmission 期間，PA8 的部分 High/Low 維持時間拉長，與主迴圈遭 Blocking API 暫停的行為相符。
5. **USART2 初始排查**：先前 `USART2 → ST-LINK VCP → COM3` 可收到 `Hello ZeroPlus!`；但從外部 PA2/D1 量測未成功觸發。這不足以證明 PA2 未輸出，相關 Pin Routing / 接點仍未查明。之後改用 USART1 PA9 完成實驗。

### ZeroPlus UART Decode 操作摘要

1. 設定內部取樣時脈、Sampling Rate、Memory Depth、Threshold 與 A1 Falling Edge Trigger。
2. 擷取 A1 UART TX 波形。
3. 只選取 **A1**，右鍵 **Group into Bus**（`Ctrl+G`）。
4. 在 **Bus1** 右鍵開啟 **Bus Property**，選擇 **UART Protocol Analyzer**。
5. 設定 **A1 / 115200 / 8N1 / LSB → MSB / Normal Polarity**。
6. 在 **進制顯示** 啟用 **ASCII**，查看解碼字元。

詳細步驟與故障排查見 [`docs/zeroplus_uart_setup.md`](docs/zeroplus_uart_setup.md)。

### 後續 Lab

1. **Timer PWM**：產生硬體定時的方波，與 `HAL_Delay()` Toggle 比較。
2. **I²C Sensor**：以 MPU6050 等模組擷取 START、Address、ACK、Repeated START、NACK、STOP。
3. **I²C Debug**：練習 Protocol Decode、Trigger 與異常交易保存。
