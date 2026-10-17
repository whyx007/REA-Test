# ktrashscan.dll 文件级扫描分析

样本：`C:\Users\Aimer\AppData\Local\kdiskmgr_sogou\ktrashscan.dll`

## 证据

- `ev_e81a20d6d7a9d2d08e1955a262a1c7db1dc52bc9826f6a379af899fe1c24cce1`：文件回调函数，字符串标记 `ScanFile`。
- `ev_8e7dd488efca1e28ba9c05640668df21044cb81a9e28aaa340404994f2bdfc13`：`ScanFile` 字符串交叉引用来自 `0x100cfaa1`，函数入口为 `0x100cfa8f`。
- `ev_b5d6cb6a5b5e496b55abf05583362323c025072c7e52206cbbad5d0aacedd309`：`NeedDel` 时间过滤日志和阈值逻辑。

## 当前确认的流程

1. `FUN_100cf849`（字符串标记 `ScanDir`）使用 `FindFirstFileW`/`FindNextFileW`，跳过 `.`/`..`。
2. 非目录项计数器 `*param_2 += 1`，随后调用 `FUN_100cd4f5`（字符串标记 `AddFile`）。
3. 目录项计数器 `param_2[1] += 1`，拼接 `\\` 后递归调用 `FUN_100cf849`。
4. `FUN_100cfa8f`（`ScanFile`）再次检查 `dwFileAttributes & 0x10`；该位为 0 才继续，说明它只处理普通文件，不把目录作为清理项。
5. `ScanFile` 从文件路径建立内部字符串对象，调用 `FUN_100cd4f5` 添加文件结果；随后根据 `param_2[8]` 更新一个线程安全的累计量（`FUN_100f9d75` 是加法/锁封装）。
6. `FUN_100cd4f5`（`AddFile`）把路径、属性、规则字段和上下文传给 `FUN_100da6f1`；静态伪代码显示它先确保路径分隔符，再传入 `(*param_2 & 0x10 | 8) >> 3`、规则文本、`param_2 + 1`、上下文及计数值。这是结果对象登记的关键入口。
7. 在 `OnGotOneFile` 回调中，先调用 `FUN_100cecde`（`NeedDel`）；返回非零时才继续调用 `ScanFile`。因此时间过滤发生在文件结果登记之前。

## NeedDel

- `GetLastWriteTime` 失败：返回不可删除。
- 最后写入时间晚于当前时间：返回不可删除。
- `limit < 0`：使用 `0x93A80` 秒，即 7 天。
- `limit >= 0`：使用 `limit * 60` 秒。
- 文件年龄必须严格大于阈值才通过；等于阈值仍不可删除。

## 尚未确认

- `FUN_100da6f1` 内部结果结构的完整字段布局；
- 文件大小究竟来自 `WIN32_FIND_DATAW.nFileSizeHigh/Low`、规则字段，还是后续句柄查询；
- 扩展名/脚本规则过滤和误报过滤的精确顺序；
- 删除阶段是否再次调用 `NeedDel`，以及失败重试策略。

## AddResult / 结果对象（新增）

`FUN_100da6f1` 的字符串标记是 `AddResult`，由 `AddFile` 调用。

- 先复制规则/类别字符串 `param_3`，并把文件路径 `param_1` 与其拼接。
- `param_2` 是结果类型/层级；当它等于 1 时会额外把 `param_5`/`param_6` 作为 64 位数值设置到结果对象，说明文件大小或数值统计不是通过路径即时计算的，而是由调用方传入。
- 普通分支通过虚表创建结果项，并设置：数值字段、路径字段、规则/描述字段、初始状态 `0`、所属分类/上下文、显示层级以及 `param_7`。
- 另一分支使用 `FUN_100c5e5b` 创建特殊结果容器，设置同样的字段后直接挂到父结果。
- `this+0x28` 是结果过滤/去重管理器；若其虚表 `+0x50` 判断已存在，则调用 `this` 的 `+0x18` 回收/合并结果，否则把结果项状态设为 `1`。
- 当 `param_7 == 0` 且日志级别允许时记录 `AddResult, no trash, path:`，说明零值/无垃圾结果会被特别记录。

## 删除 API（新增）

`DeleteFileW` 在该 DLL 中有多个调用点（证据 `ev_69b3d23d6086c64097786271f4e227b51a4d559590a0f764504b368c915c92c7`），但当前已确认的 `0x100f07d7` 是规则缓存维护函数，不是用户扫描结果删除：它清理模块目录下 `data\\~strash.dat`，验证后用 `DeleteFileW` 删除临时缓存文件。

因此不能把所有 `DeleteFileW` 交叉引用都当作实际清理入口；需要继续筛选调用点上下文和结果项虚表删除方法。

## 实际文件/目录删除原语（新增）

函数 `FUN_100ebb6b`（入口 `0x100ebb6b`，证据 `ev_5057a7f378c6f23d06aeea765111fde14d67a1ff46d582b28c85fa7f68a38248`）是目前确认的通用删除包装器：

```text
status = ResolveItem(path, &kind, &aux)
if status == success:
    if mode == 2:
        resolved = NormalizePath(path)
        if aux == 0:
            return DeleteFileW(resolved)
        else:
            return RemoveDirectoryW(resolved)
return 0
```

静态证据表明：

- 删除接口依据解析出的 `aux` 区分文件和目录；
- 文件使用 `DeleteFileW`，目录使用 `RemoveDirectoryW`；
- 返回值直接保留 Win32 布尔结果，没有看到自动重试、改名后删除或回收站转移；
- 该包装器有三个内部调用者（`0x100e5efb`、`0x100e7160`、`0x100e8095`），它们属于清理/残留处理路径，比单独的 `DeleteFileW` 交叉引用更接近真实清理流程；
- `FUN_100f07d7`/`0x100c3f7b` 等函数仅维护 `data\\~strash.dat`、`data\\~sregtrash.dat` 缓存，不能视为用户文件清理。

## 对安全复刻的影响

真实清理实现至少要先解析目标类型，再分别调用文件删除和空目录删除；不能对目录调用 `DeleteFileW`。第一版仍建议只实现“预览 + 用户确认”，把删除动作封装为显式的二次确认步骤，并在删除前重新检查路径类型和最后写入时间。

## 复刻实现建议

第一版可安全实现为：递归枚举 -> 跳过目录/`.`/`..` -> 按最后写入时间过滤（默认 7 天） -> 记录路径、大小、属性、规则 ID -> 预览 -> 用户明确确认后删除。注册表、驱动仓库、脚本扫描和真实删除先保持关闭。

## 清理调用者分析（2026-10-17 新增）

`FUN_100e8095` 是目前最清晰的结果项清理路径。它从结果项读取路径、类型和状态，经过校验后计算删除模式：`(type == 2) + 1`，随后调用 `FUN_100ebb6b`。失败时写入错误状态；成功后继续更新父结果。对目标路径的父目录还会逐级截断并调用 `RemoveDirectoryW`，因此顺序是“先目标项，后尝试移除空父目录”。

`FUN_100e7160` 同样包含目录处理、`RemoveDirectoryW`、状态更新和 `Sleep`，更接近层级/目录清理分支。

`FUN_100e5efb` 中的删除调用位于 `KSoftDirMoveEngine::StartSlimMoveInstDirOneImpl rollback failed: RemoveSoftLink` 相关路径，主要用于软件目录瘦身或链接回滚，不应直接作为普通垃圾清理模型。

当前删除模型：

```text
结果项 -> 路径/类型/状态校验 -> 文件 DeleteFileW 或目录 RemoveDirectoryW
       -> 失败记录状态 -> 成功更新结果 -> 尝试删除空父目录
```

尚未从静态证据确认：回收站转移、自动重试，以及删除前是否再次调用 `NeedDel`。

## 删除边界复核（2026-10-17）

- 字符串和导入中存在 `SHEmptyRecycleBinW`、`SHQueryRecycleBinW`，但它们归属于 `KRecycleBinWorkItem` 的回收站扫描/清空功能；在普通结果项删除调用者中没有发现 `SHFileOperationW` 的可解析交叉引用。因此不能把普通文件清理推断为“移入回收站”。
- 搜索 `retry`/`Retry` 未发现业务重试字符串。已见删除调用者在 `DeleteFileW`/`RemoveDirectoryW` 失败后主要写入错误状态或回滚状态，未看到同一路径立即重复删除的明确循环。
- `Sleep(1000)` 出现在清理调用者中，但结合其位置更像结果刷新、线程同步或批处理节流；静态证据不足以称为删除重试。
- `NeedDel` 字符串仅位于扫描阶段的 `KCommonXmlScanWorkItem`，当前删除调用者未解析出对 `NeedDel` 的直接调用。因此“删除前二次时间检查”仍未证实。

安全实现建议：普通清理默认直接删除前重新做一次本地安全检查（路径存在、类型未变化、年龄仍超过阈值），但这属于复刻时的防护设计，不应冒充为已确认的搜狗运行时行为；回收站清空应作为单独功能开关。

## 误报过滤与失败回调（2026-10-17 新增）

`FUN_100a87fc` 是 `KTrashScanFalseAlarmKiller` 相关函数。静态伪代码显示：先读取/解密过滤数据并调用 `FUN_100a858d` 判断过滤状态；判断为未命中时才调用 `DeleteFileW`，否则记录 `_DeCryptFile fail`。这证明误报过滤位于某些清理动作之前，但过滤文件的完整格式和字段尚未恢复。

`FUN_100b5ec8` 是另一条删除包装器：

```text
if path == null: return E_INVALIDARG
ok = DeleteFileW(path)
if !ok and GetLastError() != ERROR_FILE_NOT_FOUND:
    return error_callback(path)
return 0
```

它没有重试删除；失败时转交错误回调，且“文件已不存在”被视为非错误。该行为可作为复刻程序的失败处理参考。

## 配置/缓存模块

已确认字符串入口：`data\\trashscan.ini`、`data\\trashtime.dat`、`kduba_trashscan_use_file_cache`，以及 `regfilter`、`softfilter`、`pluginfilter`。这些表明扫描引擎有独立的时间配置、文件缓存和误报过滤配置。当前尚未确认其加密/序列化格式，因此第一版不应直接依赖这些文件作为可信规则源。
