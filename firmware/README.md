# firmware/ — 固件源码

> M2 阶段（开发板固件）正式建立工程。建立前此处只保留本说明。

## 计划结构（详见 docs/03_软件开发/README.md）

```text
firmware/
├── project/        工程文件（EIDE 工程 + CubeMX .ioc）
├── Core/           CubeMX 生成：启动、时钟(550MHz)、外设初始化
├── rtos/           RT-Thread 标准版 5.x 内核源码（FinSH/DFS 组件随选配）
├── drivers/        FDCAN、SDMMC、USB CDC、IMU、UART、LED/KEY（统一接口）
├── middleware/     CAN环形缓冲、SD写入管线、DFS/FatFs适配、时间戳服务、配置存储、主机协议
├── app/            记录控制状态机、主应用逻辑
└── build/          构建输出（已 git 忽略）
```

## 建立工程的前置条件（已满足 ✅）

1. M0 评审完成：主控 **STM32H723VGT6**（WeAct 核心板）、工具链 **EIDE + arm-none-eabi-gcc + CubeMX** 已定；
2. 硬件清单：WeAct H723VGT6 核心板 ×1、ST-Link/DAP-Link ×1、**CAN FD 收发器模块 ×2（TJA1044/MCP2562FD，勿买经典型）**、USB-TTL ×1、microSD 高耐久卡 ×1；
3. M1 系统设计完成并冻结后开工（时钟树、引脚、缓冲参数以 M1 文档为准）。
