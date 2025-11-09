# pico_keyboard_ime_terminal


**版本:** 1.8.0 (多分頁 UI 重構)  
**建置:** 里程碑 3.1 除錯建置 (v2.2 - Tag ID 記錄)

本文件旨在深入說明 PicoType Keyboard 韌體的內部架構、核心演算法與設計理念，主要供開發者與技術愛好者參考。

## 核心技術特點

此專案在資源有限的微控制器 (MCU) 上實現了複雜的中文輸入、圖形化介面與網路通訊功能，其核心技術包含以下幾點：

### 1. 模組化 UI 框架 (分頁系統)

為了應對日益複雜的功能，v1.8.0 版本引入了一套基於函式指標 (Function Pointer) 的分頁系統，取代了原有的 `enum` 狀態機。

*   **`Tab` 結構體:**
    ```c++
    struct Tab {
      const char* name;
      DrawFunction draw_func;
      InputHandler input_handler;
    };
    ```    此結構體定義了一個 UI 分頁，其中包含分頁名稱、一個指向其專屬**繪圖函式**的指標 (`draw_func`)，以及一個指向其**按鍵處理函式**的指標 (`input_handler`)。

*   **運作機制:**
    系統透過一個全域索引 `g_current_tab_index` 來追蹤目前作用中的分頁。主迴圈中的 `drawUI()` 和 `processKeyEvent()` 函式會根據此索引，從 `g_tabs[]` 陣列中呼叫對應的繪圖與輸入處理函式。
    ```c++
    // 繪圖時
    g_tabs[g_current_tab_index].draw_func();

    // 處理按鍵時
    g_tabs[g_current_tab_index].input_handler(finalKeyCode);
    ```

*   **優勢:** 這種設計具備高度可擴展性。新增一個功能分頁（例如「設定」或「地圖」）只需：
    1.  撰寫其對應的繪圖與輸入處理函式。
    2.  在 `g_tabs[]` 陣列中新增一筆 `Tab` 實體。
    無需修改核心的 UI 調度邏輯。

### 2. 高效能中文注音輸入法引擎 (IME)

`ImeEngine` 類別是實現中文輸入的核心。它採用了針對嵌入式系統優化的兩階段查詢演算法，以在極低的記憶體佔用下實現快速的字詞查詢。

*   **資料結構:** 所有輸入法資料（注音索引、字詞庫）都被預先處理並儲存在 `PROGMEM`（程式記憶體）中，由 `picotype_data_optimized.h` 引入。
    *   `zhuyin_idx_opt[]`: 注音查詢索引表。每一項包含一個指向字詞庫中「查詢鍵」的指標 (`key_offset`) 和長度，以及指向候選字詞資料的指標 (`data_offset`) 和長度。此索引表**已預先按查詢鍵的字母序排序**。
    *   `zhuyin_pool_opt[]`: 包含所有查詢鍵與候選字詞的巨大字串池。

*   **查詢流程:**
    1.  **按鍵轉注音:** `mapKeyToBopomofo()` 將鍵盤輸入 (如 "su3") 轉換為 UTF-8 注音符號 (如 "ㄙㄨˇ")，用於在螢幕上顯示。
    2.  **注音格式化為查詢鍵:** `formatBopomofoToKey()` 將注音符號轉換為用於在索引中查找的標準化查詢鍵 (如將 "ㄙㄨˇ" 轉換為 "ㄙㄨ3")。聲調被統一轉換為數字 `1`-`5` 並置於尾部，以利於排序與搜尋。
    3.  **二分搜尋 (Binary Search):** `binarySearch()` 函式在已排序的 `zhuyin_idx_opt` 索引表中對查詢鍵執行二分搜尋。這是在數千筆資料中實現毫秒級查找的關鍵。
    4.  **提取候選字:** 一旦找到匹配的索引記錄，便根據記錄中的 `data_offset` 和 `data_len` 從 `zhuyin_pool_opt` 字串池中直接讀取對應的候選字詞字串，並返回給上層 UI 顯示。

### 3. 優化的中文字型渲染

`FontRenderer` 類別負責將 UTF-8 編碼的字串（包含 ASCII 與中文字元）渲染到 ILI9341 螢幕上。

*   **資料結構:**
    *   `font_map_opt[]`: 字型映射表。每一項包含一個 Unicode 碼點，以及該字元在點陣資料池中的偏移量、寬度、高度和繪製偏移等資訊。此映射表**已預先按 Unicode 碼點排序**。
    *   `font_bitmap_data_opt[]`: 儲存所有字元點陣圖資料的巨型位元組陣列。

*   **渲染流程:**
    1.  **UTF-8 解碼:** `decodeUtf8()` 將輸入的字串逐字元解碼為 32 位元的 Unicode 碼點。
    2.  **ASCII 快速路徑:** 如果 Unicode 小於 128，則直接使用 `Adafruit_GFX` 內建的字型進行繪製，效率更高。
    3.  **中文字元查找:** 對於非 ASCII 字元，`findChar()` 函式會在 `font_map_opt` 中執行**二分搜尋**，快速定位該 Unicode 對應的字型資訊。
    4.  **點陣繪製:** 找到字元資訊後，`drawChar()` 函式會使用 `memcpy_P` 從 `PROGMEM` 中將點陣資料高效地複製到一個 RAM 緩衝區 (`g_char_bitmap_buffer`)，然後逐點將其繪製到 TFT 螢幕上。

### 4. Meshtastic Protobuf 通訊協定處理

裝置透過 UART 與 Meshtastic 節點通訊，所有資料交換均遵循其 Protobuf 序列化格式。

*   **封包結構:** Meshtastic 韌體使用一個簡單的封包格式來傳輸 Protobuf 資料：
    `[0x94, 0xc3]` (起始位元組) + `[Len_H, Len_L]` (2位元組長度) + `[Payload]` (Protobuf 資料)

*   **接收與解碼 (`handleProtoRx`):**
    此函式是一個狀態機，用於解析 UART 上接收到的位元組流。
    1.  等待 `0x94` 和 `0xc3` 起始位元組。
    2.  讀取接下來的兩個位元組，組合成承載資料的長度。
    3.  讀取指定長度的承載資料到 `proto_rx_buffer`。
    4.  使用 `nanopb` 的 `pb_decode()` 函式將緩衝區內的二進位資料反序列化為 `meshtastic_FromRadio` 結構體。
    5.  根據結構體中的 `which_payload_variant` 欄位（Tag ID），判斷封包類型（如 `my_info_tag`, `node_info_tag`, `packet_tag` 等），並分派給相應的處理邏輯（如更新節點資料庫、在聊天視窗顯示訊息等）。

*   **編碼與傳送:**
    傳送資料的流程相反。例如 `sendProtobufTextMessage()`:
    1.  建立一個 `meshtastic_ToRadio` 結構體。
    2.  填充需要傳送的資料（目標 ID、訊息內容等）。
    3.  使用 `nanopb` 的 `pb_encode()` 函式將結構體序列化為位元組流，存入 `proto_tx_buffer`。
    4.  依照上述封包結構，加上起始位元組和長度資訊，透過 `Serial1.write()` 傳送出去。

### 5. 硬體抽象 (鍵盤掃描)

鍵盤矩陣的掃描是透過兩顆移位暫存器（74HC595 負責輸出掃描訊號，74HC165 負責讀取按鍵狀態）完成的。

*   `updateKeyboardAndLeds_TwoPulse()` 函式負責整個掃描週期。它透過 `shiftOut` 向 74HC595 傳送資料來依序啟用鍵盤的每一列，同時也順便更新了 LED 的狀態。
*   在啟用每一列後，透過自訂的 `myShiftIn` 函式從 74HC165 讀回該列上所有按鍵的狀態。
*   為了處理硬體設計上的時序問題，函式中使用了兩次 Latch Pin 的脈衝 (`digitalWrite(LATCH_PIN, ...)`), 確保資料能被正確鎖存和讀取。

## 程式碼結構

*   **全域物件與變數:**
    *   `tft`, `renderer`, `ime`: 核心功能的類別實體。
    *   `g_tabs[]`, `g_current_tab_index`: UI 框架的核心。
    *   `g_node_db[]`, `g_channels[]`: Meshtastic 網路的本地資料快取。
    *   `inputBuffer_keys`, `editor_content`: 輸入與編輯區的狀態緩衝區。

*   **主要流程 (`setup()` & `loop()`):**
    *   `setup()`: 初始化硬體、序列埠、顯示器，並載入初始狀態。
    *   `loop()`:
        1.  處理 Protobuf/Terminal 的序列埠接收。
        2.  定期執行 Protobuf 的心跳與握手重試。
        3.  呼叫 `updateKeyboardAndLeds_TwoPulse()` 掃描鍵盤。
        4.  比較本次與上次的按鍵狀態，若有變化則呼叫 `processKeyEvent()` 進行處理。
        5.  更新 `lastKeyState` 狀態。
        6.  若 `needs_redraw` 旗標為真，則呼叫 `drawUI()` 更新螢幕。

## 編譯與依賴

*   **函式庫:**
    *   `Adafruit_GFX` & `Adafruit_ILI9341`: 顯示驅動。
    *   `nanopb`: Google Protobuf 的嵌入式 C 語言實現。
*   **核心資料檔案:**
    *   `picotype_data_optimized.h`: **極為關鍵**。此檔案包含了由外部工具預先處理好的字型點陣、字元映射表、注音索引與字詞庫。缺少此檔案，專案將無法編譯。
*   **Protobuf 生成檔案:**
    *   `mesh.pb.h`, `channel.pb.h` 等檔案需要從 Meshtastic 專案的 `.proto` 定義檔，使用 `nanopb_generator.py` 工具生成。

## 硬體 Pinout

| 功能 | 接腳 (GPIO) | 說明 |
| :--- | :--- | :--- |
| **TFT 顯示器** | | |
| CS | 17 | 片選 |
| RST | 21 | 重置 |
| DC | 20 | 資料/命令 |
| BLK | 22 | 背光控制 |
| **鍵盤移位暫存器** | | |
| DATA_OUT | 15 | 序列資料輸出 (至 74HC595) |
| LATCH | 14 | 鎖存訊號 |
| CLOCK | 26 | 時脈訊號 |
| DATA_IN | 27 | 序列資料輸入 (自 74HC165) |
| **UART 通訊** | | |
| TX | 0 | 至 Meshtastic 裝置的 RX |
| RX | 1 | 至 Meshtastic 裝置的 TX |
