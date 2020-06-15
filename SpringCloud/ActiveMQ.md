### ActiveMQ



#### 一、ActiveMQ入门概述



#### 二、ActiveMQ安装和控制台

##### 1. docker run 直接运行docker命令方式

```Linux
docker run --name='activemq' -d \
-e 'ACTIVEMQ_CONFIG_NAME=amqp-srv1' \
-e 'ACTIVEMQ_CONFIG_DEFAULTACCOUNT=false' \
-e 'ACTIVEMQ_ADMIN_LOGIN=admin' -e 'ACTIVEMQ_ADMIN_PASSWORD=admin' \
-e 'ACTIVEMQ_USERS_myproducer=p1234' -e 'ACTIVEMQ_GROUPS_writes=myproducer' \
-e 'ACTIVEMQ_USERS_myconsumer=c1234' -e 'ACTIVEMQ_GROUPS_reads=myconsumer' \
-e 'ACTIVEMQ_JMX_user1_role=readwrite' -e 'ACTIVEMQ_JMX_user1_password=j1234' \
-e 'ACTIVEMQ_JMX_user2_role=read' -e 'ACTIVEMQ_JMX_user2_password=j2345' \
-e 'ACTIVEMQ_CONFIG_TOPICS_topic1=mytopic1' -e 'ACTIVEMQ_CONFIG_TOPICS_topic2=mytopic2'  \
-e 'ACTIVEMQ_CONFIG_QUEUES_queue1=myqueue1' -e 'ACTIVEMQ_CONFIG_QUEUES_queue2=myqueue2'  \
-e 'ACTIVEMQ_CONFIG_MINMEMORY=1024' -e  'ACTIVEMQ_CONFIG_MAXMEMORY=4096' \
-e 'ACTIVEMQ_CONFIG_SCHEDULERENABLED=true' \
-v /data/activemq:/data \
-v /var/log/activemq:/var/log/activemq \
-p 8161:8161 \
-p 61616:61616 \
-p 61613:61613 \
webcenter/activemq:5.14.3
```

##### 2. docker compose 方式

[参考docker compose文件](../DockerFiles/dc-activemq.yml)

```linux
docker-compose up
```



##### 3. 开放端口号

- firewall-cmd --add-port=8161/tcp --permanent
- firewall-cmd --add-port=61616/tcp --permanent
- firewall-cmd --add-port=616163/tcp --permanent
- firewall-cmd --add-port=5672/tcp --permanent
- firewall-cmd --reload
-  firewall-cmd --list-all

##### 4. 参考链接

- [docker hub activemq](https://hub.docker.com/r/webcenter/activemq/)
- [Compose file version 3 reference](https://docs.docker.com/compose/compose-file/)
- [firewall-cmd](https://wangchujiang.com/linux-command/c/firewall-cmd.html)

##### 5. 查看后台进程方法

```
# 查看activemq进程
ps -ef | grep activemq | grep -v grep
# 查看activemq进程61616端口
netstat -anp | grep 61616
# 输出系统中所有打开文件的信息
lsof -i:61616
```

##### 6. 访问ActiveMQ控制台

-   http://192.168.80.71:8161/
- 默认用户名密码为admin/admin



#### 三、Java编码实现ActiveMQ通讯

##### 队列演示

> 生产者和消费者时一对一的关系。

###### 生产者

```java
// 1. 创建连接工场, 按照给第的URL地址，采用默认的用户名和密码
ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(ACTIVEMQ_URL);
// 2. 通过连接工场，获得连接connection并启动访问
Connection connection = factory.createConnection();
connection.start();
// 3. 创建会话session
// 两个参数，第一个叫事物，第二个叫签收
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
// 4. 创建目的地（具体是队列还是主题topic）
Queue queue = session.createQueue(QUEUE_NAME);
// 5. 创建消息的生产者
MessageProducer producer = session.createProducer(queue);
// 使用生产者发送消息
for (int i = 0; i < 10; i++) {
    // 7. 创建消息
    TextMessage textMessage = session.createTextMessage("MessageListener --> " + i);
    // 8. 通过producer发送消息
    producer.send(textMessage);
}
// 9. 关闭资源
producer.close();
session.close();
connection.close();

System.out.println("****** 消息发布到MQ完成");
```

###### 消费者

```java
System.out.println("我是1号消费者");
// 1. 创建连接工场, 按照给第的URL地址，采用默认的用户名和密码
ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(ACTIVEMQ_URL);
// 2. 通过连接工场，获得连接connection并启动访问
Connection connection = factory.createConnection();
connection.start();
// 3. 创建会话session
// 两个参数，第一个叫事物，第二个叫签收
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
// 4. 创建目的地（具体是队列还是主题topic）
Queue queue = session.createQueue(QUEUE_NAME);
// 5. 创建消费者
MessageConsumer consumer = session.createConsumer(queue);
/* 同步阻塞方式（receive()）
while (true) {
TextMessage message = (TextMessage)consumer.receive();
if (message != null) {
System.out.println("******消费者接收到消息： " + message.getText());
}else {
break;
}
}
consumer.close();
session.close();
connection.close();*/
// 消息监听方式
consumer.setMessageListener(new MessageListener() {
    public void onMessage(Message message) {
        if (message != null && message instanceof TextMessage) {
        TextMessage textMessage = (TextMessage) message;
        try {
            System.out.println("******消费者接收消息： " + textMessage.getText());
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
});
System.in.read();
consumer.close();
session.close();
connection.close();
/**
* 多个消费者时，采用平均分配的方式，类似负载均衡的轮询方式。
*/
```



##### 主题演示

> 生产者和消费者是一对多的关系。生产者将消息发布到topic中，每个消息可以有多个消费者。
>
> 生产者 和消费者之间有时间上的相关性。订阅某一个主题的消费者只能消费自他订阅之后发布的消息。
>
> 生产者生产时，topic不保存小心他是无状态的，加入无人定于就去生产，那就是一条废消息，所有一般先启动消费者在启动生产者。

生产者

```java
// 1. 创建连接工场, 按照给第的URL地址，采用默认的用户名和密码
ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(ACTIVEMQ_URL);
// 2. 通过连接工场，获得连接connection并启动访问
Connection connection = factory.createConnection();
connection.start();
// 3. 创建会话session
// 两个参数，第一个叫事物，第二个叫签收
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
// 4. 创建目的地（具体是队列还是主题topic）
Topic topic = session.createTopic(TOPIC_NAME);
// 5. 创建消息的生产者
MessageProducer producer = session.createProducer(topic);
// 使用生产者发送消息
for (int i = 0; i < 10; i++) {
    // 7. 创建消息
    TextMessage textMessage = session.createTextMessage("TOPIC_NAME --> " + i);
    // 8. 通过producer发送消息
    producer.send(textMessage);
}
// 9. 关闭资源
producer.close();
session.close();
connection.close();

System.out.println("****** TOPIC_NAME消息发布到MQ完成");
```

消费者

```java
// 1. 创建连接工场, 按照给第的URL地址，采用默认的用户名和密码
ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(ACTIVEMQ_URL);
// 2. 通过连接工场，获得连接connection并启动访问
Connection connection = factory.createConnection();
connection.start();
// 3. 创建会话session
// 两个参数，第一个叫事物，第二个叫签收
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
// 4. 创建目的地（具体是队列还是主题topic）
Topic topic = session.createTopic(TOPIC_NAME);
// 5. 创建消费者
MessageConsumer consumer = session.createConsumer(topic);
// 消息监听方式
consumer.setMessageListener(message -> {
    if (message != null && message instanceof TextMessage) {
        TextMessage textMessage = (TextMessage) message;
        try {
       		System.out.println("******TOPIC_NAME消费者接收消息： " + textMessage.getText());
        } catch (JMSException e) {
       		e.printStackTrace();
        }
    }
});
System.in.read();
consumer.close();
session.close();
connection.close();
```



#### 四、JMS规范（java message service）

##### 1. 是什么？

 Java 消息服务指的是两个应用程序之间进行异步通信的API。 

JMS provider ： 实现JMS接口和规范的消息中间件，也就是我们的MQ服务器

JMS producer：消息生产者，创建和发送JMS消息的客户端应用

JMS consumer：消息消费者，接收和处理JMS消息的客户端应用

JMS message：

​		消息头：

​				JMSDestination：消息发送的目的地，主要是指Queue和Topic

​				JMSDeliveryMode：消息持久和非持久模式

​				JMSExpiration：消息过期时间

​				JMSPriority：消息优先级

​				JMSMessageID：唯一识别每个消息的标识由MQ产生

​		消息体：

​				封装具体的消息数据

​				5中消息体格式：

​						TextMessage：普通字符串，包含一个string

​						MapMessage：一个Map类型的消息，key为string类型，value为Java的基本类型

​						BytesMessage：二进制数组消息，包含一个byte[]

​						StreamMessage：Java数据流信息

​						ObjectMessage：对象消息

​				发送和接收的消息体类型必须一致

 		消息属性：

​				识别，去重，重点标注等操作非常有用的方法

##### 2. JMS的可靠性

###### 	Persistent：持久性（对比redis持久化，rdb和aof两种持久化方式）

​      参数设置声明：

​						非持久（DeliveryMode.NON_PERSISTENT）：当服务器宕机，消息不在存在

​						持久（DeliveryMode.PERSISTENT）：当服务器宕机，消息依然存在（默认）

​               持久Queue：

​				持久Topic：消费者需要订阅

###### 	Transaction：事务

​				事务的四大特性（ACID）：原子性（Atomicity），一致性（Consistent），隔离性（Isolation），持久性（Duration）

​				producer提交时的事务

​						false： 只要执行send，就可以进入到队列中；关闭事务，第二个签收参数的设置需要有效

​						true：先执行send再执行commit，消息才被真正的提交到队列中；消息需要批量发送，需要缓冲区处理

​				事务偏生产者，签收偏消费者

###### 	Acknowledge：签收

​				自动签收（默认）

​				手动签收

​				允许重复消息

​				签收和事物的关系

​						在事务性会话中，当一个事务被成功提交则消息被自动签收，如果事后回滚，则消息会被再次传送。

​						非事务性会话中，消息何时被确认取决于创建会话时的应答模式。

###### 	总结：

​				- 非持久订阅只有当客户端处于激活状态，也就是说MQ保持连接状态才能收到发送到某个主题的消息。

​				- 如果消费者处于离线状态，生产者发送的主题消息将会丢失作废，消费者永远不会收到。

​				- 一句话：先要订阅注册才能接收到发布的消息，只给订阅者发布消息。

​				- 客户端首先向MQ注册一个自己的身份ID识别号，当这个客户端处于离线时，生产者会为这个ID保存所有发送到主题的消息。

​				  当客户再次连接到MQ时，会根据消费者的ID得到所有当前处于离线时发送到主题的消息。

​				- 非持久订阅状态下，不能恢复或者重新派送一个未签收的消息。持久订阅则可以。

​				- 当所有的消息必须被接受，则用持久订阅，当消息允许丢失则使用非持久订阅







