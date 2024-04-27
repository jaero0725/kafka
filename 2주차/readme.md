### 목차

-   파티셔너
-   메시지 전송방식 

---

## **파티셔너**

[##_Image|kage@bw2tIt/btsGZLHmFhy/EEgOvA5HwP3j3e6aafYlWK/img.png|CDM|1.3|{"originWidth":1173,"originHeight":515,"style":"widthContent","caption":"카프카 프로듀서의 동작"}_##]

-   프로듀서가 전송하려는 메시지들은 프로듀서의 send() 메소드를 통해 시리얼라이저, 파티셔너를 거쳐 카프카로 전송된다.
-   이 과정을 파티셔닝이라고 한다.
-   카프카의 토픽은 성능 향상을 위해 병렬 처리가 가능하도록 파티션으로 나누고, 각 파티션에 프로듀서가 전송한 메시지가 로그 세그먼트에 저장된다.
-   프로듀서는 토픽으로 메시지를 보낼 때 해당 토픽의 어느 파티션으로 메시지를 보내야 할지를 결정 하는데, 이때 사용하는 것이 "파티셔너(partitioner)"입니다.
-   기본적으로 메시지(레코드)의 키를 Hash처리 해 파티션을 구하는 방식을 사용하는데, 메시지의 키 값이 동일하면 해당 메시지들은 모두 같은 파티션으로 전송되는 방식으로 구현된다.

---

#### **배치전송 옵션**

| buffer.memory | 버퍼메모리 옵션, 기본값 : 32mb | 기본값 32mb |
| --- | --- | --- |
| batch.size | 메세지들을 묶는 단위, 배치 크기 옵션 기본값 16kb    \- 배치 크기 ,  배치 다 차면 전송 | 기본값 16kb |
| linger.ms | 메모리에서 대기하는 메지들의 최대 대기시간을 설정하는 옵션, 단위는 (ms) 기본값 0 (즉시전송)   \- 전송 대기 시간, 대기 시간없으면 배치 바로 전송함.        | 기본 값 0 |

\- 처리량을 높일지, 지연 없는 전송을 해야할지에 따라 배치전송 옵션을 잘 설정 하면 될 것.

처리량 높이기 -> batch.size++ , liner.ms++  / 압축방식 - gzip, zstd   
지연없는 전송 -> bath.size--, liner.ms--  / 압축방식 : lz4, snappy 

## **파티셔닝 방식  
**

\- 메시지의 key값이 지정된 메시지는 해시 알고리즘을 통해 파티션에 매핑이됩니다.   
\- 메시지의 키를 이용해 카프카로 메시지를 전송할때 의도와 다르게 메시지 전송이 이루어질 수있으므로, 파티션 수를 변경하지 않는 것을 권장   
\- 키값이 null일 경우 아래 2가지 방식중 하나의 방식으로 파티셔닝이 이루어집니다.  

### **1\. 라운드 로빈 방식**

[##_Image|kage@sBzWH/btsGZKuP2WQ/LuM1ZUPIK5Rnd5W94li9VK/img.png|CDM|1.3|{"originWidth":561,"originHeight":446,"style":"widthContent","caption":"라운드로빈 방식 - 레코드 5개"}_##]

\- 메시지 각 파티션에 순서대로 하나씩 넣는다.  
\- 이렇게 하면 각 파티션에 동일한 수의 메시지가 들어가게 되고, 자연스럽게 각 파티션을 가져가는 컨슈머도 동일한 부하를 가지도록 한다.  
\- batch.size가 3으로 설정되고, 5개의 메시지가 존재하는 경우, batch.size에 도달하지 않았기 대문에 메시지는 브로커로 전달이 되지 않는다. (불필요한 지연 발생)

### **2\. 스티키 파티셔닝 방식**

[##_Image|kage@bdOL08/btsG0bMlXP2/kCyIKWJFhfuCd5q8AdyTY0/img.png|CDM|1.3|{"originWidth":541,"originHeight":453,"style":"widthContent"}_##]

\- 스티키 파티셔닝 전략은 카프카 2.4버전 부터 default로 지정된 전략입니다.  
\- 라운드 로빈 전략에서는 배치 전송을 위한 필요 레코드 수(배치 사이즈)를 채우지 못해 카프카 배치전송을 하지 못했던 것과 달리, 하나의 파티션에 레코드 수를 먼저 채워서 카프카로 빠르게 배치 전송하는 전략을 말합니다. 

\* 스티키 파티셔닝 전략을 적용해 약 30% 이상 지연시간 감소, 프로듀서 CPU 사용률도 줄어듬

[##_Image|kage@L4rur/btsG0Ja2bKS/JIts8O33suKE2fknWgPHDK/img.png|CDM|1.3|{"originWidth":718,"originHeight":446,"style":"alignCenter","caption":"https://www.conduktor.io/kafka/producer-default-partitioner-and-sticky-partitioner"}_##]

\- 메시지의 순서가 그다지 중요하지 않은 경우라면 스티키 파티셔닝 전략을 적용하기를 권장 

---

## **카프카에서 메시지 전송 구현방식** 

#### \- 적어도 한 번 전송(at-least-once) : 중복 메시지 발생 가능  
\- 최대 한번 전송 (at-most-once) : 메시지 누락 가능  
\- 중복없는 전송 : 성능 저하 있음  
\- 정확히 한번 전송 : 성능 저하 있음

## **1\. 적어도 한번 전송 (중복 가능성O,** **메시지 손실 가능성****X)**

[##_Image|kage@bbTqGl/btsGZ1C84pn/EvORdw0zwfLJZ4be0B7LBk/img.png|CDM|1.3|{"originWidth":621,"originHeight":442,"style":"alignCenter"}_##]

**\- 브로커로부터 메시지 수신 응답이 올 때까지, 메시지를 계속 재전송하는 전략  
\- 일부 메시지 중복이 발생할 수 있지만, 최소한 하나의 메시지는 반드시 보장한다.  
\- 카프카는 기본적으로 이와 같은 적어도 한 번 전송 방식을 기반으로 동작한다.** 

## **2\. 최대 한번 전송** **(중복 가능성X, 메시지 손실 가능성O)**

[##_Image|kage@b3iuSP/btsGYozR8UT/3gCTUr4dnL4VOUQXa2fLoK/img.png|CDM|1.3|{"originWidth":611,"originHeight":466,"style":"alignCenter"}_##]

**\- 브로커의 메시지 수신 여부에 상관없이 메시지를 보내는 전략  
\- 수신 신호를 받지 못해도, 별도의 처리 없이 바로 다음 메시지를 보낸다.   
\- 메시지 중복 가능성을 회피하기 위해 재전송 하지 않는다. \* 일부 메시지 손실을 감수하고 중복전송을 하지 않음.  
\- 유실이 있어도 높은 처리량을 필요로하는 대량 로그 수집, IoT환경에서 사용**

## **3\. 중복 없는 전송**

[##_Image|kage@bqWh4Q/btsGY1YuJUz/NOWhIydjH4bjRDGmkQ0ogK/img.png|CDM|1.3|{"originWidth":747,"originHeight":545,"style":"alignCenter","caption":"중복없는 전송"}_##]

**\- 적어도 한번 전송 + PID를 추가하여 중복 여부를 확인 하는 전략  
\- 프로듀서의 경우 메시지를 보내고, 브로커로부터 수신 응답이 없으면 재전송 하는 로직 동일 - 여기서 주고 받는 메시지에는 프로듀서 ID와 메시지 번호가 포함되어 있다.  
\- 따라서 메시지를 중복 전송하여도, 브로커는 해당정보로 메시지 중복여부를 식별한다. 중복 메시지일 경우 별도 동작없이 ACK신호를 프로듀서로 전송한다.**

**\# PID와 메시지 번호(시퀀스 번호)**  
\- 프로듀서가 중복없는 전송을 시작하면, 프로듀서는 고유한 PID를 할당받게 되고, 이 PID와 메시지에 대한 번호를 메시지의 헤더에 포함해 메시지를 전송한다.  
\- 브로커에서는 각 메시지마다 PID값과 시퀀스 번호를 메모리에 유지, 이 정보를 이요해 브로커에 기록된 메시지들의 중복 여부를 판단  
\- PID, 메시지 번호는 프로듀서에 의해 자동 생성  
\- 중복을 피하기 위해 브로커의 메모리에 유지되고, 리플리케이션 로그에도 저장된다.  
\- 브로커 장애 상황이 발생하여 리더가 변경되어도 새로운 리더가 PID와 시퀀스 번호를 정확히 알 수 있기 때문에 중복 없는 메시지 전송이 가능하다.

**\# 중복을 피하기 위한 메시지 비교 동작에는 오버헤드가 존재함.** 

> After much thought, we settled on a design that involves minimal overhead per transaction (~1 write per partition and a few records appended to a central transaction log). This shows in the measured performance of this feature. For 1 KB messages and transactions lasting 100 ms, the producer throughput declines only by 3%, compared to the throughput of a producer configured for at least once, in-order delivery (acks=all, max.in.flight.requests.per.connection=1), and by 20% compared to the throughput of a producer configured for most once delivery with no ordering guarantees (acks=1, max.in.flight.requests.per.connection=5), which is the current default.  
>   
> **적어도 한 번, 순서대로 전송을 위해 구성된 프로듀서와 비교하여 프로듀서의 처리량은 오직 3%만 감소**하였고, **최대 한 번 전송하며 순서대로 전송하지 않도록 구성된 프로듀서와 비교하여 20% 감소**

> [https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)

\- 메시지에 단순한 숫자 필드만 추가하는 방법으로 구현해서 오버헤드가 생각보다 크진 않다.   
\- 중복없는 전송을 적용한후 기존 대비 최대 약 20% 성능 감소가 발생함.  
\- 따라서 프로듀서 전송 성능에 민감하지 않는다면, 이 방식을 적용하는 것을 권장

**\# 중복없는 전송을 위한 프로듀서 설정  
**

| 프로듀서 옵션 | 값 | 설명 |
| --- | --- | --- |
| enable.idempotence | true | 프로듀서가 중복 없는 전송을 허용할지 결정하는 옵션.   기본 값은 false. true로 하는 경우, 아래의 옵션도 적절하게 설정해줘야 함. |
| max.in.flight.request.per.connection | 1 ~ 5 | 프로듀서 ACK와 관련된 설정. all로 설정해야함. |
| acks | all | 프로듀서 ACK와 관련된 설정. all로 설정해야함.  |
| retries | 5 | 프로듀서 재전송과 관련된 설정. 0보다 큰 값으로 설정해야함.  |

\- ack = 0 : 서버 응답을 기다리지 않음  
\- ack = 1 : 파티션의 리더에 저장되면 응답 받음. 리더 장애시 메시지 유실 가능  
\- ack = all or -1  : 모든 리플리카에 저장되면 응답 받음.

**\# 중복없는 전송 구현**

```
# producer.config
enable.idempotence=true
max.in.flight.requests.per.connection=5
retries=5
acks=all
```

```
$ kafka-console-producer.sh --bootstrap-server localhost:9092 \
--topic topic-4 \
--producer.config ~/producer.

>> a
>> b
>> c
>> d
```

\- 발송 이후 스냅샷 파일 생성   
\- 스냅샷 파일은 브로커가 PID, SEQ 번호를 주기적으로 저장하는 파일

```
$ kafka-dump-log.sh --files 00000000000000000007.snapshot --print-data-log

Dumping 00000000000000000007.snapshot
producerId: 1002 
producerEpoch: 0 
coordinatorEpoch: -1 
currentTxnFirstOffset: None 
lastTimestamp: 1669689243854 
firstSequence: 4 
lastSequence: 6 
lastOffset: 6 
offsetDelta: 2 
timestamp: 1669689243854
```

\- dump 명령어로 파일 확인  
\- pid : 1002 / 시퀀스 번호 firstSequence :5, lastSequence :6 확인

## **4\. 정확히 한번 전송**

\- 중복없는 정송은 '정확히 한번 전송'의 일부 기능으로 이해 할 수 있음.   
\- '정확히 한번 전송'은 별도의 프로세스가 있는데  트랜잭션 처리와 같이 전체적인 프로세스 처리를 의미한다.

 프로듀서가 카프카로 '정확히 한번 전송' 방식으로 메세지를 보낸다면, 프로듀서가 보내는 메세지들은 원자적으로 처리되어 전송에 성공하거나 실패하게 된다. 이것은 브로커에 있는 '트랜잭션 코디네이터'라는 것이 도와준다. 트랜잭션 코디네이터는 다음 역할을 한다.

**\# 트랜잭션 코디네이터**  
**\- 서버 측에는 프로듀서에 의해 전송된 메시지를 관리하고, 커밋 또는 중단 등을 표시하는 역할, 해당 트랜잭션 전체를 관리**

1) Transaction ID, Producer ID를 맵핑해서 보관함.  
2) 프로듀서에 의해 전송된 메세지를 관리하며, 메세지의 상태를 표시한다.   
3) \_\_transaction\_state에 트랜잭션 로그를 저장한다.

\- 트랜잭션 관리를 위해서 브로커는 컨슈머를 \_\_consumer\_offset 토픽에 관리하는 것처럼 트랜잭션 로그를 \_\_transaction\_state에 저장한다.   
\- \_\_transaction\_state 토픽은 기본값으로 50개의 파티션, 3개의 replication factor로 생성된다. 

```
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class ExactlyOnceProducer {
    public static void main(String[] args) {
        String bootstrapServers = "peter-kafka01.foo.bar:9092";
        Properties props = new Properties();
        props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.setProperty(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // 정확히 한번 전송을 위한 설정
        props.setProperty(ProducerConfig.ACKS_CONFIG, "all"); // 정확히 한번 전송을 위한 설정
        props.setProperty(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, "5"); // 정확히 한번 전송을 위한 설정
        props.setProperty(ProducerConfig.RETRIES_CONFIG, "5"); // 정확히 한번 전송을 위한 설정
        props.setProperty(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "peter-transaction-01"); // 정확히 한번 전송을 위한 설정

        Producer<String, String> producer = new KafkaProducer<>(props);

        producer.initTransactions(); // 프로듀서 트랜잭션 초기화
        producer.beginTransaction(); // 프로듀서 트랜잭션 시작
        try {
            for (int i = 0; i < 1; i++) {
                ProducerRecord<String, String> record = new ProducerRecord<>("peter-test05", "Apache Kafka is a distributed streaming platform - " + i);
                producer.send(record);
                producer.flush();
                System.out.println("Message sent successfully");
            }
        } catch (Exception e){
            producer.abortTransaction(); // 프로듀서 트랜잭션 중단
            e.printStackTrace();
        } finally {
            producer.commitTransaction(); // 프로듀서 트랜잭션 커밋
            producer.close();
        }
    }
}

//https://github.com/onlybooks/kafka2/blob/main/chapter5/%EC%98%88%EC%A0%9C/%EC%98%88%EC%A0%9C%205-3
```

\- 여기서 중요한 것은 중복 없는 전송과 정확히 한 번 전송의 옵션 설정에서 큰 차이점은 TRANSACTIONAL\_ID\_CONFIG이다. 

\- 해당 옵션은 실행하는 프로듀서 프로세스마다 고유한 아이디로 설정해야한다. 다시 말해, n개의 프로듀서가 존재하면 n개의 다른 아이디로 설정해야한다는 뜻.

### **단계별 동작  
  
**

**1) 트랜잭션 코디네이터 찾기**

[##_Image|kage@u97cj/btsGY3ICxPV/pnax9Hk2uqUWf57EhB6kT0/img.png|CDM|1.3|{"originWidth":697,"originHeight":175,"style":"alignCenter"}_##]

\- 가장 먼저 트랜잭션 코디네이터를 찾습니다.  
\- 프로듀서는 브로커에게 FindCoordinatorRequest를 보내어 트랜잭션 코디네이터 위치를 찾는다.  
\- 트랜잭션 코디네이터가 있는 브로거, 프로듀서가 전송하는 메시지를 받는 브로커 가 다르다!

**2) 프로듀서 초기화**

[##_Image|kage@Bumqq/btsGY6L5AUG/rSDXpjLNbrMYD7GXkKvXo0/img.png|CDM|1.3|{"originWidth":554,"originHeight":271,"style":"alignCenter","width":513,"height":251}_##]

```
producer.initTransactions(); // 프로듀서 트랜잭션 초기화
```

\- producer는 initTransactions() 메소드를 이용해 트랜잭션 전송을 위한 initPidRequst를 트랜잭션 코디네이터에게 보냄.  
\- 이때 TID가 설정된 경우 initPidRequest와 함께 TID를 전송  
\-  트랜잭션 코디네이터는 TID(transaction id), PID(producer id)를 매핑하고 트랜잭션 로그에 기록  
\- PID에포크를 한 단계 올리는 동작을 하고 PID에포크가 올라감에 따라 이전의 동일한 PID와 이전 에포크에 대한 쓰기 요청을 무시한다. (새로운 트랜잭션과 이전 트랜잭션을 구분하는 용도) 

**3) 트랜잭션 시작**

[##_Image|kage@cxaH2o/btsGY50I1IF/Gr2DjCUImjCfWTazpXPW80/img.png|CDM|1.3|{"originWidth":544,"originHeight":302,"style":"alignCenter","width":524,"height":291}_##]

```
producer.beginTransaction(); // 프로듀서 트랜잭션 시작
```

\- 내부적으로 트랜잭션 시작을 기록   
\- 트랜잭션 코디네이터는 첫 번째 레코드가 전송될 때까지 트랜잭션이 시작된 것이 아님.

**4) 트랜잭션 상태 추가**

[##_Image|kage@da74vR/btsGZkXEyuU/MvVV2VzjcklIFlL9HKQlz0/img.png|CDM|1.3|{"originWidth":568,"originHeight":349,"style":"alignCenter"}_##]

\- TID, P0의 정보가 트랜잭션 로그에 기록  
\- 트랜잭션의 현재 상태를 진행중으로 표시  
\- 만약 트랜잭션 로그에 추가 되는 첫 번째 파티션이라면, 트랜잭션 코디네이터는 해당 트랜잭션에 대한 타이머 시작  
\- 기본값 1분 동안 트랜잭션 상태에 대한 업데이트가 없다면 해당 트랜잭션을 실패로 처리 (타임아웃)

**5) 메시지 전송** 

[##_Image|kage@cGVppx/btsGZKuSyvx/lmWRbcAiNIAF5iXPsNGYo0/img.png|CDM|1.3|{"originWidth":553,"originHeight":341,"style":"alignCenter"}_##]

\- 프로듀서는 대상 토픽의 파티션으로 메시지를 전송한다.  
\- 해당 메시지는 PID, 에포크, 시퀀스 번호가 함께 전송됨.

**6) 트랜잭션 종료 요청**

[##_Image|kage@bRsPlT/btsGYQbFOH9/ErkXjAXXJKP1JKh8khhl21/img.png|CDM|1.3|{"originWidth":585,"originHeight":345,"style":"alignCenter"}_##]

```
 catch (Exception e){
    producer.abortTransaction(); // 프로듀서 트랜잭션 중단
    e.printStackTrace();
} finally {
    producer.commitTransaction(); // 프로듀서 트랜잭션 커밋
    producer.close();
}
```

\- 메시지 전송을 완료한 프로듀서는 commitTransaction() 메소드 또는 abortTransaction() 메소드를 반드시 호출해야 한다.  
\- 해당 메소드 호출을 통해 트랜잭션이 완료됨을 트랜잭션 코디네이터에게 알립니다.  
\- 완료됨을 알게된 트랜잭션 코디네이터는 두 단계의 커밋 과정을 시작한다. 

**7) 커밋과정 8) 트랜잭션완료**

[##_Image|kage@MTl3e/btsG0Tq7las/KKoSFJPTRrrLMTNl61hnnK/img.png|CDM|1.3|{"originWidth":784,"originHeight":380,"style":"alignCenter","caption":"커밋과정 -&gt; 트랜잭션 완료"}_##]

**(1) 해당 트랜잭션에 대한 PreapareCommit or PrepareAbort를 기록한다.**   
\- commitTransaction() 이 오면 이 트랜잭션이 종료되었음을 트랜잭션 코디네이터에게 알려준다.  
\- 트랜잭션 코디네이터는 이 신호를 받고, 해당 TID의 파티션 상태를 'Prepare' 로 변경하고, \_\_transaction\_state에 기록.

**(2)이 파티션을 가지고 있는 브로커에게 WriteTxnMarkerRequest를 요청한다.**   
\- 이 요청을 받게 되면 해당 브로커의 파티션의 세그먼트 파일에는 커밋되었다는 컨트롤 메시지가 기록된다.  
\- 컨트롤 메시지는 컨슈머가 메시지를 가져갈 때, 트랜잭션이 끝났는지 알려주는 용도로 사용된다.  
\- 트랜잭션 커밋이 끝나지 않은 메시지는 컨슈머에게 반환 하지 않음.  
\- 오프셋의 순서 보장을 위해 트랜잭션 성공 또는 실패를 나타내는 LSO(Last Stable Offset)이라는 오프셋을 유지한다.

**(3) 브로커는 트랜잭션 코디네이터에 이 transaction.id의 파티션이 commmit 된 것을 \_\_transaction\_state에 기록 하고 정상적으로 커밋된 것을 프로듀서에게 응답해준다.**

---

**Reference**

 [실전 카프카 개발부터 운영까지 | 고승범 - 교보문고

실전 카프카 개발부터 운영까지 | 아파치 카프카의 공동 창시자 준 라오(Jun Rao)가 추천한 책!국내 최초이자 유일한 컨플루언트 공인 아파치 카프카 강사(Confluent Certified Trainer for Apache Kafka)와 공

product.kyobobook.co.kr](https://product.kyobobook.co.kr/detail/S000001932756)
