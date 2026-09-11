# 分层职责 (Layer Responsibilities)

## 一句话定义

分层职责是每一层「**该放什么、不该放什么**」的约定。
[分层架构](layered-architecture.md)解决「有几层」，分层职责解决「东西放哪一层」。

判断口诀：**接口层管翻译，应用层管流程，领域层管规则，基础设施层管技术。**

## 具体例子

还是「确认订单」这件事，拆到四层各写什么：

| 层 | 该写 | 不该写 |
| --- | --- | --- |
| 接口层 `api` | `@PostMapping`、读 path/body、把异常翻译成 HTTP 状态码、组装响应 | 业务判断（`if (status == DRAFT)`）、直接调仓储 |
| 应用服务层 `application` | `@Transactional`、加载聚合、调用领域方法、保存、发领域事件、DTO 转换 | 业务规则本身、SQL、HTTP 相关的东西 |
| 领域层 `domain` | 状态流转、金额计算、不变式校验、领域异常 | `@Transactional`、`@Autowired`、`HttpServletRequest`、JSON 注解 |
| 基础设施层 `infrastructure` | JPA 仓储实现、数据库脚本、调用外部系统、缓存 | 业务规则、流程编排 |

写成代码就是：

```java
// ① 接口层：只管翻译
@PostMapping("/orders/{orderNo}/confirm")
public OrderResponse confirm(@PathVariable String orderNo) {
    return OrderResponse.from(appService.confirm(orderNo));   // 不写任何业务判断
}

// ② 应用层：只管编排
@Transactional
public SalesOrder confirm(String orderNo) {
    SalesOrder order = orders.findByOrderNo(orderNo).orElseThrow(OrderNotFoundException::new);
    order.confirm();          // 规则交还给领域层
    return orders.save(order);
}

// ③ 领域层：只管规则
public void confirm() { /* 状态与明细校验，见贫血/充血词条 */ }
```

**每一层都很薄，且只做自己的事**——这是分层能带来可维护性的前提。
只要有一层开始「顺手帮别人干活」，收益就会迅速消失。

## 在本项目里怎么用

落到验收清单上，每一步实现完都按这张表自查：

- [ ] Controller 里没有 if/else 业务判断，行数通常在 10 行以内。
- [ ] 应用服务里没有 SQL、没有 HTTP 类型。
- [ ] 领域层不 import Spring / JPA 的**逻辑类**（注解例外，见分层架构的务实提示）。
- [ ] 仓储接口在领域层，JPA 实现在基础设施层。

## 常见误解

- **误解一**：DTO 转换放哪层无所谓。放应用层最合适——
  接口层做会让领域对象泄到 Controller，领域层做则等于让领域认识传输格式。
- **误解二**：应用服务只写一行 `repository.save()` 就是多余的一层。
  它承担事务边界和用例编排，这两个职责无法由实体承担；层薄不等于层多余。
- **误解三**：校验都在接口层做完最省事。输入格式校验放接口层没问题，
  但业务不变式必须留在领域层，否则绕过 HTTP 入口（定时任务、批处理）时就失守了。
