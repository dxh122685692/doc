https://mp.weixin.qq.com/s/lqFGnIUtqTFZ_GHp46z48Q
https://blog.51cto.com/wyait/1977544
https://blog.csdn.net/u014799292/article/details/90167096
https://www.cnblogs.com/williamjie/p/9481774.html
https://www.jianshu.com/p/eaafb1581e55
https://www.cnblogs.com/MicroHeart/p/10635611.html
一、队列
1.1.java队列l
队列：队列是一种先进先出的数据结构。
阻塞?
底层实现：数组、链表和堆(堆一般情况下是为了实现带有优先级特性的队列)
通过不加锁的方式实现的队列都是无界的（无法保证队列的长度在确定的范围内）；而加锁的方式，可以实现有界队列。在稳定性要求特别高的系统中，为了防止生产者速度过快，导致内存溢出，只能选择有界队列；同时，为了减少Java的垃圾回收对系统性能的影响，会尽量选择array/heap格式的数据结构。这样筛选下来，符合条件的队列就只有ArrayBlockingQueue
讲解demo队列实现，引出问题，还有哪些实现的方法？
1.2.阻塞队列
BlockingQueue
两个实现：
ArrayBlockingQueue
final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;


public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}


private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}

有界队列，有界也就意味着，它不能够存储无限多数量的对象。所以在创建 ArrayBlockingQueue 时，必须要给它指定一个队列的大小。
LinkedBlockingQueue
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();


public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}

Blocking queue的缺点主要有两个：
一个是producer只能从head放数据，producer之间会竞争head指针，存在写竞争。consumer之间会竞争tail指针，它们之间也存在写竞争。并且很多情况下，queue是处于全空状态，head/tail指针指向同一个entry，producer和consumer之间也存在写竞争。因此需要lock来实现synchronization。
另一个缺点是heal/tail指针的false sharing。
1.为什么ArrayBlockingQueue是全局锁？LinkedBlockingQueue是两个锁？

2.能用synchronized替换 ArrayBlockingQueue或 LinkedBlockingQueue中的锁吗？
ConcurrentLinkedQueue
1.3、非阻塞队列
2.1.java内置无锁并发队列
ConcurrentLinkedQueue
采用CAS实现

2.2、disruptor

三、消息队列：
消息(Message) ：应用间传送的数据。消息可以非常简单，比如只包含文本字符串、
JSON 等，也可以很复杂，比如内嵌对象。
消息队列（Message Queue 简称MQ）：是指利用高效可靠的消息传递
机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传
递和消息排队模型，它可以在分布式环境下扩展进程间的通信。
个人理解：实现了消息堆积能力的一个容器
解耦、异步、削峰

四、JMS和AMQ

五、RabbitMQ
Erlang语言实现
rabbitMQ中 消息包含两部分:有效载荷（payload）和标签（label），有效载荷就是传输的数据，标签描述了有效载荷，并且rabbitmq用它来决定谁将获得消息的拷贝。
5.1RabbitMQ结构
交换机、队列、绑定


1.为什么要有信道channel？
TCP连接创建和销毁开销大，复用TCP连接
2.为什么要有交换机，生产者直接发送到队列不可以？
原生Java链接demo代码：



5.2 四种模式：Direct fanout toppic header
Direct：通过 Routing key 来分配消息 应该分配给那个消息队列。在给交换机绑定 消息对列的时候需要指定  路由关键字，并且之歌路由关键字必须是不包含通配符。
特点：消息明确，只有一个对列会消费这个消息。
官方解释：转发消息到routingKey中指定的队列
要求队列绑定时使用的bindingKey和发送时使用routingKey的保持一致，保证只			              			       有key匹配的队列中才可以进行收发消息
fanout：把消息分给这个 交换机下面的所有 消息队列，值得注意的是 fanout 类型的 绑定 消息对列的时候不需要指配  Routing key 。所以fanout查询rabbit_route忽略了路由键
特点：分配给全部的绑定在这个交换机上的消息队列。类似于发布订阅机制。
官方解释：转发消息到与该交换机绑定的所有队列
只要接收端和发送端使用同一个交换机，所有端都可以收发消息
toppic: fanout理想的综合，把消息分配绑定在这个交换机上的多个消息队列，但是 不一定是全部。可能一个也没有，可能全部都有。通过  带有通配符的 路由关键在来指定分配规则。
在 绑定 交换机 和queue 关系的时候 ，Routing key  配置成带有 通配符的 。
发消息的 时候 发一个明确的消息 Routing key ，这样 这个消息就会分配到 合适的 消息队列中了。
特点：分配给 多个 消息队列。可以灵活指定。
官方解释：转发消息到所有关心routingkey中指定话题的队列，只要队列关心的主题(bindingkey)能与消息带的routingkey模糊匹配，就可以将消息发送到该队列。队列绑定时提供的主题可以使用"*"和"#"来的表示关键字，"*"表示一个关键字，"#"代表0个或若干个关键字。
关键字之间用"."分隔，如：有routingkey:"log","log.out","log.a.bug"; bindingKey为"log.*"的队列只能接收"log.out"的消息，而bindingKey为"log.#"的队列可以接收前面三个消息。

5.3 事务
生产端
有两种选择，transaction   和   confirm
消息确认
　　在生产者发送消息到消费者消费消息的流程中，有两个地方需要消费确认：
生产者要确认发出的消息到达RabbitMQ。
消息从队列到达消费者的过程。队列要确认发出的消息被消费者消费，才会将消息从队列中删除。
　为了保证，生产者的消息到达RabbitMQ，可以通过事务机制和发送方确认机制实现。
事务实现：
channel.txSelect();  //将当前信道设置成事务模式
channel.txCommit();  //提交事务
channel.txRollback(); //事务回滚

发送方确认实现:
channel.confirmSelect(); //将当前信道设置成事务模式

发送方确认机制：
　　生产者将信道设置成  (确认)模式，一旦信道进入confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID(从l开始)，一旦消息被投递到所有匹配的队列之后，RabbitMQ就会发送一个确认(Basic.Ack) 给生产者(包含消息的唯一ID) ，这就使得生产者知晓消息已经正确到达了目的地了(如上图的流程1)。如果消息和队列是可持久化的，那么确认消息会在消息写入磁盘之后发出。
　　生产者调用channel.ConfirmSelect将信道设置为confirm模式，事务机制和Publisher confirm机制确保的是消息能够正确地发送至RabbitMQ，这里的“发送至RabbitMQ”的含义指消息被正确地发送到交换器。
　　事务机制在一条消息发送之后会使发送端阻塞，以等待RabbitMQ 的回应，之后才能继续发送下一条消息。相比之下， 发送方确认机制最大的好处在于它是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用程序便可以通过回调方法来处理该确认消息。
消息确认机制
为了保证，队列中发出的消息被消费者消费 消费者必须对消息进行确认
　　消费者订阅队列时，可以指定autoAck参数，autoAck等于false，RabbitMQ会等待消费者显示地回复确认信号才能从队列后中删除(如上图的流程2)。autoAck等于true,会在消息发送去后删除，不管消费者是否真正消费到这条消息。当autoAck 参数置为false ，对于RabbitMQ 服务端而言，队列中的消息分成了两个部分:一部分是等待投递给消费者的消息、一部分是己经投递给消费者，但是还没有收到消费者确认信号的消息。
　　如果RabbitMQ 一直没有收到消费者的确认信号，并且消费此消息的消费者己经断开连接，则RabbitMQ 会安排该消息重新进入队列，等待投递给下一个消费者，当然也有可能还是原来的那个消费者。RabbitMQ 不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否己经断开。只到消息处理完成，这样可以防止RabbitMQ持续不断的消息涌向你的应用而导致过载。

5.3 权限系统
从1.6.1版本开始，RabbitMQ实现了一套访问控制列表（ACL）风格的权限系统。
第一级控权单位是virtual host，virtual host下面第二级的控权单位是resource（包含exchange和queue）。两个相同名称的resource如果分属不同的virtual host，则算是不同的resource。
当用户访问MQ时，首先触发第一级控权，判断用户是否有访问该virtual host的权限。
若可访问，则进行第二级控权，判断用户是否具有操作（operation）所请求的资源的权限。
读-有关消费消息的任何操作，包括"清除"整个队列（同样需要绑定操作成功）
写-发布消息（同样需要绑定操作成功）
配置-队列和交换器的创建和删除
RabbitMQ控制台页面

5.4 集群
1.

 




