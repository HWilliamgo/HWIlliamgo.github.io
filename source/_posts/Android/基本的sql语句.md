我没学过数据库，但是作为一个程序员得了解一些基本的语句，以下为来自网站https://www.codecademy.com/learn/learn-sql的学习笔记

数据库4中数据类型：INTEGER(整型), TEXT(字符串类型),DATA(日期),REAL(浮点型)。


查询整张表：
``` sql
SELECT * FROM <表名>;
```

创建表：
``` sql
CREATE TABLE table_name (
   (列名) (数据类型), 
   column_2 data_type <限制符> <限制符> ..., 
   column_3 data_type
);
//**
限制符是在创建表的时候规定的某一列的信息。有如下几类：
1.PRIMARY KEY：表示这是主键，一张表只能有一个主键，当插入的row的主键在表中已经存在时，将无法将这条数据插入。
2.UNIQUE:这一列的每一行的值都将不同，有点类似主键，唯一区别是一张表可以有多个UNIQUE列。
3.NOT NULL:这一列必须有值，否则无法插入
4.DEFAULT <default_value>:如果新的row没有给这一列赋值，当插入之后这一列会赋上一个默认值<defalut_value>。

**//
```

插入一行数据：
``` sql
INSERT INTO <table_name> (column1,column2,column3...) VALUES (value1,value2,value3...);
```

查询：
``` sql
//查询表中的某一列数据：
SELECT <column_name> FROM <table_name>;

//查询表中的多列数据：
SELECT <column_1>,<column_2>... FROM <table_name>;

//查询表中的某列数据并以化名作为列名返回（table中的列名不会更改）
SELECT <某一列名> AS '化名' FROM <table_name>;

// DISTINCT关键字：过滤掉重复的值。
SELECT DISTINCT <column_name> FROM <table_name>;

//Where条件查询，条件语句的操作符包括：=, ！=, >, <, >=, <=
SELECT <column_name> FROM <table_name> WHERE <条件语句>

//LIKE
//从表movies中选出name中"_"部分可以为任意一个字符的值。"_"是通配符
SELECT * FROM movies WHERE name LIKE 'Se_en';
```

更新：
``` sql
//在column_y=b的地方将column_x的值更新为a。
//比如：UPDATE celebs SET age = 22 WHERE id = 1;
UPDATE <table_name> SET <column_x>=a WHERE <column_y>=b;
```

为表添加新的列：
``` sql
ALTER TABLE <table_name> ADD COLUMN <column_name> <type>;
```

删除选中的行：
``` sql
//删除是所有column_name是Null的行。
DELETE FROM <table_name> WHERE <column_name> IS NULL;
```
