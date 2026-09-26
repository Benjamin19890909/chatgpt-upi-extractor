# CHATUPI / chatgpt-upi-extractor —— EOL 判定

> 范围：只读代码审阅。不接触任何端点，不复现任何当前有效的令牌 / 指纹 / 版本号 / coupon ID / 金额阈值——
> 所有此类常量一律以其**符号名或角色**指代。
> 目的：判定"到期"状态与失效方式，不提供修复方案。
> 前提（沿用此前失效分析）：一个系统同时依赖多个独立移动的外部快照，其可用期约等于其中变化最快那一个的周期。

代码基线：最后提交 2026-07-22；判定日 2026-09-26。核心链路：`api_python/upi_link_service.py`。

---

## 【一】失效模式清单：把"到期"展开成表

产物定义：整条链路的唯一有效产物是一个 `long_url`（`upi://pay...` 或 Stripe 的 UPI 跳转/instructions URL）。
代码判定"成功"的实质标准只有一条——**是否 `_is_upi_url()` 匹配到了一个 URL**。它**从不校验**这个 URL 是否对应"免费首月"这件事。
这决定了下面第六列里所有"伪装成功"的存在。

图例：**硬失败** = 抛异常、任务落 `failed`；**静默降级** = 不报错、流程继续；**伪装成功** = 无报错且落 `completed`，但产物是错的/无效的/非免费的。

| # | 依赖项（代码位置） | 抄的是谁的什么状态 | 该状态的真实变更频率 | 失效后的可观测表现 | 硬失败 / 静默降级 | 是否伪装成功 |
|---|---|---|---|---|---|---|
| 1 | `IN_CHROME_PROFILES` + fallback 档案（`_pick_india_chrome_profile`） | Chrome 稳定版机队的 TLS/JA3+UA+sec-ch-ua 指纹；以及 `curl_cffi` 所支持的 impersonate 目标名 | Chrome 大版本约 4 周一轮；curl_cffi 支持列表随其自身发版 | 所有档案都不被 curl_cffi 支持 → `RuntimeError`；或指纹整体过旧、被风控加权 | 找不到 profile 时硬失败；能回退到更旧档案时**静默降级**（指纹更可疑） | **是**——回退旧指纹后继续跑，可能触发静默风控 |
| 2 | `CHATGPT_CLIENT_VERSION` / `CHATGPT_CLIENT_BUILD` | ChatGPT Web 客户端的版本号 / build 号（`oai-client-version` 头） | 前端持续发布，日～周级 | 头部版本陈旧 | 通常**静默降级**（服务器多半仍受理旧版本，但抬高风控权重） | **是**——不报错，只是更易被静默拦截 |
| 3 | `STRIPE_VERSION` / `STRIPE_RUNTIME_VERSION` + 两个 checkout beta 开关名 | Stripe API 版本 pin、JS runtime 版本、两个 checkout beta 特性开关的名字 | Stripe API 版本按季度级；beta 开关可随时"毕业"或被移除 | beta 名失效 → init/confirm 行为变化或字段缺失 | beta 名致 400 → 硬失败；语义静默变化 → **静默降级** | 可能——见 #5/#6 的 schema 漂移 |
| 4 | ChatGPT 私有结账协议：`/backend-api/payments/checkout`·`/update`·`/taxes`·`/approve` 的端点、请求体、响应字段（`submission_attempt.state==requires_approval`、`next_action.upi_handle_redirect_or_display_qr_code.hosted_instructions_url` 等） | 一套**未公开、无兼容承诺**的私有 API 的请求/响应契约快照 | 未公开 = 随时，无通知 | 端点 404 / 字段改名 | 4xx 经 `_json_or_error` → 硬失败；**字段改名多半静默**（`_first_string`/`_find_*` 取不到 → 空串/落空） | **是**——字段改名 → 提取到空或错误 URL |
| 5 | Stripe payment_pages 协议：`/v1/payment_pages/{cs}/init`·`/confirm`·`/payment_methods`·poll 的 form 字段、`stripe_hosted_url`、`init_checksum`、`config_id` | Stripe checkout 内部端点契约快照 | 内部端点，无兼容承诺 | 缺 `stripe_hosted_url` → `RuntimeError`；其余字段漂移 → 悄悄传空 | 混合：关键字段缺失硬失败；次要字段**静默降级** | **是** |
| 6 | UPI 跳转 URL 形状：`_is_upi_url` / `_extract_upi_from_text` / `_find_upi_action_url`（`hooks.stripe.com/redirect/*`、含 `upi` 的 stripe 路径、`upi://`） | Stripe/PSP 的 UPI 跳转拓扑与 URL 模式快照 | 支付跳转拓扑，季度级、不通知 | 正则抓不到 → poll 超时；正则**抓错**一个宽松匹配的 stripe URL | 抓不到 → 硬失败；**抓错 → 静默** | **是（重点）**——`_is_upi_url` 过于宽松（任何 `/redirect/` 或含 `upi` 的 stripe 路径都算数），可能返回一个非支付链接却判成功 |
| 7 | `UPI_PROMO_COUPON` / `promo_campaign_id` | 一个**具体限时营销活动**的存在性与命名 | 营销活动，周级，可随时下线 | coupon 不存在 → `check_coupon` 返回不合格或改 shape | precheck 抛"not eligible" → 硬失败；返回不可解析 → 见 #8 **静默** | **是** |
| 8 | Promo 合格性语义：`check_coupon` 的 `state` 枚举（eligible/not_eligible/ineligible）、`redemption.redeemed_by_user`（`_fetch_promo_coupon_state`/`_assert_upi_promo_eligible`） | promo API 的状态枚举与字段名 | 跟随 promo 系统 | 端点移动 / `state` 改名 / 非 200 / 非 JSON → 一律折成 `'unknown'` | **静默降级**——`_assert_upi_promo_eligible` 把 `unknown` 与 `eligible` 一并放行 | **是（重点）**——外部合格性判据一旦失联，代码默认"合格"并继续 |
| 9 | `AMOUNT_POLICY_MAX` + `_expected_amount` 字段路径（`total_summary.due` / `invoice.amount_due` / `line_items[].amount`） | "促销后应付金额"的定价快照 + 金额字段 schema | 定价 + promo，随活动变化 | 金额字段全部改名 → `_expected_amount` 回退到 `0` → `0 > MAX` 为假 → 金额守卫通过，且 `expected_amount=0` 进入 confirm | **静默降级** | **是（最危险）**——既**绕过**金额守卫，又可能生成一个**收费/错误金额**的链接（违背"免费"这一唯一目的） |
| 10 | Sentinel 反自动化：`openai-sentinel-token`、flow 名、可选 `chatgpt_rt_oauth` 模块（`_attach_approve_sentinel`） | OpenAI Sentinel 挑战算法（**运行时计算**，且模块可缺失） | 反滥用系统，随时、对抗性演进 | 无 token → approve 记 `warn` 继续；Sentinel 被强制后 → approve 403/`blocked` | 先**静默降级**（warn 继续），强制后转**硬失败** | 部分——step 标 `warn` 而非 `fail`，运维扫"是否有 ok"时**可能误读为正常** |
| 11 | 会话/JWT 结构：`SESSION_COOKIE_NAMES`、`/api/auth/session→accessToken`、JWT claim 路径（`…/auth.chatgpt_account_id`、profile.email）（`_refresh_access_token_from_session`/`_jwt_*`） | NextAuth 会话形状 + OpenAI JWT 声明结构快照 | 认证栈，季度级但破坏性强 | cookie 名/claim 路径变 → 刷新失败、`account_id` 空、email 回退占位 | 混合：刷新失败**静默回退原 token**；email **静默回退占位地址** | **是**——用占位 email / 未刷新 token 继续跑 |
| 12 | 代理厂商 session DSL：`region-XX`、`sid-…-t-`、`-st-…-sid-`、厂商识别（`proxy_for_region`/`_proxy_with_fresh_sid`/`_uses_lowercase_proxy_region`） | 某特定代理供应商的凭证/会话字符串语法与地理路由约定 | 供应商侧，可随套餐/后台改动 | 正则不匹配 → region 未改写 → 用错国家出口 → checkout 不 offer UPI | `_ensure_upi_offered` 硬失败 | 否（多为硬失败），**但根因被误报为结账阶段**（见【二】） |
| 13 | 地理/支付方式/套餐假设：UPI 仅印度、`chatgptplusplan`、默认 `processor_entity`、INR/IN | 产品在特定地区的支付方式可用性与套餐命名快照 | 产品策略，季度级 | 套餐改名 / UPI 下线 / entity 变更 | `_ensure_upi_offered` 或 checkout 硬失败 | 若响应缺省而套用默认 `processor_entity` 到错误实体，可能**静默** |

**"伪装成功"聚焦（第六列 = 是）**：#6（宽松 URL 匹配抓错链接）、#8（promo 失联即视为合格）、#9（金额字段漂移 → 守卫失效 + 收费链接）是三处最典型的"到期不报错"。
其中 **#9 是整份代码里风险最高的一处**：它同时击穿了金额安全阀与"免费"业务目标，产物看起来是一条正常 UPI 链接，实则可能是一条要付费的链接。

---

## 【二】评审代码自带的失败分类法

分类法的实现主要是 `_infer_fail_stage()`，加上散落在链路里的 `step(name, …)` 名称、以及少数显式写死的 stage：`precheck`、`stale_timeout`。
`_infer_fail_stage` 的逻辑是：**若有 step，就取"最后一个 step 的名字"**；否则按错误字符串里的关键字命中 `checkout_auth / approve / stripe_redirect_poll / promotion_amount / stale_timeout / checkout`，全不中则 `unknown`。

这里有一个贯穿全局的结构性事实，先摆出来，因为后面三问都由它派生：
> **`step()` 只在每个阶段"成功/警告"后才被追加。真正抛异常的那一步从不追加 step。**
> 因此 `steps[-1]` 记录的是"最后一个走通的阶段"，而 `_infer_fail_stage` 把它当成失败阶段。
> 结论：`fail_stage` 标注的是"我走到了哪"，不是"什么坏了"——它系统性地指向失败点的**前一步**。
> 例：`stripe confirm` 请求抛错时，最后成功的 step 是 `payment method`，于是 `fail_stage='payment method'`。

### 1. 哪些类别根因在程序之外，哪些在程序之内？

这套分类法几乎完全按**流水线位置**（走到了协议的哪一环）来切分，而每一环都是一个外部触点。因此：

- **根因在程序之外（外部世界变了）**：`checkout` / `checkout_auth`、`stripe init`、`VN promotion update`、`payment method`、`stripe confirm`、`approve`（含 sentinel/风控）、`stripe_redirect_poll`、`promotion_amount`、`precheck`（promo 真的不合格/已兑换）、以及绝大多数 step 名。它们本质上是"外部快照漂移或外部策略变化"的镜像。
- **根因在程序之内（代码假设错了）**：真正属于这一类的**只有隐性的、且没有被赋予类别的那些**——金额回退到 0、`unknown` 视作合格、宽松 URL 误匹配、email/token 静默回退。它们**不出现在分类法里**（见第 3 问）。显式类别中唯一带内部成分的是 `stale_timeout`：它把"外部太慢"和"内部预算（360s）太短"混为一谈，根因跨内外。

一句话：**这套分类法编码的是"外部世界"，几乎没有为"代码自己的假设错了"留位置。**

### 2. 哪些类别即使把所有硬编码快照更新到最新，依然会落入？（核心）

把 #1–#13 的所有**静态值**（版本号、build、Stripe 版本、指纹、coupon、金额阈值、代理 DSL、套餐名）全部刷到最新，以下类别**依旧会命中**，因为它们的根因不是"值旧了"，而是**协议语义**或**业务规则**：

- **`approve` / sentinel**：Sentinel token 是**运行时计算**的挑战应答，且依赖一个**可选、可能根本没装**的 `chatgpt_rt_oauth` 模块。挑战算法在服务端演进，刷常量无济于事；模块缺失时永远拿不到 token。
- **`approve` 的 `result=blocked`（风控拦截）**：这是服务端的**行为判定**，不是某个值。刷新指纹/版本至多降低概率，无法消除；代码自己也把它标为不可重试（`_is_rebuild_retryable` 对 `blocked` 返回 False）。
- **`stripe_redirect_poll` 超时**：UPI 的 `next_action` 何时、是否出现，取决于 PSP 的异步时序与流程设计。若上游改了"何时下发跳转"，任何常量更新都救不回来。
- **`promotion_amount` / `precheck`（promo 不合格）**：如果这个营销活动**本身已经结束**，就不存在"最新的 coupon"可填——目标不存在，属于**业务规则变更**，是"刷新值"这一动作在定义上无法覆盖的。

即：**sentinel/approve、风控 blocked、redirect 时序、promo 活动消失** 这四类，是"刷新快照"永远治不好的。它们恰好也是变化最快、对抗性最强的几类——按前提，系统半衰期由它们决定。

### 3. 分类法的漏项：会被错归、或落入 error/unknown 兜底的外部变更？（核心）

漏项才是这套可观测性设计的真实价值所在。至少有四处：

- **漏项 A —— "2xx 但 schema 变了"没有类别。** 外部返回 200、但字段改名/挪位，代码不会进入任何失败分支：它要么**伪装成功**（#6/#8/#9），要么在**下游**某步炸掉，然后按"最后成功步"被**错归**。这一整类"静默 schema 漂移"在分类法里是**隐形**的。
- **漏项 B —— "产出了链接但链接是错的/收费的"没有类别。** 成功路径缺少"该链接是否对应免费促销"的校验，于是"错误的成功"（wrong success）完全不可观测，连 `fail_stage` 都不会被写。
- **漏项 C —— `fail_stage` 系统性错位。** 如上，标签恒等于失败点的**前一步**。因此凡是外部世界在"某一步"变了，日志都会把它记到"上一步"头上——分类看起来有值，但**指错了地方**。例如代理地理路由失效（#12）真正炸在 `_ensure_upi_offered`，却会被记成 `stripe init` 或 `checkout` 之类的指纹/初始化阶段；一次**业务/地理策略变更**因而被伪装成一次**基础设施阶段故障**。
- **漏项 D —— `unknown` / 关键字兜底的塌缩。** precheck 阶段（尚无任何 step）的报错只能靠错误字符串里的关键字归类；措辞一变就落 `unknown`。更关键的是 #8：promo 端点一旦移动，`check_coupon` 折成 `'unknown'` → 被当作合格 → **根本不产生任何失败**，于是这类外部变更连 `unknown` 都进不了，直接被"伪装成功"吞掉。

结论：这套分类法的条目数不少，但它的**盲区正好覆盖了最危险的失效方式**——静默 schema 漂移、错误的成功、以及失败点错位。一个"分类完备"的假象，恰恰掩盖了"到期"最典型的表现。

---

## 【三】EOL 声明草稿

### 失效层级判定

CHATUPI 的依赖不落在单一层级，而是三层叠加；按前提"可用期 ≈ 变化最快那一层的周期"，**判定应取最快、最不可逆的那一层**，而非最容易修的那一层。

| 层级 | 本项目中的体现 | 单独看是否可救 |
|---|---|---|
| A. 外部依赖漂移（可更新修复） | 版本/build/Stripe 版本/指纹/代理 DSL/coupon 命名/金额阈值（#1–#3, #7, #9-值, #12） | 值可刷新 |
| B. 协议语义变更（需重写） | 私有结账/Stripe/UPI 跳转的契约、Sentinel 运行时挑战、风控行为、redirect 异步时序（#4–#6, #10, #13） | 刷值无效，需重写交互 |
| C. 业务规则变更（目标已不存在） | 那个"印度首月免费"营销活动本身是否仍然存在（#7, #8） | 若活动下线，无可修 |

**总判定：不是"外部依赖漂移"这一层可以了结的。** 该工具的功能**下限由 B 层与 C 层共同决定**——只要 Sentinel/风控/promo 活动中任一项发生语义级或业务级变更，A 层无论多新都不能恢复功能。因此在缺少运行时举证的前提下，**保守判定为"协议语义变更为主、且可能已叠加业务规则消失"，即倾向"需重写 / 目标可能已不存在"，而非"可更新修复"。**

### 每一层的举证要求（读者要看到什么证据才能接受该判定）

- **要接受"A 层可更新修复"**：需看到——(1) 刷新全部静态常量后，`checkout create` 返回合法 `cs_`；(2) `stripe init` 中 `_payment_method_types` 含 `upi`；(3) `check_coupon` 返回明确的 `eligible`（**不是** `unknown`）；(4) `_expected_amount` 从真实字段解析出**非 0** 且符合免费预期的金额；(5) 走到 `provider redirect` 并产出通过 `_is_upi_url` 且确为支付跳转的 URL。**五条齐备**才成立。
- **要接受"B 层协议语义变更"**：需看到任一——(1) 私有端点 404 或响应字段路径（如 `submission_attempt.state`、`upi_handle_redirect_or_display_qr_code`）不再存在；(2) `approve` 稳定返回 `blocked` 或恒缺 sentinel/`chatgpt_rt_oauth` 不可得；(3) redirect poll 在最大次数内从不出现 UPI `next_action`。这些**刷值不解**，即证语义已变。
- **要接受"C 层业务规则变更"**：需看到——`check_coupon` 对该 promo 稳定返回 `not_eligible`/活动不存在，或官方渠道确认该"印度首月免费"活动已下线。此时"最新 coupon"在定义上不存在。

> 注意：由于【二】所述的 `fail_stage` 错位与 `unknown`/伪装成功盲区，**现有日志本身不足以作为上述任一层的举证**。任何判定都需绕开该分类法、直接观察各阶段的原始响应。这一条本身就是"可观测性已到期"的证据。

### 若判定为"可更新修复"——更新的最小集合（只列项，不列值）

以下是**假设 B/C 层未变**时，恢复功能所需刷新的**最小项集合**（每项均为"需要重新对齐的一类外部快照"，不含任何具体值）：

1. `curl_cffi` 版本及其 impersonate 目标 → 对齐当前 Chrome 稳定版机队（`IN_CHROME_PROFILES` 及 fallback）。
2. ChatGPT Web 客户端版本 / build 常量。
3. Stripe API 版本 / runtime 版本 / 两个 checkout beta 开关名。
4. 促销活动标识（`UPI_PROMO_COUPON` / `promo_campaign_id`）。
5. 促销后金额阈值常量，并同步 `_expected_amount` 的字段路径。
6. 代理厂商的 region/session 令牌 DSL（若更换供应商或其格式变动）。
7. 会话 cookie 名与 JWT claim 路径（若认证栈变动）。
8. 套餐名 / 默认 `processor_entity` / 地理与币种假设（若产品策略变动）。

### 最后一条（判定的必要构成，非修复指南）

上面的最小集合**能被列举，但不充分**：它只覆盖 A 层。凡是落入【二】第 2 问的四类——**Sentinel/approve、风控 blocked、redirect 时序、promo 活动是否仍在**——都**无法通过更新任何值来消除**，因此它们**不属于**"最小更新集合"，也无法被补进去。

按前提的判据："如果最小更新集合为空、或无法穷举，则结论为需重写或目标已不存在。"
这里的情况是其变体、但导向同一结论：**最小更新集合虽可穷举，却不足以恢复功能**——决定可用性的项落在集合之外且不可枚举。
故最终判定：

> **CHATUPI 不是"坏了"，是"到期了"。**
> 它已越过"外部依赖漂移"层，进入"协议语义变更"层，并很可能已触及"业务规则变更"层（促销活动的存续）。
> 刷新全部硬编码快照**不构成充分修复**；能否恢复取决于集合之外、不可枚举的运行时/业务变量。
> 因此判定为 **需重写（若免费促销仍存在）／目标已不存在（若该促销已下线）**，而非"可更新修复"。

---

## 【补】"可更新修复"的逐层检验（思想实验：把每处硬编码快照替换为运行时探测的当前值）

前提：不考虑任何外部变更；假设外部协议/schema 与代码预期完全一致；仅把每处硬编码快照换成"运行时探测到的当前值"。按调用顺序逐层过，找"外部值再新也过不去"的第一处。

| 层 | 调用 | 依赖 | 换成当前值能过？ |
|---|---|---|---|
| 0 | `resolve_upi_proxy`/`resolve_upi_access_token`→`/me` | 代理配置、用户活 token | ✅ 值 |
| 1 | `_fetch_promo_coupon_state`/`_assert_upi_promo_eligible` | coupon 标识 | ✅ 值 |
| 2 | `_pick_india_chrome_profile`/`UpiHttpClient` | curl_cffi impersonate + Chrome 版本元组 | ✅ 值 |
| 3 | `payments/checkout` | plan/coupon/region 请求体→`cs_` | ✅ 值 |
| 4 | `stripe init` | Stripe 版本/pk→`stripe_hosted_url` | ✅ 值 |
| 5 | `_ensure_upi_offered` | 外部不变（题设） | ✅ |
| 6 | `checkout/update`(VN) | promo 请求体 | ✅ 值 |
| 7 | 重生 IN client / `sentinel/ping` 热身 | 无需 token 的裸 ping | ✅ |
| 8 | `stripe init` #2 + `AMOUNT_POLICY_MAX` | 金额阈值 | ✅ 值 |
| 9 | `taxes`/tax region | 账单地址常量 | ✅ 值 |
| 10 | `payment_methods`→`pm_` | 表单值 | ✅ 值 |
| 11 | `confirm`→`requires_approval` | `expected_amount` 值 | ✅ 值 |
| **12** | **`approve` 循环 → `_attach_approve_sentinel`** | **`openai-sentinel-token`** | ❌ **过不去** |

> ⚠️ **本节的初版结论（第一堵墙 = sentinel 缺失，1105–1122）已在下方【补·勘误】中被推翻——保留原文仅为留痕。真正的第一堵墙是 1135–1136 的 `result=='blocked'` 风控裁决。**

**第一堵墙 = approve 阶段的 Sentinel token（`upi_link_service.py` 第 1105–1122 行）。为什么值刷新到底也过不去：**

1. 它不是被存下来的常量——仓库里没有任何 sentinel token 可供"替换"。它是 `build_sentinel_token(...)` 每次请求现算、绑定服务端 challenge 的一次性证明。"探测它的当前值"在定义上不成立：只有"当前算法 + 当前 challenge → 现算"，没有可探测的静态值。
2. 算出它的算法不在本仓库——`_import_build_sentinel_token()` 尝试的 `chatgpt_rt_oauth(.core.direct_protocol)` 两条路径仓库内均不存在（已确认）。故 `build_sentinel_token` 恒为 `None`，`_attach_approve_sentinel` 只写 `_upi_sentinel_error='...module not installed'`，不带 token 发出 approve。
3. 结果是结构性必败，与外部新旧无关：approve 循环跑满 `MAX_APPROVE_ATTEMPTS` 仍拿不到 `approved`，最终抛 `approve failed after retries`/`blocked`。即便在提交当日、外部完全符合预期，这份 as-shipped 代码也产不出 token——缺的是**运行时能力**，不是**过期的值**。

**对第一轮归类的检验结论：** 存在这样一层，所以把它归进"外部依赖漂移（可更新修复）"是**不完整的**。第一轮的"可更新修复"桶隐含假设"每个依赖非静态值即协议 schema"，但 approve 依赖的是**第三类——运行时生成的密码学证明**，其生成器不在交付物里。这一类与"值刷新"这条轴正交：上游 11 层的值无论多新，控制流都会抵达第 12 层而无法以"探测当前值"满足。故 **"可更新修复"不成立**，该层属"需重写（须自带 sentinel 生成能力）"，回归总判定——本工具已越过"外部依赖漂移"层。

（并印证【二】：缺 token 只记 `warn` 不记 `fail`，且 `fail_stage` 按"最后成功步"标注，这处"缺能力"的必败会显示为"approve 重试耗尽/blocked"，真正的第一堵墙在日志里是隐形的。）

---

## 【补·勘误】第一堵墙的重定位（复核后更正）

上一节把第一堵墙定在"sentinel 缺失（1105–1122）"，**引错了循环范围、也认错了 failure signature**。逐行复核如下。

**循环真实范围**：approve 循环体是 **1110–1143**，收敛在 **1144–1145**；旧引用只截到发出 approve 请求的 1122，把两条终止分支排除在外。

**Q1 — `result=='blocked'` 时循环执行几次**：恰好一次迭代，且不回下一轮 `for i`。
1135–1136 `if result=='blocked': raise …` 直接抛异常退出整个循环——不经 1137 quick poll、不经 1143 `time.sleep`、`i` 不前进到 2。`MAX_APPROVE_ATTEMPTS=6` 对 blocked 完全不生效。

**Q2 — `_is_rebuild_retryable(blocked)`**：1154–1155 返回 **False**；且 `extract_upi_link` 的 989–990 在调用它之前就 `if 'blocked by risk control' … raise` 先行重抛，`MAX_REBUILD_ATTEMPTS=2` 亦不生效。叠加：blocked 账号全脚本内 **approve POST 1 次、approve 重试 0 次、rebuild 0 次 = 总计 1 次，硬失败、非重试**。

**Q3 — 补上 sentinel 能力后能否进入"多次尝试"**：不能（若服务端裁决为 blocked）。循环分支完全由服务端 `result` 驱动，与本地是否持有 token 无关；sentinel 是客户端真实性证明，`blocked` 是风控对账号/行为的裁决，二者正交——补 sentinel 不能把 blocked 变 approved。且 1115/1122 只记 `warn` 后照常发请求，**从不中止执行**；中止执行的是 1135–1136（blocked）与 1144–1145（重试耗尽，属非 blocked 的另一条路径）。

**重定位**：第一堵墙 = **1135–1136 的 `result=='blocked'` 风控裁决**，由 **1154–1155** 与 **989–990** 两处"非重试"判定加固。这才是"外部值再新、且补上 sentinel 也过不去"的最早不可约点。

**层级归类修正**：`blocked` 是交易对手的对抗性行为闸门——非 A（值漂移）、非 B（schema 变更本义），在 A/B/C 中最贴近 **C**，语义是"可被稳定自动化的免费提取目标不存在（对手方主动裁决并封锁）"。

> ⚠️ **本节末尾"下滑到 C 邻域"的归类已被【补·再勘误】作废——2026-09-18 的成功观测证伪了"目标已不存在"。blocked 是概率闸门，正确层级为 D（良率衰减）。**

**对"需重写"的修正**：补 sentinel 是**必要但不充分**——它清不掉排在其后的 blocked 裁决，而该裁决对客户端侧任何重写都不可解。故"需重写=充分修复"**不成立**；约束层是对抗性闸门，判定从 B（需重写）**下滑到 C 邻域（目标已不存在/不可稳定自动化）**。净效果：总判定进一步偏向"需重写/目标已不存在"一侧，但机理由"缺 sentinel 能力"更正为"blocked 风控裁决 + 显式非重试设计"。

---

## 【补·再勘误】概率 ≠ 可达性；引入第四层 D，作废 C 归类

**自查出的逻辑错误**：【二】Q2 说 blocked "至多降低概率、无法消除"（关于 `p` 的陈述），【补·勘误】却据此归入 C"目标已不存在"（关于可达性=0 的陈述）。二者不相容——`p>0` 的闸门其通过事件也以 `p>0` 发生。

**决定性证据**：2026-09-18 实际产出过目标（`payments.stripe.com/upi/instructions/<token>`、`amount_due=0`、INR、upi）。这是可达性>0 的存在性证明；且该日期在最后提交 2026-07-22 之后，即 as-shipped 代码本身产出过目标。

**据此更正**：

1. **区分 blocked（概率）与促销下线（C，决定性）的代码判据**：看 `steps`/`fail_stage` 的 promo 授予链（checkout create → stripe init amount → VN checkout/update success → IN reentry amount ≤ policy）——全绿而死于 approve = 概率闸门（非 C）；precheck 或授予链上确定性变红 = 目标下线（C）。文档佐证：README 称 sentinel 影响"成功率"（率），非"可用性"。

2. **"需重写"(B) 与"目标已不存在"(C) 皆不能由"blocked 频发"单独判定**：频发是率值，对二者欠定。C 需可达性=0 的证据（precheck 级确定性不可用、跨全新账号复现 / 授予链变红 / 官方宣告）；B 需"通过率是客户端行为的函数"的证据（合法 sentinel 或特定指纹显著抬升通过率）。

3. **"目标已不存在"被 9-18 成功证伪**，C 作废。主约束既非 A（值层 9-18 完好）、非 B（协议层 9-18 完好）、非 C（可达），需第四层：

   > **D. 概率良率衰减 / 对抗性稀释**：目标以 `p(t)` 可达，`p(t)` 随对手方适配衰减；EOL 判据是 `p(t)` 跌破运营阈值，而非 `p(t)=0`。这正是最初前提"半衰期以周计……到期而非坏了"的准确表述——一个良率/半衰期陈述，非可达性陈述。

4. **"第一堵墙 = blocked 裁决"更正**：blocked 是概率闸门（`1-(1-p)^N→1`），非确定性封锁。真正的结构事实是**错配**——代码以确定性 1-shot 策略处理概率闸门（1135–1136 抛出、1154–1155 False、989–990 抢先重抛，blocked 账号总计 1 次），主动丢弃"重复尝试+换身份"这一唯一逼近杠杆。该约束是**客户端侧 A/B 可改**（或有意的烧号/烧 IP 规避权衡），**不是 C、不是不可解**。9-18 端到端成功证明不存在确定性的"第一堵墙"：每级以某概率可过，产物是复合成功概率。

**总判定最终校准**：不是"外部依赖漂移(A)"、不是"协议语义变更(B)"、不是"目标已不存在(C)"，而是 **D：概率良率衰减**。按良率阈值（而非可用性）判定 EOL；截至 2026-09-18 `p>0`。
