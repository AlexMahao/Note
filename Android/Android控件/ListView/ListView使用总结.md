## ListView使用总结

虽然随着`RecyclerView`的不断普及，相应的资源也越来越多，许多的项目都在使用`RecyclerView`，但作为他的前辈`ListView`，加深对`ListView`的使用有助于我们更好的适应到`RecyclerView`的使用中。

### ListView的优化

`ListView`的优化主要包括两个方面，分别是对自身的优化以及其适配器（`Adapter`）的优化。

#### `ListView`自身的优化

主要包括一条。对于`ListView`的`layout_height`和`layout_width`设置为`match_parent`,如果设置为`match_parent`,一般`ListView`的宽高会测量三次以上。具体的源码没有深入研究。但为什么会要测量多次，如果对于自定义`View`稍微有点基础的会知道，对于`View`的测量大小有三个类型：
	- `UNSPECIFIED`:未指定的，父类不对子类施加任何限制。
	- `EXACTLY`:确定的，父类确定其子类控件的大小。
	- `AT_MOST`:最大值，需要子类去测量自身大小确定。
	
如果我们设置宽高为`wrap_content`，即`AT_MOST`,表示其宽高有控件本身去测量确定，而如果是`match_parent`，则`EXACTLY`，表示确定的大小。

#### `Adapter`优化。

对于Adapter的优化，主要包括以下几个步骤：

- 复用`convertView`,减少子布局的生成。

- 定义`ViewHolder`，减少`findViewById()`的次数。

下面看一下代码

```java 

/**
 * 最基础的adapter
 * Created by Alex_MaHao on 2016/5/17.
 */

public class SimpleBaseAdapter extends BaseAdapter {

    private List<String> datas;


    public SimpleBaseAdapter(List<String> datas) {
        this.datas = datas;
    }

    @Override
    public int getCount() {
        return datas==null?0:datas.size();
    }

    @Override
    public Object getItem(int position) {
        return null;
    }

    @Override
    public long getItemId(int position) {
        return 0;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {

        ViewHolder vHolder;

        if(convertView==null){
            //初始化item布局
            convertView = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_listview_sample,parent,false);

            //创建记事本类
            vHolder = new ViewHolder();

            //查找控件，保存控件的引用
            vHolder.tv = ((TextView) convertView.findViewById(R.id.listview_sample_tv));

            //将当前viewHolder与converView绑定
            convertView.setTag(vHolder);
        }else{
            //如果不为空，获取
            vHolder = (ViewHolder) convertView.getTag();
        }
        vHolder.tv.setText(datas.get(position));

        return convertView;
    }


    /**
     * 笔记本类，保存对象的引用
     */
    class ViewHolder{
        TextView tv;
    }
}

```

根据代码分析思路：
- `ListView`中的`item`滑出屏幕时，滑出的`item`会在`getView()`方法中返回，及`converView`参数。所以在这里判断，是否为null，如果不为null，表示我们可以对该布局重新设置数据并返回到列表中显示。
- `ViewHolder`，保存`item`中子控件的引用。因为我们复用了`converView`,那么对于同一个`converView`布局，其子控件的引用应该是不变的。所以我们可以获取到其上的所有子控件的引用并通过`ViewHolder`进行保存。
- `setTag()，getTag()`：该方法实现了`ViewHolder`和`converView`的绑定，就类似于通过`setTag()`方法，将`ViewHolder`打包成一个包裹，放在了`converView`上，再通过`getTag()`方法，将这个包裹取出。


> 在很多优化中，会将`ViewHolder`定义为`static`。但在这里我并没加上。因为经过测试，加或不加，`ViewHolder`的创建次数不变。网上查了很多资料，也没有找到一个让我信服的理由。唯一有点理的就是基于java的特性。静态内部类的对象不依赖于其所在的外部类对象。


### Adapter的封装--BaseAppAdapter

上一节说了`Adapter`的优化，但如果我们每次写都要写这么多的优化代码，这不符合程序员懒惰的天性。那我们只能把他封装，提取出一个公共的基类，在基类中，我们把布局加载，优化等都默认实现，只需让子类构造布局文件，以及绑定数据。

那么，从我们上一节的代码看，有以下模块都可以提取为基类：

- 数据集合datas：对于数据集合，我们通常都是一个`List`集合，在这里定义泛型来表述其所包含的内容。
- 数据优化：布局的复用以及`ViewHolder`与`converView`的绑定。
- `ViewHolder`：定义一个`ViewHolder`，通过`map`保存控件与id；

子类所需实现的：

- 数据的初始化
- 确定`item`的布局文件
- 将数据与视图绑定。


那么直接看一下我们继承好的代码：

```java 
public abstract  class BaseAppAdapter<T> extends BaseAdapter {

    /**
     * 泛型，保存数据
     */
    protected List<T> datas;

    /**
     * 构造方法，子类必须实现其构造方法，并初始化数据
     * @param datas
     */
    public BaseAppAdapter(List<T> datas) {
        this.datas = datas;
    }

    @Override
    public int getCount() {
        return datas==null?0:datas.size();
    }

    @Override
    public Object getItem(int position) {
        return null;
    }

    @Override
    public long getItemId(int position) {
        return 0;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {

        BaseViewHolder vHolder;

        if(convertView==null){
            convertView = LayoutInflater.from(parent.getContext()).inflate(getItemLayoutId(),parent,false);

            vHolder = new BaseViewHolder(convertView);

            convertView.setTag(vHolder);
        }else{
            vHolder = (BaseViewHolder) convertView.getTag();
        }

        /**
         * 数据绑定的回调
         */
        bindData(vHolder,datas.get(position));

        return convertView;
    }

    /**
     * 子类实现，获取item布局文件的id
     * @return
     */
    protected  abstract int getItemLayoutId();

    /**
     * 子类实现，绑定数据
     * @param vHolder  对应position的ViewHolder
     * @param data 对应的数据绑定
     */
    protected abstract void bindData(BaseViewHolder vHolder,T data);


    /**
     * ViewHolder类
     */
    class BaseViewHolder{

        /**
         * 保存view，以id-view的形式
         */
        Map<Integer,View> mapView;
        View rootView;

        public BaseViewHolder(View rootView){
            this.rootView = rootView;
            mapView = new HashMap<Integer,View>();
        }


        /**
         * 通过id查找控件
         * @param id
         * @return
         */
        public View getView(Integer id){
            View view = mapView.get(id);
            if(view==null){
                view = rootView.findViewById(id);
                mapView.put(id,view);
            }

            return view;
        }

    }

}

```

在以上代码中有几个关键点：

- 有参的构造方法：子类实现时，必须调用父类的构造方法，用以保存数据。
- `getItemLayoutId()`:`item`的布局文件id。抽象方法，子类必须实现。对于每一个`BaseAppAdapter`的子类，都需要通过该方法返回他们所特有的`item`的id。
- `bindData(BaseViewHolder vHolder,T data)`：数据绑定方法，子类必须实现。子类在此方法中将数据设置到试图上
- `BaseViewHolder`中的`getView(Integer id)`:该方法设计比较巧妙，因为，对于基类我们不知道有哪些`id`需要查询，如果直接通过`findViewById()`方法，则并没有减少查询次数。在这里，通过保存一个`map`对象，通过`id`先从`map`中取，如果不存在，说明是第一次获取，则使用`findViewById`查找控件，并将控件以键值对的形式保存到`map`中，则下次查找，会从`map`中取，减少了`findViewById`的次数。可能有人会认为，添加了一个`map`对象，内存占用不是增大了吗，但是，第一，我们`map`中保存的都是引用。第二，`findViewById()`是按照深度优先遍历查询的，如果不懂，遍历肯定能明白吧。

### ListView 分割线的高度和颜色设置。

简单的两个属性

- `android:divider`:设置颜色，背景
- ` android:dividerHeight`:设置高度

如果设置无分割线，可以设置`android:divider="@null"`。

> 该两个属性必须同时使用，如果只设置`divider`，则没有效果，同时默认的分割线也会消失。

当然，我们也可以在`item`布局文件中添加分割线（在底部添加一个线），麻烦点而已。


### 隐藏ListView 的滚动条

`android:scrollbars="none"`

### 取消ListView的点击反馈效果

对于`ListView`，在android5.0一下是一个变色，在android5.0以上是一个波纹动画。我们可以通过一些设置取消掉他的反馈效果。使用如下代码

` android:listSelector="#0000"`:点击颜色设置为透明