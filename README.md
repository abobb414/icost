# Icost 自动记账快捷指令

DeepSeek 驱动的 iCost「自动记账」快捷指令：截一张账单截图，自动识别并记进 [iCost](https://apps.apple.com/app/icost-%E8%AE%B0%E8%B4%A6/id1542976087)。

## 工作原理

```
截屏/选图 → 提取图像文本 → DeepSeek 解析成结构化账单
        → 人工确认（金额/日期/分类/账户）
        → 写入 iCost（支出 / 收入 / 转账）
```

- 模型：`deepseek-v4-flash`
- 端点：`https://api.deepseek.com/chat/completions`
- `max_tokens`: 300
- 支持 **支出 / 收入 / 转账** 三种类型，自动匹配 iCost 内的账户与分类
- 解析失败时可手动修正各项后再入账

## 安装

1. 下载 [`Icost.shortcut`](./Icost.shortcut)
2. 双击导入「快捷指令」App
3. 打开该指令，找到 **获取 URL 内容** 这个动作，把 `Authorization` 请求头里的
   `YOUR_DEEPSEEK_API_KEY` 换成你自己的 [DeepSeek API Key](https://platform.deepseek.com/)
4. 运行

> 前提：本机已安装 iCost App 并建好了账户和分类，否则匹配不到会走手动选择。

## 安全说明

- 公开版 **不含任何 API Key**，Key 字段是占位符 `YOUR_DEEPSEEK_API_KEY`
- `.shortcut` 文件是 Apple 加密归档（AEA），但**不要**把填了自己 Key 的版本分享给别人
