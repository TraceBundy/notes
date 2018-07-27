##MySQL全文索引
###概述
* 全文索引是搜索引擎中的核心技术之一。首先需要通过分词进行各词的提取。
* 支持在VARCHAR，CHAR，TEXT等类型上创建全文索引
* MySQL 5.6版本之前仅MyISAM支持全文索引
* MySQL 5.6版本Innodb支持全文索引
* MySQL 5.7版本支持中文、日文的全文索引（真正生产环境可用）
* 一张表只能有一个全文索引
* 添加全文索引时，表是只读的，不可写入与更新

###使用
```
CREATE TABLE articles (
          id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
          title VARCHAR(200),
          body TEXT,
        ) ENGINE=InnoDB;
```

在title,body列上创建全文索引
```
ALTER TABLE articles ADD FULLTEXT INDEX IDX_ARTICLE(title, body) 
```

数据
```
(root@localhost) [test]>select * from articles;
+----+-----------------------+------------------------------------------+
| id | title                 | body                                     |
+----+-----------------------+------------------------------------------+
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...             |
|  2 | How To Use MySQL Well | After you went through a ...             |
|  3 | Optimizing MySQL      | In this tutorial we will show ...        |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ...      |
|  5 | MySQL vs. YourSQL     | In the following database comparison ... |
|  6 | MySQL Security        | When configured properly, MySQL ...      |
+----+-----------------------+------------------------------------------+
6 rows in set (0.01 sec)
```
全文索引有几种方式
```
* IN NATURAL LANGUAGE MODE
* IN BOOLEAN MODE
* WITH QUERY EXPANSION
```
* IN NATURAL LANGUAGE MODE

```
查询文章中包含database的文章

(root@localhost) [test]>SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('database' IN NATURAL LANGUAGE MODE);
+----+-------------------+------------------------------------------+
| id | title             | body                                     |
+----+-------------------+------------------------------------------+
|  1 | MySQL Tutorial    | DBMS stands for DataBase ...             |
|  5 | MySQL vs. YourSQL | In the following database comparison ... |
+----+-------------------+------------------------------------------+
2 rows in set (0.00 sec)
```
```
查找文章，并计算相似度
(root@localhost) [test]>SELECT id, body, MATCH (title,body) AGAINST
    ->     ('Security implications of running MySQL as root'
    ->     IN NATURAL LANGUAGE MODE) AS score
    ->     FROM articles WHERE MATCH (title,body) AGAINST
    ->     ('Security implications of running MySQL as root'
    ->     IN NATURAL LANGUAGE MODE);
+----+------------------------------------------+----------------------------+
| id | body                                     | score                      |
+----+------------------------------------------+----------------------------+
|  4 | 1. Never run mysqld as root. 2. ...      |         0.6055193543434143 |
|  6 | When configured properly, MySQL ...      |         0.6055193543434143 |
|  1 | DBMS stands for DataBase ...             | 0.000000001885928302414186 |
|  2 | After you went through a ...             | 0.000000001885928302414186 |
|  3 | In this tutorial we will show ...        | 0.000000001885928302414186 |
|  5 | In the following database comparison ... | 0.000000001885928302414186 |
+----+------------------------------------------+----------------------------+
6 rows in set (0.00 sec)
```
* IN BOOLEAN MODE


```
包含MySQL不包含YourSQL

(root@localhost) [test]>SELECT * FROM articles WHERE MATCH (title,body)
    ->     AGAINST ('+MySQL -YourSQL' IN BOOLEAN MODE);
+----+-----------------------+-------------------------------------+
| id | title                 | body                                |
+----+-----------------------+-------------------------------------+
|  6 | MySQL Security        | When configured properly, MySQL ... |
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...        |
|  2 | How To Use MySQL Well | After you went through a ...        |
|  3 | Optimizing MySQL      | In this tutorial we will show ...   |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ... |
+----+-----------------------+-------------------------------------+
5 rows in set (0.00 sec)
```
* WITH QUERY EXPANSION

```
查询与database相关的所有数据
(root@localhost) [test]>SELECT * FROM articles
    ->     WHERE MATCH (title,body)
    ->     AGAINST ('database' WITH QUERY EXPANSION);
+----+-----------------------+------------------------------------------+
| id | title                 | body                                     |
+----+-----------------------+------------------------------------------+
|  5 | MySQL vs. YourSQL     | In the following database comparison ... |
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...             |
|  3 | Optimizing MySQL      | In this tutorial we will show ...        |
|  6 | MySQL Security        | When configured properly, MySQL ...      |
|  2 | How To Use MySQL Well | After you went through a ...             |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ...      |
+----+-----------------------+------------------------------------------+
6 rows in set (0.00 sec)
```




