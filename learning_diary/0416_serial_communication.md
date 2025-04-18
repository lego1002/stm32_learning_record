### 1. 發生的問題  
- **看不到「終端機輸入」回應**：以為 MobaXterm 無法送字元到板子，實際上燈一直在按預期閃爍，只是沒注意到。  
- **中斷接收沒觸發**：初始用錯參數 `HAL_UART_Receive_IT(&huart2, &from_computer, from_computer);`，導致 RX 中斷根本沒啟動。  

### 2. 解決流程  
1. **確認燈號行為**  
   - 把 `HAL_UART_Transmit()` 放到 `while(1)`，持續印出測試訊息 → 確認 UART TX 路徑正常、終端機確實能看到訊息。  
   - 目視板上 LED，發現它其實一直在閃，問題出在接收，不是輸入／終端機。  

2. **修正接收啟動**  
   - 把 `HAL_UART_Receive_IT(&huart2, &from_computer, from_computer);` → `HAL_UART_Receive_IT(&huart2, &from_computer, 1);`  
   - 在 `HAL_UART_RxCpltCallback()` 裡使用字元常數比較：  
     ```c
     if (from_computer == '1') {
       HAL_GPIO_TogglePin(A4_blue_GPIO_Port, A4_blue_Pin);
     }
     // 再次啟用下一次中斷
     HAL_UART_Receive_IT(&huart2, &from_computer, 1);
     ```
   - 確認 callback 只保留一份、不重複定義，並在回呼裡重新呼叫 `Receive_IT`。

3. **中斷設定檢查**  
   - 確認 `USART2_IRQn` 已在 HAL_MspInit／MX_USART2_UART_Init 內啟用：  
     ```c
     HAL_NVIC_SetPriority(USART2_IRQn, 0, 0);
     HAL_NVIC_EnableIRQ(USART2_IRQn);
     ```
   - 確認 `stm32l4xx_it.c` 有 `USART2_IRQHandler()` 並呼叫 `HAL_UART_IRQHandler(&huart2)`。

---

### 3. 重點筆記  

| 項目                     | 說明                                       |
|-------------------------|-------------------------------------------|
| **User Code 區塊**       | `BEGIN 0/1/2/3/4`：CubeMX 自動保留區段；<br>只修改對應區塊，不要在同一區塊重複定義函式。 |
| **Transmit**            | 用 `HAL_UART_Transmit(&huart2, buf, len, timeout)`。   |
| **Receive_IT**          | 第三參數 **固定** 1 byte；每次中斷後要 **再呼叫** 一次。      |
| **Callback**            | 在 `HAL_UART_RxCpltCallback` 中處理收到資料；<br>可加 `if(huart->Instance==USART2)` 避免多 UART 混淆。 |
| **主要程式放置**         | 初始化後：<br>```c
HAL_UART_Receive_IT(...);
while(1) {
  // 主迴圈空跑／低功耗，所有 I/O 由中斷驅動
}

``` 
```
## USB CDC
USB Communications Device Class其實就是「USB 上的虛擬 COM 埠」標準，用來讓 USB 裝置跟電腦模擬成一般串列埠。在 Nucleo‑L432KC 上，**ST‑Link/V2‑1 那顆晶片已經幫你把板上 USART2（PA2/PA3）接到電腦，並以 USB CDC 的方式呈現成一個 COM Port**，所以你其實根本不用碰任何 USB Device 的程式碼。

---

### 重點整理

1. **你只要插 USB 線**  
   Nucleo‑L432KC 上電後，Windows/Mac/Linux 會自動偵測到一個「STMicroelectronics Virtual COM Port」。  
   你開終端（MobaXterm、TeraTerm…）選這個 COM Port 就能 RX/TX。

2. **在 STM32 程式裡，依舊用 USART2**  
   ```c
   MX_USART2_UART_Init();               // CubeMX 自動生成
   HAL_UART_Transmit(&huart2, buf, len, timeout);
   HAL_UART_Receive_IT(&huart2, &from_computer, 1);
   ```
   從 MCU 角度來看就是普通的 UART，完全跟 USB CDC 無關。

3. **何時才要自己做 USB CDC？**  
   只有當你要直接使用 STM32 的 USB OTG 外設（OTG_FS）自行實現 USB Device（像是要做自家 USB 滑鼠、鍵盤、或自定義 VCP）時，才會在 CubeMX 裡打開 USB Device → CDC Class，那時候才要寫 `usbd_cdc_if.c`、呼叫 `CDC_Transmit_FS()`、`USBD_CDC_ReceivePacket()`。  
   **但 Nucleo‑L432KC 已經幫你做完這段了**，所以你不用管。

---

> **小結**：  
> • **USB CDC = USB 虛擬 COM 埠**  
> • Nucleo 的 ST‑Link 已經內建 CDC，只要插 USB 線、開對應的 COM Port，就像用一般 UART 一樣呼叫 `HAL_UART_Transmit/Receive_IT`。  
> • 完全不用啟用 CubeMX 裡的 USB DEVICE → CDC 選項，也不用動 `usbd_cdc_if.c`。  
>  
> 直接專心在 USART2 初始化和中斷／迴圈邏輯就好！
| **Debug 小技巧**        | 1. 把 TX 測試放到 `while` 不間斷印 → 確認連線正常<br>2. 目視 LED 閃爍 → 區分 TX／RX 問題<br>3. 終端機要關閉再燒錄，避免 COM port 被佔用。 |
**結論**：因為是第一次用 STM32L432KC 的 ST‑Link Virtual COM，Serial 設定與 USB‑CDC 不同，須特別留意 `HAL_UART_Receive_IT` 的參數與中斷重啟機制，以及 CubeMX User Code 各區塊的用途。這樣才能正確接收終端輸入並控制 LED。


