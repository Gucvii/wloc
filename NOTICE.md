# 选点页来源与改动说明

本目录的 `index.html` 是 WLOC 地图选点页的**静态托管副本**。

## 上游

- 来源仓库：<https://github.com/xepes0/wloc>
- 源文件：`worker/src/page.js` 导出的 HTML（Worker 动态渲染版）
- 上游许可证：**AGPL-3.0**（本目录 `LICENSE` 为原样副本）
- 生成日期：2026-10-24

## 本副本的改动

只做两处，其余保持原样：

1. **模块订阅链接**改为指向本仓库（`Gucvii/wloc`），不再指向上游。
2. **移除 `/api/parse` 链接解析**：该功能依赖 Worker 服务端展开短链并做 GCJ-02→WGS84
   换算，静态托管无法提供。页面会提示改用地图选点或手填经纬度。

地图底图、坐标换算、本地存储、`gs-loc.apple.com/wloc-settings/save` 写入逻辑
均与上游一致。

## 生成方式（可复现）

```bash
git clone https://github.com/Gucvii/wloc
cd wloc/worker/src

# project.js 中的 SOURCE_URL 与 MODULE_LINKS 已指向本仓库
node -e "import('./page.js').then(m=>require('fs').writeFileSync('index.html', m.getPageHtml()))"
```

随后对产出的 `index.html` 应用上述两处改动。

## 隐私说明

- 页面本身**无后端**，不收集、不上传任何数据。
- 坐标通过 `https://gs-loc.apple.com/wloc-settings/save` 写入设备，该请求由手机端
  代理脚本在本地截住，不离开设备。
- 页面会向地图瓦片服务商（OpenStreetMap / ArcGIS / 高德 / Carto）请求当前视野的
  瓦片，**服务商可据此得知你正在查看的区域**。这是地图类页面的固有特性。
- 地名搜索会把关键词发送至 `nominatim.openstreetmap.org`。

## 许可证

AGPL-3.0。详见 `LICENSE`。
