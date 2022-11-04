---
title: "Android-Glide"
date: "2022-11-04"
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

## 前言

Glide是一个快速高效的Android图片加载库，注重于平滑的滚动。统一了显示本地图片和网络图片的接口。

## 使用步骤

### 1.添加Glide库的依赖：
```gradle
implementation 'com.github.bumptech.glide:glide:4.9.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.9.0'
annotationProcessor 'androidx.annotation:annotation:1.1.0'
```

### 2.添加权限
```kotlin
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

### 3.初步使用

```kotlin
val url = "http://host/image.png"
Glide.with(context)
	 .load(url)
	 .into(imageView)
```

### 4.转换

```kotlin
val options = RequestOptions()
			// 变换显示样式
			.centerCrop()
			// 请求时占位
			.placeholder(R.mipmap.loading)
			// 请求失败
			.error(R.mipmap.error)
			// 请求为空的时候
			.fallback(R.mipmap.empty)

val url = "http://host/image.png"
Glide.with(context)
	 .load(url)
	 .apply(options)
	 .into(imageView)
```

### 5.缓存

默认情况下，Glide会再开始一个新的图片请求之前检查以下多级的缓存:

+ 活动资源(Active Resources) : 现在是否有另一个View正在展示这张图片
+ 内存缓存(Memory Cache) : 该图片是否最近被加载过并仍存在于内存
+ 资源类型(Resources) : 该图片是否之前曾被解码、转换并写入过磁盘缓存
+ 数据来源(Data) : 构建这个图片的资源是否之前有被写入过文件缓存

#### 内存缓存

通过skipMemoryCache(true)可以设置不进行内存缓存

#### 磁盘缓存

通过diskCacheStrategy(DiskCacheStrategy)设置缓存策略

#### 磁盘缓存策略

+ DiskCacheStrategy.ALL
	+ 既缓存原图又缓存处理图
+ DiskCacheStrategy.NONE
	+ 什么都不缓存
+ DiskCacheStrategy.SOURCE
	+ 只缓存原图
+ DiskCacheStrategy.RESULT
	+ 只缓存处理图

#### 清除内存缓存

```kotlin
Glide.with(context).clearMemory()
```

清除磁盘缓存

```kotlin
GlobalScope.launch { 
    Glide.with(context).clearDiskCache()         
}
```

> 清除磁盘缓存需要子线程运行

#### 使用缩略图

```kotlin
val url = "http://host/image.png"
Glide.with(context)
	 .load(url)
	 // 缩略图大小未原来的1/10
	 .thumbnail(0.1f) 
	 .into(imageView)
```

#### 使用Target

```kotlin
val url = "http://host/image.png"
val target = object : SimpleTarget<Bitmap>() {
        override fun onResourceReady(resource: Bitmap, transition :Transition<in Bitmap>?) {
            imageview.setImageBitmap(resource)
        }
}
Glide.with(context)
	 .load(url)
	 .into(target)

```

