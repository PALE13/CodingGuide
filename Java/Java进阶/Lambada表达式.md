## Lambada表达式


**面向对象的思想**

做一件事情，找一个能解决这个事情的对象，调用对象的方法，完成事情。

**函数式编程思想**

只要能获取到结果，谁去做的，怎么做的都不重要，重视的是结果，不重视过程



**冗余的Runnable代码**

当需要启动一个线程去完成任务时，通常会通过java.lang.Runnable接口来定义任务内容，并使用java.lang.Thread类来启动该线程。

代码如下：

```java
public class Demo01Runnable {
    public static void main(String[] args) {
        // 匿名内部类
        Runnable task = new Runnable() {
            @Override
            public void run() { // 覆盖重写抽象方法
                System.out.println("多线程任务执行！");
            }
        };
        new Thread(task).start(); // 启动线程
    }
}
```



本着一切皆对象的思想，这种做法是无可厚非的：首先创建一个Runnable接口的匿名内部类对象来指定任务内容，再将其交给一个线程来启动。

对于Runnable的匿名内部类用法，可以分析出几点内容：

Thread类需要Runnable接口作为参数，其中的抽象run方法是用来指定线程任务内容的核心；

为了指定run的方法体，不得不需要Runnable接口的实现类；

为了省去定义一个RunnableImpl实现类的麻烦，不得不使用匿名内部类；

必须覆盖重写抽象run方法，所以方法名称、方法参数、方法返回值不得不再写一遍，且不能写错；

而实际上，似乎只有方法体才是关键所在。

 

**编程思想转换**

我们真的希望创建一个匿名内部类对象吗？不，我们只是为了做这件事情而不得不创建一个对象。我们真正希望做的事情是：将run方法体内的代码传递给Thread类知晓。

传递一段代码，这才是我们真正的目的。而创建对象只是受限于面向对象语法而不得不采取的一种手段方式。那，有没有更加简单的办法？如果我们将关注点从“怎么做”回归到“做什么”的本质上，就会发现只要能够更好地达到目的，过程与形式其实并不重要。

 



#### **Lambda表达式的标准格式**

```
 (参数列表) -> {一些重写方法的代码};
```

解释说明格式：

()：接口中抽象方法的参数列表，没有参数就空着，有参数就写出参数，多个参数使用逗号分隔

->：传递的意思，把参数传递给方法体{}

{}：重写接口的抽象方法的方法体

 

**Lambda的使用前提**

Lambda的语法非常简洁，完全没有面向对象复杂的束缚。但是使用时有几个问题需要特别注意：

- 使用Lambda必须具有接口，**且要求接口中有且仅有一个抽象方法**。无论是JDK内置的Runnable、Comparator接口还是自定义的接口，只有当接口中的抽象方法存在且唯一时，才可以使用Lambda。
- 使用Lambda必须具**有上下文推断**。也就是方法的参数或局部变量类型必须为Lambda对应的接口类型，才能使用Lambda作为该接口的实例

备注：有且仅有一个抽象方法的接口，称为“**函数式接口**”。

```java
public class Demo02Lambda {
    public static void main(String[] args) {
        //使用匿名内部类的方式,实现多线程
        new Thread(new Runnable(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+" 新线程创建了");
            }
        }).start();

        //使用Lambda表达式,实现多线程
        new Thread(()->{
            System.out.println(Thread.currentThread().getName()+" 新线程创建了");
        }
        ).start();


        //优化省略Lambda
        new Thread(()->System.out.println(Thread.currentThread().getName()+" 新线程创建了")).start();
    }
}
```





##### **省略格式**

凡是根据上下文推导出来的内容，都可以省略书写

可以省略的内容:

- (参数列表): 括号中**参数列表的数据类型**可以省略不写
- (参数列表): 括号中的**参数只有一个**，**那么类型和()都可以省略**
- {一些代码}: 如果{}中的代码**只有一行**，无论是否有返回值，**都可以省略( {},return,分号 )**

注意：要省略{}，return，分号必须一起省略。



给定一个计算器Calculator接口，内含抽象方法calc可以将两个int数字相加得到和值

```
public interface Calculator {
    //定义一个计算两个int整数和的方法并返回结果
    public abstract int calc(int a,int b);
}
```

在下面的代码中，请使用Lambda的标准格式调用invokeCalc方法，完成120和130的相加计算

```java
public class Demo01Calculator {
    public static void main(String[] args) {
        //调用invokeCalc方法,方法的参数是一个接口,可以使用匿名内部类
        invokeCalc(10, 20, new Calculator() {
            @Override
            public int calc(int a, int b) {
                return a+b;
            }
        });

        //使用Lambda表达式简化匿名内部类的书写
        invokeCalc(120,130,(int a,int b)->{
            return a + b;
        });
        //优化省略Lambda
        invokeCalc(120,130,(a,b)-> a + b);
    }


    /*
        定义一个方法
        参数传递两个int类型的整数
        参数传递Calculator接口
        方法内部调用Calculator中的方法calc计算两个整数的和
     */
    public static void invokeCalc(int a,int b,Calculator c){
        int sum = c.calc(a,b);
        System.out.println(sum);
    }
}
```





#### **方法引用**

我们用Lambda表达式来实现匿名方法。但有些情况下，我们用Lambda表达式仅仅是调用一些已经存在的方法，除了调用动作外，没有其他任何多余的动作，在这种情况下，我们倾向于通过方法名来调用它，而Lambda表达式可以帮助我们实现这一要求，它使得Lambda在调用那些已经拥有方法名的方法的代码更简洁、更容易理解。

**方法引用可以理解为Lambda表达式的另外一种表现形式。方法引用可以根据方法自动推断参数的类型**



##### **实例方法的引用**

```java
/*
    定义一个打印的函数式接口
*/
        @FunctionalInterface
        public interface Printable {
            //定义字符串的抽象方法
            void print(String s);
        }
```



```java
public class Demo01Printable {
    //定义一个方法,参数传递Printable接口,对字符串进行打印
    public static void printString(Printable p) {
        p.print("HelloWorld");
    }

    public static void main(String[] args) {
        //调用printString方法,方法的参数Printable是一个函数式接口,所以可以传递Lambda
        printString((s) -> {
            System.out.println(s);
        });
/*
    分析:
        Lambda表达式的目的,打印参数传递的字符串
        把参数s,传递给了System.out对象,调用out对象中的方法println对字符串进行了输出
        注意:
            1.System.out对象是已经存在的
            2.println方法也是已经存在的
        所以我们可以使用方法引用来优化Lambda表达式
        可以使用System.out方法直接引用(调用)println方法
 */
        printString(System.out::println);
    }
}
```

例如上例中， System.out 对象中有一个重载的 println(String) 方法恰好就是我们所需要的。

那么对于 printString 方法的函数式接口参数，对比下面两种写法，完全等效。

**Lambda表达式写法：** 

```
s -> System.out.println(s); 
```

**方法引用写法**： 

```
System.out :: println；
```

第一种语义是指：拿到参数之后经Lambda之手，继而传递给 System.out.println 方法去处理

第二种等效写法的语义是指：直接让 System.out 中的 println 方法来取代Lambda

两种写法的执行效果完全一 样，而第二种方法引用的写法复用了已有方案，更加简洁。

**注：Lambda 中传递的参数一定是方法引用中的那个方法可以接收的类型，否则会抛出异常**





##### **静态方法的引用**

```
@FunctionalInterface
public interface Calcable {
    //定义一个抽象方法,传递一个整数,对整数进行绝对值计算并返回
    int calsAbs(int number);
}
```

静态方法引用要用类名

```java
class Demo01StaticMethodReference {
    //定义一个方法,方法的参数传递要计算绝对值的整数,和函数式接口Calcable
    public static int method(int number,Calcable c){
        return c.calsAbs(number);
    }

    public static void main(String[] args) {
        //调用method方法,传递计算绝对值得整数,和Lambda表达式
        int number = method(-10,(n)->{
            //对参数进行绝对值得计算并返回结果
            return Math.abs(n);
        });
        System.out.println(number);


/*
    使用方法引用优化Lambda表达式
    Math类是存在的
    abs计算绝对值的静态方法也是已经存在的
    所以我们可以直接通过类名引用静态方法
 */
        int number2 = method(-10, Math::abs);
        System.out.println(number2);
    }
}
```



##### **类的构造器引用**

```java
@FunctionalInterface
public interface PersonBuilder {
    //定义一个方法,根据传递的姓名,创建Person对象返回
    Person builderPerson(String name);
}

public class Person {
    private String name;
    public Person() {}
    public Person(String name) {this.name = name;}
    public String getName() {return name;}
    public void setName(String name) {this.name = name;}
}
```

构造方法new Person(String name) 已知，创建对象已知 new

就可以使用Person引用new创建对象

```java
public class Demo {
    //定义一个方法,参数传递姓名和PersonBuilder接口,方法中通过姓名创建Person对象
    public static void printName(String name,PersonBuilder pb){
        Person person = pb.builderPerson(name);
        System.out.println(person.getName());
    }

    public static void main(String[] args) {
        //调用printName方法,方法的参数PersonBuilder接口是一个函数式接口,可以传递Lambda
        printName("迪丽热巴",(String name)->{
            return new Person(name);
        });
/*
    使用方法引用优化Lambda表达式
    构造方法new Person(String name) 已知
    创建对象已知 new
    就可以使用Person引用new创建对象
 */
        printName("古力娜扎",Person::new);//使用Person类的带参构造方法,通过传递的姓名创建对象
    }
}
```



##### **数组的构造器引用**

```java
@FunctionalInterface
public interface ArrayBuilder {
    //定义一个创建int类型数组的方法,参数传递数组的长度,返回创建好的int类型数组
    int[] builderArray(int length);
}
```

已知创建的就是int[]数组，数组的长度也是已知的

可以使用方法引用int[] 引用new，根据参数传递的长度来创建数组

```java
public class Demo {
    /*
        定义一个方法
        方法的参数传递创建数组的长度和ArrayBuilder接口
        方法内部根据传递的长度使用ArrayBuilder中的方法创建数组并返回
     */
    public static int[] createArray(int length, ArrayBuilder ab){
        return  ab.builderArray(length);
    }


    public static void main(String[] args) {
        //调用createArray方法,传递数组的长度和Lambda表达式
        int[] arr1 = createArray(10,(len)->{
            //根据数组的长度,创建数组并返回
            return new int[len];
        });
        System.out.println(arr1.length);//10


/*
    使用方法引用优化Lambda表达式
    已知创建的就是int[]数组
    数组的长度也是已知的
    就可以使用方法引用
    int[]引用new,根据参数传递的长度来创建数组
 */
        int[] arr2 =createArray(10,int[]::new);
        System.out.println(Arrays.toString(arr2));
        System.out.println(arr2.length);//10
    }
}
```







