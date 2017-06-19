---
layout: post
title: "UIApplication​Delegate的launch​Options"
date: 2014-01-08 10:14
published: true
comments: true
categories: [iOS, 翻译]
---

AppDelegate就是iOS的垃圾场。

App的生命周期管理、URL处理、通知、CoreData、第三方SDK的初始化，还有那些看起来放到哪里都不合适的函数，统统都被塞到AppDelegate.m里面！

其中，`application:didFinishLaunchingWithOptions:`是最拥挤的一个。

对于许多开发者来说，launchOptions参数如同Java main函数的String[]参数一样，被忽视了。然而，摆在眼前的事实是，launchOptions包含了许多关键性知识，涉及了app在iOS上的众多启动方式。

这个周，我们就谈谈这个UIKit里面最重要的方法，揭秘一下这个知之甚少的launchOptions参数。

<!--more-->

## 

每个app都从UIApplicationDelegate的`application:didFinishLaunchingWithOptions:`方法开始启动(更精确点说，是`application:willFinishLaunchingWithOptions:`方法，如果你把它实现了的话)。应用程序通过调用这个方法来通知它的delegate：启动程序已经完成，差不多已经准备好运行了。

除了点击桌面上的应用图标可以启动程序外，还有其他几种场景可以使程序启动。例如，app如果注册了自定义的URL schemme，比如twitter://，就可以被open URL的方式调用开启程序。也可以通过推送通知或定位服务的方式调用开启程序。

launchOptions参数的所用就是用来判断一个app是如何被启动的。类似于userInfo字典，在`application:didFinishLaunchingWithOptions:`方法中，可以通过launchOptions参数来获取指定key的相关信息。

> 其中的许多key同样使用于UIApplicationDidFinishLaunchingNotification通知。详细信息可以查阅相关文档。

launchOptions包含了大量的key。根据app的启动方式，这些key大体可以被分为以下这几类：

## **Open URL**

通过URL，程序可以调用启动其他程序：

```objc
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"app://..."]];
```

例如，“http://” 的URL可以打开Safari，“mailto://” 的URL可以打开邮件程序，“tel://” 的URL可以拨打电话。

这些情形下，launchOptions里面会包含一个叫`UIApplicationLaunchOptionsURLKey`的Key。

> UIApplicationLaunchOptionsURLKey: 标示了该应用程序是为了打开一个URL启动。这个Key对应的值是一个NSURL对象，表示要打开的URL。

app被URL启动的时候，还可以附带一些系统信息。当app是被UIDocumentInteractionController或AirDrop启动的时候，launchOptions里面还会附带下面这些key：

> UIApplicationLaunchOptionsSourceApplicationKey: 标示了要求启动你的程序的那个app。对应的值是一个NSString，表示那个app的bundle ID。
>
> UIApplicationLaunchOptionsAnnotationKey: 标示了要求打开URL的那个app提供的自定义数据。对应的值是一个property-list类型的对象， 包含自定义的数据。 

```objc
NSURL *fileURL = [[NSBundle mainBundle] URLForResource:@"Document" withExtension:@"pdf"];
if (fileURL) {
    UIDocumentInteractionController *documentInteractionController = [UIDocumentInteractionController interactionControllerWithURL:fileURL];
    documentInteractionController.annotation = @{@"foo": @"bar"};
    [documentInteractionController setDelegate:self];
    [documentInteractionController presentPreviewAnimated:YES];
}
```

## **通知**

这里的通知不是指[NSNotification](http://nshipster.com/nsnotification-and-nsnotificationcenter/)，而是指推送通知和本地通知。

#### **推送通知**

自从iOS3引入推送通知以后，推送通知就成了移动平台最典型的功能之一。

要注册推送通知，在`application:didFinishLaunchingWithOptions:`里面调用`registerForRemoteNotificationTypes:`方法即可：

```objc
[application registerForRemoteNotificationTypes:
    UIRemoteNotificationTypeBadge |
    UIRemoteNotificationTypeSound |
    UIRemoteNotificationTypeAlert];
```

注册成功后，会收到`application:didRegisterForRemoteNotificationsWithDeviceToken:`这个回调。然后，就可以接收推送通知了。

接收到推送通知后，如果app当前处于前台运行状态，appDelgate的`application:didReceiveRemoteNotification:`方法会被调用。然而，当app是因为用户滑动通知中心的推送消息而启动时，`application:didFinishLaunchingWithOptions:`方法会被调用。这个时候，launchOption里面会包含名为`UIApplicationLaunchOptionsRemoteNotificationKey`的key：

> UIApplicationLaunchOptionsRemoteNotificationKey: 表明app有一个推送通知等待处理。这个key对应的值是一个包含了推送通知负载信息的NSDictionary，包括以下这些信息：
> 
> **alert**：alert可以是一个字符串，表示提示信息；也可以是一个包含两个key(**body**和**show-view**)的字典。
> 
>  **badge**：badge是一个数字，用于标示可从提供者下载的数据数量。这数字会被显示在app的图标上。如果推送通知中不包含badge这个字段，则app图标上标记的数字会被移除掉。
>  
>  **sound**：指定app bundle里面用作提示音的声音文件的名字。如果为"default"，则会播放系统默认的提示音。

这样看来，就有两个地方要写处理推送通知的代码。因此，一个通常的做法是在`application:didFinishLaunchingWithOptions:`里面手动调用`application:didReceiveRemoteNotification:`：

```objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ...
    if (launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey]) {
        [self application:application didReceiveRemoteNotification:launchOptions[UIApplicationLaunchOptionsRemoteNotificationKey]];
    }
}
```

#### **本地通知**

[本地通知](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html#//apple_ref/doc/uid/TP40008194-CH103-SW1)从iOS4引入。令人惊奇的是，时至今日，仍然有很多人没有把它搞明白。

app可以定时触发一些`UILocalNotification`通知。通知触发时候，如果app正处于前台运行状态，appDelegate的`application:didReceiveLocalNotification:`方法会被调用。如果app处于非活动状态，通知就会被发布到通知中心。

与推送通知不同，UIApplication的delegate方法提供了一个统一的处理本地通知的地方。如果一个app是被本地通知启动的，会先调用`application:didReceiveLocalNotification:`方法，然后才会调用`application:didFinishLaunchingWithOptions:`(因此，就不需要在`application:didFinishLaunchingWithOptions:`里面手动调用`application:didReceiveLocalNotification`了)。

本地通知在launchOptions里面的key为`UIApplicationLaunchOptionsLocalNotificationKey`，对应的值为`UILocalNotification`对象(原文此处有误，本地通知和推送通知的结构不是相同的)。

> UIApplicationLaunchOptionsLocalNotificationKey: 如果这个key出现在launchOptions里面，则说明app有一个本地通知等待处理。对应的值是一个UILocalNotification对象，表示触发的本地通知。

为了直观的展示，当app处于前台活动状态时应该如何显示一个本地通知的提示框。请看下面这个例子：

```objc
@import AVFoundation;

@interface AppDelegate ()
@property (readwrite, nonatomic, assign) SystemSoundID localNotificationSound;
@end
```


```objc
- (void)application:(UIApplication *)application
didReceiveLocalNotification:(UILocalNotification *)notification
{
    if (application.applicationState == UIApplicationStateActive) {
        UIAlertView *alertView =
            [[UIAlertView alloc] initWithTitle:notification.alertAction
                                       message:notification.alertBody
                                      delegate:nil
                             cancelButtonTitle:NSLocalizedString(@"OK", nil)
                             otherButtonTitles:nil];

        if (!self.localNotificationSound) {
            NSURL *soundURL = [[NSBundle mainBundle] URLForResource:@"Sosumi"
                                                      withExtension:@"wav"];
            AudioServicesCreateSystemSoundID((__bridge CFURLRef)soundURL, &_localNotificationSound);
        }
        AudioServicesPlaySystemSound(self.localNotificationSound);

        [alertView show];
    }
}

- (void)applicationWillTerminate:(UIApplication *)application {
    if (self.localNotificationSound) {
        AudioServicesDisposeSystemSoundID(self.localNotificationSound);
    }
}
```

## **定位事件**

想创造一个基于地理位置信息的本地社交图片签到软件？好吧，你已经晚了4年了。

但是不用怕！使用iOS的位置监测，你的app可以被定位事件启动：

> UIApplicationLaunchOptionsLocationKey: 表明app是为了响应定位事件才启动的。这个key对应的值是一个包含一个BOOL值的NSNumber对象。监测到这个key后，你应当创建CLLocationManager对象并再次启动定位服务。定位数据只会被投递给location manager的delegate，不会用这个key投递。

下面这个例子，展示了当app收到一个显著的位置变化(significant location)事件而启动时的处理过程：

``` objc
@import CoreLocation;

@interface AppDelegate () <CLLocationManagerDelegate>
@property (readwrite, nonatomic, strong) CLLocationManager *locationManager;
@end
```

``` objc
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ...

    if (![CLLocationManager locationServicesEnabled]) {
        [[[UIAlertView alloc] initWithTitle:NSLocalizedString(@"Location Services Disabled", nil)
                                    message:NSLocalizedString(@"You currently have all location services for this device disabled. If you proceed, you will be asked to confirm whether location services should be reenabled.", nil)
                                   delegate:nil
                          cancelButtonTitle:NSLocalizedString(@"OK", nil)
                          otherButtonTitles:nil] show];
    } else {
        self.locationManager = [[CLLocationManager alloc] init];
        self.locationManager.delegate = self;
        [self.locationManager startMonitoringSignificantLocationChanges];
    }

    if (launchOptions[UIApplicationLaunchOptionsLocationKey]) {
        [self.locationManager startUpdatingLocation];
    }
}
```

## **Newsstand**

所有Newsstand开发者，欢呼吧。

好吧好吧。

Newsstand可以在有新的下载资源时被启动。

你可以像这样注册：

``` objc
[application registerForRemoteNotificationTypes:
    UIRemoteNotificationTypeNewsstandContentAvailability];
```

接下来就是launchOptions里面的关键部分：

> UIApplicationLaunchOptionsNewsstandDownloadsKey: 表明有新的Newsstand资源可供你的app下载。这个key对应的值是一组字符串标示符，它们标示了可供下载的资源对应的NKAssetDownload对象。虽然，这些标示符可以用作重复核对的用途，但是，你应当通过NKLibrary对象(表示Newsstand app的资源库)的downloadingAssets属性来明确的获取NKAssetDownload对象数组(它们表示了正在下载或者有错误的资源)。

除此之外，就没有什么需要多说的了。

## **Bluetooth**

iOS7引入了一个新功能，允许app被蓝牙外围设备重新启动。

如果app用特定的标示符实例化了一个[CBCentralManager](https://developer.apple.com/library/ios/documentation/CoreBluetooth/Reference/CBCentralManager_Class/translated_content/CBCentralManager.html)或一个[CBPeripheralManager](https://developer.apple.com/library/ios/documentation/CoreBluetooth/Reference/CBPeripheralManager_Class/Reference/CBPeripheralManager.html#//apple_ref/doc/uid/TP40013015)对象，并且连接了其他蓝牙外围设备，app就可以被来自蓝牙系统的中央操作(certain actions)重新启动。取决于接到通知的是中央设备管理器还是外围设备管理器，launchOptions里面会包含下面这两个key中的其中一个：

> UIApplicationLaunchOptionsBluetoothCentralsKey: 表明app先前有一个或多个CBCentralManager对象，现在app被蓝牙系统重新启动以继续处理这些对象的相关操作。这个key对应的值是一个包含了一个或多个NSString对象的数组。 每个字符串代表了一个中央设备管理器对象的复位标示符。
> 
> UIApplicationLaunchOptionsBluetoothPeripheralsKey: 表明app先前有一个或多个CBPeripheralManager对象，现在app被蓝牙系统重新启动以继续处理这些对象的相关操作。这个key对应的值是一个包含了一个或多个NSString对象的数组。每个字符串代表了一个外围设备管理器对象的复位标示符。

```objc
@import CoreBluetooth;

@interface AppDelegate () <CBCentralManagerDelegate>
@property (readwrite, nonatomic, strong) CBCentralManager *centralManager;
@end
```

```objc
self.centralManager = [[CBCentralManager alloc] initWithDelegate:self queue:nil options:@{CBCentralManagerOptionRestoreIdentifierKey:(launchOptions[UIApplicationLaunchOptionsBluetoothCentralsKey] ?: [[NSUUID UUID] UUIDString])}];

if (self.centralManager.state == CBCentralManagerStatePoweredOn) {
    static NSString * const UID = @"7C13BAA0-A5D4-4624-9397-15BF67161B1C"; // generated with `$ uuidgen`
    NSArray *services = @[[CBUUID UUIDWithString:UID]];
    NSDictionary *scanOptions = @{CBCentralManagerScanOptionAllowDuplicatesKey:@YES};
    [self.centralManager scanForPeripheralsWithServices:services options:scanOptions];
}
```

## 总结

要记住这么多app启动的方法和手段可能会让人筋疲力尽。幸运的是，任何一个app可能只需要处理其中的一两种情况。

知道什么是可能的，通常是能让一个app从概念演变成实现的必要条件。因此记住所有这些选项吧，以备你脑海中浮现出来的下一个伟大想法。

译自: [NSHipster: UIApplication​Delegate launch​Options](http://nshipster.com/launch-options/)