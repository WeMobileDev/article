# 背景

WCDB开源至今已两个月有余，我们在不断迭代功能、完善文档的同时，也与来自世界各地的开发者进行交流，帮助他们更快地了解、掌握WCDB。这其中，也不乏使用FMDB的开发者。他们正准备将项目的数据库模块改为WCDB。

对于一个已经上线运行的项目，数据库这类基础组件与业务的耦合通常较多，迁移有一定工作量的。因此，开发者通常会做很多预研，以确定是否进行迁移。

WCDB在Github的wiki上提供了专门的教程，帮助使用FMDB的开发者进行迁移。同时，也希望通过本文全面地介绍WCDB和FMDB在使用方式、性能等方面的差异，以及迁移中可能遇到的问题，帮助开发者决定是否进行迁移。

# 平滑迁移

#### 文件格式

由于FMDB和WCDB都基于SQLite，因此两者在数据库的文件格式上一致。用FMDB创建、操作的数据库，可以直接通过WCDB打开、使用。因此开发者无需做额外的数据迁移。

#### 表结构

WCDB提供了ORM的功能，将类的属性绑定到数据库表的字段。在日常实践中，类的属性名和表的字段名通常不一致。因此，WCDB提供了`WCDB_SYNTHESIZE_COLUMN(className, propertyName, columnName)`宏，用于映射属性名。

对于

- 表：`CREATE TABLE message (db_id INTEGER, db_content TEXT)`

- 类：

  ```objective-c
  //Message.h
  @interface Message : NSObject

  @property int localID;
  @property(retain) NSString *content;

  @end

  //Message.mm
  @implementation Message

  @end
  ```

这里表字段都加了"db_"的前缀，并且使用了不一样的字段名。通过WCDB的ORM，可以映射为

```objective-c
//Message.h
@interface Message : NSObject <WCTTableCoding>

@property int localID;
@property(retain) NSString *content;

WCDB_PROPERTY(localID)
WCDB_PROPERTY(content)

@end

//Message.mm
@implementation Message

WCDB_IMPLEMENTATION(Message)
WCDB_SYNTHESIZE_COLUMN(Message, localID, "db_id")
WCDB_SYNTHESIZE_COLUMN(Message, content, "db_content")
  
@end
```

通过`WCDB_SYNTHESIZE_COLUMN`宏映射后，WCDB同样能兼容FMDB的表结构，开发者也不需要做数据迁移。

因此，开发者可以平滑地从FMDB迁移到WCDB。

# 性能比较

对于已经上线运行的项目，解决性能瓶颈会是一个常见的迁移理由。相较于FMDB直白的封装，WCDB上到OC层的ORM，下到SQLite源码，都做了各类性能优化。

为了验证优化效果，我们提供了benchmark，并将性能测试结果和测试代码上传到了Github。同时，benchmark中也加入了FMDB的测试代码，用于横向比较。

以下性能测试均为WAL模式、缓存大小2000字节、页大小4 kb：

- `PRAGMA cache_size=-2000`
- `PRAGMA page_size=4096`
- `PRAGMA journal_mode=WAL`

测试数据均为含有一个整型和一个二进制数据的表：`CREATE TABLE benchmark(key INTEGER, value BLOB)`，二进制数据长度为100字节。

#### 读操作性能测试

![](assets/migrate_to_wcdb/baseline_read.png)

#### 写操作性能测试

![](assets/migrate_to_wcdb/baseline_write.png)

#### 批量写操作性能测试 (事务)

![](assets/migrate_to_wcdb/baseline_batch_write.png)

对于读操作，SQLite速度很快，因此封装层的消耗占比较多。FMDB只做了最简单的封装， 而WCDB还包括ORM、WINQ等操作，因此执行的指令会比FMDB多，从而导致性能**劣于FMDB 5%**。

而写操作通常是性能的瓶颈，WCDB对其做了许多针对性的优化，使得写操作和批量写操作的性能分别**优于FMDB  28% 和 180%**。

#### 多线程读并发性能测试

![](assets/migrate_to_wcdb/multithread_read_read.png)

#### 多线程读写并发性能测试

![](assets/migrate_to_wcdb/multithread_read_write.png)

#### 多线程写并发性能测试

![](assets/migrate_to_wcdb/multithread_write_write.png)

在多线程读操作的测试中，WCDB多线程并发的优势，将读操作的性能劣势拉了回来，使得最终结果与FMDB**基本持平**，而多线程读写操作性能则优于FMDB **62%** 。

在多线程写操作的测试中，FMDB直接返回错误`SQLITE_BUSY`，无法完成。

#### 初始化性能测试

![](assets/migrate_to_wcdb/initialization.png)

SQLite连接的初始化速度会随着数据库内表的数量增加而逐渐上升，WCDB也针对这个场景做了优化。相较于没有优化的FMDB，WCDB 有**107%** 的性能优势。

# 易用性比较

与已经上线运行项目不同，新项目更关注开发的效率。此时数据库的易用和便捷更重要。

对于等价的功能，WCDB所需的代码量往往会比FMDB少很多。而更少的代码量通常意味着更快的开发效率和更少的错误。

## 基础操作

ORM是现代客户端数据库比较普遍的功能。CoreData、Realm都支持ORM，WCDB也不例外。

FMDB因其直白的封装，没有提供该功能。但在设计数据库表时，开发者通常会对数据进行建模。因此开发者只需将已有建模用WCDB的ORM表达出来即可。

对于在FMDB的一组定义：

* 表：`CREATE TABLE message (localID INTEGER PRIMARY KEY AUTOINCREMENT, content TEXT, createTime INTEGER, modfiedTime INTEGER)`
* 索引：`CREATE INDEX message_index ON message(createTime)`
* 类：

```objective-c
//Message.h
@interface Message : NSObject

@property int localID;
@property(retain) NSString *content;
@property(retain) NSDate *createTime;
@property(retain) NSDate *modifiedTime;

@end
//Message.mm
@implementation Message

@end
```

WCDB需要对其建模，可以定义为

```objective-c
//Message.h
@interface Message : NSObject <WCTTableCoding>

@property int localID;
@property(retain) NSString *content;
@property(retain) NSDate *createTime;
@property(retain) NSDate *modifiedTime;

WCDB_PROPERTY(localID)
WCDB_PROPERTY(content)
WCDB_PROPERTY(createTime)
WCDB_PROPERTY(modifiedTime)

@end

//Message.mm
@implementation Message

WCDB_IMPLEMENTATION(Message)
WCDB_SYNTHESIZE(Message, localID)
WCDB_SYNTHESIZE(Message, content)
WCDB_SYNTHESIZE(Message, createTime)
WCDB_SYNTHESIZE_COLUMN(Message, modifiedTime, "db_modifiedTime")

WCDB_PRIMARY_AUTO_INCREMENT(Message, localID)
WCDB_INDEX(Message, "_index", createTime)

@end
```

其中：

* `WCDB_IMPLEMENTATION(className)`用于定义进行绑定的类
* `WCDB_PROPERTY(propertyName)`和`WCDB_SYNTHESIZE(className, propertyName)`用于声明和定义字段。
* `WCDB_PRIMARY_AUTO_INCREMENT(className, propertyName)`用于定义主键且自增。
* `WCDB_INDEX(className, indexNameSubfix, propertyName)`用于定义索引。

虽然WCDB多了一步ORM的操作，但这是一劳永逸的，并且会给我们后续的使用带来很大的便利。

经过ORM的类，大部分操作都只需要一行代码即可完成。Talk is cheap，直接看代码对比：

### 查询操作

```objective-c
/*
 FMDB Code
 */
FMResultSet *resultSet = [fmdb executeQuery:@"SELECT * FROM message"];
NSMutableArray<Message *> *messages = [[NSMutableArray alloc] init];
while ([resultSet next]) {
    Message *message = [[Message alloc] init];
    message.localID = [resultSet intForColumnIndex:0];
    message.content = [resultSet stringForColumnIndex:1];
    message.createTime = [NSDate dateWithTimeIntervalSince1970:[resultSet doubleForColumnIndex:2]];
    message.modifiedTime = [NSDate dateWithTimeIntervalSince1970:[resultSet doubleForColumnIndex:3]];
    [messages addObject:message];
}
```

```objective-c
/*
 WCDB Code
 */
NSArray<Message *> *messages = [wcdb getAllObjectsOfClass:Message.class fromTable:@"message"];
```

### 插入操作

```objective-c
/*
 FMDB Code
 */
[fmdb executeUpdate:@"INSERT INTO message VALUES(?, ?, ?, ?)", @(message.localID), message.content, @(message.createTime.timeIntervalSince1970), @(message.modifiedTime.timeIntervalSince1970)];
```

```objective-c
/*
 WCDB Code
 */
[wcdb insertObject:message into:@"message"];
```

可以看到，

* 对于查询操作，FMDB需要进行很多拼装组合，而WCDB只需要一行代码就能完成。
* 对于插入操作，FMDB也只用了一行代码，但其需要将property逐个拆分为最基本的类型。而WCDB所需要关注的只有object和表名两个参数。

## 数据库升级

SQLite的数据库升级一直是一个比较繁杂的问题。

通常的做法是，开发者自行定义一个版本号，并保存下来。数据库创建时每次检查版本号，若版本号较低，则对其字段进行升级，并更新版本号。但在多个版本的增增减减之后，版本的处理逻辑会越来越复杂，甚至可能弄错表内哪些字段是新增的，哪些是废弃的。

WCDB将数据库升级和ORM结合起来，对于需要增删改的字段，只需直接在ORM层面修改，并再次调用`createTableAndIndexesOfName:withClass:`接口即可自动升级。以下是一个数据库升级的例子。

```objective-c
//Message.h
@interface Message : NSObject <WCTTableCoding>

@property int localID;
@property(assign) const char *content;//NSString *content;  
//@property(retain) NSDate *createTime;
@property(retain) NSDate *aNewModifiedTime;
@property(retain) NSDate *aNewProperty;

WCDB_PROPERTY(localID)
WCDB_PROPERTY(content)
//WCDB_PROPERTY(createTime)
WCDB_PROPERTY(modifiedTime)
WCDB_PROPERTY(newProperty)

@end

//Message.mm
@implementation Message

WCDB_IMPLEMENTATION(Message)
WCDB_SYNTHESIZE(Message, localID)
WCDB_SYNTHESIZE(Message, content)
//WCDB_SYNTHESIZE(Message, createTime)
WCDB_SYNTHESIZE_COLUMN(Message, aNewModifiedTime, "modifiedTime")
WCDB_SYNTHESIZE(Message, aNewProperty)

WCDB_PRIMARY_AUTO_INCREMENT(Message, localID)
//WCDB_INDEX(Message, "_index", createTime)
WCDB_UNIQUE(Message, aNewModifiedTime)
WCDB_INDEX(Message, "_newIndex", aNewProperty)

@end
```

```objective-c
WCTDatabase* db = [[WCTDatabase alloc] initWithPath:path];
[db createTableAndIndexesOfName:@"message" withClass:Message.class];
```

#### 删除字段

如例子中的`createTime`字段，删除字段只需直接将ORM中的定义删除即可。

#### 增加字段

如例子中的`aNewProperty`字段，增加字段只需直接添加ORM的定义即可。

#### 修改字段类型

如例子中的`content`字段，字段类型可以直接修改，但需要确保新类型与旧类型兼容；

#### 修改字段名称

如例子中的`aNewModifiedTime`，字段名称可以通过`WCDB_SYNTHESIZE_COLUMN(className, propertyName, columnName)`重新映射。

#### 增加约束

如例子中的`WCDB_UNIQUE(Message, aNewModifiedTime)`，新的约束只需直接在ORM中添加即可。

#### 增加索引

如例子中的`WCDB_INDEX(Message, "_newIndex", aNewProperty)`，新的索引只需直接在ORM添加即可。

## 多线程操作

WCDB与FMDB都支持多线程操作。

在FMDB内，当开发者需要进行多线程操作时，需要使用另外一个类`FMDatabasePool`来进行操作。

而WCDB基础的CRUD接口都支持多线程，因此开发者不需要额外关心线程安全的问题。同样的，WCDB多线程使用的代码量也比FMDB少得多。

```objective-c
/*
 FMDB Code
 */
//thread-1 read
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    [fmdbPool inDatabase:^(FMDatabase *_Nonnull db) {
        NSMutableArray *messages = [[NSMutableArray alloc] init];
        FMResultSet *resultSet = [db executeQuery:@"SELECT * FROM message"];
        while ([resultSet next]) {
            Message *message = [[Message alloc] init];
            message.localID = [resultSet intForColumnIndex:0];
            message.content = [resultSet stringForColumnIndex:1];
            message.createTime = [NSDate dateWithTimeIntervalSince1970:[resultSet doubleForColumnIndex:2]];
            message.modifiedTime = [NSDate dateWithTimeIntervalSince1970:[resultSet doubleForColumnIndex:3]];
            [messages addObject:message];
        }
        //...
    }];
});
//thread-2 write
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    [fmdbPool inDatabase:^(FMDatabase *_Nonnull db) {
		[db beginTransaction]
        for (Message *message in messages) {
            [db executeUpdate:@"INSERT INTO message VALUES(?, ?, ?, ?)", @(message.localID), message.content, @(message.createTime.timeIntervalSince1970), @(message.modifiedTime.timeIntervalSince1970)];
        }
        if (![db commit]) {
            [db rollback];
        }
    }];
});
```

```objective-c
/*
 WCDB Code
 */
//thread-1 read
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    NSArray *messages = [wcdb getAllObjectsOfClass:Message.class fromTable:@"message"];
    //...
});
//thread-2 write
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    [wcdb insertObjects:messages into:@"message"];
});
```

# 功能完整性比较

#### 加密

WCDB基于SQLCipher提供了加密功能

```objective-c
[database setCipherKey:password];
```

#### 统计

WCDB内提供统计的接口注册获取数据库操作的SQL、性能、错误等，开发者可以将这些信息打印到日志或上报到后台，以调试或统计

```objective-c
//Error Monitor
[WCTStatistics SetGlobalErrorReport:^(WCTError *error) {
	NSLog(@"[WCDB]%@", error);
}];

//Performance Monitor
[WCTStatistics SetGlobalPerformanceTrace:^(WCTTag tag, NSDictionary<NSString *, NSNumber *> *sqls, NSInteger cost) {
	NSLog(@"Database with tag:%d", tag);
	NSLog(@"Run :");
	[sqls enumerateKeysAndObjectsUsingBlock:^(NSString *sqls, NSNumber *count, BOOL *) {
		NSLog(@"SQL %@ %@ times", sqls, count);
	}];
    NSLog(@"Total cost %ld nanoseconds", (long)cost);
}];

//SQL Execution Monitor
[WCTStatistics SetGlobalSQLTrace:^(NSString *sql) {
	NSLog(@"SQL: %@", sql);
}];
```

#### 修复

WCDB提供了数据库修复工具，以应对数据库损坏无法使用的极端情况。

```objective-c
WCTDatabase *recover = [[WCTDatabase alloc] initWithPath:recoverPath];
[database close:^{
  [recover recoverFromPath:path withPageSize:pageSize backupCipher:backupCipher databaseCipher:databaseCipher];
}];
```

# 总结

与FMDB对比，WCDB使得开发者可以写更少的代码，但能获得更高的性能。开发者不需要额外关注数据库升级和多线程操作的问题。同时，WCDB还提供了加密、统计、修复等功能。

因此，对于新项目，我们推荐使用WCDB，以获得更好的性能和开发效率。对于已经上线、稳定运行的项目，如果遇到性能瓶颈，或者对加密、统计、修复等功能有需求，我们建议参考Github上的文档进行迁移。

后续我们还将加入更多的功能，欢迎来Github关注我们。