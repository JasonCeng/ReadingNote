# 9.1 异常处理改进

## 目录
* <a href="#1-trywithresource语句">1 try-with-resource语句</a>



### 1 try-with-resource语句
* **一般格式：**
```java
打开一个资源
try{
  使用该资源
}
finally{
  关闭一个资源
}
```
其中资源所属的类必须实现了AutoCloseable接口。该接口只有一个方法：void close() throws Exception

 **注意：**Java中还有一个Closeable接口。它是AutoCloseable接口的一个子接口，也只有一个close方法，但该方法被声明为抛出一个IOException。

* **最简形式：**
```java
try(Resource res = ...){
  使用res
}
```
当try语句块退出时，会自动调用res.close()方法。

 **注意：**try-with-resource语句自己也可以含有catch和finally分支。他们都会在关闭资源之后执行。在实践中，不建议单个try语句放置太多逻辑代码。

### 2 忽略异常
* **假设一种情况：**当产生了一个IOException，接下来在关闭资源时，close方法又抛出了另一个异常。这个时候Java会如何处理呢？

* **Java 7以前的处理方法（不妥）：**在finally分支中抛出的异常会丢弃掉之前的异常。这不仅听上去很不合理，实际上也确实不合理。毕竟，用户对原始的异常会更感兴趣。

* **try-with-resource语句修正了这个行为：**当AutoCloseable对象的close方法抛出异常时，原来的异常（这里指IOException）会被重新抛出，而调用close方法产生的异常会被捕获，并被标注为“被忽略”异常。具体使用方法如下：

  当捕获到首次异常时，可以通过调用getSuppressed方法来获取那些二次异常：
```java
try{
  ...
}catch(IOException ex){
  Throwable[] secondaryExceptions = ex.getSuppressed();
}
```
如果不能使用try-with-resources语句，又希望自己实现这一的机制，你可以调用：```ex.addSuppressed(secondaryException);```

* **注意：**Throwable、Exception、RuntimeException和Error类的构造参数都可以接受两个参数，分别用来**禁用忽略异常**和**禁用堆栈跟踪**。

  1)当忽略异常被禁用时，调用addSuppressed就不会偶效果，并且getSuppressed方法会返回一个长度为0的数组。

  2)当禁用堆栈跟踪时，调用fillInStackTrace不会有效果，并且getStackTrace方法会返回一个长度为0的数组。着可以用于因内存不够而产生的VM错误，然后抛出一个与之前所忽略异常无关且完全不同的异常。

### 3 捕获多个异常
在Java SE 7中，你可以在同一个catch分支中捕获多个异常类型。

例如，假设处理丢失的文件和未知主机的行为一致，那么你可以将这里两个异常写在一个catch分支中：
```Java
try{
  可能会抛出异常的一段代码
}catch(FileNotFoundException | UnknownHostException ex){
  对丢失文件和未知主机等紧急情况的紧急处理
}catch(IOException ex){
  对其他所有I/O问题的紧急处理
}
```
**注意：**只要当捕获的异常均不是其他异常的子类时，才能用这种方法。

**优点：**

1）代码更简洁。

2）执行效率更高。因为生产的字节码会包含一个含有共享catch分支的代码块。
