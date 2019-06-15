---
title: InnoDB锁
description: InnoDB锁
date: 2018-11-27 00:00:41
keywords: mysql
categories : [mysql]
tags : [mysql, DB]
comments: true
---

# 事务

## ACID

事务是由一组SQL语句组成的逻辑处理单元，事务具有以下4个属性，通常简称为事务的ACID属性。

- 原子性（Atomicity）：事务是一个原子操作单元，其对数据的修改，要么全都执行，要么全都不执行
- 一致性（Consistent）：在事务开始和完成时，数据都必须保持一致状态。这意味着所有相关的数据规则都必须应用于事务的修改，以保持数据的完整性；事务结束时，所有的内部数据结构（如B树索引或双向链表）也都必须是正确的
- 隔离性（Isolation）：数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的“独立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见的，反之亦然
- 持久性（Durable）：事务完成之后，它对于数据的修改是永久性的，即使出现系统故障也能够保持

## 事务并发带来的问题

- 更新丢失（Lost Update）：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题－－最后的更 新覆盖了由其他事务所做的更新。例如，两个编辑人员制作了同一文档的电子副本。每个编辑人员独立地更改其副本，然后保存更改后的副本，这样就覆盖了原始文 档。最后保存其更改副本的编辑人员覆盖另一个编辑人员所做的更改。如果在一个编辑人员完成并提交事务之前，另一个编辑人员不能访问同一文件，则可避免此问题
- 脏读（Dirty Reads）：一个事务正在对一条记录做修改，在这个事务完成并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加 控制，第二个事务读取了这些“脏”数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做"脏读"
- 不可重复读（Non-Repeatable Reads）：一个事务在读取某些数据后的某个时间，再次读取以前读过的数据，却发现其读出的数据已经发生了改变、或某些记录已经被删除了！这种现象就叫做“不可重复读”
- 幻读（Phantom Reads）：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为“幻读”

## 事务隔离级别

防止更新丢失，并不能单靠数据库事务控制器来解决，需要应用程序对要更新的数据加必要的锁来解决，因此，防止更新丢失应该是应用的责任。“脏读”、“不可重复读”和“幻读”，其实都是数据库读一致性问题，必须由数据库提供一定的事务隔离机制来解决。

数据库实现事务隔离的方式，基本上可分为以下两种

- 在读取数据前，对其加锁，阻止其他事务对数据进行修改。
- 不用加任何锁，通过一定机制生成一个数据请求时间点的一致性数据快照（Snapshot)，并用这个快照来提供一定级别（语句级或事务级）的一致性读取。从用户的角度来看，好像是数据库可以提供同一数据的多个版本，因此，这种技术叫做数据多版本并发控制（MultiVersion Concurrency Control，简称MVCC或MCC），也经常称为多版本数据库

4种隔离级别比较

| | 读数据一致性 | 脏读 | 不可重复读 | 幻读 |
| :-: | :-: | :-: | :-: | :-: |
| 未提交读（Read uncommitted） | 最低级别，只能保证不读取物理上损坏的数据 | 是 | 是| 是|
| 已提交度（Read committed） | 语句级 | 否 | 是 | 是 |
| 可重复读（Repeatable read | 事务级 | 否 | 否 | 是 |
| 序列化（Serializable）| 最高级别，事务级 | 否 | 否 | 否 |

各具体数据库并不一定完全实现了上述4个隔离级别，MySQL 支持全部4个隔离级别，但在具体实现时，有一些特点，比如在一些隔离级别下是采用MVCC一致性读，但某些情况下又不是。

# 锁

## 锁的类型

按锁的粒度划分

- 行级锁：行级锁分为共享锁和排它锁。行级锁是Mysql中锁定粒度最细的锁。InnoDB引擎支持行级锁和表级锁，只有在通过索引条件检索数据的时候，才使用行级锁，否就使用表级锁。行级锁开销大，加锁慢，锁定粒度最小，发生锁冲突概率最低，并发度最高
- 表级锁：表级锁分为表共享锁和表独占锁。表级锁开销小，加锁快，锁定粒度大、发生锁冲突最高，并发度最低
- 页级锁：页级锁是MySQL中锁定粒度介于行级锁和表级锁中间的一种锁。表级锁速度快，但冲突多，行级冲突少，但速度慢。所以取了折衷的页级，一次锁定相邻的一组记录。BDB支持页级锁。 开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般
- 间隙锁（Next-Key）：对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓 的间隙锁（Next-Key锁）

按锁级别划分

InnoDB实现了以下两种类型的行锁

- 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
- 排他锁（X)：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

另外，为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁。

- 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
- 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。

InnoDB行锁模式兼容性列表

| | X | IX | S | IS |
| :-: | :-: | :-: | :-: | :-: |
| X | 冲突 | 冲突 | 冲突| 冲突|
| IX | 冲突 | 兼容 | 冲突 | 兼容 |
| S | 冲突 | 冲突 | 兼容 | 兼容 |
| IS| 冲突 | 兼容 | 兼容 | 兼容 |

如果一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁授予该事务；反之，如果两者不兼容，该事务就要等待锁释放。

## 行锁

- 在不通过索引条件查询的时候，InnoDB使用的是表锁，而不是行锁
- 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的
- 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁
- 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比 如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁
- 检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁

## 间隙锁（Next-Key）

当用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件 的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，或者叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁（Next-Key锁）。如果使用相等条件请求给一个不存在的记录加锁，InnoDB也会使用间隙锁。

InnoDB 使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求；另外一方面，是为了满足其恢复和复制的需要

举例：

### 幻读

```
Select * from  emp where id > 100 for update;
```
是一个范围条件的检索，InnoDB不仅会对符合条件的记录加锁，也会对id大于101（这些记录并不存在）的“间隙”加锁.要是不使用间隙锁，如果其他事务插入了id大于100的任何记录，那么本事务如果再次执行上述语句，就会发生幻读

### 恢复和复制

对于“insert  into target_tab select * from source_tab where ...”和“create  table target_tab ...select ... From  source_tab where ...(CTAS)”这种SQL语句，MySQL对这种SQL语句做了特别处理。
因为不加锁的话，如果在上述语句执行过程中，其他事务对source_tab做了更新操作，造成不符合target_tab的数据在source_tab中变成了符合target_tab，就可能导致数据恢复的结果错误

## 表锁

对于InnoDB表，在绝大部分情况下都应该使用行级锁，因为事务和行锁往往是我们之所以选择InnoDB表的理由。但在个别特殊事务中，也可以考虑使用表级锁。

- 事务需要更新大部分或全部数据，表又比较大，如果使用默认的行锁，不仅这个事务执行效率低，而且可能造成其他事务长时间锁等待和锁冲突，这种情况下可以考虑使用表锁来提高该事务的执行速度。
- 事务涉及多个表，比较复杂，很可能引起死锁，造成大量事务回滚。这种情况也可以考虑一次性锁定事务涉及的表，从而避免死锁、减少数据库因事务回滚带来的开销。


## 锁在不同条件的差异

InnoDB在不同隔离级别下的一致性读及锁的差异
<table style="text-align:center">
   <tr>
      <td> SQL </td>
      <td>条件</td>
      <td>Read Uncommited</td>
      <td>Read Commited</td>
      <td>Repeatable Read</td>
      <td> Serializable</td>
   </tr>
   <tr>
      <td rowspan="2"> select </td>
      <td>相等</td>
      <td>None locks</td>
      <td>Consisten read/None lock</td>
      <td>Consisten read/None lock</td>
      <td> Share locks</td>
   </tr>
   <tr>
      <td>范围</td>
      <td>None locks</td>
      <td>Consisten read/None lock</td>
      <td>Consisten read/None lock</td>
      <td> Share Next-Key</td>
   </tr>
   <tr>
      <td rowspan="2"> update </td>
      <td>相等</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
   </tr>
   <tr>
      <td>范围</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
   </tr>
   <tr>
      <td> Insert </td>
      <td>N/A</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
   </tr>
   <tr>
      <td rowspan="2">replace</td>
      <td>无键冲突</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
   </tr>
   <tr>
      <td>键冲突</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
   </tr>
   <tr>
      <td rowspan="2"> delete </td>
      <td>相等</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
   </tr>
   <tr>
      <td>范围</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
   </tr>
   <tr>
      <td rowspan="2"> Select ... from ... Lock in share mode </td>
      <td>相等</td>
      <td>Share locks</td>
      <td>Share locks</td>
      <td>Share locks</td>
      <td>Share locks</td>
   </tr>
   <tr>
      <td>范围</td>
      <td>Share locks</td>
      <td>Share locks</td>
      <td>Share Next-Key</td>
      <td>Share Next-Key</td>
   </tr>
   <tr>
      <td rowspan="2">Select * from ... For update</td>
      <td>相等</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
      <td>exclusive locks</td>
   </tr>
   <tr>
      <td>范围</td>
      <td>exclusive locks</td>
      <td>Share locks</td>
      <td>exclusive next-key</td>
      <td>exclusive next-key</td>
   </tr>
   </table>

## 死锁

MyISAM表锁是deadlock free的，这是因为MyISAM总是一次获得所需的全部锁，要么全部满足，要么等待，因此不会出现死锁。但在InnoDB中，除单个SQL组成的事务外，锁是逐步获得的，这就决定了在InnoDB中发生死锁是可能的

发生死锁后，InnoDB一般都能自动检测到，并使一个事务释放锁并回退，另一个事务获得锁，继续完成事务。但在涉及外部锁，或涉及表锁的情况下，InnoDB并不能完全自动检测到死锁，这需要通过设置锁等待超时参数 innodb_lock_wait_timeout来解决。在并发访问比较高的情况下，如果大量事务因无法立即获得所需的锁而挂起，会占用大量计算机资源，造成严重性能问题，甚至拖跨数据库。

死锁的关键在于：两个(或以上)的Session加锁的顺序不一致。通常来说，死锁都是应用设计的问题，通过调整业务流程、数据库对象设计、事务大小，以及访问数据库的SQL语句，绝大部分死锁都可以避免。

避免死锁的常用方法

- 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会。在下面的例子中，由于两个session访问两个表的顺序不同，发生死锁的机会就非常高！但如果以相同的顺序来访问，死锁就可以避免。
- 在程序以批量方式处理数据的时候，如果事先对数据排序，保证每个线程按固定的顺序来处理记录，也可以大大降低出现死锁的可能。
- 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁，更新时再申请排他锁，因为当用户申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁。