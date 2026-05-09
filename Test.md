
关系模式如下：

```sql
Customer(customer_id, name, city, phone)

Product(product_id, product_name, category, price, stock)

Orders(order_id, customer_id, order_date, total_amount)

OrderItem(order_id, product_id, quantity, unit_price)
```

## 9.

查询购买过订单 `'O001'` 中所有商品的客户编号和姓名。

也就是说，只要某个商品出现在订单 `'O001'` 中，该客户都至少购买过一次该商品。

---

## 10.

向 `Customer` 表插入一条客户记录：

```text
customer_id = 'C100'
name = 'Zhang San'
city = 'Beijing'
phone = '13800000000'
```

---

## 11.

将所有 `'Electronics'` 类商品的价格上调 10%。

---

## 12.

将购买过 `'Xiaomi Phone'` 的客户所在城市改为 `'Key Customer City'`。

---

## 13.

删除从未下过订单的客户。

---

## 14.

创建一个供应商表 `Supplier`，包含以下字段：

```text
supplier_id
supplier_name
city
phone
```

要求：

- `supplier_id` 为主键；
    
- `supplier_name` 不能为空；
    
- `city` 默认值为 `'Unknown'`。
    

---

## 15.

给 `Product` 表添加一列：

```text
supplier_id CHAR(10)
```

并让该字段引用 `Supplier` 表中的 `supplier_id`。

---

## 16.

创建一个视图 `ElectronicsProductView`，包含所有 `'Electronics'` 类商品的：

```text
product_id, product_name, price, stock
```

---

## 17.

判断下面 SQL 是否能正确查询“从未下过订单的客户”。如果不能，请说明原因并改正。

```sql
SELECT C.customer_id, C.name
FROM Customer C
JOIN Orders O ON C.customer_id = O.customer_id
WHERE O.order_id IS NULL;
```

---

## 18.

判断下面 SQL 是否能正确查询“每个客户的总消费金额”。如果不能，请说明原因并改正。

```sql
SELECT C.customer_id, C.name, SUM(OI.quantity * OI.unit_price) AS total_spending
FROM Customer C
JOIN Orders O ON C.customer_id = O.customer_id
JOIN OrderItem OI ON O.order_id = OI.order_id;
```

---

## 19.

判断下面 SQL 是否能正确查询“至少购买过两种不同类别商品的客户”。如果不能，请说明原因并改正。

```sql
SELECT C.customer_id, C.name
FROM Customer C
JOIN Orders O ON C.customer_id = O.customer_id
JOIN OrderItem OI ON O.order_id = OI.order_id
JOIN Product P ON OI.product_id = P.product_id
WHERE COUNT(DISTINCT P.category) >= 2
GROUP BY C.customer_id, C.name;
```

---

## 20.

判断下面 SQL 是否能正确删除所有 `'Electronics'` 类商品。如果不能，请说明原因并改正。

```sql
DROP TABLE Product
WHERE category = 'Electronics';
```

---

你先做 **1～10 题**。做完直接贴答案，我按“考试给分标准”给你批改。