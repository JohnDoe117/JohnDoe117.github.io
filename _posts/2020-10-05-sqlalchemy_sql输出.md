---
title: Python -- sqlalchemy显示原始sql最佳方法
description: python小技巧
categories:
 - python小技巧
tags:
 - python小技巧
---

### Sqlalchemy获取原生语句直接调试的方法

最近工作是对数据库执行效率进行调优，项目使用的是sqlalchemy的orm框架，自然要观察框架生成的sql语句：

目前网上博客的处理方式大致是在创建engine的同时指定echo参为数True

```python
engine = create_engine('mysql+pymysql://root:123456789@localhost:3306/test', echo=True)
```

这样能够在控制台输出sql的详细信息:

```
2020-11-05 14:37:07,630 INFO sqlalchemy.engine.base.Engine SHOW VARIABLES LIKE 'sql_mode'
2020-11-05 14:37:07,630 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,636 INFO sqlalchemy.engine.base.Engine SHOW VARIABLES LIKE 'lower_case_table_names'
2020-11-05 14:37:07,636 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,639 INFO sqlalchemy.engine.base.Engine SELECT DATABASE()
2020-11-05 14:37:07,639 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,639 INFO sqlalchemy.engine.base.Engine show collation where `Charset` = 'utf8mb4' and `Collation` = 'utf8mb4_bin'
2020-11-05 14:37:07,639 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,642 INFO sqlalchemy.engine.base.Engine SELECT CAST('test plain returns' AS CHAR(60)) AS anon_1
2020-11-05 14:37:07,642 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,643 INFO sqlalchemy.engine.base.Engine SELECT CAST('test unicode returns' AS CHAR(60)) AS anon_1
2020-11-05 14:37:07,643 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,643 INFO sqlalchemy.engine.base.Engine SELECT CAST('test collated returns' AS CHAR CHARACTER SET utf8mb4) COLLATE utf8mb4_bin AS anon_1
2020-11-05 14:37:07,644 INFO sqlalchemy.engine.base.Engine {}
2020-11-05 14:37:07,644 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
INSERT INTO user (name, passwd, insert_time) VALUES (john, 111, 2020-11-05 14:37:07.645797)
2020-11-05 14:37:07,645 INFO sqlalchemy.engine.base.Engine INSERT INTO user (name, passwd, insert_time) VALUES (%(name)s, %(passwd)s, %(insert_time)s)
2020-11-05 14:37:07,645 INFO sqlalchemy.engine.base.Engine {'name': 'john', 'passwd': '111', 'insert_time': datetime.datetime(2020, 11, 5, 14, 37, 7, 645797)}
2020-11-05 14:37:07,647 INFO sqlalchemy.engine.base.Engine COMMIT
```

但是其实很多信息是我们并不关心的。

据了解mybaits有个pycharm插件是通过命令行输出拼装出能够直接执行的sql插件

mybaits-log

但是这种实现方式有平台的局限性，所以想直接通过代码获取sql然后进行组装

---

经过了解sqlalchemy为了防止注入并不不会拼装sql参数进入语句中，但是这并不要紧，我们可以自己拼装进去。

经过翻阅文档发现这样一个类`sqlalchemy.events.``ConnectionEvents`[¶](https://docs.sqlalchemy.org/en/14/core/events.html?highlight=after_cursor#sqlalchemy.events.ConnectionEvents)）

> The methods here define the name of an event as well as the names of members that are passed to listener functions.

所以我们基于这样的事件可以完成以下功能（具体细节可以参阅上面的文档 已添加链接）：

```Python
from sqlalchemy import event
@event.listens_for(engine, "before_cursor_execute")
def comment_sql_calls(conn, cursor, statement, parameters, context, executemany):
    raw_sql = statement%parameters
    print(raw_sql)
```

此时我们去掉echo的参数

每次sql执行之前就会输出对应的语句

```
INSERT INTO user (name, passwd, insert_time) VALUES (john, 111, 2020-11-05 14:51:13.377550)
```

只需要添加这几行代码就可以完美解决sql输出的问题

当然这个事件触发还可以完成很多功能。



demo地址：https://github.com/JohnDoe117/sqlalchemy_sql_log

