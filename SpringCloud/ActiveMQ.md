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



#### 五、ActiveMQ的Broker

##### 		是什么？

​				Broker是MQ服务器的一个实例。



#### 六、ActiveMQ和Spring框架整合

#### 七、ActiveMQ和Springboot框架整合

##### 		  队列：

​				生产者：触发投送，间隔投送（@scheduled，@EnableScheduling）

​				消费者：监听接收消息（@JmsListener）

##### 		  主题：

​				生产者：触发投送，间隔投送（@scheduled，@EnableScheduling）

​				消费者：监听接收消息（@JmsListener）

#### 八、[ActiveMQ传输协议](https://activemq.apache.org/protocols)

##### 	TCP(Transmission Control Protocol)

###### 		这是默认的Broker配置，TCP的Client监听端口61616

###### 		在网络传输数据前，必须要序列化数据，消息是通过一个叫wire protocol的来序列化纯字节流。默认情况下ActiveMQ吧wire  protocol叫做OpenWire，他的目的是促使网络上的效率和数据快速交互。

###### 		TCP连接到URI形式如：tcp://hostname:port?key=value&key=value，后面的参数是可选

###### 		TCP传输的优点

​			TCP协议传输可靠性高，稳定性强

​			高效性：字节流方式传递，效率高	

​			有效性，可用性：应用广泛，支持任何平台

###### 		[关于Transport协议的可配置参数可以参考官网](https://activemq.apache.org/tcp-transport-reference)

##### 	NIO(New I/O API Protocol)

​		[配置NIO协议](https://activemq.apache.org/activemq-connection-uris)：修改${ACTIVEMQ_HOME}/conf/activemq.xml

```xml
<transportConnectors>
    <transportConnector name="nio" uri="nio://0.0.0.0:61616"/>  
</<transportConnectors>
```

​		默认所有协议使用BIO网络模型，配置NIO网络模型后，仅支持tcp传输协议。因此要[开启自动NIO](https://activemq.apache.org/auto)

```xml
<transportConnector name="auto+nio" uri="auto+nio://0.0.0.0:61608?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600&amp;org.apache.activemq.transport.nio.SelectorManager.corePoolSize=20&amp;org.apache.activemq.transport.nio.SelectorManager.maximumPoolSize=50"/>
```

##### 	其他协议

- AMQP协议
- STOMP协议
- SSL协议（Secure Socket Layer Protocol）
- MQTT协议（Message Queuing Telemetry Transport）：消息队列遥测传输，即时通讯协议，IOT相关，未来有前景
- WS协议

​		

#### 九、ActiveMQ持久化机制

##### 		MQ高可用：

​			1. 事务  		MQ自带

​			2. 持久化	  MQ自带	

​			3. 签收          MQ自带

​			4. [可持久化机制](https://activemq.apache.org/persistence)

​					MQ服务器宕机了，消息也不会丢失的机制。JDBC、AMQ、KahaDB、LevelDB等来实现。

##### 		AMQ Message Store（了解）：

​				基于文件的存储方式，是以前默认的消息存储，现在不用了。

##### 		[KahaDB](https://activemq.apache.org/kahadb)消息存储（默认）：

​				基于日志文件，V5.4开始默认持久化插件。消息存储使用一个事务日志和一个索引文件来存储他所有的地址。

​				KahaDB是一个转么针对消息持久化的解决方案，他对典型的消息使用模式进行了优化。

​				数据被追加到data logs中。当不再需要log文件中的数据的时候log文件会被丢弃。

​				KahaDB文件解析：

​						db-1.log是日志数据。

​						db.data文件是一个**BTree**索引。

​						db.free当前db.data文件里哪些页面是空闲的，文件具体内容是所有空闲页面的ID

​						db.free是db.data和db.free的恢复和容灾机制。

​						lock锁文件，写独占查询共享和公开。

​			    回顾mysql原理，每创建一张表都会生成三个文件：

​						user.frm（表结构） 

​						user.MYD（新华字典正文） 

​						user.MYI（新华字典索引）

##### LevelDB消息存储（了解）

​		基于文件的本地数据库存储形式，比KahaDB更快的持久性。不是有B-Tree实现索引预写日志，二十使用基于LevelDB的索引。

##### JDBC消息存储

	- MQ + Mysql
	- 添加mysql数据的驱动包到lib文件夹
	- jdbcPersistenceAdapter配置
	- 数据库连接池配置
	- SQL建仓和建表说明
	- 代码运行验证
	- 数据库情况

##### JDBC Message store with ActiveMQ journal

- 消息的生成和接收可以不直接操作数据库，为提高效率，可以把消息保存到journal文件中，如果消费者的消费速度下降，就可以批量将消息保存到DB中。



#### 十、[ActiveMQ集群](https://activemq.apache.org/masterslave)

​	![image-20200620074336084](C:\Users\bigtiger\AppData\Roaming\Typora\typora-user-images\image-20200620074336084.png)



##### zookeeper集群

1. 使用docker compose搭建zookeeper集群

```yml
version: '3.7'

services:
  zoo1:
    container_name: zoo1
    image: zookeeper
    hostname: zoo1
    networks:
      - zoo
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo2:
    container_name: zoo2
    image: zookeeper
    hostname: zoo2
    networks:
      - zoo
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181
    
  zoo3:
    container_name: zoo3
    image: zookeeper
    hostname: zoo3
    networks:
      - zoo
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    
networks:
  zoo:
```

2. 启动客zookeeper户端

```shell
# 客户端连接单个zookeeper服务
docker run -it --rm --link zoo1:zookeeper --net myzookeeper_zoo zookeeper zkCli.sh -server zookeeper

# 客服端连接zookeeper集群
docker run -it --rm \
        --link zoo1:zk1 \
        --link zoo2:zk2 \
        --link zoo3:zk3 \
        --net myzookeeper_zoo \
        zookeeper zkCli.sh -server zk1:2181,zk2:2181,zk3:2181
        
docker run -it --rm zookeeper zkCli.sh -server lrmq-server:2181
```

添加端口

```shell
 firewall-cmd --zone=public --list-ports
 firewall-cmd --zone=public --add-port=2182/tcp --permanent
 firewall-cmd --zone=public --add-port=2183/tcp --permanent
 firewall-cmd --reload
 firewall-cmd --zone= public --query-port=80/tcp
 firewall-cmd --zone=public --query-port=80/tcp
 firewall-cmd --zone=public --query-port=2181/tcp
 firewall-cmd --zone=public --query-port=2182/tcp
 firewall-cmd --zone=public --query-port=2183/tcp
 firewall-cmd --zone=public --remove-port=61613/tcp --permanent

```

[replicated-leveldb-store](https://activemq.apache.org/replicated-leveldb-store)

#### 十一、面试题

- **引用消息队列之后该如何保障其高可用性？**

  > 持久化、事务、签收；持久化机制（KahaDB、JDBC、JDBC Journal、zookeeper+replicated-leveldb-store**主从集群**）

- **异步投递是什么？**

  > ActiveMQ支持同步和异步两种发送模式将消息发送到broker，模式的选择对发送演示有巨大的影响。使用一步发送可以显著的提高发送的性能。
  >
  > 同步模式：声明指定使用同步发送模式；未使用事务的前提下发送持久化的消息。
  >
  > **没有使用事务且发送的是持久化**的消息，每次发送都是同步发送的且会阻塞producer直到broker返回一个确认，表示消息已经被安全的持久化到磁盘。确认及时体统了消息的安全保障，但同时阻塞客户端带来了很大的延迟。
  >
  > 很多高性能的应用，允许在失败的情况下有少量的数据丢失。如果你的应用满足这个特点，你可以使用异步发送来提高生产率，即使是发送的持久化的消息。
  >
  > **异步投递应用场景**：通常在发送消息量比较密集的情况下使用异步发送。容忍消息丢失的可能。
  >
  > **异步投递问题**：小号较多的客户端内存的同时，也会导致broker性能消耗增加。不能有效的保证消息的发送成功，消息可能丢失。慢消费的情况下造成消息积压。
  >
  > [异步投递设置](https://activemq.apache.org/async-sends)

- **如果保证异步发送消息成功？**

  > 正确的异步发送方法是需要使用接收回调的。
  >
  > 同步发送和异步发送的区别就在此，同步发送等send不阻塞了就表示一定发送成功了。异步发送需要接收回执并由客户端在判断一次是否发送成功。



**[延时投递和定时投递](https://activemq.apache.org/delay-and-schedule-message-delivery)**

![image-20200620220806974](C:\Users\bigtiger\AppData\Roaming\Typora\typora-user-images\image-20200620220806974.png)

**分发策略**

**[重发机制](https://activemq.apache.org/redelivery-policy)  **

- 具体哪些情况会引发消息重发？

  > Client用了transactions，session调用了rollback方法
  >
  > Client用了transactions，在调用commit方法之前关闭或者没有commit
  >
  > Client在CLIENT_ACKNOWLEDGE的传递模式，session中调用了recover方法

- 请说说消息重发时间价格和重发次数？

  > 间隔：1
  >
  > 次数：6

- 有毒消息Poison ACK谈谈你的理解？

  > 一个消息被redelivered超过默认的最大重发次数时，消费端会给MQ发送一个poison ack表示消息有毒，告诉broker不要再发了。这个是后borker会把消息放到DLQ（死信列队）。

**[死性队列(Dead Letter Queue)](https://activemq.apache.org/message-redelivery-and-dlq-handling)**

- 是什么

  > 死信队列是异常处理消息的集合。
  >
  > ActiveMQ中引入了死信队列的概念.即一条消息在被重发了多次后（默认为重发六次），将会被ActiveMQ移入死信队列。开发人员可以在这个Queue中查看处理错误的消息，进行人工干预。

##### 幂等性

- 是什么

  > 网络传输中，会造成进行MQ重试中，在重试过程中，可能会造成重复消费。
  >
  > 如果消息是做数据库的插入操作，给这个消息做一个唯一主键，那么就算出重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。
  >
  > 如果上面两种情况还不行，准备一个第三服务放来做消费记录。一redis为例，给消息分配一个全局ID，只要消费国该消息，将id，message以k-v形式写入redis。消费者开始消费前，先去redis中查询有没有消费记录即可。

#### 十二、搭建过程中遇到的问题

1. LevelDB配置错误

   端口配置错误：tcp://0.0.0.0:**0:**63631

   ```xml
   <persistenceAdapter>
       <replicatedLevelDB
       directory="${activemq.data}/leveldb"
       replicas="3"
       bind="tcp://0.0.0.0:63631" # 端口配置错误
       zkAddress="192.168.80.71:2181,192.168.80.71:2182,192.168.80.71:2183"
       zkPath="/activemq/leveldb-stores"
       hostname="lrmq-server"  # 主机地址
       sync="local_disk" />
   </persistenceAdapter>
   ```

   原因不明？？？

   ![image-20200621130249429](C:\Users\bigtiger\AppData\Roaming\Typora\typora-user-images\image-20200621130249429.png)

   

   