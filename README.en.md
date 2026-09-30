<p align="center"><a href="./README.md">简体中文</a> | <b>English</b></p>

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
