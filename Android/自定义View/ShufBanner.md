## 无限大图轮播--ShufBanner

轮播图作为一个app的宣传，展示等，往往占据着一个很重要的地位，大部分app都将其放在首页。那么通常的做法都是使用ViewPager，使其能够作用滑动，而无限轮播无外乎两种做法。
- 第一种是将ViewPager的size定义为无限大，定义其初始显示的位置为中间，这样的话因为左或者右都有很多的页面，所以造成了一种可以无限轮播的假象。同时因为ViewPager的特性，其只是加载当前显示page以及左和右的三个页面，不用担心OOM。
- 第二种是，将ViewPager的最前和最后的页面复制一份之后，分别加入到最后和最前，当ViewPager滑动到最前，或者最后的位置，直接跳转到相对应的位置。即可。

本次自定义的ShufBanner使用的是第二种方式。



首先看图
![](shufBanner.gif)

他的使用方法很简单，如下即可在xml文件中添加控件
```xml 
  <mahao.alex.shuffingbanner.ShufBanner
        android:id="@+id/shuf"
        android:layout_width="match_parent"
        android:layout_height="300dp"/>

```

```java 

public class MainActivity extends AppCompatActivity  {


    private ShufBanner mShufBanner;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        List<String> mImages = new ArrayList<>();
        mImages.add("http://pic.huodongjia.com/event/2015-11-18/event151612.jpg");
        mImages.add("http://pic.huodongjia.com/event/2016-03-17/event177131.jpg");
        mImages.add("http://pic.huodongjia.com/event/2015-12-03/event156106.jpg");


        mShufBanner = ((ShufBanner) findViewById(R.id.shuf));

        //启动轮播图
        mShufBanner.startShuf(mImages,true);


        //设置监听
        mShufBanner.setItemClcikListener(new ShufBannerClickListener() {
            @Override
            public void onClick(int position) {
                Toast.makeText(MainActivity.this, ""+position, Toast.LENGTH_SHORT).show();
            }
        });
    }
    
}

```
只需要启动轮播，设置监听即可。下面我们就开始封装。

- 首先考虑布局，布局应该是一个ViewPager，下面是一个线性布局用来放置导航点。
```xml 
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="300dp"
    android:orientation="vertical"
    >


    <android.support.v4.view.ViewPager
        android:id="@+id/vp"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
    <RelativeLayout
        android:layout_alignParentBottom="true"
        android:background="#6fff"
        android:layout_width="match_parent"
        android:layout_height="20dp">

        <LinearLayout
            android:id="@+id/ll_navigation"
            android:layout_centerInParent="true"
            android:orientation="horizontal"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

    </RelativeLayout>

</RelativeLayout>

```
ViewPager没有什么疑问，而之所以在这里放置一个LinearLayout，是因为我们并不确定有多少页，所以我们需要根据实际情况在代码里添加。

- 创建类shufBanner继承RelativeLayout,加载布局文件，初始化ViewPager，查找控件等等。
```java 

    public ShufBanner(Context context, AttributeSet attrs) {
        super(context, attrs);

        inflate(getContext(), R.layout.widget_shufbanner, this);

        initImageLoader();

        initViewPager();

    }

    /**
     * 初始化ViewPager
     */
    private void initViewPager() {
        mVp = ((ViewPager) findViewById(R.id.vp));
		//图片url地址
        mImages = new ArrayList<>();
		//imageView对象
        mImageViews = new ArrayList<>();

        mAdapter = new ShufBannerAdapter(getContext(), mImageViews);

        mVp.setAdapter(mAdapter);

        mVp.addOnPageChangeListener(this);

  		mNavigationLayout = (LinearLayout) findViewById(R.id.ll_navigation);


    }
```
在这里我们首先将布局文件加载到ShufBanner中，其次初始化ViewPager，并设置其Adapter，添加滚动监听，监听事件的逻辑后面再说。同时查找到我们mNavigationLayout(导航点的父控件)，我们看一下ShufBannerAdapter的代码
```java 
public class ShufBannerAdapter extends PagerAdapter {

    private Context context;
    private List<ImageView> mImageViews;

    public ShufBannerAdapter(Context context, List<ImageView> mImages) {
        this.context = context;
        this.mImageViews = mImages;

    }

    @Override
    public int getCount() {
        return mImageViews==null?0:mImageViews.size();
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        container.addView(mImageViews.get(position));

        return mImageViews.get(position);
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView(mImageViews.get(position));
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }
}
```
ShufBannerAdapter很简单，只是一个最基本的适配器，没有在里面处理什么逻辑。

基本的东西已经添加完，下面就是开始实现具体的逻辑，即完成startShuf（）方法的逻辑。
```java 
 public void startShuf(List<String> urls, boolean isStartShuf) {

        if (urls.size() == 0 || urls == null) {
            return;
        }
        /**
         * 清楚数据
         */
        clearAllData();


        mImages.addAll(urls);
        /**
         * 保存图片地址
         */
        saveUrl();

        /**
         * 根据图片url创建imgView
         */
        createImageView();

        /**
         * 生成导航点
         */
        addPoint2Navigation();


        //开始刷新
        mAdapter.notifyDataSetChanged();
        mVp.setCurrentItem(1);

        //发送循环请求
        this.isStartShuf = isStartShuf;
        if (this.isStartShuf&&mImages.size()>3) {
            handler.sendEmptyMessageAtTime(1, 2000);
        }
    }
```
首先判断为null之类的都懂。下面清除数据，该方法主要目的是，当我们刷新数据时，我们必须把之前老的数据清楚，同时一些属性置空。

`mImages.addAll(urls);`将我们的图片集合添加到当前集合中，因为我们需要实现无限循环，即把最前和最后添加一张图片，所以我们通过saveUrl()方法添加数据。
```java

    private void saveUrl() {

        String startImageUrl = mImages.get(0);
        String endImageUrl = mImages.get(mImages.size() - 1);

        mImages.add(0, endImageUrl);
        mImages.add(mImages.size(), startImageUrl);
    }

```
我们获取到我们图片url集合的第一张和最后一张，将第一张加到List末尾，将最后一张加到list的最前端。就好比，我们本来有三个url：1，2，3。经过saveUrl方法之后，url：_3，1，2，3，_1。

createImageView()方法根据url创建对应的imageView对象并存储到list集合中。

addPoint2Navigation()方法是根据当前的ImageView添加导航点，我们看一下他的详细实现过程：
```java 
 /**
     * 添加导航点到布局中
     */
    private void addPoint2Navigation() {
        LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.WRAP_CONTENT,
                LinearLayout.LayoutParams.WRAP_CONTENT
        );
        int pointPadding = (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 5, getResources().getDisplayMetrics());
        params.setMargins(pointPadding, 0, pointPadding, 0);
        for (int i = 0; i < mImageViews.size() - 2; i++) {
            ImageView point = new ImageView(getContext());
            point.setLayoutParams(params);
            point.setImageResource(R.drawable.select_navgation_point);
            if (i == 0) {
                point.setEnabled(true);
            } else {
                point.setEnabled(false);
            }
            mNavigationLayout.addView(point);
        }
    }

```
我们添加导航点的时候需要注意，因为我们添加了两张图片，所以我们在添加的实际导航点数 = 图片数-2；导航点使用的shape图形资源画出的，同时导航点因为有选中和不选中的两种状态，通过定义selector改变颜色变化，因为使用的是enabled属性来判断，所以在这里设置Enable属性。

```java 
 //发送循环请求
        this.isStartShuf = isStartShuf;
        if (this.isStartShuf&&mImages.size()>3) {
            handler.sendEmptyMessageAtTime(1, 2000);
        }
```

ok 下面开始进入到了轮播的逻辑了。

我们发送一个message给handler。
```java 
private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {

            int showPonit = mVp.getCurrentItem();

            showPonit = showPonit + 1;

            //getChildCount()获取的是当前存在的子类，销毁机制

           /* if (showPonit >= mImageViews.size()) {

                showPonit = 1;
            }*/

            //获得的是正常位置
            mVp.setCurrentItem(showPonit);
            
            if (isStartShuf) {
                sendEmptyMessageDelayed(1, 2000);
            }
        }
    };
```

在handler中，我们获取到当前的的页面，并+1。

判断是否超过了最大页面，如果超过了最大页面，就置为1，即我们的第一页，但后来又被我注掉，因为，我发现根本不需要。有点想多了。我们先继续往下看，设置页面，判断是否轮播，继续发送message。


到这里，我们的页面开始滚动了，但我们还需要处理逻辑，因为我们的页面现在是_3，1，2，3，_1.现在我们要加入判断，我们利用setCurrentItem(int,boolean）方法，到页面_1时，跳转到1，当页面在_3时，我们默认跳转到3。很明显，添加mVp.addOnPageChangeListener(this);

该方法有三个回调，分别是onPageScrolled：页面滚动距离，onPageSelected：页面选择，onPageScrollStateChanged：页面滚动状态的选择。

在这里我选择在onPageScrolled方法
```java
  @Override
    public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

		//是否滚动过渡完结
        if (positionOffset == 0) {
            if (position == 0) {
                mVp.setCurrentItem(mImageViews.size() - 2, false);
            }

            if (position == mImageViews.size() - 1) {
                mVp.setCurrentItem(1, false);
            }
        }
    }
```

最初，我是在onPageSelected(int position)方法中，添加跳转逻辑，但有一个问题。当我们调用
setCurrentItem（）方法时,会立即回掉onPageSelected，但此时我们页面还是在滚动过渡中，如果此时3想_1滚动，但滚动到一半时，有调用了跳转到1的方法，切该方法无过渡，导致页面闪烁。所以我们在onPageScrolled判断，当滚动完毕，我们开始进行对应跳转。

在之前handler中，我添加了一个边界的判断，但在这里会发现，我们有了这个过渡跳转，所以不会存在边界。

页面已经滚动，下面开始就是要将页面的滚动和导航点联动，这个在onPageSelected中即可
```java 
    @Override
    public void onPageSelected(int position) {
        /**
         * 修改当前点击点的位置
         */
        changePosition(position);

        /**
         * 修改点的状态
         */
        changePointState(position);
    }
```
changePointState(position);为修改点的状态。

```java 
**
     * 改变导航点的状态
     *
     * @param position
     */
    private void changePointState(int position) {

        if (position == 0) {
            position = mImageViews.size() - 2;
        }
        if (position == mImageViews.size() - 1) {
            position = 1;
        }

        position = position - 1;

        for (int i = 0; i < mNavigationLayout.getChildCount(); i++) {
            if (i == position) {
                mNavigationLayout.getChildAt(i).setEnabled(true);

            } else {
                mNavigationLayout.getChildAt(i).setEnabled(false);
            }
        }
    }
```
这个就是判断一下是否是_3和_1，改变position对应的3和1，最后遍历修改状态。

changePosition(position);这个方法，是干什么呢。这就涉及到下一个关键，及ShufBanner的点击事件回调。

对于轮播图，往往都会有详情页，及跳转链接。如果我们对于每个ImageView都加入监听，这无疑很麻烦，我们还要判断具体的点击，以及_3,_1的问题。所以在这里我定义了一个字段
```java
    /**
     * 点击事件的位置
     */
    private int position = -1;

```
该字段，用来保存当前ViewPager的position。默认等于-1。
```java
   /**
     * 改变当前点击的点
     *
     * @param position
     */
    private void changePosition(int position) {
        if (position == 0) {
            position = mImageViews.size() - 2;
        }
        if (position == mImageViews.size() - 1) {
            position = 1;
        }

        position--;
        this.position = position;
    }

```
因为我需要接口回调
```java 
public interface ShufBannerClickListener {
    /**
     *
     * @param position -1 未设置任何点击
     */
    void onClick(int position);
}
```
我们如果直接记录position，我们自己内部悄悄添加了两个页面。这样，对于调用者来说，我传入的数据和返回的角标数据无法一一对应，这样肯定是不对。所以在这里改变点的状态。

这样应该差不多了，我们看一下调用方式

```java

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        List<String> mImages = new ArrayList<>();
        mImages.add("http://pic.huodongjia.com/event/2015-11-18/event151612.jpg");
        mImages.add("http://pic.huodongjia.com/event/2016-03-17/event177131.jpg");
        mImages.add("http://pic.huodongjia.com/event/2015-12-03/event156106.jpg");


        mShufBanner = ((ShufBanner) findViewById(R.id.shuf));

        //启动轮播图
        mShufBanner.startShuf(mImages,true);


        //设置监听
        mShufBanner.setItemClcikListener(new ShufBannerClickListener() {
            @Override
            public void onClick(int position) {
                Toast.makeText(MainActivity.this, ""+position, Toast.LENGTH_SHORT).show();
            }
        });
    } 
```

Over。

> 总结：
> 1. 大图轮播实现的原理：1，2，3三个页面，我们最后和之前各添加一个页面，_3,1,2,3,_1,然后实现逻辑当_3页面时跳转到3，_1跳转1.
> 2. ViewPager的getCount()方法，ViewPager的自带销毁机制，所以，和我们想要获取的值会有出入。

该项目于已发布与github，有意者请移步[ShuffingBanner](https://github.com/AlexSmille/shuffingBanner)