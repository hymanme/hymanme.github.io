---
layout: post
title: BitmapFactory.decodeResource 解码资源文件返回 NULL
summary: 我们经常在使用 BitmapFactory 来创建 bitmap 时，会使用以下方法，但是有时候你会发现明明一切都很正常，最后返回的 `bitmap == NUll`...
date: 2018-08-27 20:20:08
categories: Android
tags: [Android, Bitmap, SVG]
featured-img: work
---

## 遇到 decodeResource bitmap 返回 NULL
我们经常在使用 BitmapFactory 来创建 bitmap 时，会使用以下方法，但是有时候你会发现明明一切都很正常，最后返回的 `bitmap == NUll`。

```
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher, options);
    Log.d(TAG, "options：outWidth:"+options.outWidth);
    Log.d(TAG, "options：outHeight:"+options.outHeight);
    Log.d(TAG, "options：inBitmap:"+options.inBitmap);
```
## 分析问题
1. 资源文件不存在，即 ic_launcher 不存在
2. 文件存在，但是存在多个适配资源文件，如`mipmap-anydpi-v26`文件夹

在最新的 Android Studio 上默认会给你生成一个 mipmap-anydpi-v26 文件夹，会使用 `xml` 生成一份 ic_launcher 图片资源，此时使用`BitmapFactory.decodeResource()`方法默认加载的是该文件夹下的`xml` 图片，由于版本兼容问题，是加，最终导致返回的 bitmap 为 NULL。

## 解决
1. 删除 `mipmap-anydpi-v26` 文件夹

    如果用不上 `xml` 图片资源可以删除，只是用 png 图片资源
2. 使用 xml 图片加载
    
    如果待使用的图片资源就是 `xml` 问价，如 `vector` 图片，可以使用以下方法加载:
    
    ```
        public static Bitmap getBitmap(Context context, int vectorDrawableId) {
        final VectorDrawableCompat drawable = VectorDrawableCompat.create(context.getResources(), vectorDrawableId, null);
        if (drawable == null) {
            return null;
        }
        Bitmap bitmap = Bitmap.createBitmap(drawable.getIntrinsicWidth(),
                drawable.getIntrinsicHeight(), Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(bitmap);
        drawable.setBounds(0, 0, canvas.getWidth(), canvas.getHeight());
        drawable.draw(canvas);
        return bitmap;
    }
    ```
## 总结
出现这种问题是因为兼容性问题，其实在 drawable 文件夹下的资源文件也会出现类似问题，主要问题点是`BitmapFactory.decodeResource()`加载 `vector`资源文件会失败，需要针对性做兼容处理。



