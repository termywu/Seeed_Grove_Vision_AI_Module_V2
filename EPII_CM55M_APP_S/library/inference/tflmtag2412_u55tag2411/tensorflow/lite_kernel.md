# `@lite/kernels/**` 程式架構說明

## 總體概述

`@lite/kernels/` 目錄核心功能是提供 TensorFlow Lite for Microcontrollers (TFLM) 中各種神經網路運算子 (Operator, Op) 的具體實現 (Kernel Implementation)。這些實現涵蓋了從基本的數學運算到複雜的神經網路層，如卷積、池化等。

## 程式架構分析

這個目錄的結構設計旨在實現 **可移植性** 和 **性能優化** 的平衡。主要可以分為以下幾個部分：

### 1. 頂層通用工具與定義

*   `kernel_util.h` / `.cc`: 提供了一系列輔助函式，用於在 Kernel 實現中方便地訪問和操作 `TfLiteTensor`，例如獲取輸入/輸出張量的形狀和數據。
*   `op_macros.h`: 定義了 TFLM 中常用的宏，如 `TFLITE_DCHECK`，用於斷言和錯誤檢查，確保程式的穩健性。
*   `padding.h`: 提供了計算卷積、池化等操作中邊緣填充（Padding）的相關函式，這是圖像處理相關運算中的常見步驟。

### 2. `internal/` 目錄 (核心實現)

這是 Kernel 實現的核心所在，包含了絕大部分的演算法邏輯。

*   **通用基礎 (`common.h`, `types.h`, `quantization_util.h`)**:
    *   `common.h` / `.cc`: 定義了跨越多個 Kernel 的通用函式和資料結構，例如定點數數學運算 (`MultiplyByQuantizedMultiplier`)、啟動函數 (`ActivationFunctionWithMinMax`) 等。
    *   `types.h`: 定義了各種運算子的參數結構，如 `ConvParams`, `PoolParams`, `DepthwiseParams` 等，用於在 Kernel 內部傳遞配置信息。
    *   `quantization_util.h` / `.cc`: 包含了量化計算相關的工具函式，是實現量化神經網路的關鍵，例如將浮點數量化為定點數的 `QuantizeMultiplier`。

*   **`reference/` (參考實現)**:
    *   **此目錄至關重要**，它包含了所有 TFLM 運算子的 **純 C++ 參考實現**。
    *   這些實現是 **可移植的 (portable)**，不依賴任何特定的硬體加速指令（如 NEON），確保了 TFLM 可以在任何支援 C++ 的平台上運行。
    *   它們是功能正確性的基準，也是新平台移植時的基礎版本。
    *   `integer_ops/` 子目錄專門存放 **整數量化** 運算子的參考實現，這是微控制器上最常用的類型。

*   **`optimized/` (優化實現)**:
    *   此目錄用於存放針對特定硬體平台（如 ARM Cortex-M 搭配 NEON/MVE）的 **優化版本** Kernel。
    *   例如，它可能包含使用 NEON 或 MVE 指令集重寫的 Kernel，以大幅提升性能。
    *   `neon_check.h` 這類文件用於在編譯時檢查當前平台是否支持 NEON，從而決定是否啟用優化版本的程式碼。

### 架構總結

`@lite/kernels/` 的架構設計清晰地分離了 **通用工具**、**可移植的參考實現** 和 **平台相關的優化實現**。這種分層設計帶來了以下好處：

*   **高可移植性**: 只要有 C++ 編譯器，`reference/` 目錄下的程式碼就可以確保 TFLM 的核心功能在任何微控制器上運行。
*   **高性能**: 對於追求極致性能的平台，可以在 `optimized/` 目錄下提供客製化的實現，而無需修改上層邏輯。
*   **易於維護和擴展**: 新的運算子可以先實現參考版本以確保功能正確，然後再逐步為不同平台添加優化版本。

總之，這是一個兼顧了廣泛適用性和極致性能的務實架構。
