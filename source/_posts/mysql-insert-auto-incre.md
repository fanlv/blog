---
title: MySQL Insert 源码分析
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2022-05-25 00:12:56
updated: 2022-05-25 00:12:56
---

#一、背景

最近我们在做线上的数据迁移测试（可以理解就是把`A`数据中心的数据迁移到`B`数据中心，`A`和`B`数据中心的`MySQL`是同构的，迁移过程中，`A`、`B`的`MySQL`都有正常的业务数据写入，具体背景可以看 [数据传输系统落地和思考](https://fanlv.wiki/2022/02/26/dts/) 这篇文章）。部分表的数据每次我们触发迁移的时候，业务方写入数据的时候就会有`Error 1062: Duplicate entry 'xxx' for key 'PRIMARY'`这样的错误。业务方同学还反馈他们写数据的时候并没有指定`ID`，所以他们对这样的报错比较困惑，具体他们的数据写入的伪代码如下：

	type Data struct {
		ID           int64     `gorm:"primaryKey;column:id"`
		PageID       string    `gorm:"column:page_id`
		CreateTime   time.Time `gorm:"column:create_time"`
		ModifiedTime time.Time `gorm:"column:modified_time"`
	}

	data := &Data{
					PageID:       uuid.NewString(),
					CreateTime:   now,
					ModifiedTime: now,
				}
	
	err := db.Create(data).Error
	if err != nil {
		return err
	}
	
再交代一下其他的背景。

1. 业务上这个表的写入的`QPS`相对比较高。迁移的数据量也比较大。
2. 我们做数据迁移的时候，从`A`数据中心迁移到`B`数据中心的时候，会抹掉`binlog`中的`ID`数据，然后用一个中心的发号器`IDGenerator`生成一个新的`ID`然后填到`binlog`中去，然后再插入这个数据。

由于，每次都是在数据迁移的时候，报这个`PK Duplicate Error`的错误，基本肯定是我们做数据迁移导致的`PK`冲突。引出几个问题：

1. 生成自增`ID`实现方式？并发生成`ID`会不会冲突？
2. 生成自增`ID`是否会加锁，锁的释放机制是啥？
3. 生成自增`ID`和`唯一索引冲突检查`流程是怎么样的？

问了一圈身边的朋友，好像大家对这些细节都不太了解，所以决定自己撸下源码看下。


# 二、AUTO_INCREMENT 基础知识

`MySQL`的[《AUTO_INCREMENT Handling in InnoDB》](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html#innodb-auto-increment-lock-modes) 这篇官方文档，其实把`AUTO_INCREMENT`相关特性都介绍很清楚了，我们做个简单总结。

1. `InnoDB`提供了一种可配置的锁定机制，可以显着提高向具有`AUTO_INCREMENT`列的表添加行的`SQL`语句的可伸缩性和性能。
2. 定义为`AUTO_INCREMENT`的列，必须是索引的第一列或者是唯一列，因为需要使用`SELECT MAX(ai_col)`查找以获得最大值列值。不这样定义，`Create Table`的时候会报`1075 - Incorrect table definition; there can be only one auto column and it must be defined as a key`错误。
3. `AUTO_INCREMENT`的列，可以只定义为普通索引，不一定要是`PRIMARY KEY`或者`UNIQUE`，但是为了保证`AUTO_INCREMENT`的唯一性，建议定义为`PK`或者`UNIQUE`


## 2.1 MySQL插入语句的几种类型

在介绍`AUTO_INCREMENT`的锁模式之前，先介绍下，`MySQL`插入的几种类型：

* `Simple inserts`，可以预先确定要插入的行数（当语句被初始处理时）的语句。 这包括没有嵌套子查询的单行和多行`INSERT`和`REPLACE`语句。如下：

		INSERT INTO t1 (c2) VALUES ('xxx');
		
* `Bulk inserts`，事先不知道要插入的行数（和所需自动递增值的数量）的语句。 这包括`INSERT ... SELECT`，`REPLACE ... SELECT`和`LOAD DATA`语句，但不包括纯`INSERT`。 `InnoDB`在处理每行时一次为`AUTO_INCREMENT`列分配一个新值。

		INSERT INTO t1 (c2) SELECT 1000 rows from another table ...

* `Mixed-mode inserts`，这些是`Simple inserts`语句但是指定一些（但不是全部）新行的自动递增值。 示例如下，其中`c1`是表`t1`的`AUTO_INCREMENT`列： 

		INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');
		
	另一种类型的`Mixed-mode inserts`是`INSERT ... ON DUPLICATE KEY UPDATE`，其在最坏的情况下实际上是`INSERT`语句随后又跟了一个`UPDATE`，其中`AUTO_INCREMENT`列的分配值不一定会在`UPDATE`阶段使用。

* `INSERT-like` `statements` ，以上所有插入语句的统称。


## 2.2 AUTO_INCREMENT 锁模式


`MySQL`可以通过设置`innodb_autoinc_lock_mode` 变量来配置`AUTO_INCREMENT`列的锁模式，分别可以设置为`0`、`1`、`2` 三种模式。[对应的源码中的三种类型如下](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L145)：


### 传统模式（traditional）

1. 传统的锁定模式提供了与引入`innodb_autoinc_lock_mode`变量之前相同的行为。由于语义上可能存在差异，提供传统锁定模式选项是为了向后兼容、性能测试和解决“混合模式插入”问题。
2. 在这一模式下，所有的`insert`语句(`insert like`) 都要在语句开始的时候得到一个表级的`auto_inc`锁，在语句结束的时候才释放这把锁，注意呀，这里说的是语句级而不是事务级的，一个事务可能包涵有一个或多个语句。
3. 它能保证值分配的可预见性，与连续性，可重复性，这个也就保证了`insert`语句在复制到`slave`的时候还能生成和`master`那边一样的值(它保证了基于语句复制的安全)。
4. 由于在这种模式下`auto_inc`锁一直要保持到语句的结束，所以这个就影响到了并发的插入。


### 连续模式（consecutive）

1. 在这种模式下，对于`simple insert`语句，`MySQL`会在语句执行的初始阶段将一条语句需要的所有自增值会一次性分配出来，并且通过设置一个互斥量来保证自增序列的一致性，一旦自增值生成完毕，这个互斥量会立即释放，不需要等到语句执行结束。所以，在`consecutive`模式，多事务并发执行`simple insert`这类语句时， 相对`traditional`模式，性能会有比较大的提升。
2. 由于一开始就为语句分配了所有需要的自增值，那么对于像`Mixed-mode insert`这类语句，就有可能多分配了一些值给它，从而导致自增序列出现`空隙`。而`traditional`模式因为每一次只会为一条记录分配自增值，所以不会有这种问题。
3. 另外，对于Bulk inserts语句，依然会采取AUTO-INC锁。所以，如果有一条Bulk inserts语句正在执行的话，Simple inserts也必须等到该语句执行完毕才能继续执行。


### 交错模式（interleaved）

在这种模式下，对于所有的`insert-like`语句，都不会存在表级别的`AUTO-INC`锁，意味着同一张表上的多个语句并发时阻塞会大幅减少，这时的效率最高。但是会引入一个新的问题：当`binlog_format`为`statement`时，这时的复制没法保证安全，因为批量的`insert`，比如`insert ..select..`语句在这个情况下，也可以立马获取到一大批的自增`ID`值，不必锁整个表，`slave`在回放这个`SQL`时必然会产生错乱（`binlog`使用`row`格式没有这个问题）。



### 其他

* 自增值的生成后是不能回滚的，所以自增值生成后，事务回滚了，那么那些已经生成的自增值就丢失了，从而使自增列的数据出现空隙。
* 正常情况下，自增列是不存在`0`这个值的。所以，如果插入语句中对自增列设置的值为`0`或者`null`，就会自动应用自增序列。那么，如果想在自增列中插入为`0`这个值，怎么办呢？可以通过将[SQL Mode](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_auto_value_on_zero)设置为`NO_AUTO_VALUE_ON_ZERO`即可。
* 在`MySQL 5.7`以及更早之前，自增序列的计数器(`auto-increment counter`)是保存在内存中的。`auto-increment counter`在每次`MySQL`重新启动后通过类似下面的这种语句进行初始化：

		SELECT MAX(AUTO_INC_COLUMN) FROM table_name FOR UPDATE

* 而从`MySQL 8`开始，`auto-increment counter`被存储在了`redo log`中，并且每次变化都会刷新到`redo log`中。另外，我们可以通过`ALTER TABLE … AUTO_INCREMENT = N`来主动修改`auto-increment counter`。





我们线上生产环境配置是`innodb_autoinc_lock_mode = 2`，`binlog_format = ROW`

	mysql> show variables like 'innodb_autoinc_lock_mode';
	+--------------------------+-------+
	| Variable_name            | Value |
	+--------------------------+-------+
	| innodb_autoinc_lock_mode | 2     |
	+--------------------------+-------+
	1 row in set (0.07 sec)
	
	mysql> show variables like 'binlog_format';
	+---------------+-------+
	| Variable_name | Value |
	+---------------+-------+
	| binlog_format | ROW   |
	+---------------+-------+
	1 row in set (0.36 sec)


# 三、Insert 流程源码分析

## 3.1 Insert 执行过程

[`mysql_parse`](https://github.com/mysql/mysql-server/blob/5.7/sql/sql_parse.cc#L5438) -> [`mysql_execute_command`](https://github.com/mysql/mysql-server/blob/5.7/sql/sql_parse.cc#L2456) -> [`Sql_cmd_insert::execute`](https://github.com/mysql/mysql-server/blob/5.7/sql/sql_insert.cc#L3176) -> [`Sql_cmd_insert::mysql_insert`](https://github.com/mysql/mysql-server/blob/5.7/sql/sql_insert.cc#L428) -> [`write_record`](https://github.com/mysql/mysql-server/blob/5.7/sql/sql_insert.cc#L1512) -> [`handler::ha_write_row`](https://github.com/mysql/mysql-server/blob/5.7/sql/handler.cc#L8153) -> [`ha_innobase::write_row`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7506)

这里我们主要关注`innodb`层的数据写入函数[`ha_innobase::write_row`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7506) 相关的代码就好了，自增`ID`的生成和`唯一索引冲突检查`都是在这个函数里面完成的。

## 3.2 innodb 数据插入流程

通过[`ha_innobase::write_row`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7506) 代码我们可以知道，在`innodb`层写入数据主要分为`7`步：

	1. Validation checks before we commence write_row operation.
	2. Intermediate commit if original operation involves ALTER table with algorithm = copy. Intermediate commit ease pressure on recovery if server crashes while ALTER is active.
	3. Handling of Auto-Increment Columns.
	4. Prepare INSERT graph that will be executed for actual INSERT (This is a one time operation)
	5. Execute insert graph that will result in actual insert.
	6. Handling of errors related to auto-increment. 
	7. Cleanup and exit. 

我们主要关注，自增列相关的`步骤三`和`步骤六`，数据写入的`步骤六`。

### 自增ID的相关处理过程

[先看第三步代码：Handling of Auto-Increment Columns](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7631)，主要的函数栈如下：

    ->ha_innobase::write_row
        ->handler::update_auto_increment // 调用 update_auto_increment 函数更新auto increment的值
            ->ha_innobase::get_auto_increment // 获取 dict_tabel中的当前 auto increment 值，并根据全局参数更新下一个 auto increment 的值到数据字典中
                ->ha_innobase::innobase_get_autoinc // 读取 autoinc 值
                    ->ha_innobase::innobase_lock_autoinc
                       -> dict_table_autoinc_lock(m_prebuilt->table); // lock_mode = 2 的时候
                    -> dict_table_autoinc_unlock(m_prebuilt->table); // 解锁
            ->set_next_insert_id // 多行插入的时候设置下一个插入的id值



[三种模式对应的加锁源码](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7387)：


	static const long AUTOINC_OLD_STYLE_LOCKING = 0;
	static const long AUTOINC_NEW_STYLE_LOCKING = 1;
	static const long AUTOINC_NO_LOCKING = 2;

	
	dberr_t
	ha_innobase::innobase_lock_autoinc(void)
	/*====================================*/
	{
		DBUG_ENTER("ha_innobase::innobase_lock_autoinc");
		dberr_t		error = DB_SUCCESS;
		long		lock_mode = innobase_autoinc_lock_mode;
	
		ut_ad(!srv_read_only_mode
		      || dict_table_is_intrinsic(m_prebuilt->table));
	
		if (dict_table_is_intrinsic(m_prebuilt->table)) {
			/* Intrinsic table are not shared accorss connection
			so there is no need to AUTOINC lock the table. */
			lock_mode = AUTOINC_NO_LOCKING;
		}
	
		switch (lock_mode) {
		case AUTOINC_NO_LOCKING:
			/* Acquire only the AUTOINC mutex. */
			dict_table_autoinc_lock(m_prebuilt->table);
			break;
	
		case AUTOINC_NEW_STYLE_LOCKING:
			/* For simple (single/multi) row INSERTs, we fallback to the
			old style only if another transaction has already acquired
			the AUTOINC lock on behalf of a LOAD FILE or INSERT ... SELECT
			etc. type of statement. */
			if (thd_sql_command(m_user_thd) == SQLCOM_INSERT
			    || thd_sql_command(m_user_thd) == SQLCOM_REPLACE) {
	
				dict_table_t*	ib_table = m_prebuilt->table;
	
				/* Acquire the AUTOINC mutex. */
				dict_table_autoinc_lock(ib_table);
	
				/* We need to check that another transaction isn't
				already holding the AUTOINC lock on the table. */
				if (ib_table->n_waiting_or_granted_auto_inc_locks) {
					/* Release the mutex to avoid deadlocks. */
					dict_table_autoinc_unlock(ib_table);
				} else {
					break;
				}
			}
			/* Fall through to old style locking. */
	
		case AUTOINC_OLD_STYLE_LOCKING:
			DBUG_EXECUTE_IF("die_if_autoinc_old_lock_style_used",
					ut_ad(0););
			error = row_lock_table_autoinc_for_mysql(m_prebuilt);
	
			if (error == DB_SUCCESS) {
	
				/* Acquire the AUTOINC mutex. */
				dict_table_autoinc_lock(m_prebuilt->table);
			}
			break;
	
		default:
			ut_error;
		}
	
		DBUG_RETURN(error);
	}
	


[步骤六：插入成功以后，还需要更新 autoinc 值](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/handler/ha_innodb.cc#L7719)


				if (auto_inc >= m_prebuilt->autoinc_last_value) {
	set_max_autoinc:
					/* This should filter out the negative
					values set explicitly by the user. */
					if (auto_inc <= col_max_value) {
						ut_a(m_prebuilt->autoinc_increment > 0);
	
						ulonglong	offset;
						ulonglong	increment;
						dberr_t		err;
	
						offset = m_prebuilt->autoinc_offset;
						increment = m_prebuilt->autoinc_increment;
	
						auto_inc = innobase_next_autoinc(
							auto_inc,
							1, increment, offset,
							col_max_value);
	
						err = innobase_set_max_autoinc(
							auto_inc);
	
						if (err != DB_SUCCESS) {
							error = err;
						}
					}
				}


# 参考资料


[AUTO_INCREMENT Handling in InnoDB](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html#innodb-auto-increment-lock-modes)

[Mysql之AUTO_INCREMENT浅析](https://blog.csdn.net/scientificCommunity/article/details/122846585)

[MySQL · 内核分析 · InnoDB主键约束和唯一约束的实现分析](https://www.bookstack.cn/read/aliyun-rds-core/ea7a43cf992eca56.md)

