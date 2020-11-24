> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/guolin_blog/article/details/8881711)

转载请注明出处：[http://blog.csdn.net/guolin_blog/article/details/8881711](http://blog.csdn.net/guolin_blog/article/details/8881711)[](http://blog.csdn.net/sinyu890807/article/details/8868758)  

我们都知道，Android 上的界面展示都是通过 Activity 实现的，Activity 实在是太常用了，我相信大家都已经非常熟悉了，这里就不再赘述。

但是 Activity 也有它的局限性，同样的界面在手机上显示可能很好看，在平板上就未必了，因为平板的屏幕非常大，手机的界面放在平板上可能会有过分被拉长、控件间距过大等情况。这个时候更好的体验效果是在 Activity 中嵌入 "小 Activity"，然后每个 "小 Activity" 又可以拥有自己的布局。因此，我们今天的主角 Fragment 登场了。

**Fragment 初探**

为了让界面可以在平板上更好地展示，Android 在 3.0 版本引入了 Fragment(碎片) 功能，它非常类似于 Activity，可以像 Activity 一样包含布局。Fragment 通常是嵌套在 Activity 中使用的，现在想象这种场景：有两个 Fragment，Fragment 1 包含了一个 ListView，每行显示一本书的标题。Fragment 2 包含了 TextView 和 ImageView，来显示书的详细内容和图片。

如果现在程序运行竖屏模式的平板或手机上，Fragment 1 可能嵌入在一个 Activity 中，而 Fragment 2 可能嵌入在另一个 Activity 中，如下图所示：

![](https://img-blog.csdn.net/20130505194512607)  

而如果现在程序运行在横屏模式的平板上，两个 Fragment 就可以嵌入在同一个 Activity 中了，如下图所示：

![](https://img-blog.csdn.net/20130505194518044)  

由此可以看出，使用 Fragment 可以让我们更加充分地利用平板的屏幕空间，下面我们一起来探究下如何使用 Fragment。

首先需要注意，Fragment 是在 3.0 版本引入的，如果你使用的是 3.0 之前的系统，需要先导入 android-support-v4 的 jar 包才能使用 Fragment 功能。

新建一个项目叫做 Fragments，然后在 layout 文件夹下新建一个名为 fragment1.xml 的布局文件：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#00ff00" >
 
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="This is fragment 1"
        android:textColor="#000000"
        android:textSize="25sp" />
 
</LinearLayout>
```

可以看到，这个布局文件非常简单，只有一个 LinearLayout，里面加入了一个 TextView。我们如法炮制再新建一个 fragment2.xml ：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#ffff00" >
 
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="This is fragment 2"
        android:textColor="#000000"
        android:textSize="25sp" />
 
</LinearLayout>
```

然后新建一个类 Fragment1，这个类是继承自 Fragment 的：

```
public class Fragment1 extends Fragment {
 
	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
		return inflater.inflate(R.layout.fragment1, container, false);
	}
 
}
```

我们可以看到，这个类也非常简单，主要就是加载了我们刚刚写好的 fragment1.xml 布局文件并返回。同样的方法，我们再写好 Fragment2 ：

```
public class Fragment2 extends Fragment {
 
	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
		return inflater.inflate(R.layout.fragment2, container, false);
	}
 
}
```

然后打开或新建 activity_main.xml 作为主 Activity 的布局文件，在里面加入两个 Fragment 的引用，使用 android:name 前缀来引用具体的 Fragment：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:baselineAligned="false" >
 
    <fragment
        android:id="@+id/fragment1"
        android:
        android:layout_width="0dip"
        android:layout_height="match_parent"
        android:layout_weight="1" />
 
    <fragment
        android:id="@+id/fragment2"
        android:
        android:layout_width="0dip"
        android:layout_height="match_parent"
        android:layout_weight="1" />
 
</LinearLayout>
```

最后打开或新建 MainActivity 作为程序的主 Activity，里面的代码非常简单，都是自动生成的：

```
public class MainActivity extends Activity {
 
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
	}
 
}
```

现在我们来运行一次程序，就会看到，一个 Activity 很融洽地包含了两个 Fragment，这两个 Fragment 平分了整个屏幕，效果图如下：

![](https://img-blog.csdn.net/20130505195012949)  

**动态添加 Fragment**  

你已经学会了如何在 XML 中使用 Fragment，但是这仅仅是 Fragment 最简单的功能而已。Fragment 真正的强大之处在于可以动态地添加到 Activity 当中，因此这也是你必须要掌握的东西。当你学会了在程序运行时向 Activity 添加 Fragment，程序的界面就可以定制的更加多样化。下面我们立刻来看看，如何动态添加 Fragment。

还是在上一节代码的基础上修改，打开 activity_main.xml，将其中对 Fragment 的引用都删除，只保留最外层的 LinearLayout，并给它添加一个 id，因为我们要动态添加 Fragment，不用在 XML 里添加了，删除后代码如下：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/main_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:baselineAligned="false" >
 
</LinearLayout>
```

然后打开 MainActivity，修改其中的代码如下所示：

```
public class MainActivity extends Activity {
 
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		Display display = getWindowManager().getDefaultDisplay();
		if (display.getWidth() > display.getHeight()) {
			Fragment1 fragment1 = new Fragment1();
			getFragmentManager().beginTransaction().replace(R.id.main_layout, fragment1).commit();
		} else {
			Fragment2 fragment2 = new Fragment2();
			getFragmentManager().beginTransaction().replace(R.id.main_layout, fragment2).commit();
		}
	}
 
}
```

首先，我们要获取屏幕的宽度和高度，然后进行判断，如果屏幕宽度大于高度就添加 fragment1，如果高度大于宽度就添加 fragment2。动态添加 Fragment 主要分为 4 步：

1. 获取到 FragmentManager，在 Activity 中可以直接通过 getFragmentManager 得到。

2. 开启一个事务，通过调用 beginTransaction 方法开启。

3. 向容器内加入 Fragment，一般使用 replace 方法实现，需要传入容器的 id 和 Fragment 的实例。

4. 提交事务，调用 commit 方法提交。

现在运行一下程序，效果如下图所示：

![](https://img-blog.csdn.net/20130505195313641)  

如果你是在使用模拟器运行，按下 ctrl + F11 切换到竖屏模式。效果如下图所示：

![](https://img-blog.csdn.net/20130505195318396)  

**Fragment 的生命周期**  

和 Activity 一样，Fragment 也有自己的生命周期，理解 Fragment 的生命周期非常重要，我们通过代码的方式来瞧一瞧 Fragment 的生命周期是什么样的：

```
public class Fragment1 extends Fragment {
	public static final String TAG = "Fragment1";
 
	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
		Log.d(TAG, "onCreateView");
		return inflater.inflate(R.layout.fragment1, container, false);
	}
 
	@Override
	public void onAttach(Activity activity) {
		super.onAttach(activity);
		Log.d(TAG, "onAttach");
	}
 
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		Log.d(TAG, "onCreate");
	}
 
	@Override
	public void onActivityCreated(Bundle savedInstanceState) {
		super.onActivityCreated(savedInstanceState);
		Log.d(TAG, "onActivityCreated");
	}
 
	@Override
	public void onStart() {
		super.onStart();
		Log.d(TAG, "onStart");
	}
 
	@Override
	public void onResume() {
		super.onResume();
		Log.d(TAG, "onResume");
	}
 
	@Override
	public void onPause() {
		super.onPause();
		Log.d(TAG, "onPause");
	}
 
	@Override
	public void onStop() {
		super.onStop();
		Log.d(TAG, "onStop");
	}
 
	@Override
	public void onDestroyView() {
		super.onDestroyView();
		Log.d(TAG, "onDestroyView");
	}
 
	@Override
	public void onDestroy() {
		super.onDestroy();
		Log.d(TAG, "onDestroy");
	}
 
	@Override
	public void onDetach() {
		super.onDetach();
		Log.d(TAG, "onDetach");
	}
 
}
```

可以看到，上面的代码在每个生命周期的方法里都打印了日志，然后我们来运行一下程序，可以看到打印日志如下：

![](https://img-blog.csdn.net/20130506203450902)  

这时点击一下 home 键，打印日志如下：

![](https://img-blog.csdn.net/20130506203611071)  

如果你再重新进入进入程序，打印日志如下：

![](https://img-blog.csdn.net/20130506203845394)  

然后点击 back 键退出程序，打印日志如下：

![](https://img-blog.csdn.net/20130506204611615)  

看到这里，我相信大多数朋友已经非常明白了，因为这和 Activity 的生命周期太相似了。只是有几个 Activity 中没有的新方法，这里需要重点介绍一下：

*   onAttach 方法：Fragment 和 Activity 建立关联的时候调用。
*   onCreateView 方法：为 Fragment 加载布局时调用。
*   onActivityCreated 方法：当 Activity 中的 onCreate 方法执行完后调用。
*   onDestroyView 方法：Fragment 中的布局被移除时调用。
*   onDetach 方法：Fragment 和 Activity 解除关联的时候调用。

**Fragment 之间进行通信**  

通常情况下，Activity 都会包含多个 Fragment，这时多个 Fragment 之间如何进行通信就是个非常重要的问题了。我们通过一个例子来看一下，如何在一个 Fragment 中去访问另一个 Fragment 的视图。

还是在第一节代码的基础上修改，首先打开 fragment2.xml，在这个布局里面添加一个按钮：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="#ffff00" >
 
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="This is fragment 2"
        android:textColor="#000000"
        android:textSize="25sp" />
    
    <Button 
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Get fragment1 text"
        />
 
</LinearLayout>
```

然后打开 fragment1.xml，为 TextView 添加一个 id：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#00ff00" >
 
    <TextView
        android:id="@+id/fragment1_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="This is fragment 1"
        android:textColor="#000000"
        android:textSize="25sp" />
 
</LinearLayout>
```

接着打开 Fragment2.java，添加 onActivityCreated 方法，并处理按钮的点击事件：

```
public class Fragment2 extends Fragment {
 
	@Override
	public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
		return inflater.inflate(R.layout.fragment2, container, false);
	}
 
	@Override
	public void onActivityCreated(Bundle savedInstanceState) {
		super.onActivityCreated(savedInstanceState);
		Button button = (Button) getActivity().findViewById(R.id.button);
		button.setOnClickListener(new OnClickListener() {
			@Override
			public void onClick(View v) {
				TextView textView = (TextView) getActivity().findViewById(R.id.fragment1_text);
				Toast.makeText(getActivity(), textView.getText(), Toast.LENGTH_LONG).show();
			}
		});
	}
 
}
```

现在运行一下程序，并点击一下 fragment2 上的按钮，效果如下图所示：

![](https://img-blog.csdn.net/20130506222034283)

我们可以看到，在 fragment2 中成功获取到了 fragment1 中的视图，并弹出 Toast。这是怎么实现的呢？主要都是通过 getActivity 这个方法实现的。getActivity 方法可以让 Fragment 获取到关联的 Activity，然后再调用 Activity 的 findViewById 方法，就可以获取到和这个 Activity 关联的其它 Fragment 的视图了。

好了，以上就是关于 Fragment 你所须知道的一切。如果想要切身体验一下 Fragment 的实战，请继续阅读 [Android 手机平板两不误，使用 Fragment 实现兼容手机和平板的程序](http://blog.csdn.net/guolin_blog/article/details/8744943) 以及 [Android Fragment 应用实战，使用碎片向 ActivityGroup 说再见](http://blog.csdn.net/guolin_blog/article/details/13171191) 。

blockquote{border-left: 10px solid rgba(128,128,128,0.075); background-color: rgba(128,128,128,0.05); border-radius: 0 5px 5px 0; padding: 15px 20px; }

关注我的技术公众号，每天都有优质技术文章推送。关注我的娱乐公众号，工作、学习累了的时候放松一下自己。

微信扫一扫下方二维码即可关注：

![](https://img-blog.csdn.net/20160507110203928)         ![](https://img-blog.csdn.net/20161011100137978)