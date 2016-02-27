# ALL_I_KNOW
source  

2月25日早上，Android官网更新了Support Lirary 23.2版本，其中Design Support Library库新加一个新的东西：Bottom Sheets。然后，第一时间写了篇Teach you how to use Design Support Library: Bottom Sheets，只是简单的讲了它的使用和使用的一些规范。

这篇文章我带大家撸撸BottomSheetBehavior的源码，能力有限，写的不好的地方，请尽力吐槽。好了，不说废话，直接主题

我们先简单的看下用法
```java
        // The View with the BottomSheetBehavior
        View bottomSheet = coordinatorLayout.findViewById(R.id.bottom_sheet);
        final BottomSheetBehavior behavior = BottomSheetBehavior.from(bottomSheet);
        behavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {
            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                //这里是bottomSheet 状态的改变回调
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                //这里是拖拽中的回调，根据slideOffset可以做一些动画
            }
        });
```
BottomSheetBehavior的定义如下

```java
    public class BottomSheetBehavior<V extends View> extends CoordinatorLayout.Behavior<V> 
```
    
继承自CoordinatorLayout.Behavior，BottomSheetBehavior.from(V view)方法获得了BootomSheetBehavior的实例，我们进去看看它怎么实现的。

```java
    public static <V extends View> BottomSheetBehavior<V> from(V view) {
        ViewGroup.LayoutParams params = view.getLayoutParams();
        if (!(params instanceof CoordinatorLayout.LayoutParams)) {
            throw new IllegalArgumentException("The view is not a child of CoordinatorLayout");
        }
        CoordinatorLayout.Behavior behavior = ((CoordinatorLayout.LayoutParams) params)
                .getBehavior();
        if (!(behavior instanceof BottomSheetBehavior)) {
            throw new IllegalArgumentException(
                    "The view is not associated with BottomSheetBehavior");
        }
        return (BottomSheetBehavior<V>) behavior;
    }
```
源码中看出根据传入的参数view的LayoutParams是不是 CoordinatorLayout.LayoutParams，若不是，将抛出"The view is not a child of CoordinatorLayout"的异常，通过 ((CoordinatorLayout.LayoutParams) params).getBehavior()获得一个behavior并判断是不是BottomSheetBehavior，若不是，就抛出异常"The view is not associated with BottomSheetBehavior",都符合就返回了BottomSheetBehavior的实例。这里我们可以知道behavior保存在 CoordinatorLayout.LayoutParams里，那它是
怎么保存的呢，怀着好奇心，我们去看看CoordinatorLayout.LayoutParams中的源码，在LayoutParams的构造函数中，有这么一句：

```java
            if (mBehaviorResolved) {
                mBehavior = parseBehavior(context, attrs, a.getString(
                        R.styleable.CoordinatorLayout_LayoutParams_layout_behavior));
            }
```
顺藤摸瓜，我们在跟进去看看parseBehavior做了什么

```java

     static final Class<?>[] CONSTRUCTOR_PARAMS = new Class<?>[] {
        Context.class,
        AttributeSet.class
    };

    static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
       /*
        *省略部分代码
        */
        try {
           /*
            *省略部分代码
            */
            Constructor<Behavior> c = constructors.get(fullName);
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
                        context.getClassLoader());
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```
这里做的事情很简单，就是在实例化CoordinatorLayout.LayoutParams时反射生成Behavior实例，这就是为什么自定义behavior需要重写如下的构造函数
```java
    public class CjjBehavior extends CoordinatorLayout.Behavior{
        public CjjBehavior(Context context, AttributeSet attrs) {
            super(context, attrs);
        }
    }
```
不然就会看到"Could not inflate Behavior subclass ..."异常 。

目前为止，我们只是了解了CoordinatorLayout.Behavior相关的东西，还是不知道BottomSheetBehavior实现的原理，别急，这就和你说说。

###view布局

CoordinatorLayout 在 onLayout 的时候会调用BottomSheetBehavior.onLayoutChild方法
```java
    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection) {
        // First let the parent lay it out
        if (mState != STATE_DRAGGING && mState != STATE_SETTLING) {
            parent.onLayoutChild(child, layoutDirection);
        }
        // Offset the bottom sheet
        mParentHeight = parent.getHeight();
        mMinOffset = Math.max(0, mParentHeight - child.getHeight());
        mMaxOffset = mParentHeight - mPeekHeight;
        if (mState == STATE_EXPANDED) {
            ViewCompat.offsetTopAndBottom(child, mMinOffset);
        } else if (mHideable && mState == STATE_HIDDEN) {
            ViewCompat.offsetTopAndBottom(child, mParentHeight);
        } else if (mState == STATE_COLLAPSED) {
            ViewCompat.offsetTopAndBottom(child, mMaxOffset);
        }
        if (mViewDragHelper == null) {
            mViewDragHelper = ViewDragHelper.create(parent, mDragCallback);
        }
        mViewRef = new WeakReference<>(child);
        mNestedScrollingChildRef = new WeakReference<>(findScrollingChild(child));
        return true;
    }
```
这里主要做了 调用父类 对 View 进行布局,根据 mPeekHeight 和 mState 对 View 位置的进行偏移,偏移到合适的位置,因为mState 默认为STATE_COLLAPSED,偏移量为mParentHeight - mPeekHeight, 调用了findScrollingChild () 寻找第一个 NestedScrollingChild 组件。
并且初始化了ViewDragHelper 类.负责 View 的滑动。

吐槽下BottomSheetBehavior的一个坑，mPeekHeight的默认值是0，或者我们在代码中设置mPeekHeight的高度为0时，mMaxOffset=mParentHeight，  ViewCompat.offsetTopAndBottom(child, mMaxOffset)刚好看不到BottomSheetBehavior 视图，自己测试了下，
在android5.0的机子底部的BottomSheetBehavior 视图滑动不出来，4.0的又是正常的，你可以亲自试试，体会下......

我的好基友dim给出了解决方案[Android support 23.2 使用BottomSheetBehavior 的坑](http://www.jianshu.com/p/21bb14e3be94)

在说说一个小技巧，Android官网中有这样一句话：[Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android](http://developer.android.com/intl/zh-cn/training/articles/memory.html),就是说枚举比静态常量更加耗费内存，我们应该避免使用，然后我看BottomSheetBehavior源码中 mState 是这样定义的：
```java
    public static final int STATE_DRAGGING = 1;
    public static final int STATE_SETTLING = 2;
    public static final int STATE_EXPANDED = 3;
    public static final int STATE_COLLAPSED = 4;
    public static final int STATE_HIDDEN = 5;

    @IntDef({STATE_EXPANDED, STATE_COLLAPSED, STATE_DRAGGING, STATE_SETTLING, STATE_HIDDEN})
    @Retention(RetentionPolicy.SOURCE)
    public @interface State {}
    
    @State
    private int mState = STATE_COLLAPSED;

```
弥补了Android不建议使用枚举的缺陷。

###事件拦截
在CoordinatorLayout的onInterceptTouchEvent()方法和onTouchEvent()方法中会回调给BottomSheetBehavior的onInterceptTouchEvent、onTouchEvent
```java
if (!intercepted && b != null) {
    switch (type) {
        case TYPE_ON_INTERCEPT:
            intercepted = b.onInterceptTouchEvent(this, child, ev);
            break;
        case TYPE_ON_TOUCH:
            intercepted = b.onTouchEvent(this, child, ev);
            break;
    }
    if (intercepted) {
        mBehaviorTouchView = child;
    }
}
```
我们看看BottomSheetBehavior是怎样处理事件的
```java
 @Override
    public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent event) {
     
        /*
        * 省略部分代码
        */
        //拦截条件是touch 点击的位置是该 View 的范围内,并且是 MotionEvent.ACTION_MOVE 操作时候拦截,移动交给mViewDragHelper
        if (!mIgnoreEvents && mViewDragHelper.shouldInterceptTouchEvent(event)) {
            return true;
        }
        
        View scroll = mNestedScrollingChildRef.get();
        return action == MotionEvent.ACTION_MOVE && scroll != null &&
                !mIgnoreEvents && mState != STATE_DRAGGING &&
                !parent.isPointInChildBounds(scroll, (int) event.getX(), (int) event.getY()) &&
                Math.abs(mInitialY - event.getY()) > mViewDragHelper.getTouchSlop();
    }

    @Override
    public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent event) {
        if (!child.isShown()) {
            return false;
        }
        int action = MotionEventCompat.getActionMasked(event);
        if (mState == STATE_DRAGGING && action == MotionEvent.ACTION_DOWN) {
            return true;
        }
        mViewDragHelper.processTouchEvent(event);
        // Record the velocity
        if (action == MotionEvent.ACTION_DOWN) {
            reset();
        }
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(event);
        // The ViewDragHelper tries to capture only the top-most View. We have to explicitly tell it
        // to capture the bottom sheet in case it is not captured and the touch slop is passed.
        if (action == MotionEvent.ACTION_MOVE) {
            if (Math.abs(mInitialY - event.getY()) > mViewDragHelper.getTouchSlop()) {
                mViewDragHelper.captureChildView(child, event.getPointerId(event.getActionIndex()));
            }
        }
        return true;
    }
```

### NestedScrolling事件处理
当 View 的子控件有 NestedScrollingChild 产生 Nested 事件的时候.会调用onStartNestedScroll 这个方法
```java
    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child,
            View directTargetChild, View target, int nestedScrollAxes) {
            return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;//滑动Y轴方向的判断
    }
```
返回 true 表示 View 要和NestedScrollingChild 配合消耗这个 Nested 事件,进入onNestedPreScroll()方法
```java
  @Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dx,
            int dy, int[] consumed) {
        View scrollingChild = mNestedScrollingChildRef.get();
        if (target != scrollingChild) {
            return;
        }
        int currentTop = child.getTop();
        int newTop = currentTop - dy;
        if (dy > 0) { // Upward
            if (newTop < mMinOffset) {
                consumed[1] = currentTop - mMinOffset;
                ViewCompat.offsetTopAndBottom(child, -consumed[1]);
                setStateInternal(STATE_EXPANDED);
            } else {
                consumed[1] = dy;
                ViewCompat.offsetTopAndBottom(child, -dy);
                setStateInternal(STATE_DRAGGING);
            }
        } else if (dy < 0) { // Downward
            if (!ViewCompat.canScrollVertically(target, -1)) {
                if (newTop <= mMaxOffset || mHideable) {
                    consumed[1] = dy;
                    ViewCompat.offsetTopAndBottom(child, -dy);
                    setStateInternal(STATE_DRAGGING);
                } else {
                    consumed[1] = currentTop - mMaxOffset;
                    ViewCompat.offsetTopAndBottom(child, -consumed[1]);
                    setStateInternal(STATE_COLLAPSED);
                }
            }
        }
        dispatchOnSlide(child.getTop());
        mLastNestedScrollDy = dy;
        mNestedScrolled = true;
    }

```
这里是说 发起的 Nested 事件的不是onLayoutChild获取到的 View 一致的话.不消耗 y 轴
当一致的的时候消耗相对应的 y 轴,本来 nestedChild 想滑动20像素, 你消耗了10 那么nestedChild就只能滑动10像素.
消耗 y 轴,使用ViewCompat.offsetTopAndBottom(child, -consumed[1]); 这样就感觉滑动事View 被滑动了
这个方法也是ViewDragHelper实现滑动的方法.

```java
 @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child,
            View directTargetChild, View target, int nestedScrollAxes) {
   
    }

    @Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dx,
            int dy, int[] consumed) {
       
    }

    @Override
    public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) {

    }

    @Override
    public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
            float velocityX, float velocityY) {

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





  
  







  











