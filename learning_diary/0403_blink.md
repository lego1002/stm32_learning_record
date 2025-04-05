### STM32 Nucleo-F401RE 開發筆記

感謝賽車隊的影片指導，我成功將程式燒錄到 STM32 Nucleo-F401RE 開發板上！  
原本的目標是利用板子上的按鈕來控制 LED 的開關，但由於不確定按鈕對應的 GPIO 腳位，加上期中考的壓力，暫時放下這個問題，先專注於透過 GPIO digital output 實現更多應用。

這次的實作內容是讓外接的 LED 燈和板子內建的 LED 燈以不同頻率閃爍，概念類似於機電整合課程中學到的 `millis()`，但在 STM32 的開發環境中語法有所不同。

---

### 基礎概念整理

1. **HAL (Hardware Abstraction Layer)**  
   硬體抽象層，介於軟體與硬體之間的中介層，簡化硬體操作。

2. **GPIO 狀態**  
   - `GPIO_PIN_SET`: 高電位  
   - `GPIO_PIN_RESET`: 低電位  

3. **GPIO 腳位命名範例**  
   - `PD0`: Port D 的 Pin 0

4. **嵌入式作業系統 (Embedded Operating System)**  
   - 常見的系統有 MBED、RTOS 等。

---

### STM32 開發流程

1. **Configuration & Generate Code**  
   使用 CubeMX 配置硬體資源並生成程式碼。

2. **Compile & Debug**  
   使用 CubeIDE 將程式碼編譯為機器碼並進行除錯。

3. **Run Machine Code**  
   使用 CubeProgrammer 將機器碼燒錄到開發板並執行。

---

### 小技巧

- 使用 Bluepill 開發板時，確保在燒錄程式時將 Boot0 腳位設為高電位。

---

### 補充建議

1. **按鈕 GPIO 腳位確認**  
   可以參考 Nucleo-F401RE 的資料手冊 (User Manual) 或原理圖，確認按鈕對應的 GPIO 腳位。

2. **不同頻率閃爍的實現**  
   使用 Timer 中斷或 HAL 的非阻塞延遲函數 (`HAL_Delay` 會阻塞程式執行)，以實現更精確的多頻率控制。

3. **學習資源**  
   - [STM32 官方文件](https://www.st.com/en/development-tools/stm32cubemx.html)  
   - [MBED 開發資源](https://os.mbed.com/)  
   - [RTOS 教學](https://www.freertos.org/)

希望這些整理與補充能幫助你更清晰地記錄與學習 STM32 的開發過程！