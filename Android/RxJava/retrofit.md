## RxJava前奏之Retrofit2.0的学习

### 什么是Retrofit
因为RxJava是基于响应式编程的链式调用，需要具备支持RxJava的网络请求框架。其中Retrofit提供了这样的支持。

Retorfit其实基于okhttp封装的。okhttp会在以后学习，总结。此次暂时放一放。

Retrofit网络请求框架，使用起来可以分为三个部分，

- 网络请求回返数据的实体类
- 管理请求的服务类
- 调用我们的服务类


### 举个栗子
我们看一个例子：百度api请求身份证信息的例子

url：http://apis.baidu.com/apistore/idservice/id

请求参数两个，Get方式
- 请求参数(header) :apikey：用户账号的key  
- 请求参数(urlParam) :id ： 身份证号

返回参数
```java 

{
    "errNum": 0,
    "retMsg": "success",
    "retData": {
        "sex": "M", //M-男，F-女，N-未知
        "birthday": "1987-04-20", //出生日期
        "address": "湖北省孝感市汉川市" //身份证归属地 市/县
    }
}
```
在使用Retrofit之前，我们需要导入几个包。

```
    compile 'com.squareup.okhttp3:okhttp:3.2.0'
    compile 'com.squareup.retrofit2:retrofit:2.0.1'
    compile 'com.google.code.gson:gson:2.6.2'
    compile 'com.squareup.retrofit2:converter-gson:2.0.1'
```
因为Retrofti是基于okhttp，所以我们引入这两个包

gson以及converter-gson是为了在Retrofit中引入gson所必须的包。


#### 我们需要根据我们的json数据创建实体类
```java
public class IDBean {


    /**
     * errNum : 0
     * retMsg : success
     * retData : {"sex":"M","birthday":"1987-04-20","address":"湖北省孝感市汉川市"}
     */

    private int errNum;
    private String retMsg;
    /**
     * sex : M
     * birthday : 1987-04-20
     * address : 湖北省孝感市汉川市
     */

    private RetDataEntity retData;

    public int getErrNum() {
        return errNum;
    }

    public void setErrNum(int errNum) {
        this.errNum = errNum;
    }

    public String getRetMsg() {
        return retMsg;
    }

    public void setRetMsg(String retMsg) {
        this.retMsg = retMsg;
    }

    public RetDataEntity getRetData() {
        return retData;
    }

    public void setRetData(RetDataEntity retData) {
        this.retData = retData;
    }

    public static class RetDataEntity {
        private String sex;
        private String birthday;
        private String address;

        public String getSex() {
            return sex;
        }

        public void setSex(String sex) {
            this.sex = sex;
        }

        public String getBirthday() {
            return birthday;
        }

        public void setBirthday(String birthday) {
            this.birthday = birthday;
        }

        public String getAddress() {
            return address;
        }

        public void setAddress(String address) {
            this.address = address;
        }
    }
}


```

在这里推荐一款插件GsonFormat，它是AndroidStudio的一款插件，通过他我们可以直接生成对应json的实体类

#### 创建服务类接口
```java 
public interface IDService {
    
    @GET("/apistore/idservice/id")
    Call<IDBean> getResult(@Header("apikey") String apikey, @Query("id") String id);

}
```
@GET表示请求方式为GET请求，其内为取出根目录的地址。

该方法有两个参数，非别为在Http Header的参数以及地址后面拼接的参数，参数前面的注解中的字符串对应请求参数的参数名，最后返回一个Call对象。关于参数的分析，后面说。

#### 创建调用实例
```java
private static final String BASE_URL = "http://apis.baidu.com";

    private static final String API_KEY = "ea32ae1b4aed196338c19b65349e488c";
    
    private void query(){
        //1,创建Retrofit对象
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)//请求的目录，会与我们定义的服务方法的注解中的地址进行拼接
                .addConverterFactory(GsonConverterFactory.create()) //加入Gson解析
                .build();


        //2，创建服务的接口对象
        IDService idService = retrofit.create(IDService.class);
        Call<IDBean> call = idService.getResult(API_KEY, et_id.getText().toString().trim());

        //3，开启异步任务
        call.enqueue(new Callback<IDBean>() {
            @Override
            public void onResponse(Call<IDBean> call, Response<IDBean> response) {
                //4.处理结果
                if (response.isSuccessful()){
                    IDBean result = response.body();
                    if (result != null){
                        IDBean.RetDataEntity entity = result.getRetData();

                        Toast.makeText(IDActivity.this, entity.getAddress(), Toast.LENGTH_SHORT).show();
                    }
                }
            }

            @Override
            public void onFailure(Call<IDBean> call, Throwable t) {
                Toast.makeText(IDActivity.this, "fail", Toast.LENGTH_SHORT).show();
            }
        });


    }
```

调用总共分为四个步骤，注释写的很行处，但有一点需要注意，在我们解析失败的时候，onResponse也会调用，不过其response.body()方法也会被调用，只不过返回值为空，所以我们需要加空判断。

如果我们访问的页面为404等，则 response.errorBody().string()会返回错误。其中`response.isSuccessful()`当返回值为`return code >= 200 && code < 300;`;

call.cancel()可以取消网络请求


### 服务类接口中注解的使用

对于方法的注解对应我们HTTP中的七中网络请求，但在这里直说我们最常用的@GET 和@POST

#### GET 请求

- GET 的无参请求

```java 
 	@GET("/apistore/idservice/id")
    Call<IDBean> getResult();
```

- GET 的有参请求

```java 
 @GET("/apistore/idservice/id")
    Call<IDBean> getResult(@Header("apikey") String apikey, @Query("id") String id);
```

在参数中分别有两个不同的注解:

@Header表示该参数为Header中进行传参,参数名为注解中的"apikey"为字段名，参数为值。

@Query 表示为在地址后面拼接的参数，其中"id"为字段名，参数为传递的值。

- GET请求路径中有可变参数

```java

    @GET("/apistore/idservice/id/{userid}")
    Call<IDBean> getResult(@Path("userid") String userid);
```
其中@GET路径中，通过{userid}来创建一个可变字段，在参数中通过注解@Path，则该参数在实际调用中，会找到路径中的{userid}并把参数替换。


#### POST

POST的无参与路径的用法和GET的用法基本相同，把@GET改为@POST,即可。

- POST 带参请求

```java 
  @POST("/apistore/idservice/id")
    Call<IDBean> getResult(@Field("id") String id)
```

唯一区别是@Filed代替了@Query

- POST 带参请求的另一种方法
```java 
  @POST("/apistore/idservice/id")
    Call<IDBean> getResult(@FieldMap Map<String,String> map);
```

他的好处时，无需输入过多的参数，直接传入一个map对象，键是参数名，值是参数的值。


####  @FormUrlEncoded

在方法中加入此注解，则请求的参数会按照url编码进行传输