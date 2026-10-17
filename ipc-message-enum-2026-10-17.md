# IPC 消息枚举补充（2026-10-17）

本轮继续对 `ksearchservice.exe` 做静态地址级分析，未运行样本。

## 已确认消息类型

除此前已确认的 `1`、`2` 外，又恢复了两种编码函数：

### `message_type = 3`

`FUN_4da13efe` 序列化：

```text
int    3
ushort field_1
int    field_2
```

之后执行 Base64，并追加 ASCII `<<`。该消息由服务广播给全部当前连接。

证据：`ev_38008fcaa6bb35fdbb5c6f607c1fb848e5b168043b86e4ead6d3ad186d75a869`。

### `message_type = 8`

`FUN_4da14107` 序列化：

```text
int    8
int    field_1
ushort field_2
```

同样采用 Base64 + `<<` 封装并广播。

证据：`ev_f22e5bbe3d77baa7ed5059b11be76a9b15886f8abf248f7a0babf23723595393`。

## 读取方向的边界

当前外部导入中能看到 `QLocalSocket::bytesAvailable`、`canReadLine`、`waitForReadyRead` 和 `readData`，但关键调用地址落在 Ghidra 未归属函数的尾部/导入跳板区域，未能恢复成稳定的高层读取函数。因此目前可以确认“存在按可读数据/分隔符读取的接收端”，但不能安全断言具体的入站消息类型或是否严格按 `<<` 分帧。

证据：`ev_0ff87115a916f9cbd4508949e70e68ee2b162bb367e775dca1b62f798741eaa5`、`ev_7e483d692a34608b7923b1a69ecf50b03283af329b5d5fa95c9e25174535379d`、`ev_3aaa74069d7b2396ad263d48588da0f39b10e3364606a3386ca55923655edee2`。

## 当前枚举状态

已从静态序列化函数直接确认：`1, 2, 3, 8`。其余类型可能由未恢复的调用者或共享内存事件产生，尚未建立证据映射。消息字段语义仍需结合上层调用点和盘扫描对象字段确认。

