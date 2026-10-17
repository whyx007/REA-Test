# 过滤数据与系统清理插件补充（2026-10-17）

本轮只做静态文件检查和 Ghidra 分析，未加载或运行搜狗 DLL。

## 1. 白名单文件

已从本机数据目录复制到 `supporting-data`：

- `whitelist.dat`：311 字节，明文 XML。
- `winsoftscanwhitelist.dat`：395 字节，二进制，前缀 `KAEF`。
- `strash.dat`：172 字节，前缀 `HLKBP`，符合已知规则容器头。
- `kclearak.dat`：1096 字节，前缀 `KAEF`。

`whitelist.dat` 的内容是 10 个软件 ID：

```xml
<object>
  <item val="60000234" />
  <item val="60001448" />
  <item val="60039381" />
  <item val="70000981" />
  <item val="60025942" />
  <item val="60037746" />
  <item val="60040548" />
  <item val="61000010" />
  <item val="60048173" />
  <item val="70002847" />
</object>
```

静态意义：这是软件 ID 级白名单，不是路径白名单；它应在系统/软件清理插件选择候选项时参与过滤。

## 2. `ksysslim.dll` 文件与设备操作

样本 SHA-256：`953d217d39b2557cff8bed317e222a8597320c12ad4c09c2240ea960fe0eb99c`。

### 2.1 文件读取/写入包装

`FUN_10004c80` 以只读方式打开文件，记录句柄、当前位置和失败码；`FUN_10006124` 将内部数据流写入目标路径，支持相对路径与绝对路径，使用临时/目标路径和 `CreateFileW`、`WriteFile`、`SetFileTime`。

这说明系统清理引擎除删除外还包含文件生成/替换能力，复刻时必须限制在明确的缓存/临时目录，不能照搬为任意路径写入。

### 2.2 大文件读取限制

`FUN_1000a9d6` 读取文件时将单次缓冲限制在 `0x200000`（2 MiB）以内，读取后交给内部解析器。这与 `kscanengine.dll` 的约 2 MiB 规则读取上限一致，但属于不同模块的独立证据。

### 2.3 物理卷/设备控制

`FUN_1000af6b` 使用 `CreateFileW("\\\\.\\%c:")`、`DeviceIoControl(0x560000)`，随后可能打开 `\\\\.\\PhysicalDrive%d` 并调用 `DeviceIoControl(0x2d1400)`。

静态上可确认它具备卷/物理磁盘控制路径；尚未确认这些控制码在当前清理流程中的具体业务用途。安全复刻绝不能直接实现这类物理设备控制，只能把它标记为不支持或改为受控的只读查询。

## 3. 数据格式边界

- `whitelist.dat`：明文 XML，已可直接解析。
- `strash.dat`：`HLKBP` 容器，可能沿用规则 XOR/数据段格式，需要单独解码。
- `winsoftscanwhitelist.dat`、`kclearak.dat`：`KAEF` 二进制，当前未恢复字段和加密/校验算法。
- `slimdata.dat`、`config3a.dat`、`kaccclear.dat`、`diskaccclear2.dat`：系统规则/配置数据，体积较大，尚未从 DLL 中建立完整解析入口。

## 4. 安全复刻要求

系统插件的物理卷控制、任意文件写入和潜在低层清理能力不能直接复制。复刻版本应：

1. 仅实现规则解析和候选扫描；
2. 删除前做白名单、路径保护和用户确认；
3. 对卷设备 API 默认禁用；
4. 将 `KAEF` 与 `HLKBP` 数据文件作为未完全解码的输入，不把未知内容当作可执行指令。

