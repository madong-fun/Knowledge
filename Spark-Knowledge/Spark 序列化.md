## 序列化含义：
序列化就是指将一个对象转化为二进制的byte流（注意，不是bit流），然后以文件的方式进行保存或通过网络传输，等待被反序列化读取出来。序列化常被用于数据存取和通信过程中。
### java应用序列化一般方法：
- class实现序列化操作是让class 实现Serializable接口，但实现该接口不保证该class一定可以序列化，因为序列化必须保证该class引用的所有属性可以序列化。
- static和transient修饰的变量不会被序列化，可以让不能序列化的引用用static和transient来修饰。（static修饰的是类的状态，而不是对象状态，所以不存在序列化问题。transient修饰的变量，是不会被序列化到文件中，在被反序列化后，transient变量的值被设为初始值，如int是0，对象是null）
- 还可以实现readObject()方法和writeObject()方法来自定义实现序列化

## Spark哪些操作需要序列化
1. 代码中对象在driver本地序列化 
2. 对象序列化后传输到远程executor节点(广播变量也需要序列化)
3. 远程executor节点反序列化对象 
4. 最终远程节点执行 

故对象在执行中需要序列化通过网络传输，则必须经过序列化过程。

## Spark 序列化
1、在默认情况下，Spark采用Java的ObjectOutputStream序列化一个对象。该方式适用于所有实现了java.io.Serializable的类。通过继承java.io.Externalizable，你能进一步控制序列化的性能。Java序列化非常灵活，但是速度较慢，在某些情况下序列化的结果也比较大。

![737BDB54-1764-43EE-AB53-F90B4FF093D3.png](resources/0A64938AF300D1A14B057499C936A5F9.jpg =944x408)

2、Spark也能使用Kryo序列化对象。Kryo不但速度极快，而且产生的结果更为紧凑（通常能提高10倍）。Kryo的缺点是不支持所有类型，为了更好的性能，你需要提前注册程序中所使用的类（class），Kryo默认注册了scala Tuple类型。
![D78DAA88-C06A-455F-96C0-2C299C478465.png](resources/5DAC0C4C0AA28D002A2C4DAF73BFC7B4.jpg =1342x603)

3、UnsafeRowSerializer,为序列化UnsafeRow而生。基于Java的unsafe包来实现（Tungsten的功能），对Row中每个字段的操作都转换为字节的操作，换句话说它底层实际存储结构是byte[]，而且支持Kryo序列化，相比使用Java序列化工具来序列化数组/Row之类的复杂数据结构，它的性能肯定要好很多。此序列化方式涉及Tungsten，可参阅
`https://blog.csdn.net/zy_zhengyang/article/details/79040583`

## Spark 中使用Kryo序列化
```
// 配置序列化方式
config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
// 注册序列化class
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
// 设置kryo 缓存大小，默认64kb，最大不能超过2048mb
config("spark.kryoserializer.buffer","64")
```
![595E0CF6-64E6-40B9-9091-506ECE934750.png](resources/423A9249D81E8CA02CA4A2A679A2CC3A.jpg =1100x430)

## SerializerManager