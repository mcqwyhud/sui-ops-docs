# 节点运行监控与健康度 API

> 本文档面向外部监控系统或需要自研链路容灾、熔断机制的租户。介绍如何通过总控提供的节点列表及网络健康检测接口，实时分析底层每个入站节点（Inbound）的延迟、状态和故障率。

---

## 1. 基础信息

* **接口模块基地址**：https://你的总控域名/api/node
* **鉴权方式**：HTTP Header 中需携带有效令牌：`Authorization: Bearer <您的API管理员Token>`

---

## 2. 接口列表

### 2.1 获取全量节点列表（带分页与条件筛选）

本接口会同步从持久化数据库（固定vps）与当前在线动态vps中实时汇总节点数据

* **请求方式**：GET
* **请求路径**：/list

**Query 筛选参数（全部选填，默认返回全量）**：

| 参数键名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `name` | String | 按节点名称模糊搜索。 |
| `host` | String | 按节点域名/IP地址过滤。 |
| `region` | String | 按节点所属区域（如：香港、日本）过滤。 |
| `serverId` | String | 关联的特定 VPS 服务器 ID。 |
| `enabled` | Boolean | 节点是否启用（true/false）。 |
| `protocol` | String | 协议类型（如：shadowsocks、vmess 等）。 |
| `page` | int | 当前页码，默认 `1`。 |
| `limit` | int | 每页返回条数，默认 `10`。 |

**成功返回示例**：

{
    "code": 200,
    "msg": "success",
    "data": {
    "total": 45,
    "page": 1,
    "limit": 2,
    "data": [
        {
            "id": 501,
            "uniqueId": "node_hk_ss_01",
            "name": "香港 01 | 游戏专线",
            "host": "hk01.suiops.com",
            "port": 443,
            "protocol": "shadowsocks",
            "region": "香港",
            "description": "",
            "serverId": "vps_hk_smart_01",
            "enabled": true,
            "available": true,
            "priority": 10,
            "userCount": 38,
            "latency": 15,
            "createTime": 1772582400000,
            "updateTime": 1772668800000,
            "state": "normal",
            "subUrlEnum": "smart"
        },
        {
            "id": 502,
            "uniqueId": "node_jp_vmess_02",
            "name": "日本 02 | 视频大带宽",
            "host": "jp02.suiops.com",
            "port": 80,
            "protocol": "vmess",
            "region": "日本",
            "description": "",
            "serverId": "vps_jp_fixed_01",
            "enabled": true,
            "available": true,
            "priority": 5,
            "userCount": 150,
            "latency": 65,
            "createTime": 1772582400000,
            "updateTime": 1772668800000,
            "state": "congestion",
            "subUrlEnum": "standard"
        }
    ]
    }
}

---

### 2.2 获取全局节点健康度与故障率

此接口由总控内置的探测链路系统（`NodeCheckSystem`）提供，能够高精、高频地吐出当前集群中所有活动入站节点的底层失败概率。是自研自动化链路熔断脚本的核心依据。

* **请求方式**：GET
* **请求路径**：/node-health

**成功返回示例**：

{
    "code": 200,
    "msg": "success",
    "data": [
        {
            "nodeKey": "hk01.suiops.com:443",
            "status": "normal",
            "failureRate": 0.0
        },
        {
            "nodeKey": "jp02.suiops.com:80",
            "status": "congestion",
            "failureRate": 0.12
        },
        {
            "nodeKey": "us01.suiops.com:443",
            "status": "fault",
            "failureRate": 1.0
        }
    ]
}

---

## 3. 核心监控字段深度剖析

对于需要对接自动化监控运维系统的开发者，请严格参照以下业务指标：

### 3.1 节点网络状态码 (`state` / `status`)
系统将每一个网络端点定义为三种状态：
* **normal**：健康。链路畅通，丢包极低，可放心分配给用户。
* **congestion**：拥堵。当前承载的用户数过多或由于晚高峰国际出口劣化导致部分丢包，但仍勉强可用。
* **fault**：故障。节点已彻底失联、心跳断开或端口已被阻断。

### 3.2 精准故障率 (`failureRate`)
该值是由检测系统实时计算得出的浮点数，范围为 `0.0 ~ 1.0`：
* `0.0`：代表检测周期内该节点 100% 成功连通。
* `0.12`：代表该节点近期有 12% 的探测遭遇了超时或失败（链路质量正在劣化，通常伴随 `congestion` 状态）。
* `1.0`：代表连续多次探测完全失败，此时系统会将状态直接断言为 `fault`。

---

## 4. 开发者进阶：如何做自动化链路保护？

如果您对接了三方看板（如 Uptime Kuma）或自研熔断脚本，建议如下：
1. **秒级抓取**：外接脚本每隔 15~30 秒调用一次 `/api/node/node-health`。
2. **业务熔断**：当脚本发现某个特定节点的 `failureRate` 连续 3 次超过 `0.50`，说明其可用性已经基本断绝。此时可调用子节点的api直接控制s-ui进行主动下线隔离，保护用户端软件不再自动尝试连接它，降低客服工单率。
3. **拥堵分流**：若检测到 `status == "congestion"`，可在您自己的发卡或财务系统前端对新购用户优先推荐其他 `normal` 的低负载节点。
