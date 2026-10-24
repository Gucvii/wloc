# WLOC 选点页（自建版）

## 来源

协议细节参考 <https://github.com/xepes0/wloc>（**AGPL-3.0**）：
`worker/src/wloc-settings.js`（本地写入接口）与 `worker/src/gcj-browser.js`（GCJ-02 换算）。
许可证副本见本目录 `LICENSE`。

## 接口（由代理脚本本地拦截）

```
GET https://gs-loc.apple.com/wloc-settings/save?action=query
  → {success, longitude, latitude, accuracy, randomRadius, updatedAt}
GET ...?action=clear
  → {success}
GET ...?lat=..&lon=..&acc=..&randomRadius=..
  → {success, longitude, latitude, accuracy, randomRadius}
```

参数同时接受 `lat`/`latitude`、`lon`/`longitude`、`acc`/`accuracy`，`action` 缺省为 `save`。

## 功能

- 纬度 / 经度 / 精度 / 扰动半径独立输入
- 地图选点：卫星 / 标准 / 高德 / 暗色，高德自动 GCJ-02 ↔ WGS84 换算
- 储存到设备 / 查询当前 / 恢复真实定位 / 读取设备定位并比对距离
- 扰动半径与申报精度的自相矛盾提示
- 上次填写内容保存在浏览器 localStorage

## 页面不含预设坐标

页面源码中没有任何硬编码经纬度，地图初始为世界视图，不显示标记。
坐标来源只有三种：手动输入、地图选点、按出口 IP 匹配。

## 按出口 IP 匹配

按钮会依次请求 `ipwho.is` 与 `ipinfo.io`（均允许跨域），取当前出口 IP 的
城市级经纬度填入输入框。这是**起点**，不是结论：IP 定位通常只到城市级，
可能与真实机房位置相差几十公里。查询会把你的 IP 发送给上述服务。

## 隐私

- 页面**无后端**，不上传坐标。
- 写入请求由手机端代理脚本本地拦截，不离开设备。
- 地图瓦片请求会暴露正在查看的区域给瓦片服务商（OSM / Esri / 高德 / Carto）。
- IP 匹配是显式操作，不会在页面加载时自动发起。

## 许可证

AGPL-3.0。
