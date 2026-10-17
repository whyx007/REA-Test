# 搜索服务 IPC 协议补充（2026-10-17）

本轮对 `ksearchservice.exe` 做地址级静态追踪，未运行程序。

样本 SHA-256：`d29ee24ebd398ea9382322b3f117e99151453def39fb51f41e5ccfa802484440`。

## 1. 连接建立与初始化消息

`FUN_4da12651` 是 `QLocalServer` 新连接处理路径：

1. 从连接句柄创建 `QLocalSocket`，调用 `setSocketDescriptor`。
2. 把连接加入服务连接集合，并挂接 `disconnected`、`errorOccurred` 等信号。
3. 遍历已经初始化的盘符工作对象，将每个对象的初始化状态序列化后写入该 socket。
4. 如果服务初始化完成，额外发送一个 `MSG_InitServiceDone` 消息。

该函数直接调用 `QIODevice::write`，每次写完都调用 `QLocalSocket::flush`。

证据：`ev_68a6e9bcd6111bd5c558eaef1992d3f149719c89fcbe631c3549c460632c8a67`。

## 2. 已恢复的消息编码形态

### 2.1 初始化/盘符状态消息

`FUN_4da135fe` 使用 `QDataStream` 写入以下字段：

```text
int      message_type = 1
QString  drive_or_name
ushort   field_2
int      field_3
int      field_4
```

随后把整个 `QByteArray` 转为 Base64，并在末尾追加 ASCII `<<`。因此线上的写入形态不是裸 `QDataStream`，而是：

```text
Base64(QDataStream(payload)) + "<<"
```

证据：`ev_551a4b1dfedf5bbc21a99d3952b822863ba0c09097ebd3a3f80771d48a0b755b`。

### 2.2 `MSG_InitServiceDone`

`FUN_4da136c4` 写入一个单字段消息：

```text
int message_type = 2
```

同样经过 Base64，并追加 `<<`。

证据：`ev_0f73a9a624ecc7afe81b0d3f63e663e0bed351950c6e5752cd201851f4a87ee5`。

### 2.3 通用进度/结果消息候选

`FUN_4da13fa4` 的字段布局为：

```text
int      message_type
qint64   field_pair_1
int      field_3
```

`FUN_4da1404e` 的字段布局为：

```text
int      message_type
qint64   field_pair_1
qint64   field_pair_2
int      field_6
```

两者都采用相同的 Base64 + `<<` 封装，并由 `FUN_4da13378` / `FUN_4da133eb` 广播到当前连接集合。

证据：`ev_40d3ce77a7b4114a1a179c44a20ad8aff77da7e93055fb8e5dc017e493ccc5ba`、`ev_ec45c7a1e881118ac09b6331a66a52857a51465f3e19a86b15462526bb12c3b5`、`ev_abc9902d598ae639b6bfaa3e765d99a16191d88c2737f6a13db44800488ca0be`、`ev_30fa2e839c95e4de937343a80bf4fac605f4a862fa693de632152df1cd37d28f`。

字段语义仍需结合调用者和规则对象确认；目前只能称为布局候选。

## 3. 共享内存分片

`FUN_4da14bbe` 表明服务还使用 `QSharedMemory` 保存较大的盘扫描数据：

- 首次创建失败时将块大小从当前值减半，直到不超过 `0x20000`。
- 既有块会尝试 attach；创建或 attach 成功后取得 `data()` 指针。
- `FUN_4da1522a` 从共享内存读取固定头 `0x20` 字节，再读取若干变长分片。
- 每个分片包含长度字段；首字节/标志和第二字节参与分片遍历，空分片会终止当前读取。
- `FUN_4da15689` 读取分片数量，限制小于 `0x2711`（10001），按序建立对象并将其插入结果列表。

这说明本地 socket 主要负责控制/状态消息，而较大扫描结果通过共享内存分片传输；复刻时不能只实现单纯文本套接字。

证据：`ev_cf846316cfd5f2fc6af2d8c8e021c8a57681712a0e1b7c724c9514e0b838403f`、`ev_3e551daff0f44e64f23878ccb7ea4e63191ffb74d6cceb491364a83c23355f0d`、`ev_364e905948b969366f21e68fdfc1df8d0477ee77f0076bbae523764dff76773c`。

## 4. 仍未完全确定的部分

- `message_type` 的完整枚举仍未恢复；当前明确看到 `1` 和 `2`。
- `FUN_4da13fa4` / `FUN_4da1404e` 中 qint64 与整数的业务语义仍需从更上层调用点确认。
- 共享内存 native key 的具体生成规则、固定头字段含义及结果对象字段尚未完整恢复。
- 现有证据足以指导安全复刻的传输层原型，但还不足以声称与原程序协议完全兼容。

