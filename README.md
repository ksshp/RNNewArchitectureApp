# RUN

This branch contains all the step executed to:

1. Create an App starting from the version 0.67.4
1. Migrate it to React Native 0.70.0, adopting the New Architecture.
1. Create a TurboModule.
1. Create a Fabric Component.

## Table of Content

* App Setup
    * [[Setup] Run npx react-native init AwesomeApp --version 0.67.4](#setup)
    * [[Migration] Upgrade to 0.69](#move-to-0.70)
    * [[iOS] Use Objective-C++ (.mm extension)](#configure-objcpp)
    * [[Android] Configure Gradle for CodeGen](#android-setup)
* TurboModule Setup
    * [[TurboModule Setup - iOS] Ensure your App Provides an `RCTCxxBridgeDelegate`](#ios-tm)
    * [[TurboModule Setup - iOS] Provide a TurboModuleManager Delegate](#ios-tm-manager-delegate)
    * [[TurboModule Setup - iOS] Install TurboModuleManager JavaScript Bindings](#ios-tm-js-bindings)
    * [[TurboModule Setup - iOS] Enable TurboModule System](#ios-enable-tm)
    * [[TurboModule Setup - Android] Enable NDK and the native build](#turbomodule-ndk)
    * [[TurboModule Setup - Android] Provide a `ReactPackageTurboModuleManagerDelegate`](#java-tm-delegate)
    * [[TurboModule Setup - Android] Adapt your ReactNativeHost to use the `ReactPackageTurboModuleManagerDelegate`](#java-tm-adapt-host)
    * [[TurboModule Setup - Android] C++ Provide a native implementation for the methods in your *TurboModuleDelegate class](#cpp-tm-manager)
    * [[TurboModule Setup - Android] Enable the useTurboModules flag in your Application onCreate](#enable-tm-android)
* Fabric Setup
    * [[Fabric Setup - iOS] Update your root view](#fabric-root-view)

## Steps

### <a name="setup" />[[Setup] Run npx react-native init AwesomeApp --version 0.67.4]()

1. `npx react-native init AwesomeApp --version 0.67.4`
1. `cd AwesomeApp`
1. `npx react-native start (in another terminal)`
1. `npx react-native run-ios`
1. `npx react-native run-android`

### <a name="move-to-0.70" />[[Migration] Upgrade to 0.70]()

1. `cd AwesomeApp`
1. `yarn add react@18.0.0` to upgrade to React18
1. `yarn add react-native@0.70.0-rc.0`
1. Open the `AwesomeApp/ios/AwesomeApp/AppDelegate.m` file and update it as it follows:
    ```diff
        - (NSURL *)sourceURLForBridge:(RCTBridge *)bridge
        {
        #if DEBUG
    -       return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
    +       return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index"];
        #else
            return [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
        #endif
        }
    ```
1. Open the `ios/Podfile` file and update it as it follows:
    ```diff
    - platform :ios, '11.0'
    + platform :ios, '12.4'
    ```
1. At the same level of the `AwesomeApp.xcodeproj`, create an `.xcode.env` file with this content:
    ```
    export NODE_BINARY=$(command -v node)
    ```
    Eventually, replacing the `$(command -v node)` with the actual path to node.
1. `cd ios`
1. `bundle exec pod install`
1. Open the `android/build.gradle` file and update the `buildscript.ext` block with the following code
    ```kotlin
    buildscript {
        ext {
            buildToolsVersion = "31.0.0"
            minSdkVersion = 21
            compileSdkVersion = 31
            targetSdkVersion = 31
            if (System.properties['os.arch'] == "aarch64") {
                // For M1 Users we need to use the NDK 24 which added support for aarch64
                ndkVersion = "24.0.8215888"
            } else {
                // Otherwise we default to the side-by-side NDK version from AGP.
                ndkVersion = "21.4.7075529"
            }
        }
    }
    ```
1. Open the `android/app/src/main/AndroidManifest.xml` file and add this line:
    ```diff
    android:windowSoftInputMode="adjustResize"
    + android:exported="true">
    <intent-filter>
    ```
1. Open the `android/app/build.gradle` and add the following function:
    ```kotlin
    def isNewArchitectureEnabled() {
        // To opt-in for the New Architecture, you can either:
        // - Set `newArchEnabled` to true inside the `gradle.properties` file
        // - Invoke gradle with `-newArchEnabled=true`
        // - Set an environment variable `ORG_GRADLE_PROJECT_newArchEnabled=true`
        return project.hasProperty("newArchEnabled") && project.newArchEnabled == "true"
    }
    ```
1. Open the `android/gradle.properties` and add the `newArchEnabled=true` property to it.
1. `npx react-native run-ios && npx react-native run-android`

### <a name="configure-objcpp">[[iOS] Use Objective-C++ (.mm extension)]()

1. Open the `AwesomeApp/ios/AwesomeApp.xcworkspace` in Xcode
1. Rename all the `.m` files to `.mm`:
    1. `main.m` will be renamed to `main.mm`
    1. `AppDelegate.m` will be renamed to `AppDelegate.mm`
1. Run `npx react-native run-ios`

**Note:** Renaming files in Xcode also updates the `xcodeproj` file automatically.

### <a name="android-setup" />[[Android] Configure Gradle for CodeGen]()

1. Navigate to `AwesomeApp/android` folder
1. Update Gradle running: `./gradlew wrapper --gradle-version 7.3.3 --distribution-type=all`
1. Open the `AwesomeApp/android/settings.gradle` file and add the following lines:
    ```diff
    apply from: file("../node_modules/@react-native-community/cli-platform-android/native_modules.gradle"); applyNativeModulesSettingsGradle(settings)
    include ':app'
    + includeBuild('../node_modules/react-native-gradle-plugin')

    + include(":ReactAndroid")
    + project(":ReactAndroid").projectDir = file('../node_modules/react-native/ReactAndroid')
    + include(":ReactAndroid:hermes-engine")
    + project(":ReactAndroid:hermes-engine").projectDir = file('../node_modules/react-native/ReactAndroid/hermes-engine')
    ```
1. Open the `AwesomeApp/android/build.gradle` file and update the gradle dependency:
    ```diff
        //...
        repositories {
            google()
            mavenCentral()
        }
        dependencies {
    -        classpath("com.android.tools.build:gradle:4.2.2")
    +        classpath("com.android.tools.build:gradle:7.2.0")

    +        classpath("com.facebook.react:react-native-gradle-plugin")
    +        classpath("de.undercouch:gradle-download-task:4.1.2")

            // NOTE: Do not place your application dependencies here; they belong
            // in the individual module build.gradle files

        }
    }
    ```
1. Open the `android/app/build.gradle` and add the following snippet:
    ```diff
    project.ext.react = [
    -    enableHermes: true,  // clean and rebuild if changing
    +    enableHermes: true,  // clean and rebuild if changing
    ]
    // ...

    }

    if (enableHermes) {
    -    def hermesPath = "../../node_modules/hermes-engine/android/";
    -    debugImplementation files(hermesPath + "hermes-debug.aar")
    -    releaseImplementation files(hermesPath + "hermes-release.aar")
    +    //noinspection GradleDynamicVersion
    +    implementation("com.facebook.react:hermes-engine:+") { // From node_modules
    +        exclude group:'com.facebook.fbjni'
    +    }
    } else {

    // ...

    + configurations.all {
    +     resolutionStrategy.dependencySubstitution {
    +         substitute(module("com.facebook.react:react-native"))
    +                 .using(project(":ReactAndroid"))
    +                 .because("On New Architecture we're building React Native from source")
    +         substitute(module("com.facebook.react:hermes-engine"))
    +                .using(project(":ReactAndroid:hermes-engine"))
    +                .because("On New Architecture we're building Hermes from source")
    +     }
    + }

    // Run this once to be able to run the application with BUCK
    // puts all compile dependencies into folder libs for BUCK to use
    task copyDownloadableDepsToLibs(type: Copy) {
    ```
1. Open the `AwesomeApp/android/app/proguard-rules.pro` and update the file adding these lines:
    ```sh
    -keep class com.facebook.hermes.unicode.** { *; }
    -keep class com.facebook.jni.** { *; }
    ```
1. Run `./gradlew clean`
1. Go back to the `AwesomeApp` folder
1. `npx react-native run-android` (Don't worry if it takes some time to complete.)

### <a name="ios-tm" /> [[TurboModule Setup - iOS] Ensure your App Provides an `RCTCxxBridgeDelegate`]()

1. Open the `AppDelegate.mm` file
1. Add the following imports:
    ```objc
    #import <reacthermes/HermesExecutorFactory.h>
    #import <React/RCTCxxBridgeDelegate.h>
    #import <React/RCTJSIExecutorRuntimeInstaller.h>
    ``
1. Add the following `@interface`, right before the `@implementation` keyword
    ```obj-c
    @interface AppDelegate () <RCTCxxBridgeDelegate> {
    // ...
    }
    @end
    ```
1. Add the following function at the end of the file, before the `@end` keyword:
    ```obj-c
    #pragma mark - RCTCxxBridgeDelegate

    - (std::unique_ptr<facebook::react::JSExecutorFactory>)jsExecutorFactoryForBridge:(RCTBridge *)bridge
    {
    return std::make_unique<facebook::react::HermesExecutorFactory>(facebook::react::RCTJSIExecutorRuntimeInstaller([bridge](facebook::jsi::Runtime &runtime) {
        if (!bridge) {
            return;
        }
        })
    );
    }
    ```
1. Open the `AwesomeApp/ios/Podfile` and change the `hermes_enabled` from `false` to `true`
1. Open the `Podfile` and update the pod configurations as it follows:
    ```diff
    use_react_native!(
        :path => config[:reactNativePath],
        # to enable hermes on iOS, change `false` to `true` and then install pods
        :hermes_enabled => true,
    +    :fabric_enabled => true,
    +    :app_path => "#{Pod::Config.instance.installation_root}/.."
    )
    ```
1. `cd ios && RCT_NEW_ARCH_ENABLED=1 bundle exec pod install`
1. From the `AwesomeApp` folder, run the app: `npx react-native ru-ios`

### <a name="ios-tm-manager-delegate" />[[TurboModule Setup - iOS] Provide a TurboModuleManager Delegate]()

1. Open the `AwesomeApp/ios/AwesomeApp/AppDelegate.mm`
1. Add the following imports:
    ```objc
    #import <ReactCommon/RCTTurboModuleManager.h>
    #import <React/CoreModulesPlugins.h>

    #import <React/RCTDataRequestHandler.h>
    #import <React/RCTHTTPRequestHandler.h>
    #import <React/RCTFileRequestHandler.h>
    #import <React/RCTNetworking.h>
    #import <React/RCTImageLoader.h>
    #import <React/RCTGIFImageDecoder.h>
    #import <React/RCTLocalAssetImageLoader.h>
    ```
1. Add the following code in the `@interface`
    ```objc
    @interface AppDelegate () <RCTCxxBridgeDelegate, RCTTurboModuleManagerDelegate> {
        // ...
        RCTTurboModuleManager *_turboModuleManager;
    }
    @end
    ```
1. Implement the `getModuleClassFromName`:
    ```c++
    #pragma mark RCTTurboModuleManagerDelegate

    - (Class)getModuleClassFromName:(const char *)name
    {
    return RCTCoreModulesClassProvider(name);
    }
    ```
1. Implement the `(std::shared_ptr<facebook::react::TurboModule>) getTurboModule:(const std::string &)name jsInvoker:(std::shared_ptr<facebook::react::CallInvoker>)jsInvoker`:
    ```c++
    - (std::shared_ptr<facebook::react::TurboModule>)
        getTurboModule:(const std::string &)name
             jsInvoker:(std::shared_ptr<facebook::react::CallInvoker>)jsInvoker {
        return nullptr;
    }
    ```
1. Implement the `(id<RCTTurboModule>)getModuleInstanceFromClass:(Class)moduleClass` method:
    ```c++
    - (id<RCTTurboModule>)getModuleInstanceFromClass:(Class)moduleClass
    {
        // Set up the default RCTImageLoader and RCTNetworking modules.
        if (moduleClass == RCTImageLoader.class) {
            return [[moduleClass alloc] initWithRedirectDelegate:nil
                loadersProvider:^NSArray<id<RCTImageURLLoader>> *(RCTModuleRegistry * moduleRegistry) {
                return @ [[RCTLocalAssetImageLoader new]];
                }
                decodersProvider:^NSArray<id<RCTImageDataDecoder>> *(RCTModuleRegistry * moduleRegistry) {
                return @ [[RCTGIFImageDecoder new]];
                }];
        } else if (moduleClass == RCTNetworking.class) {
            return [[moduleClass alloc]
                initWithHandlersProvider:^NSArray<id<RCTURLRequestHandler>> *(
                    RCTModuleRegistry *moduleRegistry) {
                return @[
                    [RCTHTTPRequestHandler new],
                    [RCTDataRequestHandler new],
                    [RCTFileRequestHandler new],
                ];
                }];
        }
        // No custom initializer here.
        return [moduleClass new];
    }
    ```
1. Run `npx react-native run-ios`

### <a name="ios-tm-js-bindings" />[[TurboModule Setup - iOS] Install TurboModuleManager JavaScript Bindings]()

1. Open the `AwesomeApp/ios/AwesomeApp/AppDelegate.mm`
1. Update the `- (std::unique_ptr<facebook::react::JSExecutorFactory>)jsExecutorFactoryForBridge:(RCTBridge *)` method:
    ```c++
    - (std::unique_ptr<facebook::react::JSExecutorFactory>)jsExecutorFactoryForBridge:(RCTBridge *)bridge
    {
    // Add these lines to create a TurboModuleManager
    if (RCTTurboModuleEnabled()) {
        _turboModuleManager =
            [[RCTTurboModuleManager alloc] initWithBridge:bridge
                                                delegate:self
                                                jsInvoker:bridge.jsCallInvoker];

        // Necessary to allow NativeModules to lookup TurboModules
        [bridge setRCTTurboModuleRegistry:_turboModuleManager];

        if (!RCTTurboModuleEagerInitEnabled()) {
        /**
        * Instantiating DevMenu has the side-effect of registering
        * shortcuts for CMD + d, CMD + i,  and CMD + n via RCTDevMenu.
        * Therefore, when TurboModules are enabled, we must manually create this
        * NativeModule.
        */
        [_turboModuleManager moduleForName:"DevMenu"];
        }
    }

    // Add this line...
    __weak __typeof(self) weakSelf = self;

    // If you want to use the `JSCExecutorFactory`, remember to add the `#import <React/JSCExecutorFactory.h>`
    // import statement on top.
    return std::make_unique<facebook::react::HermesExecutorFactory>(
        facebook::react::RCTJSIExecutorRuntimeInstaller([weakSelf, bridge](facebook::jsi::Runtime &runtime) {
        if (!bridge) {
            return;
        }

        // And add these lines to install the bindings...
        __typeof(self) strongSelf = weakSelf;
        if (strongSelf) {
            facebook::react::RuntimeExecutor syncRuntimeExecutor =
                [&](std::function<void(facebook::jsi::Runtime & runtime_)> &&callback) { callback(runtime); };
            [strongSelf->_turboModuleManager installJSBindingWithRuntimeExecutor:syncRuntimeExecutor];
        }
        }));
    }
    ```
1. Run `npx react-native run-ios`

### <a name="ios-enable-tm" />[[TurboModule Setup - iOS] Enable TurboModule System]()

1. Open the `AwesomeApp/ios/AwesomeApp/AppDelegate.mm`
1. Update the `(BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions` method
    ```diff
        - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
        {
    +       RCTEnableTurboModule(YES);

            RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self
                                                      launchOptions:launchOptions];
    ```
1. Run `npx react-native run-ios`

### <a name="turbomodule-ndk" />[[TurboModule Setup - Android] Enable NDK and the native build]()

1. Open the `AwesomeApp/android/app/build.gradle` file and update it as it follows:
    1. Add the following additional configurations:
        ```diff
            defaultConfig {
                applicationId "com.awesomeapp"
                minSdkVersion rootProject.ext.minSdkVersion
                targetSdkVersion rootProject.ext.targetSdkVersion
                versionCode 1
                versionName "1.0"

        +        externalNativeBuild {
        +            ndkBuild {
        +                arguments "APP_PLATFORM=android-21",
        +                    "APP_STL=c++_shared",
        +                    "NDK_TOOLCHAIN_VERSION=clang",
        +                    "GENERATED_SRC_DIR=$buildDir/generated/source",
        +                    "PROJECT_BUILD_DIR=$buildDir",
        +                    "REACT_ANDROID_DIR=$rootDir/../node_modules/react-native/ReactAndroid",
        +                    "REACT_ANDROID_BUILD_DIR=$rootDir/../node_modules/react-native/ReactAndroid/build",
        +                    "NODE_MODULES_DIR=$rootDir/../node_modules/"
        +                cFlags "-Wall", "-Werror", "-fexceptions", "-frtti", "-DWITH_INSPECTOR=1"
        +                cppFlags "-std=c++17"
        +                // Make sure this target name is the same you specify inside the
        +                // src/main/jni/Android.mk file for the `LOCAL_MODULE` variable.
        +                targets "awesomeapp_appmodules"
        +            }
        +        }
            }

        +    externalNativeBuild {
        +        ndkBuild {
        +            path "$projectDir/src/main/jni/Android.mk"
        +        }
        +    }
        ```
    1. After the `applicationVariants`, before closing the `android` block, add the following code:
        ```diff
                }
            }
        +    def reactAndroidProjectDir = project(':ReactAndroid').projectDir
        +    def packageReactNdkLibs = tasks.register("packageReactNdkLibs", Copy) {
        +        dependsOn(":ReactAndroid:packageReactNdkLibsForBuck")
        +        from("$reactAndroidProjectDir/src/main/jni/prebuilt/lib")
        +        into("$buildDir/react-ndk/exported")
        +    }
        +
        +    afterEvaluate {
        +        preBuild.dependsOn(packageReactNdkLibs)
        +        configureNdkBuildDebug.dependsOn(preBuild)
        +        configureNdkBuildRelease.dependsOn(preBuild)
        +    }
        +
        +    packagingOptions {
        +        pickFirst '**/libhermes.so'
        +        pickFirst '**/libjsc.so'
        +    }
        }
        ```
    1. Finally, in the `dependencies` block, perform this change:
        ````diff
        implementation fileTree(dir: "libs", include: ["*.jar"])
        //noinspection GradleDynamicVersion
        - implementation "com.facebook.react:react-native:+"  // From node_modules
        + implementation project(":ReactAndroid")  // From node_modules
        ```
1. Create an `AwesomeApp/android/app/src/main/jni/Android.mk` file, with the following content:
    ```makefile
    THIS_DIR := $(call my-dir)

    include $(REACT_ANDROID_DIR)/Android-prebuilt.mk

    # If you wish to add a custom TurboModule or Fabric component in your app you
    # will have to include the following autogenerated makefile.
    # include $(GENERATED_SRC_DIR)/codegen/jni/Android.mk

    # Includes the MK file for autolinked libraries
    include $(PROJECT_BUILD_DIR)/generated/rncli/src/main/jni/Android-rncli.mk

    include $(CLEAR_VARS)

    LOCAL_PATH := $(THIS_DIR)

    # You can customize the name of your application .so file here.
    LOCAL_MODULE := awesomeapp_appmodules

    LOCAL_C_INCLUDES := $(LOCAL_PATH) $(PROJECT_BUILD_DIR)/generated/rncli/src/main/jni
    LOCAL_SRC_FILES := $(wildcard $(LOCAL_PATH)/*.cpp) $(wildcard $(PROJECT_BUILD_DIR)/generated/rncli/src/main/jni/*.cpp)
    LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH) $(PROJECT_BUILD_DIR)/generated/rncli/src/main/jni

    # If you wish to add a custom TurboModule or Fabric component in your app you
    # will have to uncomment those lines to include the generated source
    # files from the codegen (placed in $(GENERATED_SRC_DIR)/codegen/jni)
    #
    # LOCAL_C_INCLUDES += $(GENERATED_SRC_DIR)/codegen/jni
    # LOCAL_SRC_FILES += $(wildcard $(GENERATED_SRC_DIR)/codegen/jni/*.cpp)
    # LOCAL_EXPORT_C_INCLUDES += $(GENERATED_SRC_DIR)/codegen/jni

    # Here you should add any native library you wish to depend on.
    LOCAL_SHARED_LIBRARIES := \
    libfabricjni \
    libfbjni \
    libfolly_runtime \
    libglog \
    libjsi \
    libreact_codegen_rncore \
    libreact_debug \
    libreact_nativemodule_core \
    libreact_render_componentregistry \
    libreact_render_core \
    libreact_render_debug \
    libreact_render_graphics \
    librrc_view \
    libruntimeexecutor \
    libturbomodulejsijni \
    libyoga

    # Autolinked libraries
    LOCAL_SHARED_LIBRARIES += $(call import-codegen-modules)

    LOCAL_CFLAGS := -DLOG_TAG=\"ReactNative\" -fexceptions -frtti -std=c++17

    include $(BUILD_SHARED_LIBRARY)
    ```
1. From the `AwesomeApp` folder, run `npx react-native run-android`

**NOTE:** Make sure that the `targets` property in the `externalNativeBuild/ndkBuild` of the `gradle.build` file matches the `LOCAL_MODULE` property of the `Android.mk` file

### <a name="java-tm-delegate" />[[TurboModule Setup - Android] Provide a `ReactPackageTurboModuleManagerDelegate`]()

1. run this command:
    ```
    cp node_modules/react-native/template/android/app/src/main/java/com/helloworld/newarchitecture/modules/MainApplicationTurboModuleManagerDelegate.java android/app/src/main/java/com/awesomeapp/MainApplicationTurboModuleManagerDelegate.java
    ```
    This will copy the `MainApplicationTurboModuleManagerDelegate.java` from the template to the proper folder
1. Open the `android/app/src/main/java/com/awesomeapp/MainApplicationTurboModuleManagerDelegate.java` and update it as follow:
    ```diff
    - package com.helloworld.newarchitecture.modules;
    + package com.awesomeapp;

    import com.facebook.jni.HybridData;
    import com.facebook.react.ReactPackage;

    // ...

    @Override
    protected synchronized void maybeLoadOtherSoLibraries() {
        // Prevents issues with initializer interruptions.
        if (!sIsSoLibraryLoaded) {
    -        SoLoader.loadLibrary("helloworld_appmodules");
    +        SoLoader.loadLibrary("awesomeapp_appmodules");
            sIsSoLibraryLoaded = true;
        }
    }

    ```
    **Note:** Make sure that parameter of the `SoLoader.loadLibrary` function in the `maybeLoadOtherSoLibraries` is the same name used in the `LOCAL_MODULE` property of the `Android.mk` file.
1. `npx react-native run-android`

### <a name="java-tm-adapt-host" /> [[TurboModule Setup - Android] Adapt your ReactNativeHost to use the `ReactPackageTurboModuleManagerDelegate`]()

1. Open the `AwesomeApp/android/app/src/java/main/MainApplication.java` file
1. Add the imports:
    ```java
    import androidx.annotation.NonNull;
    import com.facebook.react.ReactPackageTurboModuleManagerDelegate;
    ```
1. After the `getJSMainModuleName()` method, within the `ReactNativeHost` construction, add the following method:
    ```java
    @NonNull
    @Override
    protected ReactPackageTurboModuleManagerDelegate.Builder getReactPackageTurboModuleManagerDelegateBuilder() {
        return new MainApplicationTurboModuleManagerDelegate.Builder();
    }
    ```
1. `npx react-native run-android`

### <a name="cpp-tm-manager" />[[TurboModule Setup - Android] C++ Provide a native implementation for the methods in your *TurboModuleDelegate class]()

Referring to [this step](https://reactnative.dev/docs/new-architecture-app-modules-android#5-c-provide-a-native-implementation-for-the-methods-in-your-turbomoduledelegate-class), we now have to add a few files in the `AwesomeApp/android/app/src/main/jni` folder:

- `MainApplicationTurboModuleManagerDelegate.h`
- `MainApplicationTurboModuleManagerDelegate.cpp`
- `MainApplicationModuleProvider.h`
- `MainApplicationModuleProvider.cpp`
- `OnLoad.cpp`

1. Run this command from the root of your app.
    ```
    cp node_modules/react-native/template/android/app/src/main/jni/MainApplicationTurboModuleManagerDelegate.h android/app/src/main/jni/MainApplicationTurboModuleManagerDelegate.h
    cp node_modules/react-native/template/android/app/src/main/jni/MainApplicationTurboModuleManagerDelegate.cpp android/app/src/main/jni/MainApplicationTurboModuleManagerDelegate.cpp
    cp node_modules/react-native/template/android/app/src/main/jni/MainApplicationModuleProvider.h android/app/src/main/jni/MainApplicationModuleProvider.h
    cp node_modules/react-native/template/android/app/src/main/jni/MainApplicationModuleProvider.cpp android/app/src/main/jni/MainApplicationModuleProvider.cpp
    cp node_modules/react-native/template/android/app/src/main/jni/OnLoad.cpp android/app/src/main/jni/OnLoad.cpp
    ```
    This command will copy these 5 files from the React Native template into your app.
1. Open the `AwesomeApp/android/app/src/main/jni/MainApplicationTurboModuleManagerDelegate.h` and update it as follows:
    ```diff
    // ...
    class MainApplicationTurboModuleManagerDelegate
        : public jni::HybridClass<
          MainApplicationTurboModuleManagerDelegate,
          TurboModuleManagerDelegate> {
    public:
    // Adapt it to the package you used for your Java class.
    static constexpr auto kJavaDescriptor =
    -    "Lcom/helloworld/newarchitecture/modules/MainApplicationTurboModuleManagerDelegate;";
    +    "Lcom/awesomeapp/MainApplicationTurboModuleManagerDelegate;";
    ```
1. Open the `AwesomeApp/android/app/src/main/jni/OnLoad.cpp` file and update it as follow:
    ```diff
    #include <fbjni/fbjni.h>
    #include "MainApplicationTurboModuleManagerDelegate.h"

    - #include "MainComponentsRegistry.h"
    + // #include "MainComponentsRegistry.h"

    JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *) {
        return facebook::jni::initialize(vm, [] {
            facebook::react::MainApplicationTurboModuleManagerDelegate::
                registerNatives();
    -        facebook::react::MainComponentsRegistry::registerNatives();
    +//        facebook::react::MainComponentsRegistry::registerNatives();
        });
    }

    ```
    Comment out Fabric-specific code, we will need it later.
1. run `npx react-native run-android`

### <a name="enable-tm-android" />[[TurboModule Setup - Android] Enable the useTurboModules flag in your Application onCreate]()

* Open the `AwesomeApp/android/app/src/main/java/com/awesomeapp/MainApplication.java` file
* Add the import for the feature flags
    ```diff
    import com.facebook.react.ReactPackage;
    + import com.facebook.react.config.ReactFeatureFlags;
    import com.facebook.soloader.SoLoader;
    ```
* Modify the `OnCreate` file as it follows:
    ```diff
        @Override
        public void onCreate() {
            super.onCreate();
    +        ReactFeatureFlags.useTurboModules = true;
            SoLoader.init(this, /* native exopackage */ false);
            initializeFlipper(this, getReactNativeHost().getReactInstanceManager());
        }
    ```
* Run `npx react-native run-android`

### <a name="fabric-root-view" />[[Fabric Setup - iOS] Update your root view]()

1. Open the `AwesomeApp/ios/AwesomeApp/AppDelegate.mm` file.
1. Add the following `imports`:
    ```objective-c
    #import <React/RCTFabricSurfaceHostingProxyRootView.h>
    #import <React/RCTSurfacePresenter.h>
    #import <React/RCTSurfacePresenterBridgeAdapter.h>
    #import <react/config/ReactNativeConfig.h>
    ```
1. Add the following properties in the `AppDelegate` interface:
    ```diff
    @interface AppDelegate () <RCTCxxBridgeDelegate,
                           RCTTurboModuleManagerDelegate> {

    +   RCTSurfacePresenterBridgeAdapter *_bridgeAdapter;
    +   std::shared_ptr<const facebook::react::ReactNativeConfig> _reactNativeConfig;
    +   facebook::react::ContextContainer::Shared _contextContainer;
    @end
    ```
1. Update the `rootView` property as it follows:
    ```diff
    RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
    - RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:bridge
                                                   moduleName:@"AwesomeApp"
                                            initialProperties:nil];
    + _contextContainer = std::make_shared<facebook::react::ContextContainer const>();
    + _reactNativeConfig = std::make_shared<facebook::react::EmptyReactNativeConfig const>();

    + _contextContainer->insert("ReactNativeConfig", _reactNativeConfig);

    + _bridgeAdapter = [[RCTSurfacePresenterBridgeAdapter alloc]
            initWithBridge:bridge
          contextContainer:_contextContainer];

    + bridge.surfacePresenter = _bridgeAdapter.surfacePresenter;

    + UIView *rootView = [[RCTFabricSurfaceHostingProxyRootView alloc] initWithBridge:bridge
                                                                           moduleName:@"AwesomeApp"
                                                 initialProperties:@{}];
    ```
1. From `AwesomeApp`, run `npx react-native run-ios`
