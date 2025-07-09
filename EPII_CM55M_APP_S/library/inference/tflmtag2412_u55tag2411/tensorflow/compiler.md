### TensorFlow Lite 編譯器 (`@compiler`) 架構說明

`@compiler` 目錄包含了 TensorFlow Lite 的模型轉換與優化工具，其核心是基於 MLIR (Multi-Level Intermediate Representation) 的編譯器後端。MLIR 提供了一個可擴展的基礎設施來建構特定領域的編譯器。此目錄下的程式碼主要負責將 TensorFlow 模型轉換為 TensorFlow Lite 的 FlatBuffer 格式（`.tflite`），並進行各種優化。

#### 主要子目錄結構與功能：

1.  **`mlir/lite/`**: 這是 TFLite MLIR 編譯器的主要程式碼所在地。
    *   **`core/`**: 存放編譯器的核心元件和 API。
        *   `api/`: 定義了編譯器對外的主要介面。例如 `error_reporter.h` 提供了一個錯誤回報機制，是任何編譯器不可或缺的基礎功能，允許開發者自訂錯誤處理方式。
        *   `c/`: 包含 C 語言的資料結構定義，這些定義是 TFLite 模型與底層實現之間的橋樑。
            *   `tflite_types.h`: 定義了 TFLite 中使用的基本資料型別，如 `TfLiteType` (e.g., `kTfLiteFloat32`, `kTfLiteInt8`)，這是描述張量 (Tensor) 資料型別的基礎。
            *   `builtin_op_data.h`: 定義了所有 TFLite 內建運算子（Operator）的參數結構。例如，`TfLiteConvParams` 用於描述卷積層的步長（stride）、填充（padding）等參數。編譯器需要這些結構來解析和處理模型中的每一個運算。

    *   **`kernels/`**: 存放與運算子核心（Kernel）相關的程式碼。Kernel 是運算子的具體實現。此目錄下的檔案可能包含輔助函式或內部定義，供 Kernel 實現使用。
        *   `internal/`: 包含供內部使用的輔助工具，例如 `compatibility_macros.h` 用於處理不同版本之間的相容性問題。

    *   **`schema/`**: 存放與 TFLite 模型檔案格式（Schema）相關的程式碼。TFLite 使用 FlatBuffers 作為其序列化格式，Schema 定義了模型的結構。
        *   `schema_generated.h`: 這是由 FlatBuffers 的 Schema 定義檔（`.fbs`）自動產生的 C++ 標頭檔。它包含了所有模型結構（如 `Model`, `OperatorCode`, `Tensor`）的 C++ 類別，使得程式可以直接讀取和操作 `.tflite` 檔案的內容。
        *   `schema_utils.h` 和 `schema_utils.cc`: 提供了一些輔助函式來處理 Schema。例如，`GetBuiltinCode` 函式解決了不同 TFLite Schema 版本之間 `builtin_code` 欄位的相容性問題，確保無論是舊版還是新版的模型都能被正確解析。

#### 總結

`@compiler` 目錄的架構清晰地分離了不同的功能：
- **`core`** 提供基礎 API 和資料結構。
- **`kernels`** 處理運算子的實現細節。
- **`schema`** 負責模型的序列化格式和解析。

整個架構圍繞 MLIR 框架，旨在提供一個高效、可擴展的工具鏈，將高階的機器學習模型轉換為適合在邊緣裝置上運行的輕量化 TFLite 格式。
