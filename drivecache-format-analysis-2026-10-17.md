# DriveCache 缓存格式补充（2026-10-17）

本轮静态分析确认了 `DriveCache%1.dat` 的写入格式。

## 缓存保存流程

`FUN_4da0a7f8`：

1. 解析默认/配置缓存根路径。
2. 创建缓存目录。
3. 拼接 `DriveCache%1.dat`，其中 `%1` 来自盘符/卷标字段。
4. 以写入模式打开文件。
5. 通过 `QDataStream` 写入：

```text
int       format_or_version = 0x16   // 十进制 22
qint64     cache_timestamp_or_marker
QDateTime last_scan_time
int       item_count_or_limit
```

6. 写入盘扫描对象及其结果分片。
7. 关闭文件，删除旧目标，再通过临时路径重命名替换，属于“写临时文件后替换”的更新策略。

证据：`ev_990e4654cee428b1c82b77dddb183e26363609f537d0d93188932eedd23edac7`。

## 扫描对象序列化

`FUN_4da15884` 写入对象数量，然后对每个盘对象写入：

```text
int object_count
for each object:
    raw 0x20-byte shared-memory header
    repeat each segment:
        int segment_size
        raw segment bytes
```

`FUN_4da152a5` 是单个共享内存对象的序列化版本：

```text
raw 0x20-byte header
for each segment:
    segment_size = 10                    if flag bit 0 == 0
                 = *(uint16)(segment+0x2e) + 0x39  otherwise
    int segment_size
    raw segment bytes
```

分片遍历步进使用 segment 的第二字节；这与此前恢复的共享内存头/分片迭代逻辑一致。

证据：`ev_1728835314a60bbf907d24b09e4ee96220560ee2f7cc7099967a0d59fa86d038`、`ev_1e8f1c757ff57461d93ac313c6508b999de046d58fe573ac9d45d3ef3793a3f0`。

## 缓存读取校验

对应读取函数会先读取 `format_or_version`，当前明确比较值为 `0x16`；随后读取 64 位字段、`QDateTime` 和整数，并依据缓存时间及数量阈值判断是否可复用。静态代码显示：

- 数量小于 `20000` 时，缓存时间距当前不足约 30 天才可能继续复用；
- 数量达到/超过 `20000` 时，时间窗口收紧到约 3 天；
- 超过数量/时间条件则放弃该缓存并重新扫描。

证据：`ev_4b5673a15bea1e92207d5a9b15e1551e1688cb0e837b256224cba467f3a05587`。

## 复刻意义

现在可以实现一个兼容方向的缓存层：固定版本 22、64 位标记、Qt 日期时间、数量字段、0x20 字节共享内存头和变长分片序列。尚未确定的是 64 位字段与数量字段的最终业务命名，以及 Qt `QDateTime/QDataStream` 的精确字节端序细节；若追求协议兼容，应使用 Qt 同版本或按 Qt 序列化规范实现。

