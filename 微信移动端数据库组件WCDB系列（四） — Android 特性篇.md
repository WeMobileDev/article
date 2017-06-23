# 微信移动端数据库组件WCDB系列（四） — Android 特性篇

微信的移动端数据库组件 WCDB 已经正式开源了，有关注的小伙伴可能已经用上了。如果还没用上，
可以翻到文末关注我们的 GitHub 和公众号其他文章。

之前我们已经发过几篇 iOS 和修复的文章，Android 由于接口跟系统几乎一样，相信大家都比较熟悉，
不熟悉用法也可以到 Android Developer 官网看一下。但是，我们也有一些特色功能和优化大家可能不容易注意到，
现在就单独拿出来说说。

# 加密接口

WCDB 使用了 SQLCipher 的 C 层库，但没有直接使用 SQLCipher Android 的封装层。SQLCipher Android
封装层中很多设置需要手写 PRAGMA 语句实现，比如设置 KDF 迭代次数（兼容老版本 SQLCipher DB）、设置
Page Size 等操作。

```java
private static SQLiteDatabaseHook hook = new SQLiteDatabaseHook() {
    @Override
    public void postKey(SQLiteDatabase db) {
        db.rawExecSQL("PRAGMA kdf_iter = 4000;");
        db.rawExecSQL("PRAGMA cipher_page_size = 1024;");
    }

    @Override
    public void preKey(SQLiteDatabase db) {
        // ......
    }
};
SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(path, "password", 
        null /*factory*/, hook);
```

对于开发者来说，这需要了解 SQLCipher 底下的 PRAGMA 指令，更重要的是要搞清楚这些指令正确的调用顺序。
哪些是需要在设置 key 之前执行的？哪些是只有设置了 key 之后才生效的？开发者往往必须仔细查阅 SQLCipher
的文档来了解这些细节。

WCDB 对这个部分做了改进，封装了 `SQLiteCipherSpec` 用于设置加密参数，设置好了传给 `SQLiteDatabase`
工厂方法就好了，不需要考虑 PRAGMA 语法和调用顺序。

```java
SQLiteCipherSpec cipher = new SQLiteCipherSpec()
        .setPageSize(1024)
        .setKDFIteration(4000);
SQLiteDatabase db = SQLiteDatabase.openOrCreateDatabase(path, "password".getBytes(),
        cipher, null /*factory*/);
```

使用 `SQLiteCipherSpec` 另一个好处是，同样的结构可以传给 `RepairKit` 用于恢复损坏 DB，不需要两套
接口了。由于 RepairKit 底层不使用 PRAGMA，原来 hook 的形式不能满足需要。

另外，WCDB 将 `String` 类型的密码改为 `byte[]` 类型，可以支持非打印字符作为密码（比如 
`hash(user id)` 方式），原来字符类型密码只要转换为 UTF-8 的 byte 数组即可，和 SQLCipher Android
兼容。

# 数据迁移

SQLCipher 提供了 `sqlcipher_export` SQL 函数用于导出数据到挂载的另一个 DB，可以用于数据迁移。
但这个函数用于 Android 的 `SQLiteOpenHelper` 并不方便。

`SQLiteOpenHelper` 主要帮助开发者做 Schema 版本管理，通过它打开 SQLite 数据库，会读取 `user_version`
字段来判断是否需要升级，并调用子类实现的 `onCreate`、`onUpgrade` 等接口来完成创建或升级操作。
`sqlcipher_export` 由于是导出而非导入，就跟 `onCreate` 等接口不搭了，因为要关闭原来的 DB，
打开老的 DB，执行 export 到新 DB，再重打开。

为了方便使用，WCDB 就做了扩展，将 `sqlcipher_export` 扩展为可以接受第二个参数表示从哪里导出，
从而实现了导入。

```sql
ATTACH DATABASE '/path/to/old/db' AS old;

-- 将 old 的数据导出到 main，也就是从 old 导入数据
SELECT sqlcipher_export('main', 'old');
```

如此就可以不关闭原来的数据库实现数据导入，可以兼容 `SQLiteOpenHelper` 的接口了。详细可以看
我们的 Sample。

# 设备锁定

WCDB 在加密的基础上加上的一个小功能，

# 全文搜索分词器与动态 ICU 加载

WCDB Android 自带了一个 FTS3/4 分词器，名为 `mmicu`，用于实现 [SQLite 全文搜索][sqlite-fts]。
分词器的使用与 SQLite 自带的 `simple`、`icu` 等分词器一样，创建虚拟表的时候带上名字即可：

```java
SQLiteDatabase db = getDB();
db.execSQL("CREATE VIRTUAL TABLE message USING fts4 (content, tokenize=mmicu);")

```

MMICU 分词器与官方 ICU 分词器类似，但对中文（象形文字）分词以及 ICU 库加载做了特殊处理。
ICU 对中文的分词是基于词库的，Android 系统不同版本会附带不同版本的 ICU，捎带不同版本的中文
词库，当然也会带来不同的分词结果，这个对于统一产品体验是非常不利的。

另外，ICU 自带的中文词库并非非常完整，组词效果也一般，但若自带一个完整好用的词库，
又需要非常大的空间，这个空间会体现在 APK 体积上。最终，我们做了折中，
**中文字全部单字成词，其他文字则使用 ICU 默认规则。**

ICU 还有一个严重的问题是动态库和自带的数据文件体积很大，超过 10MB，编译进 APK 里相当不划算，
最好能直接加载系统自带的 ICU 库。但加载系统库有另一个障碍：**ICU 库不同版本会在函数名称后面
带上版本号后缀**，直接编译时连接行不通。

为了克服这个障碍，WCDB 做了一个兼容层 `icucompat`，通过系统带的数据文件推断 ICU 版本，
通过 `dlopen` 动态加载不同的符号名称，然后通过宏来模拟直接调用方便开发。最终实现效果便是
**在不需要自带 ICU 库的前提下使用 ICU 库的断词、归一化等功能**，为最终 APK 包省下 10MB 
以上空间。

有了 ICU 兼容层，要实现 Android 框架自带的 ICU 相关功能就简单了，比如 `LOCALIZED` 排序。
但是，WCDB 目前没有接入（主要是没有相关需求），有这方面需要的话，可以到我们的 GitHub
提 Issue 哦。

# 日志重定向与性能监控

SQLite 和 WCDB 框架在运行中会产生日志，这些日志默认会打印到系统日志（logcat），但这可能不是
所有开发者都希望的行为。比如担心日志里带有敏感信息，直接输出到系统不妥，或者希望将日志写到文件
用于上报和分析，WCDB 提供接口来完成日志重定向。

```java
// 不打印任何日志
Log.setLogger(Log.LOGGER_NONE);

// 或者使用自定义日志逻辑
Log.setLogger(new Log.LogCallback() {
    @Override
    public void println(int priority, String tag, String msg) {
        // 处理日志
    }
});
```

要实现高性能日志持久化，可以考虑使用我们 mars 里面的 xlog 组件哦。

WCDB 还提供了性能监控接口 `SQLiteTrace`，实现接口并绑定到 `SQLiteDatabase` 可以在每次
执行 SQL 语句或连接池拥堵的时候得到回调。

```java
SQLiteTrace myTrace = new SQLiteTrace() {
    @Override
    public void onSQLExecuted(SQLiteDatabase db, String sql, int type, long time) {
        // 每次执行完一条 SQL 语句时回调
        Log.i(TAG, "trace", "Execute " + sql + " took " + time + "milliseconds");
    }

    @Override
    public void onConnectionPoolBusy(SQLiteDatabase db, String sql, List<String> requests,
            String message) {
        // 等待连接池超过3秒时回调，一般是因为别的操作占用连接池全部连接
        Log.i(TAG, "trace", "SQL: " + sql + " is waiting for execution. Message: " + message);
    }

    @Override
    public void onDatabaseCorrupted(SQLiteDatabase db) {
        // 数据库损坏时回调，只有使用默认 DatabaseErrorHandler 才会回调，
        // 若使用自定义 DatabaseErrorHandler 可直接在里面执行对应逻辑
        Log.i(TAG, "trace", "Database corrupted!");
    }
}

SQLiteDatabase db = getDB();
db.setTraceCallback(myTrace);
```

`SQLiteDatabase` 也开放了 `dump` 方法，可以打印出数据库的当前状态，包括连接池内所有连接
被持有的状态以及最近执行的 SQL 语句和耗时，对排查性能和死锁问题也有很大帮助。

# 优化 Cursor 实现

在 WCDB 发布时，我们的一篇文章上提到 Cursor 实现优化。对于 **查询获取 Cursor → 遍历 → 关闭**
这种简单的场景，我们通过 `SQLiteDirectCursor` 直接操作 SQLite 底层的查询，避免 CursorWindow
的重复分配带来的损耗。

可以看一下我们发布时的文章。

需要注意的是 Direct Cursor 未关闭前会占用一个数据库连接，使用完需要尽快关闭，否则会一直占用
造成别的线程无法请求到连接。遍历 Cursor 过程中同一线程不做其他 DB 操作，遍历完关闭，配合 WAL 
使用，是最佳实践。

# 关注我们

到 GitHub 关注 WCDB 的最新动态。

使用上的疑问（包括 iOS 和 Android）可以加QQ群讨论哦。
[点击链接加入群【WCDB技术交流群】](https://jq.qq.com/?_wv=1027&k=4AnuKCu)
（加群备注 WCDB 讨论）

[sqlite-fts]: https://sqlite.org/fts3.html
