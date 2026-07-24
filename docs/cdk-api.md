# CDK 兑换码开放接口 API

> 本文档面向外部发卡平台、租户自研财务系统以及三方支付网关。介绍如何通过 API 自动化创建、查询以及全自动、无感地为用户激活（使用）CDK 兑换码。

---

## 1. 基础信息

* **接口模块基地址**：https://你的总控域名/api/cdk
* **鉴权方式**：HTTP Header 中需携带有效令牌：`Authorization: Bearer <您的API管理员Token>`

---

## 2. 核心业务流转场景（开发者必读）

除了用于传统的发卡网 囤货或实时生成卡密外，本套接口完美支持**租户自研前端与独立支付系统**的对接，实现用户“全自动无感充值下发”：

1. **用户前台下单**：用户在您的自研前台选择套餐，拉起独立支付系统。
2. **支付回调接收**：当您的系统收到支付网关发送的“付款成功”异步回调（Webhook）后。
3. **后台全自动闭环**：
    * 您的后端首先调用 **3.1 `/create`** 接口，在总控生成一个对应权益的 CDK。
    * 紧接着，您的后端自动把拿到的 CDK 码和该用户的账号传入 **3.3 `/use-cdk`** 接口执行激活。
4. **前端无感交付**：激活成功后，直接在前台向用户展示全新生成的订阅链接，用户全程**不需要手动复制和粘贴任何卡密**，完成完美的自动化商业闭环。

---

## 3. 接口列表

### 3.1 单个创建 CDK

用于收到付款回调成功后，实时向总控申请生成一个定制的激活码。

* **请求方式**：POST
* **请求路径**：/api/cdk/create
* **Content-Type** : application/json

**传参核心规则（二选一说明）**：
* **【推荐】方案 A（传 planId）**：如果您填写了 `planId`（套餐 ID），总控会自动继承后台该套餐的所有预设权益（流量、时长、节点数等）。此时，**除了必填的 `creatorId` 和属于上级代理分销的 `prevAdminId` 之外，其余所有权益与定价参数均可不填（不传或传 null/0）**。
* **方案 B（不传 planId）**：如果您需要临时为某个用户单独定制专属权益，请不传 `planId`，此时必须手动填入 `monthlyTraffic` 和 `vipDuration` 等参数。

**请求 Body 参数说明**：

| 字段键名 | 类型 | 是否必填 | 语义与说明 |
| :--- | :--- | :---: | :--- |
| **creatorId** | Long | 是 | **创建者（API管理员）的 ID**。 |
| planId | String | **推荐** | **关联的总控套餐（订阅模板）ID**。填了它，下方的权益参数可全不传。 |
| prevAdminId | Long | 选填 | 上级代理/分销商管理员 ID（用于分销结算）。 |
| monthlyTraffic | Long | 选填 | 月流量上限（单位：Byte 字节）。方案 A 可不传。 |
| vipDuration | Integer | 选填 | VIP 有效时长（单位：天）。方案 A 可不传。 |
| vpsCount | Integer | 否 | 静态 VPS 额度数量，默认 0。 |
| dynamicVpsType | String | 否 | 动态 VPS 资源类型分类。 |
| price | Double | 否 | 销售定价，默认 0.0。 |
| cost | Double | 否 | 本钱/成本，默认 0.0。 |
| vpsType | String | 否 | 静态 VPS 资源类型分类。 |

**方案 A 请求 Body 示例（填了 planId 的极简调用）**：

{
"creatorId": 10024,
"planId": "plan_flagship_monthly",
"prevAdminId": 10001
}

**成功返回示例**：

{
"code": 200,
"message": "success",
"data": {
"id": 500123,
"cdkCode": "SUI-AK9F8E7D6C5B",
"monthlyTraffic": 107374182400,
"vipDuration": 30,
"status": 0,
"planId": "plan_flagship_monthly"
}
}

> **业务安全提示**：如果采用方案 B 自定义控制流量，由于流量输入为字节数，例如 100GB 的流量请在外接端后端换算为：100 * 1024 * 1024 * 1024 = 107374182400。

---

### 3.2 批量创建 CDK

用于站长或大代理在第三方平台提前批量囤货，生成激活码列表导入库存。

* **请求方式**：POST
* **请求路径**：/api/cdk/batchCreate
* **Content-Type** : application/json

**请求 Body 参数说明**：
同样支持**传 `planId` 从而免填其他权益字段**的精简调用。在 3.1 的基础参数之上，Body 额外增加一个控制数量的必填字段：

| 字段键名 | 类型 | 是否必填 | 语义与说明 |
| :--- | :--- | :---: | :--- |
| **count** | Integer | 是 | **本次批量生成的 CDK 数量**。 |

**方案 A 批量请求 Body 示例（填了 planId 的极简调用）**：

{
"count": 5,
"creatorId": 10024,
"planId": "plan_startup_quarterly"
}

**成功返回示例**：

{
"code": 200,
"message": "success",
"data": [
{ "id": 500124, "cdkCode": "SUI-BATCH001A" },
{ "id": 500125, "cdkCode": "SUI-BATCH001B" }
]
}

---

### 3.3 激活/使用 CDK（核心业务）

支付系统自动化调用（无感流转）或用户在前台手动兑换时调用，输入用户账号和 CDK 码，直接下发订阅配置。

* **请求方式**：POST
* **请求路径**：/api/cdk/use-cdk
* **Content-Type** : application/x-www-form-urlencoded

**Query 传参说明（直接追加在 URL 后面或表单提交）**：

| 参数键名 | 类型 | 是否必填 | 语义与说明 |
| :--- | :--- | :---: | :--- |
| **account** | String | 是 | **目标用户账号**（邮箱或用户名）。 |
| **cdk** | String | 是 | **待激活的 CDK 兑换码**。 |

**成功返回示例**：

{
"code": 200,
"message": "订阅激活成功！",
"data": {
"subName": "旗舰版月付套餐",
"subs": "https://xxx.com/...",
"subs1": "https://xxx.com/..."
}
}

> **业务流转说明**：激活成功后，data 域中会同步下发主订阅链接（subs）与高可用容灾备用订阅链接（subs1）。无论是您的自研系统还是第三方系统，拿到后直接引导客户端连接此配置即可。

---

### 3.4 根据 CDK 码查询详情

财务系统用于核验 CDK 是否存在、是否有效或查看其包含的权益信息。

* **请求方式**：GET
* **请求路径**：/api/cdk/code/{cdkCode}
* **示例路径**：/api/cdk/code/SUI-AK9F8E7D6C5B

**成功返回示例**：

{
"code": 200,
"message": "success",
"data": {
"id": 500123,
"cdkCode": "SUI-AK9F8E7D6C5B",
"monthlyTraffic": 107374182400,
"vipDuration": 30,
"status": 0
}
}

**失败返回示例**：

{
"code": 500,
"message": "CDK不存在",
"data": null
}