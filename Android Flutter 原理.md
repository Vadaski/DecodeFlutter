# Android Flutter 原理

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

### 使用缓存的引擎

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

### FlutterFragment

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

### 生命周期时序

- ` FlutterActivity()` ：创建 `FlutterActivity` 初始化 `LifecycleRegistry`
- `onCreate(@Nullable Bundle savedInstanceState)`
  - 切换到常规主题 switchLaunchThemeForNormalTheme
  - ⚠️ 创建 **FlutterActivityAndFragmentDelegate**：这个类实现了全部 `FlutterActivity` 以及 `FlutterFragment` 的全部公共逻辑。
  - 调用

## FlutterActivityAndFragmentDelegate

