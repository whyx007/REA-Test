# 路径发现与 IPC 补充分析（2026-10-17）

本轮继续使用 Ghidra 12.1.4 对功能包内二进制做静态分析；未运行搜狗 EXE/DLL，以下结论不是运行时观测。

## 1. `kscanengine.dll` 路径与文件处理

样本 SHA-256：`93612c224d0c4de2749364849d400c81763a06d6cf389d64422b27fbc1d42b09`。

### 1.1 系统目录/配置路径

`FUN_1003b597` 调用 `SHGetSpecialFolderPathA(NULL, ..., 0x1a, TRUE)`，随后构造并比较 `config\\systemprofile`。这表明引擎存在一次性系统目录检测/配置分支；静态代码没有给出微信或 QQ 固定目录表。

证据：`ev_a9f5cdabd5fff46e3ea8acd89a5d35734cbf911ad8326293f051c3919818ebbf`。

### 1.2 规则/日志文件路径

`FUN_10029b8b` 使用模块文件名，`PathRemoveFileSpecW` 后追加 `\\log\\`，再追加模块/资源派生名称和 `.log`。因此日志目录是“模块目录\\log\\<派生名>.log”，不是固定用户目录。该函数还通过 `CreateDirectoryW` 创建目录。

证据：`ev_fa5d233a2b6ce71ca6f54874992ea3d667ee4ab11835e244f68785a8e6f521f7`。

### 1.3 文件扫描与元数据

`FUN_100259e3` 在可选存在性检查通过后读取文件，支持 ANSI、UTF-8 与宽字符路径/内容转换，并通过回调把转换后的内容送入对象；当路径不存在时调用 `FUN_1002507d`。

`FUN_10025d15` 使用 `CreateFileW` 只读共享、`GetFileSize`，限制为小于 `0x200001` 字节（约 2 MiB），完整读取后返回缓冲区。

`FUN_1004261c` 从 Qt 规则对象提取路径、大小、时间等字段；在规则标志要求时使用 `CreateFileW`/`GetFileSize`/`GetFileTime` 补充元数据。该路径是规则驱动的，不是硬编码微信/QQ路径表。

证据：`ev_44012df5f21a53240629d0adcb849d68035a883ded72fa1d5db35323f7c41819`、`ev_057fb86131f601ac7b6939f9a781fe885d67d5dd9396d4e20c1339f5b6b4cdf4`、`ev_504bdfa4af1baa19f9ca3404d7ab003d9e612d81e6419820f5708c0c9cfe07dd`。

## 2. `ksearchservice.exe` IPC/服务结构

样本 SHA-256：`d29ee24ebd398ea9382322b3f117e99151453def39fb51f41e5ccfa802484440`。

核心初始化函数 `FUN_4da112c8` 的静态伪代码显示：

- 创建命名互斥体 `mutex{430724BB-B7C6-46B6-8CF0-01F0A60D0A57}`；已存在时记录重复启动并退出路径。
- 创建 Qt `QLocalServer`，监听名 `server{77D40466-DBC1-4FD5-9104-6133882729E0}`。
- 调用 `GetLogicalDriveStringsW(0x100, ...)` 枚举逻辑盘符。
- 为每个盘符创建工作对象/线程，并连接 `QLocalServer::newConnection`、QLocalSocket 读写/断开等信号槽。

这证明搜索服务使用 Qt 本地 IPC（QLocalServer/QLocalSocket），不是 Win32 命名管道；服务启动时按逻辑盘符创建搜索工作线程。具体消息帧字段和命令仍需继续从各连接槽函数恢复，当前不能声称已完全还原协议。

证据：`ev_4d9848565be75827842bf67a0f7b64f99bdcc032178de4301a6cc9e9be738be0`。

## 3. `kdiskcleancore.exe` 初步边界

样本 SHA-256：`181737585c4c29695024e22e3264e97a506a6e02ecdf0c8024b6c2918c966549`。

静态导入显示 Qt、Win32 文件删除和线程/配置相关 API，但本轮尚未恢复稳定的请求/结果协议入口；下一轮应针对其 Qt 元对象槽、`QDataStream`/`QSharedMemory` 和 `QFile::remove` 调用者做地址级追踪。

## 4. 当前边界

- 已确认动态路径构造、模块相对日志、逻辑盘枚举和本地 IPC。
- 未发现可直接作为明文的 `Tencent`、`WeChat`、`MicroMsg`、`AppData` 字符串；这只是限定搜索范围内的静态否定，不排除规则值、注册表或运行时输入生成路径。
- 复刻时应实现“规则提供候选根路径 + 逻辑盘枚举 + 本地 IPC 服务”的抽象，不应把固定目录表写死为已证实事实。

