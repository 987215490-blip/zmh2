# 基于FPGA的LeNet-5 CNN手写数字识别系统

## 项目简介

本项目在Altera/Intel FPGA平台上实现了一个纯硬件的LeNet-5卷积神经网络，用于实时手写数字识别。系统通过OV7670摄像头采集图像，经灰度转换与下采样后送入CNN推理引擎，最终在ILI9341 TFT屏幕上显示识别结果。全部推理过程由FPGA硬件电路完成，无需软核处理器参与。

## 系统架构

```
OV7670摄像头 ──I2C配置──┐
                        │ RGB数据流
                        ▼
           ┌─────────────────────┐
           │   图像预处理模块     │
           │  RGB→灰度→28×28     │
           └─────────┬───────────┘
                     │ 灰度矩阵
                     ▼
           ┌─────────────────────┐
           │  LeNet-5推理引擎     │
           │  卷积→池化→全连接    │
           │  11位定点数运算      │
           └─────────┬───────────┘
                     │ 识别结果(0-9)
                     ▼
           ┌─────────────────────┐
           │   TFT显示模块       │
           │  SPI协议驱动屏幕    │
           └─────────────────────┘
```

## 模块说明

### 顶层模块
| 模块名 | 文件 | 功能 |
|--------|------|------|
| `cam_proj_top` | `top/cam_proj_top.v` | 系统顶层，连接摄像头、SDRAM、TFT、预处理和CNN模块 |
| `TOP` | `code/neuroset/TOP.v` | CNN推理引擎顶层，状态机调度卷积/池化/全连接 |

### 神经网络模块
| 模块名 | 文件 | 功能 |
|--------|------|------|
| `conv_TOP` | `code/neuroset/conv_TOP.v` | 卷积层控制，管理滑动窗口、权重加载、累加计算 |
| `conv` | `code/neuroset/conv.v` | 3×3卷积核乘累加运算单元 |
| `maxp` | `code/neuroset/maxpooling.v` | 2×2最大池化模块 |
| `dense` | `code/neuroset/dense.v` | 全连接层，16→11→10输出 |
| `result` | `code/neuroset/result.v` | 10路输出比较，判决识别数字 |
| `border` | `code/neuroset/border.v` | 卷积边界检测，判断像素是否在图像边缘 |

### 存储模块
| 模块名 | 文件 | 功能 |
|--------|------|------|
| `RAM` | `code/neuroset/RAM.v` | 双端口RAM，存储像素、中间结果和权重 |
| `database` | `code/neuroset/database.v` | 权重ROM，固化5460个11位定点数参数 |
| `memorywork` | `code/neuroset/RAMtoMEM.v` | 存储管理，负责从外部加载数据到内部RAM |
| `addressRAM` | `code/neuroset/addressRAM.v` | 地址管理，按step分配不同层的存储地址 |

### 外设驱动模块
| 模块名 | 文件 | 功能 |
|--------|------|------|
| `pre_v2` | `code/gray_28x28/grayscale.v` | 图像预处理：RGB转灰度 + 8×8块均值下采样至28×28 |
| `tft_ili9341` | `code/lcd/tft_ili9341.sv` | ILI9341 TFT显示屏驱动，SPI协议 |
| `tft_ili9341_spi` | `code/lcd/tft_ili9341_spi.sv` | SPI时序发生器 |
| `hellosoc_top` | `code/lcd/hellosoc_top.sv` | TFT顶层控制，帧缓冲管理 |
| `cam_wrp` | `code/synt/cam_wrp.v` | OV7670摄像头数据采集与FIFO缓冲 |
| `sdram_controller` | `code/synt/sdram_controller.v` | SDRAM控制器，帧数据存储 |

## CNN推理流程

| 阶段 | 操作 | 输入尺寸 | 输出尺寸 |
|------|------|---------|---------|
| TOPlvl 1-2 | 第1组卷积×2 | 28×28×1 | 28×28×4 |
| TOPlvl 3 | 最大池化 | 28×28×4 | 14×14×4 |
| TOPlvl 4-5 | 第2组卷积×2 | 14×14×4 | 14×14×8 |
| TOPlvl 6-7 | 最大池化×2 | 14×14×8 | 7×7×8 |
| TOPlvl 8-9 | 第3组卷积×2 | 7×7×8 | 7×7×16 |
| TOPlvl 10 | 全局池化+全连接 | 7×7×16 | 10类 |

## 定点数格式

采用11位定点数表示，格式为：
```
[符号位(1bit)][整数部分(2bit)][小数部分(8bit)]
```

- 像素值范围：0~2.0（归一化后）
- 权重范围：约-2.0~2.0
- 乘累加结果：21位，取高11位作为输出（右移10位）
- 溢出处理：饱和截断，防止数据溢出

## 硬件平台

- FPGA芯片：Altera/Intel Cyclone系列
- 摄像头：OV7670（320×240 RGB）
- 显示屏：ILI9341 TFT（SPI接口）
- 存储：SDRAM（帧缓冲）

## 仿真验证

运行testbench进行功能仿真：

```bash
# 使用ModelSim/QuestaSim
vsim -do sim.do

# 或使用VCS
vcs -f filelist.f
./simv
```

testbench中包含一张手写数字"8"的28×28灰度图像数据，仿真完成后应输出`RESULT: 8`。

## 文件结构

```
verilog/
├── code/
│   ├── neuroset/          # 神经网络核心模块
│   │   ├── TOP.v          # CNN推理顶层
│   │   ├── conv_TOP.v     # 卷积层控制
│   │   ├── conv.v         # 卷积运算单元
│   │   ├── maxpooling.v   # 最大池化
│   │   ├── dense.v        # 全连接层
│   │   ├── result.v       # 结果判决
│   │   ├── border.v       # 边界检测
│   │   ├── RAM.v          # 双端口RAM
│   │   ├── database.v     # 权重ROM
│   │   ├── RAMtoMEM.v     # 存储管理
│   │   └── addressRAM.v   # 地址管理
│   ├── gray_28x28/        # 图像预处理
│   │   └── grayscale.v    # 灰度转换与下采样
│   ├── lcd/               # TFT显示驱动
│   │   ├── hellosoc_top.sv
│   │   ├── tft_ili9341.sv
│   │   └── tft_ili9341_spi.sv
│   ├── synt/              # 摄像头与SDRAM
│   │   ├── cam_wrp.v
│   │   ├── sdram_controller.v
│   │   └── ...
│   └── testbench.v        # 仿真测试文件
├── top/
│   └── cam_proj_top.v     # 系统顶层
└── imp/                   # Quartus工程文件
    ├── cam_proj.qpf
    ├── cam_proj.qsf
    └── output_files/
```

## 参与贡献

1. Fork 本仓库
2. 新建 Feat_xxx 分支
3. 提交代码
4. 新建 Pull Request
