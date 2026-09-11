# 依赖方向 / 依赖倒置 (Dependency Direction / Inversion)

## 一句话定义

- **依赖方向**：模块之间「谁能 import 谁」的规则。分层架构里方向是单向的。
- **依赖倒置（DIP）**：把「高层依赖低层」改成「两者都依赖抽象」，
  用接口把箭头掉个头。

核心思想：**谁更稳定、谁更核心，就应该由谁来定义接口。**

## 具体例子

**问题**：领域层的 `SalesOrder` 需要把订单存起来，
但存储是「技术细节」，领域层不该认识 JPA。怎么做到既存储又不依赖 JPA？

```java
// ❌ 直连：领域层被迫认识 JPA，无法脱离数据库测试
package com.example.erp.sales.domain;
import org.springframework.data.jpa.repository.JpaRepository;
public interface SalesOrderRepository extends JpaRepository<SalesOrder, Long> { }
```

```java
// ✅ 倒置：接口定义在领域层，实现放在基础设施层
package com.example.erp.sales.domain;              // 领域层的接口
public interface SalesOrderRepository {
    Optional<SalesOrder> findByOrderNo(String orderNo);
    SalesOrder save(SalesOrder order);
}
```

```java
package com.example.erp.sales.infrastructure;      // 基础设施层的实现
public class JpaSalesOrderRepository implements SalesOrderRepository {
    private final SpringDataOrderDao dao;          // 这里才出现 JPA
    ...
}
```

**编译期的依赖箭头因此指向了内部**：

```
改动前：  domain ──依赖──▶ infrastructure(JPA)
改动后：  infrastructure ──依赖──▶ domain（的接口）
                                    ▲
                              领域层不认识实现
```

这就是「倒置」二字的由来——箭头被翻过来了。

## 在本项目里怎么用

需要接口倒置的地方其实很少，认准「**会替换**」或「**领域不该认识**」两种情形：

| 场景 | 接口放哪 | 实现放哪 |
| --- | --- | --- |
| 仓储（会换存储、要能单测） | `domain` | `infrastructure` |
| 外部系统网关（如对接第三方物流/税控） | `domain` 或 `application` | `infrastructure` |

**不需要倒置的地方**（本项目大部分代码）：

- 应用服务：直接是具体类，没人会替换它。
- 领域服务：纯计算，无外部依赖。
- 工具类：直接调用。

> 本项目对存储的**务实让步**：领域对象上允许写 JPA 注解（`@Entity`、`@Id`），
> 以换取「不写映射层」的简洁。因此准确地说，本项目做到了
> **仓储接口层面的依赖倒置**，但领域对象在注解层面仍与 JPA 有弱耦合。
> 这是一处有意识的取舍，会记录在 `docs/architecture.md`。

## 常见误解

- **误解一**：每个类都要有接口。这是「为倒置而倒置」，只会增加跳转成本。
  只有真正需要隔离实现的地方才抽接口。
- **误解二**：依赖倒置就是依赖注入。注入（`@Autowired`）只是**手段**，
  倒置讲的是**接口归属和箭头方向**。用注入却把接口定义在实现侧，并没有倒置。
- **误解三**：分层就是倒置。普通分层是单向向下依赖；倒置是刻意让箭头反向，
  两者是不同的事。
