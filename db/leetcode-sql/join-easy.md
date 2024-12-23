---
description: 조인 관련 문제 풀어보기
---

# join - easy

### 1.  Replace Employee Id with The Unique ID

```
Employees table:
+----+----------+
| id | name     |
+----+----------+
| 1  | Alice    |
| 7  | Bob      |
| 11 | Meir     |
| 90 | Winston  |
| 3  | Jonathan |
+----+----------+
EmployeeUNI table:
+----+-----------+
| id | unique_id |
+----+-----------+
| 3  | 1         |
| 11 | 2         |
| 90 | 3         |
+----+-----------+
Output: 
+-----------+----------+
| unique_id | name     |
+-----------+----------+
| null      | Alice    |
| null      | Bob      |
| 2         | Meir     |
| 3         | Winston  |
| 1         | Jonathan |
+-----------+----------+
```

* Employee 테이블의 PK대신, 따로 정의한 UID로 변경하여 반환해야한다. 제한사항으로는 UID가 정의되지 않았다면 NULL을 반환하는 것이다. 교집합 + LEFT이므로 LEFT JOIN을 사용한다.

````
```mysql
select e_u.unique_id, e.name
    from Employees e
    left join EmployeeUNI e_u
    on e.id = e_u.id;
```
````



### 2.[Product Sales Analysis I](https://leetcode.com/problems/product-sales-analysis-i/)&#x20;



```
Sales table:
+---------+------------+------+----------+-------+
| sale_id | product_id | year | quantity | price |
+---------+------------+------+----------+-------+ 
| 1       | 100        | 2008 | 10       | 5000  |
| 2       | 100        | 2009 | 12       | 5000  |
| 7       | 200        | 2011 | 15       | 9000  |
+---------+------------+------+----------+-------+
Product table:
+------------+--------------+
| product_id | product_name |
+------------+--------------+
| 100        | Nokia        |
| 200        | Apple        |
| 300        | Samsung      |
+------------+--------------+
```

````
```mysql
select p.product_name, s.year, s.price
    from Product p
    inner join Sales s
    on s.product_id = p.product_id;
```
````



