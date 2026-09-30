<div align="center">

# Icost 自动记账

**一张截图进账本。154 个动作，从 OCR 到入账全自动。**

截一张账单 → DeepSeek 解析成结构化账单 → 逐项确认 → 写入 iCost。
不是「打开 App 手动记」，而是「看到账单的那一刻就记完了」。

[下载捷径](./Icost.shortcut) &nbsp;·&nbsp; [快速开始](#快速开始) &nbsp;·&nbsp; [工程笔记](#工程笔记那些踩过的坑) &nbsp;·&nbsp; [Issues](https://github.com/abobb414/icost/issues)

[![Actions](https://img.shields.io/badge/actions-154-8b5cf6?style=flat-square)](#预览)
[![App Intents](https://img.shields.io/badge/iCost%20Intents-16-0ea5e9?style=flat-square)](#账单契约)
[![Branches](https://img.shields.io/badge/conditional-45%20branches-f59e0b?style=flat-square)](#账单契约)
[![API Key](https://img.shields.io/badge/API%20Key-占位符%2C%20需自填-22c55e?style=flat-square)](#快速开始)
[![Watch](https://img.shields.io/badge/Watch-兼容-000000?style=flat-square)](#限制)

</div>

<details>
<summary><b>English</b>（点击展开英文版 · Click to expand）</summary>

<div align="center">

# Icost Auto Bookkeeping

**One screenshot into the ledger. 154 actions, fully automated from OCR to entry.**

Snap a bill → DeepSeek parses it into structured data → confirm item by item → write into iCost.
It's not "open the app and log it by hand" — it's "the moment you see the bill, it's already booked".

[Download Shortcut](./Icost.shortcut) &nbsp;·&nbsp; [Quick Start](#quick-start) &nbsp;·&nbsp; [Engineering Notes](#engineering-notes-pitfalls-we-hit) &nbsp;·&nbsp; [Issues](https://github.com/abobb414/icost/issues)

[![Actions](https://img.shields.io/badge/actions-154-8b5cf6?style=flat-square)](#preview)
[![App Intents](https://img.shields.io/badge/iCost%20Intents-16-0ea5e9?style=flat-square)](#bill-contract)
[![Branches](https://img.shields.io/badge/conditional-45%20branches-f59e0b?style=flat-square)](#bill-contract)
[![API Key](https://img.shields.io/badge/API%20Key-placeholder%2C%20fill%20your%20own-22c55e?style=flat-square)](#quick-start)
[![Watch](https://img.shields.io/badge/Watch-compatible-000000?style=flat-square)](#limitations)

</div>

---

## Preview

There's no UI to screenshot, so here's the **anatomy** instead — per-action stats gathered by decrypting `Icost.shortcut` with a script:

| Composition | Count | Notes |
|---|---|---|
| Total actions | **154** | The full pipeline from screenshot to ledger entry |
| Conditional branches | **45** | Type detection, null fallbacks, account/category matching |
| Variable assignments | 29 | Intermediate state for amount, date, category, account, note |
| iCost App Intents | **16** | 13 queries + 3 writes |
| Manual selections | 16 | 12 menus + 4 lists, the fallback when parsing fails |
| OCR | 2 | Take screenshot + extract text from image |
| Network requests | 1 | DeepSeek chat/completions |

> The numbers are real: computed on 2026-09-30 by decrypting this repo's `Icost.shortcut` (an AEA-encrypted archive) and counting actions with a script,
> byte-for-byte identical to the version in daily use on my Mac (only the API Key is a placeholder).

---

## Table of Contents

- [How It Works](#how-it-works)
- [Features](#features)
- [Bill Contract](#bill-contract)
- [Quick Start](#quick-start)
- [Engineering Notes: Pitfalls We Hit](#engineering-notes-pitfalls-we-hit)
- [Limitations](#limitations)

---

## How It Works

```mermaid
flowchart TD
    A["Screenshot / pick from Photos<br/>takescreenshot"] --> B["Extract text from image<br/>OCR"]
    B --> C["DeepSeek parsing<br/>chat/completions"]
    C --> D{"JSON returned?"}
    D -->|"yes"| E["Item-by-item confirm menu<br/>10 items"]
    D -->|"no"| F["Exit and retry"]
    E --> G{"Type detection"}
    G -->|"expense"| H["Write into iCost<br/>Outcome"]
    G -->|"income"| I["Write into iCost<br/>Income"]
    G -->|"transfer"| J["Write into iCost<br/>Transfer"]
    E -->|"an item is wrong"| K["Manual fix<br/>amount/date/category/account"]

    style A fill:#0ea5e9,color:#fff
    style C fill:#8b5cf6,color:#fff
    style D fill:#f59e0b,color:#fff
    style E fill:#f59e0b,color:#fff
    style H fill:#22c55e,color:#fff
    style I fill:#22c55e,color:#fff
    style J fill:#22c55e,color:#fff
    style F fill:#64748b,color:#fff
    style K fill:#64748b,color:#fff
```

The main path is only 6 steps; the other 148 actions are all spent on **confirmation and fallbacks** — this is intentional: OCR and LLMs both make mistakes,
and a ledger that's off by a single cent is a real problem, so every item has a manual entry point.

### The account & category matching chain

DeepSeek only outputs **text** (e.g. "China Merchants Bank"), while iCost needs **entities**. App Intent queries bridge the gap:

```mermaid
flowchart LR
    A["DeepSeek output<br/>plain text name"] --> B{"ICSearchAssetEntity<br/>exact match?"}
    B -->|"hit"| C["Book it directly"]
    B -->|"no hit"| D["choosefromlist<br/>pick by hand from real accounts"]
    D --> E["Write into iCost"]

    style A fill:#0ea5e9,color:#fff
    style B fill:#f59e0b,color:#fff
    style C fill:#22c55e,color:#fff
    style D fill:#64748b,color:#fff
    style E fill:#22c55e,color:#fff
```

> Expenses/income query categories, transfers query both accounts — 13 query Intents in total. **Nothing is created**: no new accounts, no new categories.
> If a match can't be found, you pick by hand, and the entry goes through as usual. Rather tap a few extra times than stuff dirty data into the ledger.

---

## Features

| | Feature | In one line |
|---|---|---|
| 🔖 | **All three types covered** | Expense / income / transfer each go through their own App Intent, with fields tailored to each |
| 🧾 | **10-item confirm menu** | Review every item before booking; any single item can be fixed on its own |
| 🏦 | **Real entity matching** | 13 query Intents bind to iCost's existing accounts and categories — nothing created, nothing polluted |
| 🔁 | **Duplicate-entry protection** | Regex-cleans the reply + duplicate detection; re-running never books twice |
| ⌚ | **Watch compatible** | Declared as a Watch shortcut, runs straight from the watch face |
| 🔒 | **Key placeholder** | The repo contains no API Key at all; fill in your own after importing |

### The confirm menu: not "confirm / cancel" — every item is editable

Of the 10 menu items, 8 are **individually fixable fields** (type / amount / date / category / source account /
destination account / currency / note); only the last two are "all correct" and "bill is wrong, redo the booking".

> A design trade-off: cutting it down to just two buttons — "confirm / redo" — would shave 40 actions off the shortcut.
> But an OCR error like reading "招商银行" (China Merchants Bank) as "招商银 行" will be wrong again on every redo. Per-item editing means one wrong item costs one fix.

### Deduplication: clean first, then compare

DeepSeek replies often come wrapped in markdown fences or with stray whitespace, which trips up a naive comparison. The shortcut first cleans
the reply text with regex substitutions (`text.replace` + the regex switch), then runs the duplicate check, and only then decides whether to book.

---

## Bill Contract

What DeepSeek receives is **a single user message with no system prompt** — its content is the raw OCR'd bill text.
The model infers the JSON below from the bill text on its own (thanks to `deepseek-v4-flash`'s familiarity with payment bill formats):

| Field | Type | Purpose |
|---|---|---|
| `账单金额` (bill amount) | number | Amount to book |
| `账单时间` (bill time) | text | Normalized afterwards by "Get Dates from Input" |
| `账单分类` (bill category) | text | Matched against iCost categories |
| `收支类型` (income/expense type) | expense / income / transfer | Determines which booking chain runs |
| `转出账户` (source account) | text | Matched against iCost accounts |
| `转入账户` (destination account) | text | Transfers only |
| `备注` (note) | text | Entry note |

> This is a bold omission: no system prompt means the parsing rules live nowhere in the pipeline, and a model upgrade could change
> the output format. What it buys is a request body capped at 300 `max_tokens` and an extremely low cost. In testing, the current model
> reliably outputs the fields above — if you switch models and the field names change, check here first.

**Request parameters** (measured from inside the shortcut):

```text
POST  https://e-flowcode.cc/v1/chat/completions
Authorization: Bearer YOUR_DEEPSEEK_API_KEY   ← placeholder, replace after import
Content-Type: application/json

{
  "model": "deepseek-v4-flash",
  "max_tokens": 300,
  "messages": [{ "role": "user", "content": "<bill text merged after OCR>" }]
}
```

> The endpoint is a **third-party relay**, not DeepSeek's official `api.deepseek.com`. To use the official endpoint, just swap the URL
> in the "Get Contents of URL" action for `https://api.deepseek.com/chat/completions` —
> everything else stays the same.

---

## Quick Start

```bash
# 1. Download the shortcut file
curl -LO https://github.com/abobb414/icost/releases/latest/download/Icost.shortcut
#    or download Icost.shortcut directly from this repo

# 2. Double-click to import into the Shortcuts app (macOS 12+ / iOS 15+)

# 3. Open the shortcut, find the "Get Contents of URL" action,
#    and replace YOUR_DEEPSEEK_API_KEY in the Authorization header with your key
#    → apply at https://platform.deepseek.com/

# 4. Prerequisite: iCost must be installed, with at least 1 account and a few categories set up
#    (a manual picker pops up when matching fails, but an empty ledger leaves you nothing to pick)
```

How to run: take a screenshot of a bill, run it in the Shortcuts app, or **share** the screenshot to the shortcut.

---

## Engineering Notes: Pitfalls We Hit

### `.shortcut` is not a black box

The first 12 bytes of the file are the `AEA1` magic + version + header length, followed by a plist holding `SigningCertificateChain`;
the body is an Apple Encrypted Archive. The built-in `aea` and `aa` commands alone recover the plaintext
`Shortcut.wflow` (a binary plist):

```bash
# 1. Extract the public key from DER cert #0 in the header's certificate chain
openssl x509 -inform DER -in c0.der -pubkey -noout > pub.pem
# 2. Decrypt
aea decrypt -i Icost.shortcut -o out.bin -profile 0 -sign-pub pub.pem
# 3. Unpack
aa extract -i out.bin -d outdir   # → Shortcut.wflow
```

That's exactly how the action stats at the top of this README were counted. **This step is mandatory before pushing to a public repo** —
`git grep` finds nothing inside an encrypted archive; the previous version nearly got pushed with a real API Key inside.

### Any change must be re-signed

Editing the wflow and packing it back doesn't work — the signature won't match and Shortcuts refuses the file. Re-sign with the built-in command:

```bash
shortcuts sign -m anyone -i clean.wflow -o out.shortcut
```

`shortcuts sign` generates a fresh signing certificate and normalizes `WFWorkflowClientVersion` from `4711`
to `5037.0.17` — verification compares action-by-action rather than by file hash, and these two differences are expected.

### No programmatic export on macOS 27

The `shortcuts` CLI is down to four subcommands — `run / list / view / sign`; `~/Library/Shortcuts/`
is locked down by TCC, and AppleScript offers no export interface either. To get a file for a local shortcut, your only option is
right-click → "Export…" in the Shortcuts app. For backups, the system leaves no automation door open.

---

## Limitations

- **Depends on iCost's App Intents**: if an iCost update changes the Intent signatures, the shortcut breaks; this version matches iCost's current release
- **No system prompt**: a model upgrade could change the output field names, see [Bill Contract](#bill-contract)
- **OCR is the weak link**: blurry screenshots, dark mode, and watermarks can all cause misreads — the confirm menu exists precisely for this
- **One bill at a time**: a single screenshot per run; combined bills or long bill lists are not handled
- Watch compatibility is as-is on real devices; no watch-face-specific tuning was done

---

> Ledger entries are never trivial. Before anything is booked, treat the confirm menu as the source of truth; the program's output constitutes no bookkeeping guarantee whatsoever.
</details>

---

## 预览

没有界面可截图，直接上**解剖数据** —— 由脚本对 `Icost.shortcut` 解密后逐动作统计：

| 构成 | 数量 | 说明 |
|---|---|---|
| 总动作数 | **154** | 从截屏到入账的完整链路 |
| 条件分支 | **45** | 类型判定、空值兜底、账户/分类匹配 |
| 变量赋值 | 29 | 金额、日期、分类、账户、备注的中间态 |
| iCost App Intent | **16** | 13 个查询 + 3 个入账 |
| 人工选择 | 16 | 12 个菜单 + 4 个列表，解析失败时的兜底 |
| OCR | 2 | 截屏 + 提取图像文本 |
| 网络请求 | 1 | DeepSeek chat/completions |

> 数据是真的：2026-09-30 对本仓库 `Icost.shortcut`（AEA 加密归档）解密后用脚本统计得出，
> 与本人 Mac 上正在使用的版本逐字节一致（仅 API Key 为占位符）。

---

## 目录

- [工作原理](#工作原理)
- [特性](#特性)
- [账单契约](#账单契约)
- [快速开始](#快速开始)
- [工程笔记：那些踩过的坑](#工程笔记那些踩过的坑)
- [限制](#限制)

---

## 工作原理

```mermaid
flowchart TD
    A["截屏 / 相册选图<br/>takescreenshot"] --> B["提取图像文本<br/>OCR"]
    B --> C["DeepSeek 解析<br/>chat/completions"]
    C --> D{"返回 JSON?"}
    D -->|"是"| E["逐项确认菜单<br/>10 项"]
    D -->|"否"| F["退出重试"]
    E --> G{"类型判定"}
    G -->|"支出"| H["写入 iCost<br/>Outcome"]
    G -->|"收入"| I["写入 iCost<br/>Income"]
    G -->|"转账"| J["写入 iCost<br/>Transfer"]
    E -->|"某项有误"| K["手动修正<br/>金额/日期/分类/账户"]

    style A fill:#0ea5e9,color:#fff
    style C fill:#8b5cf6,color:#fff
    style D fill:#f59e0b,color:#fff
    style E fill:#f59e0b,color:#fff
    style H fill:#22c55e,color:#fff
    style I fill:#22c55e,color:#fff
    style J fill:#22c55e,color:#fff
    style F fill:#64748b,color:#fff
    style K fill:#64748b,color:#fff
```

主链路只有 6 步，剩下 148 个动作全在**确认与兜底**上 —— 这是有意的：OCR 和大模型都会出错，
账目错一分钱都麻烦，所以每一项都留了人工入口。

### 账户与分类的匹配链

DeepSeek 只输出**文本**（如「招商银行」），iCost 需要的是**实体**。中间靠 App Intent 查询补上：

```mermaid
flowchart LR
    A["DeepSeek 输出<br/>纯文本名称"] --> B{"ICSearchAssetEntity<br/>精确匹配?"}
    B -->|"命中"| C["直接入账"]
    B -->|"未命中"| D["choosefromlist<br/>从真实账户列表手选"]
    D --> E["写入 iCost"]

    style A fill:#0ea5e9,color:#fff
    style B fill:#f59e0b,color:#fff
    style C fill:#22c55e,color:#fff
    style D fill:#64748b,color:#fff
    style E fill:#22c55e,color:#fff
```

> 支出/收入查分类、转账查双账户，共 13 个查询 Intent。**不新建任何账户或分类** ——
> 匹配不到就让你手选，选完照常入账。宁可多按一次，不往账本里塞脏数据。

---

## 特性

| | 特性 | 一句话 |
|---|---|---|
| 🔖 | **三类型全覆盖** | 支出 / 收入 / 转账走各自独立的 App Intent，字段各取所需 |
| 🧾 | **10 项确认菜单** | 入账前逐项过目，任何一项可单点修正 |
| 🏦 | **真实实体匹配** | 13 个查询 Intent 对接 iCost 已有账户与分类，不新建不污染 |
| 🔁 | **防重复入账** | 正则清洗回复 + 重复检测，重跑不会记两笔 |
| ⌚ | **Watch 兼容** | 声明为 Watch 类型，表盘上直接跑 |
| 🔒 | **Key 占位符** | 仓库里不含任何 API Key，导入后自己填 |

### 确认菜单：不是「确认 / 取消」，是逐项可改

10 个菜单项里 8 项是**可单独修正的字段**（类型 / 金额 / 日期 / 分类 / 转出账户 / 转入账户 /
货币 / 备注），只有最后两项是「确认无误」和「账单有误，重新记账」。

> 设计取舍：改成纯「确认 / 重来」两个按钮，捷径能砍掉 40 个动作 —— 但 OCR 把「招商银行」
> 认成「招商银 行」这种错，重来一遍还是错。逐项可改意味着错一项只改一项。

### 防重复：先清洗，再判断

DeepSeek 回复里常带 markdown 围栏或多余空白，直接判断会误判。捷径先做正则替换
（`text.replace` + 正则开关）清洗回复文本，再做重复比对，最后才决定是否入账。

---

## 账单契约

DeepSeek 收到的是**一条不带 system 提示词的 user 消息** —— 内容就是 OCR 出来的账单原文。
模型自己从账单文本里推断出下面的 JSON（靠 `deepseek-v4-flash` 对支付账单格式的熟悉程度）：

| 字段 | 说明 | 用途 |
|---|---|---|
| `账单金额` | 数字 | 入账金额 |
| `账单时间` | 文本 | 再经「获取日期」规范化 |
| `账单分类` | 文本 | 匹配 iCost 分类 |
| `收支类型` | 支出 / 收入 / 转账 | 决定走哪条入账链 |
| `转出账户` | 文本 | 匹配 iCost 账户 |
| `转入账户` | 文本 | 仅转账 |
| `备注` | 文本 | 入账备注 |

> 这是个**大胆的省略**：没有 system 提示词，意味着解析规则完全不在链路里、模型升级可能改变
> 输出格式。换来的是请求体只有 300 token 的 `max_tokens` 和极低的成本。实测当前模型稳定输出
> 上述字段 —— 如果你换模型后发现字段名变了，先检查这里。

**请求参数**（实测自快捷指令内部）：

```text
POST  https://e-flowcode.cc/v1/chat/completions
Authorization: Bearer YOUR_DEEPSEEK_API_KEY   ← 占位符，导入后替换
Content-Type: application/json

{
  "model": "deepseek-v4-flash",
  "max_tokens": 300,
  "messages": [{ "role": "user", "content": "<OCR 合并后的账单文本>" }]
}
```

> 端点是**第三方中转**，不是 DeepSeek 官方 `api.deepseek.com`。要走官方端点，把
> 「获取 URL 内容」里的 URL 换成 `https://api.deepseek.com/chat/completions` 即可，
> 其余参数不变。

---

## 快速开始

```bash
# 1. 下载捷径文件
curl -LO https://github.com/abobb414/icost/releases/latest/download/Icost.shortcut
#    或直接下载仓库里的 Icost.shortcut

# 2. 双击导入「快捷指令」App（macOS 12+ / iOS 15+）

# 3. 打开该指令，找到「获取 URL 内容」动作，
#    把 Authorization 请求头里的 YOUR_DEEPSEEK_API_KEY 换成你的 Key
#    → https://platform.deepseek.com/ 申请

# 4. 前提：本机已安装 iCost，并至少建好 1 个账户和几个分类
#    （匹配不到时会弹出手动选择，但空账本连选都没得选）
```

运行方式：截一张账单截图，在快捷指令 App 里跑，或把截图**分享**给该捷径。

---

## 工程笔记：那些踩过的坑

### `.shortcut` 不是黑盒

文件头 12 字节是 `AEA1` magic + 版本 + 头部长度，跟着一段 plist 装着 `SigningCertificateChain`，
主体是 Apple Encrypted Archive。用系统自带的 `aea` + `aa` 两条命令就能解出明文的
`Shortcut.wflow`（二进制 plist）：

```bash
# 1. 从头部证书链第 0 个 DER 提取公钥
openssl x509 -inform DER -in c0.der -pubkey -noout > pub.pem
# 2. 解密
aea decrypt -i Icost.shortcut -o out.bin -profile 0 -sign-pub pub.pem
# 3. 解包
aa extract -i out.bin -d outdir   # → Shortcut.wflow
```

本 README 顶部的动作统计就是这么数出来的。**推公开仓库前必须走这一步** ——
`git grep` 对加密归档什么都搜不出来，上一版差点带着真实 API Key 就推上去了。

### 改完必须重签

直接改 wflow 再打包回去没用 —— 签名对不上，Shortcuts 拒收。用系统自带命令重签：

```bash
shortcuts sign -m anyone -i clean.wflow -o out.shortcut
```

`shortcuts sign` 会生成新的签名证书，并把 `WFWorkflowClientVersion` 从 `4711`
规范化成 `5037.0.17` —— 验证时逐 action 对比而不是比文件 hash，这两处差异是预期的。

### macOS 27 上没有程序化导出

`shortcuts` CLI 只剩 `run / list / view / sign` 四个子命令，`~/Library/Shortcuts/`
被 TCC 锁死，AppleScript 也没有导出接口。想拿本地捷径的文件，只能在 Shortcuts App 里
右键 →「导出…」手动来。备份这件事，系统没有留自动化的门。

---

## 限制

- **依赖 iCost 的 App Intent**：iCost 更新改了 Intent 签名，捷径会断；本版对应 iCost 当前版本
- **无 system 提示词**：模型升级可能改变输出字段名，见[账单契约](#账单契约)
- **OCR 是短板**：截图模糊、深色模式、水印都可能认错字 —— 确认菜单就是为此存在的
- **单账单**：一次一张截图，不处理合并账单或长账单列表
- Watch 兼容性以实际设备为准，未在表盘端做专门优化

---

> 账目无小事，入账前请以确认菜单里的内容为准，程序输出不构成任何记账保证。
