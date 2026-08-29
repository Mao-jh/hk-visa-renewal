# HTTP 402 支付流程（香港身份续签·风险预审）

本文是本 Skill 内嵌的支付执行流程：从 402 待支付账单提取、支付、查询到自动履约。
支付能力来自支付宝官方 `alipay-bot` CLI（npm 包 `@alipay/agent-payment`），
本 Skill 不碰私钥、不做验签、不碰资金，只编排官方命令。

## 第 1 步 · 保存待支付需求

收到 `POST https://airpriv.top/api/hk-visa-renewal` 的 `402` 响应后：

- 完整保存账单原文到文件（文件名只用字母、数字、连字符、下划线、点号）：
  - 首选：把响应体中的 `payment_needed_token` 字符串原样写入文件（与 `Payment-Needed` header 内容一致）；或
  - 直接保存 `Payment-Needed` header 原文（base64url，不解码、不改写）。
- 同时保存：资源 URL（`https://airpriv.top/api/hk-visa-renewal`）、method（`POST`）、
  原请求 body JSON（`--data` 参数要原样复用）。
- 从本次用户请求生成 `--intent-summary`，如：`原始请求：香港身份续签风险预审（高才/优才/IANG/专才个人情况）`。
- 发起支付前**不要**重新请求资源来刷新账单。

## 第 2 步 · CLI 就绪检查（缺失时安装）

执行支付命令前，若 shell 明确返回找不到或无法解析 `alipay-bot`：

1. **先检查常见已装路径**（用户可能装过支付宝官方 AI 付体验版，CLI 存在但不在 PATH）：
   - `~/.openclaw-autoclaw/alipay-bot-cli/bin/alipay-bot`（Windows 为 `alipay-bot.cmd`）
   - 找到后直接用完整路径执行后续命令，不重复安装。

2. 常见路径也没有时，向用户说明需要安装支付宝官方支付 CLI
   （npm 包 `@alipay/agent-payment@1.0.20`），**先取得用户明确同意**。

3. 校验该版本发布完整性：

```bash
npm view '@alipay/agent-payment@1.0.20' dist.integrity --registry 'https://registry.npmjs.org'
```

4. 结果必须逐字符等于：

```text
sha512-OR78BdjoHueJ7XgKPNxawUFbptc9E4CX9PpV8dHLguGOKQeM+CYRPQuRrxltPqZzyTeta372B/xyFb6uwRYfew==
```

不一致或无法查询时停止，不安装。

5. 一致时安装精确版本：

```bash
npm install -g '@alipay/agent-payment@1.0.20' --registry 'https://registry.npmjs.org'
```

6. 安装成功后仅重试最初失败的命令一次。

用户拒绝安装 → 回到 SKILL.md「降级路径」（手动 proof 模式）。

## 第 3 步 · 发起支付

`sessionId` 取当前框架业务会话的真实会话 ID（UUID 格式，如 sessionId / threadId /
conversationId）；无法取得时停止并说明缺少业务会话标识。

```bash
alipay-bot 402-buyer-pay --session-id '<sessionId>' --file '<第1步保存的账单文件>' --resource-url 'https://airpriv.top/api/hk-visa-renewal' --intent-summary '原始请求：香港身份续签风险预审' --method 'POST' --data '<原请求body JSON>'
```

- `--data` 必须与原请求完全一致（含引号时注意 shell 转义）。
- 命令会输出支付二维码/支付卡片 → 原样转述给用户，请用户用支付宝扫码支付（¥2.99）。
- 用户支付属于本人决策；Agent 不得代替用户确认支付。
- 待确认支付：保存输出中对客标签后的号码，等用户完成支付或要求查询时进入第 4 步：
  - 「交易号」标签后的值（本服务实测 buyer-pay 输出通常**只有**这个标签）；
  - 「订单号」或「查询单号」标签后的值：32 位数字且第 11–14 位为 `8282` 时才是有效查询材料。
  ⚠️ 查询号码**只从 buyer-pay 输出的对客标签取**；402 账单里的商户单号（`ORDER_` 开头的
  out_trade_no）**不是**查询号码，传给 `--out-shake-no` 会查询失败（实测）。
- 「支付方式」下的链接仅供用户扫码操作，不是查询或履约参数；**不要用链接自查支付状态**
  （实测支付成功后链接查询仍可能显示「业务单不存在」，状态判定一律以第 4 步 CLI 查询为准）。

## 第 4 步 · 查询支付状态（支付后）

用户完成扫码支付后（**首选交易号路径**，本服务实测稳定）：

```bash
# 首选：buyer-pay 输出的「交易号」标签后的值
alipay-bot 402-query-payment-status --trade-no '<交易号>' --resource-url 'https://airpriv.top/api/hk-visa-renewal' --method 'POST' --data '<原请求body JSON>'

# 仅当输出含 32 位数字（第 11–14 位为 8282）的「订单号/查询单号」时可用：
alipay-bot 402-query-payment-status --out-shake-no '<查询单号>' --resource-url 'https://airpriv.top/api/hk-visa-renewal' --method 'POST' --data '<原请求body JSON>'
```

- 查询成功后 CLI 自动：验付 → 重发资源请求 → 返回风险报告 → 上报履约回执。
- 仅命令超时或网络失败可重试一次；业务失败不重试。
- 没有任何号码时说明支付会话已失效并停止；重新触发一次资源请求走新账单即可。

## 第 5 步 · 资源与履约

- 原样输出命令返回的报告与错误，不自行补充成功结论；报告转述遵守 SKILL.md
  「转述纪律」（保留口径/案例编号、逐条复述必知清单、复述免责）。
- buyer-pay/query 取得非空资源后已自动发送履约回执，**不追加**独立履约或重复查询。
- 同一次支付中 CLI 明确表示履约回执发送失败并给出「交易号」、且用户后续要求恢复时，
  执行一次：

```bash
alipay-bot 402-buyer-fulfillment-ack --trade-no '<同次输出的交易号>'
```

该命令只恢复回执，不重新请求资源。
