---
title: Android Architecture Components -- Room
categories: Android
comments: true
tags: [Architecture Components, Room]
description: 本文介绍 Android 数据库架构组件 Room 的基本使用
date: 2017-12-2 10:00:00
---

## 概述

[官方文档](https://developer.android.com/training/data-storage/room/index.html)
首先来介绍一下 Google 推出的 Android 架构组件（Architecture Components），测试版组件 Google 2017 IO 大会上发布，1.0 稳定版已经于2017年11月07正式推出。为 App 开发构架提供指南，并为常见任务，如生命周期管理、数据持久性等提供了一系列库。有了这些基础组件的帮助，开发者能够使用更少的样板代码写出模块化 App，将精力用于创新而非重复体力劳动。

Room 相当于一个 ORM 框架，使用他能帮助程序员方便快捷地实现数据存储功能。

## 使用

### 组件介绍

 - 数据库（Database）：你可以使用该组件创建数据库的持有者。该注解定义了实体列表，该类的内容定义了数据库中的 DAO 列表。这也是访问底层连接的主要入口点。注解类应该是抽象的并且扩展自 `RoomDatabase`。在运行时，你可以通过调用 `Room.databaseBuilder()` 或者 `Room.inMemoryDatabaseBuilder()` 获取实例。
 - 实体（Entity）：这个组件代表了持有数据库表记录的类。对每种实体来说，创建了一个数据库表来持有所有项。你必须通过 Database中 的 `entities` 数组来引用实体类。实体的每个成员变量都被持久化在数据库中，除非你注解其为 `@Ignore`。
 - 数据访问对象（DAO）：这个组件代表了作为 DAO 的类或者接口。DAO 是 Room 的主要组件，负责定义访问数据库的方法。被注解 `@Database` 的类必须包含一个无参数的抽象方法并返回被 `@Dao` 注解的类型。当编译时生成代码时，Room 会创建该类的实现。

### 添加依赖

在 build.gradle 中添加一下依赖：

```
allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }
    }
}

dependencies {
    implementation "android.arch.persistence.room:runtime:1.0.0"
    annotationProcessor "android.arch.persistence.room:compiler:1.0.0"
}
```

## 数据库

```
@Database(entities = {User.class}, version = 1, exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

## 实体

```
@Entity(tableName = "user")
public class User {
    @PrimaryKey
    private int uid;
    @ColumnInfo(name = "first_name")
    private String firstName;
    @ColumnInfo(name = "last_name")
    private String lastName;
    @Ignore
    Bitmap picture;//不进行持久化

    public int getUid() {
        return uid;
    }

    public void setUid(int uid) {
        this.uid = uid;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public Bitmap getPicture() {
        return picture;
    }

    public void setPicture(Bitmap picture) {
        this.picture = picture;
    }

}
```

当一个类被 `@Entity` 注解，并被 `@Database` 注解的 `entities` 属性引用时，Room 为这个实体在数据库中创建一个表。
默认情况，Room 为实体类的每个成员变量创建一个列。如果一个实体类的某个成员变量不想被持久化，你可以使用 `Ignore` 注解标记。
为了持久化成员变量，Room 必须可以访问它。你可以使成员变量是公共的，或者提供 `getter和setter` 方法。如果你使用 `getter/setter` 方法，请记住它们在 Room 中遵循 Java Beans 的概念。
默认情况下，Room 使用类名作为数据库表的表名。如果你想要数据库表有一个其他的名字，设置 `@Entity` 注解的 `tableName` 属性即可。
注意：SQLite中的表名是大小写敏感的。
和 `tablename` 属性相似，Room 使用成员名作为列名，如果你想要改变类名，在成员上添加 `@ColumnInfo` 注解并设置 `name` 即可。

### 数据访问对象

```
@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List<User> getAll();

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    List<User> loadAllByIds(int[] userIds);

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND "
            + "last_name LIKE :last LIMIT 1")
    User findByName(String first, String last);

    @Insert
    void insertAll(User... users);

    @Delete
    void delete(User ... user);

    @Update
    void update(User... user);

}
```

### 创建数据库

```
    public DBManager(Context context) {
        mDatabase = Room.databaseBuilder(context.getApplicationContext(),
                AppDatabase.class, "users").build();
    }
```

编译过程中如果出现下面的警告：

```
Error:(11, 17) 警告: Schema export directory is not provided to the annotation processor so we cannot export the schema. You can either provide `room.schemaLocation` annotation processor argument OR set exportSchema to false.
```

解决办法：
在 `AppDatabase` 中添加 `exportSchema = false`：
```
@Database(entities = {User.class}, version = 1, exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

### 数据库升级

比如我们想在数据库中新加一个字段：

```
public class User {
    ...
    @ColumnInfo(name = "is_male")
    private boolean isMale;

    private String address;
    ...

    public boolean isMale() {
        return isMale;
    }

    public void setMale(boolean male) {
        isMale = male;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```
Room允许你编写 `Migration` 类来保护用户数据。每个 `Migration` 类指定一个 startVersion 和 endVersion。在运行时，Room 运行每个 `Migration` 类的 `migrate()` 方法，使用正确的顺序迁移至数据库的更新版本。
然后在创建数据库的时候使用 `addMigrations()` 方法：
`AppDatabase` 中的 `version` 必须增加：

```
@Database(entities = {User.class}, version = 3, exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```
否则会报错：
```
E AndroidRuntime: java.lang.IllegalStateException: Room cannot verify the data integrity. Looks like you've changed schema but forgot to update the version number. You can simply fix this by increasing the version number.
E AndroidRuntime: 	at android.arch.persistence.room.RoomOpenHelper.checkIdentity(RoomOpenHelper.java:119)
E AndroidRuntime: 	at android.arch.persistence.room.RoomOpenHelper.onOpen(RoomOpenHelper.java:100)
```

```
    public DBManager(Context context) {
        mDatabase = Room.databaseBuilder(context.getApplicationContext(),
                AppDatabase.class, "users").addMigrations(MIGRATION_1_2,  MIGRATION_2_3).build();
    }

    static final Migration MIGRATION_1_2 = new Migration(1, 2) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("ALTER TABLE user ADD is_male INTEGER NOT NULL DEFAULT 0");
        }
    };

    static final Migration MIGRATION_2_3 = new Migration(2, 3) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("ALTER TABLE user ADD address TEXT");
        }
    };
```

这里添加 `boolean` 型的字段，SQL 语句必须设置为 `NOT NULL`，而且要设置默认值，否则会报错。

## 参考文档

http://www.jianshu.com/p/587f48dccf0a
https://developer.android.com/topic/libraries/architecture/room.html

