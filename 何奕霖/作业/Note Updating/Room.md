Room允许我们使用面向对象的思维来和数据库进行交互，无需再使用繁杂的SQL语句让项目整体代码变得混乱

**Room主要由Entity、Dao和Database这三部分组成**。

- **Entity**：用于定义封装实际数据的实体类，每个实体类都会在数据库中有一张对应的表，并且表中的列是根据实体类中的字段自动生成的。
- **Dao**：Dao是数据访问对象的意思，通常在这里对数据库的各项操作进行封装。
- **Database**：用于定义数据库中的关键信息，包括数据库的版本号、包含哪些实体类以及提供Dao层的访问实例。

## 一、前置准备

再build.gradle中添加依赖

```kotlin
plugins {
    ...
    id("kotlin-kapt")
}
...
dependencies {
    ...
    implementation("androidx.toom:room-runtime:2.1.0")
    kapt(libs.androidx.room.compiler)
    ...
}
```

## 二、代码实现

**1.声明实体类**

```kotlin
@Entity
data class Water(var goal: Int, var time: String, var totalVolume: Float, var gap: Int, var date: String) {

    @PrimaryKey(autoGenerate = true)
    var id: Int = 0
}
```

**2.新建Dao接口**

```kotlin
@Dao
interface WaterDao {

    @Insert
    fun insertWater(water: Water): Long

    @Update
    fun updateWater(water: Water)

    @Query("select * from Water")
    fun loadAllWater(): List<Water>

    @Query("select * from Water where date = :date")
    fun loadWaterByDate(date: String): List<Water>

    @Delete
    fun deleteWater(water: Water)

    @Query("delete from Water where date = :date")
    fun deleteWaterByDate(date: String)
}
```

**3.建立Database**

```kotlin
@Database(version = 1, entities = [Water::class])
abstract class WaterDatabase : RoomDatabase() {

    abstract fun waterDao(): WaterDao

    companion object {

        private var instance : WaterDatabase? = null

        @Synchronized
        fun getDatabase(context: Context): WaterDatabase {
            instance?.let {
                return it
            }
            return Room.databaseBuilder(context.applicationContext,
                WaterDatabase::class.java, "water_database")
                .build().apply {
                    instance = this
                }
        }
    }
}
```

至此我们就把Room的所有东西都定义好了。

**接下来就是对Room数据库的基本操作（CRUD）了。**

**1.添加记录**

```kotlin
if (topic == "water/volume") {
    val time = java.text.SimpleDateFormat("HH:mm", java.util.Locale.getDefault()).format(java.util.Date())
    val volume = msgStr.toFloatOrNull() ?: return
    val volumeRounded = String.format("%.1f", volume).toFloat()
    //更新WaterFragment的UI
    waterFragment?.let { fragment ->
        runOnUiThread {
            fragment.getAdapter().addRecord(WaterRecord(time, volume))
            fragment.totalVolume += volume
            fragment.updateProgressView(fragment.currentGoal, fragment.totalVolume)
            fragment.getBinding().waterRecyclerView.smoothScrollToPosition(0)
        }
    }
    //保存到数据库
    Thread {
        val db = WaterDatabase.getDatabase(this@MainActivity)
        val waterDao = db.waterDao()
        val todayDate = java.text.SimpleDateFormat("yyyy-MM-dd", java.util.Locale.getDefault()).format(java.util.Date())
        val water = Water(
            goal = waterFragment?.currentGoal ?: 0,
            time = time,
            totalVolume = volumeRounded,
            gap = waterFragment?.getCurrentGap() ?: 0,
            date = todayDate
        )
        waterDao.insertWater(water)
    }.start()
}
```

上面展示的是手机通过Mqtt通信协议接收到智能底座返回的喝水记录之后，将记录插入数据库的操作。

**2.删除记录**

```kotlin
Thread {                                                                            //清空今日数据库记录，方便重新开始导入喝水记录
    val db = WaterDatabase.getDatabase(requireContext())
    val waterDao = db.waterDao()
    waterDao.deleteWaterByDate(todayDate)
}.start()
```

上面展示的是点击设置喝水目标按钮重置所有喝水记录时，清空数据库方便后续插入记录。

**3.修改记录**

```kotlin
Thread {                                                                            //更新数据库中的目标记录
    val db = WaterDatabase.getDatabase(requireContext())
    val waterDao = db.waterDao()
    val todayRecords = waterDao.loadWaterByDate(todayDate)
    todayRecords.forEach { water ->                                                 //更新所有今日记录的目标和间隔
        water.goal = goalText	//重新输入的喝水目标
        water.gap = gapText		//重新输入的喝水间隔
        waterDao.updateWater(water)
    }
}.start()
```

上面展示的是重置喝水目标和喝水时间之后的修改数据库的操作。

**4.查询记录**

```kotlin
Thread {
    val calendar = Calendar.getInstance()
    calendar.add(Calendar.DATE, -1)
    val yesterday = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()).format(calendar.time)
    val db = WaterDatabase.getDatabase(this)
    val waterDao = db.waterDao()
    val yesterdayRecords = waterDao.loadWaterByDate(yesterday)
    val historyGoal = yesterdayRecords.lastOrNull()?.goal ?: 0
    val historyVolume = yesterdayRecords.sumOf { it.totalVolume.toDouble() }.toFloat()
    runOnUiThread {
        currentFragment.updateShareView(historyDialog, historyGoal, historyVolume)
        MaterialAlertDialogBuilder(this)
            .setView(historyDialog)
            .create()
            .show()
    }
}.start()
```

以上是查找昨日记录并加载到Dialog里的例子。

**5.升级数据库**

如果应用还没有正式发布，那数据库可以随意删除二不怕信息泄露，可以直接用fallbackToDestructiveMigration()方法简单粗暴地升级数据库。

```kotlin
Room.databaseBuilder(context.applicationContext, AppDatabase::class.java,"app_database")
	.fallbackToDestructiveMigration()
	.build()
	...
```

但是如果应用已经发布，随意删除数据库会很大程度上影响用户的使用体验，所以要老老实实升级数据库。

升级数据库有两种方法，一种是直接升级全部，另一种是添加了新的Column后进行升级。

（1）**直接升级**：

```kotlin
@Database(version = 2, entities = [Water::class, person::class])
abstract class WaterDatabase : RoomDatabase() {

    abstract fun waterDao(): WaterDao
    
    abstract fun personDao(): PersonDao

    companion object {
        
        var MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("create table Person (id integer primary key autoincrement not null, name text not null)")
            }
        }

        private var instance : WaterDatabase? = null

        @Synchronized
        fun getDatabase(context: Context): WaterDatabase {
            instance?.let {
                return it
            }
            return Room.databaseBuilder(context.applicationContext,
                WaterDatabase::class.java, "water_database")
                .addMigrations(MIGRATION_1_2)
            	.build().apply {
                    instance = this
                }
        }
    }
}
```

以上就是在Database里添加了一张新的表，就要用到建表语句了。同时也要修改数据库的版本号，以及声明Migration。

（2）**添加Column**：



```kotlin
@Database(version = 2, entities = [Water::class, person::class])
abstract class WaterDatabase : RoomDatabase() {

    abstract fun waterDao(): WaterDao

    companion object {

        var MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("alter table Water add column weather text not null default 'unknown'")
            }
        }

        private var instance : WaterDatabase? = null

        @Synchronized
        fun getDatabase(context: Context): WaterDatabase {
            instance?.let {
                return it
            }
            return Room.databaseBuilder(context.applicationContext,
                WaterDatabase::class.java, "water_database")
                .addMigrations(MIGRATION_1_2)
                .build().apply {
                    instance = this
                }
        }
    }
}
```
以上就是只在原来的表的基础上只增加一列的用法。