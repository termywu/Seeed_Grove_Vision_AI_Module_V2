# `@lite/core/**` 程式架構說明

## 總體概述

`@lite/core/**` 目錄是 TensorFlow Lite for Microcontrollers (TFLM) 的最核心部分，它定義了 TFLM 的基礎 API、資料結構和執行框架。這個目錄是整個 TFLM 運作的基石，負責模型的解析、執行流程的控制以及與底層硬體和作業系統的互動。

## 程式架構分析

這個目錄的結構主要圍繞著定義 TFLM 的核心抽象和提供與 FlatBuffers 模型文件進行互動的介面。

### 1. `api/` (核心 API)

此目錄定義了 TFLM 的核心抽象介面，是框架的基礎。

*   `error_reporter.h` / `.cc`: 定義了一個抽象的錯誤報告介面 (`ErrorReporter`)。TFLM 的所有元件都透過這個介面來報告錯誤，而不是直接使用 `printf` 或 `log`。這使得錯誤處理機制可以被輕易地替換，以適應不同的硬體平台（例如，將錯誤訊息輸出到 UART、日誌文件或除錯器）。
*   `flatbuffer_conversions.h` / `.cc`: 這是連接模型文件和執行時的橋樑。它包含了一系列函式，負責解析由 [FlatBuffers](https://google.github.io/flatbuffers/) 序列化的 TensorFlow Lite 模型 (`.tflite` 檔案)。這些函式會將模型中定義的運算元、張量、量化參數等資訊，轉換為 TFLM 執行時所使用的 C 語言資料結構（如 `TfLiteTensor`, `TfLiteConvParams` 等）。這個過程是模型載入和初始化的關鍵步驟。
*   `tensor_utils.h` / `.cc`: 提供了一些對張量進行操作的實用工具函式，例如變數張量的重置 (`ResetVariableTensor`)。

### 2. `c/` (C 語言核心定義)

為了最大的可移植性，TFLM 的核心資料結構和 API 都是用 C 語言定義的。

*   `common.h` / `.cc`: 這是 TFLM 中 **最重要** 的頭文件之一。它定義了所有核心的 C 語言資料結構，包括：
    *   `TfLiteContext`: 執行上下文，提供了運算元 (Kernel) 在執行時所需的一切資訊和工具，如張量訪問、錯誤報告、記憶體分配等。
    *   `TfLiteTensor`: 張量的執行時表示，包含了數據類型、形狀 (dims)、量化參數和指向數據緩衝區的指標。
    *   `TfLiteNode`: 計算圖中一個節點（即一個運算元）的表示，包含了輸入、輸出張量的索引。
    *   `TfLiteRegistration`: 一個運算元的 "註冊" 結構，包含了指向該運算元 `init`, `prepare`, `invoke` 等實現函式的指標。
    *   `TfLiteDelegate`: 用於將部分計算委派給硬體加速器（如 DSP、GPU）的代理機制。
*   `builtin_op_data.h`: 定義了所有內建運算元（如卷積、池化）的參數結構。這些結構在 `flatbuffer_conversions.cc` 中被填充，並在運算元的 `prepare` 和 `invoke` 階段被使用。
*   `c_api_types.h`: 定義了 TFLM C API 中使用的基本類型，如 `TfLiteStatus`，用於表示函式執行的成功或失敗狀態。

### 3. `macros.h`

*   定義了一些在整個 TFLM 程式碼庫中廣泛使用的宏，例如 `TFLITE_NOINLINE` 用於防止函式內聯，`TFLITE_ATTRIBUTE_WEAK` 用於定義弱符號，這在嵌入式系統中對於提供可選的平台特定實現非常有用。

## 架構總結

`@lite/core/` 目錄的架構體現了 TFLM 的設計哲學：

*   **C 語言核心**: 核心執行框架和資料結構都使用 C 語言定義，確保了最大的可移植性和與各種嵌入式系統的相容性。
*   **抽象與解耦**:
    *   `ErrorReporter` 的設計將錯誤報告與具體實現解耦。
    *   `TfLiteRegistration` 使得新增一個運算元只需要實現一組標準介面函式，而無需修改核心框架。
    *   `TfLiteDelegate` 提供了一個標準化的方式來整合硬體加速。
*   **模型與執行時分離**: `flatbuffer_conversions` 模組清晰地劃分了從靜態模型文件解析數據到建立執行時資料結構的過程，使得執行時本身不直接依賴於 FlatBuffers 的具體格式。

總之，`@lite/core/` 構建了一個輕量、可擴展且高度可移植的深度學習推論框架，使其能夠在資源極其受限的微控制器上高效運行。
