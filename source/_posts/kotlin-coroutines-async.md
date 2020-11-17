---
title: Kotlin -- 异步操作
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 协程的异步操作
date: 2019-12-28 10:00:00
---

## 概述

在使用 Java 处理异步操作，比如网络请求后更新UI展示，我们首选的是 RxJava，那么转到 Kotlin 上面，我们可以使用协程来替代 RxJava 进行异步操作。

## 使用 Coroutines 代替 RxJava

```
interface GankService {
    @GET("data/category/GanHuo/type/{category}/page/{page}/count/{pageCount}")
    fun getGankData(
        @Path("category") category: String, @Path("pageCount") pageCount: Int, @Path("page") page: Int
    ): Call<GankDataResponse>
}
```

```
    fun getGankData(category:String, pageCount:Int, page:Int, onSuccess:(reponse:GankDataResponse) -> Unit = {}, onFail: () -> Unit ={}){
        GlobalScope.launch(Dispatchers.IO) {
            val reponse = service.getGankData(category, pageCount, page).execute()
            withContext(Dispatchers.Main) {
                if (reponse.isSuccessful) {
                    onSuccess(reponse.body() as GankDataResponse)
                }else{
                    onFail()
                }
            }
        }
    }
```

还可以这样写：

```
    fun getGankData(category:String, pageCount:Int, page:Int, onSuccess:(reponse:GankDataResponse) -> Unit = {}, onFail: () -> Unit ={}){
        GlobalScope.launch(Dispatchers.Main) {
            val reponse = withContext(Dispatchers.IO) {
                service.getGankData(category, pageCount, page).execute()
            }

            if (reponse.isSuccessful) {
                onSuccess(reponse.body() as GankDataResponse)
            }else{
                onFail()
            }
        }
    }
```

```
    override fun loadData(type: String) {
        RequestManager.instance.getGankData(type, 20, 1,
            onSuccess = {gankDataResponse: GankDataResponse ->
                mView.setRefreshing(false)
                val list = ArrayList<GankItem>()
                if (gankDataResponse != null && gankDataResponse.data != null) {
                    for (bean in gankDataResponse.data) {
                        list.add(GankContentItem(bean))
                    }
                }
                mView.updateData(list)
                mView.updateSuccess(list.isEmpty())
            },


            onFail =  {
                mView.setRefreshing(false)
                mView.updateError()
            }

        )
    }
```

## 线程切换

比如，有这样的使用场景，我们查询数据库，然后更新 UI，然后通过网络请求数据，最后再在 UI 上面进行展示，因为数据库和网络请求都是耗时操作，我们要在后台线程执行这样的操作，两次的 UI 展示要在主线程进行操作，这样就涉及到多次线程之间的切换，我们可以通过下面的代码示例解决：

```
    fun httpGet() = GlobalScope.async(Dispatchers.IO){
        delay(100)
        Log.e("Test","httpGet")
        "httpGet return"
    }
    fun queryDB()  = GlobalScope.async(Dispatchers.IO){
        delay(100)
        Log.e("Test","queryDB")
        "queryDB return"
    }

    fun updateUI1(p:String)  = GlobalScope.async(Dispatchers.Main){
        Log.e("Test","updateUI1 $p")
    }

    fun updateUI2(p:String)  = GlobalScope.async(Dispatchers.Main){
        Log.e("Test","updateUI2 $p")
    }
```

```
        GlobalScope.launch(Dispatchers.IO) {
            val ret1 = queryDB().await()
            Log.e("Test",ret1)
            updateUI1(ret1)
            val ret2 = httpGet().await()
            Log.e("Test",ret2)
            updateUI2(ret2)
            Log.e("Test","GlobalScope.launch")
        }
        Log.e("Test","end")
```
