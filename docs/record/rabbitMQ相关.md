
### rabbitMQ 工作模式


#### 普通 (1-发送 1-接收)

普通收发模式(P2P)

```java
public class NormalWorkQueueMode {

    private static String QUEUE_NAME = "normal_queue";

    static class Sender {
        public static void main(String[] args) throws Exception {
            new Sender().send();
        }

        public void send() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            String message = "hello world";
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            channel.close();
            connection.close();
        }
    }

    static class Receiver {
        public static void main(String[] args) throws Exception {
            new Receiver().start();
        }

        public void start() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            DefaultConsumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    //交换机
                    String exchange = envelope.getExchange();
                    //消息id，mq在channel中用来标识消息的id，可用于确认消息已接收
                    long deliveryTag = envelope.getDeliveryTag();
                    // body 即消息体
                    String msg = new String(body, "utf-8");
                    System.out.println(" [x] received : " + msg + "!");
                }
            };
            channel.basicConsume(QUEUE_NAME, true, consumer);
        }
    }
}

```

#### 订阅模式

1、一个生产者多个消费者
2、每个消费者都有一个自己的队列
3、生产者没有将消息直接发送给队列，而是发送给exchange(交换机、转发器)
4、每个队列都需要绑定到交换机上
5、生产者发送的消息，经过交换机到达队列，实现一个消息被多个消费者消费
例子：注册->发邮件、发短信

+ 广播模式(Fanout)
```java
package mq.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;

public class FanoutMode {

    private static String EXCHANGE_NAME = "normal_exchange";
    private static String QUEUE_NAME = "fanout_queue";

    static class Sender {
        public static void main(String[] args) throws Exception {
            new FanoutMode.Sender().send();
        }

        public void send() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
            String message = "hello world";
            channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
            channel.close();
            connection.close();
        }
    }
    static class Receiver {
        public static void main(String[] args) throws Exception {
            new Receiver().start();
        }

        public void start() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");
            DefaultConsumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg = new String(body, "utf-8");
                    System.out.println(" [x] received : " + msg + "!");
                }
            };
            channel.basicConsume(QUEUE_NAME, true, consumer);
        }
    }
}

```


```
1、publish/subscribe与work queues有什么区别。

区别：

1）work queues不用定义交换机，而publish/subscribe需要定义交换机。

2）publish/subscribe的生产方是面向交换机发送消息，work queues的生产方是面向队列发送消息(底层使用默认交换机)。

3）publish/subscribe需要设置队列和交换机的绑定，work queues不需要设置，实际上work queues会将队列绑定到默认的交换机 。

相同点：

所以两者实现的发布/订阅的效果是一样的，多个消费端监听同一个队列不会重复消费消息。

2、实际工作用 publish/subscribe还是work queues。

建议使用 publish/subscribe，发布订阅模式比工作队列模式更强大（也可以做到同一队列竞争），并且发布订阅模式可以指定自己专用的交换机。

```



+ 路由模式

P：生产者，向Exchange发送消息，发送消息时，会指定一个routing key。

X：Exchange（交换机），接收生产者的消息，然后把消息递交给 与routing key完全匹配的队列

```java
public class RoutingMode {

    private static String EXCHANGE_NAME = "normal_routing";
    private static String QUEUE_SMS_NAME = "routing_sms_queue";
    private static String QUEUE_EMAIL_NAME = "routing_email_queue";


    static class Sender {
        public static void main(String[] args) throws Exception {
            new Sender().send();
        }

        public void send() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);
            String message = "hello world";
            channel.basicPublish(EXCHANGE_NAME, "sms", null, message.getBytes());
            channel.close();
            connection.close();
        }
    }

    static class SmsReceiver {
        public static void main(String[] args) throws Exception {
            new SmsReceiver().start();
        }

        public void start() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_SMS_NAME, false, false, false, null);
            channel.queueBind(QUEUE_SMS_NAME, EXCHANGE_NAME, "sms");
            DefaultConsumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg = new String(body, "utf-8");
                    System.out.println(" [x] received : " + msg + "!");
                }
            };
            channel.basicConsume(QUEUE_SMS_NAME, true, consumer);
        }
    }

    static class EmailReceiver {
        public static void main(String[] args) throws Exception {
            new EmailReceiver().start();
        }

        public void start() throws Exception {
            ConnectionFactory factory = ConnectionFactoryUtil.getConnectionFactory();
            Connection connection = factory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_EMAIL_NAME, false, false, false, null);
            channel.queueBind(QUEUE_EMAIL_NAME, EXCHANGE_NAME, "email");
            DefaultConsumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg = new String(body, "utf-8");
                    System.out.println(" [x] received : " + msg + "!");
                }
            };
            channel.basicConsume(QUEUE_EMAIL_NAME, true, consumer);
        }
    }

}

```