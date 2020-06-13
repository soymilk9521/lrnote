### ActiveMQ





#### 环境搭建

##### 1. docker run

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

##### 2. docker compose

[参考docker compose文件](../DockerFiles/dc-activemq.yml)

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