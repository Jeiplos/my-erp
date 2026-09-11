# sales 服务规格

## 1. 职责

管理销售订单从创建到发货的完整流转，负责：

- 销售订单创建、修改、确认、取消
- 订单确认时触发库存预占
- 支持部分发货，生成发货单
- 发货时触发库存预占消耗与出库
- 取消未发货部分时释放剩余预占

核心目标：

- 结合 Amazon 订单流转，表达真实销售场景
- 明确预占在销售流程中的关键作用
- 支持部分发货，同时避免超发
- 所有写操作幂等
- 状态机清晰，不允许非法跳转

## 2. 与 Amazon 订单流转的映射

### 2.1 Amazon 订单常见状态

| Amazon 状态 | 含义 |
|---|---|
| Pending | 待处理，付款未完成或待验证 |
| Unshipped | 已付款，待发货 |
| PartiallyShipped | 部分发货 |
| Shipped | 全部发货 |
| Canceled | 取消 |
| 退款/退货 | 后置流程，MVP 不做 |

### 2.2 ERP 销售订单状态映射

| Amazon | ERP | 库存动作 |
|---|---|---|
| 草稿 | DRAFT | 不预占 |
| Pending | PENDING | 可选：不预占，或临时预占，MVP 不预占 |
| Unshipped | CONFIRMED | **确认时预占库存** |
| PartiallyShipped | PARTIALLY_SHIPPED | 已发部分扣减，剩余继续预占 |
| Shipped | SHIPPED | 全部发货，预占清零 |
| Canceled | CANCELLED | 释放未发货预占 |

### 2.3 状态机
```
DRAFT
│ 确认
▼
CONFIRMED
│ 部分发货
▼
PARTIALLY_SHIPPED
│ 全部发货
▼
SHIPPED

DRAFT / CONFIRMED / PARTIALLY_SHIPPED
│ 取消未发货部分
▼
CANCELLED
```


说明：

- DRAFT：草稿，可编辑，不预占
- CONFIRMED：已确认，已预占，待发货
- PARTIALLY_SHIPPED：部分发货，剩余继续预占
- SHIPPED：全部发货，终态
- CANCELLED：取消，释放未发货预占

状态跳转规则：

- DRAFT → CONFIRMED：触发预占
- DRAFT → CANCELLED：直接取消，无预占
- CONFIRMED → PARTIALLY_SHIPPED：部分发货
- CONFIRMED → SHIPPED：全部发货
- CONFIRMED → CANCELLED：取消，释放全部预占
- PARTIALLY_SHIPPED → PARTIALLY_SHIPPED：多次部分发货
- PARTIALLY_SHIPPED → SHIPPED：剩余全部发完
- PARTIALLY_SHIPPED → CANCELLED：取消剩余，已发部分不可取消

禁止：

- DRAFT 直接发货
- SHIPPED 再取消
- CANCELLED 再确认
- 已发部分取消

## 3. 领域模型

### 3.1 销售订单 SalesOrder

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| order_no | 订单号，业务 key，唯一 |
| customer_code | 客户编码 |
| status | 订单状态 |
| currency | 币种，字段预留 |
| exchange_rate | 汇率，字段预留 |
| total_amount | 总金额，字段预留 |
| tax_amount | 税额，字段预留 |
| remark | 备注 |
| idempotency_key | 幂等键 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| confirmed_at | 确认时间 |
| shipped_at | 全部发货时间 |
| cancelled_at | 取消时间 |
| version | 乐观锁 |

### 3.2 销售订单行 SalesOrderLine

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| order_no | 订单号 |
| line_no | 行号 |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| ordered_qty | 订购数量 |
| reserved_qty | 已预占数量 |
| shipped_qty | 已发货数量 |
| unit_price | 单价，字段预留 |
| amount | 金额，字段预留 |
| currency | 币种，字段预留 |
| status | 行状态 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

行状态：

- OPEN：未发货或部分发货
- SHIPPED：已全部发货
- CANCELLED：已取消

行数量约束：

- `reserved_qty <= ordered_qty`
- `shipped_qty <= ordered_qty`
- 剩余可发 = `ordered_qty - shipped_qty`
- 剩余预占 = `reserved_qty`

### 3.3 发货单 SalesShipment

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| shipment_no | 发货单号，业务 key，唯一 |
| order_no | 订单号 |
| status | 发货单状态 |
| shipped_at | 发货时间 |
| idempotency_key | 幂等键 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

发货单状态：

- CREATED：已创建，未确认
- CONFIRMED：已确认，已扣库存
- CANCELLED：已取消，未扣库存

MVP 建议：

- 发货单创建即确认，简化流程
- 或保留 CREATED → CONFIRMED 两步，便于对接仓库系统
- 文档保留两步，实现可选

### 3.4 发货单行 SalesShipmentLine

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| shipment_no | 发货单号 |
| order_no | 订单号 |
| order_line_no | 订单行号 |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| shipped_qty | 本次发货数量 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

## 4. 业务规则

### 4.1 创建订单

- 生成订单号，业务 key 唯一
- 初始状态 DRAFT
- 可包含多行，每行一个 SKU
- 校验 SKU 存在且 ACTIVE
- 校验客户存在且 ACTIVE
- 不预占
- 必须带幂等键

### 4.2 修改订单

- 仅 DRAFT 可修改
- 可增删行、改数量
- 已确认订单不可修改
- 必须带幂等键

### 4.3 确认订单

- 仅 DRAFT 可确认
- 逐行预占，调用 inventory 的 `reserve`
- 全部成功才确认，任一行失败整体回滚
- 预占成功后状态 → CONFIRMED
- 记录 confirmed_at
- 必须带幂等键
- 幂等键与订单号共同防止重复预占

预占失败场景：

- 可用量不足，返回 `INSUFFICIENT_STOCK`
- 部分行成功、部分失败，必须释放已成功预占
- 释放使用 `release`，保证不残留

### 4.4 发货

- 仅 CONFIRMED 或 PARTIALLY_SHIPPED 可发货
- 创建发货单，包含本次发货行
- 校验 `本次发货数量 <= 剩余可发数量`
- 校验 `本次发货数量 <= 剩余预占`
- 调用 inventory 的 `consume`，消耗预占并出库
- 更新订单行 shipped_qty、reserved_qty
- 更新订单状态：
  - 全部行发完 → SHIPPED
  - 否则 → PARTIALLY_SHIPPED
- 必须带幂等键
- 发货单号唯一，防止重复发货

超发规则：

- 严格禁止 `本次发货数量 > 剩余可发数量`
- 返回 `OVER_SHIPMENT`

### 4.5 部分发货

- 允许，订单行维护 ordered_qty、reserved_qty、shipped_qty
- 每次发货生成独立发货单
- 剩余未发继续预占
- 全部发完才 SHIPPED
- 多次部分发货允许
- 并发发货必须锁订单行或乐观锁

部分发货的坑：

1. 多次发货：每次生成发货单
2. 超发：禁止
3. 预占释放：发货后 reserved_qty 减少
4. 状态机：部分发货后是 PARTIALLY_SHIPPED
5. 部分取消：已发部分不可取消
6. 并发发货：锁行或乐观锁
7. 幂等：发货接口重试不能重复扣库存
8. 退货：MVP 不做，文档说明非目标

### 4.6 取消

- DRAFT 可直接取消
- CONFIRMED 可取消，释放全部预占
- PARTIALLY_SHIPPED 可取消剩余，释放剩余预占，已发部分不可取消
- SHIPPED 不可取消
- 调用 inventory 的 `release`
- 记录 cancelled_at
- 必须带幂等键

运营人工介入：

- 系统不做规范
- 文档说明可能存在人工调整
- 后续可加审批流

### 4.7 幂等

- 创建订单：幂等键防重复创建
- 确认订单：幂等键 + 状态机防重复预占
- 发货：幂等键 + 发货单号防重复扣库存
- 取消：幂等键防重复释放

## 5. 并发策略

- 订单行使用乐观锁 version
- 确认订单时逐行预占，失败回滚
- 发货时校验剩余可发数量，条件更新
- 并发发货同一订单行，只有一个成功
- 跨服务调用需幂等，失败重试安全

## 6. 业务能力

### 6.1 订单管理

- `createOrder(command)`  
  创建销售订单。入参：客户编码、订单行列表（SKU、仓库、数量、单价）、幂等键。  
  返回订单号。初始状态 DRAFT，不预占。  
  校验 SKU、客户存在且 ACTIVE。

- `updateOrder(command)`  
  修改草稿订单。入参：订单号、订单行列表、幂等键。  
  仅 DRAFT 可修改，已确认不可改。

- `getOrder(orderNo)`  
  查询订单详情，含订单行、状态、数量汇总。

- `listOrders(query)`  
  分页查询订单，支持按客户、状态、创建时间范围过滤。

### 6.2 订单流转

- `confirmOrder(command)`  
  确认订单。入参：订单号、幂等键。  
  逐行预占库存，全部成功才确认。  
  失败返回 `INSUFFICIENT_STOCK`，并释放已成功预占。  
  状态 DRAFT → CONFIRMED。

- `cancelOrder(command)`  
  取消订单。入参：订单号、幂等键。  
  释放未发货预占。  
  DRAFT 直接取消，CONFIRMED 释放全部，PARTIALLY_SHIPPED 释放剩余。  
  已发部分不可取消。

### 6.3 发货

- `createShipment(command)`  
  创建发货单。入参：订单号、发货行列表（订单行号、发货数量）、幂等键。  
  校验剩余可发数量，生成发货单，状态 CREATED。

- `confirmShipment(command)`  
  确认发货。入参：发货单号、幂等键。  
  调用 inventory 消耗预占并出库，更新订单行与订单状态。  
  发货单状态 → CONFIRMED。  
  超发返回 `OVER_SHIPMENT`。

- `getShipment(shipmentNo)`  
  查询发货单详情。

- `listShipments(query)`  
  分页查询发货单，支持按订单号、状态、时间范围过滤。

### 6.4 查询

- `getOrderLines(orderNo)`  
  查询订单行，含 ordered_qty、reserved_qty、shipped_qty。

- `getOrderSummary(orderNo)`  
  查询订单汇总，含总数量、已发数量、剩余数量。

## 7. 验收场景

1. 创建订单，状态 DRAFT，不预占
2. 确认订单，库存充足，预占成功，状态 CONFIRMED
3. 确认订单，库存不足，失败，`INSUFFICIENT_STOCK`，无残留预占
4. 确认后部分发货 10，状态 PARTIALLY_SHIPPED，reserved 减少 10
5. 再次部分发货 20，剩余继续预占
6. 全部发完，状态 SHIPPED，reserved 清零
7. 发货数量超过剩余可发，失败，`OVER_SHIPMENT`
8. 确认后取消，释放全部预占，状态 CANCELLED
9. 部分发货后取消剩余，释放剩余预占，已发部分不变
10. 已发货订单取消，失败
11. 重复确认同一订单，幂等返回首次结果，不重复预占
12. 重复发货同一发货单，幂等返回首次结果，不重复扣库存
13. 并发两个发货请求，只有一个成功
14. 订单行数量约束：shipped_qty <= ordered_qty
15. 订单行数量约束：reserved_qty <= ordered_qty

## 8. 非目标

- 不做退货、换货
- 不做退款
- 不做价格协议
- 不做折扣、促销
- 不做多币种实际结算
- 不做税实际计算
- 不做发货单与物流对接
- 不做审批流
- 不做分布式事务