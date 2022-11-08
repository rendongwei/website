---
title: "Android-Room"
date: "2022-11-07"
categories:
    - "Android"
tags:
    - "Android"
    - "Android框架"
toc: true
indent: false
original: true
draft: false
---

### 前言

room是官方推出的数据库映射框架。在没有room之前，比较出名的就是greendao库。既然官方都出了，还是用官方的吧。
使用room或者然后第三方框架，最好使用android 4.2以及之后的版本。因为这些新版本支持Database Inspector功能。也就是直接查看数据库的功能，以前的版本只能导出查看非常的麻烦。

可以通过`View->Tool Window->Database Inspector`打开这个功能。
在一些新版本中，这个功能被移动到App Inspection里面。
同时需要SDK26及以上版本。

### 一、添加依赖

```gradle
def room_version="2.4.2"
implementation "androidx.room:room-runtime:$room_version"
annotationProcessor "androidx.room:room-compiler:$room_version"
//implementation "androidx.room:room-rxjava2:$room_version"
//implementation "androidx.room:room-rxjava3:$room_version"
//implementation "androidx.room:room-guava:$room_version"
//testImplementation "androidx.room:room-testing:$room_version"
//implementation "androidx.room:room-paging:2.5.0-alpha01"
```

### 二、创建数据库

#### 1. 创建映射实体类

创建实体类，这个实体类就是数据库映射的接受类，对应的是数据库的user表，可以将数据库的表结构映射成Bean。使用注解标记相应的功能，通过名字就可以非常清楚的知道。

```kotlin
// 表结构实体
@Entity
class User {
    // 主键
    @PrimaryKey
    var uid : Int? = null
    
    // 列/字段信息
    @ColumnInfo(name = "first_name")
    var firstName : String? = null
    
    // 列/字段信息
    @ColumnInfo(name = "last_name")
    var lastName : String? = null
    
}
```

#### 2.创建Dao接口

创建我们的Dao接口，这种实现方式和Retrofit是非常相似的。其实最感觉的就是SQL语句。这里我们先只关系getAll这个方法就行，语句是最简单的`SELECT * FROM user`。
对应的注解也是非常容易理解,分别对应增删改查。

```kotlin
interface UserDao {
    
    // 查询
    @Query("SELECT * FROM user")
    fun getAll() : MutableList<User> 
    
    // 条件查询
    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    fun getUserByIds(userIds : IntArray) : MutableList<User> 
    
    // 条件查询
    @Query("SELECT * FROM user WHERE first_name LIKE :first AND last_name LIKE :last LIMIT 1")
    fun getUserByName(first : String , last : String) : User
    
    // 插入
    @Insert
    fun insertAll(vararg users : User)
    
    // 删除
    @DELETE
    fun delete(user : User)
}
```

#### 3.创建数据库抽象接口

需要继承RoomDatabase，通过@Database指定需要创建的表。还可以指定版本。

```kotlin
@Database(entities = {User::class} , version = 1)
abstract class AppDatabase : RoomDatabase {
    abstract fun userDao() : UserDao
}
```

我们发现这个我们自己定义的AppDatabase还是抽象类，因为她的实现需要移除在Activity或者Fragment里面实现。通过下面的代码创建真正的实现类。到这一步，数据库，表和UserDao都已经创建好了，通过AppDatabase的userDao方法，我们就可以获取到UserDao的实现。然后就可以直接调用对应的方法实现增删改查。

```kotlin
val db : AppDatabase = Room.databaseBuilder(applicationContext,AppDatabase::class,"app.db").build()
```

下面是完整的测试代码，我们需要创建一个协程。我们插入两条数据到user表中。

```kotlin
GlobalScope.launch {
    val db : AppDatabase = Room.databaseBuilder(applicationContext,AppDatabase::class,"app.db").build()
    val userDao = db.userDao()
    val user = User(1,"Tom","Cat")
    val user1 = User(2,"Tom","Cruise")
    userDao.insertAll(user,user1)
    val users : MutableList<User> = userDao.getAll()
    Log.d("tag","users : $users")
}
```

### 三、Room的增删改查实现和细节

在数据库操作中其实最重要的是SQL语句，别的操作反而是次要的。

#### 1.插入操作

使用@Insert注解声明的Dao接口方法。

下面的代码分别可以插入一个和插入多条数据，只需要传入相应的User对象。这种方式是最常用的。

```kotlin
@Insert
fun insert(user : User)

@Insert
fun insertAll(vararg users : User)
```

这种方式是典型的数据库映射，也是room的这种ORM的精髓。

#### 2.删除操作

和插入操作类似。可以删除一条数据或者根据条件删除多条数据。

```kotlin
@DELETE
fun delete(user : User)

@Query("DELETE FROM user WHERE uid > 0")
fun deleteAll()
```

#### 3.查询操作

查询专门指SELECT语句，而不是@Query注解，这个注解可以做很多事情，前面我们看到，这个注解就是用来执行SQL语句的。

```kotlin
@Query
fun getAll() : MutableList<User>
```

#### 4.更新操作

可以通过room提供的@Update注解来实现，当然也可以通过@Query直接执行SQL语句。

```kotlin
@Update
fun update(user : User)

@Query("UPDATE user SET last_name = :lastName WHERE first_name = :firstName")
fun updateOne(firstName : String , lastName : String)
```



