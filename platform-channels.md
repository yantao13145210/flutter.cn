---
layout: page
title: Writing custom platform-specific code with platform channels
permalink: /platform-channels/
---

This guide describes how to write custom platform-specific code. If you just
want to use existing platform-specific plugins, please see [platform
plugins](/platform-plugins/).

* TOC
{:toc}

Flutter uses a flexible system that allows you to call platform-specific APIs
whether available in Java code on Android, or in ObjectiveC or Swift code on
iOS.

Flutter's platform-specific API support does not rely on code generation, but
rather on a flexible message passing style:

* The Flutter portion of your app sends messages to it's *host*, the iOS or
 Android portion of your app, over a platform channel.

* The *host* listens on the platform channel, and receives the message. It then
 calls into any number of platform-specific APIs -- using the native
 programming language -- and sends back a response to the *client*, the Flutter
 portion of your app.

## Architectural overview: platform channels

Messages are passed between the client (UI) and host (platform) using platform
channels as illustrated in this diagram:

![Platform channels architecture](/images/PlatformChannels.png)

Messages and responses are passed asynchronously, to ensure the user interface
remains responsive.

On the client side, `PlatformMessageChannel` ([API][PlatformMessageChannel])
enables sending general purpose messages, and `PlatformMethodChannel`
([API][PlatformMethodChannel]) enables sending messages that correspond to
method calls.

On the platform side, `FlutterMessageChannel` ([Android
API][FlutterMessageChannelAndroid], [iOS API][FlutterChanneliOS]) enables
receiving messages from the client, and optionally sending back a result, and
`FlutterMethodChannel` ([Android API][FlutterMethodChannelAndroid], [iOS
API][FlutterChanneliOS]) enables receiving method calls and sending back a
result. These classes allow you to develop a platform plugin with very little
'boilerplate' code.

*Note*: If desired, method calls can also be sent in the reverse direction, with
the platform acting as client to methods implemented in Dart.

[PlatformMessageChannel]: https://docs.flutter.io/flutter/services/PlatformMessageChannel-class.html
[PlatformMethodChannel]: https://docs.flutter.io/flutter/services/PlatformMethodChannel-class.html
[FlutterMessageChannelAndroid]: https://docs.flutter.io/javadoc/io/flutter/plugin/common/FlutterMessageChannel.html
[FlutterMethodChannelAndroid]: https://docs.flutter.io/javadoc/io/flutter/plugin/common/FlutterMethodChannel.html
[FlutterChanneliOS]: https://github.com/flutter/engine/blob/master/shell/platform/darwin/ios/framework/Headers/FlutterChannels.h

## Platform channel data types support and codecs

The standard platform channels use a standard message codec that supports
efficient binary serialization of simple JSON-like values, such as booleans,
numbers, Strings, byte buffers, and List and Maps of these (see
[`StandardMessageCodec`](https://docs.flutter.io/flutter/services/StandardMessageCodec-class.html))
for details). The serialization and deserialization of these values to and from
messages happens automatically when you send and receive values.

For more specialized use cases, you may also use the
[`BinaryCodec`](https://docs.flutter.io/flutter/services/BinaryCodec-class.html),
[`StringCodec`](https://docs.flutter.io/flutter/services/StringCodec-class.html),
or
[`JSONMessageCodec`](https://docs.flutter.io/flutter/services/JSONMessageCodec-class.html).

## Example: Calling platform-specific iOS and Android code using platform channels

The following demonstrates how to call a platform-specific API to retrieve and
display the current battery level. It uses the Android `BatteryManager` API, and
the iOS `device.batteryLevel` API, via a single platform message,
`getBatteryLevel`.

*Note*: The full, runnable source-code for this example is available in
[`/examples/platform_channel/`](https://github.com/flutter/flutter/tree/master/examples/platform_channel)
for Android with Java and iOS with Objective-C. For iOS with Swift, see
[`/examples/platform_channel_swift/`](https://github.com/flutter/flutter/tree/master/examples/platform_channel_swift).

### Step 1: Create a new app project

Start by creating a new app using:

* In a terminal: `flutter create batterylevel`, or

* In IntelliJ: *File > New > Project...*, select Flutter, and use the project
 name 'battery level'

### Step 2: Create the Flutter platform client

The app's `State` class holds the current app state. We need to extend that to
hold the current battery state.

First, we construct the channel. We use a `PlatformMethodChannel` with a single
platform method that returns the battery level.

The client and host sides of a channel are connected through a channel name
passed in the channel constructor. All channel names used in a single app must
be unique; we recommend prefixing the channel name with a unique 'domain
prefix', e.g. `samples.flutter.io/battery`.

<!-- skip -->
```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
...
class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;
  static const platform = const PlatformMethodChannel('samples.flutter.io/battery');

  // Get battery level.
}
```

Next, we invoke a method on the method channel, specifying the concrete method
to call via the String identifier `getBatteryLevel`. The call may fail -- for
example if the platform does not support the platform API (such as when running
in a simulator), so we wrap the `invokeMethod` call in a try-catch statement.

We use the returned result to update our user interface state in `_batteryLevel`
inside `setState`.

<!-- skip -->
```dart
  // Get battery level.
  String _batteryLevel = 'Unknown battery level.';

  Future<Null> _getBatteryLevel() async {
    String batteryLevel;
    try {
      final int result = await platform.invokeMethod('getBatteryLevel');
      batteryLevel = 'Battery level at $result % .';
    } on PlatformException catch (e) {
      batteryLevel = "Failed to get battery level: '${e.message}'.";
    }

    setState(() {
      _batteryLevel = batteryLevel;
    });
  }
```

Finally, we replace the `build` method from the template to contain a small user
interface that displays the battery state in a string, and a button for
refreshing the valuer.

<!-- skip -->
```dart
@override
Widget build(BuildContext context) {
  return new Material(
    child: new Center(
      child: new Column(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          new RaisedButton(
            child: new Text('Get Battery Level'),
            onPressed: _getBatteryLevel,
          ),
          new Text(_batteryLevel),
        ],
      ),
    ),
  );
}
```


### Step 3a: Add an Android platform-specific implementation

Start by opening the Android host portion of your Flutter app in Android Studio:

1. Start Android Studio

1. Select the menu item 'File > Open...'

1. Navigate to the directory holding your Flutter app, and select the `android`
folder inside it. Click OK.

1. Open the file `MainActivity.java` located in the `java` folder in the Project
view.

Next, create a `FlutterMethodChannel` and set a `MethodCallHandler` inside the
`onCreate` method. Make sure to use the same channel name as was used on the
Flutter client side.

```java
import io.flutter.app.FlutterActivity;
import io.flutter.plugin.common.FlutterMethodChannel;
import io.flutter.plugin.common.FlutterMethodChannel.MethodCallHandler;
import io.flutter.plugin.common.FlutterMethodChannel.Response;
import io.flutter.plugin.common.MethodCall;

public class MainActivity extends FlutterActivity {
    private static final String CHANNEL = "samples.flutter.io/battery";

    @Override
    public void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);

        new FlutterMethodChannel(getFlutterView(), CHANNEL).setMethodCallHandler(
                new MethodCallHandler() {
                    @Override
                    public void onMethodCall(MethodCall call, Response response) {
                        // TODO
                    }
                });
    }
}
```

Next, we add the actual Android Java code that uses the Android battery APIs to
retrieve the battery level. This code is exactly the same as you would have
written in a native Android app.

Add the following as a new method in the activity class, below the `onCreate`
method, and also add the corresponding imports:

```java
private int getBatteryLevel() {
  int batteryLevel = -1;
  if (VERSION.SDK_INT >= VERSION_CODES.LOLLIPOP) {
    BatteryManager batteryManager = (BatteryManager) getSystemService(BATTERY_SERVICE);
    batteryLevel = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY);
  } else {
    Intent intent = new ContextWrapper(getApplicationContext()).
        registerReceiver(null, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
    batteryLevel = (intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) * 100) /
        intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1);
  }

  return batteryLevel;
}
```

```java
import android.content.ContextWrapper;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.BatteryManager;
import android.os.Build.VERSION;
import android.os.Build.VERSION_CODES;
import android.os.Bundle;
```

Finally, we complete the `onMethodCall` method we added earlier. We need to
handle a single platform method, `getBatteryLevel`, so we test for that in the
`call` argument. The implementation of this platform method simply calls the
Android code we wrote in the previous step, and passes back a response for both
the success and error cases using the `response` argument. If an unknown method
is called, we report that instead. Replace:

```java
public void onMethodCall(MethodCall call, Response response) {
    // TODO
}
```

with:

```java
@Override
public void onMethodCall(MethodCall call, Response response) {
    if (call.method.equals("getBatteryLevel")) {
        int batteryLevel = getBatteryLevel();

        if (batteryLevel != -1) {
            response.success(batteryLevel);
        } else {
            response.error("UNAVAILABLE", "Battery level not available.", null);
        }
    } else {
        response.notImplemented();
    }
}               
```

You should now be able to run the app on Android. If you are using the Android
Emulator, you can set the battery level in the Extended Controls panel
accessible from the `...` button in the toolbar.

### Step 3b: Add an iOS platform-specific implementation using Objective-C

*Note*: The following steps use Objective-C. If you prefer Swift, skip to step
3c.

Start by opening the iOS host portion of your Flutter app in Xcode:

1. Start Xcode

1. Select the menu item 'File > Open...'

1. Navigate to the directory holding your Flutter app, and select the `ios`
folder inside it. Click OK.

1. Make sure the Xcode projects builds without errors.

1. Open the file `AppDelegate.m` located under Runner > Runner in the Project
navigator.

Next, create a `FlutterMethodChannel` and add a handler inside the `application
didFinishLaunchingWithOptions:` method. Make sure to use the same channel name
as was used on the Flutter client side.

```objectivec
#import <Flutter/Flutter.h>

@implementation AppDelegate
- (BOOL)application:(UIApplication*)application didFinishLaunchingWithOptions:(NSDictionary*)launchOptions {
  FlutterViewController* controller = (FlutterViewController*)self.window.rootViewController;

  FlutterMethodChannel* batteryChannel = [FlutterMethodChannel
                                          methodChannelWithName:@"samples.flutter.io/battery"
                                          binaryMessenger:controller];

  [batteryChannel setMethodCallHandler:^(FlutterMethodCall* call, FlutterResultReceiver result) {
    // TODO
  }];
  return YES;
}
```

Next, we add the actual iOS ObjectiveC code that uses the iOS battery APIs to
retrieve the battery level. This code is exactly the same as you would have
written in a native iOS app.

Add the following as a new method in the `AppDelegate` class, just before `@end`:

```objectivec
- (int)getBatteryLevel {
  UIDevice* device = UIDevice.currentDevice;
  device.batteryMonitoringEnabled = YES;
  if (device.batteryState == UIDeviceBatteryStateUnknown) {
    return -1;
  } else {
    return (int)(device.batteryLevel * 100);
  }
}
```

Finally, we complete the `setMethodCallHandler` method we added earlier. We need
to handle a single platform method, `getBatteryLevel`, so we test for that in
the `call` argument. The implementation of this platform method simply calls the
iOS code we wrote in the previous step, and passes back a response for both
the success and error cases using the `result` argument. If an unknown method
is called, we report that instead.

```objectivec
[batteryChannel setMethodCallHandler:^(FlutterMethodCall* call, FlutterResultReceiver result) {
  if ([@"getBatteryLevel" isEqualToString:call.method]) {
    int batteryLevel = [self getBatteryLevel];

    if (batteryLevel == -1) {
      result([FlutterError errorWithCode:@"UNAVAILABLE"
                                 message:@"Battery info unavailable"
                                 details:nil]);
    } else {
      result(@(batteryLevel));
    }
  } else {
    result(FlutterMethodNotImplemented);
  }
}];
```

You should now be able to run the app on iOS. If you are using the iOS
Simulator, note that it does not support battery APIs, and the app will thus
display 'battery info unavailable'.

### Step 3c: Add an iOS platform-specific implementation using Swift

*Note*: The following steps are similar to step 3b, only using Swift rather than
Objective-C.

Start by opening the iOS host portion of your Flutter app in Xcode:

1. Start Xcode

1. Select the menu item 'File > Open...'

1. Navigate to the directory holding your Flutter app, and select the `ios`
folder inside it. Click OK.

Next, we add support for Swift in the standard template setup that uses
Objective-C:

1. Expand Runner > Runner in the Project navigator.

1. If present, delete the files `AppDelegate.h`, `AppDelegate.m`, and
`Supporting files/main.h`. Select 'Move to trash' when asked.

1. Add a new `AppDelegate.swift` main file:
  * Right-click the Runner>Runner folder, and select **New File...**. 
  * Select `Swift File`, and click **Next**
  * Name the file `AppDelegate.swift`
  * Select **Create Bridging Header** when asked.
1. Open `Runner-Bridging-Header.h`, and add a Flutter import:
    ```
    #import "../Flutter/Flutter.framework/Headers/Flutter.h"
    ```
1. Open the file `AppDelegate.swift` located under Runner > Runner in the Project
navigator, and replace `import Foundation` with:
    ```swift
    import UIKit
    import Flutter

    @UIApplicationMain
    @objc class AppDelegate: FlutterAppDelegate {
    }
    ```
1. Validate that the project builds (shortcut `cmd-b`).

With the project converted to use Swift for the main application class we can
now begin.

First we override `didFinishLaunchingWithOptions` to contain a
`FlutterMethodChannel` tied to the channel name `samples.flutter.io/battery`:

```swift
@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    let controller : FlutterViewController = window?.rootViewController as! FlutterViewController;
    let batteryChannel = FlutterMethodChannel.init(name: "samples.flutter.io/battery",
                                                   binaryMessenger: controller);
  
    batteryChannel.setMethodCallHandler({
      (call: FlutterMethodCall, result: FlutterResultReceiver) -> Void in
      // Handle battery messages.
    });
   
    return true
  }
}
```

Next, we add the actual iOS Swift code that uses the iOS battery APIs to
retrieve the battery level. This code is exactly the same as you would have
written in a native iOS app.

Add the following as a new method at the bottom of `AppDelegate.swift`:

```swift
private func receiveBatteryLevel(result: FlutterResultReceiver) {
  let device = UIDevice.current;
  device.isBatteryMonitoringEnabled = true;
  if (device.batteryState == UIDeviceBatteryState.unknown) {
    result(FlutterError.init(code: "UNAVAILABLE",
                             message: "Battery info unavailable",
                             details: nil));
  } else {
    result(Int(device.batteryLevel * 100));
  }
}
```

Finally, we complete the `setMethodCallHandler` method we added earlier. We need
to handle a single platform method, `getBatteryLevel`, so we test for that in
the `call` argument. The implementation of this platform method simply calls the
iOS code we wrote in the previous step. If an unknown method
is called, we report that instead.

```swift
batteryChannel.setMethodCallHandler({
  (call: FlutterMethodCall, result: FlutterResultReceiver) -> Void in
  if ("getBatteryLevel" == call.method) {
    receiveBatteryLevel(result: result);
  } else {
    result(FlutterMethodNotImplemented);
  }
});
```

You should now be able to run the app on iOS. If you are using the iOS
Simulator, note that it does not support battery APIs, and the app will thus
display 'Battery info unavailable.'.

## Separate platform plugin code from UI code

If you expect to use your platform-specific code in multiple Flutter apps, it
can be useful to separate the code into a platform plugin located in a directory
outside your main application. See [platform plugins](/platform-plugins/) for
details.

## Publish a platform plugin {#publish}

If you wish to share your platform-specific with other developers in the
Flutter ecosystem, please see [platform plugins](/platform-plugins/) for
details.

{% include note.html content="Sharing of platform plugins is currently not
supported, but Flutter is expected to support this in the future." %}