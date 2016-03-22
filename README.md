# ALL_I_KNOW


#那些年，我们一起点过的赞

手在键盘敲很轻，欲说却不知从何吐槽......咳，刚装逼了，现在进入正题。几乎每个社交App都有点赞功能，但是就国内的app来说，你可能记得点赞的内容而压根忽视了点赞的效果。举个例子，就用户最多的微信、QQ来说，点赞也就是个心形和拇指的放大动画（自己去体验下），这里顺便吐槽下网易的点赞，动画做的不错，虽然我手机小小不流畅，可是不能取消赞是怎么回事？ 也许，现在你觉得无非就是个点赞效果，随便做个点击效果就好了，也许产品设计的人也是这样觉得的，也许用户根本就不在乎。我想说的是，友好的交互，会影响用户的体验，做好细节，会让App更成功。

介绍下我认为点赞做的比较好的App及如何实现。

Periscope一款用户可以向其他人直播视音频的App,点赞效果让人眼前一亮。大概在一年前，程序亦非猿就写了[一步一步教你实现Periscope点赞效果](http://www.jianshu.com/p/03fdcfd3ae9c)，我也在个人项目中用了，感觉良好。

![](http://ww3.sinaimg.cn/mw690/7ef01fcagw1f25od1so1kg205k07ittz.gif)

非猿是这样实现这个效果：
* 1.爱心出现在底部并且水平居中
* 2.爱心的颜色/类型 随机
* 3.爱心进入时候有一个缩放的动画
* 4.缩放完毕后,开始变速向上移动,并且伴随alpha渐变效果
* 5.爱心移动的轨迹光滑,是个曲线

其实，难点就是曲线运动，

```java
   /**
     * 获取中间的两个 点
     *
     * @param scale
     */
    private PointF getPointF(int scale) {
        PointF pointF = new PointF();
        pointF.x = random.nextInt((mWidth - 100));//减去100 是为了控制 x轴活动范围,看效果 随意~~
        //再Y轴上 为了确保第二个点 在第一个点之上,我把Y分成了上下两半 这样动画效果好一些  也可以用其他方法
        pointF.y = random.nextInt((mHeight - 100)) / scale;
        return pointF;
    }
```





























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
                Handler handler = new Handler();  
            }  
        }).start();  
    }  
```
我们进去看看Handler()的构造函数的源码吧（一听到要看源码，一定会说：卧槽，看什么源码，那是大神做的事，会用就得了，别瞎折腾了，你以为小七想吗，不看，装逼的深度就低了一个档次）
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
如果英语好的童鞋可以看看那方法上面的一大推解释，英语不好不要看了，越看越不懂，呵呵，直接看这句就可以了 mLooper =  Looper.myLooper();mLooper在Handler.java中是这样声明的 final Looper mLooper; Looper.myLooper()方法获取了一个Looper对象，如果Looper对象为空，则会抛出那句话了。

接下来我们继续深入看看Looper.myLooper()，咳，一入源码深似海 从此小七是猿人......
```java
  /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
  ```
  
  从源码可以看出sThreadLocal.get()返回一个Looper对象，那sThreadLocal又是什么东西来的，别急，小七我也不知道，我们去看它的声明
  ```java
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
 ```
卧槽，那个写注释的也太好人了 sThreadLocal.get() will return null unless you've called prepare().，Looper的对象为null是因为没有调用
Looper.prepare(),等下我们在瞧瞧注释说的是不是真的。（你丫见过注释是假的吗？）
  
我们继续看看ThreadLocal究竟是什么东西来的（用我烂的英语水平来看就是本地线程，是个线程对吧，啊哈哈）
```java
    /**
 * Implements a thread-local storage, that is, a variable for which each thread
 * has its own value. All threads share the same {@code ThreadLocal} object,
 * but each sees a different value when accessing it, and changes made by one
 * thread do not affect the other threads. The implementation supports
 * {@code null} values.
 *
 * @see java.lang.Thread
 * @author Bob Lee
 */
public class ThreadLocal<T> {
    《自己进去看看》
}

```

这是个泛型类，作者Bob Lee 的注释中说 实现一个线程本地的存储，每个线程中的变量有它自己的值，所有的线程共享相同的ThreadLocal对象，但是
每看到一个不同的值去访问它,更改的线程不会影响其他线程。（对，我翻译的就是这样了，bob lee 如果误解了你的意思，我只能说抱歉，啊哈哈）
如果你看得懂我翻译的，麻烦告诉我下，这是什么鬼？

既然那么抽象，我们就用个例子演示下吧。我们在oncreate中运行下面代码：

```java
   final ThreadLocal<String> threadLocal = new ThreadLocal<>();
        threadLocal.set("cjj");

        new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set("小七");
                Log.i("threadLocal", "value---->" + threadLocal.get());
            }
        }).start();

        Log.i("threadLocal", "value---->" + threadLocal.get());
  ```
  
  控制台打印了
  ```java
12-05 18:11:41.101 4239-4291/com.small7.demo I/threadLocal: value---->小七
12-05 18:11:41.101 4239-4239/com.small7.demo I/threadLocal: value---->cjj
```
这能说明，在不同的线程访问同一个ThreadLocal对象，获取到的值是不一样的，如果你怀疑，可以多开几个线程赋值，看看结果是不是这样...

当我们不给TheadLocal设置值的时候，就返回null(不信，自己去试试)，那我们看看他是怎样赋值的，有get就一定要set吧，没赋值，也就获取不到吧。
然后，我们看看ThreaLocal的set()方法
```java
  /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
  ```
  代码只有这几句而已（呵呵，而已你妹啊），获取了当前的currentThread，传入values();在进去看看values()这个方法看看：
  ```java
     /**
     * Gets Values instance for this thread and variable type.
     */
    Values values(Thread current) {
        return current.localValues;
    }
    ```
    而current.localValues的声明是：
    ```java
     /**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;
  ```
  想要知道他是什么东西，我们只有看看ThreadLocal.Values这个内部类了
  ```java
     /**
     * Per-thread map of ThreadLocal instances to values.
     */
    static class Values {

        /**
         * Size must always be a power of 2.
         */
        private static final int INITIAL_SIZE = 16;
        /**
         * Map entries. Contains alternating keys (ThreadLocal) and values.
         * The length is always a power of 2.
         */
        private Object[] table;
        --------------省略n多代码，自己去看--------------------
        
```
看Values的源码我们知道它的作用是存储ThreadLocal的数据，ThreadLocal中的值就存储在table这个数值中。

我们继续看看ThreaLocal的set()方法的下面几句代码Values values = values(currentThread);这句获取到了Values对象，如果Values对象为空，则
初始化它 if (values == null) {values = initializeValues(currentThread);} 可以进去看看初始化的方法
```java
 /**
     * Creates Values instance for this thread and variable type.
     */
    Values initializeValues(Thread current) {
        return current.localValues = new Values();
    }
```
现在我们来到了最关键的一步：   values.put(this, value); 进去看代码...
```java
         /**
         * Sets entry for given ThreadLocal to given value, creating an
         * entry if necessary.
         */
        void put(ThreadLocal<?> key, Object value) {
            cleanUp();

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }
  ```
  卧槽，我知道你看到现在已经很枯燥无味了，我丫也是，这什么鬼。
  
  前面我说过，ThreadLocal的值就是存储在Values中的table[]中了，所以，理解了这点就好了（卧槽，其实是小七对这算法理解实在浅陋，怕说错了，呵呵）
 
 到此ThreaLocal的set()方法我们已经看完，该看看ThreaLocal的get()方法了
 ```java
   /**
     * Returns the value of this variable for the current thread. If an entry
     * doesn't yet exist for this variable on this thread, this method will
     * create an entry, populating the value with the result of
     * {@link #initialValue()}.
     *
     * @return the current value of the variable for the calling thread.
     */
    @SuppressWarnings("unchecked")
    public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {
                return (T) table[index + 1];
            }
        } else {
            values = initializeValues(currentThread);
        }

        return (T) values.getAfterMiss(this);
    }
```
前两句代码和set()是一样的，然后就是判断Values是否为空，如果是就初始化，上文已经给出它是怎么初始化出一个Values对象的了，如果不为空就从
Values里的table数组获取。

就是这样set()、get()可以做到ThreadLocal可以在多个线程中互不干扰的存储和修改数据，之所以理解ThreadLocal，是为了方便理解Looper的工作原理





  
  







  











