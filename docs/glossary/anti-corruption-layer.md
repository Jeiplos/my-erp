# 防腐层 (Anti-Corruption Layer, ACL)

## 一句话定义

防腐层是**夹在自己领域与外部系统之间的一层转换**，
把外部系统的模型和术语翻译成自己的语言，防止外部概念污染自己的领域。

一句话记法：**外部怎么乱是外部的事，进我的门就得按我的规矩说话。**

## 具体例子

假设要对接到第三方物流系统下单。对方的接口长这样（字段名、状态值都是它的）：

```json
{ "waybill_no": "SF123", "stat": "3", "consignee_info": { "tel": "138...", "addr": "..." } }
```

**不加防腐层的写法**（污染直接发生）：

```java
// ❌ 领域层里出现了对方的字段名和魔法状态码
if ("3".equals(shipment.getStat())) { order.markShipped(); }
```

一旦对方把 `stat` 改成枚举、或把 `waybill_no` 改名，你的领域代码就得跟着改。

**加防腐层的写法**：

```java
// 领域侧定义自己的接口和模型，用自己习惯的术语
public interface ShipmentGateway {
    ShipmentResult createShipment(SalesOrder order);   // 我的语言
}

// 基础设施层的实现里做翻译（防腐层就在这里）
public class SfExpressGateway implements ShipmentGateway {
    public ShipmentResult createShipment(SalesOrder order) {
        SfRequest req = SfRequest.from(order);          // 我的模型 → 对方格式
        SfResponse resp = client.post(req);             // 对方返回
        return new ShipmentResult(resp.waybill_no, mapStatus(resp.stat));  // 对方格式 → 我的模型
    }
    private ShipmentStatus mapStatus(String stat) { /* "3" → ShipmentStatus.DELIVERED */ }
}
```

收益：对方改接口时，**只改这一个类**；领域层毫不知情，测试也不受牵连。

## 在本项目里怎么用

MVP 阶段**没有外部系统**，所以防腐层暂时不需要——这很正常，不要为了架构完整性提前造它。

它会在以下迭代节点真正派上用场：

| 迭代动作 | 需要防腐层吗 | 理由 |
| --- | --- | --- |
| MVP（本机 H2，无外部依赖） | 不需要 | 没有外部模型要隔离 |
| 换成 MySQL | 不需要 | JDBC/JPA 自己就是抽象 |
| 对接物流/支付/税控 | **需要** | 对方模型会渗入领域 |
| 对接遗留 ERP 的老接口 | **需要** | 遗留系统的术语通常与新模式冲突 |

届时的落点：`infrastructure` 包里放 `XxxGatewayImpl`，接口定义在 `domain` 或 `application`。

## 常见误解

- **误解一**：防腐层就是再包一层工具类。区别在于 ACL 承担**语义翻译**
  （模型映射、状态码翻译、异常转换），而不只是转发调用。
- **误解二**：所有外部调用都要 ACL。只有**对方模型会渗入你的领域**时才需要。
  发个短信通知、拉个配置，包一层没多大意义。
- **误解三**：ACL 要提前设计好。它应由**真实的外部耦合**驱动，
  提前造只会得到一堆空壳转换代码。
