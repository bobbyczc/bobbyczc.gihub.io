---
layout: post
title: '消息中间件（二）'
subtitle: 'RabbitMQ入门及消息分发机制'
date: 2022-02-26
categories: mq
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: mq rabbitmq
---

# RabbitMQ入门及消息分发机制

## 1. RabbitMQ 简介

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗

### 特点

- 可靠性
- 灵活路由
- 消息集群

- 高可用
- 插件机制
- 多种协议

- 多语言客户端
- 管理界面
- 跟踪机制

## 2. RabbitMQ 安装

### 环境准备

> CentOs8
Erlang
> 
1. **安装Erlang**
    - 选择合适版本
      
        地址： [https://github.com/rabbitmq/erlang-rpm/releases](https://github.com/rabbitmq/erlang-rpm/releases)
        
        el8对应CentOS8的版本：[https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.10/erlang-23.3.4.10-1.el8.x86_64.rpm](https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.10/erlang-23.3.4.10-1.el8.x86_64.rpm)
        
    - 下载
      
        ```bash
        wget https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.10/erlang-23.3.4.10-1.el8.x86_64.rpm
        ```
        
    - 安装，执行下面代码,一路按y
      
        ```bash
        yum install erlang-23.3.4.10-1.el8.x86_64.rpm
        ```
        
    - 查看erlang 版本
      
        ```bash
        [root@VM-12-12-centos rabbitmq]# erl
        Erlang/OTP 23 [erts-11.2.2.9] [source] [64-bit] [smp:1:1] [ds:1:1:10] [async-threads:1] [hipe]
        
        Eshell V11.2.2.9  (abort with ^G)
        1> halt().
        ```
    
2. **安装RabbitMQ**
    - 选择版本
      
        地址：[https://www.rabbitmq.com/which-erlang.html](https://www.rabbitmq.com/which-erlang.html) 可以查看erlang以及rabbitmq的旧版本兼容性
        
        地址：[https://github.com/rabbitmq/rabbitmq-server/releases](https://github.com/rabbitmq/rabbitmq-server/releases) 选择指定的版本下载，我们选择 3.9.10的el8版本下载
        
    - 下载
      
        ```bash
        wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.10/rabbitmq-server-3.9.10-1.el8.noarch.rpm
        ```
        
    - 安装
      
        ```bash
        yum install rabbitmq-server-3.9.10-1.el8.noarch.rpm
        ```
    
3. **启动**
   
    ```bash
    #启动：
    systemctl start rabbitmq-server.service
    #停止：
    systemctl stop rabbitmq-server.service
    #查看状态：
    systemctl status rabbitmq-server.service
    ```
    
4. **启动UI插件**
   
    ```bash
    rabbitmq-plugins enable rabbitmq_management
    #查看页面 http://ip:15672/#/
    #查看用户列表,默认只有一个guest用户
    rabbitmqctl list_users
    #guest用户只能本地登录，因此需要手动添加一个用户
    rabbitmqctl add_user {username} {password}
    #设置用户权限
    rabbitmqctl set_user_tags {username} administrator
    #登录
    ```
    
5. 设置防火墙

## 3. RabbitMQ基本配置

1. 默认配置
   
    RabbbitMQ有一套默认配置，能够满足日常开发需求，如果需要修改，需要自己创建一个配置文件 touch /etc/rabbitmq/rabbitmq.conf
    
    - 配置文件示例：[https://github.com/rabbitmq/rabbitmq- server/blob/master/deps/rabbit/docs/rabbitmq.conf.example](https://github.com/rabbitmq/rabbitmq-)
    - 配置项说明：[https://www.rabbitmq.com/configure.html#config-items](https://www.rabbitmq.com/configure.html#config-items)
2. RabbitMQ端口
    - 5672, 5671：AMQP 0-9-1 和 1.0 客户端端口，没有使用SSL和使用SSL的端口。
    - 4369：是Erlang的端口/结点名称映射程序，用来跟踪节点名称监听地址，在集群中起到一个类似 DNS的作用。
    - 25672：用于RabbitMQ节点间和CLI工具通信，配合4369使用。（命令行操作时）
    - 15672：HTTP_API端口，管理员用户才能访问，用于管理RbbitMQ，需要启用management插件。
    - 61613,61614：当STOMP插件启用的时候打开，作为STOMP客户端端口（根据是否使用TLS选择）。
    - 1883, 8883：当MQTT插件启用的时候打开，作为MQTT客户端端口（根据是否使用TLS选择）。
    - 15674：基于WebSocket的STOMP客户端端口（当插件Web STOMP启用的时候打开）。
    - 15675：基于WebSocket的MQTT客户端端口（当插件Web MQTT启用的时候打开）

## 4. RabbitMQ管理界面

<aside>
💡 RabbitMQ 安装包中带有管理插件，但需要手动激活
rabbitmq-plugins enable rabbitmq_management

</aside>

1. **RabbitMQ角色**
    - none：不能访问management_plugin
    - management：

## 5. Hello Word

1. java程序
    - maven依赖
      
        ```xml
        <dependency>
        	<groupId>com.rabbitmq</groupId>
        	<artifactId>amqp-client</artifactId>
        	<version>5.5.1</version>
        </dependency>
        ```
        
    - 生产者
      
        ```java
        import com.rabbitmq.client.Channel;
        import com.rabbitmq.client.Connection;
        import com.rabbitmq.client.ConnectionFactory;
        
        import java.io.IOException;
        import java.util.concurrent.TimeoutException;
        
        public class Producer {
        
            public static void main(String[] args) {
                // 1. 创建连接工厂
                ConnectionFactory factory = new ConnectionFactory();
                // 2. 设置连接属性
                factory.setHost("175.178.15.23");
                factory.setPort(5672);
                factory.setUsername("admin");
                factory.setPassword("admin");
        
                Connection connection = null;
                Channel channel = null;
        
                try {
                    // 3.从连接工厂获取连接
                    connection = factory.newConnection("Producer");
        
                    // 4.从连接中创建通道
                    channel = connection.createChannel();
        
                    /**
                     * 5.声明（创建）队列
                     * 如果队列不存在，才会创建
                     * RabbitMQ不允许声明两个队列名相同，属性不同的队列，否则会报错
                     *
                     * queueDeclare参数说明
                     * @param queue 队列名称;
                     * @param durable 队列是否持久化
                     * @param exclusive 是否排他，即是否私有的，如果为true，会对当前队列加锁，其他通道不能访问没并且在连接关闭时自动删除，不受是否持久化和自动删除的影响
                     * @param autoDelete 是否自动删除，当最后一个消费者断开连接之后是否自动删除
                     * @param arguments 队列参数，设置队列的有效期，消息最大长度，队列中所有消息的生命周期等等
                     */
                    channel.queueDeclare("queue1",false,false,false,null);
        
                    //消息内容
                    String message = "Hello World";
        
                    //6.发送消息
                    channel.basicPublish("","queue1",null,message.getBytes());
                    System.out.println("消息已发送");
        
                } catch (TimeoutException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                }finally {
                    //7. 关闭通道
                    if(channel != null && channel.isOpen()){
                        try {
                            channel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        } catch (TimeoutException e) {
                            e.printStackTrace();
                        }
                    }
        
                    //8. 关闭连接
                    if(connection != null && connection.isOpen()){
                        try {
                            connection.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
        ```
        
    - 消费者
      
        ```java
        import com.rabbitmq.client.*;
        import java.io.IOException;
        import java.util.concurrent.TimeoutException;
        
        public class Consumer {
            public static void main(String[] args) {
                // 1. 创建连接工厂
                ConnectionFactory factory = new ConnectionFactory();
                // 2. 设置连接属性
                factory.setHost("175.178.15.23");
                factory.setUsername("admin");
                factory.setPassword("admin");
        
                Connection connection = null;
                Channel channel = null;
        
                try {
                    // 3.从连接工厂获取连接
                    connection = factory.newConnection("Consumer");
        
                    // 4.从连接中创建通道
                    channel = connection.createChannel();
        
                    /**
                     * 5.声明（创建）队列
                     * 如果队列不存在，才会创建
                     * RabbitMQ不允许声明两个队列名相同，属性不同的队列，否则会报错
                     *
                     * queueDeclare参数说明
                     * @param queue 队列名称;
                     * @param durable 队列是否持久化
                     * @param exclusive 是否排他，即是否私有的，如果为true，会对当前队列加锁，其他通道不能访问没并且在连接关闭时自动删除，不受是否持久化和自动删除的影响
                     * @param autoDelete 是否自动删除，当最后一个消费者断开连接之后是否自动删除
                     * @param arguments 队列参数，设置队列的有效期，消息最大长度，队列中所有消息的生命周期等等
                     */
                    channel.queueDeclare("queue1",false,false,false,null);
        
                    //6.定义收到消息后的回调函数
                    DeliverCallback callback = new DeliverCallback() {
                        public void handle(String s, Delivery delivery) throws IOException {
                            System.out.println("收到消息：" + new String(delivery.getBody(),"UTF-8"));
                        }
                    };
        
                    //7.监听队列
                    channel.basicConsume("queue1", true, callback, new CancelCallback() {
                        public void handle(String s) throws IOException {
        
                        }
                    });
                    System.out.println("开始接收消息");
                    System.in.read();
        
                } catch (TimeoutException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                }finally {
                    //8. 关闭通道
                    if(channel != null && channel.isOpen()){
                        try {
                            channel.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        } catch (TimeoutException e) {
                            e.printStackTrace();
                        }
                    }
        
                    //9. 关闭连接
                    if(connection != null && connection.isOpen()){
                        try {
                            connection.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
        ```
    
2. Spring中使用
3. SpringBoot使用

## 6. AMQP协议

<aside>
💡 AMQP（Advanced Message Queuing Protocol）高级队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计

</aside>

1. AMQP结构
    - **Transport Layer:** 位于最低层，主要传输二进制数据流， 提供帧的处理、信道复用、错误检测和数据表示等。
    - Session Layer: 位于中间层，主要负责将客户端的命令发送给服务器，再将服务端的应答返回给客户端，主要为客服端与服务器之间的通信提供可靠性同步机制和错误处理。
    - Module Layer: 位于最高层，主要定义了一些供客户端调用的命令，客户端可以利用这些命令实现自己的业务逻辑。
    
2. AMQP生产者流转过程
   
    ![AMQP生产者流转过程.png](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/63c95b7f-11c1-494c-8f6d-080d9eeb40b2/AMQP%E7%94%9F%E4%BA%A7%E8%80%85%E6%B5%81%E8%BD%AC%E8%BF%87%E7%A8%8B.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220326%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220326T124424Z&X-Amz-Expires=86400&X-Amz-Signature=f3522422e886050c577e6d0393c5770b8308d122fc5c3991fb6cc41bfac9606b&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22AMQP%25E7%2594%259F%25E4%25BA%25A7%25E8%2580%2585%25E6%25B5%2581%25E8%25BD%25AC%25E8%25BF%2587%25E7%25A8%258B.png%22&x-id=GetObject)
    
3. AMQP消费者流转过程
   
    ![AMQ消费者流转过程.png](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/41b3e503-12b0-4fbe-b8ee-f016c14abe42/AMQ%E6%B6%88%E8%B4%B9%E8%80%85%E6%B5%81%E8%BD%AC%E8%BF%87%E7%A8%8B.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220326%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220326T124510Z&X-Amz-Expires=86400&X-Amz-Signature=28f73ed74b23e2c0a16fddafc4fe063fd53db82de24b221bd5f84a6ddf40b183&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22AMQ%25E6%25B6%2588%25E8%25B4%25B9%25E8%2580%2585%25E6%25B5%2581%25E8%25BD%25AC%25E8%25BF%2587%25E7%25A8%258B.png%22&x-id=GetObject)
    

## 7. RabbitMQ核心概念

![rabbitmq.jpg](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/06f6a736-0de2-46a5-9bf1-3224bf9d5b6c/rabbitmq.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220326%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220326T124541Z&X-Amz-Expires=86400&X-Amz-Signature=251151734388ee77cb4915a07d04a075f126669d1183cc544e982a37dd6829a1&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22rabbitmq.jpg%22&x-id=GetObject)

1. Producer：生产者，投递消息的一方。生产者创建消息，然后发布到RabbitMQ中
2. 消息：分为两个部分，消息体和附加信息
    - 消息体：在实际应用中，消息体一般是一个带有业务逻辑结构的数据，比如一个JSON字符串。当然可以进一步对这个消息体进行序列化操作
    - 附加信息：用来表述这条消息，比如目标交换器的名称、路由键和一些自定义属性等
3. Broker：消息中间件的服务节点
4. Virtual Host：虚拟主机，表示一批交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。
5. Channel：频道或信道，是建立在Connection连接之上的一种轻量级的连接。
   
    大部分的操作是在Channel这个接口中完成的，包括定义队列的声明queueDeclare、交换机的声明 exchangeDeclare、队列的绑定queueBind、发布消息basicPublish、消费消息basicConsume等。
    
6. **RoutingKey：**路由键。生产者将消息发给交换器的时候，一般会指定一个 RoutingKey，用来指定这个消息的路由规则
   
    RoutingKey需要与交换器类型和绑定键 (BindingKey) 联合使用
    
7. Exchange：交换器，生产者将消息发送到 Exchange (交换器，通常也可以用大写的“X”来表示)， 由交换器将消息路由到一个或者多个队列中。如果路由不到，或返回给生产者，或直接丢弃
    - fanout：扇型交换机 它会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中
    - direct：直连交换机 它会把消息路由到那些 BindingKey 和 RoutingKey完全匹配的队列中
    - topic：主题交换机 与direct类似，但它可以通过通配符进行模糊匹配
    - headers：头交换机 不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中的 headers 属性进行匹配
8. Queue：队列，是RabbitMQ的内部对象，用于存储消息
9. Binding：绑定，RabbitMQ 中通过绑定将交换器与队列关联起来，在绑定的时候一般会指定一个绑定键 ( BindingKey ) ，这样 RabbitMQ 就知道如何正确地将消息路由到队列了。
10. Consumer：消费者，就是接收消息的一方。消费者连接到 RabbitMQ 服务器，并订阅到队列上
    
    当消费者消费一条消息时，只是消费消息的消息体(payload ) 。在消息路由的过程中，消息的标签会丢弃，存入到队列中的消息只有消息体，消费者也只会消费到消息体，也就不知道消息的生产者是谁，当 然消费者也不需要知道