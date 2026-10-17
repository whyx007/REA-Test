# 系统规则与数据格式补充（2026-10-17）

本轮对本机已存在数据文件做只读解码和格式比对，没有运行清理程序。

## `HLKBP` 规则容器

`strash.dat`、`slimdata.dat` 均以 `HLKBP` 开头，并遵循此前恢复的格式：固定头约 `0x50` 字节，正文从 `0x50` 开始逐字节 XOR `0xAB`。

`strash.dat` 解码后得到：

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes" ?>
<items />
<time>2026.10.15</time>
```

这说明 `strash.dat` 是带更新时间的垃圾项/黑名单容器，目前为空。

`slimdata.dat` 解码后是完整 XML `<slims>` 规则集，包含系统输入法、示例媒体、Office 缓存、旧系统缓存、帮助文件、转储文件、休眠文件、更新缓存、Defender、诊断数据、缩略图、DirectX 缓存、驱动包、升级日志和系统错误报告等条目。

每个 `<slim>` 至少包含：

```text
id / gid / autocheck / name / description
dir os="..."
path os="..."
```

路径支持 `%Systemroot%`、`%SystemDrive%`、`%Public%`、`%localappdata%` 等环境宏，并按 Windows 版本区分 `win7`、`win8`、`vista`、`win10`、`all`。

因此系统清理规则主体已经可以直接解析，不必依赖二进制反编译推测所有路径。

## `KAEF` 数据容器

`winsoftscanwhitelist.dat` 与 `kclearak.dat` 均以：

```text
KAEF 00 01 12 AE ...
```

开头，正文未呈现明文 XML；本轮尚未恢复其压缩/加密/校验算法，继续标记为二进制未知格式。

## `config3a.dat`

`config3a.dat` 不以 `HLKBP/KAEF` 开头，前部包含 `ksystemr`、日期样式和多个偏移/长度字段，属于独立配置/索引格式，不能按规则容器直接解码。

## 复刻结论

- `slimdata.dat` 可直接作为系统规则源，优先实现 XML 解析和环境宏展开。
- `strash.dat` 可按相同 XOR 容器读取，并用 `<time>` 做更新时间判断。
- `KAEF` 文件暂不应被伪解密；安全复刻版应把它们作为可选、未实现的过滤数据库。
- 系统规则中的高风险项目（休眠文件、旧系统、驱动包、物理卷相关项目）必须默认关闭或要求显式确认。

