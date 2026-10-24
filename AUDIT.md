# 审计记录 AUDIT.md

本文件记录 `Gucvii/wloc`（fork 自 [`xepes0/wloc`](https://github.com/xepes0/wloc)）
的审计结论。目的是把"信任"变成可复现的检查，而不是靠印象。

- **审计日期**：2026-10-24
- **审计对象**：`dist/wloc.js`、`dist/wloc-settings.js`、`modules/wloc.module`
- **审计方法**：Git 溯源 + SHA-256 完整性校验 + Prettier 格式化后静态扫描
- **审计范围**：**仅脚本层**。宿主（Shadowrocket 等闭源代理客户端）不在可审计范围内，见下文"边界"。

---

## 一、溯源结论：基线 commit 真实存在

`xepes0` 声称本仓库基于 `529fcd8`（2026-09-04）恢复，此说法经核实**成立**：

| 检查项 | 结果 |
| --- | --- |
| commit `529fcd841952571e79f40faf2d3e0ef90a7bdfa6` 是否存在 | ✅ 存在 |
| 作者 | `Yu9191`（原作者），2026-09-04 |
| 签名 | ❌ `verified: false` — unsigned |
| 历史连续性 | ✅ 有 parent（`e126446`），非孤立 commit |
| fork 后是否保留 | ✅ `Gucvii/wloc` 中可 fetch 到同一 commit |

原作者 `Yu9191/wloc` 目前 404。该仓库的 `docs/PROVENANCE.md` 对这一点表述克制：
"它只证明当时公开不可访问，不能确定作者是否主动删除仓库" — 这个说法是准确的，
未过度宣称。

---

## 二、完整性：三份副本字节完全一致

同一份 `dist/wloc.js` 在本仓库记录、基线 commit、以及第三方镜像中**哈希相同**：

| 来源 | SHA-256 |
| --- | --- |
| `docs/upstream-integrity.json` 记录值 | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| 基线 commit `529fcd8:dist/wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| `main` 分支 `dist/wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |
| `zxishere/wloc` 的 `wloc.js` | `a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1` |

`dist/wloc-settings.js` 同样三方一致：`cbff047a7f42055615ee19e208ddfc2ec66833e6f2b75555eac84f1c299a66cf`

**意义**：多份独立镜像提供同一份二进制，且都等于原作者基线 commit 的内容。
篡改需要同时改动所有镜像，成本较高。这是本次审计最强的正面证据。

---

## 三、静态扫描：脚本本身无外发、无动态执行

方法：

```bash
npx prettier@3 --parser babel dist/wloc.js > wloc.pretty.js   # 2562 行
```

### ① 外发调用 — 零命中

```bash
grep -nE '\$httpClient|\$task\.fetch|[^a-zA-Z]fetch\(|XMLHttpRequest|WebSocket|sendBeacon' wloc.pretty.js
# → 无输出
```

**结论**：脚本不会把任何数据发往任何地方（包括作者自己的服务器）。

### ② 动态执行 — 零命中

```bash
grep -nE 'eval\(|new Function\(|[^a-zA-Z.]Function\(' wloc.pretty.js
# → 无输出
```

**结论**：无运行时代码生成，不存在"表面干净、实际执行别的代码"的路径。

### ③ 硬编码 URL — 零命中

```bash
grep -oE "['\"\`]https?://[^'\"\`]+" wloc.pretty.js | sort -u
# → 无输出
```

**结论**：脚本内不含任何 URL 常量。目标域名全部来自模块的 `pattern` / `hostname`
配置，可人工核对。

### ④ 持久化读写 — 仅限本地存储 API

```bash
grep -nE '\$persistentStore|\$prefs' wloc.pretty.js
```

命中 7 处，全部为 `$persistentStore.read/write` 与 `$prefs.valueForKey/setValueForKey/
removeValueForKey/removeAllValues`。无 Keychain 访问，无文件系统访问。

---

## 四、MITM 范围核查

`modules/wloc.module` 的 `[MITM] hostname` 共 5 个域名：

```
gs-loc.apple.com
gs-loc-cn.apple.com
gsp-ssl.ls.apple.com
bluedot.is.autonavi.com
bluedot.is.autonavi.com.gds.alibabadns.com
```

- ✅ **无通配符**（不存在 `*.apple.com` 这类越权）
- ✅ `[Script]` 的 `pattern` 精确匹配 `/clls/wloc` 路径
- ⚠️ `bluedot.is.autonavi.com` 是**高德**域名。拦截它是为了覆盖国内高德定位生态，
  属功能必要，但**扩大了 MITM 面**。使用者应知晓：MITM 不止作用于 Apple。

---

## 五、已知不足（如实记录）

### 1. `dist/wloc.js` 没有源码

仓库只有压缩后的 `dist/wloc.js`，**没有对应源码**。`docs/PROVENANCE.md` 已明说：

> 未找到完整 `wloc.js` 源码、根 package.json 或 Rollup 工程。

**影响**：本次审计是对**压缩产物**做的，无法验证"源码 → 产物"的构建过程。
理论上存在源码干净、产物被注入的可能，但本次扫描未发现任何外发 / 动态执行痕迹，
且三方哈希一致，实际风险较低。

**这不是隐瞒，是上游原始状况。** 仓库对此披露是诚实的。

### 2. commit 未签名

`529fcd8` 的 `verification.verified = false`。GitHub 上的签名状态无法证明它确实出自
`Yu9191` 之手（虽然 author 字段如此宣称）。

### 3. 无 tag

仓库没有 release tag，因此只能锁 commit SHA，没有语义化的版本锚点。

### 4. 历史是恢复的，不是原库

46 条可达提交来自镜像恢复。Issues / PR 讨论 / Release 附件未恢复，
意味着**上游的讨论上下文丢失**，无法回溯某些设计决策的原因。

---

## 六、审计边界（必须明确）

本次审计**只覆盖脚本**。以下**不在范围内**：

| 对象 | 状态 |
| --- | --- |
| `dist/wloc.js` / `dist/wloc-settings.js` | ✅ 已审计，结论见上 |
| `modules/*.module` | ✅ 已人工核对 |
| Shadowrocket / Surge / Loon 等宿主 | ❌ **闭源，不可审计** |
| iOS 系统 `locationd` | ❌ 闭源 |
| 第三方选点网页 / iCloud 快捷指令 / Cloudflare Worker | ⚠️ 取决于具体使用哪一个 |

**关键提醒**：宿主客户端持有你信任的根证书，理论上能解密你的全部 HTTPS 流量。
脚本干净 ≠ 流量安全。降低这部分风险只能靠**缩短信任栈**（如用完即关、
或改用本地开源 MITM），不能靠审计脚本达成。

---

## 七、本 fork 的改动

**只增加文件，不改动任何被审计的二进制。**

| 新增 | 说明 |
| --- | --- |
| `AUDIT.md` | 本文件 |
| `modules/wloc-pinned.module` | 脚本路径锁定到 `529fcd8`，不使用 `main` |

审计对象文件保持原样。可用以下命令验证未被修改：

```bash
shasum -a 256 dist/wloc.js
# 期望 a1b361e60f0b434585260fb59c65d1ddbe3bff89ace3639f592e7d8af432b3c1
shasum -a 256 dist/wloc-settings.js
# 期望 cbff047a7f42055615ee19e208ddfc2ec66833e6f2b75555eac84f1c299a66cf
```

---

## 八、复现审计的命令

```bash
git clone https://github.com/Gucvii/wloc && cd wloc

# 1. 溯源
git cat-file -t 529fcd841952571e79f40faf2d3e0ef90a7bdfa6
git log --format='%H %ci %an %s' --all | tail -5

# 2. 完整性
shasum -a 256 dist/wloc.js dist/wloc-settings.js
cat docs/upstream-integrity.json

# 3. 静态扫描
npx prettier@3 --parser babel dist/wloc.js > /tmp/wloc.pretty.js
grep -nE '\$httpClient|\$task\.fetch|[^a-zA-Z]fetch\(|XMLHttpRequest|WebSocket|sendBeacon' /tmp/wloc.pretty.js
grep -nE 'eval\(|new Function\(' /tmp/wloc.pretty.js
grep -oE "['\"\`]https?://[^'\"\`]+" /tmp/wloc.pretty.js | sort -u
grep -nE '\$persistentStore|\$prefs' /tmp/wloc.pretty.js
```

---

## 九、重要限制：iOS 27 beta 6 起失效

上游 `README.md` 与基线 commit 的说明指出：

> iOS 27 beta 6 起，`locationd` 在 **TLS 证书校验层** 拒绝任何非 Apple CA 签发的证书，
> 即使该 CA 已在「证书信任设置」中完全信任。同一设备同一 CA 拦截其他域名是成功的。

**这条限制的影响被广泛低估**：因为校验发生在**设备内部的 `locationd` 进程**，
而非流量路径上，所以：

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
