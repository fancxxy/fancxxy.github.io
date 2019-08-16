---
title: 'driver: bad connection'
date: 2019-08-16 15:13:38
tags: golang
---

生产环境采集器有指标未上报，从配置页面查看该指标相关信息，发现该指标配置了cron命令，按周期执行任务，cron表达式为`0 10 9 * * ?`，每天的9点10分执行一次。查看9点10分的后台日志，发现程序在执行`db.Ping()`时返回了错误`driver: bad connection`，使用的数据库连接被关闭了。初始化mysql连接时只设置了最大连接数和最大空闲连接数，并没有设置客户端的超时时间。
<!--more-->

查看mysql非互交式连接超时时间，默认为8小时，为方便模拟问题，临时设置为5秒。
```
mysql root@localhost:(none)> show global variables like "%wait_timeout%";
+--------------------------+----------+
| Variable_name            | Value    |
+--------------------------+----------+
| innodb_lock_wait_timeout | 50       |
| lock_wait_timeout        | 31536000 |
| mysqlx_wait_timeout      | 28800    |
| wait_timeout             | 28800    |
+--------------------------+----------+
4 rows in set
Time: 0.014s
mysql root@localhost:(none)> set global wait_timeout=5;
Query OK, 0 rows affected
Time: 0.002s
mysql root@localhost:(none)>
```

测试代码如下，`sql.Open()`函数返回的`*sql.DB`对象包含一个连接池，初始化时建立了10个空闲连接，每隔10秒调用一次`Ping()`函数，函数在运行到第二个周期时取到的超时的连接。
```go
package main

import (
	"database/sql"
	"log"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "username:password@/dbname")
	if err != nil {
		log.Fatalf("sql.Open error: %v", err)
	}
	db.SetMaxOpenConns(100)
	db.SetMaxIdleConns(10)

	for {
		if err := db.Ping(); err != nil {
			log.Println("ping failed:", err)
		} else {
			log.Println("ping succeed")
		}

		time.Sleep(time.Second * 10)
	}
}
```

![](http://ww1.sinaimg.cn/large/60abf8f3ly1g61iy9ystlj219o110gy8.jpg)

修改的方法是在客户端加上超时控制，把已超时的连接从连接池剔除。
```go
func main() {
    // ...
	db.SetMaxOpenConns(100)
	db.SetMaxIdleConns(10)
	db.SetConnMaxLifetime(time.Second * 3)
    // ...
}
```
再次执行不会再报`bad connection`的错误了。
![](http://ww1.sinaimg.cn/large/60abf8f3ly1g61iyvbxv9j219o110drd.jpg)


