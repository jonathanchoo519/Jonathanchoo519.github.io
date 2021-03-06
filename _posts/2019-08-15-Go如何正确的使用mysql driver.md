---
layout:     post
title:      Go如何正确的使用mysql driver
subtitle:   连接池
date:       2019-08-15
author:     JC
header-img: img/mysql.jpg
catalog: true
tags:
    - Mysql
    - Golang
---

官方文档https://golang.org/pkg/database/sql/#DB，

虽然写了如何使用它来执行SQL数据库查询和语句的例子，但是没有很好的说明如何正确的配置sql.DB改善性能，SetMaxOpenConns(), SetMaxIdleConns() and SetConnMaxLifetime()这三个方法也通常被初学者忽略。

## Open and idle connections:

一个sql.DB是包含许多’open’和’idle’连接的数据库连接池的对象。 当你使用它执行数据库任务时（例如执行SQL语句或查询行），连接标记为打开。 任务完成后，连接变为空闲状态。

当你使用sql.DB执行数据库任务时，它将首先检查池中是否有空闲连接。 如果有一个可用，那么Go将重新使用现有连接并在任务持续期间将其标记为打开。 如果游泳池中没有空闲的连接，那么Go会创建一个新的连接并“打开”它。

MySQL短连接每次请求操作数据库都需要建立与MySQL服务器建立TCP连接，这是需要时间开销的。TCP连接需要3次网络通信。这样就增加了一定的延时和额外的IO消耗。请求结束后会关闭MySQL连接，还会发生3/4次网络通信。close操作不会增加响应延时，原因是close后是由操作系统自动进行通信的，应用程序感知不到，长连接就可以避免每次请求都创建连接的开销，节省了时间和IO消耗。

	type Config struct {
        DSN         string
        Active      int            // pool
        Idle        int            // pool
        IdleTimeout xtime.Duration // connect max life time.
    }

    // NewMysql initialize mysql connection .
    func NewMysql(c *Config) (db *sql.DB) {
        // TODO add query exec and transaction timeout .
        db, err := Open(c)
        if err != nil {
         panic(err)
        }
          return
     }

    func Open(c *Config) (db *sql.DB, err error) {
        db, err = sql.Open("mysql", c.DSN)
        if err != nil {
          log.Error("sql.Open() error(%v)", err)
          return nil, err
         }
        db.SetMaxOpenConns(c.Active)
        db.SetMaxIdleConns(c.Idle) // 默认情况下，sql.DB允许在连接池中保留最多2个空闲连接
        db.SetConnMaxLifetime(time.Duration(c.IdleTimeout))
        return db, nil
    }


从理论上讲，在连接池中允许更多的空闲连接会提高性能，因为它不太可能需要从头开始建立新的连接，因此有助于节省资源。

## 那么我们应该保持一个大的空闲连接池？

其实不然，保持空闲连接是有代价的，它需要占用的内存的，这点需要注意。设置多大的Idle应该根据自身应用程序来定。如果连接闲置时间过长，则可能无法使用。例如，MySQL的wait_timeout设置会自动关闭任何8小时未使用的连接（默认情况下），所以我们看到上面的代码我们设置了自己的超时时间。发生这种情况时，sql.DB会优雅地的关掉它。在关闭之前，连接会自动重试两次，此时Go将从池中移除连接并创建一个新连接。因此，如果将MaxIdleConns设置得太高，实际上可能会导致连接变得无法使用，并且比使用更少空闲连接池（使用更频繁的连接数更少）时使用的资源更多。

## 总结：

1.对于大多数使用SetMaxOpenConns（）来限制打开连接的最大数量的程序，都会对性能产生负面影响，但如果数据库资源比较紧张的情况下，这么做还是有好处的。

2.如果程序突发或定期同时执行两个以上的数据库任务，那么通过SetMaxIdleConns（）增加空闲连接池的大小可能会产生积极的性能影响。 但是需要注意的是设置过大可能会适得其反。上线之前最好做个压测已到达最佳性能。

3.对于大多数通过SetConnMaxLifetime（）设置连接超时的应用程序，都会对性能产生负面影响。 
但是，如果你的数据库本身强制实现一个短的连接生命周期，那么在sql.DB对它进行设置是有价值的，以避免尝试和重试错误连接的开销。

4.如果希望程序在数据库达到硬连接限制时等待连接释放（而不是返回错误），则应该明确设置SetMaxOpenConns（）和SetMaxIdleConns（）。
