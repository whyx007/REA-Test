# QQ/微信清理插件接口静态分析

目标样本：`pkg_qq_we_pp-extracted/kqqcleanup.dll`。本轮仅使用 Ghidra 静态字符串与符号证据，没有加载或运行 DLL。

## 1. 插件元配置

DLL 内嵌两段 `<soft>` XML，分别对应微信和 QQ：

### 微信

```xml
<soft_id>70005042</soft_id>
<kismain_command>kwechatcleanup</kismain_command>
<engine_cfg>kwcdeepclean.dat</engine_cfg>
<rule_cfg>kwclitesearch.dat</rule_cfg>
<pre_scan_rule_cfg>kwclitesearch_prescan.dat</pre_scan_rule_cfg>
<cache_group>wechatclean</cache_group>
<free_switch>kwechatcleanup_vip_free_switch</free_switch>
<permission>kclean_weixin</permission>
<activity_scene>28</activity_scene>
```

### QQ

```xml
<soft_id>60000116</soft_id>
<kismain_command>kqqcleanup</kismain_command>
<engine_cfg>kqqdeepclean.dat</engine_cfg>
<rule_cfg>kqqlitesearch.dat</rule_cfg>
<pre_scan_rule_cfg>kqqlitesearch.dat</pre_scan_rule_cfg>
<cache_group>QQclean</cache_group>
<free_switch>kqqcleanup_vip_free_switch</free_switch>
<permission>kclean_qq</permission>
<activity_scene>29</activity_scene>
```

两段配置还含有 `feedback_vfrom`、`vip_page_type`、`vip_spid`、全局 UI/预扫描互斥体、隐私名称和统计字段。它们说明插件由统一宿主加载，规则文件和引擎文件通过配置选择，而不是硬编码在主界面中。

## 2. 可确认的信号/槽边界

定义字符串包括：

- `scanFinishSignal`
- `changePreScanSignal`
- `changeAutoScanSignal`
- `changeScanStartTimeSignal`
- `changePreScanMutexSignal`
- `changePreScanRuleCfgSignal`
- `updateScanProgressSlot`
- `updateSlimProgressSlot`
- `restartScanSlot`
- `scanFinishSlot`

这些名称支持以下复刻层次：

```text
宿主 -> 插件：加载配置、开始预扫描/正式扫描、取消或重启
插件 -> 宿主：扫描开始时间、预扫描状态、规则配置变化、进度、完成
```

这是接口方向和事件名称的静态证据；Qt 信号的完整参数列表仍未从现有证据中恢复。

## 3. 结果上报线索

RTTI/符号字符串中出现：

```text
reportScanData@KFunctionSet@@YAXH_N0@Z
```

按 MSVC 修饰名可观察到返回类型为 `void`，参数数量候选为 3 个，其中包含两个布尔参数；第一个参数可能是整型分类/结果码。由于当前没有完整 PDB 类型信息，不能把这三个参数命名成确定业务字段。

另有统计模板字符串：

```text
duba_diskmanagement_bigscantime:8378
duba_diskmanagement_prescan:8371
```

它们携带 `froming`、`scantime`、`filetype`、`product_id`、`version`、`size`、`number`、`scene`、`memory`、`filegroup`、`recommend` 等字段，表明扫描结果至少会被转换成分类、数量、大小和推荐状态等 UI/统计数据。

## 4. 安全复刻接口建议

第一版不要复刻 ABI，而应定义进程内抽象接口：

```text
PluginMetadata load_metadata()
ScanHandle start_prescan(ScanContext)
ScanHandle start_scan(ScanContext)
void cancel(ScanHandle)
event progress(percent, phase)
event result(category, path, size, risk, reason)
event finished(summary)
```

将 `soft_id`、规则文件名、权限名、互斥体名称和活动场景作为元数据字段保存。未知的原始 Qt 参数、COM/ATL 对象布局和 DLL 导出 ABI 不应凭猜测实现。

## 5. 证据边界

- `ev_82cfe5bf5e79cb50167cb4477c216eb0d03ca89d4e833021b1246fbef316f420`：配置 XML、扫描事件和统计模板字符串。
- `ev_237921009a4902a8b81192e862bd868b195103576918abef1254fb9f136a3c6c`：进度槽、扫描完成槽和 Qt UI 类型字符串。
- `ev_1a1392c3c987ce997c4ad98744bb33889deac6f605a10b4a72d9938fe28e8d5c`：QQ/微信配置全文及 `reportScanData` 修饰名。

上述证据证明的是二进制中存在这些字符串和符号，不等于已经观察到运行时调用顺序或参数值。
