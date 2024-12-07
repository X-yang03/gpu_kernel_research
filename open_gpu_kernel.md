## 参考英伟达开源的内核态GPU驱动[NVIDIA/open-gpu-kernel-modules: NVIDIA Linux open GPU kernel module source](https://github.com/NVIDIA/open-gpu-kernel-modules/tree/main)

### 1. 22年5月发布，支持Turing和Ampere等较新的架构

### 2. 用户态驱动（OpenGL, CUDA）等仍是闭源，难以进行逆向等操作，且最终都是要通过内核态来进行显存，算力的分配

### 3. 该开源代码允许他人使用，修改和发布

[TOC]



## 编译模块

#### **1. `nvidia.ko`（核心）**

- **核心驱动模块**，是所有其他模块的基础，管理GPU硬件资源
- 功能
  - 提供 GPU 与内核之间的基本交互接口，如内存管理等
  - 负责设备初始化、PCIe 通信、显存管理、命令队列等核心功能
  - 为其他模块（ `nvidia-drm.ko`、`nvidia-modeset.ko` 和 `nvidia-uvm.ko`）提供 API
- 首先加载，其他模块依赖于它

#### **2. `nvidia-drm.ko`**

- **Direct Rendering Manager (DRM) 接口模块**，用于支持 Linux 图形显示堆栈。
- 功能
  - 实现与 Linux DRM 子系统的集成
  - 负责 KMS（Kernel Mode Setting），用于配置显示输出设备（如分辨率和刷新率）
  - 为图形显示提供基础支持（如 Xorg、Wayland 和 Vulkan）
- 依赖 `nvidia.ko` 和 `nvidia-modeset.ko` 提供的底层功能

#### **3. `nvidia-modeset.ko`**

- **显示模式设置模块**，主要用于多显示器和显示设备的管理，与2. nvidia-drm.ko组成显示控制层
- 功能
  - 提供显示模式设置（Mode Setting）功能
  - 管理显示控制器的分辨率、刷新率、色深等配置
  - 支持动态调整屏幕输出的特性
- 依赖
  - 依赖 `nvidia.ko` 提供的设备访问能力。
  - 被 `nvidia-drm.ko` 使用来完成 DRM 层的显示管理。

#### **4. `nvidia-uvm.ko`**（重要）

- **Unified Virtual Memory (UVM)统一内存管理模块**，用于支持 GPU 和 CPU 的统一虚拟内存。
- 功能
  - 管理 GPU 与 CPU 的内存共享和一致性
  - 支持 CUDA 应用程序进行内存分页和传输操作
- 依赖依赖 `nvidia.ko` 进行设备访问和内存操作。



## 加载顺序

**`1. nvidia.ko`**：

- 必须最先加载，提供底层核心功能；初始化 GPU 设备和硬件接口。

**`2. nvidia-modeset.ko`**：

- 在 `nvidia.ko` 加载后加载；显示模式设置

**`3. nvidia-drm.ko`**：

- 依赖 `nvidia.ko` 和 `nvidia-modeset.ko`。

**`4. nvidia-uvm.ko`**：

- 最后加载，加载后 CUDA 等高性能计算框架才能使用 GPU 的统一内存。



## 项目结构

```bash
.
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── COPYING
├── Makefile
├── README.md
├── SECURITY.md
├── kernel-open      --内核态驱动模块的源代码和配置文件，用于构建各个ko文件 
│   ├── Kbuild       -- 构建规则
│   ├── Makefile
│   ├── common       --一些头文件
│   ├── conftest.sh  --检测环境
│   ├── count-lines.mk
│   ├── dkms.conf
│   ├── header-presence-tests.mk
│   ├── nvidia         --nvidia.ko的代码
│   ├── nvidia-drm     
│   ├── nvidia-modeset
│   ├── nvidia-peermem   --GPU间共享内存
│   └── nvidia-uvm
├── nouveau      --关联不大，支持开源Nouveau驱动的代码
│   ├── extract-firmware-nouveau.py
│   └── nouveau_firmware_layout.ods
├── nv-compiler.sh
├── src          --核心功能模块的代码， 更接近硬件层
│   ├── common   --通用代码
│   ├── nvidia
│   └── nvidia-modeset
├── utils.mk
└── version.mk

12 directories, 19 files

```

### kernel-open/nvidia

#### **硬件抽象和接口层**

主要负责与硬件交互的抽象化操作

| **文件/文件夹**  | **作用**                                                     |
| ---------------- | ------------------------------------------------------------ |
| `nv.c`           | 驱动的核心入口文件，包含主要的初始化、资源分配和清理逻辑，整个模块的核心 |
| `nv-acpi.c`      | 实现与 ACPI（高级配置与电源接口）的交互，用于管理电源状态和设备资源 |
| `nv-pci.c`       | 处理 PCI 设备的初始化、探测、资源分配和通信逻辑              |
| `nv-pci-table.c` | 管理支持的 GPU PCI 设备信息表，用于设备匹配和识别            |
| `nv-msi.c`       | 管理 MSI（消息信号中断）机制，优化 GPU 的中断处理性能        |
| `nvlink_linux.c` | NVIDIA NVLink 的实现，处理 GPU 之间的高速通信接口            |
| `os-pci.c`       | OS 层面与 PCI 的交互函数，例如读取 PCI 配置空间等            |

#### **内存管理相关**

| **文件/文件夹**      | **作用**                                                     |
| -------------------- | ------------------------------------------------------------ |
| `nv-dma.c`           | 实现 DMA（直接内存访问）机制，用于高效的数据传输             |
| `nv-mmap.c`          | 提供用户态程序访问 GPU 显存的内存映射功能                    |
| `nv-memdbg.c`        | 用于调试内存管理问题，例如检测内存泄漏和无效访问             |
| `nv-usermap.c`       | 实现用户空间与内核空间之间的内存映射支持                     |
| `nv-vm.c`            | GPU 显存管理核心代码，处理分配、释放和虚拟内存映射逻辑       |
| `nv_uvm_interface.c` | UVM（统一虚拟内存）相关的接口代码，用于与 `nvidia-uvm` 模块交互 |

#### **显示管理和错误处理**

| **文件**                 | **作用**                                                   |
| :----------------------- | ---------------------------------------------------------- |
| `nv-modeset-interface.c` | 提供与模式设置相关的接口，用于控制分辨率、刷新率等显示属性 |
| `nv-modeset-interface.h` | 与模式设置功能相关的头文件，定义了接口函数和结构体         |
| `nv-report-err.c`        | 错误报告和记录模块，用于诊断和定位 GPU 驱动问题            |
| `nv-report-err.h`        | 错误报告相关的头文件，定义了错误代码和报告机制             |

#### **操作系统相关适配**

这些文件处理驱动与操作系统的交互。

| **文件**         | **作用**                                                     |
| ---------------- | ------------------------------------------------------------ |
| `os-interface.c` | 提供跨平台操作系统接口的实现，例如锁机制、延迟等系统服务调用 |
| `os-mlock.c`     | 管理内存锁定操作，确保关键内存不会被交换出内存               |
| `os-registry.c`  | 提供操作系统注册表或配置文件的读取和写入功能                 |

#### **特殊硬件支持**

| **文件**                  | **作用**                                               |
| ------------------------- | ------------------------------------------------------ |
| `nv-ibmnpu.c`             | 支持 IBM NPU 的特殊功能，如内存操作和通信              |
| `nv-ibmnpu.h`             | IBM NPU 支持的头文件，定义了接口和相关数据结构         |
| `i2c_nvswitch.c`          | 管理与 NVIDIA NVSwitch 的 I2C 通信接口                 |
| `ioctl_common_nvswitch.h` | NVSwitch 相关的 IOCTL 定义文件，用于用户空间与内核交互 |
| `linux_nvswitch.c`        | 提供 NVSwitch 的具体实现，支持 GPU 间的高效通信        |

#### **安全和加密**

提供加密和签名支持。

| **文件**             | **作用**                                                     |
| -------------------- | ------------------------------------------------------------ |
| `libspdm_aead.c`     | 实现 AEAD（Authenticated Encryption with Associated Data）加密机制 |
| `libspdm_hmac_sha.c` | HMAC-SHA 的实现，用于数据完整性校验                          |
| `libspdm_x509.c`     | X.509 证书的解析和验证功能                                   |

#### **通用工具和支持**

这些文件提供通用的驱动支持功能。

| **文件**        | **作用**                                                     |
| --------------- | ------------------------------------------------------------ |
| `nv-dmabuf.c`   | 支持 DMA-BUF 共享缓冲区机制，用于与其他驱动共享数据          |
| `nv-cray.c`     | 针对 Cray 超算系统的优化和支持代码（可能涉及 GPU 在 HPC 环境中的使用） |
| `nvlink_caps.c` | 提供 NVLink 的功能检测和管理功能                             |

------

### **重点理解的文件(kernel-open/nvidia)**

根据你的目标（GPU 虚拟化研究），以下文件尤为重要：

1. **内存管理相关**：
   - `nv-vm.c`、`nv-dma.c`、`nv-mmap.c`、`nv-usermap.c`
   - 这些文件是 GPU 内存管理的核心，可以帮助你理解内存分配、共享和映射的机制。
2. **硬件抽象层**：
   - `nv.c`、`nv-pci.c`、`nvlink_linux.c`
   - 这些文件处理与硬件交互和设备初始化，了解它们有助于掌握驱动的底层硬件逻辑。
3. **操作系统适配**：
   - `os-interface.c`、`os-pci.c`
   - 理解这些文件可以帮助你扩展或调整驱动在不同环境下的适应性。
4. **虚拟化相关**：
   - `nv_uvm_interface.c`
   - 如果涉及 GPU 虚拟化，UVM 模块的接口实现尤为重要。
5. **调试与错误处理**：
   - `nv-report-err.c`、`nv-memdbg.c`
   - 用于调试和分析内存和硬件问题。

------

### **文件之间的关系**

- **`nv.c`** 是核心文件，其它模块通过调用 `nv.c` 提供的接口完成初始化和功能实现。
- **内存管理相关文件** 为 GPU 和 CPU 的资源分配提供支持，与 `nv_uvm_interface.c` 共同完成跨设备的统一内存管理。
- **错误处理和调试** 文件有助于在开发和调试阶段识别潜在问题。
- **操作系统适配层** 确保驱动代码能在不同 Linux 内核版本上运行。



### **kernel-open/nvidia-uvm**

#### **核心**

| **文件**    | **作用**                                      |
| ----------- | --------------------------------------------- |
| `uvm.c`     | uvm模块主入口，初始化(init)和清理(exit)的核心 |
| `uvm_gpu.c` |                                               |

#### **硬件架构支持**

| **目录/文件**   | **作用**                                                     |
| --------------- | ------------------------------------------------------------ |
| `hwref/ampere/` | 针对 Ampere 架构的硬件寄存器定义、特性支持等                 |
| `hwref/hopper/` | 针对 Hopper 架构的支持                                       |
| `hwref/turing/` | 针对 Turing 架构的支持，类似其他架构目录                     |
| 架构`.c` 文件   | `uvm_ampere.c`、`uvm_hopper.c` 等分别实现了不同架构的内存管理和故障处理逻辑 |

#### 内存管理

| **文件**         | **作用**                                                     |
| ---------------- | ------------------------------------------------------------ |
| `uvm_va_space.c` | 管理虚拟地址空间（Virtual Address Space），协调 CPU 和 GPU 的地址分配 |
| `uvm_va_block.c` | 管理地址块（VA Block）的分配与释放，支持地址映射的细粒度管理 |
| `uvm_va_range.c` | 管理地址范围（VA Range），实现虚拟内存的分段管理             |
| `uvm_mem.c`      | 实现内存分配和管理功能，包括显存、系统内存的分配             |
| `uvm_migrate.c`  | 数据迁移模块，处理 GPU 与 CPU 之间的内存数据迁移             |
| `uvm_hmm.c`      | HMM（Heterogeneous Memory Management）相关功能，支持 CPU 和 GPU 的异构内存管理 |

#### **故障处理**

| **文件**                          | **作用**                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| `uvm_fault_buffer*.c`             | 管理 GPU 的故障缓冲区（Fault Buffer），记录内存访问故障信息。 |
| `uvm_gpu_replayable_faults.c`     | 处理可重现的内存访问故障（Replayable Faults），支持错误恢复机制。 |
| `uvm_gpu_non_replayable_faults.c` | 处理不可重现的内存访问故障。                                 |
| `uvm_gpu_isr.c`                   | 中断处理逻辑，捕获和响应 GPU 故障中断。                      |

#### **性能优化**

通过事件、预测等机制优化内存访问性能

| **文件**                | **作用**                                                 |
| ----------------------- | -------------------------------------------------------- |
| `uvm_perf_events.c`     | 管理性能事件（如数据预取、访问热点），收集和记录性能指标 |
| `uvm_perf_heuristics.c` | 性能启发式算法模块，优化数据迁移和内存访问路径           |
| `uvm_perf_prefetch.c`   | 数据预取（Prefetching）模块，减少内存访问延迟            |

#### **设备间通信**

| **文件**           | **作用**                                                     |
| ------------------ | ------------------------------------------------------------ |
| `uvm_pushbuffer.c` | 实现 GPU 推送缓冲区（Push Buffer），用于协调多 GPU 通信      |
| `uvm_pmm_gpu.c`    | 管理 GPU 内存池（Physical Memory Manager），为设备间通信分配内存 |
| `uvm_pte_batch.c`  | 批量处理页表更新，优化设备间内存访问                         |

#### **与操作系统交互**

| **文件**       | **作用**                                        |
| -------------- | ----------------------------------------------- |
| `uvm_linux.c`  | 实现 UVM 模块在 Linux 系统上的适配代码          |
| `uvm_ioctl.h`  | 定义 IOCTL 接口，用于用户态和内核态的通信       |
| `uvm_procfs.c` | 提供 `/proc` 文件系统接口，暴露调试信息给用户态 |

#### **测试和调试**

| **文件**          | **作用**                                             |
| ----------------- | ---------------------------------------------------- |
| `uvm_test*.c`     | 各种功能测试代码，如内存分配、数据迁移、故障恢复等。 |
| `uvm_mem_test.c`  | 测试内存分配和管理功能。                             |
| `uvm_push_test.c` | 测试 GPU 推送缓冲区相关功能。                        |

------

### 







## 代码结构

- **kernel-open** : 与linux内核交互的接口层，包含了很多linux内核的函数与头文件，<linux/pci> <linux/module> <linux/firmware>等

  - /nvidia: nvidia.ko与linux内核的接口层

  - /nvidia-drm
  - nvidia-modeset
  - nvidia-uvm

- **src** :  内核态驱动底层代码

  - src/common：被多个ko文件调用的一些工具函数

- **nouveau** ：与Nouveau开源驱动的交互代码（与GPU虚拟化基本无关）



## 编译生成

1. **nvidia-drm.ko**: 

   **主要功能**：

   - 提供 Linux 的 Direct Rendering Manager (DRM) 接口，用于图形显示管理。
   - 支持内核模式设置（Kernel Mode Setting, KMS），管理图形相关资源，如帧缓冲区，分辨率等。
   - 支持 Vulkan 和 OpenGL 的显示扩展。
   - 实现与 `nvidia-modeset.ko` 的交互，管理显示硬件资源。

   **主要代码内容**：

   - ```
     nvidia-drm.c
     ```

     - 驱动的入口文件，注册 DRM 驱动。

   - ```
     nvidia-drm-helper.c
     ```

     - 实现一些 DRM 辅助功能，例如帧缓冲分配、提交任务等。

   - ```
     nvidia-drm-gem.c
     ```

     - 实现 GPU 显存管理（Graphics Execution Manager, GEM）。

   **生成模块：`nvidia-drm.ko`**

   - 提供 DRM/KMS 接口，与 Linux 内核的 DRM 子系统交互。
   - 必须加载此模块才能正常支持图形显示。

   

2. **nvidia-modeset.ko** ：显示硬件的模式设置

   **主要功能**：

   - 负责 GPU 显示模式设置和显示资源分配。
   - 直接与硬件通信，控制显示器分辨率、刷新率等参数。
   - 提供模式设置的硬件抽象层，与 `nvidia-drm.ko` 协作完成显示功能。

   **主要代码内容**：

   - ```
     nvidia-modeset-linux.c
     ```

     - 驱动入口文件，初始化显示硬件并注册显示资源。

   - ```
     nvidia-modeset-core.c
     ```

     - 包含显示模式设置的核心逻辑，例如显示资源初始化和上下文切换。

   **生成模块：`nvidia-modeset.ko`**

   - 独立负责模式设置，作为 `nvidia-drm.ko` 的支持模块。

   - 加载时初始化显示硬件资源，为 `nvidia-drm.ko` 提供底层支持。

     

3. **nvidia-uvm.ko** : 统一内存管理（Unified Virtual Memory），和虚拟化相关度较高

​	**主要功能**：

- 管理 GPU 显存和 CPU 内存之间的统一虚拟地址空间。
- 提供显存分配、映射和访问控制功能。
- 支持 CUDA 等计算框架中，GPU 和 CPU 之间的高效数据共享。
- 实现多个进程之间的显存隔离。

**主要代码内容**：

- ```
  uvm_gpu.c
  ```

  - 管理 GPU 的内存分配和映射。

- ```
  uvm_mem.c
  ```

  - 提供虚拟内存管理功能，例如跨进程共享显存。

- ```
  uvm_perf.c
  ```

  - 包含性能优化逻辑，例如 GPU 内存访问的预取。

**生成模块：`nvidia-uvm.ko`**

- 必须加载此模块才能启用 CUDA 或其他计算框架对 GPU 内存的支持。
- 处理多进程的显存共享和隔离，特别适用于深度学习和科学计算。





## 入口点 kernel-open/nvidia/nv.c 各种init和exit函数，以及aclloc等





## 各模块一般分为两个部分：

- **OS - agnostic：** 与操作系统无关的部分，一般以一个二进制文件打包，用户不必每次都编译；nvidia.ko的该部分为nv-kernel.o_binary， nvidia-modeset.ko的该部分为nv-modeset-kernel.o_binary. nvidia-drm.ko和nvidia-uvm.ko都没有OS - agnostic部分
- **kernel interface layer：** 与Linux内核版本以及配置有关，只能根据用户的平台编译构建

