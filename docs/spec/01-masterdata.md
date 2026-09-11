# masterdata 服务规格

## 1. 职责

管理商品、客户、供应商、仓库等主数据，为其他服务提供基础数据。

## 2. 领域模型

### 2.1 商品 Product

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| sku | 业务 key，唯一，不可复用 |
| name | 名称 |
| category | 分类 |
| base_unit | 基本单位 |
| status | ACTIVE / INACTIVE |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

规则：

- SKU 唯一
- 停用后不可再被新单据引用
- 停用后 SKU 不可复用
- 物理删除禁止

### 2.2 客户 Customer

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| code | 业务 key，唯一，不可复用 |
| name | 名称 |
| contact | 联系方式 |
| address | 地址 |
| status | ACTIVE / INACTIVE |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

### 2.3 供应商 Supplier

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| code | 业务 key，唯一，不可复用 |
| name | 名称 |
| contact | 联系方式 |
| address | 地址 |
| status | ACTIVE / INACTIVE |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

### 2.4 仓库 Warehouse

| 字段 | 说明 |
|---|---|
| id | 内部 ID |
| code | 业务 key，唯一，不可复用 |
| name | 名称 |
| status | ACTIVE / INACTIVE |
| created_at / by | 审计 |
| updated_at / by | 审计 |
| version | 乐观锁 |

一期单仓，但模型支持多仓。

## 3. 业务规则

### 3.1 新增

- SKU / code 必须唯一
- 必填字段校验
- 初始状态 ACTIVE

### 3.2 修改

- 允许修改名称、联系方式、地址等
- 不允许修改 SKU / code
- 停用后不允许修改关键字段

### 3.3 停用

- 逻辑删除，status = INACTIVE
- 保留记录
- 不可再被新单据引用
- 已有单据仍可引用

### 3.4 查询

- 支持按 SKU / code 精确查询
- 支持按名称模糊查询
- 支持分页
- 支持按状态过滤
- 默认只返回 ACTIVE

## 4. 状态机
```
ACTIVE ──停用──▶ INACTIVE
  ▲                 │
  └──启用（可选）────┘
```


MVP 可只做 ACTIVE → INACTIVE，启用后续再说。

## 5. 业务能力

- `createProduct`
- `updateProduct`
- `deactivateProduct`
- `getProductBySku`
- `listProducts`
- `createCustomer`
- `updateCustomer`
- `deactivateCustomer`
- `getCustomerByCode`
- `listCustomers`
- `createSupplier`
- `updateSupplier`
- `deactivateSupplier`
- `getSupplierByCode`
- `listSuppliers`
- `createWarehouse`
- `updateWarehouse`
- `deactivateWarehouse`
- `getWarehouseByCode`
- `listWarehouses`

## 6. 验收场景

1. 创建商品，SKU 唯一，成功
2. 重复 SKU 创建，失败，错误码 `SKU_DUPLICATE`
3. 停用商品，再创建同 SKU，失败
4. 停用商品，已有订单仍可引用
5. 修改商品 SKU，失败
6. 查询商品列表，分页正确
7. 查询商品列表，默认只返回 ACTIVE
8. 客户、供应商、仓库同理

## 7. 非目标

- 不做层级分类
- 不做多单位
- 不做价格协议
- 不做信用额度
- 不做联系人多记录