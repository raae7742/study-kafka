# CHAPTER 3. 카프카 기본 개념

## 카프카 브로커, 클러스터, 주키퍼

### 브로커

- 카프카 클라이언트와 데이터를 주고받기 위해 사용하는 주체
- 하나의 서버에 하나의 카프카 브로커 프로세스를 실행
- 안전한 데이터 보관을 위해 3대 이상의 브로커 서버를 1개의 클러스터로 묶어 운영

<br>

**브로커가 하는 일**

1. 데이터 저장, 전송
- 파일 시스템에 데이터를 저장한다.
- ‘페이지 캐시’를 이용해 높은 처리 속도를 보장한다.

<br>

2. 데이터 복제, 싱크
- 파티션 단위로 데이터를 복제하며 원본을 리더, 복사본을 팔로워라고 부른다.
- 리더 브로커가 다운되면 팔로워 중 하나가 리더 지위를 넘겨받는다.

<br>

3. 컨트롤러
- 하나의 브로커가 컨트롤러 역할을 한다.
- 다른 브로커들의 상태를 체크하며 리더 파티션을 재분배한다.

<br>

4. 데이터 삭제
- 데이터 삭제는 오직 브로커만 가능하다.
- 로그 세그먼트: 삭제되는 파일 단위

<br>

5. 컨슈머 오프셋 저장
- 컨슈머 그룹: 토픽이 어떤 파티션에서 어느 레코드까지 가져갔는지 확인하기 위해 오프셋을 커밋
- `offsets` 토픽에 저장된 오프셋을 토대로 컨슈머 그룹이 다음 레코드를 가져간다.

<br>

6. 코디네이터
- 하나의 브로커가 코디네이터 역할을 한다.
- 컨슈머 그룹의 상태를 체크하고 파티션을 컨슈머와 매칭되도록 분배한다.
- 리밸런스: 파티션을 컨슈머로 재할당하는 과정

<br>
<br>

### 주키퍼

- 카프카의 메타데이터 관리

<br>

### 토픽과 파티션

**토픽**

- 데이터 구분 단위
- 1개 이상의 파티션 소유
- 레코드: 프로듀서가 보내 파티션에 저장되는 데이터
- 카프카 병렬처리의 핵심
- FIFO 구조: 컨슈머는 먼저 들어간 레코드를 먼저 가져간다. ↔ 가져간 레코드는 삭제되지 않는다.

<br>

**토픽 이름 작명 방법**

- 어떤 개발환경에서 사용되는 것인지 판단 가능해야 한다.
- 어떤 애플리케이션에서 어떤 데이터 타입으로 사용되는지 유추할 수 있어야 한다.
- 카멜케이스보다는 케밥케이스, 스네이크 표기법이 어울린다.

<br>

예시)

- prd.marketing-team.sms-platform.json
- commerce.payment.prd.notification
- dev.email-sender.jira-1234.email-vo-custom
- aws-kafka.live.marketing-platform.json

<br>

### 레코드

- 타임스탬프 + 메시지 키 + 메시지 값 + 오프셋
- 브로커에 적재되면 수정할 수 없고 로그 리텐션 기간 또는 용량에 따라서만 삭제된다.
- 동일한 메시지 키는 동일 파티션에 들어간다.
- 키가 null인 레코드는 프로듀서 기본 설정 파티셔너에 따라 분배된다.
- 메시지 키, 값은 직렬화되어 브로커에 전송되므로, 컨슈머가 이용할 때에는 동일한 형태로 역직렬화를 해야 한다.

<br>

## 카프카 클라이언트

### 프로듀서 API

자바 애플리케이션과 카프카 라이브러리를 통해 프로듀서를 구현할 수 있다.

프로듀서는 데이터 전송 시 내부적으로 파티셔너, 배치 생성 단계를 거친다.

<br>

**프로듀서 주요 옵션**

- 필수 옵션
    - `bootstrap.servers` : 전송할 호스트이름:포트
    - `key.serializer` : 키를 직렬화하는 클래스
    - `value.serializer` : 값을 직렬화하는 클래스
- 선택 옵션
    - `acks` : 전송 성공 여부 확인 옵션
        - == 1(default) : 데이터 저장 시 성공으로 판단
        - == 0 : 데이터 전송 즉시 성공으로 판단
        - == -1(all) : min.insync.replicas 개수에 해당하는 리더/팔로워 파티션에 저장 시 성공으로 판단
    - `buffer.memory` : 배치 버퍼 메모리 양 (default: 33554432(32MB))
    - `retries` : 에러 이후 재전송 시도 횟수 (default: 2147483647)
    - `batch.size` : 배치로 전송할 최대 레코드 용량 (default: 16384)
    - `[linger.ms](http://linger.ms)` : 배치 전송 전 기다리는 최소 시간 (default: 0)
    - `partitioner.class` : 파티셔너 클래스 지정
    - `enable.idempotence` : 멱등성 프로듀서 동작 여부 설정 (default: false)
    - `[transactional.id](http://transactional.id)` : 레코드 전송 시 트랜잭션 단위로 묶을지 여부 설정(default: null)

<br>

**커스텀 파티셔너를 만들어보자.**

 메시지 키에 따라 파티션을 다르게 지정하고 싶을 때에는 직접 파티셔너를 만들어야 한다.

<br>

**브로커 정상 전송 여부 확인하기**

 send() 메서드가 반환하는 Future 객체는 비동기 결과를 표현하는 것으로 정상적으로 적재되었는지 알려준다.

사용자 정의 Callback 클래스를 생성해 비동기로 결과를 확인할 수 있다.

<br>
<br>

### 컨슈머 API

컨슈머는 적재된 데이터를 가져와 필요한 처리를 한다.

ex) 토픽에서 고객 데이터를 가져와 마케팅 문자를 고객에게 보낸다.


<br>

**컨슈머 중요 개념**

컨슈머를 운영하는 방법

1. 컨슈머 그룹을 운영한다.
2. 토픽의 특정 파티션만 구독하는 컨슈머를 운영한다.

<br>

**컨슈머에 장애가 발생한다면?**

> **리밸런싱**

컨슈머 크룹의 일부 컨슈머에 장애가 발생하면, 장애 발생 컨슈머에 할당된 파티션은 장애가 발생하지 않은 컨슈머에 소유권이 넘어간다. 그리고 이슈 발생 컨슈머를 그룹에서 제외해 가용성을 높인다.
 
- 리밸런싱은 자주 발생하면 안된다.
- 그룹 조정자가 컨슈머 그룹에 컨슈머 추가 및 삭제 시 리밸런싱을 발동시킨다.

<br>

**오프셋 커밋**

데이터 중복 처리를 방지하기 위해 컨슈머 애플리케이션이 오프셋 커밋을 정상 처리했는지 검증 과정이 필요하다.

- 비명시 오프셋 커밋: 일정 간격마다 자동으로 커밋
    - 리밸런싱 또는 컨슈머 강제종료 시에 처리 중이던 데이터가 중복 또는 유실될 수 있다.
- 명시 오프셋 커밋: `poll()` → `commitSync()` 호출

<br>

**컨슈머 주요 옵션**

필수 옵션

- `bootstrap.servers`
- `key.deserializer`
- `value.deserializer`

<br>

선택 옵션

- `group.id` : 컨슈머 그룹 ID(default: null)
- `auto.offset.reset` : 특정 파티션을 읽을 때 저장된 컨슈머 오프셋이 없는 경우 어디부터 읽을지 선택
    - == latest: 가장 높은 오프셋부터 읽음(default)
    - == earliest: 가장 낮은 오프셋부터 읽음
    - == none : 컨슈머 그룹 커밋 기록 확인
- `enable.auto.commit` : 자동 커밋 여부(default: true)
- `auto.commit.interval.ms` : 자동 커밋일 경우 오프셋 커밋 간격(default: 5000)
- `max.poll.records` : poll()로 반환되는 레코드 개수(default: 500)
- `session.timeout.ms` : 컨슈머가 브로커와 연결이 끊기는 최대 시간
    - 이 시간을 넘기면 이슈라고 판단하고 리밸런싱 진행
    - 보통 hearbeat 시간 * 3배로 설정(default: 10000)
- `hearbeat.interval.ms` : hearbeat 전송 시간 간격(default: 3000)
- `max.poll.interval.ms` : poll() 메서드 호출 간격의 최대 시간
    - 데이터 처리 시간이 최대 시간을 넘으면 리밸런싱 진행(default: 3000000)
- `isolation.level` : 레코드를 트랜잭션 단위로 보낼 경우 사용
    - == read_committed(default)
    - == read_uncommitted

<br>

**비동기 오프셋 커밋:** `commitAsync`

<br>

**리밸런스 리스너:** `ConsumerRebalanceListener`

리밸런스 발생 시 데이터를 중복 처리하지 않으려면 리밸런스 발생 시 처리한 데이터를 기준으로 커밋을 해야 한다. 리밸런스 발생을 감지하기 위해 카프카 라이브러리는 `ConsumerRebalanceListener` 인터페이스를 지원한다.

<br>

**파티션 할당 컨슈머: `assign()`**

- 파티션을 컨슈머에 명시적으로 할당해 운영한다.
- 직접 특정 토픽, 파티션에 할당하므로 리밸런싱이 없다.

<br>

**컨슈머에 할당된 파티션 확인하기: `assignment()`**

<br>

**컨슈머 안전 종료: `wakeup()`, `close()`**

- wakeup() : 컨슈머 인스턴스 안전 종료
- close() : 컨슈머 종료 여부를 명시적으로 알려줌

wakeup() 메서드는 셧다운 훅을 사용해 호출한다.

<br>

- 셧다운 훅: 사용자 또는 OS로부터 종료 요청을 받으면 실행하는 스레드

<br>

<br>

### 어드민 API

AdminClient클래스로 내부 옵션을 설정하거나 조회할 수 있다.

```java
Properties configs = new Properties();
configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "my-kafka:9092");
AdminClient admin = AdminClient.create(configs);
```

<br>

**브로커 정보 조회**

```java
Properties configs = new Properties();
configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "my-kafka:9092");

AdminClient admin = AdminClient.create(configs);

logger.info("== Get broker information");
for (Node node : admin.describeCluster().nodes().get()) {
	logger.info("node : {}", node);
	ConfigResource cr = new ConfigResource(ConfigResource.Type.BROKER, node.idString());
	DescribeConfigsResult describeConfigs = admin.describeConfigs(Collections.singleton(cr));
	describeConfigs.all().get().forEach((broker, config) -> {
		config.entries().forEach(configEntry -> logger.info(configEntry.name() + "= " + configEntry.value()));
	});
}
	
```

<br>

**토픽 정보 조회**

```java
Map<String, TopicDescription> topicInformation = admin.describeTopics(Collections.singletonList("test")).all().get();
logger.info("{}", topicInformation);
```

<br>

어드민 API는 사용 후 명시적으로 종료 메소드를 호출해 리소스 낭비를 막아야 한다.

```java
admin.close();
```

<br>

## 카프카 스트림즈

토픽에 적재된 데이터를 실시간으로 변환해 다른 토픽에 적재하는 라이브러리

<br>

**토폴로지(topology):** 2개 이상의 노드들과 선으로 이루어진 집합

- 종류: 링형, 트리형, 성형 등

<br>

**프로세서:** 카프카 스트림즈에서 토폴로지를 이루는 노드

- 소스 프로세서
    - 데이터를 처리하기 위해 최초로 선언하는 노드
    - 하나 이상의 토픽에서 데이터를 가져오는 역할
- 스트림 프로세서
    - 다른 프로세서가 반환한 데이터를 처리하는 역할
- 싱크 프로세서
    - 데이터를 특정 카프카 토픽으로 저장하는 역할

<br>

**스트림:** 노드와 노드를 이은 선, 토픽의 데이터

<br>

<br>

**데이터 처리 구현 방법**

1. **스트림즈 DSL**
2. **프로세서 API**

<br>

<br>

### 스트림**즈DSL 기본 개념**

**KStream**

- 레코드의 흐름 → 데이터 조회시 토픽에 존재하는 모든 레코드를 출력한다.
- `**(메시지 키, 메시지 값)**`

<br>

**KTable**

- 메시지 키를 기준으로 묶어 가장 최신 레코드를 사용한다.

<br>

**GlobalKTable**

- KTable과 동일하게 메시지 키를 기준으로 묶어서 사용한다.
- KTable과 달리 모든 파티션 데이터가 각 태스크에 할당되어 사용된다.

<br>

**코파티셔닝(co-partitioning)**

조인하는 2개 데이터의 파티션 개수가 동일하도록 맞추는 작업

⇒ 코파티셔닝되지 않은 두 개의 토픽을 조인하는 경우 *TopologyException*이 발생

<br>

> **코파티셔닝되지 않은 KStream과 KTable을 조인하기 위한 두 가지 방법**
1. 리파티셔닝 과정을 거친다.
2. KTable을 GlobalKTable로 선언해 사용한다. *(작은 용량일 경우에만 추천)*

<br>

<br>

### 스트림즈DSL 주요 옵션

**필수 옵션**

- `bootstrap.servers`
- `application.id`: 애플리케이션 고유 아이디

<br>

**선택 옵션**

- `default.key.serde` : 레코드 메시지 키를 직렬화, 역직렬화하는 클래스 지정*(기본값: 바이트)*
- `default.value.serde`: 레코드 메시지 값을 직렬화, 역직렬화하는 클래스 지정*(기본값: 바이트)*
- `num.stream.threads`: 스트림 프로세싱 실행 시 실행될 스레드 개수*(기본값: 1)*
- `state.dir`: 상태기반 데이터 처리 시 데이터를 저장할 디렉토리*(기본값: /tmp/kafka-streams)*

<br>

<br>

### 스트림즈DSL - `stream()` , `to()`

특정 토픽의 데이터를 다른 토픽으로 전달할 때 사용한다.

- `stream()` : 특정 토픽을 KStream 형태로 가져온다.
- `to()` : KStream의 데이터를 특정 토픽으로 저장한다.

<br>

<br>

### 스트림즈DSL - `filter`

 토픽으로 들어온 문자열 데이터 중 길이가 5보다 큰 경우만 필터링하는 스트림즈 애플리케이션을 만들어보자.

<br>

<br>

### 스트림즈DSL - `join()`

 KTable과 KStream을 함께 사용한다면 카프카에서는 실시간으로 들어오는 데이터들을 메시지 키를 기준으로 조인할 수 있다. 이를 통해 데이터를 DB에 저장하지 않고도 조인해 이벤트 기반 스트리밍 데이터 파이프라인을 구성할 수 있다. 조인할 때에는 코파티셔닝 여부를 꼭 확인해야 한다.

<br>

<br>

**코파티셔닝이 되어있지 않은 토픽은 어떻게 조인해야 할까?**

1. 리파티셔닝을 수행하고 조인 처리를 한다.
2. KTable 토픽을 GlobalKTable로 선언해 사용한다.

<br>

<br>

### 프로세서 API

 스트림즈DSL보다 추가적인 상세 로직의 구현이 필요하면 프로세서 API를 활용한다.

 토픽의 문자열 길이가 5 이상인 데이터를 필터링해 다른 토픽으로 저장해보자.

<br>

<br>

## 카프카 커넥트

- 데이터 파이프라인 생성 시 반복 작업을 줄이고 효율적인 전송을 이루기 위한 애플리케이션
- 특정 작업 형태를 템플릿으로 만든 커넥터를 실행해 반복 작업을 줄인다.

<br>

**커넥터의 종류**

- 소스 커넥터: 프로듀서 역할
- 싱크 커넥터: 컨슈머 역할

ex) MySQL에서 데이터를 보낼 때, 저장할 때 JDBC 커넥터를 사용해 파이프라인 생성

<br>

**실행 순서**

1. 사용자가 커넥트에 커넥터 생성을 명령
2. 커넥트가 내부에 커넥터와 태스크 생성
3. 커넥터는 태스크들을 관리
4. 태스크는 실질적인 데이터 처리 수행

<br>

**커넥트 실행 방법**

1. 단일 모드 커넥트: 1개의 프로세스만으로 실행 *→ 단일 장애점이 될 수도*
2. 분산 모드 커넥트: 2대 이상의 서버에서 클러스터 형태로 운영 *→ 안전*

<br>

**+ REST API(Port: 8083)**를 사용해 현재 실행 중인 커넥트, 태스크 상태 등을 조회할 수 있다.

<br>

**단일 모드 커넥트**

1. `connect-standalone.properties` 와 `connect-file-source.properties` 설정 파일 수정
2. 단일 모드 실행
    
    ```powershell
    $ bin/connect-standalone.sh config/connect-standalone.properties \ config/connect-file-source.properties
    ```
    
<br>

**분산 모드 커넥트**

1. `connect-distributed.properties` 설정 파일 수정
2. 분산 모드 실행
    
    ```powershell
    $ bin/connect-distributed.sh config/connect-distributed.properties
    ```

<br>

<br>

### 소스 커넥터

- 소스 애플리케이션, 파일로부터 데이터를 가져와 토픽으로 넣는 역할
- 오픈소스 커넥터를 사용하거나 직접 개발해 사용한다.
- 직접 개발하는 경우에는 `SourceConnector` 와 `SourceTask` 클래스를 사용한다.
    - `SourceConnector` : 태스크 실행 전 커넥터 설정파일을 초기화하고 어떤 태스크 클래스를 사용할 것인지 정의
    - `SourceTask` : 실제로 소스에서 데이터를 가져와 토픽으로 보내는 역할 수행

<br>

**카프카 커넥터를 직접 개발할 때에는 사용자가 작성한 클래스뿐만 아니라 참조하는 라이브러리도 함께 빌드해 jar로 압축해야 한다!**

<br>

<br>

### 싱크 커넥터

- 토픽 데이터를 타깃에 저장하는 역할
- `SinkConnector`와 `SinkTask` 클래스를 사용해 직접 구현할 수 있다.
- `SinkConnector` : 태스크를 실행하기 전 설정값 초기화, 사용할 태스크 클래스 정의
- `SinkTask` : 커넥트에서 컨슈머 역할을 하며 데이터를 저장 (데이터 처리 로직)


<br>

<br>

## 카프카 미러메이커2

- 서로 다른 두 개의 카프카 클러스터 간에 토픽을 복제하는 애플리케이션
- 토픽 데이터 뿐만 아니라 설정까지 복제한다.

<br>

### 단방향 토픽 복제

1. `connect-mirror-maker.properties` 파일 수정

카프카 클러스터 A와 B가 있을 경우를 가정해 수정해보자.

```bash
# 복제할 클러스터 닉네임
cluster = A, B

# 클러스터의 접속 정보
A.bootstrap.servers = a-kafka:9092
B.bootstrap.servers = b-kafka:9092

# A->B로 복제를 진행할 것인지, 어떤 토픽을 복제할 것인지 명시
A->B.enabled = true
A->B.topics = test

B->A.enabled = false
B->A.topics = .*

# 신규 생성된 토픽 복제 개수 설정
replication.factor=1

# 내부 토픽의 복제 개수 설정
checkpoints.topic.replication.factor=1
heartbeats.topic.replication.factor=1
offset-syncs.topic.replication.factor=1
offset.storage.replication.factor=1
status.storage.replication.factor=1
config.storage.replication.factor=1
```

<br>

2. 설정 파일과 함께 미러메이커2 실행

```bash
$ bin/connect-mirror-maker.sh config/connect-mirror-maker.properties
```

<br>

<br>

### 지리적 복제(Geo-Replication)

미리메이커2는 단방향, 양방향 복제 기능, ACL복제, 새 토픽 자동 감지 등의 기능을 제공한다.

<br>

**Active-standby 클러스터 운영**

- 적용 상황: 서비스용 클러스터 외에 재해 복구를 위한 임시 클러스터를 하나 더 구성할 때
- 액티브 클러스터: 서비스와 직접 통신하는 클러스터
- 스탠바이 클러스터: 나머지 1개의 클러스터
- *복제 랙이 있어 스탠바이 클러스터에 액티브의 모든 정보가 복제되지 않았을 수도 있다.*

*⇒ 이에 대한 대응 방안을 사전에 정해둘 필요가 있다.*

*⇒ 장애 복구 훈련을 계획하고 수행하는 것이 매우 중요!*

<br>

**Active-active 클러스터 운영**

- 적용 상황: 글로벌 서비스에서 통신 지연을 최소화하기 위해 2개 이상의 클러스터를 둘 때

<br>

**Hub and spoke 클러스터 운영**

- 적용 상황:  각 팀의 소규모 클러스터 데이터를 한 클러스터에 모아 데이터 레이크로 사용하고 싶을 때
- 허브: 중앙에 있는 한 개의 점
- 스포크: 중앙의 점과 다른 점들을 연결한 선
