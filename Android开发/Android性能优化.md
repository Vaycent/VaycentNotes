[TOC]

#Android性能优化

* 布局优化

* 绘制优化

* 内存泄漏

* 响应速度优化

* ListView优化

* Bitmap优化

* 线程优化





## 1_布局优化

----

**尽量减少布局文件的层级，布局的层级少了，就意味着Android绘制时的工作量少了**

1. 非嵌套的简单界面时尽量使用LinearLayout和FrameLayout，它们是简单高效的ViewGroup，更低开销

2. 嵌套的复杂界面时还是使用RelativeLayout，因为ViewGroup的嵌套相当于增加了布局的层级

3. 采用<include>标签、<merge>标签、ViewStub



### 1.1_include标签

创建可复用的布局，将一个指定的布局文件加载到当前的布局中

```

<include layout="@layout/test_include_layout"/>

```



### 1.2_merge标签

沿用父ViewGroup布局，去除多余的布局

```

//用merge取代LinearLayout，将采用父LinearLayout的布局

<merge xmlns:android="http://schemas.android.com/apk/res/android">

<Button android:layout_width="wrap_content"

..../>

<Button android:layout_width="wrap_content"

.../>

</merge>

```



### 1.3_ViewStub加载

继承自View，它非常轻量级且宽／高都是0，因此它本身不参与任何布局和绘制的过程。可按需加载所需布局文件

```

//inflatedId是它的layout布局的根元素id

<ViewStub android:id="@+id/viewstub_id"

android:inflatedId="@+id/panel_layout"

android:layout="@layout/layout_network_error"

.../>

```

加载时可以按照以下两种方式之一进行

```

((ViewStub)findViewById(R.id.viewstub_id)).setVisibility(View.VISIBLE);

//或者

View panelLayout = ((ViewStub)findViewById(R.id.viewstub_id)).inflate();

```



## 2_绘制优化

---

**View的onDraw方法要避免执行大量的操作**

1. onDraw中**不要创建新的局部对象**，因为频繁调用且在调用瞬间产生大量临时对象，会占用过多内存和导致系统频繁GC

2. onDraw中**不要做耗时的任务**，大量的循环会抢占CPU的时间片，View绘制的帧率60fps最佳，响应时间为16ms(16ms=1000ms/60)



## 3_内存泄漏

---

1. 静态变量导致

2. 单例模式导致

3. 属性动画导致

### 3.1_静态变量导致

静态变量持有了当前Activity，所以Activity退出时无法释放



```

//错误例子，两个静态变量持有了当前Activity，都会导致Activity无法释放

private static Context sContext;

private static View sView;

@Override

protected void onCreate(Bundle saveInstanceState){

    super.onCreate(saveInstanceState);

    sContext = this;

    sView = new View(this);

}

```



### 3.2_单例模式导致

单例模式的特点是其生命周期和Application保持一致，所以Activity对象无法被及时释放

```

//错误例子

public class MemoryManager {

    private static MemoryManager instance;

    private Context context;

    private MemoryManager(Context context){
        this.context = context;
    }

    public static MemoryManager getInstance(Context context){
        if(instance == null)
            instance = new MemoryManager(context);
        return instance;
    }
}
```

```

//在Activity的OnCreate中调用单例

MemoryManager mmM = MemoryManager.getInstance(this);
```



### 3.3_属性动画导致

属性动画中有一类无限循环的动画，如果Activity中播放此类动画且没有在onDestroy中停止动画，退出时的Activity的View会被动画持有，而View又持有Activity，最终Activity无法释放。

```

//错误例子

ObjectAnimator animator = ObjectAnimator.ofFloat(toObjectAnimation,"rotation",0,360).setDuration(1000);
animator.setRepeatCount(ValueAnimator.INFINITE);
animator.start(); 
//animator.cancel();
```



## 4_响应速度优化

---

响应速度优化的核心就是**避免在主线程中做耗时操作**。主线程做太多事情会导致Activity出现黑屏甚至ANR。

Android规定，Activity如果5秒钟之内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，而BroadcastReceiver如果10秒之内还未执行操作也会出现ANR。

* 分析ANR的原因可用adb取出traces文件查看:

```

adb pull /data／anr／traces.txt ./Desktop

```

* 以下程序通过同步锁会导致黑屏甚至ANR，是错误例子:

```

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    new Thread(new Runnable() {
        @Override
        public void run() {
            testANR();
        }
    }).start();

    SystemClock.sleep(10);
    initView();
}

private synchronized void testANR(){
    SystemClock.sleep(30 * 1000);
}
```



## 5_ListView、GridView优化

---

* 采用ViewHolder

* 避免在getView中执行耗时操作

* **根据滑动状态控制任务执行频率**，如列表快速滑动时不适合开启大量异步任务

* 可尝试开启硬件加速使滑动更加流畅



## 6_Bitmap优化

---

* 通过BitmapFactory.Options对图片进行采样

* 第三方图片加载技术



## 7_线程优化

---

* 采用线程池，避免程序中存在大量Thread



## 8_其他优化

---

* 避免创建过多对象

* 不要过多使用枚举，**枚举占用的内存空间比整型大**

* 常量使用static final修饰

* **使用一些Android特有的数据结构，比如SparseArray和Pair等**

* **适当使用软引用和弱引用**

* 采用内存缓存和磁盘缓存

* 尽量采用**静态内部类，可以避免潜在的由于内部类而导致内存泄漏**