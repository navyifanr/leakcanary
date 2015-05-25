LeakCanary
---------------
一个Android和Java的内存泄露检测库

“A small leak will sink a great ship.” - Benjamin Franklin
小漏不补沉大船。——本杰明 富兰克林


![](http://7xigjz.com1.z0.glb.clouddn.com/150509screenshot.png)

<!--more-->

##开始

在项目的build.gradle文件添加：
```
 dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
 }
 ```
在Application类添加：
```
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```
当在你的debug构建过程中出现内存泄露时，LeakCanary将会自动展示一个通知栏。

##为什么我应该使用LeakCanary？

问得好！我们正好写了个[博客](https://corner.squareup.com/2015/05/leak-canary.html)回答这个问题。

##那怎么使用它呢？

使用一个RefWatcher观察引用什么时候应该被GC：

```
RefWatcher refWatcher = {...};

// We expect schrodingerCat to be gone soon (or not), let's watch it.
refWatcher.watch(schrodingerCat);
```
LeakCanary.install() 返回一个先前配置的RefWatcher，它也安装一个ActivityRefWatcher以便在Activity.onDestroy()被调用后自动检测Activity是否出现泄露。

```
public class ExampleApplication extends Application {

  public static RefWatcher getRefWatcher(Context context) {
    ExampleApplication application = (ExampleApplication) context.getApplicationContext();
    return application.refWatcher;
  }

  private RefWatcher refWatcher;

  @Override public void onCreate() {
    super.onCreate();
    refWatcher = LeakCanary.install(this);
  }
}
```

你可以使用RefWatcher观察Fragment的内存泄露
```
public abstract class BaseFragment extends Fragment {

  @Override public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
  }
}
```
##实现原理

1.RefWatcher.watch()创建一个[KeyedWeakReference](https://github.com/square/leakcanary/blob/master/library/leakcanary-watcher/src/main/java/com/squareup/leakcanary/KeyedWeakReference.java)去检测对象；
2.接着，在后台线程，它将会检查是否有引用在不是GC触发的情况下需要被清除的；
3.如果引用引用仍然没有被清除，将会转储堆到.hprof文件到系统文件中(it them dumps the heap into a .hprof file stored on the app file system.)
4.HeapAnalyzerService是在一个分离的进程中开始的，HeapAnalyzer通过使用[HAHA](https://github.com/square/haha)解析heap dump;
5.由于一个特殊的引用key和定位的泄露引用，HeapAnalyzer可以在heap dump中找到KeyedWeakReference；
6.如果有一个泄露，HeapAnalyzer计算到GC Roots的最短的强引用路径，然后创建造成泄露的引用链；
7.结果在app的进程中传回到DisplayLeakService，并展示泄露的通知消息；

##怎样拷贝leak trace?



你可以在Logcat上看leak trace:
```
In com.example.leakcanary:1.0:1 com.example.leakcanary.MainActivity has leaked:
* GC ROOT thread java.lang.Thread.<Java Local> (named 'AsyncTask #1')
* references com.example.leakcanary.MainActivity$3.this$0 (anonymous class extends android.os.AsyncTask)
* leaks com.example.leakcanary.MainActivity instance

* Reference Key: e71f3bf5-d786-4145-8539-584afaecad1d
* Device: Genymotion generic Google Nexus 6 - 5.1.0 - API 22 - 1440x2560 vbox86p
* Android Version: 5.1 API: 22
* Durations: watch=5086ms, gc=110ms, heap dump=435ms, analysis=2086ms
```

你也可以分享leak trace和heap dump文件通过action bar的菜单。


##泄露可能是由Android SDK造成的

随着时间过去越来越多熟知的内存泄露问题被制造商在android开源项目中修复。当这样一个泄露发生时，你能作为一个应用程序开发员来修复它。出于这个原因，LeakCanary有一个内置Android泄露的列表[AndroidExcludedRefs.java](https://github.com/square/leakcanary/blob/master/library/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidExcludedRefs.java)来监测它，如果你找到一个新的泄露，请用leaktrace创建一个[issue](https://github.com/square/leakcanary/issues/new),标明设备和Android版本。如果你提供一个heap dump的文件链接就更好了。

这是对于新发布的Android版本来说是特别重要的。你有机会更早地帮助检测新的内存泄露，这有益于整个Android社区。
开发版快照可以通过[Sonatype's snapshots repository](https://oss.sonatype.org/content/repositories/snapshots/)找到。

##Beyond the leak trace


有时leak trace不够清晰，你需要使用[MAT](http://eclipse.org/mat/)和[YourKit](https://www.yourkit.com/)深入研究heap dump。这里教你怎样在head dump找到泄露的实例：

1.找出包com.squareup.leakcanary.KeyedWeakReference下所有实例;
2.对于每个实例，考虑它的key域;
3.找到 KeyedWeakReference 有一个key域等于被LeakCanary报出的引用的key;
4.KeyedWeakReference的referent域是程序中内存泄露的对象；
5.从那时起，问题就转到你的手上了。一个好的开始是找到最短的GC roots的路径(排除弱引用)

##自定义

###图标和标签

DisplayLeakActivity自带一个默认的icon和label，可以通过提供的R.drawable.__leak_canary_icon和R.string.__leak_canary_display_activity_label来修改：
```
res/
  drawable-hdpi/
    __leak_canary_icon.png
  drawable-mdpi/
    __leak_canary_icon.png
  drawable-xhdpi/
    __leak_canary_icon.png
  drawable-xxhdpi/
    __leak_canary_icon.png
  drawable-xxxhdpi/
    __leak_canary_icon.png
```
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <string name="__leak_canary_display_activity_label">MyLeaks</string>
</resources>
```

###储存leak traces
DisplayLeakActivity可以在你的app目录保存7个heap dumps和leak traces，你可以在app中通过提供R.integer.__leak_canary_max_stored_leaks的值改变这个数量：
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <integer name="__leak_canary_max_stored_leaks">20</integer>
</resources>
```

###上传到服务器
你可以改变默认的行为去上传leak trace并heap dump到你选择的服务器。
创建你自己的AbstractAnalysisResultService，最容易的方式是在你debug的源码中继承DefaultAnalysisResultService：

```
public class LeakUploadService extends DefaultAnalysisResultService {
  @Override protected void afterDefaultHandling(HeapDump heapDump, AnalysisResult result, String leakInfo) {
    if (!result.leakFound || result.excludedLeak) {
      return;
    }
    myServer.uploadLeakBlocking(heapDump.heapDumpFile, leakInfo);
  }
}
```
确定在你正式发布的Application类中使RefWatcher失效：
```
public class ExampleApplication extends Application {

  public static RefWatcher getRefWatcher(Context context) {
    ExampleApplication application = (ExampleApplication) context.getApplicationContext();
    return application.refWatcher;
  }

  private RefWatcher refWatcher;

  @Override public void onCreate() {
    super.onCreate();
    refWatcher = installLeakCanary();
  }

  protected RefWatcher installLeakCanary() {
    return RefWatcher.DISABLED;
  }
}
```

在你的debug的Application类创建一个定制的RefWatcher：
```
public class DebugExampleApplication extends ExampleApplication {
  protected RefWatcher installLeakCanary() {
    return LeakCanary.install(app, LeakUploadService.class);
  }
}
```
不要忘记了在你debug的manifest中注册service:
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    >
  <application android:name="com.example.DebugExampleApplication">
    <service android:name="com.example.LeakUploadService" />
  </application>
</manifest>
```

![](http://7xigjz.com1.z0.glb.clouddn.com/150509icon_512.png)

译者：[cfanr](https://github.com/navyifanr/)
