# Myssql分页查询

测试数据 20万条, 表结构

#### 传统直接方式

SELECT * FROM page_test LIMIT 10000, 10

> OK
> Time: 0.122s

SELECT * FROM page_test LIMIT 100000, 10

> OK
> Time: 0.033s

SELECT * FROM page_test LIMIT 190000, 10

> OK
> Time: 0.051s

#### 以上一次id为起始方式

SELECT * FROM page_test WHERE id > 10000 LIMIT 10
> OK
> Time: 0.008s

SELECT * FROM page_test WHERE id > 100000 LIMIT 10
> OK
> Time: 0.008s

SELECT * FROM page_test WHERE id > 190000 LIMIT 10
> OK
> Time: 0.008s

#### id起始后再排序（上一方法的修正）

SELECT * FROM page_test WHERE id > 10000  ORDER BY id ASC LIMIT 10
> OK
> Time: 0.008s

SELECT * FROM page_test WHERE id > 100000  ORDER BY id ASC LIMIT 10
> OK
> Time: 0.008s

SELECT * FROM page_test WHERE id > 190000  ORDER BY id ASC LIMIT 10
> OK
> Time: 0.008s

这里时间较第一次没有变化，主要使我们插入后的ID本身是有序的，所以排序时间可以忽略

#### 使用子查询

SELECT * FROM page_test WHERE id > (SELECT id FROM page_test LIMIT 10000, 1)  LIMIT 10
> OK
> Time: 0.01s

SELECT * FROM page_test WHERE id > (SELECT id FROM page_test LIMIT 100000, 1)  LIMIT 10
> OK
> Time: 0.027s

SELECT * FROM page_test WHERE id > (SELECT id FROM page_test LIMIT 199000, 1)  LIMIT 10
> OK
> Time: 0.044s

