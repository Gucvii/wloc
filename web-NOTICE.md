# WLOC 选点页（自建版）

## 来源

本页面为自建实现，协议细节参考：

- <https://github.com/xepes0/wloc> — `worker/src/wloc-settings.js`（本地写入接口）
  与 `worker/src/gcj-browser.js`（GCJ-02 换算），**AGPL-3.0**
- 上游许可证副本见本目录 `LICENSE`

## 接口（由代理脚本在本地拦截）

```
GET https://gs-loc.apple.com/wloc-settings/save?action=query
  → {success, longitude, latitude, accuracy, randomRadius, updatedAt}

GET https://gs-loc.apple.com/wloc-settings/save?action=clear
  → {success}

GET https://gs-loc.apple.com/wloc-settings/save?lat=..&lon=..&acc=..&randomRadius=..
  → {success, longitude, latitude, accuracy, randomRadius}
```

参数同时接受 `lat`/`latitude`、`lon`/`longitude`、`acc`/`accuracy`。
`action` 缺省为 `save`。

## 功能

- 独立的纬度 / 经度 / 精度 / 扰动半径输入框
- 地图选点：卫星 / 标准 / 高德 / 暗色四种底图
- 高德底图自动做 GCJ-02 ↔ WGS84 换算
- 储存到设备 / 查询当前 / 恢复真实定位
- 读取设备定位并与填写坐标比对距离
- 扰动半径与精度的一致性提示
- 上次填写的坐标保存在浏览器 localStorage

## 隐私

- 页面**无后端**，不上传坐标。
- 写入请求由手机端代理脚本本地拦截，不离开设备。
- 地图瓦片请求会暴露你正在查看的区域给瓦片服务商
  （OpenStreetMap / Esri / 高德 / Carto）。这是地图页面的固有特性。

## 许可证

AGPL-3.0。
