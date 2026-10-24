# 审计记录 AUDIT.md

本文件记录 `Gucvii/wloc`（fork 自 [`xepes0/wloc`](https://github.com/xepes0/wloc)）
的审计结论，以及**审计边界**。目的是让"信不信"变成可复现的检查。

- **审计日期**：2026-10-24
- **审计方法**：Git 溯源 + SHA-256 完整性校验 + Prettier 格式化后静态扫描
- **审计范围**：见下文第六节。**只覆盖脚本层，不覆盖闭源宿主。**

---

## 一、溯源：基线 commit 真实存在

`xepes0` 声称本仓库基于 `529fcd8`（2026-09-04）恢复，此说法经核实**成立**：

| 检查项 | 结果 |
| --- | --- |
| commit `529fcd841952571e79f40faf2d3e0ef90a7bdfa6` | ✅ 存在 |
| 作者 | `Yu9191`（原作者），2026-09-04 |
| 签名 | ❌ `verified: false` — unsigned |
| 历史连续性 | ✅ 有 parent（`e126446`），非孤立 commit |
| fork 后保留 | ✅ `Gucvii/wloc` 可 fetch 同一 commit |

原作者 `Yu9191/wloc` 目前 404。本仓库 `docs/PROVENANCE.md` 对此表述克制：
"它只证明当时公开不可访问，不能确定作者是否主动删除仓库" — 说法准确，未过度宣称。

---

## 二、完整性：四份副本字节完全一致

`dist/wloc.js` 在多个独立来源中哈希相同：

| 来源 | SHA-256 |
| --- | --- |
| `docs/upstream-integrity.json` 记录值 | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| 基线 commit `529fcd8:dist/wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| `main` 分支 `dist/wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| `zxishere/wloc` 的 `wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |

`dist/wloc-settings.js` 同样一致：`cbff047a7f42055615ee19e208ddfc2ec66833e6f2b75555eac84f1c299a66cf`

**意义**：多份独立镜像提供同一份产物，且都等于原作者基线 commit 的内容。
篡改需同时改动所有镜像，成本较高。这是本次审计最强的正面证据。

---

## 三、静态扫描

方法：

```bash
npx prettier@3 --parser babel dist/wloc.js > wloc.pretty.js            # 2562 行
npx prettier@3 --parser babel dist/wloc-settings.js > settings.pretty.js  # 776 行
```

### dist/wloc.js（主改写脚本）

| 检查项 | 命令 | 结果 |
| --- | --- | --- |
| 外发调用 | `grep -E '\$httpClient\|\$task\.fetch\|fetch\(\|XMLHttpRequest\|WebSocket\|sendBeacon'` | ✅ **零命中** |
| 动态执行 | `grep -E 'eval\(\|new Function\('` | ✅ **零命中** |
| 硬编码 URL | `grep -oE "['\"\`]https?://[^'\"\`]+"` | ✅ **零命中** |
| 持久化 | `grep -E '\$persistentStore\|\$prefs'` | 7 处，仅本地存储 API |

**结论**：脚本不会把任何数据发往任何地方（包括作者自己的服务器），
无运行时代码生成，目标域名全部来自模块配置、可人工核对。

### dist/wloc-settings.js（坐标写入拦截）

| 检查项 | 结果 |
| --- | --- |
| 外发调用 | ✅ **零命中** |
| 动态执行 | ✅ **零命中** |
| 硬编码 URL | ✅ **零命中** |
| 持久化 | 7 处，仅 `$persistentStore` / `$prefs`，无 Keychain、无文件系统 |

**结论**：与主脚本同级别。坐标保存请求被本地截住，不出设备。

### worker/src/*.js（Cloudflare Worker，仅快捷指令路线使用）

| 检查项 | 结果 |
| --- | --- |
| 外发调用 | **1 处**：`parse.js:173` 的 `fetch`，用于跟随短链重定向 |
| SSRF 防护 | ✅ `isFetchable()` 拒绝 `localhost` / `*.local` / `*.internal` / IP 字面量 |
| 重定向次数 | ✅ 上限 5 次 |
| 超时 | ✅ `AbortSignal.timeout` |
| 响应体读取 | ✅ `readCapped()` 限长，读满即断 |
| 缓存 | ✅ `/api/*` 强制 `Cache-Control: no-store` |
| 存储 / 日志 | ✅ 未发现写入存储或记录日志的代码 |

**结论**：工程实现克制，未发现越界行为。
**但需注意**：Worker 路线下，**你的坐标会经过第三方 Worker**（默认
`wloc-spoofer.dongtai230.workers.dev`）。README 声明"不写存储、不记日志"，
但无法从客户端验证。**在意隐私请自行部署，或只用网页选点。**

---

## 四、MITM 范围核查

`modules/*.module` 的 `[MITM] hostname` 共 5 个域名：

```
gs-loc.apple.com
gs-loc-cn.apple.com
gsp-ssl.ls.apple.com
bluedot.is.autonavi.com
bluedot.is.autonavi.com.gds.alibabadns.com
```

- ✅ **无通配符**（不存在 `*.apple.com` 这类越权）
- ✅ `[Script]` 的 `pattern` 精确匹配 `/clls/wloc` 路径
- ⚠️ `bluedot.is.autonavi.com` 是**高德**域名，拦截它覆盖国内高德定位生态，
  属功能必要，但**扩大了 MITM 面**。使用者应知晓：MITM 不止作用于 Apple。

---

## 五、已知不足（如实记录）

| # | 不足 | 影响 |
| --- | --- | --- |
| 1 | **`dist/wloc.js` 没有源码** | 审计只能对压缩产物做，无法验证"源码 → 产物"的构建过程。`docs/PROVENANCE.md` 已明说"未找到完整 `wloc.js` 源码" |
| 2 | commit 未签名 | `verified: false`，密码学上无法确认出自 Yu9191 |
| 3 | 无 release tag | 只能锁 commit SHA，没有版本锚点 |
| 4 | 历史是恢复的 | 46 条提交来自镜像，Issues / PR 讨论 / Release 附件丢失 |
| 5 | 扫描是静态的 | 未做运行时行为监控；未做模糊测试或符号执行 |

第 1 条最值得注意：理论上存在"源码干净、产物注入"的可能。
但三项零命中 + 四份哈希一致，让实际风险较低。
**这不是隐瞒，是上游原始状况，本仓库披露是诚实的。**

---

## 六、审计边界（最重要）

### ✅ 已审计

| 对象 | 方法 | 结论 |
| --- | --- | --- |
| `dist/wloc.js` | 格式化 + 四项 grep | 干净 |
| `dist/wloc-settings.js` | 格式化 + 四项 grep | 干净 |
| `worker/src/*.js` | 人工阅读 + grep | 未见越界 |
| `modules/*.module` | 人工核对 | 域名范围合理 |
| `docs/upstream-integrity.json` | 哈希比对 | 与实物一致 |
| `scripts/check.mjs` | 人工阅读 | 校验逻辑正确 |

### ❌ 未审计 / 不可审计

| 对象 | 原因 |
| --- | --- |
| **Shadowrocket / Surge / Loon 等宿主** | **闭源**，无法审计。持有根证书，理论可解密全部 HTTPS |
| iOS `locationd` | 闭源系统进程 |
| 第三方选点网页 | 取决于用哪一个 |
| iCloud 快捷指令 | 结构复杂、不透明，未逐个审计 |
| 第三方 Cloudflare Worker | 服务端行为无法从客户端验证 |

**核心提醒**：

> **脚本干净 ≠ 流量安全。** 宿主持有你信任的根证书，
> 理论上能解密你的全部 HTTPS 流量。降低这部分风险只能靠**缩短信任栈**
> （用完即关、或改用本地开源 MITM），不能靠审计脚本达成。

---

## 七、本 fork 的改动

**只新增文件，不改动任何被审计的二进制。**

| 新增 | 说明 |
| --- | --- |
| `AUDIT.md` | 本文件 |
| `modules/wloc-audited.module` | 日常用，脚本指向本仓库 `main` |
| `modules/wloc-pinned.module` | 高安全场景，脚本锁定到 commit SHA |

### 完整性守护

本仓库自带 `npm run check`，会校验 `dist/*.js` 是否与
`docs/upstream-integrity.json` 的记录一致：

```bash
npm run check
# → 仓库配置、模块链接、文档链接和上游文件完整性检查通过。
```

**只要 `dist/` 被改动，这个检查就会失败。**
这是使用 `main` 分支时的安全网：分支可以更新，但二进制一旦变了就会被发现。

手工校验：

```bash
shasum -a 256 dist/wloc.js
# 期望 a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1
shasum -a 256 dist/wloc-settings.js
# 期望 cbff047a7f42055615ee19e208ddfc2ec66833e6f2b75555eac84f1c299a66cf
```

---

## 八、复现本次审计

```bash
git clone https://github.com/Gucvii/wloc && cd wloc

# 1. 溯源
git cat-file -t 529fcd841952571e79f40faf2d3e0ef90a7bdfa6
git log --format='%H %ci %an %s' --all | tail -5

# 2. 完整性
shasum -a 256 dist/wloc.js dist/wloc-settings.js
cat docs/upstream-integrity.json
npm run check

# 3. 静态扫描
npx prettier@3 --parser babel dist/wloc.js > /tmp/wloc.pretty.js
npx prettier@3 --parser babel dist/wloc-settings.js > /tmp/settings.pretty.js
for f in /tmp/wloc.pretty.js /tmp/settings.pretty.js; do
  echo "== $f =="
  grep -nE '\$httpClient|\$task\.fetch|[^a-zA-Z]fetch\(|XMLHttpRequest|WebSocket|sendBeacon' "$f"
  grep -nE 'eval\(|new Function\(' "$f"
  grep -oE "['\"\`]https?://[^'\"\`]+" "$f" | sort -u
done
```

---

## 九、重要限制：iOS 27 beta 6 起失效

上游 `README.md` 与基线 commit `529fcd8` 的说明指出：

> iOS 27 beta 6 起，`locationd` 在 **TLS 证书校验层**拒绝任何非 Apple CA 签发的证书，
> 即使该 CA 已在「证书信任设置」中完全信任。同一设备同一 CA 拦截其他域名是成功的。

**这条限制的影响常被低估**：校验发生在**设备内部的 `locationd` 进程**，
而非流量路径上，因此：

- ❌ 换代理客户端无效
- ❌ 软路由 / 网关侧透明代理无效
- ❌ Mac 上跑 mitmproxy 同样无效
- ❌ 换证书、重装描述文件无效

> **问题出在路径终点，不在路径。**

因此本方案（及一切基于 MITM 改 `gs-loc` 响应的方案）在
**iOS 27 beta 6 及之后不可用**，只能停留在更低版本系统。

---

## 十、合规提醒

本项目仅用于自有设备的定位测试与学习。用于考勤代打卡等场景，
在多数用人单位规章下构成可解除劳动合同的严重违纪，并可能涉及欺诈。
技术可行不等于后果可承受。
