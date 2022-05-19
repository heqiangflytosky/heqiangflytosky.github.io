---
title: Vue -- 网络请求
categories: JavaScript
comments: true
tags: [Vue]
description: 介绍一下Vue网络请求的方法
date: 2019-1-16 10:00:00
---

## ajax


## vue-resource

https://www.runoob.com/vue2/vuejs-ajax.html

### 安装与引用

1.
在package.json中添加：

```
  "dependencies": {
    ......
    "vue-resource": "^1.5.1"
  },
```
然后运行 `npm install`。
2.
或者直接运行 `npm install --save vue-resource`。
3.
或者直接运行 `npm i vue-resource --save`。

引用：

```
import VueResource from 'vue-resource'

Vue.use(VueResource)
```

### 使用

可以通过 `Vue.http` 或者 `this.$http` 来使用。

```
      this.$http
        .get("https://gank.io/api/v2/categories/GanHuo")
        .then(
          function (res) {
            console.log("获取分类");
            console.log(res.data.data);
            this.data = this.data.concat(res.data.data);
            this.loaded = true;
          },
          function (res) {}
        )
        .catch(function (res) {});
```

### 拦截器

可以把下面代码放到 main.js 里面为网络请求添加拦截器：

```
Vue.http.interceptors.push((request, next) => {
  console.log('interceptors 1')
  next((reponse) => {
    console.log('interceptors 2')
    return reponse
  })
})
```

## axios

https://www.runoob.com/vue2/vuejs-ajax-axios.html

### 安装

```
npm install --save axios
```

### 使用

```
import axios from 'axios';

      axios
        .get("https://gank.io/api/v2/categories/GanHuo")
        .then(
          function (res) {
            console.log("获取分类");
            console.log(res.data.data);
          },
          function (res) {
            console.log("error");
          }
        )
        .catch(function (res) {
          console.log("catch error");
        });
```

### 配置

```
      var http = axios.create({
        baseURL: "https://gank.io",
        timeout: 15000,
        transformRequest:[function (data) {
          //console.log('transformRequest = '+JSON.stringify(data))
          return data;
        }],
        transformResponse: [function (data) {
          //console.log('transformResponse = '+JSON.stringify(data))
          return data;
        }]
      });

      http
        .get("/api/v2/categories/GanHuo")
        .then(
          function (res) {
            console.log("获取分类");
            console.log(res);
          },
          function (res) {
            console.log("error");
          }
        )
        .catch(function (res) {
          console.log("catch error");
        });
```

### 拦截器

```
      // 添加请求拦截器
      axios.interceptors.request.use(
        function (config) {
          // 在发送请求之前做些什么
          console.log("interceptor 1");
          return config;
        },
        function (error) {
          // 对请求错误做些什么
          return Promise.reject(error);
        }
      );

      // 添加响应拦截器
      axios.interceptors.response.use(
        function (response) {
          // 对响应数据做点什么
          console.log("interceptor 2");
          return response;
        },
        function (error) {
          // 对响应错误做点什么
          return Promise.reject(error);
        }
      );
```