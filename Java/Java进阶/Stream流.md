## **Stream流**

Stream（流）是一个来自数据源的元素队列

元素是特定类型的对象，形成一个队列。 Java中的Stream并不会存储元素，而是按需计算。 数据源流的来源，可以是集合，数组等。

和以前的Collection操作不同， Stream操作还有两个基础的特征：

**Pipelining**：中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格（fluent style）。 这样做可以对操作进行优化， 比如延迟执行(laziness)和短路( short-circuiting)。 

**内部迭代**： 以前对集合遍历都是通过Iterator或者增强for的方式，显式的在集合外部进行迭代，这叫做外部迭代。 Stream提供了内部迭代的方式，流可以直接调用遍历方法。

当使用一个流的时候，通常包括三个基本步骤：**获取一个数据源（source）→ 数据转换→执行操作获取想要的结果，**每次转换原有 Stream 对象不改变，**返回一个新的 Stream 对象**（可以有多次转换），这就允许对其操作可以像链条一样排列，变成一个管道。

 

**循环遍历的弊端**

```java
public class Demo01List {
  public static void main(String[] args) {
    //创建一个List集合,存储姓名
    List<String> list = new ArrayList<>();
    list.add("张无忌");
    list.add("周芷若");
    list.add("赵敏");
    list.add("张强");
    list.add("张三丰");


    //对list集合中的元素进行过滤,只要以张开头的元素,存储到一个新的集合中
    List<String> listA = new ArrayList<>();
    for(String s : list){
      if(s.startsWith("张")){
        listA.add(s);
      }
    }


    //对listA集合进行过滤,只要姓名长度为3的人,存储到一个新集合中
    List<String> listB = new ArrayList<>();
    for (String s : listA) {
      if(s.length()==3){
	       listB.add(s);
      }
    }

    //遍历listB集合
    for (String s : listB) {
      System.out.println(s);
    }
  }
}
```

这段代码中含有三个循环，每一个作用不同：

- 首先筛选所有姓张的人； 
- 然后筛选名字有三个字的人；
- 最后进行对结果进行打印输出。

每当我们需要对集合中的元素进行操作的时候，总是需要进行循环、循环、再循环。这是理所当然的么？不是。循环是做事情的方式，而不是目的。另一方面，使用线性循环就意味着只能遍历一次。如果希望再次遍历，只能再使用另一个循环从头开始

 

**Stream的更优写法**

```java
public class Demo02Stream {
  public static void main(String[] args) {
    //创建一个List集合,存储姓名
     List<String> list = new ArrayList<>();
    list.add("张无忌");
    list.add("周芷若");
    list.add("赵敏");
    list.add("张强");
    list.add("张三丰");
    //对list集合中的元素进行过滤,只要以张开头的元素,存储到一个新的集合中
    //对listA集合进行过滤,只要姓名长度为3的人,存储到一个新集合中
    //遍历listB集合
    list.stream().filter(name->name.startsWith("张"))
    			.filter(name->name.length()==3)
				.forEach(name-> System.out.println(name));

 	}
}
```

**![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image002.png)**

 

这张图中展示了过滤、映射、跳过、计数等多步操作，这是一种集合元素的处理方案，而方案就是一种“函数模 型”。图中的每一个方框都是一个“流”，调用指定的方法，可以从一个流模型转换为另一个流模型。而最右侧的数字 3是最终结果

这里的 filter 、 map 、 skip 都是在对函数模型进行操作，集合元素并没有真正被处理。只有当终结方法 count 执行的时候，整个模型才会按照指定策略执行操作。而这得益于Lambda的延迟执行特性

 

#### **获取流**

**java.util.stream.Stream<T>** 是Java 8新加入的最常用的流接口。（这并不是一个函数式接口）

**获取一个流的方法**

![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image004.png)

 

**default Stream< E> stream()：**所有的Collection集合都可以通过stream默认方法获取流

```java
    //把单列集合转换为Stream流
    List<String> list = new ArrayList<>();
    Stream<String> stream1 = list.stream();
    Set<String> set = new HashSet<>();
    Stream<String> stream2 = set.stream();
```

 

**static < T> Stream< T> stream(T[] array)：**Arrays中的静态方法，获取数组的流

```java
    //使用Arrays静态方法获取数组的流
    Integer[] arr = {1,2,3,4,5};
    Stream<Integer> stream3 = Arrays.stream(arr);
```

 

**static < T> Stream< T> of(T... values)：**参数是一个可变参数，可以传入不同类型的参数，也可以传递一个数组

注意：传递的数组必须是引用数据类型，不能是基本数据类型，如果传递基本数据类型的数据，会把整个数组当作一个元素传入

```java
    //使用Sream.of把数组转换为Stream流
    Stream<Integer> stream4 = Stream.of(1, 2, 3, 4, 5);

    //可变参数可以传递数组，但必须是引用诗句类型的数组
    Integer[] arr = {1,2,3,4,5};
    Stream<Integer> stream5 = Stream.of(arr);

     //传入基本数据类型数组，出现错误
     int[] a = {1,2,3,4,5};
     Stream.of(a).forEach(s-> System.out.println(s));//[I@7ef20235
```

 

**双列集合获取流**

```java
    Map<String,String> map = new HashMap<>();
    //获取键,存储到一个Set集合中，然后转换为流
    Set<String> keySet = map.keySet();
    Stream<String> stream6 = keySet.stream();

    //获取值,存储到一个Collection集合中，然后转换为流
    Collection<String> values = map.values();
    Stream<String> stream7 = values.stream();

    //获取键值对(键与值的映射关系 entrySet)
    Set<Map.Entry<String, String>> entries = map.entrySet();
    Stream<Map.Entry<String, String>> stream8 = entries.stream();
```

 

 

**常用方法**

这些方法可以被分成两种： 延迟方法，终结方法



**#### 延迟方法**

返回值类型仍然是 Stream 接口自身类型的方法，因此支持链式调用。![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image006.png)

 

##### **过滤：filter**

可以通过 filter 方法将一个流转换成另一个子集流。

**Stream< T> filter(Predicate<? super T> predicate)：**该接口接收一个 Predicate 函数式接口参数（可以是一个Lambda或方法引用）作为筛选条件。

**Predicate接口：**此前我们已经学习过 java.util.stream.Predicate 函数式接口，其中唯一的抽象方法为：

**boolean test(T t)：**该方法将会产生一个boolean值结果，代表指定的条件是否满足。如果结果为true，那么Stream流的 filter 方法 将会留用元素；如果结果为false，那么 filter 方法将会舍弃元素。

 

```java
     Stream<String> stream = Stream.of("张三丰", "张翠山", "赵敏", "周芷若", "张无忌");
     
    //对Stream流中的元素进行过滤,只要姓张的人
//    Stream<String> stream2 = stream.filter(new Predicate<String>() {
//      @Override
//      public boolean test(String s) {
//        return s.startsWith("张");
//      }
//    });
//
//    //遍历stream2流
//    stream2.forEach(name-> System.out.println(name));

 
    /*
      Stream流属于管道流,只能被消费(使用)一次
      第一个Stream流调用完毕方法,数据就会流转到下一个Stream上
      而这时第一个Stream流已经使用完毕,就会关闭了
      所以第一个Stream流就不能再调用方法了
      IllegalStateException: stream has already been operated upon or closed

     */
    //stream.forEach(name-> System.out.println(name));
    //使用lamda表达式简化
    stream.filter(s -> s.startsWith("张")).forEach(s -> System.out.println(s));
```

 

##### **取用前几个：limit** 

延迟方法 

**Stream< T> limit(long maxSize)：**可以对流进行截取，只取用前n个。参数是一个long型，如果集合当前长度大于参数则进行截取；否则不进行操作

```java
    //获取一个Stream流
    String[] arr = {"美羊羊","喜洋洋","懒洋洋","灰太狼","红太狼"};
    Stream<String> stream = Stream.of(arr);
    //使用limit对Stream流中的元素进行截取,只要前3个元素
    Stream<String> stream2 = stream.limit(3);
    //遍历stream2流
    stream2.forEach(name-> System.out.println(name));
```

 

##### **跳过前几个：skip** 

如果希望跳过前几个元素，可以使用 skip 方法获取一个截取之后的新流

**Stream< T > skip(long n)：**如果流的当前长度大于n，则跳过前n个；否则将会得到一个长度为0的空流

```java
    //获取一个Stream流
    String[] arr = {"美羊羊","喜洋洋","懒洋洋","灰太狼","红太狼"};
    Stream<String> stream = Stream.of(arr);
    //使用skip方法跳过前3个元素
    Stream<String> stream2 = stream.skip(3);
    //遍历stream2流
    stream2.forEach(name-> System.out.println(name));
```

 

##### **去重：distinct**

元素去重，依赖（hashCode和equals方法）

**Stream < T > distinct()**

```java
String[] arr = {"美羊羊","喜洋洋","喜洋洋","懒洋洋","灰太狼","红太狼"};
Stream<String> stream = Stream.of(arr);
stream.distinct().forEach(s-> System.out.println(s));
```

 

##### **组合：concat** 

如果有两个流，希望合并成为一个流，那么可以使用 Stream 接口的静态方法 concat 

**static < T> Stream< T> concat(Stream<? extends T> a, Stream<? extends T> b)：**合并两个流，需要保证两个流的类型一致

备注：这是一个静态方法，与 java.lang.String 当中的 concat 方法是不同的

 

```java
    //创建一个Stream流
    Stream<String> stream1 = Stream.of("张三丰", "张翠山", "赵敏", "周芷若", "张无忌");
    //获取一个Stream流
    String[] arr = {"美羊羊","喜洋洋","懒洋洋","灰太狼","红太狼"};
    Stream<String> stream2 = Stream.of(arr);
    //把以上两个流组合为一个流，需要保证
    Stream<String> concat = Stream.concat(stream1, stream2);
    //遍历concat流
    concat.forEach(name-> System.out.println(name));
```

 

##### **映射：map** 

如果需要将流中的元素映射到另一个流中，可以使用 map 方法。

**< R> Stream< R> map(Function<? super T, ? extends R> mapper)：**

该接口需要一个 Function 函数式接口参数，可以将当前流中的T类型数据转换为另一种R类型的流，使用了map方法后，流中的数据就变成了return后的数据。

**Function接口 ：** java.util.stream.Function 函数式接口

其中唯一的抽象方法为：

**R apply(T t)：**将T类型转换为R类型，这可以将一种T类型转换成为R类型，而这种转换的动作，就称为映射

注意：只能转换引用数据类型

```java
    //获取一个String类型的Stream流
    Stream< String> stream = Stream.of("张无忌-15", "周芷若-14", "赵敏-13", "张三丰-16");

//    //使用map方法,把字符串类型的整数,转换(映射)为Integer类型的整数
//    Stream<Integer> stream2 = stream.map(new Function<String, Integer>() {
//      @Override
//      public Integer apply(String s){
//        String[] arr = s.split("-");
//        String ageStr = arr[1];;
//        return Integer.parseInt(ageStr);
//      }
//    });
//    //遍历流
//    //在map方法执行后，流上的数据就变成的整数
//    stream2.forEach(i-> System.out.println(i));

 
    //简化写法
    stream.map(s-> Integer.parseInt(s.split("-")[1])).forEach(s-> System.out.println(s));      		//15 14 13 16
```

 

 

##### **mapToInt()方法**

**IntStream mapToInt(ToIntFunction<? super T> mapper)**

将流中的元素映射为 `IntStream`，它是基本数据类型 `int` 的流。

ToIntFunction接口中唯一的抽象方法

```
new ToIntFunction<Integer>() {
    @Override
    public int applyAsInt(Integer value) {
        return value.intValue();
    }
```



toArray()无法返回基本类型的数组，以int类型为例，需要借助mapToInt()将流中数据转换为int数据类型，然后用空参的toArray()收集为int类型的数组

因为流中数据都已经为基本数据类型int，所以并不会向上转型为Object数组

```
//List<Integer> 转int类型数组
//map使用Integer中的intValue方法
int[] arr = list.stream().mapToInt(new ToIntFunction<Integer>() {
  @Override
  public int applyAsInt(Integer value) {
    return value.intValue();
  }
}).toArray();


//lamda表达式简化
int[] arr2 = list.stream().mapToInt(Integer::intValue).toArray();
```

 



#### **终结方法**

![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image008.png)

 



##### **逐一处理：forEach**

终结方法

**void forEach(Consumer<? super T> action)：**该方法接收一个 Consumer 接口函数，会将每一个流元素交给该函数进行处理

**Consumer接口：**java.util.function.Consumer<T>接口是一个消费型接口。

Consumer接口中包含抽象方法：

**void accept(T t)**，意为消费一个指定泛型的数据。

 

```java
    //获取一个Stream流
    Stream<String> stream = Stream.of("张三", "李四", "王五", "赵六", "田七");
    //使用Stream流中的方法forEach对Stream流中的数据进行遍历
//    stream.forEach(new Consumer<String>() {
//      @Override
//      public void accept(String s) {
//        System.out.println(s);
//      }
//    });

    stream.forEach(name->System.out.println(name));
    //stream.forEach(System.out::println); //方法引用
```

 

##### **统计个数：count** 

**long count()：**正如旧集合 Collection 当中的 size 方法一样，流提供 count 方法来数一数其中的元素个数：

```java
    Stream<String> original = Stream.of("张无忌", "张三丰", "周芷若");
    Stream<String> result = original.filter(s->s.startsWith("张"));
    System.out.println(result.count()); // 2
```

 

 

##### **收集方法：toArray()**

收集流的数据，放到数组中

**Object[] toArray()：**返回一个Object类型的数组

```java
     Stream<String> stream = Stream.of("张三", "李四", "王五", "赵六", "田七");
     //使用无参方法，放到Object类型数组当中
//    Object[] arr1 = stream.toArray();
//    System.out.println(Arrays.toString(arr1)); //[张三, 李四, 王五, 赵六, 田七]
```



 **int[] toArray()：**若流中数据为基本数据类型，则直接返回该类型的数组

```java
int[] array = Stream.of(1, 2, 3, 4, 5).mapToInt(Integer::intValue).toArray();
```



**< A > A[] toArray(IntFunction<A[]> generator)：**返回指定类型的数组

toArray方法参数的作用：创建一个指定类型的数组

toArray方法的底层：依次获取流的每一个数据，放入数组中

toArray方法的返回值：指定类型的数组，大小与apply方法的参数value保持一致

IntFunction接口中的抽象方法

```java
public Object[] apply(int value) {
  return new Object[0];
}
```

其中value为流中元素的个数

需要返回什么类型的数组，就return什么类型的数组

**注意：**该方法只能return引用类型的数组，不能return基本类型的数组

```java
     Stream<String> stream = Stream.of("张三", "李四", "王五", "赵六", "田七");

     //使用无参方法，放到Object类型数组当中
//    Object[] arr1 = stream.toArray();
//    System.out.println(Arrays.toString(arr1)); //[张三, 李四, 王五, 赵六, 田七]


    //放到String类型数组当中
    //IntFunction的泛型：要转换的具体类型的数组
    //apply的形参：流中数据的个数，要跟数组长度保持一致
    //apply的返回值：具体类型的数组
 
//    String[] arr = stream.toArray(new IntFunction<String[]>() {
//      @Override
//      public String[] apply(int value) {
//        return new String[value];
//      }
//    });
//

//    System.out.println(Arrays.toString(arr)); //[张三, 李四, 王五, 赵六, 田七]


    //lamda表达式简化
    String[] arr2 = stream.toArray(value -> new String[value]);
    System.out.println(Arrays.toString(arr2));
```

 



##### **收集方法：collect**

收集流的元素，放到集合中

**collect(Collectors.toList())：**收集到List中

**collect(Collectors.toSet())：**收集到Set中

```java
    ArrayList<String> list = new ArrayList<>();
    Collections.addAll(list,"张无忌-男-15","张无忌-男-15","周芷若-女-14","赵敏-女-13","张强-男-20","张三丰-男-100");

    //需求：把所有男性收集起来
    //收集到list集合当中
    List<String> newList = list.stream().filter(s -> "男".equals(s.split("-")[1])).collect(Collectors.toList());
    System.out.println(newList); //[张无忌-男-15, 张无忌-男-15, 张强-男-20, 张三丰-男-100]

    //收集到set集合当中
    Set<String> newList2 = list.stream().filter(s -> "男".equals(s.split("-")[1])).collect(Collectors.toSet());
    System.out.println(newList2); //[张强-男-20, 张三丰-男-100, 张无忌-男-15]
```

 

**int[]数组转集合List< Integer>**

```java
List< Integer> list = Arrays.stream(数组名).boxed().collect(Collectors.toList());
```

**集合List< Integer>转int[]数组**

```java
int[] res = 集合名.stream().mapToInt(Integer::valueOf).toArray();
```

 



**collect(Collectors.toMap(Function<? super T, ? extends K> keyMapper,**

​                **Function<? super T, ? extends U> valueMapper) ）：**收集到Map中

需要重写两次Function接口中的抽象方法apply

第一次重写确定键的生成规则

第二次重写确定值的生成规则

**R apply(T t)：**将T类型转换为R类型

例子：

收集性别为男的姓名和年龄到Map<String, Integer>中

```java
 	    ArrayList<String> list2 = new ArrayList<>();
        Collections.addAll(list2, "张无忌-男-15", "周芷若-女-14", "赵敏-女-13", "张强-男-20", "张三丰-男-100");

        //收集到map集合当中
        //将键：姓名，值：年龄的收集到map中
        Map<String, Integer> newMap = list2.stream().
                filter(s -> "男".equals(s.split("-")[1])).
                collect(Collectors.toMap(new Function<String, String>() {
                    @Override
                    public String apply(String s) {
                        return s.split("-")[0];
                    }
                }, new Function<String, Integer>() {
                    @Override
                    public Integer apply(String s) {
                        return Integer.parseInt(s.split("-")[2]);
                    }
                }));

        System.out.println(newMap); //{张强=20, 张三丰=100, 张无忌=15}


        //使用lamda表达式简化
        Map<String, Integer> newMap2 = list2.stream().
                filter(s -> "男".equals(s.split("-")[1])).
                collect(Collectors.toMap(s1 -> s1.split("-")[0],
                        s2 -> Integer.parseInt(s2.split("-")[2])))
                        
        System.out.println(newMap2);//{张强=20, 张三丰=100, 张无忌=15}
```

 

 

 