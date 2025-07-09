
# TFLM Signal `tensorflow_core` 架構分析

## 總覽

`tensorflow_core` 目錄扮演著橋樑的角色，將 `src` 目錄中高效能、獨立的訊號處理函式庫，與標準的 TensorFlow 框架進行整合。其主要目的是將 `src` 中的 C++ 函式封裝成 TensorFlow 的運算元 (Ops)，讓這些功能可以在 TensorFlow 的圖模型中使用，無論是在伺服器端訓練，或是在其他 TensorFlow 執行環境中進行推論。

此目錄的架構清晰地分為兩大部分：

1.  **`ops/`**: 定義 TensorFlow 運算元的介面。
2.  **`kernels/`**: 提供這些運算元在 CPU 上的具體實作 (Kernel)。

這種分離是 TensorFlow 框架的標準作法，將「運算元的定義」與「運算元的實作」解耦。

## `ops` 目錄：運算元定義

`ops` 目錄中的每個 `*_op.cc` 檔案都使用 TensorFlow 的 `REGISTER_OP` 巨集來定義一個新的運算元。這個定義過程包含了以下幾個關鍵部分：

1.  **運算元名稱 (Op Name)**:
    每個運算元都有一個唯一的名稱，例如 `"SignalFramer"` 或 `"SignalRfft"`。這個名稱是在 TensorFlow Python API 或圖定義中使用的。

2.  **屬性 (Attributes)**:
    使用 `.Attr()` 來定義運算元的靜態參數。這些是在圖建構時期設定的常數，例如 `Framer` 的 `frame_size` 和 `frame_step`，或是 `Rfft` 的 `fft_length`。屬性可以是不同的資料類型 (int, float, bool, type)。

3.  **輸入 (Inputs)**:
    使用 `.Input()` 來定義運算元期望的輸入張量，包含名稱和資料類型。例如，`"input: int16"` 表示一個名為 `input` 且類型為 `int16` 的輸入。

4.  **輸出 (Outputs)**:
    使用 `.Output()` 來定義運算元產生的輸出張量，同樣包含名稱和資料類型。

5.  **形狀推斷函式 (Shape Inference Function)**:
    這是運算元定義中非常關鍵的一部分。透過 `.SetShapeFn()` 指定一個函式 (例如 `FramerShape`)，TensorFlow 在圖建構時期會呼叫此函式，根據輸入張量的形狀和運算元的屬性，來推斷輸出張量的形狀。這使得 TensorFlow 能夠在執行圖之前，就驗證圖的結構是否正確，並預先分配記憶體。
    例如，在 `RfftShape` 中，輸出形狀的最後一個維度被計算為 `((fft_length / 2) + 1) * 2`，以容納複數結果。

6.  **文件 (Documentation)**:
    使用 `.Doc()` 提供運算元的說明文件，解釋其功能、輸入、輸出和屬性。這對於 API 的可用性至關重要。

## `kernels` 目錄：運算元實作 (Kernels)

`kernels` 目錄中的每個 `*_kernel.cc` 檔案提供了對應 `ops` 中定義的運算元的 CPU 實作。這些實作是繼承自 `tensorflow::OpKernel` 的類別。

1.  **OpKernel 類別**:
    每個核心都是一個類別，例如 `FramerOp` 或 `RfftOp`。這個類別繼承了 `tensorflow::OpKernel`。

2.  **建構函式 (Constructor)**:
    核心的建構函式接收一個 `tensorflow::OpKernelConstruction* context` 物件。它主要的工作是：
    -   使用 `context->GetAttr()` 讀取在運算元定義中設定的屬性值。
    -   進行一次性的初始化工作，例如分配核心內部狀態所需的臨時張量 (使用 `context->allocate_temp`)。對於需要狀態的核心 (如 `Framer` 或 `Rfft`)，這是一個關鍵步驟，因為它將狀態與 TensorFlow 的資源管理整合在一起。

3.  **`Compute` 方法**:
    這是核心的執行主體。每次 TensorFlow 執行到這個運算元時，就會呼叫 `Compute` 方法。它的職責是：
    -   從 `tensorflow::OpKernelContext* context` 中取得輸入張量 (`context->input(i)`)。
    -   為輸出張量分配記憶體 (`context->allocate_output(i, ...)`）。
    -   **呼叫 `src` 函式庫**: 這是核心所在。`Compute` 方法會呼叫 `src` 目錄中對應的 C++ 函式來執行實際的訊號處理。例如，`FramerOp::Compute` 呼叫 `tflite::tflm_signal::CircularBufferWrite` 和 `CircularBufferGet`。
    -   將結果填入輸出張量。

4.  **核心註冊 (Kernel Registration)**:
    使用 `REGISTER_KERNEL_BUILDER` 巨集將 `OpKernel` 類別與特定的運算元名稱和裝置 (例如 `DEVICE_CPU`) 綁定。對於像 `Rfft` 這樣支援多種資料類型的運算元，會使用 `.TypeConstraint<T>("T")` 為每種支援的類型註冊一個模板化的核心實作。

## 架構模式與設計理念

-   **重用核心邏輯**: `tensorflow_core` 的設計完美地展示了程式碼重用。它沒有重新實作任何訊號處理演算法，而是聰明地將 `src` 目錄中已經過最佳化和測試的函式庫，封裝成 TensorFlow 生態系統可以使用的元件。

-   **狀態管理**: 對於像 `Framer` 或 `OverlapAdd` 這樣需要跨次呼叫保持狀態的運算元，其狀態 (例如 `CircularBuffer` 或重疊緩衝區) 是透過在 `OpKernel` 的建構函式或首次 `Compute` 呼叫中分配的 `Tensor` 來管理的。這確保了狀態的生命週期由 TensorFlow 框架控制，避免了記憶體洩漏，並使其與 TensorFlow 的執行模型相容。

-   **模板化與類型安全**: 透過 C++ 模板 (`template <typename T, DataType E, ...>`) 和 TensorFlow 的類型約束 (`TypeConstraint`)，像 `RfftOp` 這樣的核心可以用一份程式碼優雅地支援多種資料類型 (float, int16, int32)，同時保持類型安全。

## 結論

`tensorflow_core` 目錄是一個教科書級的範例，展示了如何將一個獨立的、高效能的 C++ 函式庫，整合到一個大型的、可擴展的框架 (如 TensorFlow) 中。它嚴格遵循 TensorFlow 的運算元和核心設計模式，透過 `ops` 和 `kernels` 的分離，以及 `REGISTER_OP` 和 `REGISTER_KERNEL_BUILDER` 的機制，成功地將 `tflm_signal` 的強大功能暴露給了 TensorFlow 的使用者，讓這些專為微控制器設計的演算法也能在標準的 TensorFlow 環境中被使用和驗證。
