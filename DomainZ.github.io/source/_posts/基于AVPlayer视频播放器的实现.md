---
title: 基于AVPlayer视频播放器的实现
date: 2019-09-02 17:33:22
tags: AVPlayer
categories: 音视频
---

主流APP的播放器都是基于AVPlayer用起来高效方便，本文主要记录一下，播放器开发过程中会遇到的问题。

### Player
Player设计成单例，包含avplayer、playLayer、assetLoader、当前的播放窗口以及播放、暂停、拖动等控制方法。另外定义回调的delegate包含了下载流和item状态相关供上层使用。

0. 传入视频URL、播放窗口targetView（UIView）以及播放位置second，检查缓存是否有文件；
1. 如果不存在，切换URL的scheme（不切换的话不会走assetLoader）。把之前的asset（AVURLAsset）停掉销毁。创建新的asset，并且给asset设置assetLoader（AVAssetResourceLoader），并且设置assetLoader的delegate为自己。如果播放器有截图的需求，要自定义output（AVPlayerItemVideoOutput）替换掉原来的。
2. [targetView.layer addSublayer:self.playerLayer];
3. [self playAtSecond:second];

### AssetLoader

边下边播的需要实现AVAssetResourceLoaderDelegate协议，这里需要实现好缓存策略，如果有下载一半的文件要设置好http header的Range参数防止从头开始下载。为了尽可能的保证列表流畅NSURLConnection的保持在后台queen要注意多线程死锁崩溃。connecetion的重试和报错回调逻辑，比如不能出错就回调，至少要等到缓存播放到不能播再回调。


0. 初始化存放视频请求Header的cache（设置好过期时间防止挤压太多这里用YYMemoryCache），以及NSOperationQueue；
1. 在resourceLoader:shouldWaitForLoadingOfRequestedResource:方法中初始化connection，这里的url需要把scheme替换回去。然后用数组保存传入的request对象，数据加载好之后把这些request finish掉；

...未完待续

