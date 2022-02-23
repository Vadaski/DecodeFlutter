# Android Flutter 原理

# embedding 各类的作用
- **【app】**:旧版嵌入层实现，现已弃用
- **【embedding】**: 新版嵌入层的实现
  - **【android】**：Android 平台相关代码
    - AndroidTouchProcessor：向 Flutter 传递用户触摸的格式化信息
    - ExclusiveAppComponent：用于链接 FlutterEngine 的 Android 组件的接口
    - FlutterActivity：Flutter 页面的容器
    - FlutterActivityAndFragmentDelegate：FlutterActivity 和 FlutterFragment 的公共部分
    - FlutterActivityLaunchConfigs：启动配置文件，包括 Entrypoint、InitialRoute、cached_engine_id 等
    - FlutterEngineConfigurator：Engine 创建之后对其进行配置， 可以在 FragmentActivity 中使用
    - FlutterEngineProvider：为 FlutterActivity 和 FlutterFragment 提供引擎
    - FlutterFragment：比 FlutterActivity 更轻量的容器
    - FlutterFragmentActivity：基于  FragmentActivity 的 Flutter Activity，因为有些 Android API 只能用在 FragmentActivity 上。如果你没有这块的需求，请直接使用 FlutterActivity，因为这是最标准的实现。
    - FlutterImageView：通过 ImageReader 向 Canvas 绘制 Flutter UI。通常是在需要绘制 Flutter UI 还要渲染一个 PlatformView 的时候使用这个 View。
    - FlutterPlayStoreSplitApplication：用于实现 PlayStore 的延迟下发组件能力的扩展类。
    - FlutterSplashView：在 FlutterView 渲染首帧之前展示一个开屏页。
    - FlutterSurfaceView：在 Surface 上绘制 Flutter UI。
    - FlutterTextureView：在 SurfaceTexture 上绘制 Flutter UI。
    - FlutterView：由 FlutterEngine 绘制 Flutter UI 到 FlutterView 上。
    - KeyboardManager：向 Flutter 传递键盘事件
    - KeyChannelResponder：提供传递键盘事件的 Channel
    - MotionEventTracker：追踪 FlutterView 收到的 Motion 事件。
    - RenderMode：绘制模式，有三种：surface、texture、image
    - TransparencyMode：用于是否开启透明的 FlutterView，只能在 FlutterSurfaceView 上使用。
  - **【engine】**：Dart 虚拟机以及引擎相关实现
- **【plugin】**：插件相关实现
- **【util】**：工具辅助类
- **【view】**：无障碍相关的链接层
- BuildConfig：构建模式的配置（DEBUG | PROFILE | RELEASE | JIT_RELEASE）
- FlutterInjector：轻量的 Flutter 依赖注入类
- Log：用于打印 Flutter 相关日志



## FlutterActivity

### 职责

- 展示一个安卓的启动页
- 展示 Flutter 的闪屏页
- 配置状态栏外观
- 决定 Dart 执行 app bundle 的路径以及入口（entrypoint）
- 决定 Flutter 的起始路由（initial route）
- 可以选择以透明形式呈现
- 为子类提供钩子方法，以提供和配置 Flutter 引擎
- 保存并储存当前的状态（shouldRestoreAndSaveState）

### 概念解释

> #### Dart 执行入口 （entrypoint）
>
> 这个 Activity 默认情况下会选择 `main()` 作为执行入口，要改成其他方法的话，你需要在继承至 FlutterActivity 的子类中重写 `getDartEntrypointFunctionName` 方法。 
>
> ⚠️：对于非 `main()` 的 entrypoint 你需要在方法上添加 `@pragma('vm:entry-point')` 注解以避免该方法被 tree shaking 掉。
>
> > tree shaking：一种用于在编译期移除无用代码的技术，主要目的是缩小包体积。
>
> #### 起始路由（initial route）
> 
> 这个 Activity 会去找 Flutter 中的 `/` 作为起始路由。你也可以使用 `FlutterActivityLaunchConfigs` 中 的 `EXTRA_INITIAL_ROUTE` 配置初始路由。
>
> #### 如何改写 app bundle 的路径、Dart 执行入口 （entrypoint）、以及起始路由（initial route）
>
> 可以通过继承至 `FlutterActivity` 然后重写下面对应的方法。
>
> - `getAppBundlePath()`
> - `getDartEntrypointFunctionName()`
> - `getInitialRoute()`

### 引擎

#### 引擎缓存
你可以使用 `withCachedEngine("String")` 来获得一个缓存的引擎，而不是每次都创建一个新的。`io.flutter.embedding.engine.FlutterEngineCache` 用来保存缓存的引擎。在使用这个 `withCachedEngine(String)` 构造器之前，你必须创建一个 `Engine`，并把它放进 `FlutterEngineCache` 中，否则就会抛出 `IllegalStateException`。

>  ⚠️： 当你使用缓存的引擎的时候，这个引擎应该已经执行了 Dart 代码了，这意味着 **Dart 执行入口 （entrypoint）**、以及**起始路由（initial route）** 都已经被决定了。因此 `CachedEngineIntentBuilder` 并不提供这些入参。

通常来说，我们推荐使用缓存的引擎以避免创建一个新的引擎，但这也存在两种例外情况：

- 当 FlutterActivity 作为应用的首个被展示的 Activity 时，且只有一个 Flutter Activity 时。（因为本身打开的时候就要初始化引擎，所以提前的引擎预热无法带来任何额外的收益。）
- 当你不确定是否需要显示 Flutter 页面时。（如果 Flutter 页面都不用展示了，那提前初始化引擎只会带来内存和性能负担，负向收益比较大。）

#### 如何使用 Cached Engine

``` dart
 // 提前创建一个 FlutterEngine
 FlutterEngine flutterEngine = new FlutterEngine(context);
 flutterEngine.getDartExecutor().executeDartEntrypoint(DartEntrypoint.createDefault());
 
 // 将预热的 FlutterEngine 放进 FlutterEngineCache.
 FlutterEngineCache.getInstance().put("my_engine", flutterEngine);
```

#### 如何使用自定义的引擎
子类重写 `provideFlutterEngine` 提供一个引擎
``` java
  @Override
  public FlutterEngine provideFlutterEngine(@NonNull Context context) {
    // No-op. Hook for subclasses.
    return null;
  }

```

### 使用 FlutterFragment

如果有一些场景无法使用 Activity，你可以考虑使用 `FlutterFragment`。 

如果有一些场景只能使用 `View`,你可以考虑使用 `FlutterView`。

### 启动页 & 闪屏页

//TODO：翻译此段

```java
* <p>{@code FlutterActivity} supports the display of an Android "launch screen" as well as a
* Flutter-specific "splash screen". The launch screen is displayed while the Android application
* loads. It is only applicable if {@code FlutterActivity} is the first {@code Activity} displayed
* upon loading the app. After the launch screen passes, a splash screen is optionally displayed.
* The splash screen is displayed for as long as it takes Flutter to initialize and render its first
* frame.
*
* <p>Use Android themes to display a launch screen. Create two themes: a launch theme and a normal
* theme. In the launch theme, set {@code windowBackground} to the desired {@code Drawable} for the
* launch screen. In the normal theme, set {@code windowBackground} to any desired background color
* that should normally appear behind your Flutter content. In most cases this background color will
* never be seen, but for possible transition edge cases it is a good idea to explicitly replace the
* launch screen window background with a neutral color.
*
* <p>Do not change aspects of system chrome between a launch theme and normal theme. Either define
* both themes to be fullscreen or not, and define both themes to display the same status bar and
* navigation bar settings. To adjust system chrome once the Flutter app renders, use platform
* channels to instruct Android to do so at the appropriate time. This will avoid any jarring visual
* changes during app startup.
*
* <p>In the AndroidManifest.xml, set the theme of {@code FlutterActivity} to the defined launch
* theme. In the metadata section for {@code FlutterActivity}, defined the following reference to
* your normal theme:
*
* <p>{@code <meta-data android:name="io.flutter.embedding.android.NormalTheme"
* android:resource="@style/YourNormalTheme" /> }
*
* <p>With themes defined, and AndroidManifest.xml updated, Flutter displays the specified launch
* screen until the Android application is initialized.
*
* <p>Flutter also requires initialization time. To specify a splash screen for Flutter
* initialization, subclass {@code FlutterActivity} and override {@link #provideSplashScreen()}. See
* {@link SplashScreen} for details on implementing a splash screen.
*
* <p>Flutter ships with a splash screen that automatically displays the exact same {@code
* windowBackground} as the launch theme discussed previously. To use that splash screen, include
* the following metadata in AndroidManifest.xml for this {@code FlutterActivity}:
*
* <p>{@code <meta-data android:name="io.flutter.embedding.android.SplashScreenDrawable"
* android:resource="@drawable/launch_background" /> }
*
* <p><strong>Alternative Activity</strong> {@link FlutterFragmentActivity} is also available, which
* is similar to {@code FlutterActivity} but it extends {@code FragmentActivity}. You should use
* {@code FlutterActivity}, if possible, but if you need a {@code FragmentActivity} then you should
* use {@link FlutterFragmentActivity}.
*/
```

------------------------------------------------------------------------------------------------
### 创建 Activity 生命周期变化

```
I/FlutterActivity(20602): onCreate
I/FlutterActivity(20602): provideFlutterEngine
I/FlutterActivity(20602): getFlutterShellArgs
I/FlutterActivity(20602): shouldRestoreAndSaveState
I/FlutterActivity(20602): getCachedEngineId
I/FlutterActivity(20602): shouldAttachEngineToActivity
I/FlutterActivity(20602): getLifecycle
I/FlutterActivity(20602): providePlatformPlugin
I/FlutterActivity(20602): configureFlutterEngine
I/FlutterActivity(20602): shouldRestoreAndSaveState
I/FlutterActivity(20602): shouldAttachEngineToActivity
I/FlutterActivity(20602): onFlutterSurfaceViewCreated
I/FlutterActivity(20602): provideSplashScreen
I/FlutterActivity(20602): getRenderMode
I/FlutterActivity(20602): getBackgroundMode
I/FlutterActivity(20602): onStart
I/FlutterActivity(20602): getInitialRoute
I/FlutterActivity(20602): shouldHandleDeeplinking
I/FlutterActivity(20602): getDartEntrypointFunctionName
I/FlutterActivity(20602): getAppBundlePath
I/FlutterActivity(20602): getDartEntrypointFunctionName
I/FlutterActivity(20602): onResume
I/FlutterActivity(20602): onPostResume
```

### 生命周期时序 1：构造器

- ` FlutterActivity()` ：创建 `FlutterActivity` 初始化 `LifecycleRegistry`

### 生命周期时序 2：onCreate

- `onCreate(@Nullable Bundle savedInstanceState)`
  
  - 切换到常规主题 switchLaunchThemeForNormalTheme
  
  - 调用 `Activity onCreate`
  
  - ⚠️ 创建 **FlutterActivityAndFragmentDelegate**：这个类实现了全部 `FlutterActivity` 以及 `FlutterFragment` 的全部公共逻辑
  
  - 调用 delegate 的 onAttach，也就是说 `FlutterActivityAndFragmentDelegate` 的 onAttach 后于 `FlutterActivity` 的 onCreate 调用。
  
  - 调用 delegate 的 onRestoreInstanceState，这一步的作用是让 FlutterFramework 和 Plugin 重载状态，最终是调用了 `flutterEngine.getRestorationChannel().setRestorationData(frameworkState);` 和 `flutterEngine.getActivityControlSurface().onRestoreInstanceState(pluginState);`。
  
    > 这里发现可以通过  `FlutterActivityAndFragmentDelegate` 的 host 判断当前是否应该让 Engine Attach 到 Activity 上。如果是使用的 `FlutterActivity`，直接返回 true。如果是 `Flutter Fragment` 则是根据 `ARG_SHOULD_ATTACH_ENGINE_TO_ACTIVITY` 这个 flag 里面的值来进行判断。
  
  - ⚠️ 向 LifecycleRegistry 汇报生命周期改动。
  
  - 配置当前的 Window 是否是透明的，要改变这一点可以在打开这个 Activity 的时候，在 Intent 的 extra 里面放一个 FlutterActivityLaunchConfigs 的 EXTRA_BACKGROUND_MODE flag，来改变不透明度。
  
  - ⚠️ 通过 FlutterActivityAndFragmentDelegate 的 onCreateView 创建一个 FlutterView。这个方法会创建 View 并给它添加一个 `FlutterUiDisplayListener`，然后将 `FlutterEngine` attach 到 FlutterView 上。
  
    > ⚠️ Tips：
    >
    > 1. 这个 FlutterUiDisplayListener 似乎可以监听 Flutter 在 Android View 层级开始和停止渲染的时机。
    >
    > 2. FlutterActivity 的渲染模式是 surface，这个需要在 delegate.onCreateView 中进行指定。
    > 3. ⚠️ 这里最终会调用 `FlutterView` 的 `attachToFlutterEngine` 。
    > 4. 可以通过 isAttachedToFlutterEngine 判断当前 `FlutterEngine` 是否 attach 到 `FlutterView`。
    > 5. ⚠️ 在这个生命周期绑定 Renderer，初始化 textInputPlugin、KeyboardManager、AndroidTouchProcessor、AccessibilityBridge 等等各种基础能力 channel 的初始化。
    > 6. ⚠️ 可以通过 `FlutterView` 的 `FlutterEngineAttachmentListener` 获得引擎 Attached 到 FlutterView 的回调。
    > 7. ⚠️ 可以通过 `FlutterView` 的 `FlutterUiDisplayListener` 来获取 Flutter开始渲染和即将脱离渲染的时机。

> ### ⚠️ 如何获取 FlutterActivity 生命周期状态
>
> FlutterActivity 中有一个 LifecycleRegistry，在 Activity 构造器中就会被创建出来，Activity 全部生命周期（onCreate、onStart、onResume、onPause、onStop、onDestroy） 都会调用 `lifecycle.handleLifecycleEvent`。
>
> 你可以通过 `getLifecycle().getCurrentState();` 主动获取当前 `FlutteActivity` 的生命周期状态，也可以通过添加监听器的方式被动监听 `FlutterActivity` 的生命周期。
>
> ```java
> /// 当前代码请写在 FlutterActivity 的子类中
> getLifecycle().addObserver(
>   new LifecycleEventObserver() {
>       @Override
>      	public void onStateChanged(
>         @NonNull @org.jetbrains.annotations.NotNull LifecycleOwner source, 
>         @NonNull @org.jetbrains.annotations.NotNull Lifecycle.Event event) {
>            /// 在这里针对生命周期进行处理     
>         }
>  });
> ```


### ⚠️ 生命周期时序 2.1：FlutterActivityAndFragmentDelegate 的 onAttach

**FlutterActivityAndFragmentDelegate**: 该类实现了 `FlutterActivity` 与 `FlutterFragment` 之间的共有逻辑。

- onAttach(@NonNull Context context)
  - 确保自己没有被 release 掉：`ensureAlive()`
  - 如果 FlutterEngine 为空，则初始化一个。`FlutterActivityAndFragmentDelegate.setupFlutterEngine()`。
  - 让 engine attach 到 FlutterActivity 上。（注意这里是 attach Activity 比 attach FlutterView 要更早）
    - 如果 FlutterEngineConnectionRegistry 的 exclusiveActivity 不为空，则先让 engine 从这个 Activity 上 detach 下来。【`FlutterEngineConnectionRegistry` 被 FlutterEngine 持有，它的作用是管理和 Android App Components 与 Flutter Plugin 之间的关系】
  - ⚠️调用 `FlutterActivity` 的 `configureFlutterEngine` 注册 Plugin 的各种 Channel。
    - 通过  `GeneratedPluginRegister.registerGeneratedPlugins(flutterEngine);` 注册 Plugin 到 FlutterEngine。

> 发现的信息：
>
> 1.可以通过 delegate 的 ensureAlive 判断是否被 release 掉。

> #### ⚠️ 初始化 Flutter 引擎
>
> `FlutterActivityAndFragmentDelegate.setupFlutterEngine()`
>
> 1. 首先判断是否用 Cache 的引擎。
> 2. 然后通过 `Host#provideFlutterEngine(Context)` 尝试从 host 中去拿引擎。
> 3. 如果前两种都失败了，就创建一个新的引擎。

### 生命周期时序 3：onStart

-  `onStart()`

  - 更新生命周期 ` lifecycle.handleLifecycleEvent` 

  - 如果 Flutter 引擎还 `Attach` 着，`FlutterActivityAndFragmentDelegate` 执行 `onStart`。

    - 确保 delegate 没有被释放掉
    - 执行首次 FlutterViewRun 的初始化操作
    - 开始首次在 FlutterView 上运行 Dart
    - ⚠️ 初始化 initialRoute
      1. 先尝试从 host 去拿 initialRoute
      2. 尝试从 Intent 中去拿 initialRoute
      3. 默认是 `DEFAULT_INITIAL_ROUTE`,"/"
      4. 从引擎获取 `NavigationChannel`，设置 initialRoute
    - ⚠️ 获取 appBundlePathOverride，默认就是 `flutter_assets`，这里应该可以重写【不知道 hotpatch 是不是这里 hook 改的】
    - ⚠️ 配置 Dart entrypoint 并执行它，需要传入 appBundlePath （default：flutter_assets），以及 Dart 执行入口 `getDartEntrypointFunctionName` （default：main）

    > 你可以 getMetaData 通过配置 DART_ENTRYPOINT_META_DATA_KEY 这个 key 改变 entrypoint 名称。

### 生命周期时序 4：onResume

- ` onResume()`
  - 更新生命周期 `lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);`
    - 执行 onPostResume，去更新 Android Window 的系统 UI `updateSystemUiOverlays`
  - ⚠️ 检查 delegate 是否 `attached：stillAttachedForEvent`
    - ⚠️ delegate 执行 onResume，向 flutterEngine 通知应用已经进入可视状态 `flutterEngine.getLifecycleChannel().appIsResumed();`

### 生命周期时序 4.1：onFirstFrame **构建首帧(FlutterJNI)**

- 确保运行在主线程上 `ensureRunningOnMainThread`
- 通知所有的 `FlutterUiDisplayListener`，Flutter 开始渲染 UI 了（这个时候还是黑的，没有真正渲染出来）`onFlutterUiDisplayed`

-------------------------------------------------------------------------------------------
### FlutterActivity 退到后台时生命周期变化
```
I/FlutterActivity(20603): onUserLeaveHint
I/FlutterActivity(20603): onPause
I/FlutterActivity(20603): onFlutterUiNoLongerDisplayed
I/FlutterActivity(20603): onTrimMemory
I/FlutterActivity(20603): onStop
I/FlutterActivity(20603): onSaveInstanceState
I/FlutterActivity(20603): shouldRestoreAndSaveState
I/FlutterActivity(20603): getCachedEngineId
I/FlutterActivity(20603): shouldAttachEngineToActivity
```


### 生命周期时序 5：onUserLeaveHint
- 通知 PluginBinding 执行 onUserLeaveHint （用户要离开界面了）

### 生命周期时序 6：onPause -> AppLifecycleState.inactive
- 让 delegate 先去执行 onPause
  - Delegate 去告诉 Flutter App 要变得不活跃了，通过引擎获取 LifecycleChannel 发送 `AppLifecycleState.inactive` 事件。
- 更新生命周期 `lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);`
-------------------------------------------------------------------------------------------

### FlutterActivity 从后台到前台的生命周期变化
```
I/FlutterActivity(20603): onStart
I/FlutterActivity(20603): getCachedEngineId
I/FlutterActivity(20603): onNewIntent
I/FlutterActivity(20603): shouldHandleDeeplinking
I/FlutterActivity(20603): getMetaData
I/FlutterActivity(20603): onResume
I/FlutterActivity(20603): onPostResume
I/FlutterActivity(20603): onFlutterUiDisplayed
```

### FlutterActivity 销毁生命周期变化
```
I/FlutterActivity(20603): onPause
I/FlutterActivity(20603): onFlutterUiNoLongerDisplayed
I/FlutterActivity(20603): onTrimMemory
I/FlutterActivity(20603): onStop
I/FlutterActivity(20603): onSaveInstanceState
I/FlutterActivity(20603): shouldRestoreAndSaveState
I/FlutterActivity(20603): getCachedEngineId
I/FlutterActivity(20603): shouldAttachEngineToActivity
I/FlutterActivity(20603): onDestroy
I/FlutterActivity(20603): cleanUpFlutterEngine
I/FlutterActivity(20603): shouldAttachEngineToActivity
I/FlutterActivity(20603): getActivity
I/FlutterActivity(20603): getActivity
I/FlutterActivity(20603): shouldDestroyEngineWithHost
I/FlutterActivity(20603): getCachedEngineId
Lost connection to device.
```

## 平台插件实现类 PlatformPlugin

-------------------------------------------------------------------------------------------

##  `FlutterActivity` 与 `FlutterFragment` 之间的共有逻辑 FlutterActivityAndFragmentDelegate

Fragment support library 会让应用程序多出 100k 的二进制大小，纯 Flutter 应用不需要这段二进制。因此，Flutter 必须为 add-to-app 开发者提供一个基于 AOSP 的 Activity，以及一个独立的 FlutterFragment。

如果有一天，在纯 Flutter 应用中包含 Fragment 不再被认为是一个问题的话，这个类应该立刻将逻辑分解到 `FlutterActivity` 和 `FlutterFragment` 里面然后被移除。

>  请谨慎修改该类

任何时候，创建 delegate 的目的是封装另一个对象的内部逻辑，这个 delegate 都是与被代理对象高度耦合的。所以很容易将新的功能添加到 delegate 上，而不会添加到原始对象中。而且也很容易在 delegate 上处理 listeners 和 callbacks，而不是在原始的类里面处理。所以 delegate 很快就会变得复杂，而且难以继续维护。

所以你需要谨慎考虑添加代码到 delegate 还是 FlutterActivity / FlutterFragment。


## Flutter 应用生命周期状态 LifecycleChannel
- **inactive 不活跃（onPause）**：在 FlutterActivity 或者 FlutterFragment onPause 的时机会进入 inactive 状态
- **resumed 恢复（onResume）**：在 FlutterActivity 或者 FlutterFragment onResume 的时机会进入 inactive 状态
- **paused 暂停（onStop）**：仅在 FlutterActivity 的 onStop 下会进入 paused 状态
- **detached 移除（onDestroy / detachFromFlutterEngine）**：在 FlutterActivityAndFragmentDelegate 的 onDetach 的时候会被调用。最终是在 FlutterActivity 或者 FlutterFragment 的 detachFromFlutterEngine 或者 onDestroy 会进入 detached 状态