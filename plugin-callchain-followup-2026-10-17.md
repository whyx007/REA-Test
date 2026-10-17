# 插件调用链补充：`reportScanData`

## 静态结果

在 `kqqcleanup.dll` 中，Ghidra 字符串记录到：

```text
.?AVTask@?1??reportScanData@KFunctionSet@@YAXH_N0@Z@
```

这属于 MSVC RTTI/类型字符串，不是已解析的函数入口。按修饰名可观察到候选签名片段：返回 `void`，参数形状包含一个 `int` 和两个 `bool`。当前 Ghidra 名称索引中没有名为 `KFunctionSet` 或 `KFunctionSet::reportScanData` 的已定义过程；按该字符串地址直接反编译也不成立，因为它位于数据区而非加载映射的代码入口。

## 结论调整

- 可以把 `reportScanData` 作为插件内部任务/回调的存在证据。
- 不能把它当作已确认的导出函数、可直接调用的 ABI，或已确认的宿主回调入口。
- 参数的业务含义、调用方、调用顺序和线程归属仍未知。
- 安全复刻继续使用抽象事件接口，不复刻未验证的 C++ ABI。

## 已确认的相邻边界

同一 DLL 还存在 `scanFinishSignal`、`updateScanProgressSlot`、`updateSlimProgressSlot`、`restartScanSlot` 等 Qt 元对象字符串，以及 `QFutureInterfaceBase::reportStarted/reportFinished/isCanceled` 外部符号。这支持“异步任务 + 进度/完成事件”的架构判断，但不能单凭字符串确定具体状态机。

## 证据

- `ev_14f904b58efe2af42a10908db2272d7c050abc265a9c8edbd13a71e9d8dab971`：`reportScanData` RTTI 字符串。
- `ev_237921009a4902a8b81192e862bd868b195103576918abef1254fb9f136a3c6c`：进度与完成槽字符串。
- `ev_55f32a492131066ff8681f768b5a61c8ca9309884ff6b018e4c6e8564f84be01`：外部 Qt 异步任务符号。

所有结论均为静态证据；未运行 DLL，也未观察运行时调用。
