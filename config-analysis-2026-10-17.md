# REA 配置与时间参数静态分析

样本：`ktrashscan.dll`，SHA-256 `c623ddf515f23d14eb5f96b30dc77b84842935212e02b32c319c5b28bf4f1c24`

## `data\\trashscan.ini`

证据：`ev_5708d4dbd3851e35187e78e284f82cbb1dbc41bae159ca41a9db33e6679a99c6`

初始化函数 `FUN_100a30f2`：

- 取得模块目录；
- 拼接 `data\\trashscan.ini`；
- 将该路径交给配置对象初始化；
- 同时初始化扫描/结果相关状态字段。

因此 `trashscan.ini` 是引擎初始化配置，不是规则 XML 本身。

## `data\\trashtime.dat`

字符串交叉引用证据：`ev_b9dbabe2994481189a3d9575da96d7c143ba8a9b628166e5ff02f4e63fc19e76`

`FUN_100ac54a` 静态读取：

```text
[common]
failtime = 0      ; 默认值
failcount = 0     ; 默认值
```

读取 API 是 `GetPrivateProfileStringW`，随后按十进制转换。该函数还调用 `FUN_100accce()` 初始化一个运行时值。

另一个初始化函数 `FUN_100ada36` 读取：

```text
[common]
scandetail  = 0   ; 默认值
detectdetail = 0  ; 默认值
```

它还从环境/系统接口读取若干运行时字段，说明该文件控制扫描失败、扫描细节和检测细节统计，而不是单纯的“文件超过几天可删”阈值。已确认的 7 天默认阈值仍来自 `NeedDel` 的代码常量 `0x93A80`。

## 对复刻的影响

- 复刻程序可以兼容读取这两个 INI 风格配置，但应将未知字段保留为只读展示。
- `failtime`、`failcount`、`scandetail`、`detectdetail` 不能直接解释成删除阈值。
- 删除年龄阈值应继续采用已证实的 `NeedDel` 逻辑：负 limit 使用 7 天，非负 limit 使用 `limit * 60` 秒。
- `GetPrivateProfileStringW` 表明配置格式是普通 Windows INI 读取，不需要运行原程序即可在复刻器中安全解析。

## 尚未确认

- `trashscan.ini` 的具体键名和默认值；
- `FUN_100accce()` 返回值的精确定义；
- `failtime/failcount` 是否影响失败重试或仅用于统计；
- `kduba_trashscan_use_file_cache` 对扫描结果缓存的具体开关行为。
