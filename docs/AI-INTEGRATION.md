# 给接手 AI 的接入说明：把 Text Parsing Engine 接进 FnReader 与安卓 APK

> 读者：下一个负责把本引擎接入 `FnReader`（legado-web 服务端）和 `yuedu-pro`（安卓客户端）的 AI。
> 目标：服务端能扫描本地电子书，EPUB/MOBI 优先由在线手机解析并回传 JSON，手机都不在线时服务端本地兜底，
> 服务端只用 JSON 渲染目录和正文。
> 本仓库已在一套真实环境验证通过；请按本文顺序落地，不要凭接口名猜实现。

---

## 0. 先明确边界

本引擎只负责“把书变成 JSON”和“把 JSON 渲染成目录/正文”。

- 服务端解析 PDF / TXT 为主；EPUB / MOBI 的服务端解析只作为兜底。
- 安卓端只解析 EPUB / MOBI；PDF / TXT 不会派给手机。
- 原文件只下发一次，客户端解析完回传 JSON 分片，服务端落盘后不再解析原文件。
- 书 URL 固定为 `ebook://<sha1>`，来源（device/server）变化不改变 URL。

不要做这些事：

- 不要让客户端回传原文件或 HTML；只回传 `fnreader.ebook.v1` JSON。
- 不要在服务端渲染时重新解析 EPUB/MOBI；渲染只读 `book.json` 和 `ch/*.json`。
- 不要把 `ebookDir` 指向服务端自己的数据目录 `<dataDir>/ebook`；那里是解析产物目录。
- 不要把 PDF/TXT 派给手机；手机端会返回 `unsupported`。

---

## 1. 仓库文件与落点

| 本仓库 | 接入到 FnReader 的目标路径 | 作用 |
| --- | --- | --- |
| `server/ebook/store.js` | `legado-web/app/server/src/ebook/store.js` | 登记表、书目录、章节/分片读写 |
| `server/ebook/html.js` | 同上目录 | HTML 清洗、块级元素 → `k/t` |
| `server/ebook/epub.js` | 同上目录 | 服务端 EPUB 兜底解析 |
| `server/ebook/mobi.js` | 同上目录 | 服务端 MOBI 兜底解析 |
| `server/ebook/pdf.js` | 同上目录 | PDF 文本层解析 |
| `server/ebook/text.js` | 同上目录 | TXT 解析 |
| `server/ebook/local.js` | 同上目录 | 本地解析 + 客户端分片校验入库 |
| `server/ebook/bus.js` | 同上目录 | 派发、兜底、看门狗 |
| `server/ebook/scan.js` | 同上目录 | 目录扫描 |
| `server/ebook/render.js` | 同上目录 | `ebook://` 目录/正文 |
| `android/reference/FnReaderEbook.kt` | `app/src/main/java/io/legado/app/model/FnReaderEbook.kt` | 安卓 EPUB/MOBI 解析与分片 |

服务端模块依赖 Node 内置模块（`fs`、`path`、`crypto`、`zlib`）外加 `cheerio`：

- `server/ebook/html.js`、`epub.js`、`mobi.js` 都 `require('cheerio')`，用于 HTML/XML 解码与文本抽取；
- `cheerio` 是 legado-web 服务端已有依赖（`app/server` 的 `package.json` 里已装），**不要为此新增 npm 包**；
- 其余模块（`store.js`、`scan.js`、`bus.js`、`render.js`、`local.js`、`pdf.js`、`text.js`）只用内置模块。

安卓端依赖 `org.json`（Android 自带）和 Jsoup（legado 已有依赖）。

---

## 2. 数据流

```
扫描目录
  └─ 命中 epub/mobi/pdf/txt
       ├─ 计算 sha1，复制到 <dataDir>/ebook/files/<sha1>.<ext>
       ├─ 登记 index.json（status=discovered）
       └─ bus.dispatch
            ├─ PDF/TXT ──────────────────────────────→ 本地解析
            ├─ EPUB/MOBI + 有在线设备 → remote.create('ebook-parse')
            │     ├─ GET /api/m/ebook/file/<sha1>（整本，支持 Range）
            │     ├─ 客户端解析 → part0（meta+toc）+ partN（章节）
            │     ├─ POST /api/m/ebook/part?sha1=&seq=&total=
            │     └─ POST /api/m/remote/result（data={chapters,parts,chars}）
            └─ 无设备 / 迟迟不开工 / 掉线 ───────────→ 本地兜底
                 └─ local.parseLocal 直接写 book.json + ch/*.json
```

状态机（`index.json` 中每本书的 `status`）：

```
discovered → queued → partial → complete
     └──────────────→ parsing（本地兜底）
     └──────────────→ failed（drm/corrupt/unsupported/kf8-unsupported）
```

---

## 3. 服务端接入（FnReader / legado-web）

### 3.1 复制模块

把 `server/ebook/*.js` 复制到 `legado-web/app/server/src/ebook/`。目录内文件名保持不变，
模块之间用相对 `require`，不要改名。

### 3.2 配置默认值

在 `src/store.js` 的 `DEFAULT_CONFIG` 里增加：

```js
ebookEnabled: true,
ebookDir: '',
ebookScanSec: 60,
ebookScanDepth: 3,
ebookMaxMB: 512,
ebookFallbackEnabled: true,
ebookFallbackAfterMin: 10,
ebookOfflineGraceSec: 120,
ebookLocalConcurrency: 2,
ebookJobTtlMs: 600000,
```

`ebookDir` 留空时 `scan.js` 会退回 `<downloadDir>/ebook`。部署时把 `ebookDir` 指向独立目录，
例如 `/vol4/@appdata/legado-web-debug/ebook-in`，不要指向 `<dataDir>/ebook`。

### 3.3 扩展 `src/remote.js`

软总线原本只支持短任务（20 秒左右），ebook-parse 需要长任务和撤销能力。改动三点：

1. `create(type, payload, opts)` 支持 `opts = { waitMs, ttlMs }`，返回 `{ id, promise }`；
2. 任务对象记录 `expireAt = now + ttlMs`，`purge()` 用 `expireAt` 判断过期，不再用全局 120 秒；
3. 新增 `abandon(id)`：清掉定时器、删除任务、让等待方收到 `{ok:false,error:'abandoned'}`。

保持原有 `request(type,payload,waitMs)` 行为不变，避免影响 search/toc/content。

### 3.4 接线 `src/server.js`

**(a) 引入模块并构造上下文**（放在 `store.dataDir` 可用之后）：

```js
const ebookStore = require('./ebook/store');
const ebookScan = require('./ebook/scan');
const ebookBus = require('./ebook/bus');
const ebookRender = require('./ebook/render');
const ebookLocal = require('./ebook/local');

const ebookCtx = {
  dataDir: store.dataDir,
  config: () => store.config,
  remote,                 // 即 src/remote.js
  token: () => apiToken(),// 与 /api/m/* 的 X-Api-Token 一致
  log: (...args) => console.log(...args),
};
```

**(b) 新增路由**（`/api/m/*` 路由段内）：

| 方法 | 路径 | 要求 |
| --- | --- | --- |
| GET | `/api/m/ebook/file/<sha1>` | 40 位 sha1；按条目真实路径读文件；支持 `Range`，返回 206/416；`Content-Type` 按格式；文件不存在 404 |
| POST | `/api/m/ebook/part?sha1=&seq=&total=` | 原始 body ≤8 MB；`Content-Encoding: gzip` 时解压，解压后 ≤20 MB；JSON 交给 `ebookLocal.acceptPart`；错误码映射 413/409/400 |
| GET | `/api/m/ebook/parts?sha1=` | 返回 `{sha1,total,received,complete,status}`，给断点续传 |

**(c) 管理/调试路由**（按现有鉴权习惯决定是否加令牌）：

```js
if (p === '/api/ebook/list')   return json(res, 200, listResponse);
if (p === '/api/ebook/scan')   return json(res, 200, await ebookScan.scanOnce(ebookCtx));
if (p === '/api/ebook/reparse') {
  // 重置 ch/ 与 parts/，状态置 discovered，再 ebookBus.dispatch(ctx, entry)
}
```

**(d) 渲染分支**：在 `/api/toc` 和 `/api/content` 的最前面加：

```js
const ebookSha = ebookRender.parseUrl(searchParams.get('url'));
if (ebookSha) return json(res, 200, ebookRender.toc(store.dataDir, ebookSha));
// content 同理，但 index 越界要返回 404/空，不要抛异常
```

**(e) 派发白名单**：`/api/remote/job` 的 `type` 白名单加 `ebook-parse`，
等待上限单独放宽（实现里是 240 秒），其他类型保持原值。

**(f) 启动**：在 `server.listen` 之后启动扫描与看门狗：

```js
ebookScan.start(ebookCtx, (Number(store.config.ebookScanSec) || 60) * 1000);
ebookBus.startWatchdog(ebookCtx);
```

### 3.5 调度规则（不要改错）

`bus.dispatch` 的实际顺序：

1. `ebookEnabled === false` 直接返回；
2. 同 sha1 已有在途任务直接返回；
3. `status` 为 `complete` / `parsing` 直接返回；
4. 非 EPUB/MOBI → `enqueueLocal(reason='server-format')`；
5. 无在线设备且允许兜底 → `enqueueLocal(reason='no-device')`；
6. 有设备 → `remote.create('ebook-parse', payload, {ttlMs:600000,waitMs:600000})`，
   状态置 `queued`，同时起 `ebookFallbackAfterMin` 计时；
7. 计时到点仍无任何分片 → `remote.abandon(jobId)` → 本地兜底；
8. 结果回来：
   - 已 `complete` → 丢弃；
   - `received.length >= data.parts` → `complete/source=device`；
   - 否则 `partial/source=device`；
9. 客户端错误码：
   - `drm` / `corrupt` / `unsupported` / `kf8-unsupported` → `failed`，**不兜底**；
   - 其他（timeout/upload/offline 等）→ 本地兜底；
10. 看门狗：`source=device` 且 `status` 为 `partial/parsing`，超过 `ebookOfflineGraceSec`
    没有新分片 → 撤销任务并本地接手。

### 3.6 存储布局

```
<dataDir>/ebook/
  index.json                    登记表
  files/<sha1>.<ext>            原文件副本（下发用）
  books/<sha1>/book.json        schema/meta/toc/stats/source/status
  books/<sha1>/ch/0000.json     章节正文
  books/<sha1>/parts/0000.done  已收到分片标记
```

### 3.7 扫描的两个坑（务必保留）

1. **20 秒保护**：文件 mtime 距现在 <20 秒时跳过，避免把正在拷贝的文件当完整书。
2. **排除自建目录**：`scan.js` 的 `collect()` 接收 `skipPaths`，扫描时跳过
   `<dataDir>/ebook/files` 和 `<dataDir>/ebook/books`。否则当 `ebookDir` 指到内部目录时，
   会把解析产物复制件当新书，出现重复计数。

### 3.8 构建

服务端 bundle 用 esbuild：

```bash
cd legado-web/app/server
node node_modules/esbuild/bin/esbuild src/server.js \
  --bundle --platform=node --target=node18 --format=cjs --minify \
  --outfile=<输出目录>/server/server.bundle.js
```

`--outfile` 必须用等号，路径用正斜杠，否则 esbuild 会把参数当非法 flag。

---

## 4. 协议契约 `fnreader.ebook.v1`

### 4.1 任务 payload（服务端 → 手机）

```json
{
  "sha1": "8f0c...（40 位）",
  "name": "书名",
  "format": "epub",
  "size": 1048576,
  "fileUrl": "/api/m/ebook/file/8f0c...",
  "token": "<本次下载令牌>",
  "schema": "fnreader.ebook.v1",
  "partMaxBytes": 4194304
}
```

客户端下载时带 `X-Api-Token: <token>`；`fileUrl` 可能是相对路径，要拼服务器 base。

### 4.2 part 0：清单

```json
{
  "schema": "fnreader.ebook.v1",
  "sha1": "8f0c...",
  "format": "epub",
  "name": "书名",
  "meta": {"title":"书名","author":"","language":"zh","publisher":"","publishedAt":"","isbn":"","description":""},
  "toc": [{"id":"ch0001","title":"第一章","level":1,"index":0}],
  "stats": {"chapters":1,"chars":2400,"parts":1}
}
```

### 4.3 part N：正文

```json
{
  "schema": "fnreader.ebook.v1",
  "sha1": "8f0c...",
  "seq": 1,
  "total": 2,
  "chapters": [
    {"index":0,"id":"ch0001","title":"第一章",
     "paragraphs":[{"k":"p","t":"正文"},{"k":"quote","t":"引文"},{"k":"hr","t":""}]}
  ]
}
```

约定：

- `schema` 必须完全相等，否则整本拒绝；
- `sha1` 必须与任务一致，否则 409；
- `index` 全书连续、从 0 开始，与 `toc.index` 对齐；
- `k` 只允许 `p` / `h1`~`h6` / `quote` / `hr` / `img`；
- `t` 是去标签纯文本，空段丢弃，`hr` 可以 `t=""`；
- 每片建议 ≤4 MB（实现按 50 章或约 2 MB 切一组），按章节边界切，不拆章；
- `total` 全书固定，part 0 也算一片，所以 `total = 1 + 正文片数`。

### 4.4 接口

| 接口 | 说明 |
| --- | --- |
| `GET /api/m/ebook/file/<sha1>` | 整本下载，支持 `Range: bytes=start-`，206 续传，416 越界 |
| `POST /api/m/ebook/part?sha1=&seq=&total=` | 上传分片，明文或 gzip；返回 `{ok,received,complete}` |
| `GET /api/m/ebook/parts?sha1=` | 断点查询，重连时先查，已收分片跳过 |
| `GET /api/m/remote/pending` | 领任务，`type=ebook-parse` |
| `POST /api/m/remote/result` | 回结果，`data={chapters,parts,chars}` |

---

## 5. 安卓端接入（yuedu-pro / APK）

### 5.1 解析器

把 `android/reference/FnReaderEbook.kt` 复制到
`app/src/main/java/io/legado/app/model/FnReaderEbook.kt`。它提供：

```kotlin
object FnReaderEbook {
    const val SCHEMA = "fnreader.ebook.v1"
    fun parse(file: File, format: String, fallbackName: String): Parsed
    fun buildParts(parsed: Parsed, sha1: String, format: String, name: String): List<String>
}
```

`parse` 只处理 `epub` 和 `mobi/azw`，其他格式抛 `EbookException("unsupported")`。
`buildParts` 返回 part0 + 正文分片，`total` 已算好。

解析要点（改代码时不要破坏）：

- EPUB：`META-INF/container.xml` → OPF → manifest/spine；目录优先 `nav.xhtml`，回退 NCX；
- MOBI6：PalmDB 记录表、PalmDOC 压缩 1/2、EXTH 取 author/title/language；
  书名偏移是 record0 的 `16 + 0x54`，不是固定文件头；
- HTML → 块：镜像服务端 `html.js`，`h1~h6` 保留层级，`blockquote` → `quote`，
  `br` 不单独成段，超长段按句号切；
- KF8/AZW3：识别到 KF8 头就抛 `EbookException("kf8-unsupported")`，不要硬解。

### 5.2 下载与上传

在 `model/FnReaderRemote.kt` 增加两个方法：

```kotlin
suspend fun downloadToFile(
    base: String, token: String, fileUrl: String, dest: File, expectedSize: Long = 0
): File

suspend fun uploadEbookPart(
    base: String, token: String, sha1: String, seq: Int, total: Int, json: String
): JSONObject
```

实现要求：

- `downloadToFile`：相对 `fileUrl` 拼 `base`；带 `X-Api-Token`；若本地已有部分文件，
  带 `Range: bytes=<已有长度>-`，只有服务端返回 206 才追加写；下载完校验 `expectedSize`；
- `uploadEbookPart`：POST 到 `/api/m/ebook/part?sha1=...&seq=...&total=...`，
  `Content-Type: application/json; charset=utf-8`，带 `X-Api-Token`，body 为明文 JSON；
  非 2xx 读 errorStream 抛异常。

### 5.3 桥接服务

在 `service/FnReaderBridgeService.kt`：

1. 建一个独立单并发队列 `ebookQueue`，不要和 search/toc 共用并发池：

```kotlin
private val ebookQueue = Channel<QueuedJob>(Channel.UNLIMITED)
// 独立协程：for (item in ebookQueue) runEbookJob(item)
```

2. 轮询到任务后按类型路由：

```kotlin
"ebook-parse" -> ebookQueue.trySend(item)
```

3. `runEbookJob` 流程：

```
payload 取 sha1/format/name/size/fileUrl/token
下载到 cacheDir/ebook_bridge/<sha1>.<ext>
FnReaderEbook.parse(file, format, name)
FnReaderEbook.buildParts(parsed, sha1, format, name)
逐片 uploadEbookPart
submit(id, ok=true, data={chapters,parts,chars}, deviceId)
失败：submit(id, ok=false, error=<code>)
finally 删除临时文件
```

4. 超时：整任务 8 分钟；超时按普通错误回传，让服务端兜底。
5. 错误码映射：
   - `EbookException.code` 原样回传（`drm/corrupt/unsupported/kf8-unsupported`）；
   - 网络/IO 错误用 `FnReaderRemote.describeError(e)`，不要伪造成 `corrupt`。
6. 每次上传分片后更新前台通知进度，避免系统杀进程。

### 5.4 客户端配置与服务启动

沿用现有软总线配置键：

| 键 | 值 |
| --- | --- |
| `fnReaderServer` | `http://<服务端>:1123` |
| `fnReaderToken` | 服务端 `apiToken` |
| `fnReaderBridgeOn` | `"1"` |
| `fnReaderOnlineVersion` | 客户端版本（如 `2.0.6`） |

服务只能由设置弹窗“手机解析待命/接入 FnReader 服务器”或开机广播启动。
`am broadcast BOOT_COMPLETED` 从 adb shell 发会被 SecurityException 拒绝，
测试时用 UI 拉起，不要在这上面浪费时间。

---

## 6. 服务端与客户端版本配套

| 服务端 | 客户端 | 说明 |
| --- | --- | --- |
| 2.0.7+ | 2.0.6+ | 本引擎验证组合 |
| 旧版 | 2.0.6+ | 服务端没有 `ebook-parse` 白名单时，客户端会领不到任务，必须同步升级 |

两仓库共用版本号，迭代 +0.1；发版要同步客户端 `updateLog.md` 和 `docs/更新日志.md`。
测试用可 adb 控制的 debug 包；发版包必须去掉 debug 能力。

---

## 7. 验收清单

按顺序做，全部通过才算接完。

### 7.1 服务端静态检查

```bash
node -e "require('./legado-web/app/server/src/ebook/local')"
node -e "require('./legado-web/app/server/src/ebook/bus')"
node -e "require('./legado-web/app/server/src/ebook/scan')"
```

### 7.2 服务端本地兜底（无设备）

1. 配置 `ebookDir` 指向样本目录，放入 epub/mobi/pdf/txt 各一本；
2. `POST /api/ebook/scan`，应返回 `found=N added=N errors=0`；
3. `GET /api/ebook/list`，全部 `complete/source=server`；
4. `GET /api/toc?url=ebook://<sha1>` 返回章节；
5. `GET /api/content?url=ebook://<sha1>&index=0` 返回 `via=ebook-json` 和段落。

### 7.3 客户端代理（有设备）

1. 安卓端配置指向服务端并启动待命；
2. `GET /api/bus` 应看到设备 `online=true`；
3. `POST /api/ebook/reparse {sha1}`，等待完成；
4. 列表该条应为 `complete/source=device`，`chapters` 与书一致；
5. `GET /api/bus` 的 `handled` 递增，`pending=0`。

### 7.4 兜底

1. `POST /api/bus/toggle {id, enabled:false}` 暂停唯一设备；
2. `POST /api/ebook/reparse {sha1}`；
3. 列表应变为 `complete/source=server`；
4. 恢复设备后再 reparse，应回到 `source=device`。

### 7.5 新书自动派发

1. 往 `ebookDir` 放一本新 EPUB；
2. 等 20 秒（mtime 保护）；
3. `POST /api/ebook/scan`，应 `added=1`；
4. 稍后列表该书应为 `complete/source=device`（设备在线时）。

### 7.6 渲染

```bash
curl -H "X-Api-Token: <token>" "http://<host>:<port>/api/toc?url=ebook://<sha1>"
curl -H "X-Api-Token: <token>" "http://<host>:<port>/api/content?url=ebook://<sha1>&index=0"
```

目录要含 `chapters/total/source/via=ebook-json`；正文要含 `paragraphs/prev/next/via=ebook-json`。

---

## 8. FPK / APK 打包注意

- FPK 应用名与生产分开，例如 `legado-web-debug`，端口 `1123`，数据目录独立；
- `fnpack build -d <dir>` 在 NAS 上执行；manifest 的 `appname` 必须与目录/启动名一致；
- 应用中心同版本 `install-fpk` 不会替换 `server.bundle.js`；要更新运行实例需
  `stop` → 覆盖 bundle → `start`，或提升版本号；
- APK 用 `assembleAppDebug`，variant 名是 `app`；构建前准备 `local.properties`：
  `sdk.dir=D:\\Android\\Sdk`（UTF-8 无 BOM）。

---

## 9. 已知限制

1. PDF 只支持带文本层的 PDF；扫描版 PDF 会失败。
2. KF8/AZW3 不支持，客户端返回 `kf8-unsupported`，服务端按约定不兜底。
3. 正文图片不传二进制，`img` 只保留 alt，渲染成占位。
4. `/api/ebook/list|scan|reparse` 与 `POST /api/config` 默认没有令牌校验，只适合内网/调试；
   对外部署前必须加鉴权。
5. 大书（>3000 章）会被拒绝；分片单本 `total > 10000` 会被拒绝。

---

## 10. 你接手后的建议顺序

1. 先只接服务端本地解析（PDF/TXT + EPUB/MOBI 兜底），跑通 `/api/toc`、`/api/content`；
2. 再接 `ebook-parse` 任务与分片接口，用 curl/Postman 模拟 part0、part1；
3. 最后接安卓端，先 EPUB 再 MOBI，先小书再大书；
4. 每步都跑第 7 节对应验收，不要最后一起联调。
