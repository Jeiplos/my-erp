# 仓储 (Repository)

## 一句话定义

仓储是**用「集合」的语义来存取聚合的接口**，它把「数据存在哪、怎么写 SQL」
这些细节挡在业务代码之外。

读法上把它想成一个内存里的 `Map`：往里 `save` 一个对象，按 id `findById` 取回来。

## 具体例子

```java
// 接口定义在领域侧，只谈业务语义，不提数据库
public interface SalesOrderRepository {
    Optional<SalesOrder> findByOrderNo(String orderNo);
    List<SalesOrder> findByStatus(OrderStatus status);
    SalesOrder save(SalesOrder order);
}
```

业务代码拿到的就是一个接口对象，用起来像集合：

```java
SalesOrder order = orders.findByOrderNo(no).orElseThrow(OrderNotFoundException::new);
order.ship();          // 纯业务逻辑，不知道有数据库这回事
orders.save(order);
```

关键点：**接口属于领域，实现在基础设施层**。这样领域代码不依赖 JPA/SQL，
换存储、写单元测试（换成内存实现）都很轻松——这也是[依赖倒置](dependency-direction.md)的一个实例。

## 和 DAO 的区别

| | 仓储 Repository | DAO |
| --- | --- | --- |
| 面向 | 领域概念、聚合 | 数据库表 |
| 粒度 | 一次一个聚合（含内部对象） | 一次一张表 / 一行 |
| 语义 | 集合（save/find/remove） | CRUD（insert/update/select） |
| 归属 | 领域侧的接口 | 基础设施侧的实现细节 |

粗略地说：**Repository 处理「对象」，DAO 处理「行」。**

## 在本项目里怎么用

每个聚合根配一个仓储接口，例如 `SalesOrderRepository`、`ProductRepository`、
`InventoryBalanceRepository`。实现初期直接用 Spring Data JPA 的接口即可
（`JpaRepository` 天然就是这个形状），**不额外造一层 DAO**，避免样例项目里堆无谓的分层。

## 常见误解

- **误解一**：Repository 就是 DAO 换个名字。区别在抽象层次和粒度，见上表。
- **误解二**：Repository 应该能写任意复杂查询。复杂报表查询不适合塞进仓储
  （会污染领域接口），本项目把报表查询单独放在 `reporting` 模块。
- **误解三**：一个聚合里的每个实体都要有仓储。只有**聚合根**才配仓储。
