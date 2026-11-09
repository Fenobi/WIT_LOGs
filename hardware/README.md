# hardware/ — 硬件设计文件

> M3 阶段（硬件设计）开始填充。设计文档见 docs/02_硬件开发/。

## 目录约定

```text
hardware/
├── schematic/      原理图工程 + 导出 PDF
├── pcb/            PCB 工程 + Gerber + 贴片坐标文件
├── bom/            BOM 表（含采购链接、单价、数量）
└── datasheets/     关键芯片数据手册
```

## 约定

- EDA 工具（KiCad / 立创EDA）在 M3 启动时确定，工程文件随仓库提交；
- Gerber 每次打样单独存 `pcb/gerber_vN_N/`，与打样记录（docs/02_硬件开发/03）对应；
- 数据手册文件较大，视情况用 git LMS 或只在 BOM 中放下载链接。
