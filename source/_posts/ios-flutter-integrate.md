---
title: Integrate Flutter for existing iOS projects
date: 2018-07-03 19:49:20
tags:
---

After some baptism, Flutter finally ushered in the Release Preview 1 version, which also attracted more people's attention. Using Flutter to write an App from scratch is very easy, but more and more people find that Flutter does not seem to be very friendly to support existing App access. So this article takes you to understand how to make existing apps support Flutter.

<!-- more-->
# surroundings
> Flutter: 0.5.1
> Xcode 9.4.1
> Flutter project: flutter/examples/hello_world

# Debug: Flutter Hot Restart
We all know that in the development environment, Flutter's hot restart is very helpful for UI rapid prototyping. It is not difficult to make existing apps support Flutter and enable hot restart.

> The following assumes that your project file is `YourApp.xcodeproj`

## Flutter(engine) basic library
First, drag in `Flutter.framework` into your project. This library is Flutter Engine, which hosts the Dart runtime and drawing engine. There is a one-to-one correspondence between `Flutter.framework` and the command-line tool version. If you don’t know where to find this file, you can do a `flutter run` directly in the Flutter source code project, and then you can go to `/<project> Found it under the /ios/Flutter/` directory, just drag it into the project.


## Flutter ViewController
Next, you need to introduce the basic code of Flutter into the existing project, so that the basic Flutter ViewController can display the Flutter view. This step is very simple, just push the past in your existing ViewController:

``ʻobjc
-(void)jumpToFlutter {
    FlutterViewController *viewController = [FlutterViewController new];
    [self.navigationController pushViewController:viewController animated:YES];
}
```

But you also need to pay attention to the need to pass the life cycle events in AppDelegate to Flutter

> After Release Preview 1, the document mentioned that there is a protocol called `FlutterAppLifeCycleProvider`, but 0.5.1 is not there yet, so here comes the wild way

1. Directly let the existing AppDelegate inherit `FlutterAppDelegate`, but the negative impact of this is that the root ViewController is set to Flutter ViewController

2. Modify AppDelegate by yourself. Many medium and large apps like to modularize their business during development. Fortunately, Flutter APPDelegate is not difficult to transform, and can also support module Delegate. First let your Delegate follow the `FlutterPluginRegistry` protocol

``ʻobjc
// AppDelegate or Delegate of the module

@interface DemoFlutterBaseAppDelegate: NSObject <ModularApplicationDelegate, FlutterPluginRegistry>

/**
 FlutterBinaryMessenger
 this determines which view controller is the flutter view controller
 nomally, flutter view controller provides the binary messages
 @return root Flutter ViewController
 */
-(NSObject<FlutterBinaryMessenger> *)binaryMessenger;

/**
 FlutterTextureRegistry
 this determines which view controller is the flutter view controller
 nomally, flutter view controller provides the custom textures
 @return root Flutter ViewController
 */
-(NSObject<FlutterTextureRegistry> *)textures;

@end

```

Then support Flutter messenger and texture in the implementation


```objc

// Registrar declaration and implementation

@interface DemoFlutterAppDelegateRegistrar: NSObject<FlutterPluginRegistrar>

@property(nonatomic, copy) NSString *pluginKey;
@property(nonatomic, strong) DemoFlutterBaseAppDelegate *appDelegate;

-(instancetype)initWithPlugin:(NSString*)pluginKey appDelegate:(DemoFlutterBaseAppDelegate*)delegate;

@end

@implementation DemoFlutterAppDelegateRegistrar
-(instancetype)initWithPlugin:(NSString*)pluginKey appDelegate:(DemoFlutterBaseAppDelegate*)appDelegate {
    self = [super init];
    NSAssert(self, @"Super init cannot be nil");
    _pluginKey = [pluginKey copy];
    _appDelegate = appDelegate;
    return self;
}


-(NSObject<FlutterBinaryMessenger>*)messenger {
    return [_appDelegate binaryMessenger];
}

-(NSObject<FlutterTextureRegistry>*)textures {
    return [_appDelegate textures];
}

-(void)publish:(NSObject*)value {
    _appDelegate.pluginPublications[_pluginKey] = value;
}

-(void)addMethodCallDelegate:(NSObject<FlutterPlugin>*)delegate
                      channel:(FlutterMethodChannel*)channel {
    [channel setMethodCallHandler:^(FlutterMethodCall* call, FlutterResult result) {
        [delegate handleMethodCall:call result:result];
    }];
}

-(void)addApplicationDelegate:(NSObject<FlutterPlugin>*)delegate {
    [_appDelegate.pluginDelegates addObject:delegate];
}

-(NSString*)lookupKeyForAsset:(NSString*)asset {
    return [FlutterDartProject lookupKeyForAsset:asset];
}

-(NSString*)lookupKeyForAsset:(NSString*)asset fromPackage:(NSString*)package {
    return [FlutterDartProject lookupKeyForAsset:asset fromPackage:package];
}

@end


// AppDelegate implementation

@interface DemoFlutterBaseAppDelegate ()

@property(nonatomic, strong) DemoFlutterBaseViewController *rootController;
@property(readonly, nonatomic) NSMutableArray* pluginDelegates;
@property(readonly, nonatomic) NSMutableDictionary* pluginPublications;

@end

@implementation DemoFlutterBaseAppDelegate

/*
 * ... write and forward various life cycle events here to plugin
 */
-(BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    for (id<FlutterPlugin> plugin in _pluginDelegates) {
        if ([plugin respondsToSelector:_cmd]) {
            [plugin application:application didFinishLaunchingWithOptions:launchOptions];
        }
    }
    return YES;
}

#pragma mark-getters for flutter

// Return to FlutterViewController instance
-(FlutterViewController *)rootController {
    // ...
}

-(NSObject<FlutterBinaryMessenger> *)binaryMessenger
{
    if ([self.rootController conformsToProtocol:@protocol(FlutterBinaryMessenger)]) {
        return (NSObject<FlutterBinaryMessenger> *)self.rootController;
    }
    return nil;
}

-(NSObject<FlutterTextureRegistry> *)textures
{
    if ([self.rootController conformsToProtocol:@protocol(FlutterTextureRegistry)]) {
        return (NSObject<FlutterTextureRegistry> *)self.rootController;
    }
    return nil;
}

-(NSObject<FlutterPluginRegistrar>*)registrarForPlugin:(NSString*)pluginKey {
    NSAssert(self.pluginPublications[pluginKey] == nil, @"Duplicate plugin key: %@", pluginKey);
    self.pluginPublications[pluginKey] = [NSNull null];
    return [[DemoFlutterAppDelegateRegistrar alloc] initWithPlugin:pluginKey appDelegate:self];
}

-(BOOL)hasPlugin:(NSString*)pluginKey {
    return _pluginPublications[pluginKey] != nil;
}

-(NSObject*)valuePublishedByPlugin:(NSString*)pluginKey {
    return _pluginPublications[pluginKey];
}

@end

```


In this way, the operating environment of Flutter is actually ready, whether it is Hot Restart or AOT can support. Next we implement Debug Hot Restart
First execute the `Flutter build bundle` in your Flutter code directory, which can help us package out a Flutter Asset, and then drag the `flutter_assets` directory into the project.

Make a build of your project to make sure you can get an .app file. Then create a new folder called `Payload`, put the .app file into the Payload folder, and then compress it into a zip file. This file can be used by the Flutter command line tool.

```
flutter run --use-application-binary /path/to/Payload.zip
```

Then the effect is as follows:
![](/images/flutter-hot-restart.gif)

> Actually, Hot Reload is not implemented here, mainly because the current flutter toolchain support is not good, but Hot Restart is enough.

# Release: Flutter AOT
Release mode is different from Debug mode. We need to do several things:

1. Replace `Flutter.framework` with `flutter/bin/cache/artifacts/engine/ios-release/Flutter.framework`, because the library we used in the previous step is actually JIT Runtime
2. Execute `flutter build aot --release --target-platform ios --ios-arch armv7,arm64` under the Flutter code project. Then we can get a packaged ʻApp.framework` in the build directory, but Don't forget to put an Info.plist in it. And drag this library into the project
3. Delete the files ʻisolate_snapshot_data`, `kernel_blob.bin`, `platform.dill`, and `vm_snapshot_data` under the folder `flutter_assets` in the project
4. Compile and package to run on real machine, the effect is as follows:

<img src="/images/flutter-aot-release.gif" width="360px;"/>


# The End
In fact, Flutter officially has plans to support the integration of existing apps, and there are some introductions in the document, but in fact, the overall tool chain has not yet been supported, and the current support level is similar to the above method.
If necessary, existing projects can be integrated using the above methods. In order to reduce the work flow, some work needs to be done, such as:

1. The empty shells of App.framework and Flutter.framework are provided in the project, which is convenient to replace with scripts at any time under debug and release
2. When debugging, you can first type out the package for Flutter developers, or you can directly add a build post action, directly call the flutter command line, and integrate Xcode and flutter. Don't do all manual work like the above, which is easy to miss the necessary process
3. The automation of Release AOT must be done and integrated with existing CI

