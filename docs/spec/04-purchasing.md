# purchasing 服务规格

## 1. 职责

管理采购订单从创建到收货的完整流转，负责：

- 采购订单创建、修改、确认、取消
- 支持部分收货，生成收货单
- 收货时触发库存入库
- 取消未收货部分
- 严格禁止超收

核心目标：

- 表达真实采购场景，支持分次到货
- 明确采购不预占库存，收货才增加库存
- 支持部分收货，同时避免超收
- 所有写操作幂等
- 状态机清晰，不允许非法跳转

## 2. 采购与销售的区别

| 维度 | 销售 | 采购 |
|---|---|---|
| 库存动作 | 确认时预占，发货时出库 | 收货时入库 |
| 是否预占 | 是 | 否 |
| 部分执行 | 部分发货 | 部分收货 |
| 数量方向 | 减少库存 | 增加库存 |
| 超量规则 | 禁止超发 | 禁止超收 |
| 取消规则 | 释放未发货预占 | 取消未收货部分，无库存影响 |

采购不存在预占，因为采购是入库来源，不占用现有库存。  
采购确认只表示对供应商的承诺，不改变库存。  
只有收货才增加 `on_hand_qty`。

## 3. 领域模型

### 3.1 采购订单 PurchaseOrder

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| order_no | 采购订单号，业务 key，唯一 |
| supplier_code | 供应商编码 |
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
| received_at | 全部收货时间 |
| cancelled_at | 取消时间 |
| version | 乐观锁 |

### 3.2 采购订单行 PurchaseOrderLine

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| order_no | 采购订单号 |
| line_no | 行号 |
| sku | 商品 SKU |
| warehouse_code | 收货仓库编码 |
| ordered_qty | 订购数量 |
| received_qty | 已收货数量 |
| unit_price | 单价，字段预留 |
| amount | 金额，字段预留 |
| currency | 币种，字段预留 |
| status | 行状态 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

行状态：

- OPEN：未收货或部分收货
- RECEIVED：已全部收货
- CANCELLED：已取消

行数量约束：

- `received_qty <= ordered_qty`
- 剩余可收 = `ordered_qty - received_qty`

### 3.3 收货单 GoodsReceipt

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| receipt_no | 收货单号，业务 key，唯一 |
| order_no | 采购订单号 |
| status | 收货单状态 |
| received_at | 收货时间 |
| idempotency_key | 幂等键 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

收货单状态：

- CREATED：已创建，未确认
- CONFIRMED：已确认，已增加库存
- CANCELLED：已取消，未增加库存

MVP 建议：

- 收货单创建即确认，简化流程
- 或保留 CREATED → CONFIRMED 两步，便于对接仓库系统
- 文档保留两步，实现可选

### 3.4 收货单行 GoodsReceiptLine

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| receipt_no | 收货单号 |
| order_no | 采购订单号 |
| order_line_no | 订单行号 |
| sku | 商品 SKU |
| warehouse_code | 仓库编码 |
| received_qty | 本次收货数量 |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

## 4. 状态机
```
DRAFT
│ 确认
▼
CONFIRMED
│ 部分收货
▼
PARTIALLY_RECEIVED
│ 全部收货
▼
RECEIVED

DRAFT / CONFIRMED / PARTIALLY_RECEIVED
│ 取消未收货部分
▼
CANCELLED
```

说明：

- DRAFT：草稿，可编辑
- CONFIRMED：已确认，待收货，不改变库存
- PARTIALLY_RECEIVED：部分收货，剩余待收
- RECEIVED：全部收货，终态
- CANCELLED：取消，未收货部分不再执行

状态跳转规则：

- DRAFT → CONFIRMED：确认采购订单
- DRAFT → CANCELLED：直接取消
- CONFIRMED → PARTIALLY_RECEIVED：部分收货
- CONFIRMED → RECEIVED：全部收货
- CONFIRMED → CANCELLED：取消，未收货部分不再执行
- PARTIALLY_RECEIVED → PARTIALLY_RECEIVED：多次部分收货
- PARTIALLY_RECEIVED → RECEIVED：剩余全部收完
- PARTIALLY_RECEIVED → CANCELLED：取消剩余，已收部分不可取消

禁止：

- DRAFT 直接收货
- RECEIVED 再取消
- CANCELLED 再确认
- 已收部分取消

## 5. 业务规则

### 5.1 创建订单

- 生成采购订单号，业务 key 唯一
- 初始状态 DRAFT
- 可包含多行，每行一个 SKU
- 校验 SKU 存在且 ACTIVE
- 校验供应商存在且 ACTIVE
- 不改变库存
- 必须带幂等键

### 5.2 修改订单

- 仅 DRAFT 可修改
- 可增删行、改数量
- 已确认订单不可修改
- 必须带幂等键

### 5.3 确认订单

- 仅 DRAFT 可确认
- 不预占库存
- 不改变库存
- 状态 → CONFIRMED
- 记录 confirmed_at
- 必须带幂等键

### 5.4 收货

- 仅 CONFIRMED 或 PARTIALLY_RECEIVED 可收货
- 创建收货单，包含本次收货行
- 校验 `本次收货数量 <= 剩余可收数量`
- 严格禁止超收
- 调用 inventory 的 `receiveIn`，增加 `on_hand_qty`
- 写库存流水 `PURCHASE_IN`
- 更新订单行 received_qty
- 更新订单状态：
  - 全部行收完 → RECEIVED
  - 否则 → PARTIALLY_RECEIVED
- 必须带幂等键
- 收货单号唯一，防止重复收货

超收规则：

- 严格禁止 `本次收货数量 > 剩余可收数量`
- 返回 `OVER_RECEIPT`
- 不做容差
- 未来可扩展 `over_receipt_tolerance` 配置

### 5.5 部分收货

- 允许，订单行维护 ordered_qty、received_qty
- 每次收货生成独立收货单
- 剩余未收继续等待
- 全部收完才 RECEIVED
- 多次部分收货允许
- 并发收货必须锁订单行或乐观锁

部分收货的坑：

1. 多次收货：每次生成收货单
2. 超收：禁止
3. 库存增加：收货才增加库存
4. 状态机：部分收货后是 PARTIALLY_RECEIVED
5. 部分取消：已收部分不可取消
6. 并发收货：锁行或乐观锁
7. 幂等：收货接口重试不能重复入库
8. 退货：MVP 不做，文档说明非目标

### 5.6 取消

- DRAFT 可直接取消
- CONFIRMED 可取消，未收货部分不再执行
- PARTIALLY_RECEIVED 可取消剩余，已收部分不可取消
- RECEIVED 不可取消
- 不涉及库存释放，因为采购未预占
- 记录 cancelled_at
- 必须带幂等键

运营人工介入：

- 系统不做规范
- 文档说明可能存在人工调整
- 后续可加审批流

### 5.7 幂等

- 创建订单：幂等键防重复创建
- 确认订单：幂等键 + 状态机防重复确认
- 收货：幂等键 + 收货单号防重复入库
- 取消：幂等键防重复取消

## 6. 并发策略

- 订单行使用乐观锁 version
- 收货时校验剩余可收数量，条件更新
- 并发收货同一订单行，只有一个成功
- 跨服务调用需幂等，失败重试安全

## 7. 业务能力

### 7.1 订单管理

- `createOrder(command)`  
  创建采购订单。入参：供应商编码、订单行列表（SKU、收货仓库、数量、单价）、幂等键。  
  返回采购订单号。初始状态 DRAFT，不改变库存。  
  校验 SKU、供应商存在且 ACTIVE。

- `updateOrder(command)`  
  修改草稿采购订单。入参：订单号、订单行列表、幂等键。  
  仅 DRAFT 可修改，已确认不可改。

- `getOrder(orderNo)`  
  查询采购订单详情，含订单行、状态、数量汇总。

- `listOrders(query)`  
  分页查询采购订单，支持按供应商、状态、创建时间范围过滤。

### 7.2 订单流转

- `confirmOrder(command)`  
  确认采购订单。入参：订单号、幂等键。  
  不预占库存，不改变库存。  
  状态 DRAFT → CONFIRMED。

- `cancelOrder(command)`  
  取消采购订单。入参：订单号、幂等键。  
  取消未收货部分，不涉及库存释放。  
  DRAFT 直接取消，CONFIRMED 取消全部未收，PARTIALLY_RECEIVED 取消剩余。  
  已收部分不可取消。

### 7.3 收货

- `createReceipt(command)`  
  创建收货单。入参：采购订单号、收货行列表（订单行号、收货数量）、幂等键。  
  校验剩余可收数量，生成收货单，状态 CREATED。

- `confirmReceipt(command)`  
  确认收货。入参：收货单号、幂等键。  
  调用 inventory 增加库存，写流水 PURCHASE_IN，更新订单行与订单状态。  
  收货单状态 → CONFIRMED。  
  超收返回 `OVER_RECEIPT`。

- `getReceipt(receiptNo)`  
  查询收货单详情。

- `listReceipts(query)`  
  分页查询收货单，支持按订单号、状态、时间范围过滤。

### 7.4 查询

- `getOrderLines(orderNo)`  
  查询采购订单行，含 ordered_qty、received_qty。

- `getOrderSummary(orderNo)`  
  查询采购订单汇总，含总数量、已收数量、剩余数量。

## 8. 验收场景

1. 创建采购订单，状态 DRAFT，不改变库存
2. 确认采购订单，状态 CONFIRMED，不改变库存
3. 部分收货 10，库存增加 10，状态 PARTIALLY_RECEIVED
4. 再次部分收货 20，库存再增加 20
5. 全部收完，状态 RECEIVED，received_qty = ordered_qty
6. 收货数量超过剩余可收，失败，`OVER_RECEIPT`
7. 确认后取消，状态 CANCELLED，库存不变
8. 部分收货后取消剩余，已收部分不变，剩余不再执行
9. 已全部收货订单取消，失败
10. 重复确认同一订单，幂等返回首次结果，不重复确认
11. 重复收货同一收货单，幂等返回首次结果，不重复入库
12. 并发两个收货请求，只有一个成功
13. 订单行数量约束：received_qty <= ordered_qty
14. 库存流水正确记录 PURCHASE_IN
15. 库存余额与流水对账一致

## 9. 非目标

- 不做退货、换货
- 不做退款
- 不做供应商价格协议
- 不做折扣、促销
- 不做多币种实际结算
- 不做税实际计算
- 不做收货单与物流对接
- 不做审批流
- 不做超收容差
- 不做分布式事务