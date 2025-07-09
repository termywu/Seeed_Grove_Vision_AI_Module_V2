
# TFLM Signal `src` Architecture Analysis

## Overview

The `src` directory contains the core, hardware-agnostic signal processing algorithms that form the building blocks for the TensorFlow Lite for Microcontrollers (TFLM) signal processing operators defined in the `micro/kernels` directory. This library is written in C++ with a strong emphasis on performance, portability, and suitability for resource-constrained embedded systems. It avoids dynamic memory allocation and provides implementations for various data types (float, int16, int32) to cater to different hardware and performance requirements.

A key architectural pattern is the use of C-style functions that operate on raw pointers and configuration structs. This makes the library easy to integrate into any C/C++ project and allows for both stateful and stateless operations.

## Core Components

The library can be broken down into several functional categories:

### 1. FFT and Spectral Processing

This is the heart of the library, providing Fast Fourier Transform capabilities. The implementation is a clever wrapper around the **KissFFT** library.

- **`kiss_fft_wrappers/`**: This directory is a prime example of the library's thoughtful architecture. To support FFT operations on multiple data types (`float`, `int16_t`, `int32_t`) within the same application without linker conflicts, the KissFFT source code is included directly and wrapped in distinct C++ namespaces (`kiss_fft_float`, `kiss_fft_fixed16`, `kiss_fft_fixed32`). The `FIXED_POINT` macro is used to configure the KissFFT build for integer types.

- **`rfft_*.cc` & `irfft_*.cc`**: These files provide a clean API for Real FFT (`rfft`) and its inverse (`irfft`). They use the namespaced KissFFT implementations internally. The API is split into `Init`, `GetNeededMemory`, and `Apply` functions, a common pattern for managing stateful operations on microcontrollers.
  - `RfftInt16GetNeededMemory`, `RfftInt16Init`, `RfftInt16Apply`
  - `IrfftFloatGetNeededMemory`, `IrfftFloatInit`, `IrfftFloatApply`

- **`fft_auto_scale.cc`**: Provides a utility to analyze an input signal and determine the optimal bit shift to maximize its dynamic range before performing a fixed-point FFT, preventing overflow and preserving precision.

- **`energy.cc`**: Computes the power/energy of each frequency bin in a spectrum (`real^2 + imag^2`).

### 2. Audio Feature Extraction Pipeline (Filter Bank)

A set of components designed to work together to create audio spectrograms, often used in keyword spotting and voice recognition.

- **`filter_bank.cc`**: Implements a triangular filter bank, which is used to group FFT frequency bins into a smaller number of channels (e.g., Mel-scale channels).
- **`filter_bank_square_root.cc`**: Applies a square root to the energy of each filter bank channel. This is a common step in generating magnitude or power spectrograms.
- **`filter_bank_log.cc`**: Applies a natural logarithm to the filter bank energies, a necessary step for creating log-Mel spectrograms, which are perceptually more similar to how humans hear.

### 3. Noise Reduction and Normalization

These components are used to clean up the signal and normalize its energy.

- **`filter_bank_spectral_subtraction.cc`**: A basic but effective noise reduction technique. It estimates the noise floor of the signal and subtracts it from each filter bank channel.
- **`pcan_argc_fixed.cc`**: Implements a fixed-point Per-Channel Energy Normalization (PCAN) with Automatic Gain Control (AGC). This is a more advanced normalization technique that adapts to changing signal levels, making the model more robust to variations in volume.

### 4. Framing and Data Management

- **`circular_buffer.cc`**: A highly optimized circular buffer implementation for `int16_t` data. It's a fundamental component used by streaming operators like `Framer` and `Delay` in the `micro/kernels` directory to manage incoming audio samples efficiently.
- **`overlap_add.cc`**: Implements the overlap-add method, which is essential for reconstructing a continuous time-domain signal after it has been processed in overlapping frames (e.g., after an IFFT).
- **`window.cc`**: Applies a window function (like Hann or Hamming) to a frame of data before the FFT. This reduces spectral leakage.

### 5. Core Data Structures and Math Utilities

These are the foundational pieces used throughout the library.

- **`complex.h`**: Defines a simple `Complex<T>` struct. This avoids a dependency on the standard C++ `<complex>` header, which may not be available or optimized for all embedded toolchains.
- **`log.cc`**: A fast, lookup-table-based integer implementation of the natural logarithm.
- **`square_root*.cc`**: Integer square root implementations for 32-bit and 64-bit inputs.
- **`msb*.cc`**: Functions to find the most significant bit of a number, often used for normalization and scaling. These files contain platform-specific optimizations (e.g., for Xtensa using `XT_NSAU`).
- **`max_abs.cc`**: Finds the maximum absolute value in an array, another key function for auto-scaling. This also contains Xtensa-specific optimizations.

## Architectural Patterns

- **Modularity**: Each signal processing stage is a self-contained unit with a clear API, a- **State Management**: Stateful operations (like FFTs or circular buffers) use an `Init` function that prepares a state structure in a pre-allocated memory region. This avoids hidden dynamic memory allocation.
- **Data-Type Genericity**: While not using C++ templates in the top-level API, the library provides parallel implementations for different data types (e.g., `rfft_int16.cc`, `rfft_int32.cc`, `rfft_float.cc`). The KissFFT wrapper is a prime example of how this is managed cleanly.
- **Performance Focus**: The code uses low-level C++ and includes platform-specific assembly or intrinsics (`#if defined(XTENSA)`) for critical math functions, demonstrating a clear focus on performance for embedded targets.
