---
title: SqlBrite 使用指南和源码分析
categories: Android开源项目
comments: true
tags: [SqlBrite]
description: 介绍 SqlBrite 的使用方法
date: 2017-12-16 10:00:00
---

## 简介

SqlBrite 是对 SQLiteOpenHelper 和 ContentResolver 的轻量级封装，配合 Rxjava 使用，当数据库中的数据有更新时，它又可以自动通知观察者来更新界面。

github地址: https://github.com/square/sqlbrite

## 使用方法

引入依赖：

```
    implementation 'com.squareup.sqlbrite2:sqlbrite:2.0.0'
    implementation "io.reactivex.rxjava2:rxandroid:2.0.1"
```

### SqlBrite + SQLiteOpenHelper

创建一个 `DBOpenHelper` 继承自 `SQLiteOpenHelper`。

```
public class DBOpenHelper extends SQLiteOpenHelper {

}
```

DataSource 封装对数据的操作。

```
public class DataSource {
    private static DataSource INSTANCE;

    private Context mContext;
    private final BriteDatabase mDatabaseHelper;

    private DataSource(Context context) {
        DBOpenHelper dbHelper = DBOpenHelper.getInstance(context);
        SqlBrite sqlBrite = new SqlBrite.Builder().build();
        mDatabaseHelper = sqlBrite.wrapDatabaseHelper(dbHelper, Schedulers.io());
    }

    public static DataSource getInstance(Context context) {
        if (INSTANCE == null) {
            INSTANCE = new DataSource(context);
        }
        return INSTANCE;
    }

    // 添加数据
    public void saveData() {
        final ContentValues xiaoming = new ContentValues();

        xiaoming.put(StudentTable.Columns.NAME,"XiaoMing");
        xiaoming.put(StudentTable.Columns.GENDER,"male");
        xiaoming.put(StudentTable.Columns.GRADE,1);
        xiaoming.put(StudentTable.Columns.CLASS,1);
        xiaoming.put(StudentTable.Columns.COUNTRY,"China");
        //xiaoming.put(StudentTable.Columns.PROVINCE,null);
        xiaoming.put(StudentTable.Columns.SPECIALTY,"swimming");
        xiaoming.put(StudentTable.Columns.IS_BOARDER,true);

        mDatabaseHelper.insert(DBOpenHelper.StudentTAB.TABLE, xiaoming);
    }
    
    // 查询数据
    // 只要这里保持订阅，后面有任何的数据改动，比如增加、删除、更新数据，这里的订阅都会调用
    public void queryData() {
        String[] projection = new String []{DBOpenHelper.StudentTAB.NAME, DBOpenHelper.StudentTAB._ID};
        String sql = String.format("SELECT %s FROM %s",
                TextUtils.join(",", projection), DBOpenHelper.StudentTAB.TABLE);
        mDatabaseHelper.createQuery(DBOpenHelper.StudentTAB.TABLE, sql)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<SqlBrite.Query>() {
                    @Override
                    public void accept(SqlBrite.Query query) throws Exception {
                        Cursor cursor = query.run();
                        List<String> list = new ArrayList<String>();
                        while (cursor.moveToNext()) {
                            String username = cursor.getString(cursor.getColumnIndex(DBOpenHelper.StudentTAB.NAME));
                            int id = cursor.getInt(cursor.getColumnIndex(DBOpenHelper.StudentTAB._ID));
                            list.add(id + "--" + username);
                        }
                        cursor.close();
                    }
                });
    }
```

### SqlBrite + ContentResolver

如果是使用 ContentProvider，需要使用 ContentResolver 来创建一个 BriteContentResolver 对象：

```
private final BriteContentResolver mContentResolverHelper;

mContentResolverHelper = sqlBrite.wrapContentProvider(context.getContentResolver(), Schedulers.io());

    public void queryData() {
        mContentResolverHelper.createQuery(DemoProvider.AUTHORITY_URI.buildUpon().appendEncodedPath("common/get_info/James").build(),
                null,null,null,null,false)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<SqlBrite.Query>() {
                    @Override
                    public void accept(SqlBrite.Query query) throws Exception {
                        Cursor cursor = query.run();
                        List<String> list = new ArrayList<String>();
                        while (cursor.moveToNext()) {
                            String name = cursor.getString(cursor.getColumnIndex("name"));
                            int id = cursor.getInt(cursor.getColumnIndex("id"));
                            int age = cursor.getInt(cursor.getColumnIndex("age"));
                            Log.e("Test","name = "+name+",id="+id+",age="+age);
                            list.add(id + "--" + id+"--"+age);
                        }
                        cursor.close();
                    }
                });
    }
```

## 源码分析