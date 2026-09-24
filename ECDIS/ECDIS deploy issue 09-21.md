---
title: Dependency bug
description: map.lib's symbol does not match includes
---

## Symptom

- 链接 ECDIS（Debug/Release x64 均复现）报 LNK2019：无法解析 `GsChartManage::GetWaterLevelsInExtent(GsBBoxObject* const&, double, double, uint64_t, vector<_GsWaterLevelInfo>&)`（5 参数版）
- 引用者为 map.lib 内已编译的 `xnlib::BathymetryWaterLevelManager::PickupWaterLevelInView`。
- 复现步骤：VS 2022 或 msbuild 构建 ECDIS.vcxproj，编译通过，链接阶段失败。

## Root Cause

- map.lib 引用旧 5 参数符号（mangled `...NN_KAEAV...`），ecdis-sdk-2.lib 只实现新 6 参数版（多 `endTimestamp`，`...NN_K1AEAV...`），其余逐字节相同，map.lib 源码（`E:\Navi(MIBT-Mapbox)`）本机缺失，无法重编 map.lib
- map.lib 引用的 18 个 GsChartManage 符号仅缺此 1 个，Linux没问题

## Workaround

  1. 在 SDK 源码 `E:\mbgl-dev\ecdis-sdk-2\ecdis-sdk-2` 的 `GsChartManage.h/.cpp` 中新增 5 参数重载，转发 6 参数版（`endTimestamp=0`，语义等同旧行为）

静态库中的函数声明：

```cpp
bool GsChartManage::GetWaterLevelsInExtent(const GsBBox& extent,
                                           double lat, double lng,
                                           uint64_t timestamp,          
                                           GsWaterLevelInfos& waterLevels); 
```

源码中的函数声明：

```cpp
bool GetWaterLevelsInExtent(const GsBBox& extent,
                            const double lat, const double lng,
                            const uint64_t startTimestamp,
                            const uint64_t endTimestamp, 
                            GsWaterLevelInfos& waterLevels);
```

  2. 重建 ecdis-sdk-2.vcxproj（输出 `E:\ECDIS\x64\<Config>\ecdis-sdk-2.lib`）。
  3. 手动拷贝该 lib 到 `E:\ECDIS\NaviLib\Windows\x64\<Config>\`（ECDIS 仅从 NaviLib 链接）`GsChartManage` 备份在桌面，NaviLib 各配置在目录内 `ecdis-sdk-2.lib.bak_0920`

## Report Upstream

目前的源代码保留旧 5 参数签名，等 Navilib 更新 map.lib 后再拉取，在这之前不要拉任何东西
