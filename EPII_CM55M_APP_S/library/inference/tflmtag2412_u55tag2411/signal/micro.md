✦ TFLM Signal micro 執行的架構分析

  總覽


  micro 目錄包含專為在資源受限的微控制器環境中與 TensorFlow Lite for Microcontrollers (TFLM) 函式庫一起使用的訊號處理核心。這些核心被封裝成 TFLM
  操作，可用於建構音訊前端處理管線，例如用於語音辨識或聲音事件偵測的管線。

  此架構強調效率、模組化以及針對特定硬體目標（如 Xtensa DSP）進行最佳化的能力。

  目錄結構


  主要的程式碼位於 micro/kernels/ 目錄中。



    1 micro/
    2 └── kernels/
    3     ├── BUILD
    4     ├── delay.cc
    5     ├── delay.h
    6     ├── delay_test.cc
    7     ├── delay_flexbuffers_generated_data.cc
    8     ├── delay_flexbuffers_generated_data.h
    9     ├── ... (其他核心檔案)
   10     └── xtensa/
   11         ├── fft_auto_scale_kernel.cc
   12         ├── filter_bank_square_root.cc
   13         └── xtensa_square_root.S



   - `*.cc` / `*.h`: 這些是核心的核心實作檔案。每個檔案對（例如 delay.cc 和 delay.h）通常定義一個訊號處理操作。
   - `*_test.cc`: 這些是對應核心的單元測試。它們使用 TFLM 的測試框架來驗證核心的正確性。
   - `*_flexbuffers_generated_data.cc` / `*.h`: 這些檔案包含由 FlexBuffers 序列化的預編譯資料。TFLM 核心使用此資料進行初始化，允許在執行時期設定參數（例如
     delay_length），而無需重新編譯程式碼。
   - `BUILD`: 這是一個 Bazel 建置檔案，定義了如何編譯和連結程式庫與測試。
   - `xtensa/`: 此子目錄包含針對 Xtensa DSP 架構的特定硬體最佳化。這展示了此函式庫的設計，允許插入特定於平台的加速實作。

  核心架構


  每個訊號處理核心都遵循標準的 TFLM 運算元模式。一個運算元由一組函式指標定義，這些函式指標處理運算元的生命週期的各個階段。

  以 delay.cc 為例：


   1. 註冊 (Registration):
      每個核心都有一個註冊函式，例如 Register_DELAY()。此函式會回傳一個 TFLMRegistration 結構，其中包含指向核心方法的指標。



   1     TFLMRegistration* Register_DELAY() {
   2       static TFLMRegistration r = micro::RegisterOp(DelayInit, DelayPrepare,
   3                                                     DelayEval, nullptr, DelayReset);
   4       return &r;
   5     }



   2. 初始化 (Init):
      DelayInit 函式在模型準備期間被呼叫一次。它的主要職責是：
       - 從 Flexbuffer 中解析運算元參數（例如 delay_length）。
       - 分配一個包含運算元設定和狀態的「參數」結構 (TFLMSignalFrontendDelayParams)。此記憶體是持續性的，在運算元的整個生命週期內都存在。


   3. 準備 (Prepare):
      DelayPrepare 函式在每次 TFLM 解譯器準備執行時被呼叫。它負責：
       - 驗證輸入和輸出張量的形狀和類型。
       - 計算運算元所需的任何執行時期值。
       - 為核心的狀態分配持續性記憶體。對於 Delay 核心，這包括為每個輸入通道分配一個 CircularBuffer 和其底層的狀態緩衝區。


   4. 評估 (Eval):
      DelayEval 是執行核心運算的核心函式。它在每次解譯器執行圖時被呼叫。
       - 它接收輸入張量並產生輸出張量。
       - 對於 Delay 核心，它使用 CircularBuffer 來寫入新的輸入樣本，並讀取延遲的樣本到輸出中。


   5. 重設 (Reset):
      DelayReset 函式（如果提供）允許重設核心的內部狀態，而無需重新分配記憶體。對於 Delay 核心，它會重設 CircularBuffer
  並將其填滿零，以準備處理新的音訊串流。

  測試


  核心是使用 TFLM 的測試工具進行單元測試的。以 delay_test.cc 為例：


   - `KernelRunner`: tflite::micro::KernelRunner 類別被用來簡化設定和執行核心以進行測試的過程。它處理張量、記憶體分配和核心生命週期方法的呼叫。
   - 測試案例: 測試案例是使用 TF_LITE_MICRO_TEST 巨集定義的。每個測試都涵蓋了特定的條件，例如不同的輸入形狀、參數值和邊界情況。
   - 黃金資料 (Golden Data): 測試將核心的輸出與一組預先計算的「黃金」值進行比較，以確保正確性。
   - Flexbuffers 資料: 測試使用從 *_flexbuffers_generated_data.h 檔案匯入的 g_gen_data_* 陣列來初始化核心，模擬從模型檔案中讀取參數的過程。

  硬體加速

  kernels/xtensa/ 目錄的存在表明此函式庫支援特定於硬體的最佳化。

  以 xtensa/fft_auto_scale_kernel.cc 為例：


   - 平台偵測: 程式碼使用前置處理器巨集，如 #if XCHAL_HAVE_HIFI3，在編譯時期偵測目標硬體是否支援特定的指令集（在本例中為 HiFi3 DSP）。
   - 內建函式 (Intrinsics): 在偵測到受支援的硬體時，會使用特定於平台的內建函式（例如 ae_int16x4、AE_L16X4_IP）來取代可攜式的 C++
     程式碼。這些內建函式直接對應到高效的 DSP 指令（SIMD），從而顯著提高效能。
   - 條件式註冊: Register_FFT_AUTO_SCALE 函式會註冊 Eval 函式的適當版本——如果可用，則為 Xtensa
     最佳化版本，否則為可攜式版本。這使得函式庫的使用者可以無縫地從硬體加速中受益，而無需更改其應用程式程式碼。
   - 組合語言: 對於需要極致控制的情況，如 xtensa_square_root.S 所示，可以直接使用組合語言來實作效能關鍵的常式。

  結論


  micro 函式庫的架構設計良好，適用於嵌入式訊號處理。它遵循 TFLM 的標準運算元設計，使其易於整合。Flexbuffers
  的使用提供了執行時期配置的靈活性，而測試套件則確保了可靠性。最重要的是，透過為 Xtensa 等平台提供專門的後端，此架構能夠在不犧牲可攜性的情況下實現高效能。