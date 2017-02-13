
在使用`hibernate`做持久层框架时，我们可以定义一些属性是懒加载。甚至于一对多的关系，我们可以默认让多的一方暂时不加载对象，及定义`Lazy`属性，这样在不需要这些属性的时候，我们优化查询性能。

> 有这样一种需求，一篇博客下有多篇评论，定义评论属性是`Lazy`，只查询博客信息，并以JSON格式返回。


实现起来很简单，查询一下对象，通过`fastJson`将其转化成字符串返回就可以了，但是在转化的时候却出现了问题。

首先`fastJson`的原理是利用反射的机制，循环获取对象的属性，层层深入的。那么避免不了的去获取每一个对象，在获取每一个对象的时候就触发了`hibernate`的懒加载机制，`fastjson`遍历一个对象，`hibernate`就查询一个对象，然后就完蛋了~~~~。


怎么解决这个问题呢，好多人从数据库下手，其实我们可以从`fastJson`序列化对象时下手，在序列化时`JSON.toString(obj,SerializeFilter)`,其中第二个参数是过滤对象，如果要加载的对象时`hibernate`代理对象，且该对象还未加载，则跳过该字段，不序列化。


具体实现如下：

```java 

public class SimplePropertyFilter implements PropertyFilter {
	
	 /**
	   * 过滤不需要被序列化的属性，主要是应用于Hibernate的代理和管理。
	   * @param object 属性所在的对象
	   * @param name 属性名
	   * @param value 属性值
	   * @return 返回false属性将被忽略，ture属性将被保留
	   */
	  @Override
	  public boolean apply(Object object, String name, Object value) {
	    if (value instanceof HibernateProxy) {//hibernate代理对象
	      LazyInitializer initializer = ((HibernateProxy) value).getHibernateLazyInitializer();
	      if (initializer.isUninitialized()) {
	        return false;
	      }
	    } else if (value instanceof PersistentCollection) {//实体关联集合一对多等
	      PersistentCollection collection = (PersistentCollection) value;
	      if (!collection.wasInitialized()) {
	        return false;
	      }
	      Object val = collection.getValue();
	      if (val == null) {
	        return false;
	      }
	    }
	    
	    return true;
	  }

}

```


