---
title: Leftmost Prefix Principle In MySQL Indexing
toc: false
date: 2017-10-24 20:25:57
tags:
- MySQL
categories:
- Database
---

Suppose that there is such a table:

``` mysq
CREATE TABLE `student` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `cid` int(11) DEFAULT NULL,
  `age` int(11),
  PRIMARY KEY (`id`),
  KEY `name_cid_INX` (`name`,`cid`,`age`),
) ENGINE=InnoDB AUTO_INCREMENT=8 DEFAULT CHARSET=utf8
```

`id` is the primary key and `(name, cid, age)` is a multi-column index.

Now consider the 5 queries below:

1. `SELECT * FROM student WHERE cid=1 AND name>'bob';`
2. `SELECT * FROM student WHERE name='bob' AND cid=1;`
3. `SELECT * FROM student WHERE name='bob' AND cid=1 AND age=22;`
4. `SELECT * FROM student WHERE name='bob' AND cid>1 AND age=22;`
5. `SELECT * FROM student WHERE cid=1 AND name='bob';`

Which of them do you think can use the index?

The answer is  2, 3, 5.

<!-- more -->

That's because of the leftmost index prefixes principle in compositic (multiple-column) index. 

In short, because the underlying index implementation structure of MySQL is B+ Tree, the compositic index must be indexed by the column order. The primary index `name` makes records for this column in order, but the secondary indexes `cid` and `age` can't. The secondary indexes can only make their columns in partial order. In other words, under the condition that its left prefix columns are all equal, the column for this secondary index is in order.

Here below are the primary index and secondary index of MySQL InnoDB engine:

![B+ Tree Primary Index](/images/MySQL/B+ Tree_1.png)

![B+ Tree Secondary Index](/images/MySQL/B+ Tree_2.png)

<br/>
