---
title: iOS之CAAnimation
date: 2018-03-02 18:53:26
tags: iOS动画
category: iOS
---
UIView, CALayer, CAAnimation，CATransication等；

### 呈现树模型树以及同步

* CAAnimation是加载view.layer.presentationlayer上，而不会更新到view.layer.modellayer;
* UIView和UIView的自带layer的属性实际上是对应的是CALayer的modellayer，presentationlayer和modellayer存在同步的关系；
* 任意时刻如果CAAnimation加到view.layer上，presentationlayer优先从CAAnimation取状态要怎么刷新，动画做完将回归到modellayer的状态；
* 可以使用CADisplaylink定时器，中取view.layer.presentationlayer的bounds，positon，transform等信息同步到uiview的layer上（modellayer）；

```
[CATransaction setAnimationDuration:0.25];
[CATransaction setCompletionBlock:^{
	[self stopDisplayLink];
}];
[CATransaction begin];
[self.videoView.layer addAnimation:group forKey:@"showAnimation"];
[CATransaction commit];
[self startDisplayLink];
 
 - (void)startDisplayLink
{
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(handleDisplayLink:)];
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)syncLayer
{
    CALayer *presentationLayer = self.videoView.layer.presentationLayer;
    self.videoView.bounds = presentationLayer.bounds;
    self.videoView.center = presentationLayer.position;
    self.videoView.layer.transform = presentationLayer.transform;
}

- (void)stopDisplayLink
{
    [self syncLayer];
    [self.displayLink invalidate];
    self.displayLink = nil;
}

- (void)handleDisplayLink:(CADisplayLink *)displayLink
{
    [self syncLayer];
}
```
### UIView的block动画
* 对CALayer的animatable属性赋值的时候，会对CALayer的delegate调用[actionForLayer:forKey:]，如果该方法返回nil，则展示默认的动画，如果返回NSNull则不走动画，如果返回一个实现了CAAction协议的对象（例如CABasicAnimtion）则会走这个返回的动画；
* 如果设置Layer的animatable方法调用在uiview animation的block里面，则会返回一个CAAnimation<CAAction>；
UIView的自带layer的delegate是自己，如果不在block里面设置的话，就返回nsnull就不会动了，如果直接加在layer上就返回nil显示隐式动画；

### CAMediaTiming

* CAAnimation除了实现CAAction协议还实现了CAMediaTiming协议，主要提供控制动画过程需要的各种参数和动画模式；

{% asset_image mediatiming.jpeg mediatiming %}