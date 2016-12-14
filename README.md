**SharedPreferences 源码解析**


SharedPreferences（以下简称SP）的源码是比较简单的，一般一个小时就可以完整的撸完一遍，不信？自己去看看。

SharedPreferences的实例的获取有两种方式，一种是Activity的getPreferences方法，一种是Context的getSharedPreferences方法。而Activity的SharePreference实例获取是对Context的getSharedPreferences再一次封装而已。
//实例activity#getPreferences
```java
public SharedPreferences getPreferences(int mode) {
    return getSharedPreferences(getLocalClassName(), mode);
}
```

那我们直接看Context 的getSharedPreferences(String name, int mode) 方法。Context的具体实现是 ContextImpl 类，不懂的自行Google。方法实现如下：
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
	...省略部分代码...
    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
	//实例化一个ArrayMap成员变量,键是String，值是File
            mSharedPrefsPaths = new ArrayMap<>();
        }
	//获得文件类型对象
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
	//如果file为空，则根据传进来的name参数构建要存储数据的文件名
            file = getSharedPreferencesPath(name);
	//保存到mSharedPrefsPaths
            mSharedPrefsPaths.put(name, file);
        }
    }
    return getSharedPreferences(file, mode);
}
上述代码主要做了 给 ContextImpl 加锁，保证同步操作，声明一个key是String类型，value是File类型的ArrayMap成员变量 mSharedPrefsPaths,保存getShraredPreferencesPath方法生成的file,我们看看getShraredPreferencesPath的实现：
@Override
public File getSharedPreferencesPath(String name) {
    return makeFilename(getPreferencesDir(), name + ".xml");
}

private File getPreferencesDir() {
    synchronized (mSync) {
        if (mPreferencesDir == null) {
            mPreferencesDir = new File(getDataDir(), "shared_prefs");
        }
        return ensurePrivateDirExists(mPreferencesDir);
    }
}
通过上述代码，你可以知道为什么SP保存的数据，其原理是用xml文件存放数据，文件存放在/data/data/<package name>/shared_prefs路径下。接下来看看SP真正获取的实现
getSharedPreferences(File file, int mode)方法
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
	//检查
    checkMode(mode);
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}


