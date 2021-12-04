### final关键字

final关键字在java中可以修饰变量、方法和类。

- 当final修饰变量时，该变量一经赋值则不能修改
- 当final修饰方法时，该方法不能被子类重写
- 当final修饰类时，该类不能被子类继承

#### final修饰变量

- 该变量可以在声明时赋值，或者在初始化代码块、构造方法中赋值

  例如：

  ```java
      final String str;
      
      {
          str = "hello final";
      }
  ```

  或者：

  ```
      final String str;
  
      public FinalTest() {
          str = "hello final";
      }
  ```

- 当该变量为引用时其引用的对象内部的属性值是可以重新赋值的

  ```
      final List<String> orderList = new ArrayList<>();
  
      public bugOrder() {
          orderList.add("order1");
          orderList.add("order2");
      }
  ```

  以list为例，虽然orderList为final，但是其仍旧可以add元素。

#### final修饰方法

> 使用*final*方法的原因有两个。第一个原因是把方法锁定，以防任何继承类修改它的含义；第二个原因是效率。 

​														---------来自《java编程思想》

这个应该就是final修饰的方法可以提高性能最权威的解释了。对于目前该说法已经不存在了。

#### final修饰类

主要由两种用途：

- 防止子类继承该类，比如String、Integer类
- 创建不可变类，这个在多线程环境下的线程安全得到了应用（考虑到文章的连贯性这个笔者会在之后的章节中详细介绍）

注意：

##### 匿名内部类来自外部闭包环境的自由变量必须是final

```
public interface MyInterface {
    void doSomething();
}
public class TryUsingAnonymousClass {
    public void useMyInterface() {
        final Integer number = 123;
        System.out.println(number);

        MyInterface myInterface = new MyInterface() {
            @Override
            public void doSomething() {
                System.out.println(number);
            }
        };
        myInterface.doSomething();

        System.out.println(number);
    }   
```

myInterface为匿名内部类的对象，调用doSomething方法时会依赖number变量，此时number需要声明为final（在java8中不声明也可以，详细《参考匿名内部类使用外部变量探究》）。

### 参考文献

- [Java代码优化 Java final 修饰类或者方法能提高性能？还50%？老铁，你试了吗？]()
- [请不要再说 Java 中 final 方法比非 final 性能更好了](https://cloud.tencent.com/developer/article/1338656)（这篇文章通过JMH测试和字节码分析比较了final和非final方法的区别比较有意思）

- 匿名内部类使用外部变量探究