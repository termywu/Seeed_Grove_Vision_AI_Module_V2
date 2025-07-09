### TensorFlow Lite Micro (`@lite/micro`) 架構說明

`@lite/micro` 目錄是 TensorFlow Lite Micro (TFLM) 的核心，專為在記憶體和處理能力極其有限的微控制器（MCU）上運行機器學習模型而設計。其架構旨在實現最小的二進位檔案大小和最低的記憶體使用量。

#### 核心元件

TFLM 的核心圍繞以下幾個主要元件構建：

1.  **`micro_interpreter.h`**: 這是 TFLM 的主要進入點。它負責管理模型的整個生命週期，包括載入模型、分配記憶體（Tensor Arena）、準備和執行運算（Invoke）。應用程式開發者主要透過 `MicroInterpreter` 來與 TFLM 互動。

2.  **`micro_allocator.h`**: 記憶體管理器。TFLM 的一大特點是它在一個稱為「Tensor Arena」的單一、連續的記憶體區塊上進行所有記憶體分配，並且**不使用任何動態記憶體**（如 `malloc`）。`MicroAllocator` 負責在這個 Arena 中規劃和分配所有必要的記憶體，包括張量（Tensors）、運算元的核心（Kernels）狀態以及暫存緩衝區（Scratch Buffers）。

3.  **`micro_graph.h`**: 計算圖管理者。它負責依據模型定義的順序，依序執行計算圖中的每一個節點（Node/Op）。在 `IF` 和 `WHILE` 等控制流運算子中，`MicroGraph` 也負責調用相應的子圖。

4.  **`micro_context.h`**: 為運算元核心（Kernel）提供與 TFLM 框架互動的 API。當一個 Kernel 需要暫存記憶體或回報錯誤時，它會透過 `MicroContext` 來完成，這使得 Kernel 的實現與底層的解釋器和分配器解耦。

#### 記憶體管理 (`arena_allocator/` & `memory_planner/`)

這是 TFLM 的關鍵部分，因為微控制器上的記憶體非常寶貴。

*   **`arena_allocator/`**: 提供了記憶體分配器的具體實現。
    *   `ibuffer_allocator.h`: 定義了兩種緩衝區分配器的介面：`IPersistentBufferAllocator`（用於永久性記憶體，從 Arena 尾端分配）和 `INonPersistentBufferAllocator`（用於非永久性記憶體，如激活張量，從 Arena 頭端分配）。
    *   `single_arena_buffer_allocator.h`: 一個在單一 Arena 上同時實現上述兩種介面的分配器，是 TFLM 預設的記憶體分配策略。
    *   `non_persistent_arena_buffer_allocator.h` & `persistent_arena_buffer_allocator.h`: 將 Arena 分為兩個獨立區域，分別用於非永久性和永久性分配的分配器實現。
    *   `recording_single_arena_buffer_allocator.h`: 用於調試和分析記憶體使用情況的特殊版本。

*   **`memory_planner/`**: 負責規劃非永久性記憶體（主要是激活張量和暫存緩衝區）的佈局，以實現記憶體複用，從而最小化所需的 Arena 大小。
    *   `greedy_memory_planner.h`: 預設的規劃器，使用貪婪演算法來安排記憶體佈局。
    *   `non_persistent_buffer_planner_shim.h`: 允許使用離線（Offline）預先計算好的記憶體佈局，可以得到比貪婪演算法更優的結果，進一步減小記憶體佔用。

#### 運算元核心 (`kernels/`)

此目錄包含了 TFLM 支援的所有運算子（Ops）的具體實現，稱為 Kernel。

*   **參考實現**: 大部分的 `.cc` 檔案都是基於 TFLite 的參考實現，確保了可移植性和基礎功能。
*   **`cmsis_nn/`**: 這是一個非常重要的子目錄，它包含了使用 **CMSIS-NN** 函式庫進行優化的 Kernel 版本。CMSIS-NN 是 Arm 專為 Cortex-M 系列處理器設計的 ML 函式庫，可以大幅提升在這些裝置上的運算效能。當針對 Arm Cortex-M 平台編譯時，TFLM 會優先使用這些優化過的 Kernel。

#### 平台特定程式碼 (`cortex_m_corstone_300/`)

TFLM 具有良好的可移植性，但某些功能需要針對特定硬體平台進行實現。

*   `system_setup.h`: 提供了一個平台初始化的掛鉤（hook），例如初始化 UART 或硬體加速器。
*   `micro_time.h`: 提供計時功能，用於性能剖析（Profiling）。
*   `cortex_m_corstone_300/` 目錄就是一個範例，它為 Arm Corstone-300 平台提供了 `system_setup.cc` 和 `micro_time.cc` 的具體實現。

#### 工具與橋接層 (`tflite_bridge/`, `testing/`)

*   **`tflite_bridge/`**: 提供了一些橋接程式碼，用於處理 TFLite FlatBuffer 模型格式的解析和與 TFLM 執行時之間的轉換。
*   **`debug_log.h`, `micro_log.h`**: 提供了一個輕量級的日誌和錯誤回報系統。
*   **測試相關**: `fake_micro_context.h`, `mock_micro_graph.h`, `test_helpers.h` 等檔案是為了方便進行單元測試而設計的模擬（Mock）和輔助工具。

#### 總結

`@lite/micro` 的架構設計完全是為了適應資源受限的環境。它透過靜態記憶體分配（Tensor Arena）、可插拔的記憶體規劃策略、以及針對特定硬體的優化核心（如 CMSIS-NN），在提供完整 TensorFlow Lite 功能的同時，保持了極低的記憶體和程式碼佔用。這種設計使得在幾十 KB RAM 的微控制器上部署複雜的 AI 模型成為可能。
