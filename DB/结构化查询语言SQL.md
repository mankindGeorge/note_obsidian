# 嵌套查询

1. 查出库存量 > 所在仓库平均库存量的物资

```sql
SELECT mat_num, mat_name, speci, amount
FROM stock
WHERE amount > (
    SELECT AVG(amount)
    FROM stock AS s
    WHERE s.warehouse = stock.warehouse
)
```

2. 查出被领用过的物资信息（IN 子查询）

```sql
SELECT *
FROM stock
WHERE mat_num IN (
    SELECT mat_num
    FROM out_stock
)
```

3. 查询工程项目为"光澜站光缆抢修"抢修所使用的物资编号和名称

```sql
SELECT mat_num, mat_name
FROM stock
WHERE mat_num IN (
    SELECT mat_num
    FROM out_stock
    WHERE prj_num = (
        SELECT prj_num
        FROM salvaging
        WHERE prj_name = '观澜站光缆抢修'
    )
)
```

4. 查询出库存量超过该仓库物资平均库存量的物资编号、名称、规格及数量
- 确定子查询返回的是单值，则子查询与父查询之间可以用比较运算符连接：

```sql
SELECT mat_num, mat_name, speci, amount
FROM stock s1
WHERE amount > (
    SELECT AVG(amount)
    FROM stock s2
    WHERE s2.warehouse = s1.warehouse
)
```

5. 查询其他仓库中比供电局1#仓库的某一物资库存量少的物资名称、规格和数量
- 带有ANY或ALL谓词的子查询：

```sql
SELECT mat_name, speci, amount
FROM stock
WHERE warehouse <> '供电局1#仓库'
AND amount < ANY (
    SELECT amount
    FROM stock
    WHERE warehouse = '供电局1#仓库'
)
```

- 使用集函数MAX()实现：

```sql
SELECT mat_name, speci, amount
FROM stock
WHERE warehouse <> '供电局1#仓库'
AND amount < (
    SELECT MAX(amount)
    FROM stock
    WHERE warehouse = '供电局1#仓库'
)
```

## EXISTS子查询

**作用：** 判断子查询是否有结果，有结果-->TRUE; 无结果-->FALSE

**特点：**
- 子查询里**一般写 `SELECT *`**，不关心具体列
- 必须有关联条件，否则逻辑不对
- 常用于："存在…的…" 这类查询

**语法结构：**

```sql
SELECT 列
FROM 表1
WHERE EXISTS (
    SELECT *
    FROM 表2
    WHERE 表1.字段 = 表2.字段
)
```

查询所有使用了m001号物资的工程项目名称：

```sql
SELECT prj_name
FROM salvaging
WHERE EXISTS (
    SELECT *
    FROM out_stock
    WHERE out_stock.prj_num = salvaging.prj_num
      AND out_stock.mat_num = 'm001'
)
```

查询被**所有**的抢修工程项目都使用了物资编号及物资名称、规格：
- 不存在这样一个抢修项目：该项目没有使用这个物资 = 这个物资被所有项目都使用了。

```sql
SELECT mat_num, mat_name, speci
FROM stock s
WHERE NOT EXISTS (
    SELECT *
    FROM salvaging p
    WHERE NOT EXISTS (
        SELECT *
        FROM out_stock o
        WHERE o.prj_num = p.prj_num
          AND o.mat_num = s.mat_num
    )
)
```

- 如果是**存在**某个工程是用了这个物资的话，则只要至少一个工程用过就行，即用单层EXISTS就够了：

```sql
SELECT mat_num, mat_name, speci
FROM stock s
WHERE EXISTS (
    SELECT *
    FROM out_stock o
    WHERE o.mat_num = s.mat_num
)
```

---

# 集合查询

**定义：** 把两个或多个SELECT查询的结果集当成集合，进行合并、相交、相减。

**要求：**
- 多个查询的列数必须相同
- 对应列的数据类型要兼容
- 列名可以不同，最终以第一个查询的列名为准

**种类：**
- 并操作：`UNION`（去重）
- 交操作：`INTERSECT`
- 差操作：`EXCEPT`
- 全交集：`UNION ALL`（不去重）

**注：** 参加集合操作的结果集必须是相容的
