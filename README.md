# Text Parsing Engine

软总线（remote bus）文本解析引擎：把 EPUB / MOBI / PDF / TXT 解析成统一的
`fnreader.ebook.v1` JSON，供阅读服务端直接渲染；EPUB / MOBI 可由在线安卓客户端代理解析，
客户端不在线时由服务端本地兜底。

本仓库是 [FnReader](https://github.com/TZYLXS/FnReader)（legado-web 服务端）与
[yuedu-pro](https://github.com/TZYLXS/yuedu-pro)（安卓客户端）中电子书解析部分的独立提取，
已在 FnReader 2.0.7 debug 服务端 + yuedu-pro 2.0.6 上完成 NAS 端到端验证（2026-11-15）。

## 能力

- 目录扫描：定时扫描配置目录，识别 `.epub` `.mobi` `.prc` `.azw` `.azw3` `.pdf` `.txt`。
- 任务派发：EPUB / MOBI 优先派给在线客户端，整本下发，客户端分片回传 JSON。
- 兜底：无在线客户端、客户端迟迟不开工、解析中掉线时，服务端本地解析。
- 渲染：服务端只读 JSON 输出目录与正文，不再碰原文件。
- 客户端解析：EPUB（EPUB2 NCX / EPUB3 nav）、MOBI6（PalmDOC 压缩、EXTH 元数据）。
- 不支持：KF8/AZW3（客户端返回 `kf8-unsupported`）、图片二进制（v1 只保留占位）。

## 仓库结构

```
server/ebook/            服务端引擎（Node.js，仅额外依赖 legado-web 已有的 cheerio）
  store.js               登记表 + 书目录 + 章节/分片落盘
  html.js                HTML → 段落块（k/t）
  epub.js                EPUB 解析
  mobi.js                MOBI6 解析
  pdf.js                 PDF 文本层解析
  text.js                TXT 解析
  local.js               本机解析 + 客户端分片校验入库
  bus.js                 软总线调度、兜底、看门狗
  scan.js                目录扫描（含内部目录排除）
  render.js              ebook://<sha1> 目录/正文渲染
android/reference/
  FnReaderEbook.kt       安卓端 EPUB/MOBI 解析与分片构造（参考实现）
docs/
  AI-INTEGRATION.md      给另一个 AI 的完整接入说明（主文档）
```

## 快速判断要不要用

- 服务端要把本地电子书变成可读的 `ebook://` 书，并且希望把 EPUB/MOBI 解析卸载到手机：用本引擎。
- 只需要纯解析库、不要软总线：可以只取 `epub.js` / `mobi.js` / `pdf.js` / `text.js` 和 `local.js`。
- 安卓端已经有自己的解析器：可以只接服务端 + 协议，客户端按 `AI-INTEGRATION.md` 的 JSON 契约实现。

## 接入

先读 [`docs/AI-INTEGRATION.md`](docs/AI-INTEGRATION.md)。它按“服务端 → 协议 → 安卓端 → 验收”的顺序写了
文件落点、函数签名、接口、状态机、坑位和验收命令。

## 已验证环境

| 项 | 值 |
| --- | --- |
| 服务端 | legado-web 2.0.7（独立 debug 应用，端口 1123，NAS `/vol4`） |
| 客户端 | yuedu-pro 2.0.6 / versionCode 20006，MuMu 模拟器 Android 12L |
| 结果 | EPUB/MOBI `source=device`；暂停设备后 `source=server`；新书扫描自动派发；`/api/toc`、`/api/content` 返回 `via=ebook-json` |

## 来源

从以下仓库的工作区提取，保留原始模块划分与协议版本：

- FnReader：`legado-web/app/server/src/ebook/*`、`src/server.js`、`src/remote.js`、`src/store.js`
- yuedu-pro：`app/src/main/java/io/legado/app/model/FnReaderEbook.kt` 及桥接服务调用点
