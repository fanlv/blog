---
title: MySQL DateTime和Timestamp时区问题
tags:
  - Backend
  - MySQL
categories:
  - MySQL
date: 2021-11-28 01:00:00
updated: 2021-11-28 01:01:00
---

<img alt="cover" src="https://upload-images.jianshu.io/upload_images/12321605-fec8dda1445e78ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240">


## 一、背景

最近负责一个数据传输的项目，其中一个需求就是能把一个`DB`里面的数据拉出来 ，然后回放到另外一个同构的`DB`。两个`DB`的服务不在一个时区（其实这不是重点），可能配置不同。之前有过类似的项目，当时是基建的同事负责做数据同步，同步过去以后`DateTime`、`Timestamp`字段的时区信息都丢了。老板让我调研下问题根因，不要踩之前的坑。

最早的时候看了下同事写的当时`MySQL`时区信息丢失的问题总结文档，文档里面当时把`DateTime`和`Timestamp`两个时区问题混为一起了，也没分析本质原因，导致我当时没看太明白，然后的武断的认为，之所以时区丢失了，是因为基础组件同步`DateTime`和`Timestamp`的时候同步的是字符串，比如`2021-11-27 10:49:35.857969`这种信息，我们传输的时候，只要转`UnixTime`然后传过去就行了（这个其实只是问题之一，其实还跟`time_zone`、`loc`配置相关，后面会说）。

先说结论，如果你能保证`所有项目`连接`DB`的`DSN`配置的`loc`和`time_zone`（`time_zone`没有配置的话会用`MySQL`服务端的默认配置） 都是一样的，那不用看下去了。不管你数据在不同`DB`之间怎么传输，服务读取的`DB`的时区都是符合你的预期的。

## 二、基础知识

### 2.1 Unix时间戳能确定唯一时刻

[UNIX时间](https://zh.wikipedia.org/wiki/UNIX%E6%97%B6%E9%97%B4)，是UNIX或类UNIX系统使用的时间表示方式：从`UTC 1970年1月1日0时0分0秒`起至现在的总秒数`('1970-01-01 00:00:00' UTC)`。

时间字符串`2021-11-27 02:06:50`是不能确定确定唯一时刻的（直白点说就是中国人说的`2021-11-27 02:06:50`和美国人说的`2021-11-27 02:06:50`不是同一时刻），简单说就是 `UnixTime` = `2021-11-27 02:06:50` + `time_zone`,`UnixTime` + `time_zone` 可以得到不同地区人看到的`time_string`。

我们在数据传输和过程中，**是希望这个唯一时刻保持不变，并不是希望时区保持不变**。我发一条消息在中国时间是`2021-11-27 02:06:50`，在其他地方应该是显示其他地方的当地时间。

	t := time.Unix(1637950010, 0) // 时刻唯一确定，可以打印这个时刻不同时区的时间串
	fmt.Println(t.UTC().String()) // 2021-11-26 18:06:50 +0000 UTC
	fmt.Println(t.String()) // 2021-11-27 02:06:50 +0800 CST
	
	now := time.Now()
	fmt.Println(now.UTC().String()) // 2021-11-27 18:06:50.981506 +0000 UTC
	fmt.Println(now.String()) // 2021-11-27 02:06:50.981506 +0800 CST m=+0.000326041
	
	
<br/>	
	
### 2.2 MySQL DateTime 存储信息不带时区

DataTime 表示范围 `'1000-01-01 00:00:00' to '9999-12-31 23:59:59'`。`5.6.4` 版本之前，`DateTime`占用`8`字节，`5.6.4`之后默认是`5`字节（到秒），如果要更高精度可以配置`Fractional Seconds Precision`， `fsp=1~2`占用`1`字节 ，`3~4`占用 `2`个字节，`5~6`占用`3`个字节， 如`DATETIME(6)` 精确到秒后`6`位，一共占用`8`字节。

需要注意的是：不论是`5.6.4`之前，还是`5.6.4`之后`DateTime`字段里面都**没有带时区信息，不能确定唯一时刻**，更多可以看 [MySQL官网文档](https://dev.mysql.com/doc/internals/en/date-and-time-data-type-representation.html)。


![datetime_type.jpg](https://upload-images.jianshu.io/upload_images/12321605-eeb9a0f6cded28da.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

### 2.3 MySQL Timestamp 和 time_zone


> Timestamp: A four-byte integer representing seconds UTC since the epoch ('1970-01-01 00:00:00' UTC)
> The Timestamp data type is used for values that contain both date and time parts. Timestamp has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC.


`Timestamp`就是存的`Unix`时间戳，表示范围是`'1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07'`，是不是`Timestamp`就没有时区问题？并不是。[MySQL官方文档有如下一段话如下](https://dev.mysql.com/doc/refman/8.0/en/datetime.html)：

> MySQL converts Timestamp values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval. (This does not occur for other types such as DATETIME.) By default, the current time zone for each connection is the server's time. The time zone can be set on a per-connection basis. As long as the time zone setting remains constant, you get back the same value you store. If you store a Timestamp value, and then change the time zone and retrieve the value, the retrieved value is different from the value you stored. This occurs because the same time zone was not used for conversion in both directions. The current time zone is available as the value of the time_zone system variable. For more information, see Section 5.1.15, “MySQL Server Time Zone Support”.

简单说，每个`session`可以设置不同的`time_zone`，如果你设置`session`用的`time_zone`和读取`session`用的`time_zone`不一样，那你会得到错误/不同的值。说白了一个`Timestamp`字段，写入和读取的`session`必须一样。针对单个`DB`的场景，建议所有`session`的`dsn`都不配置`time_zone`。

`time_zone` 有三种设置方法

	set time_zone = '+8:00'; // 设置当前 session 的 time_zone，立即生效
	set global time_zone = '+8:00'; // 设置MySQL全局默认配置，新的连接才生效
	
	dsn里面指定 time_zone='+8:00'
	user:pwd@tcp(host:port)/db?charset=utf8mb4&parseTime=True&loc=Asia%2FShanghai&time_zone=%27%2B8%3A00%27


<br/>

### 2.3 SQL 数据传输时候，DataTime和Timestamp都是字符串传输

	DROP TABLE IF EXISTS `ts_test`;
	CREATE TABLE ts_test (
		`id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'pk',
		`program_insert_time` varchar(100) COMMENT '代码里面获取的时间字符串，insert 语句用的',
		`time_zone` INT COMMENT '插入的时候当前 session 的 time_zone 设置的是什么',
		`loc` varchar(20) COMMENT '插入这个语句时候，dsn 的 loc',
		`ts` Timestamp(6),
		 PRIMARY KEY (id)
	);
	
然后分别执行

	INSERT INTO `dt_test` (`loc`,`program_insert_time`,`dt`) VALUES ('Asia/Shanghai','2021-11-27 14:08:07.3751 +0000 UTC','2021-11-27 14:08:07.3751')
	SELECT * FORM `dt_test`

wireshark 抓包可知SQL传输的时候，DataTime和Timestamp都是直接传输不带时区的字符串，如`2021-11-27 14:08:07.3751`这种。

![insert_1.jpg](https://upload-images.jianshu.io/upload_images/12321605-84ef5b79baf33dcc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![insert_2.jpg](https://upload-images.jianshu.io/upload_images/12321605-4fc68b0212a26a53.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![select_req.jpg](https://upload-images.jianshu.io/upload_images/12321605-dc877ee3fc100caa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![select_resp.jpg](https://upload-images.jianshu.io/upload_images/12321605-b95c5bbf8a28a707.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>


## 三、问题分析

### 3.1 Datetime 问题分析

上面我们说过`SQL`请求和响应的`Data`里面`Datetime`和`Timestamp`字段都是用**时间字符串**，我们用`GORM`执行`SQL`的时候，我们传的对`Golang`的`time.Time`，这个`time`类型的时间是怎么最终转换成不带时区的时间字符串呢？翻了下`go-sql-driver`[代码](https://github.com/go-sql-driver/mysql/blob/master/packets.go#L1119)，看到有下面这段逻辑。

    case time.Time:
        paramTypes[i+i] = byte(fieldTypeString)
        paramTypes[i+i+1] = 0x00

        var a [64]byte
        var b = a[:0]

        if v.IsZero() {
            b = append(b, "0000-00-00"...)
        } else {
            b, err = appendDateTime(b, v.In(mc.cfg.Loc)) // v 就是我们传入的 time.Time 对象 
            if err != nil {
                return err
            }
        }


看下 [appendDateTime](https://github.com/go-sql-driver/mysql/blob/6cf3092b0e12f6e197de3ed6aa2acfeac322a9bb/utils.go#L279) 函数逻辑就是把`time.Time`转成`mc.cfg.Loc`时区的字符串。

举例说明就是，我们插入一个`SQL`的时候，假设是代码里面 `time.Now()` 获取了一个时间对象，这个时间对象是有时区信息的（或者说是能确定唯一时刻的），时区是当前系统的时区。传到`go-sql-driver`里面去以后，`driver`需要把这个对象转成不带时区的字符串，具体要转成哪个时区的字符串，就是由`mc.cfg.Loc`决定的。我们再往上跟下看下`mc.cfg.Loc`是哪里传入的。找到如下代码，由代码可以知道，`loc`信息是我们配置`dns`连接串的时候传入的,`loc`不传的话，默认是`UTC 0`时间

	https://github.com/go-sql-driver/mysql/blob/master/driver.go#L73
	// OpenConnector implements driver.DriverContext.
	func (d MySQLDriver) OpenConnector(dsn string) (driver.Connector, error) {
		cfg, err := ParseDSN(dsn) // https://github.com/go-sql-driver/mysql/blob/6cf3092b0e12f6e197de3ed6aa2acfeac322a9bb/dsn.go#L291
		if err != nil {
			return nil, err
		}
		return &connector{
			cfg: cfg,
		}, nil
	}
	
	// https://github.com/go-sql-driver/mysql/blob/6cf3092b0e12f6e197de3ed6aa2acfeac322a9bb/dsn.go#L68
	// NewConfig creates a new Config and sets default values.
	func NewConfig() *Config {
		return &Config{
			Collation:            defaultCollation,
			Loc:                  time.UTC, // loc 传的话，默认是UTC时间
			MaxAllowedPacket:     defaultMaxAllowedPacket,
			AllowNativePasswords: true,
			CheckConnLiveness:    true,
		}
	}
	

	// Connect implements driver.Connector interface.
	// Connect returns a connection to the database.
	func (c *connector) Connect(ctx context.Context) (driver.Conn, error) {
		var err error
	
		// New mysqlConn
		mc := &mysqlConn{
			maxAllowedPacket: maxPacketSize,
			maxWriteSize:     maxPacketSize - 1,
			closech:          make(chan struct{}),
			cfg:              c.cfg,
		}
		mc.parseTime = mc.cfg.ParseTime
		

	

再来看查询的时候，时间字符串的转换问题，上面用`WireShark`抓包的时候，知道我们执行`Select`查询数据的时候，`MySQL`给我们返回的也是时间字符串。那客户端代码是如何转成`time.Time`对象的？我们知道`dsn`里面有个`parseTime`字段是来控制，从`parseTime`相关代码我们可以找到[如下代码](https://github.com/go-sql-driver/mysql/blob/master/packets.go#L789)。


	if !mc.parseTime {
		continue
	}

	// Parse time field
	switch rows.rs.columns[i].fieldType {
	case fieldTypeTimestamp,
		fieldTypeDateTime,
		fieldTypeDate,
		fieldTypeNewDate:
		if dest[i], err = parseDateTime(dest[i].([]byte), mc.cfg.Loc); err != nil {
			return err
		}
	}

看下 [parseDateTime](https://github.com/go-sql-driver/mysql/blob/6cf3092b0e12f6e197de3ed6aa2acfeac322a9bb/utils.go#L109) 函数，就是用`mc.cfg.Loc`加时间字符串转换成了`time.Time`

	func parseDateTime(b []byte, loc *time.Location) (time.Time, error) {
		const base = "0000-00-00 00:00:00.000000"
		switch len(b) {
		case 10, 19, 21, 22, 23, 24, 25, 26: // up to "YYYY-MM-DD HH:MM:SS.MMMMMM"
			if string(b) == base[:len(b)] {
				return time.Time{}, nil
			}

<br/>

### 3.2 Datetime 总结

`Datetime`在`MySQL`服务端保存的只是一个字符串，时区信息都是由连接串的`loc`字符串控制的。如果要想时区保证一致，写入和读取的`loc`必须保证一致。

需要注意几点：

1. `loc`配置是给插入的时候用`time.Time`转时间字符串用的。如果你裸写插入`SQL`（RawSQL），`loc`怎么配置，都不会影响时间串，数据存的时间，就是你`Insert`语句里面拼接的时间串。
2. 如果们插入的是`time.Time` (能确定唯一时刻)对象，插入客户端所在的系统的时区信息对插入结果没影响，因为客户端是用`time.Time`+`loc`来得到时间字符串。
3. `loc` 没有配置的话，默认是`UTC0`


![datetime.jpg](https://upload-images.jianshu.io/upload_images/12321605-e3243effcb8ed367.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

### 3.3 Timestamp

`Timestamp`在`go-sql-driver`里面的处理流程跟`Datetime`一样，区别是是时间字符串到了服务端，服务端会用`time_zone`加字符串得到`UnixTime`然后保存（这部分只是个人猜想，并没有去找`MySQL`源码验证，[只是通过简单的代码测试](./time_span.go)和官方文档来验证自己的想法），从结果上来看，读入和写入的`session`的`time_zone`必须保持一致读的数据才是对的。

[time_zone 相关官方文档](https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html)

[Timestamp 存的4字节UTC时间](https://dev.mysql.com/doc/internals/en/date-and-time-data-type-representation.html)

[Timestamp 和 time_zone 关系 第七段](https://dev.mysql.com/doc/refman/8.0/en/datetime.html)

<br/>

### 3.4 Timestamp 总结

如果真的要存时间戳，建议用`binint`存，这样不管数据怎么传输，不管`loc`、`time_zone` 怎么配置，都没有时区问题。

![Timestamp.jpg](https://upload-images.jianshu.io/upload_images/12321605-ffc61b32e15393f2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

## 四、数据传输的时候如何保证数据正确

知道了上面的基本信息以后，数据传输系统要做的事就很明确了。

1. 读取和写入的数据的时候，`loc`和`time_zone`配置跟业务方保持一致就行了。
2. `DTS`数据传输的时候，因为`binlog`字段都是字符串，需要把`时间字符串`+`loc`转成时间戳，然后发送到对端。


![dts.png](https://upload-images.jianshu.io/upload_images/12321605-ba0159fd7b4faf09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 五、问题本质

`MySQL` 存储、写入读取传输时候都是时间字符。客户端发送和接收的时候需要用`loc`来标明这个字符串的时区信息，所以读取和写入的`loc`必须要保证是相同的，所以这个字符串才有相同的语义。

如果所有业务方，都不设置`loc`，统一都是默认配置。时间戳，直接用`bigint`存那就没有任何时区问题。世界美好一点不好吗？何必自己给自己折腾一堆莫名其妙问题。














