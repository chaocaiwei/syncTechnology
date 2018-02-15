# iOS性能优化的一般策略

## 1 CPU的高效使用
### 1.1 时间复杂度
衡量程序性能的指标，首先想到的是程序执行的时长，也就是时间复杂度。我们在对程序算法进行优化时，是当其冲是要降低其时间复杂度。

我们需要通过时间复杂度，或则程序耗时，来验证我们算法的运行效率。我们也可以通过Instruments查看到程序运行中cpu损耗，找到性能瓶颈的大致范围。再通过在关键代码段增加程序耗时的日志信息，来找到性能瓶颈所在。

#### 1.1.1 如何计算程序耗时

获取时间的方式主要有`NSDate ` 、 `clock_t `、 `CFAbsoluteTime ` `CACurrentMediaTime `、 `mach_absolute_time `。他们之前区别可以参考[这里](http://blog.csdn.net/skymingst/article/details/41892445)。为了更加精确更加原子性的测试代码段的时间复杂度，我们采用`CACurrentMediaTime `，他的单位是秒，9位小数，可以精确到纳秒

```
CFTimeInterval start = CACurrentMediaTime();
//some code need caculate
CFTimeInterval end = CACurrentMediaTime();
NSLog(@"cost time: %f s", end - start);
```
如果单纯运用于测试代码中，我们可以采用更具专业性的`dispatch_benchmark `。值得注意的是它是私有API不允许使用与工程代码中。具体可以参考[这里](http://nshipster.cn/benchmarking/)。它能计算出一个代码段运行指定次数后，所消耗的平均时间，精确到纳秒

```
extern uint64_t dispatch_benchmark(size_t count, void (^block)(void));

uint64_t t = dispatch_benchmark(iterations, ^{
    @autoreleasepool {
        NSMutableArray *mutableArray = [NSMutableArray array];
        for (size_t i = 0; i < count; i++) {
            [mutableArray addObject:object];
        }
    }
});
NSLog(@"[[NSMutableArray array] addObject:] Avg. Runtime: %llu ns", t);
```

#### 1.1.2 使用时间复杂度更低的方法
除了努力降低自己实现的算法的时间复杂度外，我们在调用系统API或者第三方SDK时，需要了解不同方法的时间复杂度，尽量避免使用某些时间复杂度高的方法。

拿数组去重为例，最直接的方式如下：

```
- (void)removeRepeatContainFunc:(NSArray *)array
{
    NSMutableArray *marr = [NSMutableArray new];
    for (ClassA *a in array) {
        if (![marr containsObject:a]) {
            [marr addObject:a];
        }
    }
}
```
而上述代码运行效率极低。为了找到更高效的方法，我们来看一下集合类型的一些常用方法的复杂度。

| 方法  | 耗时（ns） | 时间复杂度 |
|:------------- |:---------------:| :-------------:|
|[NSMutableArray removeObject:]|839899|O(n)|
|[NSMutableArray indexOfObject:]|733316|O(n)|
|[NSMutableArray containsObject:]|727187|O(n)|
|[NSMutableArray indexOfObject:]|704578|O(n)|
|[NSMutableDictionary allKeysForObject:]|1296325|O(n)|
|+[NSMutableSet setWithArray:]|2610939|O(n)|
|[NSMutableArray addObject:]|564|O(1)|
|[NSMutableArray setObject:atIndexedSubscript:]|168|O(1)|
|[NSMutableDictionary setObject:forKey:]|565|O(1)|
|[NSMutableSet addObject:]|582|O(1)|
|[NSMutableDictionary objectForKey:]|348|O(1)|

上表中时间复杂度为O(n)的方法，都会遍历所有元素，俩俩进行对象等同性检测。进行对象等同性检测时，往往会调用下述两个方法

```
- (BOOL)isEqual:(id)object;
@property (readonly) NSUInteger hash;
```

由此我们得到给为高效的方法：

```
- (void)removeRepeatSetFunc:(NSArray *)array
{
    NSMutableSet *mSet        = [NSMutableSet new];
    NSMutableArray *marr      = [NSMutableArray new];
    for (ClassA *a in array) {
        if (![mSet containsObject:a]) {
            [marr addObject:a];
            [mSet addObject:a];
        }
    }
}
- (void)removeRepeatDicFunc:(NSArray *)array
{
    NSMutableDictionary *mdic = [NSMutableDictionary new];
    NSMutableArray *marr      = [NSMutableArray new];
    for (ClassA *a in array) {
        if (!mdic[@(a.hash)]) {
            [marr addObject:a];
            mdic[@(a.hash)] = a;
        }
    }
}
```


| 方法 | 耗时  | 时间复杂度 |
|:------------- |:---------------:| :-------------:|
|removeRepeatContainFunc|5.987086|O(n^2)|
|removeRepeatSetFunc|3.585868|O(n)|
|removeRepeatDicFunc|3.994205|O(n)|

### 1.2 运用多线程提高运行效率
### 1.3 性能监测方式
#### 1.3.1 通过监听主线程方式来监察

首先用 CFRunLoopObserverCreate 创建一个观察者里面接受 CFRunLoopActivity 的回调，然后用 CFRunLoopAddObserver 将观察者添加到 CFRunLoopGetMain() 主线程 Runloop 的 kCFRunLoopCommonModes 模式下进行观察。

接下来创建一个子线程来进行监控，使用 dispatch_semaphore_wait 定义区间时间，标准是 16 或 20 微秒一次监控的话基本可以把影响响应的都找出来。监控结果的标准是根据两个 Runloop 的状态 BeforeSources 和 AfterWaiting 在区间时间是否能检测到来判断是否卡顿。
#### 1.3.2 界面的刷新频率
屏幕刷新频率是否能保持在60fps附近，是我们判定界面是否卡顿的指标。我们可以使用CADisplayLink来监测屏幕的刷新频率。

```
- (void)startlinkWithBlock:(void(^)(NSTimeInterval  fps))block
{
    self.fpsBlock = block;
    _link = [CADisplayLink displayLinkWithTarget:self selector:@selector(tick:)];
    [_link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)tick:(CADisplayLink *)link {
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    _count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
    if (delta < 1) return;
    _lastTime = link.timestamp;
    float fps = _count / delta;
    _count = 0;
    NSLog(@"%f",fps);
    if (self.fpsBlock) {
        self.fpsBlock(fps);
    }
}
```

## 2 高效的利用内存（内存泄漏与循环引用）
空间复杂度(Space Complexity)是对一个算法在运行过程中临时占用存储空间大小的量度。这里主要讲的是内存问题。而内存问题主要包括两个部分，一个是iOS中常见循环引用导致的内存泄露，另外就是大量数据加载及使用导致的内存警告。
### 2.1 循环引用
#### 2.1.1 delegate
delegate属性忘记设置为weak引起。代理与被代理对象之间形成了引用环
#### 2.1.2 block
不是所有block都会循环引用，只在同时满足下列条件时，会发生：

- block捕获了强引用类型变量
- block被对象引用
- 对象与block之间形成了引用环

非逃逸的闭包一般来说是可以放心使用的，譬如Massory中添加约束的代码块

解决block循环引用的方法是经典的strong/weak dance。
#### 2.1.3 NSTimer
调用下列方法时会对target强引用，如果target恰好引用NSTimer则形成引用环。

```
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti 
									target:(id)aTarget 
									selector:(SEL)aSelector 
									userInfo:(nullable id)userInfo 
									repeats:(BOOL)yesOrNo;
```
>target :
The object to which to send the message specified by aSelector when the timer fires. The timer maintains a strong reference to target until it (the timer) is invalidated.

解决方法是在控制器`viewDidDisappear`中将自身及其上timer都`invalidate`。值得注意的是，所有子视图、所有持有的生命与其相同的对象，如果存在timer，都需要在此处`invalidate`

```
- (void)viewDidDisappear:(BOOL)animated
{
    [super viewDidDisappear:animated];
    [self.timer invalidate];
}
```
### 2.2.1 内存警告
一般在占用系统内存超过20%的时候会有内存警告，而超过50%的时候，会发生Crash了。发生内存警告时，会触发下列方法：

```
// AppDelegate
- (void)applicationDidReceiveMemoryWarning:(UIApplication *)application

// UIViewController
- (void)didReceiveMemoryWarning 
```
发生内存警告的主要原因是，在内存中存放了高质量图片等大数据量的资源，而没有及时释放。可以从下述方法着手：

- 尽量减少大数据量资源的使用。
- 调查循环引用，看是否因为内存泄漏而导致的资源无法及时释放
- 建立健全大数据量资源的内存管理机制，把不必要的资源，及时从内存中释放掉。譬如设置SDWebImage的缓存上限。
- 当发生内存警告时，立刻清理缓存，及一切可以清理的不影响当前界面显示及业务逻辑的资源

##  3 I/O性能优化
##  4 电池节能优化
##  5 网络性能优化 






[深入剖析 iOS 性能优化](https://xiaozhuanlan.com/topic/2847503196)

[How To Use Runloop](https://www.jianshu.com/p/6757e964b956)

[谈谈iOS app的线上性能监测](https://www.cnblogs.com/dsxniubility/p/5493117.html)

[iOS App 稳定性指标及监测](http://www.cocoachina.com/ios/20170804/20145.html)
