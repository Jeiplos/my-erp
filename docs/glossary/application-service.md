# 应用服务 (Application Service)

## 一句话定义

应用服务负责**编排一次用例的完整流程**：取数据 → 调用领域逻辑 → 存数据（+ 开事务、发事件、转 DTO）。
它自己**不承载业务规则**。

一句话区分：**领域层回答「怎么算才对」，应用服务回答「先干什么、后干什么」。**

## 具体例子

```java
@Service
public class OrderApplicationService {

    private final SalesOrderRepository orders;
    private final InventoryService inventory;

    @Transactional                                    // ← 事务边界在这里
    public SalesOrderView ship(String orderNo) {
        SalesOrder order = orders.findByOrderNo(orderNo)      // 1. 取
                .orElseThrow(() -> new OrderNotFoundException(orderNo));

        order.ship();                                 // 2. 业务规则（在领域对象里）

        for (OrderLine line : order.getLines()) {     // 3. 协调另一个模块
            inventory.deduct(line.getProductId(), line.getQuantity());
        }

        return SalesOrderView.from(orders.save(order));  // 4. 存 + 转 DTO
    }
}
```

这个方法读起来就是一份流程清单，**没有任何 if/else 业务判断**。
所有「能不能发、发多少」的判断都发生在 `order.ship()` 和 `inventory.deduct()` 内部。

## 它到底负责哪几件事

| 职责 | 说明 |
| --- | --- |
| 事务边界 | `@Transactional` 加在这一层，加在 Controller 会造成事务过长，加在领域层则让领域依赖框架 |
| 用例编排 | 一次用户意图 = 一个方法 |
| 加载与保存 | 通过[仓储](repository.md)取聚合、存聚合 |
| 跨模块协调 | 销售发货要调库存扣减，这类协调发生在这里 |
| 返回视图/DTO | 把领域对象转成对外形状（见 [DTO](dto.md)） |

## 在本项目里怎么用

每个模块的 `application` 包下，**一个用例一个方法**，命名贴合业务动作：

| 模块 | 应用服务方法示例 |
| --- | --- |
| 销售 | `createOrder`、`confirmOrder`、`shipOrder` |
| 库存 | `receiveStock`、`issueStock` |
| 采购 | `createPurchaseOrder`、`receiveGoods` |

不建议写一个包揽全部的大 `OrderService`（里面既有创建又有发货又有查询），
那样它会长成一个几百行的上帝类。按用例切分更清爽。

## 常见误解

- **误解一**：应用服务 = Service 层，所以业务规则也该写这。
  这是[贫血模型](anemic-vs-rich-model.md)的起点，会导致规则散落。
- **误解二**：Controller 直接调仓储更省事。省的是几行代码，丢的是事务边界与用例封装。
- **误解三**：应用服务必须对应一个接口。没必要，只有在需要替换实现或反转依赖时才抽接口。
