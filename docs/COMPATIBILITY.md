# 版本兼容性核查

本文记录脚本对各 Typora 版本的适配结论与核查证据。原理说明见 [TECHNICAL.md](./TECHNICAL.md)，变更历史见 [CHANGELOG.md](./CHANGELOG.md)。

## 已验证版本

| Typora | Electron | 结论 |
|---|---|---|
| 1.13.7 | — | ✅ 长期验证稳定 |
| 1.14.6 | — | ✅ 已验证 |
| 1.14.10 | 42.2.0 | ✅ 已验证（需脚本 v1.0.4+，机器码字段回退） |

> ⚠️ 激活前必须关掉 Typora 的「自动检查更新」，否则官方安装包会覆盖安装目录，已注入的补丁被还原。

---

## 1.14.10 核查详情

核查对象：`C:\Program Files\Typora`（1.14.10，releaseId `49b5981e`，内置 Electron **42.2.0**）。

### 1.1 安装目录结构

```
Typora/
├── Typora.exe                    227 MB
└── resources/
    ├── app.asar                    390 KB   ← 注入目标
    ├── lib.asar                   8.3 MB    渲染进程库（MathJax/CodeMirror/mermaid…）
    ├── node_modules.asar          9.3 MB    native-reg / electron-fetch / node-machine-id…
    ├── page-dist/                             渲染层页面（license.html 等）
    └── window.html
```

关键点：**主进程代码全在 `app.asar` 里**，`lib.asar` / `node_modules.asar` 不需要改动，因此注入 `app.asar` 即可覆盖授权逻辑。

`app.asar` 内容（asar header 直接解析得到）：

| 文件 | 大小 | 说明 |
|---|---|---|
| `atom.compiled.dist.jsc` | 388,344 | webpack 打包后的主进程代码（V8 字节码），**授权逻辑在此** |
| `launch.dist.js` | 1,383 | 入口，负责加载 `.jsc` |
| `package.json` | 252 | `"main": "launch.dist.js"` |

### 1.2 注入点核查

脚本用 `/require\s*\([^)]*\)\s*;/` 定位插入位置。`launch.dist.js` 开头是压缩过的单行代码：

```js
"use strict";const fs=require("fs"),path=require("path"),vm=require("vm"),v8=require("v8"),
Module=require("module");v8.setFlagsFromString("--no-lazy"),…
```

前 4 个 `require()` 后面跟的是 `,` 而非 `;`，因此正则命中的是 `require("module");`（**偏移 116**），落在完整的 `const` 语句之后 —— 语法合法。

| 检查项 | 结果 |
|---|---|
| 注入点存在且合法 | ✅ 命中 `require("module");` |
| 注入代码单独解析 | ✅ `new vm.Script()` 通过（10,340 字节） |
| 拼进真实 `launch.dist.js` 后解析 | ✅ 通过（11,721 字节） |
| 与入口文件的顶层标识符冲突 | ✅ 无（注入的 41 个顶层名字与 `fs,path,vm,v8,Module` 无重叠；`fs` 复用外层作用域） |
| Hook 早于业务代码执行 | ✅ 插入点在末尾 `require("./atom.compiled.dist.jsc")` 之前 |

> 为什么能改 `.jsc`：入口 `launch.dist.js` 是纯 JS，可直接改写；而 `.jsc` 是字节码，改不动也不需要改 —— 授权校验调用的是 `crypto` / `fs` / `net` 这些 Node API，在更早的层面劫持即可。

### 1.3 Electron Fuses 核查

在 `Typora.exe` 中定位哨兵 `dL7pKGdnNz796PbbjQWNKmHXBZaB9tsX`（全文件仅 **1 处**），其后为：

| 偏移 | 值 | 含义 |
|---|---|---|
| +32 | `01` | fuse_version = V1 |
| +33 | `09` | fuse_wire_length = 9 |
| +34…+42 | `30 30 31 30 30 31 30 31 31` | 9 个开关（ASCII `'0'`/`'1'`） |

解读（顺序取自 `build/fuses/fuses.json5`）：

| # | fuse | 值 | 说明 |
|---|---|---|---|
| 0 | `run_as_node` | `0` | |
| 1 | `cookie_encryption` | `0` | |
| 2 | `node_options` | `1` | |
| 3 | `node_cli_inspect` | `0` | |
| 4 | `embedded_asar_integrity_validation` | **`0`** | **未开启** —— 不会触发 asar 完整性校验 |
| 5 | `only_load_app_from_asar` | **`1`** | **已开启** —— 正是脚本要翻转的那一位 |
| 6 | `load_browser_process_specific_v8_snapshot` | `0` | |
| 7 | `grant_file_protocol_extra_privileges` | `1` | |
| 8 | `wasm_trap_handlers` | `1` | |

两个关键结论：

1. **`embedded_asar_integrity_validation = 0`** —— 这是新版 Electron 最容易踩的坑：一旦开启，解包 asar 后 Electron 会在 `Archive::Init()` 校验头哈希，失败即 `LOG(FATAL)` 直接终止进程。1.14.10 未开启，所以这条路可用。
2. **`only_load_app_from_asar = 1`** —— 必须翻转。Electron 的 `LoadAppPackage()` 搜索顺序为：

   ```cpp
   const bool only_asar = fuses::IsOnlyLoadAppFromAsarEnabled();
   if (only_asar) {
     candidates = {"app.asar"};
   } else {
     candidates = {"app.asar", "app", "default_app.asar"};   // ← 走这条
   }
   ```

   翻转后 `app.asar`（已被改名）读不到 `package.json` 就自动回退到 `resources/app/` 目录。

**`flipFuses` 实测**（在 `Typora.exe` 的临时副本上执行，真实安装未触碰）：

```
   RunAsNode                                    0 -> 0
   EnableCookieEncryption                       0 -> 0
   EnableNodeOptionsEnvironmentVariable         1 -> 1
   EnableNodeCliInspectArguments                0 -> 0
   EnableEmbeddedAsarIntegrityValidation        0 -> 0
>> OnlyLoadAppFromAsar                          1 -> 0
   LoadBrowserProcessSpecificV8Snapshot         0 -> 0
   GrantFileProtocolExtraPrivileges             1 -> 1
   WasmTrapHandlers                             1 -> 1
```

只有第 5 位被改写，其余 8 位逐一比对未变，重读稳定。`@electron/fuses` 2.1.3 是纯 ESM 包，`require()` 需 Node ≥ 22.12。

### 1.4 授权链路核查

从 `atom.compiled.dist.jsc` 的 V8 常量池抽出的字符串（按偏移排序），确认授权相关调用链在 1.14.10 中依然存在：

| 偏移 | 字符串 | 作用 |
|---|---|---|
| 86881 | `native-reg` | 注册表存取模块 |
| 86905 | `Software\Typora` | 授权键 |
| 87138 | `[WindowsLicenseLocalStore] ` | 读取 SLicense / IDate 的日志前缀 |
| 89618 | **`publicDecrypt`** | **RSA 公钥解密 —— Hook 点** |
| 89679 | `jsDecrypt` | 纯 JS 解密兜底 |
| 93109 / 93185 | `pure = ` / `pure js failed ` | 原生解密失败时回退纯 JS |
| 93078 | `base64` | SLicense 字段解码 |
| 96893 | `machineId` | 由 `HKLM\…\Cryptography\MachineGuid` 派生 |
| 94242 | `fingerprint` | 设备指纹 |
| 96977 | `[/=+-]` | 自定义 base64 字符表 |
| 97594 | **`2nd `** | **二次校验日志前缀** |
| 98691 | `equals` | 紧邻 `2nd`，即二次校验的比较逻辑 |
| 99534 | `[watch L] hasL: ` | license 状态监听 |
| 103878 | `renew` / `[renewLicense] license renewed in 12h` | 本地续订 |
| 104684 | `[renewL]: unfill due to renew fail` | 续订失败 → 吊销 |
| 107673 | `trailRemains is ` | 试用剩余天数计算 |
| 107766 | `[L] installDate is ` | 安装日期（读注册表 `IDate`） |
| 108382 | `[L] pass~` | 校验通过标记 |
| 117994 | **`api/client/activate`** | **激活接口 —— Hook 点** |
| 118055 | `[License] response code is ` | 服务端返回码 |
| 122888 | `api/client/deactivate` | 注销接口 |
| 121257 | `[machineCode] ` | 机器码生成入口 |

结论：脚本的三类 Hook（`crypto.publicDecrypt`、`net.request`/`net.fetch`/`protocol.handle` 的 `api/client/*`、`setTimeout` 屏蔽 `2nd` 校验）在 1.14.10 上都有对应的真实调用点。

### 1.5 Electron 42 API 核查

注入代码用到的 API 在 42.2.0 上均可用：

| API | 最低版本 | 42.2.0 |
|---|---|---|
| `protocol.handle(scheme, handler)` | 25 | ✅ |
| `net.fetch(input, init)` | 22 | ✅ |
| `net.request(options)` | 7 | ✅ |
| `session.defaultSession.clearStorageData()` | — | ✅ |
| `app.on("browser-window-created")` / `whenReady()` | — | ✅ |

### 1.6 机器码字段差异（v1.0.4 修复项）

1.14.10 实际发出的激活请求体（取自 `typora.log`）：

```json
{"v":"win|1.14.10","license":"…","email":"…",
 "l":"MAS | yy | Windows","f":"8wRo3r4xbe",
 "u":"d74f3d8e-2af9-4a18-bd47-96fa21842eb8","type":"","force":false}
```

即新版用 `f` 表示指纹、`u` 表示设备 uuid。而脚本原本固定读取 `mc.i`（旧版的指纹字段），在 1.14.10 上会得到 `undefined`，注入的伪造授权里 `fingerprint` 变成字符串 `"undefined"`，可能被本地校验判定为设备不匹配而吊销授权。

v1.0.4 起解析后统一归一化：

```js
l = raw.l ?? raw.u ?? raw.f ?? ""   // deviceId
i = raw.i ?? raw.f ?? raw.u ?? ""   // fingerprint
v = raw.v ?? ""
```

控制台会打印 `machineCode keys:` 便于核对字段，字段缺失时给出告警。

> 校验日志中的设备标识可用作交叉验证：`profile.data`（hex 编码 JSON）里 `uuid` 与请求体的 `u` 一致。

### 1.7 试用天数计算

日志顺序：

```
[WindowsLicenseLocalStore] IDate : 9/30/2026
[L] installDate is 9/30/2026, trail remains: 15 days
```

说明剩余天数取自注册表 `IDate`，而非 `profile.data` 里的 `_iD`。因此脚本"启动即刷新 + 每 30 分钟刷新 `IDate` 为当天"的做法有效（`profile.data` 未被清理不影响）。

---

## 实测验证清单

激活后按顺序确认：

1. **落盘**：`resources/` 下出现 `app/`、`app.bak/`，原 `app.asar` 变成 `app.asar.bak`；`Typora.exe.bak` 存在。
2. **注册表**：`reg query HKCU\Software\Typora` → `SLicense = RHJlYW1OeWE=#0#1/1/2059`，`IDate` 为当天。
3. **Hook 生效**：`%APPDATA%\Typora\typora.log` 出现
   `[WindowsLicenseLocalStore] SLicense : RHJlYW1OeWE=#0#1/1/2059`
   （而非空值），以及 `[L] installDate is <今天>, trail remains: 15 days`。
4. **静默期**：启动后静置 **12 分钟**（覆盖约 530 秒的 `2nd` 二次校验窗口），日志不应新增 `onUnfillLicense` / `2nd`，不弹「试用 0 天」。
5. **重启**：再次启动 Typora，重复第 3 步。
6. **回滚**：重跑脚本 → 提示已激活 → 输入 `Y` → 确认 `app.asar` 恢复、`app/` 消失、注册表两项被清、Typora 恢复官方原版。

## 已知风险

| 风险 | 说明 | 应对 |
|---|---|---|
| `2nd` 校验时延变化 | 脚本按 520000–545000ms 的经验值拦截启动 120 秒内注册的第一个定时器；若 Typora 改用其它延迟则拦截空转 | 实测清单第 4 步覆盖；如失效再调整区间 |
| 误杀正常定时器 | 同上，命中区间内的正常定时器会被替换为永不触发的假 timer | 需观察启动后 10 分钟内的字典下载、自动保存等行为是否正常 |
| 自动更新覆盖补丁 | Typora 默认开启自动检查更新 | 激活前在偏好设置中关闭 |
| 机器码字段再变 | 新版本可能继续调整机器码 JSON 结构 | 看控制台 `machineCode keys:` 与告警 |
| 指纹取不到 | 机器码不含 `l`/`i`/`f`/`u` 任一字段时 `fingerprint` 为空 | 脚本会告警；重新完整复制机器码 |

## 核查方法说明

本文结论均通过只读手段得出，未修改真实安装：

- **asar 结构 / 文件内容**：直接解析 asar header（`4 字节 pickle 长度` → `4 字节 JSON 长度` → JSON 节点树）并按 `offset` 读取文件字节，无需解包。
- **字节码字符串**：在内存中扫描 `atom.compiled.dist.jsc` 的可打印 ASCII 片段（V8 常量池中的字面量），按偏移排序后即可还原模块的调用链与日志文案。
- **Fuses**：在 exe 中定位哨兵，按 `sentinel + fuse_version + fuse_wire_length + wire` 结构解码；`@electron/fuses` 的 `getCurrentFuseWire()` 可直接读出人类可读结果。
- **注入可行性**：用 `vm.Script` 只做解析校验，不落盘。
- **fuse 翻转**：在临时目录的 exe 副本上执行 `flipFuses` 后逐位比对，副本用完即删。
