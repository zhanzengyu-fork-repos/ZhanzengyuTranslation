**并发数据库访问**

假设你实现了自己的 [SQLiteOpenHelper](http://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html)。

```
public class DatabaseHelper extends SQLiteOpenHelper { ... }
```

现在你想要在多个线程中对数据库写入数据。

```
 // Thread 1
 Context context = getApplicationContext();
 DatabaseHelper helper = new DatabaseHelper(context);
 SQLiteDatabase database = helper.getWritableDatabase();
 database.insert(…);
 database.close();

 // Thread 2
 Context context = getApplicationContext();
 DatabaseHelper helper = new DatabaseHelper(context);
 SQLiteDatabase database = helper.getWritableDatabase();
 database.insert(…);
 database.close();
```

你将会在你的 *logcat* 中发现下面信息，并且你的其中一个改变不会写入数据库：

```
android.database.sqlite.SQLiteDatabaseLockedException: database is locked (code 5)
```

产生这个错误的原因是因为，每次你创建新的 `SQLiteOpenHelper` 对象，实际上你创建了新的数据库连接。如果你尝试从不同的连接同时对数据库写入数据，其中一个会失败。

>为了在多线程使用数据库，我们要确保只使用一个数据库连接。

让我们构造单例类 `DatabaseManager`，它会持有并返回单个 `SQLiteOpenHelper` 对象。

```
public class DatabaseManager {

    private static DatabaseManager instance;
    private static SQLiteOpenHelper mDatabaseHelper;

    public static synchronized void initializeInstance(SQLiteOpenHelper helper) {
        if (instance == null) {
            instance = new DatabaseManager();
            mDatabaseHelper = helper;
        }
    }

    public static synchronized DatabaseManager getInstance() {
        if (instance == null) {
            throw new IllegalStateException(DatabaseManager.class.getSimpleName() +
                    " is not initialized, call initialize(..) method first.");
        }

        return instance;
    }

    public synchronized SQLiteDatabase getDatabase() {
        return mDatabaseHelper.getWritableDatabase();
    }

}
```

在多个线程中对数据库写入数据，修改后的代码如下所示。

```
// In your application class
DatabaseManager.initializeInstance(new DatabaseHelper());

// Thread 1
DatabaseManager manager = DatabaseManager.getInstance();
SQLiteDatabase database = manager.getDatabase()
database.insert(…);
database.close();

// Thread 2
DatabaseManager manager = DatabaseManager.getInstance();
SQLiteDatabase database = manager.getDatabase()
database.insert(…);
database.close();
```

这会带来另一个奔溃。

```
java.lang.IllegalStateException: attempt to re-open an already-closed object: SQLiteDatabase
```

由于我们只使用了一个数据库连接，*Thread1* 和 *Thread2* 的 `getDatabase()` 方法都会返回同一个 `SQLiteDatabase` 对象实例。可能发生的场景是 *Thread1* 关闭了数据库，然而 *Thread2* 还在使用它。这也就是为什么我们会有 `IllegalStateException` 的奔溃的原因。

我们需要确保没有人正在使用数据库，这个时候我们才可以关闭它。[stackoveflow](http://stackoverflow.com/) 上有人推荐永远不要关闭你的 *SQLiteDatabase*。这会让你看到下面的 logcat 信息。所以我一点也不认为这是一个好的想法。

```
Leak found
Caused by: java.lang.IllegalStateException: SQLiteDatabase created and never closed
```
实战例子

一种可能的解决方案是使用计数器跟踪打开/关闭的数据库连接。
```
public class DatabaseManager {

    private AtomicInteger mOpenCounter = new AtomicInteger();

    private static DatabaseManager instance;
    private static SQLiteOpenHelper mDatabaseHelper;
    private SQLiteDatabase mDatabase;

    public static synchronized void initializeInstance(SQLiteOpenHelper helper) {
        if (instance == null) {
            instance = new DatabaseManager();
            mDatabaseHelper = helper;
        }
    }

    public static synchronized DatabaseManager getInstance() {
        if (instance == null) {
            throw new IllegalStateException(DatabaseManager.class.getSimpleName() +
                    " is not initialized, call initializeInstance(..) method first.");
        }

        return instance;
    }

    public synchronized SQLiteDatabase openDatabase() {
        if(mOpenCounter.incrementAndGet() == 1) {
            // Opening new database
            mDatabase = mDatabaseHelper.getWritableDatabase();
        }
        return mDatabase;
    }

    public synchronized void closeDatabase() {
        if(mOpenCounter.decrementAndGet() == 0) {
            // Closing database
            mDatabase.close();

        }
    }
}
```
然后如下所示来使用。
```
SQLiteDatabase database = DatabaseManager.getInstance().openDatabase();
database.insert(...);
// database.close(); Don't close it directly!
DatabaseManager.getInstance().closeDatabase(); // correct way
```

每当你需要使用数据库的时候你应该调用 `DatabaseManager` 类的 `openDatabase()` 方法。在这个方法里面，我们有一个计数器，用来表明数据库打开的次数。如果计数为 1，意味着我们需要创建新的数据库连接，否则，数据库连接已经建立。

对于 `closeDatabase()` 方法来说也是一样的。每次我们调用这个方法的时候，计数器在减少，当减为 0 的时候，我们关闭数据库连接。

现在你能够使用你的数据库并且确保是线程安全的。


