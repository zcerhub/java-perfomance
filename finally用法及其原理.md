### Finanlly

##### 类结构

![1639483284348](D:\学习\性能优化\assets\1639483284348.png)

Throwable作为所有异常和错误的公共父类。主要的子类有Error和Exception。Error表示严重的错误，需要程序立即停止下来，常见的Error为OutOfMemoryError、StackOverflowError。Exception表示可以恢复的错误，即异常，用户可以选择捕获或者抛出。可以细分为RuntimeException和非RuntimeException。RuntimeException可称为非检查异常，编译器不对该类异常检查，NullPointerException、ArrayIndexOutOfBoundsException为此类异常。非RuntimeException称为检查异常，编译器会对该类异常检查，IOException和ClassNotFoundException为此类异常。

#### Throwable源码

```java
public class Throwable implements Serializable {

    //更细节的信息
    private String detailMessage;
    
    //造成该异常的异常，比如：捕获A异常后，抛出B异常，那么B异常的cause可以设置为A异常
    private Throwable cause = this;    
    
    //无参构造，调用fillInStackTrace方法，fillInStackTrace为重点。
    public Throwable() {
        fillInStackTrace();
    }
    
    //为detailMessage赋值和调用fillInStackTrace
    public Throwable(String message) {
        fillInStackTrace();
        detailMessage = message;
    }
    
    //为detailMessage和cause赋值，调用fillInStackTrace
    public Throwable(String message, Throwable cause) {
        fillInStackTrace();
        detailMessage = message;
        this.cause = cause;
    }
    
    //jdk7新增的方法，根据writableStackTrace决定是否调用fillInStackTrace方法。为detailMessage和cause赋值
    protected Throwable(String message, Throwable cause,
                        boolean enableSuppression,
                        boolean writableStackTrace) {
        if (writableStackTrace) {
            fillInStackTrace();
        } else {
            stackTrace = null;
        }
        detailMessage = message;
        this.cause = cause;
        if (!enableSuppression)
            suppressedExceptions = null;
    }
    
    
    //老熟人了，打印异常时栈信息
    public void printStackTrace() {
        printStackTrace(System.err);
    }
    
    //将栈信息输出到指定stream中
    public void printStackTrace(PrintStream s) {
        printStackTrace(new WrappedPrintStream(s));
    }
    
    private void printStackTrace(PrintStreamOrWriter s) {
        // Guard against malicious overrides of Throwable.equals by
        // using a Set with identity equality semantics.
        Set<Throwable> dejaVu =
            Collections.newSetFromMap(new IdentityHashMap<Throwable, Boolean>());
        dejaVu.add(this);

        //并发安全
        synchronized (s.lock()) {
            //先打印异常类信息
            // Print our stack trace
            s.println(this);
            //打印栈中的信息
            StackTraceElement[] trace = getOurStackTrace();
            for (StackTraceElement traceElement : trace)
                s.println("\tat " + traceElement);

            //忽略调
            // Print suppressed exceptions, if any
            for (Throwable se : getSuppressed())
                se.printEnclosedStackTrace(s, trace, SUPPRESSED_CAPTION, "\t", dejaVu);

            //打印引起该异常的异常信息
            // Print cause, if any
            Throwable ourCause = getCause();
            if (ourCause != null)
                ourCause.printEnclosedStackTrace(s, trace, CAUSE_CAPTION, "", dejaVu);
        }
    }
    
    //爬栈方法，调用native方法获得栈信息。很耗时的方法，之后会介绍对其的优化方法
    public synchronized Throwable fillInStackTrace() {
        if (stackTrace != null ||
            backtrace != null /* Out of protocol state */ ) {
            fillInStackTrace(0);
            stackTrace = UNASSIGNED_STACK;
        }
        return this;
    }
}    
```

#### demo展示1

通过该demo学习Throwable中detailMessage和cause参数的使用。

```java
public class ExceptonDemo {

    public void f() {
        throw new RuntimeException("f方法中抛出异常");
    }

    public void g() {
        try {
            f();
        }catch (Exception e) {
            throw new RuntimeException("g方法中抛出异常", e);
        }
    }

    public static void main(String[] args) {
        ExceptonDemo demo = new ExceptonDemo();
        demo.g();
    }

}
```

结果展示：

```
Exception in thread "main" java.lang.RuntimeException: g方法中抛出异常
	at com.example.concurrent.finaly.ExceptonDemo.g(ExceptonDemo.java:13)
	at com.example.concurrent.finaly.ExceptonDemo.main(ExceptonDemo.java:19)
Caused by: java.lang.RuntimeException: f方法中抛出异常
	at com.example.concurrent.finaly.ExceptonDemo.f(ExceptonDemo.java:6)
	at com.example.concurrent.finaly.ExceptonDemo.g(ExceptonDemo.java:11)
	... 1 more
```

该异常为应用开发中较为普遍。异常的发生具有级联效应，f方法中的异常导致g方法发生异常。打印异常时先打印没有捕获异常处的栈结构，然后再打印可能触发该异常的其它异常信息。

#### demo2展示2

Throwable类中通过fillInStackTrace方法获得当前栈的信息。

```
public class TestPrintStackTrace4 {

    public static void f() throws Exception{
        throw new Exception("出问题啦！");
    }
    public static void g() throws Exception{
        try {
            f();
        }catch(Exception e) {
            e.printStackTrace();
            //先注释该句
//            throw (Exception)e.fillInStackTrace();
        }

    }

    public static void main(String[] args) {
        try {
            g();
        }catch(Exception e) {
            e.printStackTrace();
        }
    }

}
```

结果展示：

```
java.lang.Exception: 出问题啦！
	at com.example.concurrent.finaly.TestPrintStackTrace4.f(TestPrintStackTrace4.java:6)
	at com.example.concurrent.finaly.TestPrintStackTrace4.g(TestPrintStackTrace4.java:10)
	at com.example.concurrent.finaly.TestPrintStackTrace4.main(TestPrintStackTrace4.java:21)
```

在g方法处抛出异常但是异常栈顶为f方法，原因是f方法中创建Exception对象时会调用fillInStackTrace方法保存栈的信息，而此时栈顶是f方法。如果想让栈顶为g方法。打开注释即可，此时的结果为：

```
java.lang.Exception: 出问题啦！
	at com.example.concurrent.finaly.TestPrintStackTrace4.f(TestPrintStackTrace4.java:6)
	at com.example.concurrent.finaly.TestPrintStackTrace4.g(TestPrintStackTrace4.java:10)
	at com.example.concurrent.finaly.TestPrintStackTrace4.main(TestPrintStackTrace4.java:21)
java.lang.Exception: 出问题啦！
	at com.example.concurrent.finaly.TestPrintStackTrace4.g(TestPrintStackTrace4.java:14)
	at com.example.concurrent.finaly.TestPrintStackTrace4.main(TestPrintStackTrace4.java:21)
```

#### finally的性能问题

fillInStackTrace爬栈比较耗时（参考文献1中R大的回答）。解决方案有两种：

- 覆盖fillInStackTrace方法直接返回this

  ```java
      public synchronized Throwable  fillInStackTrace() {
          return this;
      }
  ```

- jdk7之后可以利用爬栈开关writableStackTrace

  ```
      protected Throwable(String message, Throwable cause,
                          boolean enableSuppression,
                          boolean writableStackTrace) {
          if (writableStackTrace) {
              fillInStackTrace();
          } else {
              stackTrace = null;
          }
          detailMessage = message;
          this.cause = cause;
          if (!enableSuppression)
              suppressedExceptions = null;
      }
  ```

  writableStackTrace设置false后创建Throwable对象不再爬栈。

#### 参考文献

- [重载 Throwable.fillInStackTrace() 方法以提高Java性能这样的做法对吗？](https://www.zhihu.com/question/21405047)

