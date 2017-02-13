## RecyclerView 封装Adapter之BaseRecyclerAdapter 

> 转载请标明出处：
[http://blog.csdn.net/lisdye2/article/details/52049857](http://blog.csdn.net/lisdye2/article/details/52049857)
本文出自:[【Alex_MaHao的博客】](http://blog.csdn.net/lisdye2?viewmode=contents)
项目中的源码已经共享到github，有需要者请移步[【Alex_MaHao的github】](https://github.com/AlexSmille/alex_mahao_sample/tree/master/systemwidgetdemo/src/main/java/com/mahao/alex/systemwidgetdemo/recycleView/base)


### **封装BaseRecyclerAdapter **

对于`ListView`，提到最多的便是性能优化，及封装`ViewHolder`和重用`View`，Android推出`RecyclerView`用以替代`ListView`,`RecyclerView`的优点便是替我们做了性能优化，不过在实际使用中，为了减少代码量，往往封装一个`Adapter`的基类，以达到复用的目的。


### **理论分析**

首先，从一个最基本的`Adapter`来说，不确定的只有四个个地方

- 具体数据源的类型。

在`Adapter`中，往往需要保存一个数据的`List`集合，以便获取数据，对不同的`item`条目进行设置，而对于数据源，我们可以明确他是一个`List`集合（一般来说），此时我们可以定义泛型以表示具体的数据源类型。

- `itemView`的布局文件。

因为每一个`RecyclerView`所对应的条目的布局文件都不一样，我们可以通过构造方法让子类传过来。但我选择采用抽象方法的方式，使子类实现抽象方法，返回具体的布局。

- `ViewHolder`类，具体视图上的控件，用以和数据进行绑定和显示。

虽然，`RecyclerView`帮我们做了性能优化，但是我们仍然需要编写`ViewHolder`类继承`RecyclerView.ViewHolder`，如果我们每写一个`Adapter`都需要定义一个`ViewHolder`，那么代码量将会增加很多，我们可以通过在`ViewHolder`中定义一个`Map`集合，保存具体的`id`和`View`,代表键值对象。

- 将数据源和视图绑定。

对于具体的数据绑定，我们只需要具体的某一个数据和对应的条目数，此时通过定义抽象方法，交给子类去实现。

### **代码实现**

根据上面的分析，进行对应的实现。

**首先是数据源**

通过泛型定义数据的类型，通过构造方法让子类传入。

```java 
  public abstract class BaseRecycleAdapter<T> extends RecyclerView.Adapter<BaseRecycleAdapter.BaseViewHolder> {

	// 数据源
    protected List<T> datas;
	// 构造方法，传入
    public BaseRecycleAdapter(List<T> datas) {
        this.datas = datas;
    }
}
```


**定义基类的`ViewHolder`**

通过`Map`保存基本的键值对。

```java 
  /**
     * 封装ViewHolder ,子类可以直接使用
     */
    public class BaseViewHolder extends  RecyclerView.ViewHolder{


        private Map<Integer, View> mViewMap;

        public BaseViewHolder(View itemView) {
            super(itemView);
            mViewMap = new HashMap<>();
        }

        /**
         * 获取设置的view
         * @param id
         * @return
         */
        public View getView(int id) {
            View view = mViewMap.get(id);
            if (view == null) {
                view = itemView.findViewById(id);
                mViewMap.put(id, view);
            }
            return view;
        }
    }

```

从代码层次上分析，在`BaseViewHolder`中，定义成员变量`mViewMap`,从其类型声明上一个`Integer`和`View`可以看出，分别保存`id`和对应的控件。

关键的逻辑在`getView()`方法中，传入对应布局的`id`，如果该`id`已经存在在`mViewMap`中，则直接获取，如果不存在，则先通过`findViewById()`方法，从`itemView`方法中查找对应的`View`,存入到`mViewMap`中，并返回。


**具体条目的布局id**


如果有点`RecyclerView`基础的，可以知道子实现`Adapter`时，需要实现的方法有三个，分别是：

- `onCreateViewHolder()` : 返回具体的`ViewHolder`对象。

- `onBindViewHolder()`, 绑定数据源和视图。

- `getItemCount()`： 返回具体的条目数量。

对于`getItemCount()`方法，比较好实现，直接返回数据源的大小即可。

```java 
 @Override
    public int getItemCount() {
        
        return datas==null?0:datas.size();
    }

```

`onBindViewHolder()`方法，在绑定数据源时在说明其实现。


`onCreateViewHolder()`:返回具体的`ViewHolder`对象，我们在上面已经定义了`BaseViewHolder`对象，现在只需要创建他即可。不过我们缺少对应的条目的布局`id`。所以定义抽象方法，交给子类实现，如下方式

```java 

   /**
     * 获取子item
     * @return
     */
    public abstract int getLayoutId();


``` 

同时类声明上也要添加`abstract`标识符。

这样，我们只需要在`onCreateViewHolder()`中构造`BaseViewHolder`对象。

```java

    @Override
    public BaseViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        View view = LayoutInflater.from(parent.getContext()).inflate(getLayoutId(),parent,false);
        return new BaseViewHolder(view);
    }

 ```


**绑定数据源**

前面铺垫了这么久，就是为了这最后一步进行铺垫的。在之前的分析中，我们绑定数据源，只需要明确具体的数据和具体的控件即可。那么我们定义抽象方法

```java 
   /**
     *  绑定数据
     * @param holder  具体的viewHolder
     * @param position  对应的索引
     */
    protected abstract void bindData(BaseViewHolder holder, int position);


```

在这里，可能有一种更好的方式，及如下定义


```java 
    /**
     *  绑定的数据
     * @param holder 具体的ViewHolder
     * @param data  具体的数据
     */
    protected abstract void bindData(BaseViewHolder holder, T data);

```

不过，我采用了第一种方式，具体的原因我也忘了~~~根据个人的需要吧

此时`onBindViewHolder()`方法中如下实现


```java 
  @Override
    public void onBindViewHolder(BaseRecycleAdapter.BaseViewHolder holder, final int position) {
		// 子类实现数据绑定
         bindData(holder,position);
        
    }


```

比较认真的同学可能会发现，直接不实现此方法，交由子类实现不就好了吗。理论上确实是这样的，不过因为后面会添加头部和底部的布局，所以才有此一茬。如果感觉不爽的可以使用第二种方式。


**添加刷新和添加数据的方法**

```java 
 /**
     * 刷新数据
     * @param datas
     */
    public void refresh(List<T> datas){
        this.datas.clear();
        this.datas.addAll(datas);
        notifyDataSetChanged();
    }


    /**
     * 添加数据
     * @param datas
     */
    public void addData(List<T> datas){
        this.datas.addAll(datas);
        notifyDataSetChanged();
    }

```

### **使用方式**

前期写了这么多，到底能不能简化我们后续的编程呢，看看实现吧。


**编写类**，继承`extends BaseRecycleAdapter<Person>`

```java 
public class PersonAdapter extends BaseRecycleAdapter<Person> {

}

```

此时会报错，我们直接Alt +　回车，两次之后，自动生成了如下代码

```java 

public class PersonAdapter extends BaseRecycleAdapter<Person> {


    public PersonAdapter(List<Person> datas) {
        super(datas);
    }

    @Override
    protected void bindData(BaseViewHolder holder, int position) {

    }

    @Override
    public int getLayoutId() {
        return 0;
    }
}

```

**指定布局文件**

```java 

  @Override
    public int getLayoutId() {
		// 条目的布局id
        return R.layout.item;
    }

```

**绑定护具**


```java 
  @Override
    protected void bindData(BaseViewHolder holder, int position) {

        // 两个参数，具体的控件id ,  对应的参数数据
        holder.getView(R.id.name,datas.get(position).getName);
    }

```


### 完整的封装

```java 
public abstract class BaseRecycleAdapter<T> extends RecyclerView.Adapter<BaseRecycleAdapter.BaseViewHolder> {


    protected List<T> datas;

    public BaseRecycleAdapter(List<T> datas) {
        this.datas = datas;
    }

    @Override
    public BaseViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        View view = LayoutInflater.from(parent.getContext()).inflate(getLayoutId(),parent,false);
        return new BaseViewHolder(view);
    }

    @Override
    public void onBindViewHolder(BaseRecycleAdapter.BaseViewHolder holder, final int position) {

        bindData(holder,position);
        
    }


    /**
     * 刷新数据
     * @param datas
     */
    public void refresh(List<T> datas){
        this.datas.clear();
        this.datas.addAll(datas);
        notifyDataSetChanged();
    }


    /**
     * 添加数据
     * @param datas
     */
    public void addData(List<T> datas){
        this.datas.addAll(datas);
        notifyDataSetChanged();
    }

    /**
     *  绑定数据
     * @param holder  具体的viewHolder
     * @param position  对应的索引
     */
    protected abstract void bindData(BaseViewHolder holder, int position);



    @Override
    public int getItemCount() {
     
        return datas==null?0:datas.size();
    }




    /**
     * 封装ViewHolder ,子类可以直接使用
     */
    public class BaseViewHolder extends  RecyclerView.ViewHolder{


        private Map<Integer, View> mViewMap;

        public BaseViewHolder(View itemView) {
            super(itemView);
            mViewMap = new HashMap<>();
        }

        /**
         * 获取设置的view
         * @param id
         * @return
         */
        public View getView(int id) {
            View view = mViewMap.get(id);
            if (view == null) {
                view = itemView.findViewById(id);
                mViewMap.put(id, view);
            }
            return view;
        }
    }

    /**
     * 获取子item
     * @return
     */
    public abstract int getLayoutId();



    /**
     * 设置文本属性
     * @param view
     * @param text
     */
    public void setItemText(View view,String text){
        if(view instanceof TextView){
            ((TextView) view).setText(text);
        }
    }
}


```