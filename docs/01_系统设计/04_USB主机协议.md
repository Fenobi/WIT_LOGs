# USB 主机协议定义

| 文档状态 | 🔄 待评审 → 通过即冻结 |
|----------|----------|
| 版本 | v0.1 |
| 日期 | 2026-11-09 |
| 关闭开放问题 | **Q4**：USB 采用 **FS CDC**（不预留 HS/ULPI PHY）；满载场景靠**流模式降级**保证观察可用，SD 记录始终完整 |

---

## 1. 传输层

- 物理通道：USB 2.0 FS，CDC-ACM 虚拟串口（主机免驱）；
- 载荷：任意 8-bit 二进制，无转义；
- 设备端收发各走 32KB/内部 CDC 缓冲 + 应用层 TX 环 32KB。

## 2. 包格式

```text
┌─────┬─────┬───────┬────────┬───────┬──────────┬─────────┐
│0xAA │0x55 │ type  │ flags  │ len   │ seq      │ payload │ crc16 │
│ 1B  │ 1B  │  1B   │  1B    │ u16LE │ u16LE    │ len 字节 │ u16LE │
└─────┴─────┴───────┴────────┴───────┴──────────┴─────────┴───────┘
```

| 字段 | 说明 |
|------|------|
| type | 包类型（§3/§4） |
| flags | bit0=需要 ACK；其余保留 0 |
| len | payload 长度，≤ 504 |
| seq | 主机命令序号（命令包）；设备回包/流包 seq=0 或回显命令 seq |
| crc16 | CRC16-CCITT(0xFFFF 初值)，覆盖 type..payload |

接收规则：同步扫描 `AA 55` → len 合法(≤504) → CRC 校验，失败整包丢弃 + 计数（不做重传——命令类由主机超时重发，流类丢了无所谓）。

---

## 3. 命令集（主机 → 设备）

| type | 命令 | payload | 说明 |
|------|------|---------|------|
| 0x10 | GET_INFO | — | → fw_ver, proto_ver, serial, hw_id |
| 0x11 | GET_STATUS | — | → 状态结构体（§5） |
| 0x12 | SET_TIME | u64 epoch_us | 主机时间同步（写 RTC + 校准 TIM2 锚点） |
| 0x13 | START_REC | u8 mode (0=新会话, 1=续记) | → ACK |
| 0x14 | STOP_REC | u8 flush (1=等待 fsync 完成) | → ACK |
| 0x15 | SET_CH_CFG | {u8 ch, u8 mode, u32 arb_bps, u32 data_bps} | 仅 IDLE 态可改 → ACK |
| 0x16 | SET_FILTER | {u8 ch, u8 mode, u8 n, (u32 id, u32 mask)×n} | mode: 0=ALL 1=LIST 2=RANGE；热生效 → ACK |
| 0x17 | ADD_TX | {u8 ch, u32 id_flags, u16 len, u8 data[], u32 period_ms} | period=0 单发；→ handle u16 |
| 0x18 | DEL_TX | u16 handle | → ACK |
| 0x19 | SET_STREAM | {u8 mode, u16 param} | mode 见 §6 |
| 0x1A | GET_CONFIG | — | → witlog.cfg 内容 |
| 0x1B | SET_CONFIG | cfg 文本 | 可选 persist 标志 → ACK |
| 0x1C | REBOOT | — | → ACK 后复位 |
| 0x1D | GET_STATS | — | → STATS 结构（同数据格式 §3.3 + ring 水位） |

## 4. 响应与异步包（设备 → 主机）

| type | 名称 | payload |
|------|------|---------|
| 0x40 | ACK | {u16 seq_cmd, u8 status, 附加数据} |
| 0x41 | NAK | 同上；status 错误码表见 §7 |
| 0xA0 | STREAM_DATA | **n × [12B记录头 + 数据]**——与 SD 卡记录格式完全相同（数据格式文档 §3） |
| 0xA1 | EVENT_PUSH | {u32 code, u32 arg}（与 WLG EVENT 记录同码表） |

包大小策略：STREAM_DATA 尽量凑满 504B（多条记录打包），降低 USB 事务开销。

---

## 5. 状态结构体（GET_STATUS 应答 payload，56B 定长）

```text
u8  state           状态机状态码（IDLE/PREP/RUNNING/FLUSH/ERROR）
u8  ch_state[2]     每通道: bit0=init bit1=bus-off bit2=err-passive
u8  stream_mode     当前流模式
u8  reserved
u32 rec_state       记录状态: session_uid hash + 段号
u32 rec_count       本段记录数
u32 drop_old / hw_overrun / busoff_cnt 合并数组 ×2
u32 card_free_mb
u32 ts_now_us       当前 TIM2 值（主机测通道延迟用）
u32 uptime_s
```

---

## 6. 流模式与降级（Q4 落地）

| mode | 名称 | 行为 | 适用 |
|------|------|------|------|
| 0 | OFF | 不推送 | 纯记录 |
| 1 | ALL | 全帧推送（复用记录格式） | 设计目标工况（≤0.43MB/s < FS 实测 0.6MB/s） |
| 2 | FILTERED | 仅通过主机过滤器(0x16)的帧 | 大流量下聚焦目标报文 |
| 3 | SAMPLED | 每 N 帧取 1（param=N） | 满载观察总线统计特征 |

**自适应降级**：USB TX 环水位 ≥75% → 推 EVENT_PUSH 警告；≥95% → 自动切 `SAMPLED(N=4)` 并推 EVENT；水位回落 <50% 恢复原模式。全程 `usb_drop` 计数。

> 明确契约：**USB 流不承载完整性**——主机要完整数据读 SD 卡（wlconv.py）。文档在此固化，避免后续误用。

---

## 7. 错误码表

| code | 含义 |
|------|------|
| 0x00 | OK |
| 0x01 | CRC 错（一般收不到——设备直接丢包） |
| 0x02 | 未知 type |
| 0x03 | 参数非法（如波特率不支持、ch 越界） |
| 0x04 | 状态不允许（如 RUNNING 中改波特率） |
| 0x05 | SD 卡错误 |
| 0x06 | 缓冲资源不足（TX 句柄用尽等） |

---

## 8. 版本与兼容

- `proto_ver=1`；新增命令只增不改；未知 type 设备静默丢包+计数，主机未知应答同样处理；
- 破坏性变更升 proto_ver，GET_INFO 可查询；
- `tools/witmon.py` 按本协议实现 CLI（--help 列出全部命令）。
