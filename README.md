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
