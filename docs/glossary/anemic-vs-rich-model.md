# 贫血模型 / 充血模型 (Anemic vs Rich Domain Model)

## 一句话定义

- **贫血模型**：对象只有字段和 getter/setter，没有业务行为；业务规则集中写在 Service 里。
- **充血模型**：对象自己带业务行为，规则住在对象内部；Service 只做编排。

「贫血」「充血」比喻的是对象里**有多少业务逻辑的血肉**。

## 具体例子

同一条规则「只有草稿状态的订单能确认」，两种写法：

**贫血写法**——对象是数据袋，规则在 Service：

```java
// SalesOrder.java：只有字段 + getter/setter
public class SalesOrder {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String s) { this.status = s; }
}

// SalesOrderService.java：规则散落在各个方法里
public void confirm(String orderNo) {
    SalesOrder o = repo.findByOrderNo(orderNo).orElseThrow(...);
    if (!"DRAFT".equals(o.getStatus())) {          // ← 规则在这里
        throw new IllegalStateException("只有草稿能确认");
    }
    if (o.getLines().isEmpty()) {                  // ← 和这里
        throw new IllegalStateException("订单必须有序明细行");
    }
    o.setStatus("CONFIRMED");
}
```

**充血写法**——规则是对象的能力，绕不过去：

```java
public class SalesOrder {
    private OrderStatus status;
    private final List<OrderLine> lines = new ArrayList<>();

    public void confirm() {
        if (status != OrderStatus.DRAFT) throw new OrderCannotBeConfirmedException(orderNo, status);
        if (lines.isEmpty()) throw new EmptyOrderException(orderNo);
        this.status = OrderStatus.CONFIRMED;
    }
}

// Service 变成纯粹的编排
public void confirm(String orderNo) {
    SalesOrder o = repo.findByOrderNo(orderNo).orElseThrow(...);
    o.confirm();          // 规则在对象里，Service 不需要知道
    repo.save(o);
}
```

## 为什么倾向充血

贫血模型的问题不在「能不能跑」，而在**规则会散**：
`confirm()`、`batchConfirm()`、`importOrder()` 每个入口都要重复写校验，
漏一处就破防（对照[业务不变式](invariant.md)）。

充血模型把规则收在一处，**新入口天然受保护**。

## 在本项目里怎么用

my-erp 的领域对象默认走充血路线：

- `SalesOrder`：`addLine` / `confirm` / `ship` 自带状态与金额校验。
- `InventoryBalance`：`deduct` / `increase` 自带不为负的校验。
- 状态字段用**枚举**而非裸 `String`，避免 `"DRAFT"` 拼错这种问题。

同时保持务实：**不是每个类都要有行为**。
纯查询用的对象、只做数据传输的 [DTO](dto.md) 贫血是天经地义的。

## 常见误解

- **误解一**：充血模型 = 不要 Service。恰恰相反，应用服务照样需要，
  只是它不再承载业务规则，而专注事务与编排。
- **误解二**：贫血模型是错的。它在**以数据为中心的简单 CRUD** 场景下很高效，
  很多成熟项目就是这么写的。它的问题只在业务规则变复杂之后才显现。
- **误解三**：充血模型意味着用 JPA 会打架。用 JPA 映射充血对象是可行的
  （注意别用 `record`、别让字段被懒加载代理搞乱即可），只是需要一点配置耐心。
