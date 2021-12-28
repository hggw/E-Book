# 一、检索数据

## 1.1 限制结果

SELECT语句返回所有匹配的行，它们可能是指定表中的每个行。返回第一行或前几行，可使用LIMIT子句。  

```Mysql
select prod_name from products limit 5;
```

可指定要检索的开始行和行数

```Mysql
select prod_name from products limit 5,5;
```

LIMIT 5, 5指示MySQL返回从行5开始的5行，也就是第6，7，8，9，10行。第一个数为开始位置，第二个数为要检索的行数。  

LIMIT中指定要检索的行数为检索的最大行数。如果没有足够的行（例如，给出LIMIT 10, 5，但只有13行）， MySQL将只返回它能返回的那么多行。  

