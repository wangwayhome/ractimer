## 从计时器出发，探寻老项目RAC改造的挖坑与填坑之旅

### 背景思考

条件：假设有三种交通工具，分别是汽车、自行车、摩托车

需求：我们需要从A点到B点，

问题：你会选择哪种交通工具？



### 开膛破肚 分析原来的代码

首先这个CMTCountDownButton 里面集合了丰富的操作

1、计时器的功能

2、各种变量 是否running  颜色设置等等

3、一堆的block回调

![image-20181114140253130](https://ws1.sinaimg.cn/large/006tNbRwly1fx7k0grkzoj30oi058765.jpg)

```objective-c

@interface CMTCountDownButton : UIButton

///**
// 剩余几秒文案
// */
//@property (nonatomic, copy) CountDownTitleChange countDownTitleChange;
//
///**
// 计时器停止时显示文案
// */
//@property (nonatomic, copy) CountDownTitleCompletion countDownTitleCompletion;
//
//@property (nonatomic, copy) CountDownEnableBlock countDownEnableBlock;
//
//@property (nonatomic, strong) UIColor *enableColor;
//
//@property (nonatomic, strong) UIColor *disableColor;
//
//@property (nonatomic, getter=isRunning) BOOL running;


@end

```

```objective-c
//
//  CMTCountDownButton.m
//  CMTApp
//
//  Created by qiuyuliang on 2018/9/25.
//  Copyright © 2018年 ucarinc.com. All rights reserved.
//

#import "CMTCountDownButton.h"

@interface CMTCountDownButton() {
//    NSTimer *_timer;
}


//@property(nonatomic, readwrite, assign) NSUInteger totalSecond;//总秒数
//
//@property(nonatomic, readwrite, assign) NSUInteger second;//剩余秒数

@end


@implementation CMTCountDownButton

#pragma mark - Public

//
//#pragma mark - Private
//
//- (void)stopCountDown {
//    if ([_timer isValid]) {
//        [_timer invalidate];
//        _second = _totalSecond;
//    }
//}
//
//- (BOOL)isRunning {
//    return [_timer isValid];
//}
//
//#pragma mark - IBActions
//
//- (void)timerStart:(NSTimer *)timer {
//    
//    if (_second == 0) {
//        self.enabled = YES;
//        if (self.countDownTitleCompletion) {
//            [self setTitle:self.countDownTitleCompletion() forState:UIControlStateNormal];
//        } else {
//            [self setTitle:@"发送验证码" forState:UIControlStateNormal];
//        }
//        
//        [self setTitleColor:self.enableColor forState:UIControlStateNormal];
//        
//        [self stopCountDown];
//        
//        if (self.countDownEnableBlock) {
//            BOOL enable = self.countDownEnableBlock();
//            if (enable) {
//                self.enabled = YES;
//                [self setTitleColor:self.enableColor forState:UIControlStateNormal];
//            } else {
//                self.enabled = NO;
//                [self setTitleColor:self.disableColor forState:UIControlStateNormal];
//            }
//        }
//        
//    } else {
//        if (self.countDownTitleChange) {
//            NSString *title = self.countDownTitleChange(_second);
//            [self setTitle:title forState:UIControlStateNormal];
//        } else {
//            NSString *title = [NSString stringWithFormat:@"还剩%lu秒", (unsigned long)_second];
//            [self setTitle:title forState:UIControlStateNormal];
//        }
//        [self setTitleColor:self.disableColor forState:UIControlStateNormal];
//        _second --;
//    }
//}

@end

```



并且分散在view controller和uiview中的 有涉及到按钮调用的地方，都要设置如下代码。

比如控制是否能继续点击啊，还有颜色的设置啊，网络请求啊，还有各种各种。。。

比如如下：

uiview类

![image-20181114141455312](https://ws4.sinaimg.cn/large/006tNbRwly1fx7kcxwuw4j310e0gqdno.jpg)

![image-20181114140952472](https://ws1.sinaimg.cn/large/006tNbRwly1fx7k7t6g6mj31kw0zw4px.jpg)

view controller 类

![image-20181114141131652](https://ws2.sinaimg.cn/large/006tNbRwly1fx7k9ej0ozj31da0kgwou.jpg)

![image-20181114141550917](https://ws1.sinaimg.cn/large/006tNbRwly1fx7kdw2nfzj31j606odlh.jpg)

##### 玉良书记这个写法，看上去也还是不错的。起码可读性100分，日后维护简单方便，无非要多码一些代码。



这种东西，没有好和不好嘛，只有合适不合适，*新技术嘛，在于来回折腾*。（此处引用喵神的话）

![image-20181114142109206](https://ws1.sinaimg.cn/large/006tNbRwly1fx7kjfhco2j30wy0c877k.jpg)

好了，接下来，开始我的折腾。

![âå¼å§ä½ çè¡¨æ¼ è¡¨æâçå¾çæç´¢ç"æ](https://ws3.sinaimg.cn/large/006tNbRwly1fx7kmemmprj30b407u74o.jpg)









## RAC 实现重构

### 先考虑计时器的功能以及网络请求应该放在哪里合适，思来想去，放在view model好了。

于是乎开始编写最核心的功能，计时器+网络请求

一开始肯定是一脸懵逼的，所以从最简单的计时器开始写起来。

```objective-c
    // RAC循环写法, 用于倒计时，计时器总次数_totalSecond
    RACSignal *timesignal = [[[RACSignal interval:1.0 onScheduler:[RACScheduler mainThreadScheduler]] take:_totalSecond] takeUntil:self.rac_willDeallocSignal];
```

于是乎一个计时器信号创建诞生。  你们要问了，啊，怎么就这么一句话，计时器干了啥？你怎么没有map 来写？啊 日，这样看来好像很奇怪，很多陌生的东西是什么呢？

这里创建了一个1s运行一次的计时器，运行了totalSecond次，mainThreadScheduler 表示运行在主线程。takeUntil:self.rac_willDeallocSignal 这句可以保证在页面销毁的时候移除通知。

### 创建按钮绑定事件

```objective-c
// sendcode command 的写法, command里面返回的是一个计时器的signal
self.sendCodeCommand = [[RACCommand alloc] initWithEnabled:limitSignal signalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
        @strongify(self)
        self.second = self.totalSecond;
        return timesignal;
    }];
```

### 写到这里，发现按钮起不起作用，我们要有个信号来控制，so，我们来搞一个limitSignal

来告诉 initWithEnabled 的 参数是否启用按钮enable

```objective-c
 //订阅限制信号，监听second的值，如果为 0 就是返回yes 让button 的RACCommand 的enable 设置为 yes。
    RACSignal *limitSignal = [RACObserve(self, second) map:^id _Nullable(id  _Nullable value) {
        return @(self.second == 0);
    }];
```

然后回过头看上面的代码。

### 我们在最后的block中返回了return timesignal; 计时器的信号，那么让他走起来的代码，我设置了另一个地方switchToLatest subscribeNext 来写计时器的内容，

switchToLatest subscribeNext：的意思就是去监听sendCodeCommand 最后一个返回的信号，然后去执行该信号的subscribeNext 块。也就是timesignal的信号的subscribeNext 事件。

如下：

```objective-c
// 监听command 返回的计时器信号，用于控制btn 文字的显示    
   [self.sendCodeCommand.executionSignals.switchToLatest subscribeNext:^(id  _Nullable x) {
        @strongify(self)
        if (self.second == 0) {
            [self.changeNameSubject sendNext:@"重新发送"];
            NSLog(@"重新发送");
        } else {
            NSLog(@"%@", [NSString stringWithFormat:@"%ld", (long)self.second]);
            [self.changeNameSubject sendNext:[NSString stringWithFormat:@"%ld", (long)--self.second]];
        }
    }];
```

好了 ，这样到此为止一个计时器完成了。说简单，也不简单，说复杂也不复杂。还有更简单的写法，我们来变一变吧？同志们做好折腾的准备。

开始吧，

### 我们还可以把计时器干活的代码迁移到其他地方去，比如说加到计时器信号开始创建的地方。

```objective-c
    // RAC循环写法, 用于倒计时，计时器总次数_totalSecond
    RACSignal *timesignal = [[[[RACSignal interval:1.0 onScheduler:[RACScheduler mainThreadScheduler]] take:_totalSecond] map:^id _Nullable(NSDate * _Nullable value) {
        @strongify(self)
        if (self.second == 0) {
            [self.changeNameSubject sendNext:@"重新发送"];
            return @(YES);
        } else {
            [self.changeNameSubject sendNext:[NSString stringWithFormat:@"还剩%lu秒", (long)--self.second]];
            return @(NO);
        }
    } ]takeUntil:self.rac_willDeallocSignal];
```

然后endCodeCommand.executionSignals.switchToLatest 这货就免了。

写到这里会发现一个尴尬的问题，特么的我网络请求还没加呢，啪啪啪 打脸，我还是在想想其他写发，疯狂return到之前到写法ing

### endCodeCommand.executionSignals.switchToLatest  这货还是捡回来吧。

然后呢，我需要在RACCommand 里面去添加网络请求事件。

```objective-c
 self.sendCodeCommand = [[RACCommand alloc] initWithEnabled:limitSignal signalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
        
     
        
    }];
```

怎么搞呢？怎么搞呢？

于是。。。

```objective-c
    self.sendCodeCommand = [[RACCommand alloc] initWithEnabled:limitSignal signalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
        
        RACSignal *netWorkSignal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
            [self getVerifyCodeRequest:self -> _phoneNumber type:CMTVerifyForgetPhone completionBlock:^(NSError *error, CMTVerifyCodeModel *model) {
                if (error) {
                    NSDictionary *userinfo = error.userInfo;
                    
                    if (userinfo) {
                        NSString *errorStr = [userinfo objectForKey:@"NSLocalizedDescription"];
                        
                        if (![NSString isBlankString:errorStr]) {
                            [self.loadingSubject sendNext:errorStr];
                        }
                    }
                } else {
                }
                
                //发送信号：网络请求结束并执行timer信号
                [subscriber sendNext:nil];
                [subscriber sendCompleted];
                
            }];
            
            return [RACDisposable disposableWithBlock:^{
                NSLog(@"信号消除了");
            }];
            
        }];
            return netWorkSignal;
        
    }];

```

block 中再嵌套一个block 来创建网络请求信号。我把return timesignal 也给干掉了。

然后在sendCodeCommand.executionSignals.switchToLatest subscribeNext 中去执行计时器的操作。后续再讲怎么改造。

这样一来，看上去还是不够完美，看上去很臃肿，怎么办。那还是

### 把网络请求的sign 拎出来。

变成如下：

```objective-c
    //订阅按钮事件的信号
    self.sendCodeCommand = [[RACCommand alloc] initWithEnabled:limitSignal signalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
        
        return netWorkSignal;//执行网络请求
        
    }];
```

简洁了很多。

### 网络请求创建信号的写法

```objective-c
@weakify(self);
    RACSignal *netWorkSignal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        @strongify(self)
        [self.loadingSubject sendNext:nil];
        [self getVerifyCodeRequest:self -> _phoneNumber type:CMTVerifyForgetPhone completionBlock:^(NSError *error, CMTVerifyCodeModel *model) {
            if (error) {
                NSDictionary *userinfo = error.userInfo;
                
                if (userinfo) {
                    NSString *errorStr = [userinfo objectForKey:@"NSLocalizedDescription"];
                    
                    if (![NSString isBlankString:errorStr]) {
                        [self.loadingSubject sendNext:errorStr];
                    }
                    
                }
                
            } else {
                [self.loadingSubject sendNext:@"短信验证码已发送"];
            }
            
            //发送信号：网络请求结束并执行timer信号
            [subscriber sendNext:nil];
            [subscriber sendCompleted];
            
        }];
        
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"信号消除了");
        }];
        
    }];
    
```



重点来了，我们关注下 这两句话       

//发送信号：网络请求结束并执行timer信号

```objective-c
[subscriber sendNext:nil];
[subscriber sendCompleted];
```



这两句话取决了 要执行sendCodeCommand.executionSignals.switchToLatest subscribeNext的关键代码

### 于是我们把sendCodeCommand.executionSignals.switchToLatest subscribeNext改造如下。

```objective-c
    //switchToLatest:用于signal of signals，获取signal of signals发出的最新信号,也就是可以直接拿到RACCommand中的信号
    [self.sendCodeCommand.executionSignals.switchToLatest subscribeNext:^(id  _Nullable x) {
        @strongify(self)
        
        self.second = self.totalSecond;
        
        [timesignal subscribeNext:^(id  _Nullable x) {//开始执行计时器
            @strongify(self)
            
            if (self.second == 0) {
                [self.changeNameSubject sendNext:@"重新发送"];
            } else {
                    [self.changeNameSubject sendNext:[NSString stringWithFormat:@"还剩%lu秒", (long)--self.second]];
            }
            
        }];
        
    }];
```



timesignal subscribeNext 这里去启动计时器。



目前为止 应该把该写的写好了，我勒个擦，写RAC是很快很爽。那么，改工程就不简单了。又遇到问题了。这些

### RAC绑定我是写在view model 中的，如何与 view controller 进行关联？



呵呵，完蛋了。还有一个需求是绑定要在相关的UI控件懒加载完之后，加在viewDidLoad、viewWillAppear等等也都不合适。

怎么办，心里凉凉，有没有更好的方式解决呢

凉哥想到了一招

非常贱的操作，也就是在view controller 的viewDidLoad之后 去调用view model绑定这些事件。

怎么绑定监听系统的viewDidLoad 呢

继续用到RAC的操作

### rac_signalForSelector设置监听方法

```objective-c
    [[vc rac_signalForSelector:@selector(viewDidLoad)] subscribeNext:^(RACTuple * _Nullable x) {
                 @strongify(vc)
                 [vc bindViewModel];
             }];
```

这样我们就实现了， 监听viewDidLoad 事件 执行之后，执行我们的相关绑定操作。

那么创建view controller 每次都要写这几句话，烦不烦。

于是加到父类的allocWithZone方法，去重写它 ，代码如下

### allocWithZone重写

```objective-c

+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    
    CMTBaseViewController *vc = [super allocWithZone:zone];
    @weakify(vc)
    [[vc rac_signalForSelector:@selector(viewDidLoad)] subscribeNext:^(RACTuple * _Nullable x) {
                 @strongify(vc)
                 [vc bindViewModel];
             }];
    return vc;
}

- (void)bindViewModel {
    
}
```

emmmm....这样我们在子类去实现一遍bindViewModel 就可以愉快的绑定RAC的相关信号了。



