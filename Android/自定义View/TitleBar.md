## 自定义Titlebar --进阶

在之前的自定义View中也曾写过自定义标题栏，但当时只是为了学习而写了一个简单的例子，功能比较简单，只是作为一个练习使用。这次搞了一个自定义能力比较强的TitleBar，满足了日常的需求。

一般的app标题栏一般分为左，中，右三部分，左部分基本上都是返回按钮，特殊情况下为自定义菜单，而中，右侧菜单只是完成了基本功能。

原理很简单，只是把一些布局进行封装成一个控件，并添加一些逻辑，难度上不是很大。

先上图
![](titleBar.gif)

- 根据我们的规划，在xml中因该有如下基本的属性
  - 左侧分为三个模式：返回，自定义菜单，不显示。
  - 中部分为两个模式：自定义菜单，不显示。
  - 右侧分为两个模式：自定义菜单，不显示。

定义attrs属性：
```xml
<declare-styleable name="TitleBar">
        <attr name="leftType" >

            <enum name="gone" value="0"/>
            <enum name="customMenu" value="1"/>
            <enum name="back" value="2"/>
        </attr>
        <attr name="leftText" format="string" />
        <attr name = "leftImage" format="reference"/>



        <attr name="centerType">
            <enum name="gone" value="0"/>
            <enum name ="customMenu" value="1"/>
        </attr>
        <attr name="centerText" format="string"/>


        <attr name="rightType">
            <enum name="gone" value="0"/>
            <enum name="customMenu" value="1"/>
        </attr>

        <attr name="rightText" format="string"/>

    </declare-styleable>

```

在定义Type属性的时候，使用到了枚举类型，该属性类似于我们常用的grivaty属性一样给你固定的几个选项，在这里需要注意的是枚举类型获取其值时获取到的是其对应的value中的整型值。

- 布局widget_titlebar.xml文件。

```xml 
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="48dp"
    android:layout_marginBottom="5dp"
    android:background="#fff"
    android:elevation="5dp"
    android:orientation="vertical">

    <LinearLayout
        android:id="@+id/ll_titlebar_left"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_marginLeft="5dp"
        android:gravity="center">

        <ImageView
            android:id="@+id/iv_titlebar_left"
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:layout_gravity="center"
            android:src="@mipmap/back_1" />

        <TextView
            android:id="@+id/tv_titlebar_left"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="左侧标题"
            android:textSize="18sp" />

    </LinearLayout>


    <LinearLayout
        android:id="@+id/ll_titlebar_center"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:gravity="center">

        <TextView
            android:id="@+id/tv_titlebar_center"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="中标题"
            android:textColor="#222"
            android:textSize="20sp" />

    </LinearLayout>


    <LinearLayout
        android:id="@+id/ll_titlebar_right"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_alignParentRight="true"
        android:layout_marginRight="10dp"
        android:gravity="center">

        <TextView
            android:id="@+id/tv_titlebar_right"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="右侧按钮"
            android:textSize="18sp" />

    </LinearLayout>
</RelativeLayout>
```
布局上没有什么问题，唯一需要注意是elevation属性，在这里定义了阴影偏移为5dp，一开始，我定义的时候，在我们的布局显示上可以看到阴影，但在真机和模拟器调试上确无法看到，这里有几个很重要的点。
	1. 需要定义该布局的背景颜色。
	2. 需要添加一个margin属性。应该我们定义了偏移量之后，在父控件中并没有考虑到该控件的偏移量大小，导致给该控件留的大小刚好等于控件自身大小，导致偏移量无法显示。

- 在Java中获取属性，并查找控件
```java
    /**
     * 获取自定义属性
     *
     * @param context
     * @param attrs
     */
    private void initAttrs(Context context, AttributeSet attrs) {
        if (attrs != null) {
            TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.TitleBar);

            mLeftText = ta.getString(R.styleable.TitleBar_leftText);

            mLeftType = ta.getInteger(R.styleable.TitleBar_leftType, TYPE_BACK);

            mLeftImage = ta.getDrawable(R.styleable.TitleBar_leftImage);

            mCenterType = ta.getInteger(R.styleable.TitleBar_centerType, TYPE_CUSTOM_MENU);
            mCenterText = ta.getString(R.styleable.TitleBar_centerText);

            mRightType = ta.getInteger(R.styleable.TitleBar_rightType, TYPE_GONE);
            mRightText = ta.getString(R.styleable.TitleBar_rightText);

        }
    }


   /**
     * 查找控件id
     */
    private void initView() {
        mLeftLinearLayout = (LinearLayout) findViewById(R.id.ll_titlebar_left);
        mLeftImageView = (ImageView) findViewById(R.id.iv_titlebar_left);
        mLeftTextView = (TextView) findViewById(R.id.tv_titlebar_left);

        mCenterTextView = (TextView) findViewById(R.id.tv_titlebar_center);
        mCenterLinearLayout = (LinearLayout) findViewById(R.id.ll_titlebar_center);

        mRightTextView = (TextView) findViewById(R.id.tv_titlebar_right);
        mRightLinearLayout = (LinearLayout) findViewById(R.id.ll_titlebar_right);
    }

```

- 根据xml文件属性初始化状态，
```java 

    /**
     * 加载默认视图
     */
    private void loadView() {
        if (mLeftType == TYPE_BACK) {
            mLeftText = "返回";
            mLeftTextView.setVisibility(View.VISIBLE);
            mLeftTextView.setText(mLeftText);

            mLeftImageView.setImageResource(R.mipmap.back_1);
            mLeftImageView.setVisibility(View.VISIBLE);

        } else if (mLeftType == TYPE_CUSTOM_MENU) {
            if (mLeftImage == null) {
                mLeftImageView.setVisibility(View.GONE);

            } else {
                mLeftImageView.setVisibility(View.VISIBLE);
                mLeftImageView.setImageDrawable(mLeftImage);
            }

            if (!TextUtils.isEmpty(mLeftText)) {
                mLeftTextView.setVisibility(View.VISIBLE);
                mLeftTextView.setText(mLeftText);
            } else {
                mLeftTextView.setVisibility(View.GONE);
            }

        } else if (mLeftType == TYPE_GONE) {
            mLeftTextView.setVisibility(View.GONE);
            mLeftImageView.setVisibility(View.GONE);
        }


        if (mCenterType == TYPE_CUSTOM_MENU) {
            if (!TextUtils.isEmpty(mCenterText)) {
                mCenterTextView.setVisibility(View.VISIBLE);
                mCenterTextView.setText(mCenterText);
            } else {
                mCenterTextView.setVisibility(View.GONE);
            }
        } else if (mCenterType == TYPE_GONE) {
            mCenterTextView.setVisibility(View.GONE);
        }


        if (mRightType == TYPE_CUSTOM_MENU) {
            if (!TextUtils.isEmpty(mRightText)) {
                mRightTextView.setVisibility(View.VISIBLE);
                mRightTextView.setText(mRightText);
            } else {
                mRightTextView.setVisibility(View.GONE);
            }
        } else if (mRightType == TYPE_GONE) {
            mRightTextView.setVisibility(View.GONE);
        }

    }

```

- 设置监听回调。定义监听回调类，在这里为了省事定义成了内部类
```java 
    /**
     * 监听回调事件
     */
    public static abstract class TitleBarClickListener {
        public abstract void onLeftClick();

        public void onCenterClick() {
        }

        public void onRightClick() {
        }
    }

```

因为对于标题栏，使用最多的是左侧按钮，所以在这里定义个抽象类，必须实现左侧点击，中心和右侧选择定义。因为使用了抽象类，相比于接口来说，继承不是很方便，有利有弊吧。

- 创建监听方法,并实现回调
```java 
   /**
     * @param mTitleBarClickListener
     */
    public void setTitleBarClickListener(TitleBarClickListener mTitleBarClickListener) {
        this.mTitleBarClickListener = mTitleBarClickListener;
    }


    /**
     * 设置左右菜单的监听
     */
    private void initClick() {
        mLeftLinearLayout.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mTitleBarClickListener != null) {
                    mTitleBarClickListener.onLeftClick();
                }
            }
        });

        mCenterLinearLayout.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mTitleBarClickListener != null) {
                    mTitleBarClickListener.onCenterClick();
                }
            }
        });

        mRightLinearLayout.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mTitleBarClickListener != null) {
                    mTitleBarClickListener.onRightClick();
                }
            }
        });
    }

```

- 最后对于左中右菜单可能会有一些比较特别的布局，添加如下方法
```java 
 public void addLeftView(View view) {
        mLeftLinearLayout.removeAllViews();
        mLeftLinearLayout.addView(view);
    }

    public void addCenterView(View view) {
        mCenterLinearLayout.removeAllViews();
        mCenterLinearLayout.addView(view);
    }


    public void addRightView(View view) {
        mRightLinearLayout.removeAllViews();
        mRightLinearLayout.addView(view);
    }

```

- 最后一步，添加布局文件。
``` xml
<mahao.alex.titlebar.TitleBar
        android:id="@+id/titlebar1"
        app:leftType="back"
        app:centerType="gone"
        app:rightType="gone"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>



    <mahao.alex.titlebar.TitleBar
        android:id="@+id/titlebar2"
        android:layout_marginTop="40dp"
        app:leftType="customMenu"
        app:leftImage="@mipmap/ic_launcher"
        app:leftText="自定义"
        app:centerType="customMenu"
        app:centerText="自定义中"
        app:rightType="customMenu"
        app:rightText="右菜单"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

```

```java 
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        requestWindowFeature(Window.FEATURE_NO_TITLE);
        setContentView(R.layout.activity_main);
        ((TitleBar) findViewById(R.id.titlebar1)).setTitleBarClickListener(new TitleBar.TitleBarClickListener(){

            @Override
            public void onLeftClick() {
                Toast.makeText(MainActivity.this, "back", Toast.LENGTH_SHORT).show();
            }
        });

        ((TitleBar) findViewById(R.id.titlebar2)).setTitleBarClickListener(new TitleBar.TitleBarClickListener(){

            @Override
            public void onLeftClick() {
                Toast.makeText(MainActivity.this, "左菜单", Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onCenterClick() {
                Toast.makeText(MainActivity.this, "中", Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onRightClick() {
                Toast.makeText(MainActivity.this, "右", Toast.LENGTH_SHORT).show();
            }
        });

    }
}
```

现在只是V1.0版本，后续继续改进o(^▽^)o。

**待改进**
- 点击效果
- 菜单大小，颜色。。。

该代码已上传至github..有兴趣者请移步https://github.com/AlexSmille/TitleBar


