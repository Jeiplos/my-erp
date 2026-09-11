# DTO (Data Transfer Object)

## 一句话定义

DTO 是**专门用于跨边界传输数据、只有字段没有业务行为**的对象。
它存在的唯一理由是：决定「对外暴露什么形状的数据」，而不是复用领域对象。

## 具体例子

领域对象和对外接口的形状，需求常常不一样：

```java
// 领域对象：有业务行为，字段按内部需要组织
public class SalesOrder {
    private Long id;                  // 数据库主键，对外无意义
    private String orderNo;
    private OrderStatus status;       // 枚举
    private List<OrderLine> lines;
    private BigDecimal totalAmount;
    // + addLine() / confirm() / ship() 等业务方法
}
```

```java
// 响应 DTO：只描述"给调用方看什么"
public record OrderResponse(
        String orderNo,
        String status,                // 枚举转成字符串，避免调用方依赖我方枚举
        List<LineResponse> lines,
        String totalAmount            // 金额转成字符串/分，避免 JSON 浮点精度问题
) {
    public static OrderResponse from(SalesOrder order) {   // 转换集中在一处
        return new OrderResponse(
                order.getOrderNo(),
                order.getStatus().name(),
                order.getLines().stream().map(LineResponse::from).toList(),
                order.getTotalAmount().toPlainString());
    }
}
```

注意最后两行的细节——`status` 和 `totalAmount` 都做了**有意转换**。
这正是 DTO 的价值：领域内部怎么表达是一回事，对外怎么说清楚是另一回事。

## 直接返回领域对象会怎样

```java
// ❌ Controller 直接返回实体
@GetMapping("/orders/{no}")
public SalesOrder get(@PathVariable String no) { return appService.find(no); }
```

后果：

1. **内部结构泄露**：`id` 等内部字段自动暴露，改字段就是破兼容性。
2. **序列化踩坑**：JPA 懒加载集合触发 `LazyInitializationException`；
   双向关联（订单 ↔ 明细）直接 JSON 循环引用爆栈。
3. **业务方法暴露**：接口形状与领域模型被绑死，领域改不动。
4. **过度暴露**：把成本价之类的敏感字段顺手带出去。

## 在本项目里怎么用

| 场景 | 做法 |
| --- | --- |
| 请求体 | `record CreateOrderRequest(...)` + Bean Validation 注解 |
| 响应体 | `record OrderResponse(...)`，带静态 `from(领域对象)` 方法 |
| 层级 | 放在 `api` 包下（或 `api/dto`），**不要**放进 `domain` |
| 转换位置 | 在[应用服务](application-service.md)或 DTO 自身的 `from` 里，集中一处 |

用 `record` 写 DTO 最合适：天然不可变、没有 getter 噪音、代码量小。

## 常见误解

- **误解一**：DTO 是过时/多余的东西，直接返回实体更快。
  小项目里确实能跑，但代价是领域模型失去演进自由——这与本项目「自己实现 MVP 再迭代」的目标直接冲突。
- **误解二**：一个 DTO 打天下。所有接口共用一个万能 DTO，最后字段全是可空，
  谁也说不清哪个接口需要哪些字段。按用例设计 DTO 更清晰。
- **误解三**：DTO 必须有 getter/setter。它是**数据载体**，
  用 `record` 或不可变类更好，不需要可写。
