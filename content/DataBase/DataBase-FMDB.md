# 1.cocoapods引入FMDB

`pod 'FMDB'`

# 2.FMDB核心类

FMDB有三个主要的类

- `FMDatabase`

一个FMDatabase对象就代表一个单独的SQLite数据库，用来执行SQL语句。

- `FMResultSe`

使用FMDatabase执行查询后的结果集

- `FMDatabaseQueue`

用于在多线程中执行多个查询或更新，它是线程安全的

# 3.打开或创建数据库

```objc
// 获取数据库文件的路径
NSString *docPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
NSString *path = [docPath stringByAppendingPathComponent:@"YLNote.sqlite"];
NSLog(@"path = %@",path);
// 1..创建数据库对象
FMDatabase *db = [FMDatabase databaseWithPath:path];
// 2.打开数据库
if ([db open]) {
    // do something
    NSLog(@"Open database Success");
    NSString *table = @"CREATE TABLE IF NOT EXISTS t_student (id integer PRIMARY KEY AUTOINCREMENT, name text NOT NULL, age integer NOT NULL)";
    [db executeUpdate:table];
} else {
    NSLog(@"fail to open database");
}


```

# 4.数据库操作

在FMDB中，除查询以外的所有操作，都称为“更新”

- create（建表）
- drop （删表）
- insert （插入数据）
- update （修改数据）
- delete（删除数据）
- select（查询数据）

FMDB提供更新的接口有以下：

* executeUpdate:
* `executeUpdateWithFormat:`
* **executeUpdate:withParameterDictionary:**

# 5.实战

1. ######  定义一个基类(实现归档协议和copy协议)

2. 定义一个数据模型（继承自上述基类）

3. 定义一个数据库管理类（实现数据库初始化、建表、删表及数据增、删、改、查的接口）

4. 写一个demo使用以上类，完成数据库相关操作

    

