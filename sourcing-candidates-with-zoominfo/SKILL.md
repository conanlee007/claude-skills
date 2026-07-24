---
name: sourcing-candidates-with-zoominfo
description: Use when 用户要招聘、找候选人、people search、根据 JD 搜索人才、candidate sourcing、recruiting、人才寻访、挖人、建候选人名单,或提到用 ZoomInfo / LinkedIn 找人选时
---

# 用 ZoomInfo 三段式搜索候选人(JD 驱动)

## Overview

核心原则:**搜索免费,enrich 扣 credit**。流程 = 免费搜名单 → LinkedIn 人工核实 → 只对确认合适者 enrich 拿联系方式。
输入永远是一份明确的 JD。JD 由用户提供并逐项确认,不明确就反复提问,直到 JD 清单全部落实才能搜索。

## Step 0 — 检查 ZoomInfo MCP 是否已安装

用 ToolSearch 搜 `search_contacts`(工具名形如 `mcp__<server-id>__search_contacts`)。找到 → 进入 Step 1。找不到 → 引导用户安装,安装完成前不得进入后续步骤:

1. **前提**:用户需要有公司的 ZoomInfo 订阅账号,且管理员已开通 MCP / AI Copilot 访问权限(没有的话请用户联系公司 ZoomInfo 管理员)
2. **claude.ai / Claude Desktop / Cowork**:Settings → Connectors(连接器)→ 搜索 "ZoomInfo" → Connect → 用 ZoomInfo 账号完成 OAuth 授权
3. **Claude Code CLI**:在交互式会话中运行 `/mcp`,添加 ZoomInfo connector 并完成浏览器 OAuth
4. **验证**:安装后重新 ToolSearch,能看到 `search_contacts` / `enrich_contacts` / `lookup` 即成功

## Step 1 — JD 澄清(硬性门槛,REQUIRED)

先向用户索要 JD 原文(职位描述)。用户没给 JD → 先要,不得替用户代拟。

拿到 JD 后,对照下表**逐项**检查并向用户复述确认。任何一项缺失或含糊 → 用 AskUserQuestion 提问;一轮最多问 4 项,没问完的下一轮继续问,**直到 8 项全部明确**。标注"可为无"的项,也必须由用户亲口确认"无要求",不能默认跳过。

| # | 必须明确的信息 | 对应搜索参数 | 可为"无" |
|---|---|---|---|
| 1 | 职位名称 + 英文同义职称 | jobTitle(用 OR 连接) | 否 |
| 2 | 管理层级(Manager/Director/VP/C-Level) | managementLevel | 否 |
| 3 | 工作地点(城市/州/邮编半径/远程) | metroRegion / state / zipCode+zipCodeRadiusMiles | 否 |
| 4 | 行业或目标公司背景(现任还是曾任) | industryCodes / companyName + companyPastOrPresent | 否 |
| 5 | 最低工作年限 | yearsOfExperience | 是 |
| 6 | 学历 / 学校要求 | degree / school | 是 |
| 7 | 必备技能或关键词 | techSkills / companyDescription | 是 |
| 8 | 排除条件(不要的职称/层级/地区) | excludeJobTitle / excludeManagementLevel / excludedRegions | 是 |

8 项全部确认后,向用户输出一份"JD 搜索条件确认单",用户点头后才进入 Step 2。

## Step 2 — 免费搜索(search_contacts)

1. 先用 `lookup` 取枚举字段合法值(metro-regions、industries、management-levels、employee-count 等)。**禁止凭感觉猜枚举值**——猜错会报错或返回 0 结果
2. 调用 `search_contacts`,除 JD 条件外建议固定附加:
   - `contactAccuracyScoreMin: "85"`(保证人还在职、信息新鲜)
   - `requiredFields: "email,phone"`(确保候选人在 ZoomInfo 有联系方式,避免核实完才发现拿不到)
   - `sort: "-contactAccuracyScore"`、`pageSize: 25`
3. 结果以表格输出:姓名 | 现职 | 公司 | 地点 | 年限 | 准确度分 | personId,并明确告知用户:**此步免费,未消耗任何 credit**
4. 结果太多 → 建议收紧条件重搜(仍免费);结果为 0 → 逐步放宽条件(先放宽职称,再放宽地区)

**实战校准(真实 API 行为):**
- `companyName` 是模糊匹配且**不支持 OR 语法**——传 "GXO OR Amazon" 或缩写(如 "UNIS")会匹配到完全无关的公司(University…)。目标公司定向搜必须:先用 `search_companies` 按公司名拿到 `companyId`,再用 `companyId` 搜联系人;拿到结果后核对公司名确实是目标公司
- 部分过滤字段可能因订阅等级不可用(如 `yearsOfExperience` 报 `Disallowed field`)——报错时移除该字段重搜,把该维度留到 LinkedIn 核实阶段人工把关,并告知用户
- 搜索结果不返回候选人所在城市,地理条件靠 `zipCode+radius` 过滤保证;具体位置在 enrich 或 LinkedIn 阶段确认

## Step 3 — 履历交叉核实(三层,先自动后人工)

**第 1 层 · 自动核实(默认执行,免费合规):** 对候选名单逐个用 WebSearch 查 `"姓名" 公司名 职能关键词 LinkedIn`,从搜索引擎公开索引的 LinkedIn 快照提取:现职头衔、所在城市、工作年限、学历、过往公司,与 ZoomInfo 记录逐项对比。输出核实状态表:✅ 吻合 / ⚠️ 有出入(注明矛盾点)/ ❓ 查不到。发现重名时对比公司+地点判别,判别不了标 ⚠️。

**第 2 层 · 登录态浏览器辅助(可选,仅限第 1 层存疑者):** 经用户同意后,用用户登录态的浏览器(如 Claude in Chrome)以人速逐个打开 LinkedIn 档案页代读,每批不超过 10 人,读完即停。

**第 3 层 · 人工兜底:** 仍无法确认的交用户自行核实。

**禁止:** 未登录爬 LinkedIn 档案页、绕过 authwall、批量自动化抓取或写脚本翻页——违反 LinkedIn 用户协议,有封号风险。第 2 层只做"代读",不做采集。

核实状态表交用户圈定最终名单后,进入 Step 4。

## Step 4 — enrich 拿联系方式(扣 credit,必须先展示候选人明细并获用户确认)

`enrich_contacts` 消耗 ZoomInfo Bulk Credits(同一联系人首次 enrich 后一年内重复 enrich 免费)。

**扣费前确认流程(REQUIRED,每次扣费前都要走一遍):**
1. 把将要 enrich 的候选人**明细表**完整展示给用户:姓名 | 现职 | 公司 | 地点 | personId
2. 明确告知:本次将对 N 人执行 enrich,消耗 credit
3. 等用户**明确回复确认**(如"确认"/"就这几个")后,才可调用 enrich_contacts

**禁止在展示明细并获确认之前调用 enrich_contacts,没有例外:**
- 用户之前说过"都可以"/"直接拿" → 仍要展示明细再确认
- 名单本来就是用户自己给的 → 仍要展示明细再确认(核对姓名与 personId 匹配无误)
- 只有一个人、"很明显" → 仍要展示明细再确认

- 用 Step 2 拿到的 `personId` 定位(最准确),每批最多 10 人
- `requiredFields` 一次拿全(同一次扣费):`email, phone, mobilePhone, employmentHistory, education, externalUrls, contactAccuracyScore, jobTitle, companyName`
- 用返回的 `employmentHistory` 和 `externalUrls`(含 LinkedIn 链接)与用户在 LinkedIn 看到的履历交叉复核,确认是同一人

## Credit 规则速查

| 操作 | Credit 消耗 |
|---|---|
| search_contacts / lookup | 免费 |
| enrich_contacts | 扣 Bulk Credits;同人一年内重复 enrich 免费 |
| contact_research | 官方未标注,按可能扣费对待,非必要不用 |

## Common Mistakes

- **JD 没问全就开搜** → 名单不匹配,浪费用户核实时间。8 项清单缺一不搜
- **跳过 lookup 猜 metroRegion / industry 枚举值** → 报错或 0 结果
- **搜完直接全量 enrich** → credit 白烧。只 enrich LinkedIn 核实过的人
- **用户给了确认名单就直接 enrich** → 仍必须先展示待 enrich 候选人明细表并获得本次扣费确认,核对人名与 personId 无误后才执行
- **jobTitle 只写一个中文职称** → 用英文,OR 连接多个同义职称(如 "Warehouse Manager OR DC Manager OR Distribution Center Manager")
- **漏了 requiredFields: "email,phone"** → 用户核实完人选,回头发现 ZoomInfo 没有其联系方式
