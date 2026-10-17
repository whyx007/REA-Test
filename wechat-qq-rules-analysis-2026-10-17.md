# 微信/QQ 清理规则与功能包分析（2026-10-17）

## 范围与限制

本报告只使用已保存的 DLL/EXE、规则文件、配置文件和历史日志进行静态分析；没有启动搜狗清理程序，也没有执行删除。日志中的运行信息仅作为样本随附证据，不能等同于本次重新运行得到的事实。

## 1. 规则文件的职责边界

`.dat` 文件有固定 0x50 字节头（包含 `HLKBP`），正文从偏移 `0x50` 开始逐字节 XOR `0xAB`，解码后是 UTF-8 XML。解码副本位于 `decoded2/`。

`kwclitesearch.dat` 和 `kqqlitesearch.dat` 的 XML 只有以下信息：

- `rule id`、`uuid`、`typeid=1`、`limit=0`
- 中文分类描述
- 注释说明该文件只关注规则格式，过滤等业务关联信息不在此处

因此，轻量规则不是路径清单。仅凭它们不能复刻扫描路径、文件匹配表达式或误报过滤。

## 2. 微信深度分类映射

来自 `kwcdeepclean.dat` 解码 XML：

| 分类 | 子分类 | rule id | 展示 | 目录删除 | 备注 |
|---|---|---:|---:|---:|---|
| 聊天视频 | 聊天视频 | 1008 | 2 | 1 | 按时间排序；虚拟年月分组；可打开 |
| 聊天图片 | 聊天图片 | 1007 | 1 | 1 | 按时间排序；虚拟年月分组 |
| 接收文件 | 办公文档/视频/图片/音乐/压缩包/其他 | 1018–1022、1009 | 0/1 | 1 | `show=0`，按时间排序 |
| 缓存 | 更新缓存/小程序缓存/公众号缓存/表情/资讯/其他/手机备份 | 1003、1004、1005、1015、1017、1006、1014 | 0/1 | 0/1 | 多数 `limit.selected=1`；表情和手机备份默认未选 |
| 日志 | 微信日志 | 1001、1002 | 0 | 0 | 一级分类 `show=2`；默认选中 |
| 朋友圈 | 朋友圈缓存 | 1016 | 0 | 1 | 一级分类 `show=2`；默认选中 |

`displaypatten` 的含义由 XML 注释定义：0 为传统文件列表，1 为图片/视频/封面展示，2 为视频/动态图展示。`deletefolder=0` 表示分类不允许把目录作为删除对象；这不是“文件绝对不会删除”，而是目录级删除开关。

## 3. QQ 深度分类映射

来自 `kqqdeepclean.dat` 解码 XML：

| 分类 | 子分类 | rule id | 展示 | 目录删除 | 默认/排序 |
|---|---|---:|---:|---:|---|
| 聊天视频 | 聊天视频 | 2001 | 2 | 1 | 默认选中，按时间，虚拟年月 |
| 聊天图片 | 聊天图片 | 2002、2003 | 1 | 1 | 默认选中，按时间，虚拟年月 |
| 缓存 | 软件/插件/更新/表情 | 2004–2009（2006为兼容保留） | 0 | 1 | 默认选中，时间排序 |
| 聊天语音 | 聊天语音 | 2011 | 0 | 1 | `selected=0`，虚拟年月 |
| 接收文件 | 办公/视频/图片/音乐/压缩包/其他 | 2015、2012–2014、2016、2010 | 0/1 | 0 | 默认选中，大小排序 |

XML 中明确注释：QQ 空间缓存规则在新版本可能没有内容，但规则仍保留并并入其他分类。接收文件各子分类 `deletefolder=0`，说明复刻时应默认只展示文件，不把整个接收目录作为删除对象。

## 4. 功能包与插件边界

插件 DLL 内部还嵌有软信息 XML，静态字符串证据显示：

- `kwechatcleanup.dll`：`soft_id=70005042`，`kismain_command=kwechatcleanup`，引擎配置 `kwcdeepclean.dat`，轻量规则 `kwclitesearch.dat`，另引用 `kwclitesearch_prescan.dat`；权限键为 `kclean_weixin`，缓存组为 `wechatclean`，预扫描/界面互斥体分别带有固定 GUID。
- `kqqcleanup.dll`：`soft_id=60000116`，`kismain_command=kqqcleanup`，引擎配置 `kqqdeepclean.dat`，轻量规则和预扫描规则均指向 `kqqlitesearch.dat`；权限键为 `kclean_qq`，缓存组为 `QQclean`，同样定义独立 UI/预扫描互斥体。

这些字段确认了“主程序 -> 专用插件 -> 规则文件”的调用边界，也确认微信存在一个单独的 `kwclitesearch_prescan.dat` 引用；当前样本目录未保存该文件，因此预扫描规则的完整内容仍未知。上述字符串来自 REA Ghidra 静态字符串证据：微信 `ev_3b80e744fefd5a96d729745ef50769039fae4892a8d2039c771db4bbe45d7b97`，QQ `ev_f0fdeb401181b4a66d965264da20a2125dcfaef2d66ada180dc5009f22525e47`。

历史随附日志 `ksgdiskcleanup.KSubpackDownload.dll.log` 显示：

- 主程序分别请求 `wechat_clean` 和 `qq_clean` 功能；两者各自声明 `subPackCount=7`。
- 首次缺少组件时下载 `pkg_bigfile` 和 `pkg_qq_we_pp`，后者 URL 形如 `.../subpacks/f74ceaf8ffc5170e486acd54315af4b4.7z`。
- 下载完成后将子包复制到主目录，并清理临时子包目录；后续启动通过“full flag file exist”跳过再次下载。

这说明实际路径脚本很可能位于 `pkg_qq_we_pp` 子包或其释放出的组件中。当前样本目录没有保留该 7z/释放目录，因此路径展开仍是未恢复项。报告不能把常见微信/QQ目录名当成已证实的原程序规则。

## 5. 风险提示与界面行为

`data/kcleanerselectallrisk.xml` 为全选删除风险提示：

- 微信（soft id `70005042`）：提示可能导致接收文件、缓存图片丢失。
- QQ（soft id `60000116`）：提示可能导致接收文件、图片、表情、聊天记录丢失。

这与深度 XML 中“接收文件/聊天媒体”是可删除分类相互印证，但不提供路径或实际删除算法。

## 6. 对安全复刻的直接规格

第一版复刻可以安全实现：

1. 读取深度 XML 建立 `soft -> category -> subcategory -> rule id` 映射。
2. 读取轻量 XML 展示规则 UUID 和描述，但不把描述当作路径。
3. 以只读扫描/预览为默认模式；每个结果记录路径、大小、最后写入时间、规则 ID、分类、`deletefolder` 和来源证据。
4. 对目录级删除严格遵守 `deletefolder`；不允许因规则命中而自动删除父目录。
5. 对 `selected=0` 的 QQ 语音、微信表情/手机备份等分类默认不勾选。
6. 在没有恢复功能包路径脚本前，微信/QQ扫描器只提供“待实现规则”状态，不猜测并执行删除。

## 7. 尚未解决的问题

- `pkg_qq_we_pp` 子包中的真实路径/脚本/过滤器格式未恢复。
- 微信/QQ安装位置发现、便携版、多用户和注册表回退逻辑未完全恢复。
- `KTrashScanFalseAlarmKiller` 的 `regfilter/softfilter/pluginfilter` 数据格式未恢复。
- UI 插件调用协议和结果合并字段仍只有部分静态证据。

## 证据索引

- 规则原件：`rules/kwc*.dat`、`rules/kqq*.dat`
- 解码副本：`decoded2/kwclitesearch.xml`、`decoded2/kqqlitesearch.xml`、`decoded2/kwcdeepclean.xml`、`decoded2/kqqdeepclean.xml`
- 风险配置：样本目录 `data/kcleanerselectallrisk.xml`
- 功能包日志：样本目录 `log/ksgdiskcleanup.KSubpackDownload.dll.log`
- REA 静态证据总包：`rea-evidence-bundle.json`

