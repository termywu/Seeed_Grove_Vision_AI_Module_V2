# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the firmware codebase for the Seeed Grove Vision AI Module V2, built on the Himax WiseEye2 (WE2) platform. The project supports ARM Cortex-M55 with Ethos-U55 NPU for edge AI applications with camera/sensor integration, machine learning inference, and various communication protocols.

### Hardware Platform
- **Processor**: Himax WiseEye2 (WE2) with ARM Cortex-M55 and Ethos-U55 NPU
- **Memory**: Support for external PSRAM and flash memory
- **Sensors**: Camera sensors (HM01B0, HM0360, HM11B1, HM2140, IMX219, IMX477, OV5647), IMU (ICM42688), PDM microphones
- **Package Options**: LQFP128, WLCSP65, QFN88, BGA64
- **Interfaces**: UART, SPI, I2C, I2S, GPIO

### Software Stack
- **Languages**: C/C++ (embedded firmware)
- **Build System**: GNU Make with cross-compilation support
- **Toolchains**: ARM Compiler 6, GNU ARM Embedded Toolchain
- **RTOS**: FreeRTOS 10.5.1 with TrustZone security support
- **AI Framework**: TensorFlow Lite Micro with CMSIS-NN 7.0.0 optimization
- **Libraries**: CMSIS-DSP, CMSIS-CV, Audio processing, Image processing, JPEG encoding
- **Security**: ARM TrustZone, Secure Boot, Hardware crypto acceleration

## Key Architecture

### Core Structure
- **EPII_CM55M_APP_S/**: Main firmware application directory
  - **app/**: Application layer with scenario-specific implementations
  - **app/scenario_app/**: Example applications and use cases
  - **drivers/**: Hardware abstraction layer for platform peripherals
  - **library/**: Reusable libraries (CMSIS-NN, TensorFlow Lite Micro, etc.)
  - **device/**: Device-specific startup and system files
  - **os/**: Real-time operating system support (FreeRTOS)
  - **board/**: Board-specific configurations
  - **trustzone/**: ARM TrustZone security configuration
  - **makefile**: Main build configuration

### Development Tools
- **build.sh**: Docker-based build script
- **flash.sh**: Firmware flashing script using XMODEM
- **xmodem/**: XMODEM protocol implementation for firmware upload
- **we2_image_gen_local/**: Image generation and secure boot tools
- **swd_debugging/**: SWD debugging tools and configurations

### Models and Examples
- **model_zoo/**: Pre-trained AI models for different applications

### Application Types
The system uses a scenario-based architecture where each application type is a complete use case:
- **Basic TensorFlow Lite**: `allon_sensor_tflm` (person detection)
- **CMSIS-NN Optimized**: `allon_sensor_tflm_cmsis_nn`
- **FreeRTOS Multitasking**: `allon_sensor_tflm_freertos`
- **FAT Filesystem**: `allon_sensor_tflm_fatfs`
- **Object Detection**: `tflm_yolov8_od`, `tflm_yolo11_od`
- **Face Detection**: `tflm_fd_fm` (face detection and facial mesh)
- **Pose Estimation**: `tflm_yolov8_pose`
- **Gender Classification**: `tflm_yolov8_gender_cls`
- **People Detection**: `tflm_peoplenet` (NVIDIA TAO)
- **Audio Processing**: `pdm_record`, `kws_pdm_record` (keyword spotting)
- **IMU Sensor**: `imu_read` (ICM42688)
- **JPEG Encoding**: `allon_jpeg_encode`
- **Edge Impulse**: `edge_impulse_firmware`, `ei_standalone_inferencing`, `ei_standalone_inferencing_camera`
- **CMSIS Libraries**: `hello_world_cmsis_dsp`, `hello_world_cmsis_cv`
- **FreeRTOS**: `hello_world_freertos_tz_s_only`
- **FAT Filesystem**: `fatfs_mmc_spi`

### Build System Configuration
The build system is controlled by the main makefile variables:
- `APP_TYPE`: Selects the scenario application (default: `allon_sensor_tflm`)
- `BOARD`: Target board (default: `epii_evb`)
- `TOOLCHAIN`: Compiler toolchain (`gnu` or `arm`)
- `IC_PACKAGE_SEL`: IC package type (default: `WLCSP65`)
- `TRUSTZONE`: Enable TrustZone security (default: `y`)
- `OS_SEL`: Operating system selection (default: `freertos`)
- `LIB_CMSIS_NN_ENALBE`: Enable CMSIS-NN optimizations
- `LIB_CMSIS_NN_VERSION`: CMSIS-NN version (default: `7_0_0`)
- `CUSTOMER`: Customer-specific configurations (default: `seeed`)
- `CIS_SEL`: Camera sensor selection
- `OLEVEL`: Optimization level (default: `O2`)
- `DEBUG`: Debug mode (default: `1`)

## Common Development Commands

### Build Commands
```bash
# Build in EPII_CM55M_APP_S directory
make clean    # Clean build artifacts
make          # Build current APP_TYPE
make all      # Full build

# Build with specific app type
make APP_TYPE=tflm_yolov8_od

# Build with CMSIS-NN optimizations
make LIB_CMSIS_NN_ENALBE=1

# Using Docker (Recommended)
./build.sh
./build.sh clean all
```

### Generate Firmware Image
```bash
# After building, generate flashable image
cd ../we2_image_gen_local/
cp ../EPII_CM55M_APP_S/obj_epii_evb_icv30_bdv10/gnu_epii_evb_WLCSP65/EPII_CM55M_gnu_epii_evb_WLCSP65_s.elf input_case1_secboot/
./we2_local_image_gen project_case1_blp_wlcsp.json
# Output: ./output_case1_sec_wlcsp/output.img
```

### Flash Firmware
```bash
# Prerequisites
pip install -r xmodem/requirements.txt
sudo setfacl -m u:[USERNAME]:rw /dev/ttyACM0  # Linux permissions

# Using xmodem protocol
python3 xmodem/xmodem_send.py --port=/dev/ttyACM0 --baudrate=921600 --protocol=xmodem --file=we2_image_gen_local/output_case1_sec_wlcsp/output.img

# Flash with model
python3 xmodem/xmodem_send.py \
  --port=/dev/ttyACM0 \
  --baudrate=921600 \
  --protocol=xmodem \
  --file=we2_image_gen_local/output_case1_sec_wlcsp/output.img \
  --model="model_zoo/tflm_yolov8_od/yolov8n_od_192_delete_transpose_0xB7B000.tflite 0xB7B000 0x00000"

# Using convenience script
./flash.sh

# Using Edge Impulse CLI
himax-flash-tool -d WiseEye2 -f output.img
```

### Development Scripts
- `build.sh`: Containerized build script
- `flash.sh`: Flash firmware to device
- `gh_flash.sh`: GitHub Actions flash script
- `init_claude.sh`: Environment setup

## Camera Sensor Support

To add support for different cameras, modify the `CIS_SUPPORT_INAPP_MODEL` in the app's `.mk` file:
- `cis_ov5647`: OV5647 camera (default)
- `cis_imx219`: IMX219 Raspberry Pi camera
- `cis_imx477`: IMX477 Raspberry Pi HQ camera
- `cis_imx708`: IMX708 Raspberry Pi camera
- `cis_hm0360`: HM0360 camera sensor
- `cis_hm01b0`: HM01B0 camera sensor
- `cis_hm11b1`: HM11B1 camera sensor
- `cis_hm2140`: HM2140 camera sensor

## Machine Learning Framework

The codebase supports TensorFlow Lite Micro with optional CMSIS-NN optimizations:
- **TFLite Models**: Stored in `model_zoo/` directory
- **CMSIS-NN**: Enable with `LIB_CMSIS_NN_ENALBE=1` for ARM Cortex-M55 optimizations
- **Model Conversion**: Use Vela compiler for Ethos-U55 NPU optimization
- **Model Placement**: Models stored in flash memory starting from address `0x200000`
- **Memory Layout**: First 2MB (0x000000-0x200000) reserved for firmware
- **Address Alignment**: Model addresses must be 4KB aligned
- **Optimization**: Models are pre-quantized (INT8) and Vela-optimized
- **Multiple Models**: Multiple models can be flashed simultaneously to different addresses

## Output Files

Build artifacts are located in:
- **ELF**: `obj_epii_evb_icv30_bdv10/gnu_epii_evb_WLCSP65/EPII_CM55M_gnu_epii_evb_WLCSP65_s.elf`
- **Firmware Image**: `we2_image_gen_local/output_case1_sec_wlcsp/output.img`
- **Libraries**: `prebuilt_libs/gnu/` or `prebuilt_libs/arm/`

## Serial Communication

Default UART settings for debugging and flashing:
- **Baud Rate**: 921600
- **Data**: 8 bit
- **Parity**: none
- **Stop**: 1 bit
- **Flow Control**: none

## Best Practices and Development Guidelines

### Code Organization
- Keep application-specific code in `app/scenario_app/[app_name]/`
- Use `common_config.h` for application-specific configurations
- Follow existing naming conventions for consistency
- Separate hardware abstraction from application logic
- Use `.mk` files for modular build configuration
- Implement `cvapp_*.cpp` for computer vision processing logic

### Memory Management
- Be mindful of memory constraints (limited SRAM)
- Use external PSRAM for large data structures
- Optimize model sizes using quantization and pruning
- Define memory layouts in linker scripts (`.ld` for GCC, `.sct` for ARM)
- Consider TCM (Tightly Coupled Memory) for performance-critical code
- Use `memory_manage.c` for dynamic memory allocation strategies

### Security Considerations
- Use TrustZone for secure/non-secure partitioning
- Implement secure boot for production deployments using `secureboot_tool/`
- Store sensitive data in secure memory regions
- Use hardware crypto acceleration (CC312 cryptographic engine)
- Configure Memory Protection Controller (MPC) and Peripheral Protection Controller (PPC)

### Performance Optimization
- Enable CMSIS-NN 7.0.0 for neural network acceleration
- Use Ethos-U55 NPU for inference acceleration with Vela-optimized models
- Optimize data paths for camera and sensor processing
- Consider power management for battery-powered applications
- Use Helium (MVE) instructions for SIMD operations
- Enable compiler optimizations (`OLEVEL = O2`)

### Testing and Debugging
- Use SWD debugging tools in `swd_debugging/` with pyOCD
- Enable debug logging with `DBG_APP_LOG` and `FRAME_CHECK_DEBUG`
- Test with Himax AI web toolkit for UART visualization
- Validate models with known test datasets
- Use `hardfault_handler.c` for crash debugging

## Common Issues and Solutions

### Build Issues
- **Toolchain Path**: Ensure correct toolchain path configuration in makefile
- **Missing Dependencies**: Check makefile variables for target platform
- **Linker Errors**: Clean build directory (`make clean`) if encountering link errors
- **Memory Overflow**: Adjust linker script memory sections if code/data doesn't fit
- **CMSIS-NN Version**: Ensure `LIB_CMSIS_NN_VERSION` matches model requirements

### Flashing Issues
- **Device Permissions**: Set proper permissions (`sudo setfacl -m u:[USERNAME]:rw /dev/ttyACM0`)
- **Bootloader Mode**: Ensure device is in bootloader mode (hold boot button while resetting)
- **COM Port**: Verify correct COM port identification
- **Cable Quality**: Check USB cable connections and power supply stability
- **Reset Sequence**: Always reset device after successful flash operation

### Model Issues
- **Vela Optimization**: Verify model is Vela-optimized for Ethos-U55 NPU
- **Memory Constraints**: Check model size fits in allocated flash space
- **Address Alignment**: Ensure model address alignment (4KB boundaries)
- **Quantization**: Validate model format and INT8 quantization
- **Model Loading**: Check `common_config.h` for correct model flash addresses

## Integration with External Tools

### Edge Impulse
- Support for Edge Impulse firmware integration
- Use `edge_impulse_firmware` and `ei_standalone_inferencing` apps
- Compatible with Edge Impulse Studio for model training

### CMSIS Libraries
- **CMSIS-DSP**: Signal processing functions (FFT, filters, statistics)
- **CMSIS-CV**: Computer vision operations (image processing, feature extraction)
- **CMSIS-NN**: Neural network acceleration (convolution, pooling, activation)

### Development Tools
- **Vela Compiler**: NPU optimization for Ethos-U55
- **pyOCD**: SWD debugging and programming
- **Himax AI Web Toolkit**: Real-time visualization and testing
- **Docker**: Containerized build environment

## Important Notes

- Always build in the `EPII_CM55M_APP_S` directory
- The build system automatically selects appropriate libraries based on configuration
- TrustZone is enabled by default (`TRUSTZONE=y`)
- FreeRTOS is the default OS (`OS_SEL=freertos`)
- GNU toolchain is recommended for development
- Models start at flash address `0x200000` with 4KB alignment
- Use Vela compiler for Ethos-U55 NPU optimization
- Enable CMSIS-NN 7.0.0 for maximum performance