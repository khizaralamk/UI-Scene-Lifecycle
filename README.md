# Fixing the "UIScene lifecycle will soon be required" Warning in React Native iOS Apps

## Introduction

If you're developing a React Native iOS app and seeing this warning in your Xcode console:

```
UIScene lifecycle will soon be required. Failure to adopt will result in an assert in the future.
```

This article will guide you through the complete solution to resolve this warning by properly adopting the UIScene-based lifecycle in your React Native iOS application.

---

## Understanding the Problem

### What is UIScene?

UIScene was introduced in iOS 13 as part of Apple's move towards a more modular app architecture. It allows apps to have multiple windows and better handle multi-window scenarios on iPad. While your React Native app might not need multiple windows, Apple is making scene-based lifecycle the standard, and future iOS versions will require it.

### The Warning Explained

This warning appears because:
- Your app is still using the traditional `UIApplicationDelegate` lifecycle
- iOS is expecting scene-based lifecycle configuration
- Future iOS versions will enforce this requirement (resulting in runtime errors)

---

## Prerequisites

- A React Native project with iOS support
- Xcode installed
- Basic understanding of iOS project structure
- Your app's `Info.plist` file
- Your app's `AppDelegate.swift` file

---

## Solution: Step-by-Step Guide

### Step 1: Create the SceneDelegate.swift File

First, we need to create a new `SceneDelegate.swift` file in your iOS project. This file will handle the scene lifecycle events.

**Location**: `ios/[YourAppName]/SceneDelegate.swift`

**Code**:

```swift
import UIKit
import React
import React_RCTAppDelegate

@available(iOS 13.0, *)
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  var window: UIWindow?
  
  weak var appDelegate: AppDelegate?
  
  func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
    guard let windowScene = (scene as? UIWindowScene) else { return }
    
    guard let appDelegate = UIApplication.shared.delegate as? AppDelegate else { return }
    self.appDelegate = appDelegate
    
    let window = UIWindow(windowScene: windowScene)
    self.window = window
    appDelegate.window = window
    
    // Make navigation bars opaque to remove iOS blur/translucency (liquid glass effect)
    let navAppearance = UINavigationBarAppearance()
    navAppearance.configureWithOpaqueBackground()
    navAppearance.backgroundColor = UIColor(red: 11.0/255.0, green: 53.0/255.0, blue: 51.0/255.0, alpha: 1.0) // matches TEAL_900
    UINavigationBar.appearance().standardAppearance = navAppearance
    UINavigationBar.appearance().scrollEdgeAppearance = navAppearance
    UINavigationBar.appearance().compactAppearance = navAppearance
    UINavigationBar.appearance().isTranslucent = false
    UITabBar.appearance().isTranslucent = false
    
    if let factory = appDelegate.reactNativeFactory {
      factory.startReactNative(
        withModuleName: "pearls", // Replace with your app name
        in: window,
        launchOptions: nil
      )
    }
  }
  
  func sceneDidDisconnect(_ scene: UIScene) {
    // Called as the scene is being released by the system.
  }
  
  func sceneDidBecomeActive(_ scene: UIScene) {
    // Called when the scene has moved from an inactive state to an active state.
  }
  
  func sceneWillResignActive(_ scene: UIScene) {
    // Called when the scene will move from an active state to an inactive state.
  }
  
  func sceneWillEnterForeground(_ scene: UIScene) {
    // Called as the scene transitions from the background to the foreground.
  }
  
  func sceneDidEnterBackground(_ scene: UIScene) {
    // Called as the scene transitions from the foreground to the background.
  }
}
```

**Key Points**:
- Replace `"pearls"` with your actual app/module name
- The `scene(_:willConnectTo:options:)` method is where your React Native app gets initialized
- All scene lifecycle methods are implemented (they can be left empty for basic functionality)

---

### Step 2: Update AppDelegate.swift

Now we need to modify your `AppDelegate.swift` to support scene configuration while maintaining backward compatibility with iOS versions below 13.

**Location**: `ios/[YourAppName]/AppDelegate.swift`

**Changes Needed**:

1. **Modify `application(_:didFinishLaunchingWithOptions:)`** to conditionally create the window:

```swift
func application(
  _ application: UIApplication,
  didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
) -> Bool {
  let delegate = ReactNativeDelegate()
  let factory = RCTReactNativeFactory(delegate: delegate)
  delegate.dependencyProvider = RCTAppDependencyProvider()

  reactNativeDelegate = delegate
  reactNativeFactory = factory

  // Only create window if not using scenes (iOS < 13 or scene delegate not available)
  if #available(iOS 13.0, *) {
    // SceneDelegate will handle window creation
  } else {
    window = UIWindow(frame: UIScreen.main.bounds)
    
    // Make navigation bars opaque to remove iOS blur/translucency (liquid glass effect)
    UINavigationBar.appearance().isTranslucent = false
    UINavigationBar.appearance().barTintColor = UIColor(red: 11.0/255.0, green: 53.0/255.0, blue: 51.0/255.0, alpha: 1.0)
    UITabBar.appearance().isTranslucent = false

    factory.startReactNative(
      withModuleName: "pearls", // Replace with your app name
      in: window,
      launchOptions: launchOptions
    )
  }

  return true
}
```

2. **Add Scene Configuration Methods** at the end of your `AppDelegate` class:

```swift
// MARK: UISceneSession Lifecycle

@available(iOS 13.0, *)
func application(_ application: UIApplication, configurationForConnecting connectingSceneSession: UISceneSession, options: UIScene.ConnectionOptions) -> UISceneConfiguration {
  let sceneConfig = UISceneConfiguration(name: nil, sessionRole: connectingSceneSession.role)
  sceneConfig.delegateClass = SceneDelegate.self
  return sceneConfig
}

@available(iOS 13.0, *)
func application(_ application: UIApplication, didDiscardSceneSessions sceneSessions: Set<UISceneSession>) {
  // Called when the user discards a scene session.
}
```

**Key Points**:
- The window is only created on iOS < 13 for backward compatibility
- On iOS 13+, the SceneDelegate handles window creation
- The scene configuration method tells iOS which delegate class to use for scenes

---

### Step 3: Update Info.plist

The `Info.plist` file needs to declare that your app uses scene-based lifecycle.

**Location**: `ios/[YourAppName]/Info.plist`

**Add the following dictionary** (typically before `UIViewControllerBasedStatusBarAppearance`):

```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
            </dict>
        </array>
    </dict>
</dict>
```

**Complete Example** (showing context):

```xml
<key>UISupportedInterfaceOrientations~ipad</key>
<array>
    <string>UIInterfaceOrientationPortrait</string>
</array>
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
            </dict>
        </array>
    </dict>
</dict>
<key>UIViewControllerBasedStatusBarAppearance</key>
<false/>
```

**Key Points**:
- `UIApplicationSupportsMultipleScenes` is set to `false` (we only need one scene)
- `$(PRODUCT_MODULE_NAME)` automatically resolves to your app's module name
- This tells iOS to use `SceneDelegate` for scene management

---

### Step 4: Add SceneDelegate.swift to Xcode Project

If you created the file manually, you need to add it to your Xcode project:

1. Open your project in Xcode
2. Right-click on your app folder in the Project Navigator
3. Select "Add Files to [YourAppName]..."
4. Navigate to and select `SceneDelegate.swift`
5. Make sure "Copy items if needed" is **unchecked** (the file is already in the right location)
6. Make sure your app target is checked
7. Click "Add"

Alternatively, you can add the file reference manually in `project.pbxproj`, but using Xcode's UI is recommended.

---

### Step 5: Clean and Rebuild

After making all the changes:

1. **Clean Build Folder**: In Xcode, go to `Product > Clean Build Folder` (Shift + Cmd + K)
2. **Delete Derived Data** (optional but recommended):
   - Close Xcode
   - Delete `~/Library/Developer/Xcode/DerivedData`
   - Or use: `rm -rf ~/Library/Developer/Xcode/DerivedData`
3. **Rebuild**: Build and run your app (Cmd + R)

---

## Verification

After rebuilding, you should:

1. ✅ No longer see the UIScene lifecycle warning in the console
2. ✅ Your app should run normally on both iOS 13+ and older versions
3. ✅ All your existing functionality should work as before

---

## Understanding the Changes

### Why This Works

1. **SceneDelegate.swift**: Handles all scene-related lifecycle events on iOS 13+. This is where your React Native app initialization happens when using scenes.

2. **AppDelegate Updates**: 
   - Maintains backward compatibility by only creating windows on iOS < 13
   - Provides scene configuration methods that iOS calls to set up scenes
   - Delegates window management to SceneDelegate on iOS 13+

3. **Info.plist Configuration**: Tells iOS that your app supports scenes and specifies which delegate class to use.

### Backward Compatibility

The solution maintains backward compatibility:
- **iOS 13+**: Uses SceneDelegate for scene-based lifecycle
- **iOS < 13**: Falls back to traditional AppDelegate window management

---

## Common Issues and Troubleshooting

### Issue 1: "Cannot find 'SceneDelegate' in scope"

**Solution**: Make sure `SceneDelegate.swift` is added to your Xcode project target. Check the Target Membership in the File Inspector.

### Issue 2: App crashes on launch

**Solution**: 
- Verify that your module name in `SceneDelegate.swift` matches your actual app name
- Check that `reactNativeFactory` and `reactNativeDelegate` are properly initialized in AppDelegate before SceneDelegate tries to use them
- Ensure all scene lifecycle methods are implemented (even if empty)

### Issue 3: Navigation bar styling doesn't work

**Solution**: Make sure you've copied your navigation bar styling code to both:
- The SceneDelegate's `scene(_:willConnectTo:options:)` method (for iOS 13+)
- The AppDelegate's `application(_:didFinishLaunchingWithOptions:)` else block (for iOS < 13)

### Issue 4: Build errors related to React Native

**Solution**: 
- Clean build folder and derived data
- Run `pod install` in the `ios` directory
- Make sure all React Native dependencies are up to date

---

## Important Notes

1. **Module Name**: Replace `"pearls"` with your actual app/module name throughout the code. This is typically the name registered in your `index.js` file.

2. **Custom Styling**: If you have custom navigation bar, tab bar, or other UI configurations, make sure they're applied in both:
   - SceneDelegate (for iOS 13+)
   - AppDelegate's else block (for iOS < 13)

3. **React Native Version**: This solution works with React Native's new architecture (Fabric) as well as the old architecture. The code uses `RCTReactNativeFactory` which is compatible with both.

4. **Testing**: Always test on both iOS 13+ devices/simulators and older versions (if you support them) to ensure backward compatibility.

---

## Conclusion

By following these steps, you've successfully adopted the UIScene-based lifecycle in your React Native iOS app. This ensures:

- ✅ Compliance with Apple's future requirements
- ✅ No more warnings in your Xcode console
- ✅ Backward compatibility with older iOS versions
- ✅ Better foundation for future iOS features

Your app is now future-proof and ready for upcoming iOS versions that will require scene-based lifecycle.

---

## Additional Resources

- [Apple's UIScene Documentation](https://developer.apple.com/documentation/uikit/app_and_environment/scenes)
- [React Native iOS Setup](https://reactnative.dev/docs/environment-setup)
- [Understanding Scene Delegates](https://developer.apple.com/documentation/uikit/uiscenedelegate)

---

**Happy Coding! 🚀**

