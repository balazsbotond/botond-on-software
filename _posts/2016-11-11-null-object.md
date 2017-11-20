---
layout: post
title:  "Dealing with Nothing in C# - Null Objects"
date:   2016-11-11 16:08:40 +0100
categories: oop c#
---

The `null` reference is so ubiquitous, such an integral part of our experience as programmers, that it would be hard to imagine the world without it. Yet it is the source of so much frustration and actual financial loss that it is well worth the effort to think about some alternatives.

## Null Objects

But what exactly is the problem?

Let's imagine a system where customers can place orders. The orders contain line items that have a reference to a product and they store the quantity ordered. They can also calculate the price based on the unit price and the quantity specified.

```csharp
public class LineItem
{
    public Product Product { get; }

    public decimal Quantity { get; }

    public decimal Price => Product.UnitPrice * Quantity;

    public LineItem(Product product, decimal quantity)
    {
        if (quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(quantity));

        if (product == null)
           throw new ArgumentNullException(nameof(product));

        this.Product = product;
        this.Quantity = quantity;
    }
}
```

A single discount of various types can be applied to an order so we use the *Strategy* pattern to decouple orders from the calculation of discounts.


```csharp
public interface IDiscount
{
    decimal Calculate(decimal orderTotal);
}

public class CouponDiscount : IDiscount
{
    public CouponDiscount(string couponCode, decimal rate) { /* ... */ }

    public decimal Calculate(decimal orderTotal) => orderTotal * rate;
}

public class FrequentBuyerDiscount : IDiscount { /* ... */ }
```

Given the order total, a discount object can calculate the reduced price according to its own set of rules.

The order class provides methods to add line items and a discount, and a property that returns the total after applying the discount.

```csharp
public class Order
{
    public IEnumerable<LineItem> LineItems => lineItems.AsReadOnly();

    public decimal Total => discount.Calculate(totalBeforeDiscount);
        private decimal totalBeforeDiscount => lineItems.Sum(i => i.Price);

    private IDiscount discount;// initialized to null

    private List<LineItem> lineItems;

    public Order()
    {
        lineItems = new List<LineItem>();
    }

    public void AddLineItem(LineItem lineItem)
    {
        if (lineItem == null) throw new ArgumentNullException(nameof(lineItem));

        lineItems.Add(lineItem);
    }

    public void AddDiscount(IDiscount discount)
    {
        if (discount == null) throw new ArgumentNullException(nameof(discount));

        this.discount = discount;
    }
}
```

The class above seems completely innocent. What can go wrong? We even added null checks for all the arguments. But it is still very likely that it will throw a `NullReferenceException`. Can you spot the bug?

```csharp
var order = new Order();

order.AddLineItem(new LineItem(product1, 2));

order.AddLineItem(new LineItem(product2, 1));

Console.WriteLine($"Total: {order.Total}"); // BOOM!!!
```

Oh, we forgot to add a discount!

But wait. Creating an order without a discount is a completely valid requirement. Let's fix the code to support that.

```csharp
public decimal Total =>
    discount == null ? totalBeforeDiscount : discount.Calculate(totalBeforeDiscount);
```

This code works, but it sure isn't beautiful. Even if we put aesthetic concerns aside, having to check for null every time we access the discount field is a recipe for disaster. Forget it even once, and a `NullReferenceException` appears. Also, we need to duplicate the default logic, returning the total before discount, every time. That logic isn't well encapsulated.

Fortunately, we can do better. Let's look at how we dealt with the exact same problem when adding the line items. We initialized the `lineItems` field to a safe default value, the empty list, in the constructor.

```csharp
public Order()
{
    lineItems = new List<LineItem>();
}
```

Having done that, there is no need to check whether lineItems is null in the AddLineItem method.

```csharp
lineItems.Add(lineItem);
```

Can we imagine such a safe default value for discount too? Of course we can, that would be a `NoDiscount` class where the `Calculate` method simply returns the `orderTotal` argument without modification.

```csharp
public class NoDiscount : IDiscount
{
    public decimal Calculate(decimal orderTotal) => orderTotal;
}
```

Such a default object is called a *Null Object*. Now we can use that as the initial value for discount in the constructor,

```csharp
public Order()
{
    lineItems = new List<LineItem>();

    discount = new NoDiscount();
}
```

and get rid of the problematic null check in the `Total` property.

```csharp
public decimal Total => discount.Calculate(totalBeforeDiscount);
```

As a final piece of advice, it is recommended to exercise restraint when applying the Null Object pattern. It is a very elegant solution when the default behavior is harmless and predictable. But if the behavior of the Null Object is not obvious to the consumer, getting a NullReferenceException is actually more useful than having a badly-designed Null Object behave unexpectedly.
