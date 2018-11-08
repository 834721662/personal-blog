 [TOC]

###KafkaProducer
[官方文档](http://kafka.apache.org/0110/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)

这里描述的是Java版本的客户端，所有的代码针对的都是0.10

**当前的内容主要通过从源代码上描述KafkaProducer的运行与机制**
---

KafkaProducer构造函数

```java
    // 客户端ID：clientId
    private String clientId;
    // 分区器Partitioner实例partitioner
    private final Partitioner partitioner;
    // 最大请求大小maxRequestSize
    private final int maxRequestSize;
    // 内存总计大小totalMemorySize
    private final long totalMemorySize;
    // 集群元数据Metadata实例metadata
    private final Metadata metadata;
    // 记录收集器RecordAccumulator实例accumulator
    private final RecordAccumulator accumulator;
    // 后台发送线程Sender实例sender
    private final Sender sender;
    // 指标度量
    private final Metrics metrics;
    // io线程ioThread
    private final Thread ioThread;
    // 压缩类型
    private final CompressionType compressionType;
    // 这个应该是错误发送起？ 不了解
    private final Sensor errors;
    // 时间器
    private final Time time;
    // key序列化器keySerializer
    private final Serializer<K> keySerializer;
    // value序列化器valueSerializer
    private final Serializer<V> valueSerializer;
    // Producer配置信息ProducerConfig实例producerConfig
    private final ProducerConfig producerConfig;
    // 最大阻塞时间maxBlockTimeMs
    private final long maxBlockTimeMs;
    // 请求超时时间requestTimeoutMs
    private final int requestTimeoutMs;
```




官方的样例

kafka客户端发布record(消息)到kafka集群。
新的生产者是线程安全的，在线程之间共享单个生产者实例，通常单例比多个实例要快。
简单的例子，使用producer发送一个有序的key/value(键值对)，放到java的main方法里就能直接运行，
```java
Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("acks", "all");
 props.put("retries", 0);
 props.put("batch.size", 16384);
 props.put("linger.ms", 1);
 props.put("buffer.memory", 33554432);
 props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
 props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

 Producer<String, String> producer = new KafkaProducer<>(props);
 for(int i = 0; i < 100; i++)
     producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i)));

 producer.close();
```
生产者的缓冲空间池保留尚未发送到服务器的消息，后台I/O线程负责将这些消息转换成请求发送到集群。
如果使用后不关闭生产者，会泄露资源。
send()方法是异步的，添加消息到缓冲区等待发送，并立即返回。生产者将单个的消息批量在一起发送来提高效率。
acks:生产者需要server端在接收到消息后，进行反馈确认的尺度，主要用于消息的可靠性传输；
acks=0表示生产者不需要来自server的确认；acks=1表示server端将消息保存后即可发送ack，而不必等到其他follower角色的都收到了该消息；acks=all(or acks=-1)意味着server端将等待所有的副本都被接收后才发送确认。
retries，如果请求失败，生产者会自动重试，我们指定是0次，如果启用重试，则会有重复消息的可能性。
producer(生产者)缓存每个分区未发送消息。缓存的大小是通过 batch.size 配置指定的。值较大的话将会产生更大的批。并需要更多的内存（因为每个“活跃”的分区都有1个缓冲区）。
默认缓冲可立即发送，即遍缓冲空间还没有满，但是，如果你想减少请求的数量，可以设置linger.ms大于0。这将指示生产者发送请求之前等待一段时间，希望更多的消息填补到未满的批中。这类似于TCP的算法，例如上面的代码段，可能100条消息在一个请求发送，因为我们设置了linger(逗留)时间为1毫秒，然后，如果我们没有填满缓冲区，这个设置将增加1毫秒的延迟请求以等待更多的消息。需要注意的是，在高负载下，相近的时间一般也会组成批，即使是 linger.ms=0。在不处于高负载的情况下，如果设置比0大，以少量的延迟代价换取更少的，更有效的请求。
buffer.memory 控制生产者可用的缓存总量，如果消息发送速度比其传输到服务器的快，将会耗尽这个缓存空间。当缓存空间耗尽，其他发送调用将被阻塞，阻塞时间的阈值通过max.block.ms设定，之后它将抛出一个TimeoutException。
key.serializer和value.serializer示例，将用户提供的key和value对象ProducerRecord转换成字节，你可以使用附带的ByteArraySerializaer或StringSerializer处理简单的string或byte类型。

---



send()
```java
public Future<RecordMetadata> send(ProducerRecord<K,V> record,Callback callback)
```
异步发送一条消息到topic，并调用callback（当发送已确认）。
send是异步的，并且一旦消息被保存在等待发送的消息缓存中，此方法就立即返回。这样并行发送多条消息而不阻塞去等待每一条消息的响应。
发送的结果是一个RecordMetadata，它指定了消息发送的分区，分配的offset和消息的时间戳。如果topic使用的是CreateTime，则使用用户提供的时间戳或发送的时间（如果用户没有指定指定消息的时间戳）如果topic使用的是LogAppendTime，则追加消息时，时间戳是broker的本地时间。
由于send调用是异步的，它将为分配消息的此消息的RecordMetadata返回一个Future。如果future调用get()，则将阻塞，直到相关请求完成并返回该消息的metadata，或抛出发送异常。
如果要模拟一个简单的阻塞调用，你可以调用get()方法。


send方法负责将缓冲池中的消息异步的发送到broker的指定topic中。异步发送是指，方法将消息存储到底层待发送的I/O缓存后，将立即返回，这可以实现并行无阻塞的发送更多消息。send方法的返回值是RecordMetadata类型，它含有消息将被投递的partition信息，该条消息的offset，以及时间戳。 
因为send返回的是Future对象，因此在该对象上调用get()方法将阻塞，直到相关的发送请求完成并返回元数据信息；或者在发送时抛出异常而退出。
```java
 byte[] key = "key".getBytes();
 byte[] value = "value".getBytes();
 ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("my-topic", key, value)
 producer.send(record).get();
```
完全无阻塞的话,可以利用回调参数提供的请求完成时将调用的回调通知。
```java
 ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", key, value);
 producer.send(myRecord,
               new Callback() {
                   public void onCompletion(RecordMetadata metadata, Exception e) {
                       if(e != null)
                           e.printStackTrace();
                       System.out.println("The offset of the record we just sent is: " + metadata.offset());
                   }
               });
```
发送到同一个分区的消息回调保证按一定的顺序执行，也就是说，在下面的例子中 callback1 保证执行 callback2 之前：
```java
producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key1, value1), callback1);
producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key2, value2), callback2);
```
注意：callback一般在生产者的I/O线程中执行，所以是相当的快的，否则将延迟其他的线程的消息发送。如果你需要执行阻塞或计算昂贵（消耗）的回调，建议在callback主体中使用自己的Executor来并行处理。



关于 producer.flush()方法：
调用该方法将使得缓冲区的所有消息被立即发送（即使linger.ms参数被设置为大于0），且会阻塞直到这些相关消息的发送请求完成。flush方法的前置条件是：之前发送的所有消息请求已经完成。一个请求被视为完成是指：根据acks参数配置项收到了相应的确认，或者发送中抛出异常失败了。


---


关于Future


Future 范型接口 ==> Future<V> ,v指代的是执行的返回类型，具体的接口方法解释：
```java
boolean cancel (boolean mayInterruptIfRunning) 取消任务的执行。参数指定是否立即中断任务执行，或者等等任务结束
boolean isCancelled () 任务是否已经取消，任务正常完成前将其取消，则返回 true
boolean isDone () 任务是否已经完成。需要注意的是如果任务正常终止、异常或取消，都将返回true
V get () throws InterruptedException, ExecutionException  等待任务执行结束，然后获得V类型的结果。InterruptedException 线程被中断异常， ExecutionException任务执行异常，如果任务被取消，还会抛出CancellationException
V get (long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException 同上面的get功能一样，多了设置超时时间。参数timeout指定超时时间，uint指定时间的单位，在枚举类TimeUnit中有相关的定义。如果计算超时，将抛出TimeoutException
```
Future的实现类有java.util.concurrent.FutureTask<V>即 javax.swing.SwingWorker<T,V>。通常使用FutureTask来处理我们的任务。FutureTask类同时又实现了Runnable接口，所以可以直接提交给Executor执行。使用FutureTask实现超时执行的代码如下：
```java
ExecutorService executor = Executors.newSingleThreadExecutor();  
FutureTask<String> future =  
    new FutureTask<String>(new Callable<String>() {//使用Callable接口作为构造参数  
     public String call() {  
      //真正的任务在这里执行，这里的返回值类型为String，可以为任意类型  
    }});  
executor.execute(future);  
//在这里可以做别的任何事情  
try {  
  result = future.get(5000, TimeUnit.MILLISECONDS); //取得结果，同时设置超时执行时间为5秒。同样可以用future.get()，不设置执行超时时间取得结果 
//注意get方法将阻塞，知道抛出异常或者是得到返回结果. 
} catch (InterruptedException e) {  
  futureTask.cancel(true);  
} catch (ExecutionException e) {  
  futureTask.cancel(true);  
} catch (TimeoutException e) {  
  futureTask.cancel(true);  
} finally {  
  executor.shutdown();  
} 
```


```java
ExecutorService executor = Executors.newSingleThreadExecutor(); 
FutureTask<String> future = 
    new FutureTask<String>(new Callable<String>() {//使用Callable接口作为构造参数 
     public String call() { 
      //真正的任务在这里执行，这里的返回值类型为String，可以为任意类型 
    }}); 
executor.execute(future); 
//在这里可以做别的任何事情 
try { 
  result = future.get(5000, TimeUnit.MILLISECONDS); //取得结果，同时设置超时执行时间为5秒。同样可以用future.get()，不设置执行超时时间取得结果 
} catch (InterruptedException e) { 
  futureTask.cancel(true); 
} catch (ExecutionException e) { 
  futureTask.cancel(true); 
} catch (TimeoutException e) { 
  futureTask.cancel(true); 
} finally { 
  executor.shutdown(); 
} 
```
不直接构造Future对象，也可以使用ExecutorService.submit方法来获得Future对象，submit方法即支持以 Callable接口类型，也支持Runnable接口作为参数.
```java
ExecutorService executor = Executors.newSingleThreadExecutor();  
FutureTask<String> future =　executor.submit(  
  new Callable<String>() {//使用Callable接口作为构造参数  
    public String call() {  
   //真正的任务在这里执行，这里的返回值类型为String，可以为任意类型  
  }});  
//在这里可以做别的任何事情  
```


```java
ExecutorService executor = Executors.newSingleThreadExecutor(); 
FutureTask<String> future =　executor.submit( 
  new Callable<String>() {//使用Callable接口作为构造参数 
    public String call() { 
   //真正的任务在这里执行，这里的返回值类型为String，可以为任意类型 
  }}); 
//在这里可以做别的任何事情 
```

