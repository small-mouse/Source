# ALL_I_KNOW
source  

小七所理解的android消息机制
==================================================

android的UI是线程不安全的，如果试图在子现场中访问UI,呵呵，你会看到这句很亲切的言语: "Can't create handler inside thread that has not called Looper.prepare()"，解决这个异常很简单，只要在当前的线程中创建Looper或者在主线程（UI线程）创建Handler即可。

好，如果听得懂上面的话，你可以离开此页面了，因为你已经掌握了。啊哈哈，如果你还想听小七吹逼，可以继续看下去。

首先，我们先重现那句情切言语的情景，Handler使用方法相信大家都会，如果有不会的，我也不会讲的，除非你是个妹子，我可以手把手教你的，啊哈哈...(天啊，怎么有这种人)，一般你在程序中开个线程，new一个handler对象，就会报异常了。
```java
  @Override  
  protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.main);  
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                Handler handler2 = new Handler();  
            }  
        }).start();  
    }  
```
我们进去看看Handler()的构造函数的源码吧（如果是cjj一听到要看源码，一定会说：卧槽，看什么源码，那是大神做的事，会用就得了，别瞎折腾了）
```java
    /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
    









