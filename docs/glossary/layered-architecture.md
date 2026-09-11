# 分层架构 (Layered Architecture)

## 一句话定义

分层架构是**按职责把代码切成若干水平层**的组织方式，核心纪律只有一条：
**依赖方向单向**，上层可以调用下层，下层绝不反向依赖上层。

## 具体例子

本项目采用四层，一次「确认订单」请求的流向：

```
HTTP 请求
   ↓
① 接口层 api/            OrderController        解析请求、参数校验、调应用服务、返回响应
   ↓
② 应用服务层 application/ OrderAppService       开事务、装配领域对象、调仓储、返回结果
   ↓
③ 领域层 domain/         SalesOrder（含规则）    业务规则、状态流转、不变式 ← 系统的核心
   ↑
④ 基础设施层 infrastructure/ JpaOrderRepository  实现仓储接口、SQL、外部系统调用
```

**注意第 ③ 层和第 ④ 层之间没有向下的箭头。** 领域层只声明
`SalesOrderRepository` **接口**，实现在基础设施层——依赖方向被接口反转了
（见[依赖方向 / 依赖倒置](dependency-direction.md)）。

这样做的收益很直接：领域层可以脱离数据库、脱离 Spring 独立跑单元测试，
因为编译期它根本不认识 JPA。

## 分层不是「项目结构」

新手最常见的误读是把分层当成多模块工程。**分层是职责划分，不是目录数量。**
本项目是单模块 Maven 工程，靠包来体现分层，完全够用：

```
com.example.erp.sales
├── api/             OrderController
├── application/     OrderAppService
├── domain/          SalesOrder, OrderLine, SalesOrderRepository（接口）
└── infrastructure/  JpaSalesOrderRepository
```

不搞多模块的理由：多模块的收益（编译期依赖隔离、独立发布）
在示例项目里体现不出来，代价（构建配置复杂度）却是实打实的。

## 在本项目里怎么用

| 层 | 包名 | 允许依赖 |
| --- | --- | --- |
| 接口层 | `api` | application、domain（只读用） |
| 应用服务层 | `application` | domain |
| 领域层 | `domain` | **无**（只用 JDK，可有极少数注解） |
| 基础设施层 | `infrastructure` | domain（实现其接口）、Spring/JPA |

验收口径：如果 `domain/` 下的代码里出现 `import org.springframework...` 或
`import jakarta.persistence...`，那这层就被污染了，需要回头看设计。

> 务实提示：**本项目允许 JPA 注解出现在领域对象上**（否则要额外写映射层，得不偿失）。
> 但「注解」和「逻辑」要分开看——注解只是标记，逻辑仍必须留在领域层。
> 这是示例项目对教科书做的一处有意妥协，会在 `docs/architecture.md` 里记录。

## 常见误解

- **误解一**：层越多越专业。层数应与复杂度匹配，四层对中小项目已足够；
  硬加「防腐层 + 门面层 + 转换层」只会让一个简单查询穿过五个类。
- **误解二**：Controller 里顺手写业务逻辑「反正小」。一旦开了头，分层就名存实亡，
  规则会同时散在 Controller 和 Service 两处。
- **误解三**：分层 = 每层都要有接口。只有**需要反转依赖**的地方才要接口（仓储、外部网关），
  应用服务直接是具体类就行。
