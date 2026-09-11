# 值对象 (Value Object)

## 一句话定义

值对象是**没有唯一标识**的对象：两个值对象的属性全部相同，就认为它们相等，
并且它一旦创建就**不应该被修改**（不可变）。见[实体](entity.md)的对比。

## 具体例子

```java
// 值对象：用 record 天然不可变
public record Money(BigDecimal amount, String currency) {
    public Money {
        if (amount == null || currency == null) throw new IllegalArgumentException("金额和币种必填");
        if (amount.scale() > 2) throw new IllegalArgumentException("金额最多两位小数");
    }

    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("币种不一致");
        return new Money(amount.add(other.amount), currency);   // 返回新对象，不改自己
    }
}
```

`Money(100, "CNY")` 和另一个 `Money(100, "CNY")` 是相等的，不需要 id 来区分。

值对象最大的价值是**把校验规则装进类型里**：只要你能拿到一个 `Money` 实例，
就说明它一定是合法的、两位小数以内的。调用方不用每次都检查。

## 在本项目里怎么用

适合做成值对象的候选：

| 概念 | 为什么适合 |
| --- | --- |
| 金额 Money | 携带币种和精度规则 |
| 数量 Quantity | 不能为负、精度限制 |
| 商品编码 Sku | 有格式规则，避免到处传裸 `String` |
| 地址 Address | 一组字段总是一起出现、一起使用 |
| 单据状态（草稿/已确认/已发货） | 用枚举表达，天然是值 |

特别推荐把**金额**做成值对象：ERP 里到处是金额计算，
用 `BigDecimal` 裸传迟早出现精度和币种问题。

## 常见误解

- **误解一**：值对象必须是 Java 的 `record`。不是必须，`record` 只是最省事的写法；
  普通类只要保证不可变、按值比较也行。
- **误解二**：值对象不能有方法。可以有，而且应该有一一它上面恰恰适合放纯计算
  （如 `plus`、`minus`、`isNegative`），只要方法不修改自身状态。
- **误解三**：所有字段都该做成值对象。过度封装会带来大量拆装代码，
  只在**有规则要守**或**总是一起出现**时才值得。
