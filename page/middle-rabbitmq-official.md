



RabbitMQ 更像是一个邮件服务器,用户发送邮件(消息),到邮箱服务器(exchange),其他用户能够保证收到消息发送者的邮件(消息).

> AMQP 服务器类似与邮件服务器, 每个交换器(exchange)都扮演了消息传送代理,每个消息队列(queue)都作为邮箱，而绑定(binding)则定义了每个传送代理中的路由表.发布者(producer)发送消息给独立的传送代理,然后传送代理(exchange)再路由(binding)消息到邮箱(queue)中.消费者(customer)从邮箱(queue)中收取消息.


RabbitMQ 有三个主要概念,生产者,队列,消费者

生产者 单纯的发送消息

队列 依赖主机的内存和磁盘,(这个可以通过配置文件修改参数)可以理解为一个缓存.

消费者 一直等待接收消息

![](image/amqp-model.png)


rabbitmq默认配置virtual host 为 "/", exchange默认AMQP default,没有默认queue.
如果exchange不指定,则exchange为默认exchange.使用virtualHost和exchange可以方便实现"分区"的概念.
比如我想有交易和会员两个系统,我可以创建两个virtualhost,分别表示两个系统,这两个系统是相互隔离的.
跟activemq不同的是,rabbitmq更加灵活一点.

# 官方文档 1 hello world

创建rabbitmq的流程跟一般的模板一样,通过ConnectionFactory--Connection--Channel--获取客户端通道.
拿到channel之后,就可以进行发布或消费.对应的jms中的点对点队列.

![](image/rabbitmq-default.png)

*在编码的时候,我们可以制定一端创建信息的规则,比如消费端进行声明exchange或者queue*

下面是消费端代码,这个代码中没有exchange,默认为类型为direct的exchange是AMQP default,这里绑定了一个队列.


    //1 工厂
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    //2 连接
    Connection connection = factory.newConnection();
    //3 渠道
    Channel channel = connection.createChannel();
    //4 创建队列
    channel.queueDeclare(QUEUE_NAME,false,false,false,null);
    //5 设置消费
    channel.basicConsume(QUEUE_NAME,true,new DefaultConsumer(channel){
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
            String msg = new String(body,"UTF-8");
            System.out.println(name + " 接收到消息 msg = " + msg);
        }
    });
    System.out.println("客户端启动.");
    latch.countDown();
    } catch (TimeoutException e) {
    e.printStackTrace();
    } catch (IOException e) {
    e.printStackTrace();
    }

下面是生产者服务端代码,注意,这里的`basicPublish()`的第二个参数,routingkey是消费端的queue的名称.
这一点,其实让人有点迷惑的.

    //1 工厂
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    //2 连接
    try (Connection connection = factory.newConnection()
    ) {
        //3 渠道
        Channel channel = connection.createChannel();
        //4 发布消息
        String msg = "hello rabbitmq";
        channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
        System.out.println(name + " 发送消息 msg = " + msg);
        channel.close();
    } catch (TimeoutException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }



如果按照生产者/消费者--exchange--routingkey绑定--queue 的标准模式,
在下面的例子中exchange为hello.world,routingkey为key-hello,队列为QUEUE_NAME

[消费端代码](http://dwz.cn/6lpyUu):

    //1 工厂
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    //2 连接
    try (Connection connection = factory.newConnection()
    ) {
        //3 渠道
        Channel channel = connection.createChannel();
        //4 发布消息
        String msg = "hello rabbitmq";
        channel.basicPublish("hello.world","key-hello",null,msg.getBytes());
        System.out.println(name + " 发送消息 msg = " + msg);
        channel.close();
    } catch (TimeoutException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

[生产端代码](http://dwz.cn/6lpyUu):


    try {
        //1 工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        //2 连接
        Connection connection = factory.newConnection();
        //3 渠道
        Channel channel = connection.createChannel();
        channel.exchangeDeclare("hello.world","direct");
        //4 创建队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.queueBind(QUEUE_NAME,"hello.world","key-hello");
        //5 设置消费
        channel.basicConsume(QUEUE_NAME,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body,"UTF-8");
                System.out.println(name + " 接收到消息 msg = " + msg);
            }
        });
        System.out.println("客户端启动.");
        latch.countDown();


# 官方文档 2 扇出模式(订阅模式)

用过jms的都知道topic订阅队列,在rabbitmq的对应的是exchange的"fanout".
实现原理,直接使用exchange,队列自动创建,不在通过routingkey绑定exchange和queue,从而实现在exchange下的queue都可以接收到消息.

![](image/rabbitmq-fanout.png)

下面是[消费端代码](http://dwz.cn/6lqrFX),创建一个exchange,类型为fanout,然后获取一个系统的queue,然后将queue和exchange绑定在一起.


    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();
    channel.exchangeDeclare(EXCHANGE_LOG, ExchangeTypes.FANOUT);
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName,EXCHANGE_LOG,"");
    Consumer consumer = new DefaultConsumer(channel){
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
            String log = new String(body);
            System.out.println(name+"<<<<  " + log);
        }
    };
    channel.basicConsume(queueName,true,consumer);
    System.out.println(name +" 客户端等待中....." );
    latch.countDown();

下面是[生产端代码](http://dwz.cn/6lqrFX)


    Channel channel = connection.createChannel();
    //channel.exchangeDeclare(EXCHANGE_LOG, ExchangeTypes.FANOUT);

    for (int i = 0; i < MSG_NUM; i++) {

        String logMsg =name +">>>> 日志.... "+i;

        channel.basicPublish(EXCHANGE_LOG,"",null,logMsg.getBytes());
    }


    channel.close();


# 官方文档 3 主题模式

首先区别正则表达式,只有2种通配符,*表示一个字(一个英文word,不是字母),#表示多个英文字.
这个实现是通过带有通配符的绑定关系,通过绑定关系,将不同的消息分发到不同的queue.

![](image/rabbitmq-topic.png)

[消费端代码](http://dwz.cn/6lqscw),创建一个exchange,类型为topic,使用通配符绑定关系绑定exchange和自身的队列.


    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(TOPIC,ExchangeTypes.TOPIC);
        String queueName = channel.queueDeclare().getQueue();
        for (int i = 0; i < routingKeys.length; i++) {
            String routingKey = routingKeys[i];
            channel.queueBind(queueName, TOPIC, routingKey);
        }
        channel.basicConsume(queueName,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println(name+ " : " +new String(body) );
            }
        });
        System.out.println(name+" 等待中.....");
        latch.countDown();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }


[生成端代码](http://dwz.cn/6lqscw)

    try (Connection connection = factory.newConnection()){
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(TOPIC, ExchangeTypes.TOPIC);
        for (String routingKey:routingKeys) {
            channel.basicPublish(TOPIC,routingKey,null,routingKey.getBytes());
        }
        channel.close();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }

# 官方文档 4 自定义路由
看了上面的简单,订阅,主题,你会更加理解exchange,routingkey,queue之间的.
上面的都是一些比较特殊场景的应用.

![](image/rabbitmq-route.png)

[消费端代码](http://dwz.cn/6lqBGu),创建一个exchange,默认类型为direct,然后通过routingkey绑定不同的queue,值得注意的是消费端可以将exchange绑定不同的queue.


    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(ROUNTING, ExchangeTypes.DIRECT);
        String queueNm = channel.queueDeclare().getQueue();
        channel.queueBind(queueNm,ROUNTING,routingKey);
        if(routingKey.contains("info")){
            channel.queueBind(queueNm,ROUNTING,"error");
        }
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body);
                System.out.println(routingKey+" : " + msg);
            }
        };
        channel.basicConsume(queueNm,true,defaultConsumer);
        System.out.println(routingKey+"客户端等待中....");
        latch.countDown();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }


[生产端代码](http://dwz.cn/6lqBGu)


    try (Connection connection = factory.newConnection()){
        Channel channel = connection.createChannel();
    //                channel.exchangeDeclare(ROUNTING, ExchangeTypes.DIRECT);
        if (name.contains("1")){
            channel.basicPublish(ROUNTING,"error",null,(name+">>error").getBytes());
            channel.basicPublish(ROUNTING,"warning",null,(name+">>warning").getBytes());
        }else {
            channel.basicPublish(ROUNTING,"info",null,(name+">>infoinfo").getBytes());
        }
        channel.close();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }

# 官方文档 5 使用mq来实现rpc
在分布式环境中,远程调用rpc有很多实现方式,比较流行的,非跨语言速度极快的java RMI,
google的基于protobuf/http2的GRPC ,facebook的IO多路复用/tcp的Thrift,使用WSDL的Web Service等.
MQ同样也可以做RPC实现,这源于MQ的天然负载均衡,以及rpc的非实时性要求.
使用rabbitmq实现rpc,用到了三点,第一是connection属性的BasicProperties,需要设置一个
应答队列replyTo,这个是在publish时带入的;第二 使用默认exchange,不需要设定exchange;
第三,应答队列的属性应当是排他自动删除的,这个使用默认无数方法生成的队列就可以,默认为排他,
自动删除,非持久队列.关于这点,可以看源码:

AutorecoveringChannel.java

    @Override
    public AMQP.Queue.DeclareOk queueDeclare() throws IOException {
        return queueDeclare("", false, true, true, null);
    }

下面是rpc服务端代码

    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.basicQos(1);
        channel.queueDeclare(REQUEST_QUEUE,false,false,true,null);
        System.out.println("RPC 服务器等待....");
        channel.basicConsume(REQUEST_QUEUE,false,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String replyQueue = properties.getReplyTo();
                AMQP.BasicProperties replyProp = new AMQP.BasicProperties().builder().correlationId(properties.getCorrelationId()).build();
                String message = new String(body);
                int n = Integer.parseInt(message);
                String responseBody =String.valueOf(fibonacci(n));
                channel.basicPublish("",replyQueue,replyProp,responseBody.getBytes());
                channel.basicAck(envelope.getDeliveryTag(),false);
                System.out.println("计算 Fibonacci ["+message+"] = "+responseBody);
            }
        });
        latch.countDown();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }
    private int fibonacci(int value){
        if(value == 0 || value == 1){
            return value;
        }else {
            return fibonacci(value-1)+fibonacci(value-2);
        }
    }

下面是rpc客户端代码,注意看没有设置exchange,队列也是使用默认的queueDeclare()

    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        //声明应答队列,默认是排他,自动删除,非持久队列,也就是说,当客户端停止了,队列就好消失
        String queueName = channel.queueDeclare().getQueue();
        String correlationId = UUID.randomUUID().toString();
        AMQP.BasicProperties properties = new AMQP.BasicProperties().builder().correlationId(correlationId).replyTo(queueName).build();
        channel.basicPublish("",REQUEST_QUEUE,properties,message.getBytes());
        BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);
        channel.basicConsume(queueName,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                if (correlationId.equalsIgnoreCase(properties.getCorrelationId())){
                    response.offer(new String(body));
                }
            }
        });
        System.out.println("接收到消息:"+response.take());
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }


# 非官方 6 被抛弃冷落的direct同胞兄弟headers 类似主题+订阅模式结合
上面的例子都是使用routingkey来进行绑定关系,在一些情况下,可能还是不能满足业务场景,
比如我想要"张三",电话"123456789"的所有消息,转到一个特殊处理(仅举例,无意义).

[消费端代码](http://dwz.cn/6lr8dv),同样是创建一个exchange,类型headers,然后构建一个map,通过BasicProperties,
传递参数.注意这里的map的value可以为java的一些基本类型(可以查阅`Frame.fieldValueSize()`),
但是不能是用户自定义的类型.rabbitmq对于不存在queue,发送的消息会丢失,所以从消息持久化的角度,
服务端和客户端都应当declare,但是只有消费端declare,并不会报错,如果消息比客户端启动更早到达,则会丢失消息.

我做了一个测试,any可以有多个,正常接收消息,类似订阅模式fanout,但是注意all只能有一个接收.

    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(HEADERS_QUEUE,ExchangeTypes.HEADERS);
        String queueName = channel.queueDeclare().getQueue();
    //                channel.queueDeclare(HEADERS_QUEUE,false,false,true,null);
        Map<String,Object> headers = new HashMap<>();
        if(name.endsWith("all")){
            headers.put("x-match","all");
            headers.put("name","张三");
            headers.put("phone","123456789");
        }else if (name.endsWith("any1")){
            headers.put("x-match","any");
            headers.put("name","张三");
            headers.put("phone","0000");
        }else{
            headers.put("x-match","any");
            headers.put("name","张三1");
            headers.put("phone","123456789");
        }
        channel.queueBind(queueName,HEADERS,"",headers);
        channel.basicConsume(queueName,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("接收消息>>"+new String(body));
            }
        });
        System.out.println("客户启动.");
        latch.countDown();
    ;            } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }



[生产端代码](http://dwz.cn/6lr8dv),服务端很简单,只需要将过滤条件添加即可.

    try {
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
    //  channel.exchangeDeclare(HEADERS, ExchangeTypes.HEADERS);
        Map<String,Object> headers = new HashMap<>();
        headers.put("name","张三");
        headers.put("phone","123456789");
        AMQP.BasicProperties properties = new AMQP.BasicProperties().builder().headers(headers).build();
        channel.basicPublish(HEADERS,"",properties,"hello rabbitmq  ".getBytes());
        channel.close();
        connection.close();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (TimeoutException e) {
        e.printStackTrace();
    }

    System.out.println("服务端启动.");


# 非官方 7 事务
事务几乎无处不在,而现在谈及事务绝不是简单的事务,而是分布式事务.遗憾的是这里的事务跟分布式事务没有必然联系.
这里单纯的谈及rabbitmq的事务.首先说一下,rabbitmq是基于tcp协议的,tcp三次握手四次挥手,这里就涉及到消息的确认
机制.而rabbitmq的事务也是依赖这个确认机制的.再来说一下确认机制,我们在使用rabbitmq或者jms默认都是
有确认机制的,只不过是默认实现,我们可以通过一些ack的参数或接口设置.一般都是默认批量自动ack,
什么时候ack呢?rabbitmq中没有消息过期的概念,只有消息被正常处理了,客户端发送了ack,才会删除.
批量ack,则是在ack到一定数量之后才一块发送ack,减少带宽,但是失败则影响较大.传统的事务,是先
开启事务,进行操作,事务提交,事务回滚,速度将减慢到原来的2倍(经过本地测试,差不多这个数).
rabbitmq提供了一个高级的Publisher Confirm机制,跟传统不太一样,实际上是将事务的提交拆分了,
等所有事务提交完毕,在最终确认.速度介于并接近非事务速度(可能测试用例问题,跟传统tx差不多?!).
当开启publisher confirm时,该信道上会为每一个消息分配一个id,当消息被发送到消费端时,rabbitmq就
会发确认到生产端,消息的发送和确认是异步.



[无事务消息代码](http://dwz.cn/6lD4ad)

    static class NoPublisher implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                try (Connection connection = factory.newConnection()) {
                    Channel channel = connection.createChannel();
                    channel.queueDeclare(NO_TRANSACTION, false, false, false, null);
                    long start = System.currentTimeMillis();
                    try {
                        for (int i = 0; i < MSG_NUM; i++) {
                            String msg = "rabbitmq msg!";
                            channel.basicPublish("", NO_TRANSACTION, null, msg.getBytes());
                        }
                        channel.basicPublish("", NO_TRANSACTION, null, "end".getBytes());
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        channel.close();
                    }
                    long end = System.currentTimeMillis();
                    System.out.println("[发送方]发送方耗时:" + (end - start));
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                e.printStackTrace();
            }
        }
    }

    static class NoConsumer implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                Connection connection = factory.newConnection();
                Channel channel = connection.createChannel();
                channel.queueDeclare(NO_TRANSACTION, false, false, false, null);
                //每次1条
                channel.basicQos(1);
                long start = System.currentTimeMillis();
                Consumer consumer = new DefaultConsumer(channel) {
                    @Override
                    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                        String msg = new String(body);
                        channel.basicAck(envelope.getDeliveryTag(),false);
                        if (msg.equalsIgnoreCase("end")){
                            long end = System.currentTimeMillis();
                            System.out.println("[接收方]接收完毕"+(end-start));
                            try {
                                channel.close();
                                connection.close();
                            } catch (TimeoutException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                };
                //手动ack
                channel.basicConsume(NO_TRANSACTION, false, consumer);
                System.out.println("[接收方]客户端等待中......");
                latch.countDown();
            } catch (TimeoutException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

输出:

    [接收方]客户端等待中......
    [发送方]发送方耗时:4080
    [接收方]接收完毕16904


[事务消息代码](http://dwz.cn/6lD4ad)



    static class TranPublisher implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                try (Connection connection = factory.newConnection()) {
                    Channel channel = connection.createChannel();
                    channel.queueDeclare(TRANSACTION, false, false, false, null);
                    long start = System.currentTimeMillis();
                    try {
                        for (int i = 0; i < MSG_NUM;) {
                            if (i%BATCH ==0){
                                //开启事务
                                channel.txSelect();
                                for (int j = 0; j < BATCH; j++) {
                                    String msg = "rabbitmq msg!";
                                    if(i + j != MSG_NUM -1 ){
                                        channel.basicPublish("", TRANSACTION, null, msg.getBytes());
                                    }else{
                                        channel.basicPublish("", TRANSACTION, null, "end".getBytes());
                                    }
                                }
                                i += BATCH;
                                //commit
                                channel.txCommit();
                            }
                        }

                    } catch (Exception e) {
                        //回滚事务
                        channel.txRollback();
                        e.printStackTrace();
                    } finally {
                        channel.close();
                    }
                    long end = System.currentTimeMillis();
                    System.out.println("[tx发送方]发送方耗时:" + (end - start)+" 批量大小="+BATCH);
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                e.printStackTrace();
            }
        }
    }

    static class TranConsumer implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                Connection connection = factory.newConnection();
                Channel channel = connection.createChannel();
                channel.queueDeclare(TRANSACTION, false, false, false, null);
                //每次1条
                channel.basicQos(1);
                long start = System.currentTimeMillis();
                Consumer consumer = new DefaultConsumer(channel) {
                    @Override
                    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                        String msg = new String(body);
                        //发送ack
                        channel.basicAck(envelope.getDeliveryTag(), false);
                        if (msg.equalsIgnoreCase("end")){
                            long end = System.currentTimeMillis();
                            System.out.println("[tx接收方]接收完毕"+(end-start));
                            try {
                                channel.close();
                                connection.close();
                            } catch (TimeoutException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                };
                //手动ack
                channel.basicConsume(TRANSACTION, false, consumer);
                System.out.println("[tx接收方]客户端等待中......");
                latch.countDown();
            } catch (TimeoutException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }


输出:

    [tx接收方]客户端等待中......
    [tx发送方]发送方耗时:8703 批量大小=100
    [tx接收方]接收完毕22160

[消息确认代码](http://dwz.cn/6lD4ad)

    static class ConfirmPublisher implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                try (Connection connection = factory.newConnection()) {
                    Channel channel = connection.createChannel();
                    long start = System.currentTimeMillis();
                    try {
                        for (int i = 0; i < MSG_NUM; ) {
                            if (i%BATCH ==0){
                                //开启confirm3
                                channel.confirmSelect();
                                for (int j = 0; j < BATCH; j++) {
                                    String msg = "rabbitmq msg!";
                                    if(i + j != MSG_NUM -1){
                                        channel.basicPublish("", CONFIRM, null, msg.getBytes());
                                    }else{
                                        channel.basicPublish("", CONFIRM, null, "end".getBytes());
                                    }
                                }
                                i += BATCH;
                                //confirm
    //                                  waitForConfirmsOrDie 相对于 waitForConfirms 来说,只要有nack就好抛出异常,同时也是一种阻塞式
                                channel.waitForConfirmsOrDie();
                                //channel.addConfirmListener(new ConfirmListener() {
    //                                    @Override
    //                                    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
    ////                                        System.out.println("ack deliveryTag = " + deliveryTag);
    //                                    }
    //
    //                                    @Override
    //                                    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
    ////                                        System.out.println("nack deliveryTag = " + deliveryTag);
    //                                    }
    //                                });
                            }
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    } finally {
                        channel.close();
                    }
                    long end = System.currentTimeMillis();
                    System.out.println("[confirm发送方]发送方耗时:" + (end - start)+" 批量大小="+BATCH);
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                e.printStackTrace();
            }
        }
    }

    static class ConfirmConsumer implements Runnable {

        /**
         * When an object implementing interface <code>Runnable</code> is used
         * to create a thread, starting the thread causes the object's
         * <code>run</code> method to be called in that separately executing
         * thread.
         * <p>
         * The general contract of the method <code>run</code> is that it may
         * take any action whatsoever.
         *
         * @see Thread#run()
         */
        @Override
        public void run() {
            try {
                Connection connection = factory.newConnection();
                Channel channel = connection.createChannel();
                channel.queueDeclare(CONFIRM, false, false, false, null);
                //每次1条
                channel.basicQos(1);
                long start = System.currentTimeMillis();
                Consumer consumer = new DefaultConsumer(channel) {
                    @Override
                    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                        String msg = new String(body);
                        //发送ack
                        channel.basicAck(envelope.getDeliveryTag(), false);
    //                     System.out.println("确认"+msg);
                        if (msg.equals("end")){
                            long end = System.currentTimeMillis();
                            System.out.println("[confirm接收方]接收完毕"+(end-start));
                            try {
                                channel.close();
                                connection.close();
                            } catch (TimeoutException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                };
                //手动ack
                channel.basicConsume(CONFIRM, false, consumer);
                System.out.println("[confirm接收方]客户端等待中......");
                latch.countDown();
            } catch (TimeoutException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

输出:

    [confirm接收方]客户端等待中......
    [confirm发送方]发送方耗时:5358 批量大小=100
    [confirm接收方]接收完毕22502


10w简单消息发送时间

无事务:15s左右

tx事务:20s左右

confirm事务:`20s左右`

本地测试,所以这里没有网络的延迟.

`这里有个疑问,confirm事务没有像官方说明的一样,接近无事务的效率.`
