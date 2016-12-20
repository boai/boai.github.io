---
title: iOS开发第三方键盘处理的那些事儿
---

最近项目中遇到了键盘处理通知被调用多次的情况，废了好半天时间才找到解决办法，今天就给小伙伴儿们唠唠第三方键盘处理的那些坑！详情请看：[『BAKeyboardDemo』](https://github.com/boai/BAKeyboardDemo) ！


## 1、聊天评论框的封装
先聊聊我项目中遇到的奇葩情况吧，一个直播界面，上面播放器，下面是分段控制器5个button，5个界面，其中三个界面最下面都是评论框，所以就封装了一个评论框公用。
但是本来用的[『IQKeyboardManager』](https://github.com/hackiftekhar/IQKeyboardManager)，开源键盘处理框架，可是在同一个界面有多个评论框就出现问题了。

## 2、先看看[『IQKeyboardManager』](https://github.com/hackiftekhar/IQKeyboardManager)的使用吧：
```
#import "AppDelegate.h"

AppDelegate 中添加这段代码，就可以全局不用管键盘的弹起收回了！

#pragma mark -  键盘处理
- (void)completionHandleIQKeyboard
{
    IQKeyboardManager *manager = [IQKeyboardManager sharedManager];
    manager.enable = YES;
    manager.shouldResignOnTouchOutside = YES;
    manager.shouldToolbarUsesTextFieldTintColor = YES;
    manager.enableAutoToolbar = YES;
}
```

## 3、具体解决办法
但是我项目中得复杂情况就不行了，键盘弹起异常，收回也异常，尤其是用了聊天室这种view，处理的更麻烦，不要急，看看博爱这些年踩过的坑吧：
先看看键盘处理事件吧：

- 1、首先注册通知：
```
- (void)registNotification
{
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardWasShown:) name:UIKeyboardWillShowNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardWasHidden:) name:UIKeyboardWillHideNotification object:nil];
}

```

- 2、再把后路想好：移除通知
```
- (void)removeNotification
{
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```

- 3、通知事件处理：
```
/*! 键盘显示要做什么 */
- (void)keyboardWasShown:(NSNotification *)notification
{
    NSDictionary *info                                  = [notification userInfo];

    double duration                                     = [info[UIKeyboardAnimationDurationUserInfoKey] doubleValue];

    CGFloat curkeyBoardHeight                           = [[info objectForKey:@"UIKeyboardBoundsUserInfoKey"] CGRectValue].size.height;
    CGRect begin                                        = [[info objectForKey:@"UIKeyboardFrameBeginUserInfoKey"] CGRectValue];
    CGRect end                                          = [[info objectForKey:@"UIKeyboardFrameEndUserInfoKey"] CGRectValue];

    CGFloat keyBoardHeight;

    /*! 第三方键盘回调三次问题，监听仅执行最后一次 */
    if(begin.size.height > 0 && (begin.origin.y - end.origin.y > 0))
    {
        keyBoardHeight                                  = curkeyBoardHeight;
        [UIView animateWithDuration:duration animations:^{

            CGRect viewFrame                            = [self getCurrentViewController].view.frame;
            viewFrame.origin.y -= keyBoardHeight;
            [self getCurrentViewController].view.frame = viewFrame;
        }];
    }
}

- (void)keyboardWasHidden:(NSNotification *)notification
{
    NSDictionary *info = [notification userInfo];
    double duration = [info[UIKeyboardAnimationDurationUserInfoKey] doubleValue];

    [UIView animateWithDuration:duration animations:^{

        CGRect viewFrame = [self getCurrentViewController].view.frame;
        viewFrame.origin.y = 0;
        [self getCurrentViewController].view.frame = viewFrame;
    }];
}

/*!
*  获取当前View的VC
*
*  @return 获取当前View的VC
*/
- (UIViewController *)getCurrentViewController
{
    for (UIView *view = self; view; view = view.superview)
    {
    UIResponder *nextResponder = [view nextResponder];
    if ([nextResponder isKindOfClass:[UIViewController class]])
    {
    return (UIViewController *)nextResponder;
    }
    }
    return nil;
}
```

具体情况是这样的，在测试过程中，其他界面的评论框都没问题，就直播这个VC有问题，就一步步往下找，后来发现：iOS的第三方键盘会在【- (void)keyboardWasShown:(NSNotification *)notification】这个方法中来回调用多次，不止三次好像，然后就想到一个办法，
```
/*! 第三方键盘回调三次问题，监听仅执行最后一次 */
if(begin.size.height > 0 && (begin.origin.y - end.origin.y > 0))
{
    keyBoardHeight                                  = curkeyBoardHeight;
    [UIView animateWithDuration:duration animations:^{

        CGRect viewFrame                            = [self getCurrentViewController].view.frame;
        viewFrame.origin.y -= keyBoardHeight;
        [self getCurrentViewController].view.frame = viewFrame;
    }];
}
```

在这里处理这个键盘弹起事件中第一次获取键盘的高度，然后就直接把上面的view给弹上去，这样就避免了第三方键盘会来回调用多次方法，造成键盘弹起异常的问题就迎刃而解了！


详情请看：[『BAKeyboardDemo』](https://github.com/boai/BAKeyboardDemo) 
More info: [『BABaseProject』](https://github.com/boai/BABaseProject)
