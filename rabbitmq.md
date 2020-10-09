1. 降低

2. 耦合，异步调用

3. activemq和rocketmq只支持java语言，kafka可以支持多语言，rabbitmq支持多种语言

4. 效率方面，activemq，rocketmq，kafka效率都是毫秒级别的，rabbitmq是微秒级别的。

5. rabbitmq严格遵循amqp协议，高吸消息队列协议，帮助我们在进程之间传递异步消息

6. 安装RabbieMQ

7. ```yml
   version: '3.1'
   services:
     rabbitmq:
       image: daocloud.io/library/rabbitmq:3.7.26-management
       restart: always
       container_name: rabbitmq
       ports:
         - 5672:5672 #mq程序的端口
         - 15672:15672 #mq图形界面的端口
       volumes:
         - /data:/var/lib/rabbitmq
   ```

8. RabbitMQ架构

   

   ![1600418555000](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600418555000.png)

   1. Publisher 生产者 发布消息到Exchange

   2. Consumer 消费者 坚挺RabbitMQ中的Queue中的消息

   3. Exchange 交换机 和生产者建立连接并接收生产者的消息

   4. Queue 队列 Exchange 会将消息分发到指定的Queue，Queue和消费者交互

   5. Routes-路由（交换机以什么杨的策略将消息发布到Queue）

      ![1600419119530](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600419119530.png)

      查看图形化界面，并创建Virtual Host

      ![1600419406737](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600419406737.png)

      之后点击用户，绑定virtual host
      
   6. java代码连接mq

      1. 创建maven工程

      2. ```xml
         导入依赖
          <dependencies>
             <!-- https://mvnrepository.com/artifact/com.rabbitmq/amqp-client -->
             <dependency>
               <groupId>com.rabbitmq</groupId>
               <artifactId>amqp-client</artifactId>
               <version>5.9.0</version>
             </dependency>
             <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
               <version>4.13</version>
             </dependency>
           </dependencies>
         ```

      3. 创建工具类连接

         ```java
         public static Connection getConnection() throws IOException, TimeoutException {
             ConnectionFactory factory = new ConnectionFactory();
             factory.setHost("10.77.100.38");
             factory.setPort(5672);
             factory.setUsername("test");
             factory.setPassword("test");
             factory.setVirtualHost("/test");
             Connection connection = factory.newConnection();
             return connection;
           }
         ```

      4. 六种模式

      5. 第一种：hello-world

      6. ![1600673225696](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600673225696.png)

      7. 创建pulisher
   
         ``` java
         
           @Test
           public void publish () throws IOException, TimeoutException {
             //1.获取connection
             Connection connection = RabbitMQClient.getConnection();
             //2.创建channel
             Channel channel = connection.createChannel();
             //3.发送消息
             String msg = "hello-world";
             //参数1：指定exchange 使用""
             //参数2：指定路由规则，使用具体的队列名称
             //参数3：指定传递的消息所携带的properties 使用null
             //参数4：指定发布的具体消息，byte[]类型
             channel.basicPublish("", "HelloWorld", null, msg.getBytes());
             //ps：exchange不会持久化保存消息，需要用queue来实现持久化
             //4.释放资源
           }
         ```
   
         创建consumer
   
         ``` java
          @Test
           public void consume() throws IOException, TimeoutException {
             //1.获取连接
             Connection connection = RabbitMQClient.getConnection();
             //2.创建channel
             Channel channel = connection.createChannel();
             //3.声明队列- HelloWorld
             //参数1：queue指定队列名称
             //参数2：durable 当前队列是否需要持久化
             //参数3：exclusive 是否排除(connection.close()。当前队列自动删除，且只能被一个消费者消费)
             //参数4： autoDelete 当前队列没有消费者消费，就自动删除
             //参数5：arguments 其他参数
             channel.queueDeclare("HelloWorld", true, false, false, null);
             //4.开启监听
             DefaultConsumer consume = new DefaultConsumer(channel){
               @Override
               public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                   byte[] body) throws IOException {
                 System.out.println("接收到消息" + new String(body, "UTF-8"));
               }
             };
             //参数1：queue指定消费队列名称
             //参数2：是否自动ack，true 接收消息后，会立即告诉rabbitmq
             //参数3： 指定消费回调
             channel.basicConsume("HelloWorld", true, consume);
             System.out.println("消费者开始监听消息");
             System.in.read();
             //5.释放资源
             channel.close();
             connection.close();
           }
         ```
   
      8. 第二种：work
   
         8.1 一个生产者，一个exchange 一个队列 两个消费者
   
         ![1600681570461](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600681570461.png)
   
         8.2 生产者
   
         ```java
         @Test
           public void publish() throws IOException, TimeoutException {
             //1.创建连接
             Connection connection = RabbitMQClient.getConnection();
             //2.创建通道
             Channel channel = connection.createChannel();
             //3.发布消息
             for (int i = 0; i < 10; i++) {
               String msg = "work======" + i;
               channel.basicPublish("", "work", null, msg.getBytes());
             }
             //4.关闭资源
             channel.close();
             connection.close();
           }
         ```
   
         8.3 消费者
   
         ```java
         @Test
           public void consume() throws IOException, TimeoutException {
             //1.建立连接
             Connection connection = RabbitMQClient.getConnection();
             //2.创建通道
             final Channel channel = connection.createChannel();
             //s声明队列
             channel.queueDeclare("work", true, false, false, null);
             //指定当前消费者 一次消费多少个消息
             channel.basicQos(1);
             //3.消费消息
             DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
               @Override
               public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                   byte[] body) throws IOException {
                 try {
                   Thread.sleep(50);
                 } catch (InterruptedException e) {
         
                 }
                 System.out.println("consumer1监听到消息=====" + new String(body, "UTF-8"));
                 //手动ack
                 channel.basicAck(envelope.getDeliveryTag(), false);
               }
             };
             channel.basicConsume("work", false, defaultConsumer);
             //4.system.in.read()持续监听
             System.out.println("consumer1开始监听消息");
             System.in.read();
             //5.关闭资源
             channel.close();
             connection.close();
           }
         ```
   
      9. 第三种模式：publish/subscribe
   
         ![1600739723866](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600739723866.png)
   
         9.1 生产者
   
         ```java
          @Test
           public void publish() throws IOException, TimeoutException {
             Connection connection = RabbitMQClient.getConnection();
             Channel channel = connection.createChannel();
             //自定义exchange-绑定某一个队列
             //参数1：exchange的名称
             //参数2：exchange的类型 FANOUT- pubsub， DIRECT-Routing TOPIC-Topics
             channel.exchangeDeclare("pubsub-exchange", BuiltinExchangeType.FANOUT);
             //绑定队列到exchange
             //参数1：队列名
             //参数2：exchange名称
             //参数3：规则
             channel.queueBind("pubsub-queue1", "pubsub-exchange", "");
             channel.queueBind("pubsub-queue2", "pubsub-exchange", "");
             for (int i = 0; i < 10; i++) {
               channel.basicPublish("pubsub-exchange", "pubsub", null, ("pubsub===" + i).getBytes());
             }
             channel.close();
             connection.close();
           }
         ```
   
         9.2消费者
   
         ```java
         @Test
           public void consume() throws IOException, TimeoutException {
             Connection connection = RabbitMQClient.getConnection();
             final Channel channel = connection.createChannel();
             channel.queueDeclare("pubsub-queue2", true, false, false, null);
             channel.basicQos(1);
             DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
               @Override
               public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                   byte[] body) throws IOException {
                 System.out.println("消费者2号收到消息====" + new String(body, "UTF-8"));
         
                 channel.basicAck(envelope.getDeliveryTag(), false);
               }
         
             };
             channel.basicConsume("pubsub-queue2", false, defaultConsumer);
         
             System.in.read();
         
             channel.close();
         
             connection.close();
         
           }
         ```
   
      10. 第四种模式：Routing
   
          ![1600741150570](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1600741150570.png)
   
          10.1publisher生产者
   
          ```java
            @Test
            public void publish() throws IOException, TimeoutException {
              Connection connection = RabbitMQClient.getConnection();
              Channel channel = connection.createChannel();
              channel.exchangeDeclare("routing-exchange", BuiltinExchangeType.DIRECT);
              //指定routingkey
              channel.queueBind("routing-queue1", "routing-exchange", "error");
              channel.queueBind("routing-queue2", "routing-exchange", "info");
          	//发送消息的时候指定routingkey
              channel.basicPublish("routing-exchange", "error", null, ("错误===error").getBytes());
              channel.basicPublish("routing-exchange", "info", null, ("信息===info").getBytes());
              channel.basicPublish("routing-exchange", "info", null, ("信息===info1").getBytes());
              channel.basicPublish("routing-exchange", "info", null, ("信息===info2").getBytes());
              channel.basicPublish("routing-exchange", "info", null, ("信息===info3").getBytes());
          
              channel.close();
              connection.close();
            }
          ```
   
          10.2消费者 consumer
   
          ```java
            @Test
            public void consume() throws IOException, TimeoutException {
              Connection connection = RabbitMQClient.getConnection();
              final Channel channel = connection.createChannel();
              channel.queueDeclare("routing-queue1", true, false, false, null);
              channel.basicQos(1);
              DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                  System.out.println("消费者1接收消息====" + new String(body, "UTF-8"));
                  channel.basicAck(envelope.getDeliveryTag(), false);
                }
              };
              channel.basicConsume("routing-queue1", false, defaultConsumer);
              System.in.read();
              channel.close();
              connection.close();
            }
          ```
   
          绑定的事情，生产者或消费者都没区别
   
      11. 第五种模式：Topic
   
          ![1601016143553](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1601016143553.png)
   
          11.1 一个生产者 两个消费者，两条队列，一个交换机
   
          11.2 生产者：绑定时可以通过#和*关键字来指定Routingkey的内容
   
          ```java
          @Test
            public void publish() throws IOException, TimeoutException {
          
              Connection connection = RabbitMQClient.getConnection();
          
              Channel channel = connection.createChannel();
              //声明topic-exchange
              channel.exchangeDeclare("topic-exchange", BuiltinExchangeType.TOPIC);
              //绑定topic-queue
              //*.red.* --->* 占位符
              //fast.#  ---># 通配符
              //#.rabbit ---> *.*.rabbit两者意思相同
              channel.queueBind("topic-queue-1", "topic-exchange", "*.red.*");
              channel.queueBind("topic-queue-2", "topic-exchange", "fast.#");
              channel.queueBind("topic-queue-2", "topic-exchange", "*.*.rabbit");
          
              channel.basicPublish("topic-exchange", "fast.red.monkey", null, "快红猴".getBytes());
              channel.basicPublish("topic-exchange", "slow.black.dog", null, "慢黑狗".getBytes());
              channel.basicPublish("topic-exchange", "fast.yellow.rabbit", null, "快黄兔".getBytes());
              channel.basicPublish("topic-exchange", "fast.white.cat", null, "快白猫".getBytes());
          
              channel.close();
          
              connection.close();
          
            }
          ```
   
          11.3消费者只是监听消息
   
          ```java
          @Test
            public void consume() throws IOException, TimeoutException {
              Connection connection = RabbitMQClient.getConnection();
          
              final Channel channel = connection.createChannel();
          
              channel.basicQos(1);
          
              channel.queueDeclare("topic-queue-1", true, false, false, null);
          
              DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                    byte[] body) throws IOException {
                  System.out.println("红色动物爱好者收到===" + new String(body, "UTF-8"));
          
                  channel.basicAck(envelope.getDeliveryTag(), false);
                }
              };
              channel.basicConsume("topic-queue-1", false, defaultConsumer);
              System.in.read();
          
          
              channel.close();
              connection.close();
            }
          ```
   
   7. springboot整合RabbitMQ
   
      7.1 导入依赖
   
      ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
          </dependency>
      ```
   
      7.2配置rabbitmq的服务器信息
   
      ```yml
      spring:
        rabbitmq:
          host: 10.77.100.38 #地址
          port: 5672 #端口
          username: test
          password: test
          virtual-host: /test
      ```
   
      7.3 配置rabbitmq的exchange和queue，并绑定
   
      ```java
      @Configuration
      public class RabbitMQConfig {
      
        //1.创建exchange，topic方式
        @Bean
        public TopicExchange getTopicExchange() {
          return new TopicExchange("boot-topic-exchange", true, false, null);
        }
      
        //2.创建队列
        @Bean
        public Queue getQueue() {
          return new Queue("boot-queue", true, false, false, null);
        }
        //3.绑定
      
        @Bean
        public Binding getBinding(TopicExchange topicExchange, Queue queue) {
          return BindingBuilder.bind(queue).to(topicExchange).with("*.red.*");
        }
      }
      ```
   
      7.4 消息生产者
   
      ```java
      	@Autowired
        private RabbitTemplate rabbitTemplate;
      
        @Test
        void publish() {
          rabbitTemplate.convertAndSend("boot-topic-exchange", "slow.red.dog", "红色大狼狗");
        }
      ```
   
      7.5消息消费者
   
      ```java
      @Component
      public class Consumer {
        //监听队列
        @RabbitListener(queues = "boot-queue")
        public void getMessage(Object message) {
          System.out.println("接受到消息 ===" + message);
        }
      }
      ```
   
      7.6实现手动ack
   
      ```yml
      #mq的配置添加以下内容，改成手动指定ack
        listener:
            simple:
              acknowledge-mode: manual
      ```
   
      ```java
      package com.zfc.rabbitmq.listen;
      
      import com.rabbitmq.client.Channel;
      import java.io.IOException;
      import org.springframework.amqp.core.Message;
      import org.springframework.amqp.rabbit.annotation.RabbitListener;
      import org.springframework.stereotype.Component;
      
      /**
       * @author ：zhangfc
       * @date ：Created in 2020/9/25 15:29
       * @description：
       */
      @Component
      public class Consumer {
      
        @RabbitListener(queues = "boot-queue")
          //换了接收参数
        public void getMessage(String msg, Channel channel, Message message) throws IOException {
          System.out.println("接受到消息 ===" + msg);
          //手动ack
          channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
      }
      
      ```
   
      手动ack可以防止异常时消息没有成功消费，却自动通知rabbitmq将队列消息删除，造成消息丢失
   
   8. RabbitMQ的其他操作
   
      可靠性：保证消息的准确发送和消费
   
      **RabbitMQ的队列有持久化机制，即使RabbitMQ宕机也可以在重启之后依然存在**
   
      **消费者宕机通过手动ack来保证消息不会丢失**
   
      生产者未能成功把消息发送到rabbitmq：RabbitMQ提供了事务操作和confirm操作
   
      **RabbitMQ的事务：事务可以保证消息100%传递，可以通过事务的回滚去记录日志，后面定时再次发送当前消息。事务的操作，效率太低**
   
      ![1601020606079](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1601020606079.png)
   
      **RabbitMQ除了事务，还提供了confirm的机制，比事务效率高很多**
   
      1. 普通Confirm机制
      2. 批量Confirm机制
      3. 异步Confirm机制
   
   9. 普通confirm机制
   
      ```java
      //开启confirm机制
          channel.confirmSelect();
          channel.basicPublish("", "hello-world", null, "普通confirm".getBytes());
          if (channel.waitForConfirms()) {
            System.out.println("发送消息成功");
          } else {
            System.out.println("消息发送失败");
          }
      ```
   
   10. 批量confirm机制
   
       ```java
           //开启confirm机制
           channel.confirmSelect();
           int i = 0;
           while (i < 100) {
             channel.basicPublish("", "hello-world", null, "普通confirm".getBytes());
             i++;
           }
           channel.waitForConfirmsOrDie(); //没有返回值，如果一个失败，全部都会失败，并且抛出io异常
       ```
   
   11. 异步confirm机制
   
       ```java
       //confirm机制
       channel.confirmSelect();
       int i = 0;
       while (i < 100) {
         channel.basicPublish("", "hello-world", null, "普通confirm".getBytes());
         i++;
       }
       channel.addConfirmListener(new ConfirmListener() {
         public void handleAck(long l, boolean b) throws IOException {
           System.out.println("消息发送成功，标识"+ l + "，是否是批量" + b);
         }
       
         public void handleNack(long l, boolean b) throws IOException {
           System.out.println("消息发送失败，标识"+ l + "，是否是批量" + b);
         }
       });
       ```
   
   12. confirm机制只能确保生产者把消息发送到exchange上，消费者监听的是queue，无法保障exchange到queue的成功
   
       ![1601024498055](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1601024498055.png)
   
       **Return机制**:监听消息是否从exchange 送到了指定的queue
   
       1. 开启Return机制
   
          ```java
          //开启Return机制，如果没送达queue执行，如果发送到queue就不执行
              channel.addReturnListener(new ReturnListener() {
                public void handleReturn(int i, String s, String s1, String s2,
                    BasicProperties basicProperties, byte[] bytes) throws IOException {
                  System.out.println(new String(bytes, "UTF-8") + "没有发送到queue中");
                }
              });
          //发送消息时，指定mandatory为true，
          channel.basicPublish("", "hello-world", true, null, "普通confirm".getBytes());
          ```
   
   13. springboot实现confirm和return机制
   
       1. 修改配置文件
   
          ```yml
          #rabbitmq下添加两个属性
          publisher-confirm-type: simple
          publisher-returns: true
          ```
   
       2. 新建config配置类
   
          ```java
          package com.zfc.rabbitmq.config;
          
          import javax.annotation.PostConstruct;
          import org.springframework.amqp.core.Message;
          import org.springframework.amqp.rabbit.connection.CorrelationData;
          import org.springframework.amqp.rabbit.core.RabbitTemplate;
          import org.springframework.beans.factory.annotation.Autowired;
          import org.springframework.stereotype.Component;
          
          /**
           * @author ：zhangfc
           * @date ：Created in 2020/9/25 17:32
           * @description：
           */
          @Component
          public class PublisherConfirmAndReturnConfig implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {
          
          
            @Autowired
            private RabbitTemplate rabbitTemplate;
          
            // @PostConstruct 修饰非静态void函数，被修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器加载一次，在构造函数之后执行，在init()函数之前执行
            @PostConstruct
            public void initMethod() {
              rabbitTemplate.setConfirmCallback(this);
              rabbitTemplate.setReturnCallback(this);
            }
          
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
              if (ack) {
                System.out.println("消息已经送到了exchange");
              } else {
                System.out.println("消息没送到exchange");
              }
            }
          
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                //通过message可以获取到消息
              System.out.println("消息没有送达到queue");
            }
          }
          ```
   
   14. 消息重复消费问题：消费者没有给rabbitmq一个 ack，导致rabbitmq被消费的消息依然存在。
   
       ![1601027448218](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1601027448218.png)
   
       重复消费消息会对非幂等性造成问题！幂等性操作：删除和修改
   
       >为了解决重复消费的问题，可以采用redis，在消费者消费消息之前，先将id放到redis中，
       >
       >id -0(正在执行业务)
       >
       >id-1(执行业务成功)
       >
       >如果ack失败，在RabbitMQ将消息交给其他消费者，先执行setnx，如果key已经存在，获取他的值，如果是0，当前消费者就什么都不做，如果是1，直接ack
   >
       >极端情况：第一个消费者在执行业务时，出现死锁，在setnx的基础上，再给key设置一个生存时间
   
   
   ​    
   ~~~java
   1. 添加jedis依赖
   
      ```xml
      <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.3</version>
          </dependency>
      ```
   
   2. 指定消息的唯一标识
   
      生产者
   
      ```java
      AMQP.BasicProperties properties = new AMQP.BasicProperties().builder()
          .deliveryMode(1) //指定消息是否需要持久化，1-需要持久化，2-不需要持久化
          .messageId(UUID.randomUUID().toString()) //指定消息的唯一标识
          .build();
      //properties传入下方的方法中
      //参数3：mandatory 是否回调，false不回调，true回调
      channel.basicPublish("", "hello-world", true, properties, "普通confirm".getBytes());
      ```
   
      消费者
   
      ```java
      DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                byte[] body) throws IOException {
              Jedis jedis = new Jedis("10.77.100.38", 6379);
              String messageId = properties.getMessageId();
              String set = jedis.set(messageId, "0", "NX", "EX", 10);
               //如果设置成功就消费消息
              if (set != null && set.equals("ok")) {
                System.out.println("获得消息===" + new String(body, "UTF-8"));
                jedis.set(messageId, "1");
                channel.basicAck(envelope.getDeliveryTag(), false);
              } else {
                String s = jedis.get(messageId);
               //如果值等于1，说明已经被消费，直接ack
                if ("1".equals(s)) {
                  channel.basicAck(envelope.getDeliveryTag(), false);
                }
              }
            }
          };
      ```
   ~~~
   
   
   ​       
   ​    
   ~~~java
   3. springboot实现上面的方法
   
      引入依赖
   
      ```xml
      <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
          </dependency>
      ```
   
      生产者
   
      ```java
      @Test
        void publish() throws IOException {
          CorrelationData data = new CorrelationData(UUID.randomUUID().toString());
          rabbitTemplate.convertAndSend("boot-topic-exchange", "slow", "红色大狼狗", data);
          System.in.read();
        }
      ```
   
      消费者
   
      ```java
      @Autowired
        private StringRedisTemplate stringRedisTemplate;
      
        @RabbitListener(queues = "boot-queue")
        public void getMessage(String msg, Channel channel, Message message) throws IOException {
          //1.设置key到redis
          String messageId = message.getMessageProperties()
              .getHeader("spring_returned_message_correlation");
          //2.消费消息
          if (stringRedisTemplate.opsForValue().setIfAbsent(messageId, "0", 10, TimeUnit.SECONDS)) {
            System.out.println("接受到消息 ===" + msg);
            //3. 设置key的value 为1
            stringRedisTemplate.opsForValue().set(messageId, "1");
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
          } else {
            //手动ack
            String s = stringRedisTemplate.opsForValue().get(messageId);
            if ("1".equals(s)) {
              channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
            }
          }
        }
      ```
   ~~~
   
   
   ​       
   


