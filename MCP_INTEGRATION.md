# MCP 集成说明

> 本文说明「麦麦代点」实际使用的麦当劳 MCP Server、Tool、调用流程与业务价值。
> 全部调用均为**真实调用**，返回样张取自 2026-10-10 的实测记录。

---

## 一、连接器配置

**MCP Server**：`https://mcp.mcd.cn`（传输方式 `streamablehttp`）

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      }
    }
  }
}
```

> ⚠️ 仓库与前端代码中**不包含任何真实 Token**。`YOUR_MCP_TOKEN` 为占位符，真实值只从服务端环境变量 `MCD_TOKEN` 读取。

---

## 二、使用的 Tool 清单

本项目共使用 **5 个只读工具**，全部为查询类，**不使用任何写操作**。

| # | Tool | 用途 | 关键入参 | 是否必需 |
|---|---|---|---|---|
| 1 | `query-nearby-stores` | 定位门店，拿 `storeCode` | `beType=1, searchType=2, city, keyword` | 必需 |
| 2 | `query-meals` | 拉真实在售菜单与价格 | `beType=1, orderType=1, storeCode` | 必需 |
| 3 | **`query-meal-detail`** | **拿配料可增减结构（核心）** | `beType=1, orderType=1, storeCode, code` | **必需** |
| 4 | `list-nutrition-foods` | 营养阈值校验 | 无（全量返回） | 必需 |
| 5 | `calculate-price` | 带特调传参做真实核价 | `storeCode, orderType, products[]` | 必需 |

**明确不使用的工具**（出于合规考虑）：

- `create-order` / `mall-create-order` —— 下单纯属用户决策，不由本工具代执行
- `auto-bind-coupons` —— 领券涉及用户资产，不做静默操作
- `draw-lottery` —— 与项目目标无关

---

## 三、调用流程

```
① 定位门店
   query-nearby-stores(beType=1, searchType=2, city="上海", keyword="华旭国际大厦")
        ↓  得到 storeCode = 1450713
② 拉菜单
   query-meals(beType=1, orderType=1, storeCode="1450713")
        ↓  得到 128 项在售餐品（含 productCode 与价格）
③ 拿配料可增减结构   ★ 核心步骤
   query-meal-detail(beType=1, orderType=1, storeCode="1450713", code="1406")
        ↓  得到 modification.items[].values[]（可去掉的配料）
④ 营养校验
   list-nutrition-foods()
        ↓  按餐品名匹配营养，逐项标注「精确 / 存疑 / 无数据」
⑤ 真实核价
   calculate-price(storeCode, orderType, products[] 含特调 key)
        ↓  拿到券后实付价
⑥ 输出三件套：餐单 / 特制清单 / 说明卡
```

**约束求解发生在 ②→⑤ 之间**：硬约束过滤候选池 → 用 ③ 的配料结构做改写 → 用 ④ 做阈值判定（存疑项跳过）→ 用 ⑤ 核价。

---

## 四、真实返回样张

### 4.1 `query-meal-detail` —— 全项目的技术地基

请求：

```json
{ "beType": 1, "code": "1406", "orderType": 1, "storeCode": "1450713" }
```

返回（节选）：

```json
{
  "code": "1406",
  "name": "板烧鸡腿堡",
  "supportModify": true,
  "modification": {
    "items": [
      {
        "maxValues": 2,
        "minValues": 0,
        "values": [
          { "code": "102201", "price": 0, "name": "烤味酱",
            "selectedKey": "0-1", "unselectedKey": "0-0" },
          { "code": "120512", "price": 0, "name": "切块生菜",
            "selectedKey": "0-1", "unselectedKey": "0-0" }
        ]
      }
    ]
  }
}
```

**结论**：

- `supportModify: true` → 该餐品支持配料改写
- `unselectedKey` 不为空 → **该配料可以去掉**
- `price: 0` → 去配料**不加价**

→ **「去掉烤味酱」是一个真实可执行、且不加价的操作。** 这证明产品能落地，不是概念演示。

### 4.2 `query-nearby-stores` —— 附带发现

请求：`{ "beType": 1, "searchType": 2, "city": "上海", "keyword": "华旭国际大厦" }`

返回：门店 `1450713`「麦当劳上海黄浦华旭国际大厦餐厅」，且**返回体包含 `businessStartTime` / `businessEndTime`**。

> 补充说明：有参赛项目在报名文中写道"MCP 未提供营业时间字段"并据此放弃了时间类功能。实测与该说法不符 —— 营业时间字段是存在的。

### 4.3 `list-nutrition-foods`

返回 **160 行**六维营养数据：能量 / 蛋白质 / 脂肪 / 碳水化合物 / 钠 / 钙。

**注意其命名口径**：营养表使用「小杯玉米杯」「中薯条」这类**规格名**，与 `query-meals` 返回的商品名不完全对齐 → 这是本项目必须处理的数据缺口（见第七节）。

---

## 五、特调传参规则（最容易出错，必须遵守）

官方文档明确要求：

> **对包含 `unselectedKey` 的特调组，该组内所有特调项都必须传入算价与下单接口。**

| 用户意图 | 传参 |
|---|---|
| 保留该配料 | 传 `selectedKey`（如 `"0-1"`） |
| 去掉该配料 | 传 `unselectedKey`（如 `"0-0"`） |

举例 —— 板烧鸡腿堡「去掉烤味酱、保留切块生菜」：

```json
{ "values": [
  { "code": "102201", "key": "0-0", "quantity": 1 },
  { "code": "120512", "key": "0-1", "quantity": 1 }
]}
```

**漏传未选中的项会导致算价错误。** 这是本项目实现中最容易踩的坑，原型与 Skill 均已完整实现。

另外两条：

- **套餐的子项**在 `rounds[].choices[]` 里，每个 choice 也可能带自己的 `modification`，需要**递归处理**
- 当特调组 `minValues=1 && maxValues=1` 且 `values` 只有 1 项时，该项为**必选**，不得向用户提供"不选"的选项

---

## 六、业务价值

### 6.1 为什么用 MCP 而不是硬编码菜单

- 餐品与价格**按门店、按时段变化**（早/正/夜市菜单不同）→ 只有实时调用才能保证方案可下单
- 配料可增减结构**只能通过 `query-meal-detail` 获得**，这是 MCP 独有、且其它数据源无法替代的能力

### 6.2 解决的真实问题

| 用户问题 | MCP 如何解决 |
|---|---|
| "他不能吃某样，这菜能改吗？" | `query-meal-detail` 的 `modification` 直接给出可去掉的配料与传参 key |
| "这顿超了多少？" | `list-nutrition-foods` 给六维数值，与用户设定阈值比对 |
| "这些改动要加钱吗？" | `query-meal-detail` 的 `price` 字段 + `calculate-price` 真实核价 |
| "去店后怎么说？" | 基于特调结构生成**可直接念给店员**的清单 |

### 6.3 调用量控制（关键设计）

门店菜单与营养表变化频率低（菜单按日活动、营养表近乎静态）→ **每店每小时拉一次**即可。

按 1000 DAU 估算，MCP 调用量约 **24 次/店/天，与用户量无关** —— 这是把"读扩散"压成"写一次"的关键，直接决定这个站能不能扛住突然的流量。

---

## 七、数据边界（涉及 MCP 能力的诚实声明）

| 官方数据缺口 | 实测依据 | 本项目的处理 |
|---|---|---|
| **营养表滞后于菜单** | 有参赛项目实测某门店 128 个在售餐品仅 28 个能匹配营养表（**21.9%**） | 每个餐品标注**营养匹配状态**：精确 / 存疑 / 无数据；**存疑项不参与阈值硬判定** |
| **营养值为标准配方值** | 板烧鸡腿堡标准配方钠 1041 mg；去酱后官方不提供新数值 | 改写项标注「已改写」，但**明确声明不量化改写后的营养变化**（只做定性说明） |
| **营养接口无过敏原字段** | `list-nutrition-foods` 只返回六项营养值，无过敏原 | 结论限定为「**配料层面能否规避**」，**不承诺医学安全**；花生/坚果/麸质等高危过敏提示咨询门店 |

> 我们选择**显式补上这三个缺口并诚实标注边界**，而不是编造数据。这是产品可信度的来源。

---

## 八、安全与合规

| 项 | 做法 |
|---|---|
| Token 存放 | 仅服务端环境变量 `MCD_TOKEN`，**不进前端、不进仓库** |
| 工具选择 | 只用 5 个只读工具，**不用任何写操作** |
| 下单 | 网页端只输出"可复制的下单指令"，由用户在 WorkBuddy 中确认后执行 |
| 输出内容 | 不携带 Token、账号、订单号等敏感信息 |
| 演示数据 | 门店名为公开营业信息；"妈妈"为虚构人物，非真实用户 |
