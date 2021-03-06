---
title: SQL优化
description: SQL优化
date: 2018-12-02 20:21:43
keywords: mysql
categories : [mysql]
tags : [mysql, DB]
comments: true
---

- 字段类型转换导致不用索引，如字符串类型的不用引号，数字类型的用引号等，这有可能会用不到索引导致全表扫描；
- mysql 不支持函数转换，所以字段前面不能加函数，否则这将用不到索引；
- 不要在字段前面加减运算；
- 字符串比较长的可以考虑索引一部份减少索引文件大小，提高写入效率；
- like % 在前面用不到索引；
- 根据联合索引的第二个及以后的字段单独查询用不到索引；
- 不要使用 select *；
- 排序请尽量使用升序 ;
- or 的查询尽量用 union 代替 （Innodb）；
- 复合索引高选择性的字段排在前面；
- order by / group by 字段包括在索引当中减少排序，效率会更高。