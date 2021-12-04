#### try-with-resource

作为java7的新增语法，可以声明一个或多个资源，try-with-resource确保在语句执行完毕后所有的资源都能得到关闭。

前提条件：该资源对象是AutoCloseable的子类。

#### 使用

java7之前操作流的语句：

```
    public void readFile() throws FileNotFoundException {
        FileReader fr = null;
        BufferedReader br = null;
        try{
            fr = new FileReader("d:/input.txt");
            br = new BufferedReader(fr);
            String s = "";
            while((s = br.readLine()) != null){
                System.out.println(s);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                br.close();
                fr.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

用于需要在finally中显示调用close方法，整个流程既繁琐又臃肿。

java7中使用try-with-resource操作流：

```
    public void readFileTryWithResouce() throws FileNotFoundException {

        try(
                FileReader fr = new FileReader("d:/input.txt");
                BufferedReader br = new BufferedReader(fr)
        ){
            String s = "";
            while((s = br.readLine()) != null){
                System.out.println(s);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

没有finally中模板化的代码，简洁明了，更为优雅的写法。

用户可以自定义类让其实现AutoCloseable接口，这样可以在try-with-resource结束时调用其close方法。

```
public class MyAutoClosable implements AutoCloseable {
    public void doIt() {
        System.out.println("MyAutoClosable doing it!");
    }

    @Override
    public void close() throws Exception {
        System.out.println("MyAutoClosable closed!");
    }

    public static void main(String[] args) {
        try(MyAutoClosable myAutoClosable = new MyAutoClosable()){
            myAutoClosable.doIt();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
}
```

结果显示：

```
MyAutoClosable doing it!
MyAutoClosable closed!
```

#### 参考文献

- [java try-with-resource语句使用](https://www.jianshu.com/p/258c5ce1a2bd)