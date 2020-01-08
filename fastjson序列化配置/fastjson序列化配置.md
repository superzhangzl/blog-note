# fastjson序列化配置

fastjson是我们常用的json序列化工具。当需要使用`JSON.toJSONString(obj)`将对象转为json时，若对象中出现Date时，会转为时间戳的形式，还需要前端做一次转换。同时其中的null元素会直接忽略掉。通过使用带参数的转换方法来解决这个问题，下面仅作备份以供后面使用。

```java
JSON.toJSONStringWithDateFormat(obj, "yyyy-MM-dd HH:mm:ss",
                SerializerFeature.DisableCircularReferenceDetect,
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullListAsEmpty);
```



#### 测试：

当前测fastjson试版本：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.46</version>
</dependency>
```

如下的class：

```java
class Info {
    // get/set省略
    private String name;
    private Integer age;
    private Date birthday;
    private List<String> friends;
    private Food food;
 }
```

```java
class Food {
    private String name;
    private Integer Calories;
    private Info info;
}
```



测试方法：

```java
public static void main(String[] args) {
        Info info = new Info();
        info.name = "Tom";
        info.birthday = new Date();
        //= 普通
        String json1 = JSON.toJSONString(info);
        System.out.println(json1);
        //= 格式化时间
        String json2 = JSON.toJSONStringWithDateFormat(info, "yyyy-MM-dd HH:mm:ss");
        System.out.println(json2);
        //= 设置null元素
        String json3 = JSON.toJSONStringWithDateFormat(info, "yyyy-MM-dd HH:mm:ss",
                SerializerFeature.WriteMapNullValue);
        System.out.println(json3);
        //= 设置空数组
        String json4 = JSON.toJSONStringWithDateFormat(info, "yyyy-MM-dd HH:mm:ss",
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullListAsEmpty);
        System.out.println(json4);
        //= 同上
        Food food = new Food();
        food.name = "potato";
        food.info = info;
        info.food = food;

        String json5 = JSON.toJSONStringWithDateFormat(info, "yyyy-MM-dd HH:mm:ss",
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullListAsEmpty);
        System.out.println(json5);
        //= 禁止循环引用，会报堆栈错误
        String json6 = JSON.toJSONStringWithDateFormat(info, "yyyy-MM-dd HH:mm:ss",
                SerializerFeature.DisableCircularReferenceDetect,
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullListAsEmpty);
        System.out.println(json6);
}
```



以下为输出结果，最后一条由于有相互引用的情况，会导致堆溢出。在增加了配置后，null元素未被过滤，null数组被转为空数组。

```
{"birthday":1578471143152,"name":"Tom"}
{"birthday":"2020-01-08 16:12:23","friends":[],"name":"Tom"}
{"age":null,"birthday":"2020-01-08 16:12:23","food":null,"friends":[],"name":"Tom"}
{"age":null,"birthday":"2020-01-08 16:12:23","food":null,"friends":[],"name":"Tom"}
{"age":null,"birthday":"2020-01-08 16:12:23","food":{"calories":null,"info":{"$ref":".."},"name":"potato"},"friends":[],"name":"Tom"}
Exception in thread "main" java.lang.StackOverflowError
```





- [使用fastjson时出现$ref: "$.list[2]"的解决办法（重复引用）](https://www.jianshu.com/p/6041242405e8)

