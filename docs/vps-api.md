# 接入云商与 VPS 集群管理 API

> 本文档面向具备自研自动化运维（DevOps）能力的租户。介绍如何通过总控提供的 VPS 列表接口，联动各大外接云商（如 AWS、Stripe、阿里云、搬瓦工等）API 实现节点的资产盘点与集群自动扩缩容（Auto-Scaling）。

---

## 1. 为什么需要 VPS API？

在 SUI-Ops 系统中，普通的节点接入**完全不需要任何 API**：您只需在购买的 VPS 上运行总控提供的“一键安装脚本”，节点便会通过底层分布式集群 自动向总控注册并上线。

**那么，为什么还要开放此接口？**
因为高级开发者需要利用总控的实时运行数据，来实现**物理级节点的动态扩缩容与自愈**。通过高频轮询本接口，您可以实时监控：
1. **负载状态判断**：监控特定地理位置（`location`）下，动态节点（`vpsEnum: "smart"`）的当前在线人数与最大承载力，从而决定是否通过云商 API 自动开机（创建全新 VPS 自动跑脚本加进来分流）。
2. **固定资源盘点**：获取节点到期时间（`serverExpireTime`）或已分配用户数，自动决定是否要向云商续费，或在固定节点人满时自动采购新机器。

---

## 2. 基础信息

* **接口模块基地址**：https://你的总控域名/api/vps
* **鉴权方式**：HTTP Header 中需携带有效令牌：`Authorization: Bearer <您的API管理员Token>`

---

## 3. 接口列表

### 3.1 获取全部 VPS 实时资产与负载列表

* **请求方式**：GET
* **请求路径**：/list

**成功返回示例**：

{
    "code": 200,
    "message": "success",
    "data": [
        {
            "id": 1024,
            "serverID": "vps_hk_smart_01",
            "online": true,
            "onlineCountValue": 45,
            "allocatedUserCountValue": 120,
            "ip": "8.8.8.8",
            "providerName": "瓦工",
            "providerUrl": "[https://bandwagonhost.com](https://bandwagonhost.com)",
            "subUrl": "[http://8.8.8.8:8080/sub/](http://8.8.8.8:8080/sub/)",
            "location": "香港",
            "serverBandwidth": 1000,
            "serverTraffic": 5000,
            "allocatedTraffic": 1250,
            "vpsType": "flagship",
            "vpsTagName": "香港高防 01",
            "vpsEnum": "smart",
            "maxUsers": 200,
            "serverExpireTime": 1783180800000,
            "createdAt": 1772582400000,
            "updatedAt": 1772668800000,
            "dynamic": true
        },
        {
            "id": 1025,
            "serverID": "vps_sg_fixed_02",
            "online": true,
            "onlineCountValue": 12,
            "allocatedUserCountValue": 50,
            "ip": "9.9.9.9",
            "providerName": "AWS",
            "providerUrl": "[https://aws.amazon.com](https://aws.amazon.com)",
            "subUrl": "[http://9.9.9.9:8080/sub/](http://9.9.9.9:8080/sub/)",
            "location": "新加坡",
            "serverBandwidth": 500,
            "serverTraffic": 2000,
            "allocatedTraffic": 800,
            "vpsType": "startup",
            "vpsTagName": "新加坡固定 02",
            "vpsEnum": "standard",
            "maxUsers": 50,
            "serverExpireTime": 1775164800000,
            "createdAt": 1772582400000,
            "updatedAt": 1772668800000,
            "dynamic": false
        }
    ]
}

---

## 4. 核心字段说明（DevOps 扩缩容必读）

为了方便您的自动化脚本对接到云商接口，请重点关注以下字段及数据库中清洗出的核心字段：

| 字段键名 | 类型 | 语义说明 | 自动化脚本开发业务指导 |
| :--- | :--- | :--- | :--- |
| **vpsEnum** | String | 节点弹性枚举。`smart`（动态节点）、`standard`（固定节点）。 | 专门筛选 `smart` 类型的机器来计算是否要触发云商动态开机。 |
| **onlineCountValue** | Integer | **该 VPS 当前实时的在线人数**。 | 用于计算瞬时负载率（当前在线 / `maxUsers`）。 |
| **maxUsers** | Integer | 该 VPS 承载的用户数上限。 | * **固定节点**：代表 CDK 允许分配给该机器的最大承载人头数。<br>* **动态节点**：代表最大在线人数（扩缩容计算分母）。 |
| **allocatedUserCountValue** | Integer | **该 VPS 已经塞进去了多少个合约用户**。 | 专门针对固定节点。如果该值逼近 `maxUsers`，说明卡槽快满了，脚本应当去云商再买一台固定节点。 |
| **location** | String | 地理位置（如：香港、日本、美国）。 | 扩容脚本在调用云商开机 API 时，应当根据该区域参数，去开对应云商相同 Region 的机子。 |
| **serverExpireTime** | Long | 该 VPS 的到期时间戳（毫秒级）。 | 您的外接财务系统可以写个定时脚本，发现到期时间小于 3 天，全自动调云商 API 自动扣款续费。 |

---

## 5. 自动化弹性扩容脚本（思路示例）

您的外接脚本逻辑伪代码应该如下：
1. 请求 `/api/vps/list` 拿到数据。
2. 过滤出 `vpsEnum == "smart"` 且 `location == "香港"` 的所有 VPS。
3. 统计香港所有动态 VPS 的总 `onlineCountValue` 和总 `maxUsers`。
4. 计算平均负载率：`CurrentLoad = TotalOnline / TotalMax`。
5. **触发开机**：如果 `CurrentLoad > 0.80`，说明香港节点要爆了。脚本立即调用 AWS/搬瓦工 API 创建一台新香港机器，在其 `UserData` 里塞入总控的一键拉起脚本。机器启动后自动通过集群网络向总控注册，香港集群瞬间完成无感扩容。
6. **触发关机**：如果 `CurrentLoad < 0.20` 且有多余的空闲 smart 机器，脚本可以调用云商 API 销毁/关机部分机器以节约运营本钱。