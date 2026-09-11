# 领域服务 (Domain Service)

## 一句话定义

领域服务是**属于业务规则、但不属于任何单个实体**的操作。
它没有状态，方法名就是一句业务话术。

## 具体例子

「把 10 个 A 商品从 1 号仓调到 2 号仓」这条规则，放在哪个实体上都不合适：

- 放在 1 号仓的库存对象上？可它必须同时改 2 号仓，那 2 号仓不该由它管。
- 放在 2 号仓上？同理。
- 放在某个「仓库」对象上？它就得同时知道两个库存对象，边界又变大了。

于是抽成领域服务：

```java
public class StockTransferService {           // 领域服务：无状态
    public void transfer(ProductId product, WarehouseId from, WarehouseId to, Quantity qty) {
        InventoryBalance src = balances.findBy(product, from).orElseThrow(...);
        InventoryBalance dst = balances.findBy(product, to).orElseThrow(...);

        src.deduct(qty);      // 规则仍然住在各自的实体里
        dst.increase(qty);    // 服务只负责"把它们凑到一起"
    }
}
```

## 和应用服务的区别（最容易混的一对）

| | 领域服务 Domain Service | 应用服务 Application Service |
| --- | --- | --- |
| 内容 | **业务规则**（怎么算才对） | **流程编排**（先干什么后干什么） |
| 状态 | 无状态、纯业务 | 负责事务、加载/保存、调用外部 |
| 会做的事 | 计算、判断、协调多个领域对象 | `@Transactional`、调仓储、发事件、转 DTO |
| 会做的事（反面） | 不管事务、不写 SQL | 不写业务规则 |
| 典型命名 | `StockTransferService`、`PricingService` | `ShipOrderAppService`、`OrderFacade` |

自检方法：**把数据库和 Spring 全部拿掉，这段逻辑还成立吗？**
成立 → 领域服务；不成立（因为要查库/开事务）→ 应用服务。

## 在本项目里怎么用

预计需要的领域服务很少，这也正常：

| 领域服务 | 职责 |
| --- | --- |
| `StockTransferService` | 跨仓库调拨（如果实现调拨功能） |
| `PricingService` | 需要按客户等级/促销算价时的价格计算 |

**能放在实体上的规则就不要抽领域服务**。领域服务是「实在没地方放」时的去处，
不是默认选项——滥用它会把对象退化成数据袋（见[贫血模型](anemic-vs-rich-model.md)）。

## 常见误解

- **误解一**：Service 类就是领域服务。Spring 里常见的 `XxxService` 多数其实是**应用服务**。
- **误解二**：领域服务里可以查数据库。查库属于编排，一般交给应用服务或仓储接口调用方。
- **误解三**：每个实体都该配一个领域服务。那样只是把贫血模型换了层皮。
