# inventory 服务规格

## 1. 职责

管理库存余额、预占、流水，提供入库、出库、预占、释放、调整等能力。

核心目标：

- 保证不出现负库存
- 保证预占可追溯、可释放、可对账
- 保证每一次库存变动都有不可变流水
- 保证并发场景下数据一致

## 2. 核心概念

### 2.1 三个数量

| 字段 | 含义 | 说明 |
|---|---|---|
| on_hand_qty | 现存量 | 仓库实际存在的数量 |
| reserved_qty | 预占量 | 已被订单锁定、尚未出库的数量 |
| available_qty | 可用量 | `on_hand_qty - reserved_qty` |

约束：

- `on_hand_qty >= 0`
- `reserved_qty >= 0`
- `on_hand_qty >= reserved_qty`
- `available_qty >= 0`

### 2.2 库存余额 InventoryBalance

按 `SKU + 仓库` 维度记录。一期单仓，但模型支持多仓。

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| on_hand_qty | 现存量 |
| reserved_qty | 预占量 |
| available_qty | 可用量，冗余字段，便于查询 |
| version | 乐观锁 |
| created_at / by | 审计 |
| updated_at / by | 审计 |

说明：

- `available_qty` 可作为冗余字段，由 `on_hand_qty - reserved_qty` 计算，写入时保持一致
- 查询可直接用冗余字段，扣减必须用条件更新保证一致

### 2.3 预占 InventoryReservation

每一笔预占对应一个来源单据行，可追溯、可释放。

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| reservation_no | 预占单号，唯一 |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| source_type | 来源类型，如 SALES_ORDER |
| source_no | 来源单号 |
| source_line_no | 来源行号 |
| reserved_qty | 预占数量 |
| released_qty | 已释放数量 |
| status | ACTIVE / RELEASED / CONSUMED |
| idempotency_key | 幂等键 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

状态说明：

- ACTIVE：预占生效中
- RELEASED：已释放（订单取消）
- CONSUMED：已消耗（发货扣减）

### 2.4 库存流水 InventoryTransaction

每一次库存变动都写一条不可变流水。

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| transaction_no | 流水号，唯一 |
| type | 流水类型 |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| on_hand_delta | 现存量变动 |
| reserved_delta | 预占量变动 |
| on_hand_after | 变动后现存量 |
| reserved_after | 变动后预占量 |
| source_type | 来源类型 |
| source_no | 来源单号 |
| source_line_no | 来源行号 |
| unit_cost | 可选，成本字段，一期不计算 |
| total_cost | 可选，成本字段，一期不计算 |
| idempotency_key | 幂等键 |
| created_at / by | 审计，流水表不保留 updated |

流水类型：

| 类型 | 含义 | on_hand_delta | reserved_delta |
|---|---|---|---|
| PURCHASE_IN | 采购入库 | + | 0 |
| SALES_RESERVE | 销售预占 | 0 | + |
| SALES_RELEASE | 销售释放 | 0 | - |
| SALES_SHIP_OUT | 销售发货出库 | - | - |
| ADJUST_IN | 手工调整入 | + | 0 |
| ADJUST_OUT | 手工调整出 | - | 0 |

说明：

- 流水不可修改、不可删除
- 余额是快照，流水是事实来源
- 余额与流水必须能对账

## 3. 预占模型（重点）

### 3.1 为什么预占是 ERP 核心复杂性

ERP 与普通电商后台最大的区别之一：**库存不是单一数字，而是多个状态的数量叠加**。

如果没有预占，会出现：

- 两个订单同时确认，都看到库存 10，都卖出去，实际只有 10 件，导致超卖
- 订单确认后迟迟不发货，库存被占用但系统不知道
- 部分发货后，剩余数量是否还能卖，无法判断
- 取消订单时，不知道释放多少库存

预占模型把库存拆成 `on_hand_qty`、`reserved_qty`、`available_qty` 三个量，所有销售确认、发货、取消都必须操作这三个量。

### 3.2 预占生命周期
```
创建预占 SALES_RESERVE
│ reserved_qty += 预占数量
│ available_qty -= 预占数量
▼
预占生效 ACTIVE
│
├── 订单取消 ──▶ 释放预占 SALES_RELEASE
│ reserved_qty -= 释放数量
│ available_qty += 释放数量
│ 状态 → RELEASED
│
└── 订单发货 ──▶ 消耗预占 SALES_SHIP_OUT
                 on_hand_qty -= 发货数量
                 reserved_qty -= 发货数量
                 状态 → CONSUMED
```


### 3.3 预占设计原则

- 预占必须可追溯：每笔预占对应来源单据行
- 预占必须可释放：订单取消、发货时释放
- 预占必须防重复：同一来源行不能重复预占
- 预占必须防负库存：`on_hand_qty >= reserved_qty >= 0`
- 预占不设过期时间：由订单状态驱动释放
- 预占表必须存在，否则无法表达部分发货、取消剩余等场景

### 3.4 部分发货下的预占

- 订单行维护 `ordered_qty`、`reserved_qty`、`shipped_qty`
- 每次发货：消耗对应预占，`reserved_qty -= 发货数量`
- 剩余未发：继续预占，状态保持 ACTIVE
- 全部发完：预占状态 → CONSUMED
- 取消剩余：释放剩余预占，状态 → RELEASED

## 4. 业务规则

### 4.1 入库

- 来源：采购收货、手工调整入
- 增加 `on_hand_qty`
- 不影响 `reserved_qty`
- 写流水 `PURCHASE_IN` 或 `ADJUST_IN`
- 必须带幂等键

### 4.2 出库

- 来源：销售发货、手工调整出
- 减少 `on_hand_qty`
- 销售发货同时减少 `reserved_qty`
- 必须校验 `on_hand_qty >= 出库数量`
- 写流水 `SALES_SHIP_OUT` 或 `ADJUST_OUT`
- 必须带幂等键

### 4.3 预占

- 来源：销售订单确认
- 增加 `reserved_qty`
- 必须校验 `available_qty >= 预占数量`
- 写流水 `SALES_RESERVE`
- 写预占记录，状态 ACTIVE
- 必须带幂等键

### 4.4 释放

- 来源：销售订单取消未发部分
- 减少 `reserved_qty`
- 写流水 `SALES_RELEASE`
- 更新预占记录，状态 RELEASED
- 必须带幂等键

### 4.5 防负库存

- 所有扣减使用条件更新：`UPDATE ... WHERE on_hand_qty >= ? AND reserved_qty >= ?`
- 或使用行锁：`SELECT ... FOR UPDATE`
- 更新影响行数为 0，抛出 `INSUFFICIENT_STOCK`
- 禁止任何绕过余额表的直接修改

### 4.6 并发策略

- 条件更新 + 乐观锁 `version`
- 高并发下可退化为行锁
- 同一 SKU + 仓库的变动串行化
- 跨 SKU 的批量操作按 SKU 排序加锁，避免死锁

## 5. 状态机

### 5.1 库存余额

无状态机，只有数量约束。

### 5.2 预占

```
ACTIVE ──释放──▶ RELEASED
│
└──消耗──▶ CONSUMED
```


- ACTIVE：生效中
- RELEASED：已释放
- CONSUMED：已消耗

## 6. 业务能力

### 6.1 库存查询

- `getBalance(sku, warehouseCode)`  
  查询单个 SKU 在指定仓库的余额，返回 on_hand、reserved、available。

- `listBalances(query)`  
  分页查询库存余额，支持按 SKU、仓库、可用量范围过滤。

- `getAvailableQty(sku, warehouseCode)`  
  查询可用量，供销售确认前校验使用。

### 6.2 预占

- `reserve(command)`  
  创建预占。入参：sku、warehouseCode、数量、来源类型、来源单号、来源行号、幂等键。  
  校验可用量，增加 reserved_qty，写流水，写预占记录。  
  失败返回 `INSUFFICIENT_STOCK`。

- `release(command)`  
  释放预占。入参：预占单号或来源行、释放数量、幂等键。  
  减少 reserved_qty，写流水，更新预占状态。

- `consume(command)`  
  消耗预占并出库。入参：预占单号或来源行、发货数量、幂等键。  
  同时减少 on_hand_qty 和 reserved_qty，写流水，更新预占状态。  
  用于销售发货。

- `getReservation(reservationNo)`  
  查询预占详情。

- `listReservations(query)`  
  分页查询预占，支持按来源单号、状态、SKU 过滤。

### 6.3 入库

- `receiveIn(command)`  
  入库。入参：sku、warehouseCode、数量、来源类型、来源单号、来源行号、幂等键。  
  增加 on_hand_qty，写流水。  
  用于采购收货、手工调整入。

### 6.4 出库

- `shipOut(command)`  
  出库。入参：sku、warehouseCode、数量、来源类型、来源单号、来源行号、幂等键。  
  校验 on_hand_qty，减少 on_hand_qty，写流水。  
  用于手工调整出。销售发货走 `consume`。

### 6.5 调整

- `adjustIn(command)`  
  手工调整入，增加 on_hand_qty，写流水 ADJUST_IN。

- `adjustOut(command)`  
  手工调整出，减少 on_hand_qty，写流水 ADJUST_OUT。

### 6.6 流水查询

- `getTransaction(transactionNo)`  
  查询单条流水。

- `listTransactions(query)`  
  分页查询流水，支持按 SKU、仓库、类型、来源单号、时间范围过滤。

### 6.7 幂等

- `getIdempotentResult(idempotencyKey)`  
  查询幂等键对应的结果，供内部使用。

## 7. 验收场景

1. 采购入库 100，on_hand=100，reserved=0，available=100
2. 销售预占 30，on_hand=100，reserved=30，available=70
3. 再预占 80，失败，`INSUFFICIENT_STOCK`
4. 释放 30，on_hand=100，reserved=0，available=100
5. 预占 30，发货 10，on_hand=90，reserved=20，available=70
6. 发货 25，失败，超过预占数量
7. 发货 20，on_hand=70，reserved=0，available=70，预占状态 CONSUMED
8. 手工调整出 100，失败，`INSUFFICIENT_STOCK`
9. 手工调整入 50，on_hand=120
10. 重复调用同一幂等键的预占，返回首次结果，不重复预占
11. 并发两个预占请求，库存只够一个，只有一个成功
12. 流水与余额对账：on_hand 变动 = 流水 on_hand_delta 之和
13. 流水与余额对账：reserved 变动 = 流水 reserved_delta 之和
14. 取消订单释放预占，预占状态 RELEASED
15. 部分发货后取消剩余，剩余预占释放，已发部分不变

## 8. 非目标

- 不做成本核算
- 不做多仓调拨
- 不做批次、序列号、有效期
- 不做盘点
- 不做预占过期自动释放
- 不做库存预警
- 不做安全库存
- 不做分布式事务