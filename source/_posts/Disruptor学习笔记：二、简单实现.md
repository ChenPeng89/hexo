---
title: 'Disruptor学习笔记:二、简单实现'
date: 2016-06-06 16:17:47
tags: Disruptor
---

## 前言
接下来实现一个简单的producer和consume，讲一个Long值从producer传递给consumer，并在consumer中打印出来。

## 实现代码
首先，定义一个数据传递的载体，Event。
```Java
public class LongEvent {
    private long value;

    public void set(long value)
    {
        this.value = value;
    }

    @Override
    public String toString() {
        return "LongEvent{" +
                "value=" + value +
                '}';
    }
}
```
为了实现Disruptor的events的预分配，我们创建一个EventFactory来创建Event。
```Java
public class LongEventFactory implements EventFactory<LongEvent> {

    @Override
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```
一旦我们定义了Event，那么肯定要有consumer来消费它。
```Java
public class LongEventHandler implements EventHandler<LongEvent> {
    @Override
    public void onEvent(LongEvent event, long sequence, boolean endOfBatch)
    {
        System.out.println("Event: " + event);
    }
}
```
接下来需要创建Producer来发布Event。
```Java
public class Producer {
    public static RingBuffer<LongEvent> ringBuffer;
    public static  Disruptor<LongEvent> disruptor;

    public void produceEvent(){
        EventFactory<LongEvent> eventFactory = new LongEventFactory();
        ExecutorService executor = Executors.newSingleThreadExecutor();
        int ringBufferSize = 1024 * 1024; // RingBuffer 大小，必须是 2 的 N 次方；
        disruptor = new Disruptor<LongEvent>(eventFactory , ringBufferSize , executor , ProducerType.SINGLE , new YieldingWaitStrategy());
        EventHandler<LongEvent> eventHandler = new LongEventHandler();
        disruptor.handleEventsWith(eventHandler);
        disruptor.start();
        ringBuffer = disruptor.getRingBuffer();
    }
}
```
发布Event。
```Java
public class Publisher {
    public void publish(){
        Producer producer = new Producer();
        producer.provideService();
        RingBuffer<LongEvent> ringBuffer = Producer.ringBuffer;
        long sequence = ringBuffer.next(); //请求下一个事件序号
        try{
            LongEvent event = ringBuffer.get(sequence);
            long data = 1000L;
            event.set(data);
            ringBuffer.publish(sequence);
        }finally {
            Producer.disruptor.shutdown();//关闭 disruptor，方法会堵塞，直至所有的事件都得到处理；
//            Producer.executor.shutdown();//关闭 disruptor 使用的线程池；如果需要的话，必须手动关闭， disruptor 在 shutdown 时不会自动关闭；
        }
    }

    public static void main(String[] args) {
        Publisher publisher = new Publisher();
        publisher.publish();
    }
}
```
运行main方法，接下来就可以看到运行结果了。
`Event: LongEvent{value=1000}`

明显可以看出，相比于使用简单的队列，事件发布机制更多地参与了进来。这是由于我们希望的Event预分配内存。最低要求通过两个阶段来发布信息，先调用ring buffer的数据槽，再发布可用的数据。同时有必要将Event的发布封装到try/finally 中。如果我们调用RingBuffer中的数据槽（calling RingBuffer.next()），那么接下来我们必须发布这个sequence。如果不这么做的话，会导致Disruptor状态异常。特别是，当使用multi-producer时，它将会使consumer阻塞，并且只能通过重启来恢复。

## Version 3 的Translators实现方式
在3.0版本，Disruptor提供了一个丰富的Lambda风格的api，来帮助开发人员封装RingBuffer里面复杂的逻辑。所以，3.0以后的版本推荐通过 Event Publisher/Event Translator来发布Event。

```Java
public class LongEventProducerWithTranslator {
    private final RingBuffer<LongEvent> ringBuffer;

    public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    private static final EventTranslatorOneArg<LongEvent , Long> TRANSLATOR = new EventTranslatorOneArg<LongEvent,Long>(){

        @Override
        public void translateTo(LongEvent longEvent, long l, Long data) {
            longEvent.set(data);
        }
    };
    public void onData(Long data){
        ringBuffer.publishEvent(TRANSLATOR , data);
    }
}
```
使用这种方法的优势在于Translator的代码可以单独做成一个类，容易进行单元测试。Disruptor为此提供了多个接口(EventTranslator, EventTranslatorOneArg, EventTranslatorTwoArg 等)。

接下来，发布Event。
```Java
public class LongEventMain {

    public static void main(String[] args) {
        //Executor用来为consumer创建新的线程
        Executor executor = Executors.newCachedThreadPool();

        LongEventFactory factory = new LongEventFactory();

        //指定ringBuffer的大小，必须为2的倍数
        int bufferSize = 1024 * 1024;

        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(factory ,bufferSize , executor);

        disruptor.handleEventsWith(new LongEventHandler());

        //启动Disruptor，启动所有线程
        disruptor.start();

        //从Disruptor获取用来发布的ringBuffer
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
        producer.onData(1000L);

    }
}
```
## 单一Producer和多Producer
提高并发系统性能的最好的方式是采用单一作家原则，Disruptor已经应用了它。如果你的Disruptor系统中只有一个Producer线程，那么你可以利用这点获得额外的性能。

## 等待策略
Disruptor默认的等待策略是BlockingWaitStrategy，BlockingWaitStrategy使用锁和条件变量来控制线程的唤醒。Disruptor还提供其它等待策略。

### SleepingWaitStrategy
在多次循环尝试不成功后，选择让出CPU，等待下次调度，多次调度后仍不成功，尝试前睡眠一个纳秒级别的时间再尝试。这种策略平衡了延迟和CPU资源占用，但延迟不均匀。

### YieldingWaitStrategy
在多次循环尝试不成功后，选择让出CPU，等待下次调。平衡了延迟和CPU资源占用，但延迟也比较均匀。

### BusySpinWaitStrategy 
自旋等待，类似Linux Kernel使用的自旋锁。低延迟但同时对CPU资源的占用也多。

## Disruptor DSL
Disruptor提供了一个简单的DSL风格的API来简化Event handler开发。

### 并行的Event Handler
首先创建和ringbuffer相关的配置。
```Java
Disruptor<ValueEvent> disruptor =
  new Disruptor<ValueEvent>(ValueEvent.EVENT_FACTORY, EXECUTOR,
                            new SingleThreadedClaimStrategy(RING_SIZE),
                            new SleepingWaitStrategy());
```
然后我们将它传入一个Executor实例，并在它的线程内执行event handler。
然后我们添加event handler来并行处理Event handler。
```Java
disruptor.handleEventsWith(handler1, handler2, handler3, handler4);
```
最后，开启event handler线程，并且获得配置好的RingBuffer。
```Java
RingBuffer<ValueEvent> ringBuffer = disruptor.start();
```
接下来Producer就可以使用RingBuffer的nextEvent并将events放入ringbuffer。

### 有依赖关系的Event handler
```Java
disruptor.handleEventsWith(handler1).then(handler2, handler3, handler4);
```
handler1必须先执行，handler2,3,4会并行。
也可以实现多个handler链。
```Java
disruptor.handleEventsWith(handler1).then(handler2);
disruptor.handleEventsWith(handler3).then(handler4);
```
### 使用自定义EventProcessor
```Java
RingBuffer<TestEvent> ringBuffer = disruptor.getRingBuffer();
SequenceBarrier barrier = ringBuffer.newBarrier();
final MyEventProcessor customProcessor = new MyEventProcessor(ringBuffer, barrier);
disruptor.handleEventsWith(processor);
disruptor.start();
```

```Java
SequenceBarrier barrier = disruptor.after(batchEventHandler1, batchEventHandler2).asBarrier();
final MyEventProcessor customProcessor = new MyEventProcessor(ringBuffer, barrier);
```

### 发布Event

```Java
 public MyPublisher(Disruptor disruptor)
  {
    this.disruptor = disruptor;
  }

  public void run()
  {
    while (true)
    {
      computedValue = doLongRunningComputation();
      disruptor.publishEvent(this);
    }
  }

  public void translateTo(MyEvent event, long sequence)
  {
    event.setComputedValue(computedValue);
  }

  private Object doLongRunningComputation()
  {
    ...
  }
}
```