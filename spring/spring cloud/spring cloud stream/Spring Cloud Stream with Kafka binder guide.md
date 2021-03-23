# Spring Cloud Stream with Kafka binder

## Example scenario

- Producer 서버에 message가 담긴 POST API를 호출하면
- Producer가 메시지를 발행한다. (Producer는 한개)
- 두개의 Consumer가 메시지를 읽어간다.(병렬처리 아님)

<br>

## Setting

```tex
⚙ 개발 환경
- IDE : Intellij
- Spring Boot version : 2.4.4
- Spring Cloud version : 2020.0.2
```

<br>

### Docker Compose

`docker-compose.yml` 을 작성하여 Kafka Broker를 띄운다.

테스트이므로 Broker는 1개만 띄운다.(Topic은 미리 생성하며 TopicPartition도 1개)

```yml
# docker-compose.yml
version: '2'
services:
  zookeeper:
    container_name: local-zookeeper
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2181:2181"
  kafka:
    container_name: local-kafka
    image: wurstmeister/kafka:2.12-2.3.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 127.0.0.1
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_CREATE_TOPICS: "test_topic:1:1"     # <Topic명>:<Partition개수>:<Replica개수>
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```





### Spring Cloud Stream Setting

1. Spring Cloud Stream dependency를 포함한 프로젝트 생성

2. Kafka dependency 추가

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.4</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <groupId>me.gnoyes</groupId>
    <artifactId>msg-producer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>msg-producer</name>
    <description>Demo project for Spring Boot</description>
    
    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>2020.0.2</spring-cloud.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream</artifactId>
        </dependency>

        <!--    Kafka        -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-kafka</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream</artifactId>
            <scope>test</scope>
            <classifier>test-binder</classifier>
            <type>test-jar</type>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

<br>



## part of Producer

MsgProducerApplication.java

```java
@SpringBootApplication
@EnableBinding(Source.class)
public class MsgProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(MsgProducerApplication.class, args);
    }
}
```

`@EnableBinding(Source.class)` 어노테이션을 적용시켜준다.

`@EnableBinding(Source.class)` 은 Spring Cloud Stream에 해당 서비스를 Binder에 Binding 하도록 명시해주는 것으로 Source 클래스에 정의된 채널들을 이용해 Binder(미들웨어의 브로커)와 통신하게된다

<br><br>

Source.java

```java
public interface Source {
    String OUTPUT = "output-channel";

    @Output(Source.OUTPUT)
    MessageChannel publish();
}
```

Source(output, outbounding)에는 `output-channel`이라는 바인딩 이름과 채널이 정의되어있다.

메시지를 produce하는 경우, 즉 source(output, outbounding)의 경우 MessageChannel을 사용한다(반대는 SubscribableChannel) 

<br><br>

MessageProducer.java

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import java.util.logging.Logger;

@Component
public class MessageProducer {
    private final Logger logger = Logger.getLogger(this.getClass().getSimpleName());
    
    @Autowired
    Source source;

    @SendTo(Source.OUTPUT)
    public void send(String msg) {
        logger.info("sending message ... msg is " + msg);
        source.publish().send(MessageBuilder.withPayload(msg).build());
    }
}

```

`@SendTo(Source.OUTPUT)` 어노테이션으로 사용할 Binding을 명시하고

`MessageBuilder`를 통해 메시지를 build한 뒤 source를 통해 메시지를 보낸다

<br><br>

application.yml

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        output-channel:
          destination: test_topic
          contentType: application/json
```

binder는 Kafka를 사용하며 broker의 주소는 localhost:9092

binding은 (source에서 정의한) output-channel을 추가한 뒤 destination(=Topic)을 명시해준다

<br><br>

## part of Consumer

MsgConsumerApplication.java(1, 2)

```java
// MsgConsumer1Application.java
@SpringBootApplication
@EnableBinding(Sink.class)
public class MsgConsumer2Application {

    public static void main(String[] args) {
        SpringApplication.run(MsgConsumer2Application.class, args);
    }
}

// MsgConsumer2Application.java
@SpringBootApplication
@EnableBinding(Sink.class)
public class MsgConsumer2Application {

    public static void main(String[] args) {
        SpringApplication.run(MsgConsumer2Application.class, args);
    }
}
```

Consumer 역할을 하는 application에서도 마찬가지로 `@EnableBinding(Sink.class)`어노테이션을 통해 Binder와 Binding함을 명시한다

<br><br>

Sink.class

```java
public interface Sink {
    String INPUT = "input-channel";

    @Input(Sink.INPUT)
    SubscribableChannel subscribe();
}
```

Sink(input, inbounding)에는 `input-channel`이라는 바인딩 이름과 SubscribableChannel 이 정의되어있다.

<br><br>

MessageConsumer.java

```java
@Component
public class MessageConsumer {

    @StreamListener(Sink.INPUT)
    public void receive(Message<?> msg) {
        System.out.println("I'm consumer, here's msg : " + msg.getPayload());
    }
}
```

`@StreamListener(Sink.INPUT)`으로 Binding을 명시하여 채널을 통해 메시지를 받아온다

<br><br>

application.yml

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        input-channel:
          group: group1
          destination: test_topic
          contentType: application/json
# ㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡ
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        input-channel:
          group: group2
          destination: test_topic
          contentType: application/json
```

Consumer 측도 Producer 측과 마찬가지로 Binder와 Binding을 명시하지만 차이점은 group이다.

Kafka에선 Consumer Group라고 있는데, 간단히 설명하면

같은 Group에 여러 Consumer가 있는 경우, 해당 Topic에 대해서 (거의 균등하게) 메시지가 분담된다 -> offset을 공유한다

다른 Group의 경우 offset을 공유하지 않는다. 즉, 같은 메시지를 여러곳에서 읽어갈 수 있다.

<br><br>

## 사용 시 주의 사항

- Event Message 구조
- Kafka 운영시 관련 설정
  - Broker 수, Partition 수, Replication factor, Retention period 등등