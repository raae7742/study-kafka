# CHAPTER 4. 카프카 상세 개념
# 토픽과 파티션

토픽을 사용함에 있어 발생하는 여러 가지 고려사항을 짚어보자.

<br>

## 적정 파티션 개수

> 토픽 생성 시 파티션 개수 고려사항
- 데이터 처리량
- 메시지 키 사용 여부
- 브로커, 컨슈머 영향도

<br>

> 데이터 처리 속도를 올리는 방법
1. 컨슈머의 처리량을 늘린다.
    1. 컨슈머 서버의 사양을 올린다.(스케일 업)
    2. GC 튜닝
    3. 파티션 개수를 늘리고 그만큼 컨슈머를 추가한다.
2. 컨슈머를 추가해 병렬처리량을 늘린다.


<br>


📝 프로듀서 전송 데이터량 < (컨슈머 데이터 처리량 * 파티션 개수)

<br>
  
1. 컨슈머 데이터 처리량은 꼭 상용 환경에서 테스트해 측정한다.
2. 프로듀서가 보내는 데이터양을 하루, 시간, 분 단위로 쪼개 예측한다.
3. 데이터 처리 순서를 지켜야 하는지 고려해 메시지 키 사용여부를 정한다.
4. 파티션 개수를 늘릴 때 브로커 당 파티션 개수를 확인하고 진행한다.

<br>
  
<br>
  
## 토픽 정리 정책(cleanup.policy)

1. delete(삭제): 데이터 완전 삭제
2. compact(압축): 동일 메시지 키의 가장 오래된 데이터 삭제

  
<br>
  
### delete policy

- 일반적인 정리 정책으로 세그먼트 단위로 삭제
- segment.bytes: 세그먼트로 저장할 용량의 단위
- retention.ms: 토픽의 데이터를 유지하는 기간(밀리초)
- retention.bytes: 토픽의 최대 데이터 크기

  
<br>
  
### compact policy

- 동일 메시지 키의 가장 오래된 데이터 삭제
- KTable과 같이 메시지 키 기반 데이터 처리 상황에서 유용
- min.cleanable.dirty.ratio : 데이터의 압축 시작 시점
- min.cleanable.dirty.ratio : 액티브 세그먼트를 제외한 남아있는 세그먼트 데이터의 tail 영역 레코드 개수와 head 영역 레코드 개수의 비율
    - tail 영역: 압축이 완료된 레코드로 중복된 메시지 키가 없다. (== clean log)
    - head 영역: 중복된 메시지 키가 있다. (== dirty log)
    
    ⇒ 옵션값이 0.5인 경우, dirty ratio가 0.5를 넘어서면 압축이 수행된다.
    
  
<br>
  
<br>

## ISR(In-Sync-Replicas)

- 리더 파티션과 팔로워 파티션이 모두 싱크가 된 상태
- replica.lag.time.max.ms: 팔로워 파티션이 데이터를 복제하는지 확인하는 주기

⇒ 데이터를 가져가지 않으면 문제가 생겼다고 판단하고 ISR 그룹에서 제외

  
<br>
  
- ISR 그룹: 팔로워 파티션은 리더로 선출될 자격을 가진다.
- 그 외: 팔로워 파티션은 리더로 선출될 자격이 없다.

⇒ unclean.leader.election.enable = true로 설정하면 그 외 팔로워 파티션도 리더로 선출할 수 있다.

  
<br>
  
> **unclean.leader.election.enable 옵션**
일부 데이터가 유실되어도 무중단 운영이 필요한 경우 → true
데이터가 유실되면 안되는 경우 → false
> 

<br>
  
<br>


# 카프카 프로듀서

프로듀서의 고급 활용법과 옵션별 동작 방식에 대해 자세히 알아보자.

<br>
  
## acks 옵션

### acks=0

- 프로듀서가 리더 파티션으로 데이터를 전송하고 데이터 저장 여부를 응답 값으로 받지 않는다.
- 데이터 전송 속도는 가장 빠르기 때문에 신뢰성보다 전송 속도가 중요할 때 사용한다.

<br>
  
### acks=1

- 프로듀서는 리더 파티션에만 정상적으로 데이터가 적재되었는지 확인한다.
- 리더 파티션에 한해서만 적재될 때까지 재시도할 수 있다.
- 팔로워 파티션이 데이터를 복제하기 직전에 장애가 발생하면 데이터가 유실될 수 있다.

<br>
  
### acks=all (or -1)

- 프로듀서가 리더와 팔로워 파티션(ISR 그룹)에 모두 정상 적재되었는지 확인한다.
- 속도가 느리지만 장애가 발생해도 안전을 보장한다.
- min.insync.replicas: 프로듀서가 데이터 적재를 확인할 최소 ISR 그룹의 파티션 개수
    - 최소 2로 설정해야 all로 설정하는 의미가 있다.
    - 브로커 개수 미만으로 꼭 설정하도록 한다.

  
<br>
  
📝 **가장 안정적인 설정 방법**
토픽 복제 개수 = 3
min.insync.replicas = 2
acks = all

  
<br>
  
  
<br>
  
## 멱등성(idempotence) 프로듀서

- 멱등성: 여러 번 연산을 수행하더라도 동일한 결과를 나타내는 것
- 멱등성 프로듀서: 동일한 데이터를 여러 번 전송해도 카프카 클러스터에 한 번만 저장됨

  
<br>
  
**enable.idempotence 옵션**

- 데이터를 브로커로 전달 시 PID와 seq #을 함께 전달해 **정확히 한번 전달을 지원**
- 동일 세션(PID 생명주기)에서만 정확히 한번 전달을 보장
- true로 설정시 retries = Integer.MAX_VALUE, acks = all로 강제 설정

  
<br>
  
<br>
  
## 트랜잭션(transaction) 프로듀서

- 다수의 파티션에 데이터 저장 시 모든 데이터에 대해 동일한 원자성을 만족시키기 위해 사용
- 원자성: 데이터들을 동일 트랜잭션으로 묶어 전체를 처리하거나 전체를 처리하지 않도록 하는 것
- `enable.idempotence = true`, `transactional.id = ${String 값}` 으로 설정
- 컨슈머에서 `isolation.level = read_committed` 로 설정
- 트랜잭션은 시작과 끝을 표현하는 레코드를 한 개 더 보내 구분한다.

  
<br>
  
<br>
  
  
# 카프카 컨슈머

컨슈머의 고급 활용법과 옵션별 동작 방식을 알아보자.

<br>
  
<br>
  
## 멀티 스레드 컨슈머

- 파티션을 여러 개로 운영한다면 파티션 개수와 컨슈머 개수를 동일하게 맞추자.
- 파티션 개수가 n개라면 컨슈머 스레드를 최대 n개 운영할 수 있다.
- 한 컨슈머 스레드에서 예외가 발생 시, 프로세스 자체가 종료될 수 있다.
- 각 스레드 간에 영향이 없도록 스레드 세이프 로직, 변수를 적용해야 한다.

> **컨슈머를 멀티 스레드로 활용하는 방식**
1. 멀티 워커 스레드 전략
2. 컨슈머 멀티 스레드 전략
> 

<br>
  
<br>
  
  
### 카프카 컨슈머 멀티 워커 스레드 전략

컨슈머 스레드는 1개만 실행하고 데이터 처리 담당인 워커 스레드를 여러 개 실행한다.

- 데이터를 워커 스레드에서 병렬 처리해 속도가 빨라진다.
- 자바의 ExecutorService 라이브러리를 사용한다.
    - Executors: 스레드 개수를 제어하는 스레드 풀 생성
    - CachedThreadPool: 스레드 실행

<br>
  
<br>
  
**레코드들을 처리하는 워커 스레드를 실행하는 코드**

```java
public class ConsumerWorker implements Runnable {
	private String recordValue;

	ConsumerWorker(String recordValue) {
		this.recordValue = recordValue;
	}

	@Override
	public void run() {
		logger.info("thread:{}\trecord:{}", Thread.currentThread().getName(), recordValue);
	}
}
```
# 카프카 컨슈머

컨슈머의 고급 활용법과 옵션별 동작 방식을 알아보자.

<br>

## 멀티 스레드 컨슈머

- 파티션을 여러 개로 운영한다면 파티션 개수와 컨슈머 개수를 동일하게 맞추자.
- 파티션 개수가 n개라면 컨슈머 스레드를 최대 n개 운영할 수 있다.
- 한 컨슈머 스레드에서 예외가 발생 시, 프로세스 자체가 종료될 수 있다.
- 각 스레드 간에 영향이 없도록 스레드 세이프 로직, 변수를 적용해야 한다.

<br>

> **컨슈머를 멀티 스레드로 활용하는 방식**
1. 멀티 워커 스레드 전략
2. 컨슈머 멀티 스레드 전략
> 

<br>

### 카프카 컨슈머 멀티 워커 스레드 전략

컨슈머 스레드는 1개만 실행하고 데이터 처리 담당인 워커 스레드를 여러 개 실행한다.

- 데이터를 워커 스레드에서 병렬 처리해 속도가 빨라진다.
- 자바의 ExecutorService 라이브러리를 사용한다.
    - Executors: 스레드 개수를 제어하는 스레드 풀 생성
    - CachedThreadPool: 스레드 실행

<br>

**주의할 점**

- 데이터 처리가 끝나지 않았음에도 커밋 → 리밸런싱, 컨슈머 장애 시 데이터 유실
- 레코드 처리의 역전현상

<br>

## 컨슈머 랙(LAG)

- 토픽의 최신 오프셋과 컨슈머 오프셋 간의 차이
- 컨슈머가 정상 동작하는지 여부를 확인할 수 있는 모니터링 지표
- 컨슈머 그룹과 토픽, 파티션 별로 생성

<br>

> **컨슈머 랙 확인 방법**
1. 카프카 명령어 사용
2. metrics() 메서드 사용
3. 외부 모니터링 툴 사용
> 

<br>

1. **카프카 명령어 사용**

```java
$ bin/kafka-consumer-groups.sh --bootstrap-server my-kafka:9092 --group my-group --describe
```

일회성 확인 방법으로 테스트용 카프카에서 주로 사용한다.

<br>

2. **`metrics()` 메서드 사용**

```java
for (Map.Entry(MetricName, ? extends Metric> entry : kafkaConsumer.metrics().entrySet()) {
	if ("records-lag-max".equals(entry.getKey().name()) |
			"records-lag".equals(entry.getKey().name()) |
			"records-lag-avg".equals(entry.getKey().name())) {
		Metric metric = entry.getValue();
		logger.info("{}:{}", entry.getKey().name(), metric.metricValue());
	}
}
```

<br>

`**metrics()` 의 문제점**

1. 컨슈머 정상 동작 상황에서만 확인 가능
2. 모든 컨슈머 애플리케이션에 컨슈머 랙 모니터링 코드 중복
3. 카프카 서드 파티 애플리케이션은 모니터링 불가능


<br>

3. 외부 모니터링 툴 사용
- 가장 최선의 방법
- 클러스터 모니터링과 컨슈머 랙을 함께 또는 컨슈머만 모니터링
- DataDog, Confluent Control Center, Burrow 등


<br>

<br>

### 카프카 버로우

- 링크드인에서 공개한 오픈소스 컨슈머 랙 체크 툴
- REST API로 컨슈머 그룹별 컨슈머 랙 조회
- 슬라이딩 윈도우 계산으로 파티션의 문제를 인식한다.
- 파티션 상태: OK, STALLED, STOPPED
- 컨슈머 상태: OK, WARNING, ERROR

<br>

**컨슈머 랙 모니터링 아키텍처**

- 버로우
- 텔레그래프: 데이터 수집 및 전달 툴
- 엘라스틱서치: 컨슈머 랙 정보 저장소
- 그라파나: 엘라스틱서치 정보를 시각화 & 특정 조건에 슬랙 알람을 보내는 웹 대시보드 툴

<br>

### 컨슈머 배포 프로세스

> 카프카 컨슈머 애플리케이션 운영 시 로직 변경으로 인한 배포 방법
1. 중단 배포
2. 무중단 배포
> 

<br>

**중단 배포**

- 한정된 서버 자원을 운영할 때 적합
- 신뢰성 있는 배포 시스템일 때 적합
- 신규 애플리케이션의 실행 전후를 특정 지점으로 나눌 수 있어 롤백할 때 유용

<br>

**무중단 배포**

- 인스턴스 발급과 반환이 유연한 가상 서버에서 유용
- 블루/그린 배포
    - 이전 버전, 신규 버전 애플리케이션을 동시에 띄우고 트래픽을 전환
    - 파티션 개수와 컨슈머 개수를 동일하게 실행하고 운영할 때 유용
    - 짧은 리밸런스 시간으로 배포 수행
- 롤링 배포
    - 리소스 낭비를 줄이면서 무중단 배포
    - 파티션 개수가 인스턴스 개수와 같거나 많아야
    - 파티션 개수와 리밸런스 시간이 비례해 파티션이 적을 때 효과적
- 카나리 배포
    - 작은 위험을 통해 큰 위험을 예방한다.
    - 데이터 일부분을 신규 버전에 먼저 배포해 이슈를 사전 탐지
    - 사전 테스트 완료 시 나머지 파티션에 할당된 컨슈머에 무중단 배포

<br>

## 스프링 카프카

카프카를 스프링 프레임워크에서 효과적으로 사용할 수 있도록 만들어진 라이브러리

<br>

### 스프링 카프카 프로듀서

카프카 템플릿은 ProducerFactory 클래스로 생성할 수 있다.

<br>

**기본 카프카 템플릿**

`application.yml` 에 프로듀서 옵션을 넣어 사용한다.

<br>

<br>

**커스텀 카프카 템플릿**

한 스프링 카프카 애플리케이션 내부에 다양한 프로듀서 인스턴스를 생성하고 싶을 때 사용한다.

<br>

### 스프링 카프카 컨슈머

**컨슈머 타입**

- 레코드 리스너(Default) : 단 1개의 레코드 처리
    - MessageListener
    - AcknowledgingMessageListener
    - ConsumerAwareMessageListener
    - AcknowledgingConsumerAwareMessageListener
- 배치 리스너 : 한번에 여러 개 레코드 처리
    - BatchMessageListener
    - BatchAcknowledgingMessageListener
    - BatchConsumerAwareMessageListener
    - BatchAcknowledgingConsumerAwareMessageListener


<br>

**커밋 타입**

- RECORD : 레코드 단위 커밋
- BATCH : 레코드 모두 처리 후 커밋
- TIME : 특정 시간 이후 커밋
- COUNT : 특정 개수만큼 처리 후 커밋
- COUNT_TIME
- MANUAL
- MANUAL_IMMEDIATE


<br>

사용방식 1. 기본 리스너 컨테이너 사용

`application.yaml`에 컨슈머와 리스너 옵션을 넣어 사용한다.

<br>

사용방식 2. **커스텀 리스너 컨테이너**

서로 다른 설정을 가진 여러 리스너들이나 리밸런스 리스너를 구현하기 위해 사용한다.

<br>

<br>
