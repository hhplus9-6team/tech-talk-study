# 기술 아티클 공유 — 작업 처리 구조 개선과 DB의 숨은 비용

최근 읽은 기술 아티클 중 시스템의 처리 구조를 개선하거나, 평소 당연하게 사용하던 기술의 내부 동작을 분석해 성능을 개선한 세 가지 사례를 정리했다.

단순히 어떤 기술을 도입해 성능이 좋아졌는지보다는, 기존 방식에는 어떤 한계가 있었고 개선된 방식은 어떤 원리로 문제를 해결했는지에 집중했다.

---

## 1. DB 작업 큐에서 상태 기반 작업 처리 구조로

**우아한형제들 | [사장님 입점신청 자동승인 주기를 10분에서 10초로](https://techblog.woowahan.com/27678/)**

`Architecture` `Scheduling` `Distributed Lock` `Retry`

### 1-1. 기존 구조와 문제점

기존 자동승인 시스템은 두 개의 스케줄러와 DB 작업 큐를 통해 동작하고 있었다.

```text
요청 테이블 (업무 데이터 및 상태)
    ↓
Enqueue Scheduler (07분, 17분, 27분 ...)
    - 후보 조회 및 상세 검증
    ↓ INSERT
DB 작업 큐 (처리 대상 저장)
    ↓ SELECT
Process Scheduler (00분, 10분, 20분 ...)
    - 실제 자동승인 수행
```

후보 수집과 처리를 분리하면 각각 독립적으로 재시도하거나 모니터링하기 쉽다는 장점이 있다.

하지만 두 스케줄러의 실행 시점이 분리되어 있어 구조적으로 최소 3분의 대기 시간이 발생했고, 최대 약 13분까지 처리가 지연될 수 있었다.

단순히 실행 주기를 줄이는 것도 어려웠다. 후보를 수집하는 과정에서 넓은 범위의 데이터 탐색, 상세 데이터 JOIN, 외부 API 호출이 함께 발생하고 있었기 때문이다.

즉, 개별 쿼리가 느린 것보다 **기존 작업 처리 구조가 더 짧은 실행 주기를 감당하기 어렵다는 것이 근본적인 문제였다.**

### 1-2. 작업 큐는 정말 필요한가?

이 글에서 가장 인상적인 부분은 폴링 주기를 줄이는 방법을 고민하기 전에, 기존에 사용하던 작업 큐의 필요성부터 다시 검토했다는 점이다.

일반적으로 작업 큐는 처리할 작업을 저장하고, 생산자와 소비자의 실행 시점을 분리하는 역할을 한다.

그런데 해당 시스템에서는 원본 요청 테이블에 이미 업무 상태가 존재하고, 이를 바탕으로 처리 대상을 다시 식별할 수 있었다.

따라서 후보 목록을 별도 테이블에 적재해 두기보다는 다음 정보만 별도로 관리하는 방식으로 변경했다.

- 현재 다른 워커가 처리하고 있는 요청
- 실패하여 재시도를 기다리는 요청
- 다음 재시도 시각과 재시도 횟수

```text
요청 테이블                  처리 상태 테이블
- 업무 데이터                - 처리 중 여부
- 처리 대상 여부             - 재시도 정보
        └─────────┬─────────┘
                  ↓
            통합 스케줄러
     (신규 및 재시도 가능 대상 조회)
                  ↓
          요청 단위 Redis Lock
                  ↓
            실제 작업 처리
     (선점 성공 후 상세 조회 및 외부 API 호출)
```

두 구조의 차이는 단순히 테이블 하나를 제거했다는 데 있지 않다.

| 구분 | 기존 구조 | 개선 구조 |
|---|---|---|
| 후보 관리 | 작업 큐에 적재 | 원본 데이터에서 재계산 |
| 신규 작업 | 큐에 NEW 상태로 저장 | 상태 레코드 없이 식별 |
| 처리 상태 | 작업 큐에서 관리 | 별도 상태 테이블에서 관리 |
| 실행 방식 | 수집 후 다음 스케줄러 대기 | 조회 후 바로 처리 |
| 상세 검증 | 후보 수집 단계 | 작업 선점 이후 |

기존 작업 큐가 처리할 대상의 목록과 실행 상태를 함께 관리했다면, 개선된 구조는 **업무 상태와 작업 실행 상태를 분리한 것**이라고 볼 수 있다.

다만 모든 DB 작업 큐를 이러한 구조로 변경할 수 있는 것은 아니다. 원본 데이터로부터 처리 대상을 다시 계산할 수 있어야 한다는 전제가 필요하다.

특정 시점의 이벤트 자체를 보존해야 하거나, 작업별 payload와 순서를 보장해야 하는 시스템이라면 별도의 큐가 여전히 필요할 수 있다.

### 1-3. 폴링 주기와 재시도 주기의 분리

기존 구조에서는 스케줄러가 실행되는 간격이 곧 재시도 간격이었다.

이 때문에 신규 작업을 빠르게 처리하려고 스케줄러의 실행 주기를 줄이면 실패한 작업의 재시도 빈도까지 증가하는 문제가 있었다.

```text
실행 주기만 줄이는 경우
시간(초)     0    10    20    30    40    50
신규 탐색    ●     ●     ●     ●     ●     ●
실패 재시도  ●     ●     ●     ●     ●     ●

폴링과 재시도를 분리한 경우
시간(초)     0    10    20    30    40    50
신규 탐색    ●     ●     ●     ●     ●     ●
실패 재시도  ● ─────── 다음 재시도까지 대기 ───
```

신규 탐색은 10초 간격으로 실행하되 실패 작업의 재시도는 기존 10분 간격을 유지한다. 이를 위해 실패한 작업은 다음 실행 가능 시각을 별도로 저장하도록 했다.

```sql
-- 재시도 가능한 작업의 조회 조건 예시
WHERE status = 'RETRY_WAITING'
  AND nextRetryTime <= NOW()
  AND retryCount < maxRetryCount
```

이러한 구조에서는 신규 작업의 발견 속도를 높이면서도 실패 작업의 재시도 빈도가 함께 증가하는 것을 방지할 수 있다. 반복해서 실패하는 작업이 제한된 처리 슬롯을 계속 차지하거나 재시도 한도를 너무 빠르게 소진하는 문제도 완화할 수 있다.

**작업을 발견하는 정책과 실패한 작업을 다시 실행하는 정책은 서로 독립적으로 관리해야 한다.**

### 1-4. 다중 워커 환경에서의 작업 선점

스케줄러를 한 서버에서만 실행하면 중복 처리 문제는 단순해지지만, 서버를 증설해도 작업 처리량을 높이기 어렵다.

따라서 모든 워커가 독립적으로 후보를 조회하고, 작업 단위로 선점하도록 설계했다.

| 방식 | 장점 | 단점 |
|---|---|---|
| 조건부 UPDATE | DB 내부에서 원자적으로 선점 가능 | 선점 실패도 쓰기 DB에 요청을 전달 |
| FOR UPDATE SKIP LOCKED | 다른 작업자가 잠근 행을 건너뛸 수 있음 | 쿼리 구조에 따라 업무 테이블에 락 발생 |
| Redis Lock | 선점 실패를 추가 쓰기 DB 접근 전에 처리 가능 | Redis와 DB 상태 간 정합성 관리 필요 |

#### 조건부 UPDATE

```sql
UPDATE TaskState
SET status = 'IN_PROGRESS'
WHERE taskId = ?
  AND status = 'RETRY_WAITING';
```

영향받은 행의 개수를 기준으로 선점 성공 여부를 판단할 수 있다. 단일 DB에서 상태 변경과 선점 여부 판단이 이루어지므로 구조가 단순하다.

하지만 여러 워커가 동일한 작업을 선점하려고 하면 실패하는 요청도 모두 쓰기 DB에 전달된다. 동일한 레코드에 대한 갱신 경쟁이 발생하면 선행 트랜잭션이 종료될 때까지 기다려야 할 수도 있다.

#### FOR UPDATE SKIP LOCKED

이미 다른 트랜잭션에서 잠근 작업을 건너뛰고 다음 작업을 가져올 수 있는 방식이다.

```sql
SELECT *
FROM Task
WHERE status = 'PENDING'
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

독립적인 작업 큐 테이블이라면 유용하지만, 해당 사례에서는 후보 조회에 실제 업무 요청 테이블이 포함되어 있었다.

여러 종류의 업무 요청과 수동 승인, 담당자 변경 등의 작업도 동일한 테이블을 사용하고 있었기 때문에 자동승인 작업의 선점 과정에서 발생하는 락이 다른 업무 처리에 영향을 줄 수 있었다.

격리 수준을 READ COMMITTED로 변경해 Gap Lock의 영향을 줄이더라도 Record Lock 문제는 남을 수 있다.

#### Redis Lock

최종적으로 요청 단위 Redis Lock을 선택했다.

```text
Worker 1      Worker 2      Worker 3
    └────────────┼────────────┘
             동일 후보 조회
                  ↓
              Redis Lock
           ┌──────┴──────┐
        선점 성공       선점 실패
           ↓               ↓
     Master DB 상태 확인   다음 후보 탐색
           ↓
        작업 처리
```

후보 조회는 여러 서버에서 수행하더라도 선점에 실패한 요청의 추가적인 쓰기 DB 접근을 줄일 수 있다.

다만 Redis와 DB라는 서로 다른 저장소를 사용하므로 락 획득만으로 작업의 안전성이 보장되는 것은 아니다.

중요한 점은 Redis가 DB보다 빠르다는 단순한 비교가 아니라, **선점 경쟁으로 발생하는 비용을 업무 데이터가 저장된 DB에서 분리했다는 것**이다.

### 1-5. 분산 락과 작업 상태는 왜 함께 필요한가?

Redis Lock에는 만료 시간이 있고 워커는 작업 도중 종료될 수 있다. 따라서 선점과 처리 과정에서는 다음과 같은 상태 관리도 필요하다.

```text
Redis Lock 획득
    ↓ 동시 선점 방지
Master DB에서 상태 재확인
    ↓ 복제 지연 및 오래된 후보 정보 방어
IN_PROGRESS 상태 기록
    ↓ 처리 중이라는 정보를 영속적으로 보존
실제 작업 수행
    ↓ 상세 업무 로직 및 외부 API 호출
처리 결과 반영
    - 성공 시 상태 삭제
    - 재시도 필요 시 상태 갱신
```

Redis Lock은 동시 선점 경쟁을 제어하고, DB 상태는 작업의 현재 실행 상태와 재시도 정보를 관리하는 역할을 한다.

후보 조회를 Replica에서 수행하는 경우 복제 지연 때문에 이미 처리 중인 작업이 다시 조회될 수 있다. 따라서 실제 선점 시점에는 Master DB의 상태를 다시 확인해야 한다.

반대로 워커가 비정상 종료되면 Redis Lock은 만료되더라도 DB에는 IN_PROGRESS 상태가 남는다. 해당 상태가 계속 유지되면 다른 워커가 작업을 다시 발견하지 못할 수 있다.

이를 해결하기 위해 별도의 복구 스케줄러를 구성했다.

| 상황 | 처리 방법 |
|---|---|
| Redis Lock 획득 실패 | 해당 작업 건너뛰기 |
| 락 만료 후 기존 작업이 진행 중 | DB 상태 확인을 통한 중복 처리 방지 |
| 워커 종료 후 IN_PROGRESS 잔존 | 일정 시간 이후 RETRY_WAITING 전환 |
| 담당자가 이미 수동으로 처리 | 불필요한 상태 레코드 삭제 |
| 재시도 한도 초과 | 자동 처리 대상에서 제외 |

복구 기준 시간이 실제 최대 작업 시간보다 너무 짧다면 정상적으로 처리 중인 작업까지 재시도 대상으로 변경할 수 있으므로 주의해야 한다. 외부 시스템 호출까지 포함한 중복 처리 방지를 위해서는 별도의 멱등성 보장 여부도 검토할 필요가 있다.

### 1-6. 결과 및 얻을 수 있는 인사이트

| 지표 | 기존 | 개선 후 |
|---|---|---|
| 처리 시작 대기 시간 | 최대 12분 59초 | 조건부 10초 이내 |
| 작업 탐색 | 후보 전체 수집 및 큐 적재 | 필요한 후보 조회 및 즉시 선점 |
| 재시도 정책 | 스케줄러 실행 주기에 종속 | 다음 재시도 시각으로 독립 관리 |

10초 이내라는 결과는 유입량이 처리 가능량을 초과하지 않는 조건을 전제로 한다. fixedDelay 방식이므로 모든 요청의 10초 이내 처리를 무조건 보장하는 것은 아니다.

이 사례는 DB 자체의 처리 성능을 60배 향상시킨 사례가 아니다. 작업 처리 구조를 재설계함으로써 불필요한 대기 시간을 제거하고, 더 짧은 실행 주기를 감당할 수 있도록 변경한 사례이다.

- 작업의 원본 정보와 실행 상태를 분리한다.
- 작업 탐색, 실행, 재시도 정책을 독립적으로 관리한다.
- 락의 범위와 실패 비용을 고려하여 선점 방식을 선택한다.
- 분산 락뿐 아니라 장애 이후의 작업 복구까지 함께 설계한다.
- 기존 구성 요소를 최적화하기 전에 해당 구성 요소가 여전히 필요한지 검토한다.

---

## 2. 읽기 전용 트랜잭션에도 비용이 발생한다

**카카오페이 | [JPA Transactional 잘 알고 쓰고 계신가요?](https://tech.kakaopay.com/post/jpa-transactional-bri/)**

`Spring` `JPA` `Transaction` `MySQL`

### 2-1. 문제 상황

카카오페이 온라인결제 서비스에서 DB 부하를 분석한 결과, 실제 비즈니스 쿼리보다 트랜잭션 및 세션 설정 관련 명령이 많은 비중을 차지하고 있었다.

| 주요 명령 | Peak QPS (근삿값) |
|---|---:|
| 전체 | 24,000 |
| SET OPTION | 약 14,000 |
| SELECT, COMMIT | 약 5,000 |
| UPDATE, INSERT | 3,000 미만 |

`autocommit`, `transaction` 등의 세션 설정 명령이 높은 비중을 차지하고 있었고, JPA의 트랜잭션 관리가 DB 부하에 영향을 미치고 있었다.

### 2-2. 읽기 전용 트랜잭션을 사용하는 이유

```java
@Transactional(readOnly = true)
public Order findOrder(Long orderId) {
    return orderRepository.findById(orderId).orElseThrow();
}
```

| 계층 | 이점 |
|---|---|
| InnoDB | 읽기 전용 작업에 대한 내부 트랜잭션 관리 비용 절감 |
| 트랜잭션 | 격리 수준에 따른 여러 조회 간 일관성 보장 |
| Hibernate | 자동 Flush 억제, Dirty Checking 비용 절감 |
| 영속성 컨텍스트 | 트랜잭션 범위 내 엔티티 관리 및 지연 로딩 |
| DataSource | 별도 구현을 통해 읽기용 DB 라우팅 가능 |

#### 데이터 일관성

여러 SELECT 결과가 동일한 시점의 데이터를 기준으로 해야 하는 경우 명시적 트랜잭션이 필요할 수 있다.

예를 들어 포인트 잔액과 사용 내역을 각각 조회하는 상황을 생각해 볼 수 있다.

| 시점 | 명시적 트랜잭션 없음 | 읽기 전용 트랜잭션 (RR) |
|---|---|---|
| 잔액 조회 | 1,000원 | 1,000원 |
| 다른 요청 | 500원 사용 및 커밋 | 500원 사용 및 커밋 |
| 사용 내역 조회 | 500원 사용 포함 | 500원 사용 미포함 |
| 결과 | 조회 시점 불일치 | 동일한 Read View 기준 조회 |

InnoDB REPEATABLE READ의 일반적인 비잠금 조회 기준이다. 명시적 트랜잭션이 없는 경우 각각의 SELECT가 독립적으로 실행된다고 가정한다.

이러한 일관성은 readOnly 속성 자체가 아니라 트랜잭션 범위와 격리 수준에 의해 보장된다. READ COMMITTED에서는 같은 트랜잭션에서도 일반 SELECT마다 새로운 Read View를 생성한다.

#### Hibernate의 읽기 전용 최적화

일반적인 Spring과 Hibernate 환경에서는 읽기 전용 트랜잭션을 통해 자동 Flush를 억제하고, 읽기 전용으로 로딩되는 엔티티의 변경 감지 및 Snapshot 관리 비용을 줄일 수 있다.

특히 다수의 엔티티를 조회하는 경우 애플리케이션의 CPU와 메모리 사용량 측면에서도 이점을 얻을 수 있다. 구체적인 적용 범위는 프레임워크 버전과 영속성 컨텍스트 상태에 따라 달라진다.

### 2-3. 단순 조회에도 명시적 트랜잭션이 필요한가?

읽기 전용 트랜잭션의 장점이 항상 명시적 트랜잭션의 생성 비용보다 큰 것은 아니다.

트랜잭션을 시작하고 종료하는 과정에서는 JDBC 설정에 따라 실제 비즈니스 쿼리 외의 추가 명령이 DB에 전달될 수 있다.

```text
명시적 읽기 트랜잭션              Autocommit 단건 조회
SET SESSION TRANSACTION READ ONLY
SET autocommit = 0
SELECT ...                       SELECT ...
COMMIT
SET autocommit = 1
SET SESSION TRANSACTION READ WRITE
```

개념적인 명령 비교이며 실제 세션 설정 명령과 순서는 JDBC 드라이버 및 커넥션 상태에 따라 달라진다.

각 명령의 실행 비용이 작더라도 높은 트래픽 환경에서는 추가적인 네트워크 통신과 DB 명령 처리 비용이 누적된다.

또한 명시적 트랜잭션 없이 일반 SELECT를 실행한다고 해서 InnoDB의 읽기 전용 최적화를 받지 못하는 것도 아니다.

InnoDB에는 AC-NL-RO(Auto Commit, Non Locking, Read Only) 최적화가 존재한다.

`autocommit=1`인 상태에서 실행되는 단일 비잠금 SELECT는 읽기 전용 작업으로 식별할 수 있으므로 트랜잭션 ID 할당이나 내부 관리 자료구조와 관련된 일부 비용을 절감할 수 있다.

즉, 단순 조회에서는 명시적으로 읽기 전용 트랜잭션을 생성하지 않는 것이 오히려 효율적일 수 있다. 읽기 전용 COMMIT이 반드시 디스크 fsync를 발생시키는 것도 아니다.

### 2-4. 개선 방법

카카오페이에서는 단독 조회 시 새로운 트랜잭션을 생성하지 않으면서 기존 읽기용 DataSource 라우팅을 유지하기 위해 Propagation.SUPPORTS를 활용했다.

```java
@Transactional(
    readOnly = true,
    propagation = Propagation.SUPPORTS
)
public Order findOrder(Long orderId) {
    // 조회 로직
}
```

| 전파 속성 | 기존 트랜잭션 없음 | 기존 트랜잭션 있음 |
|---|---|---|
| REQUIRED | 새 트랜잭션 생성 | 기존 트랜잭션 참여 |
| SUPPORTS | 생성하지 않음 | 기존 트랜잭션 참여 |
| NOT_SUPPORTED | 생성하지 않음 | 기존 트랜잭션 일시 중단 |
| NEVER | 생성하지 않음 | 예외 발생 |

SUPPORTS를 사용하면 상위 업무 트랜잭션이 존재하는 경우에는 해당 트랜잭션에 참여하면서, 독립적으로 호출되는 경우에는 명시적인 DB 트랜잭션 생성을 생략할 수 있다.

여기서 주의해야 할 점은 Spring Data JPA의 기본 Repository 메서드에도 트랜잭션이 선언되어 있다는 것이다.

Service에서 @Transactional을 제거하더라도 내부적으로 SimpleJpaRepository.findById()를 호출하면 새로운 트랜잭션이 생성될 수 있다. 따라서 카카오페이에서는 조회 빈도가 높은 일부 Repository의 조회 메서드도 별도로 구현했다.

참고로 원문의 읽기용 DataSource는 결제 데이터의 실시간성을 위해 Master DB를 바라보고 있었다. 읽기 트래픽을 Replica로 분산해서 얻은 성능 개선과는 구분해야 한다. 또한 SUPPORTS에서도 읽기 전용 라우팅이 유지되는지는 실제 라우팅 및 커넥션 획득 구현에 따라 검증해야 한다.

### 2-5. 성능 개선 결과 및 적용 시 고려 사항

| 항목 | 원문 테스트 결과 |
|---|---|
| SELECT 처리량 | 2~3배 향상 |
| API 성능 | 약 2배 향상 |

OSIV를 비활성화한 단순 조회 성능 테스트 결과이며 모든 조회에 일반화할 수는 없다.

| 조회 특성 | 고려할 방식 |
|---|---|
| 단일 SELECT 기반 DTO 조회 | 명시적 트랜잭션 생략 검토 |
| 높은 빈도의 단건 조회 | SUPPORTS 검토 |
| 여러 조회 간 일관성 필요 | 명시적 트랜잭션 및 격리 수준 검토 |
| 다수의 엔티티 조회 | Hibernate 읽기 전용 최적화 고려 |
| 지연 로딩 및 영속성 컨텍스트 유지 필요 | 명시적 트랜잭션 고려 |

현재 서비스에서 트랜잭션 관련 명령이 얼마나 발생하는지는 MySQL 상태 지표를 통해 확인해 볼 수 있다.

```sql
SHOW GLOBAL STATUS LIKE 'Com_set_option';
SHOW GLOBAL STATUS LIKE 'Com_select';
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';
```

누적 지표이므로 일정 시간 동안의 증가량을 기준으로 비교해야 한다. 또한 서버 전체 지표이기 때문에 특정 애플리케이션이나 API에 귀속시키려면 추가적인 분석이 필요하다.

**핵심은 트랜잭션을 제거하는 것이 아니라, 필요한 트랜잭션과 불필요한 트랜잭션을 구분하는 것이다.**

---

## 3. PreparedStatement는 항상 성능에 유리할까?

**카카오페이 | [우리의 애플리케이션에서 PreparedStatement는 어떻게 동작하고 있는가](https://tech.kakaopay.com/post/how-preparedstatement-works-in-our-apps/)**

`JDBC` `HikariCP` `PreparedStatement` `MySQL`

### 3-1. PreparedStatement에 대한 일반적인 이해

PreparedStatement는 SQL과 파라미터를 분리해 실행하는 방식이다.

- 파라미터 바인딩을 통한 SQL Injection 방지
- 반복적인 SQL 실행 시 파싱 비용 절감 가능
- 준비된 Statement 재사용

하지만 JDBC의 PreparedStatement 인터페이스를 사용한다는 사실이 MySQL 서버에서 실제 PreparedStatement를 생성한다는 의미는 아니다.

```text
Hibernate (SQL 생성 및 실행 요청)
    ↓
HikariCP (Connection 관리 및 실행 위임)
    ↓
MySQL Connector/J (PreparedStatement의 실제 구현 및 캐싱)
    ↓
MySQL Server (SQL 처리 및 서버 측 Statement 관리)
```

MySQL Connector/J의 설정에 따라 서버에서 SQL을 준비할 수도 있고, 클라이언트에서 SQL을 구성해 전달할 수도 있다.

### 3-2. Client-side와 Server-side PreparedStatement

```java
PreparedStatement statement = connection.prepareStatement(
    "SELECT * FROM Orders WHERE id = ?"
);
statement.setLong(1, 100L);
statement.executeQuery();
```

#### Client-side PreparedStatement

`useServerPrepStmts=false`

JDBC Driver가 파라미터 바인딩과 타입 및 이스케이프 처리를 담당한 후 완성된 SQL 문자열을 MySQL에 전달한다.

```sql
SELECT * FROM Orders WHERE id = 100;
```

MySQL 서버에는 일반적인 SQL 실행 요청(COM_QUERY)이 전달되므로 매 실행 시 SQL 파싱 과정이 발생한다.

#### Server-side PreparedStatement

`useServerPrepStmts=true`

MySQL 서버에 SQL을 준비하도록 요청한 후, 반환된 Statement ID와 파라미터를 이용하여 쿼리를 실행한다.

```text
JDBC Driver                        MySQL Server
    | ----- COM_STMT_PREPARE -------> |
    | <------- Statement ID --------- |
    | ----- EXECUTE (id=100) -------> |
    | <---------- Result ------------ |
    | ----- EXECUTE (id=200) -------> |
    | <---------- Result ------------ |
```

서버에 준비된 Statement를 재사용할 수 있으므로 반복적인 파싱 비용을 줄일 수 있다. 다만 MySQL에서 PreparedStatement를 재사용하는 것이 실행 계획까지 항상 고정적으로 재사용한다는 의미는 아니다.

### 3-3. 서버 측 PreparedStatement가 오히려 느려지는 이유

일반적인 ORM 환경에서는 개별 쿼리를 실행하는 과정에서 Statement를 획득하고 실행 후 반환하거나 종료한다.

서버 측 PreparedStatement를 활성화했지만 캐싱하지 않는다면 다음과 같은 비용이 반복적으로 발생할 수 있다.

| 호출 | Cache OFF | Cache ON (캐시 적중 가정) |
|---|---|---|
| 첫 번째 | PREPARE → EXECUTE → CLOSE | PREPARE → EXECUTE → 캐시에 반환 |
| 두 번째 | PREPARE → EXECUTE → CLOSE | 캐시에서 획득 → EXECUTE → 캐시에 반환 |

서버 측 PreparedStatement의 목적은 준비된 SQL을 재사용하여 비용을 절감하는 것이다.

하지만 매번 Statement를 새로 생성하고 해제한다면 추가 네트워크 통신과 서버 측 리소스 관리 비용만 늘어날 수 있다. COM_STMT_CLOSE는 서버 응답을 기다리는 명령은 아니지만 전송 및 리소스 해제 비용은 발생한다.

즉, PreparedStatement의 성능상 이점을 얻기 위해서는 단순히 서버 측 기능을 활성화하는 것보다 실제 재사용 여부가 중요하다.

### 3-4. 성능 테스트 결과

원문에서는 하나의 물리 Connection을 사용하여 단순 PK 조회 쿼리를 2만 번 반복 실행하고 설정별 성능을 비교했다.

| 설정 | 전체 실행 시간 |
|---|---:|
| Server-side / Cache OFF | 21분 32초 |
| Server-side / Cache ON | 4분 5초 |
| Client-side / Cache OFF | 3분 50초 |
| Client-side / Cache ON | 3분 55초 |

MySQL 8.0.23, MySQL Connector/J 9.1.0 환경의 원문 측정 결과이다.

서버 측 PreparedStatement를 사용하면서 캐싱하지 않은 경우에는 실행 시간이 크게 증가했다. 반면 나머지 세 가지 설정에서는 큰 성능 차이가 나타나지 않았다.

단순한 PK 조회에서는 SQL 파싱 비용 자체가 크지 않았을 가능성이 있기 때문이다.

따라서 이 결과는 서버 측 PreparedStatement가 항상 더 우수하다는 것을 입증하기보다는, **서버 측 Statement를 반복 생성하고 해제할 때 발생하는 불필요한 비용을 확인한 결과**라고 해석하는 것이 적절하다.

### 3-5. Connection Pool과 캐시 크기의 관계

| 설정 | 역할 |
|---|---|
| `useServerPrepStmts` | 서버 측 PreparedStatement 사용 여부 |
| `cachePrepStmts` | 관련 캐싱 활성화 |
| `prepStmtCacheSize` | 커넥션별 캐시 크기 |
| `prepStmtCacheSqlLimit` | 캐싱 대상 SQL의 길이 제한 |

여기서 중요한 점은 HikariCP가 직접 PreparedStatement 캐시를 관리하는 것이 아니라 MySQL Connector/J가 관리하며, 해당 드라이버의 캐시는 물리 Connection 단위라는 점이다.

동일한 SQL을 사용하더라도 서로 다른 Connection이라면 각 Connection에서 별도로 서버 측 Statement를 생성할 수 있다.

```text
             Application / HikariCP
                       |
       ┌───────────────┼───────────────┐
  Connection 1    Connection 2    Connection 3
  SQL A, B, C     SQL A, B, C     SQL A, B, C
       └───────────────┼───────────────┘
                  MySQL Server
     SQL 종류는 3개, 서버 측 Statement는 최대 9개
     (각 SQL을 커넥션별로 하나씩 캐싱한 예시)
```

예를 들어 다음과 같은 가상의 설정이 있다고 가정해 보자.

| 항목 | 설정값 |
|---|---:|
| Application Pod | 3개 |
| Pod당 Connection Pool | 100개 |
| prepStmtCacheSize | 250개 |
| 전체 캐시 슬롯 합계 | 75,000개 |

모든 Connection이 동일한 DB 서버에 연결되고 서버 측 Statement 캐시가 충분히 채워진다면 상당한 규모의 Statement를 관리하게 된다.

MySQL의 기본 `max_prepared_stmt_count`는 16,382개이므로 서버 측 Statement의 생성 한도를 초과할 가능성도 고려해야 한다.

물론 75,000개는 실제 생성량이 아니라 설정을 기준으로 계산한 캐시 슬롯의 합계이다. 실제 생성량은 SQL 종류, 캐시 적중률, Connection 사용량 등에 따라 달라지며 캐시 외 활성 Statement도 존재할 수 있다.

```sql
-- 서버 측 Statement 생성 한도
SHOW VARIABLES LIKE 'max_prepared_stmt_count';

-- 현재 서버 측 Statement 개수
SHOW GLOBAL STATUS LIKE 'Prepared_stmt_count';

-- Prepare, Execute, Close 관련 누적 명령 수
SHOW GLOBAL STATUS LIKE 'Com_stmt_%';
```

### 3-6. 얻을 수 있는 인사이트

PreparedStatement와 같은 최적화 기능도 내부 구현을 이해하지 않고 사용하면 기대했던 효과를 얻지 못하거나 오히려 비용이 증가할 수 있다.

특히 애플리케이션의 Scale-out이나 Connection Pool 크기 변경은 단순히 DB 연결 수에만 영향을 미치는 것이 아니다. Connection 단위로 관리되는 캐시와 DB 서버 리소스에도 영향을 줄 수 있다.

따라서 최적화 기능을 활성화하기 전에 실제 실행 방식, 캐시 재사용 여부, 서버 리소스 비용을 함께 검토해야 한다.

---

## 마무리

세 가지 사례는 서로 다른 영역의 문제를 다루고 있지만, 기존에 당연하게 사용하던 구조와 기능의 필요성을 다시 검토했다는 공통점이 있다.

| 사례 | 기존에 당연하게 사용하던 방식 | 다시 검토한 내용 |
|---|---|---|
| 자동승인 스케줄러 | DB 작업 큐와 분리된 스케줄러 | 작업 큐가 정말 필요한가? |
| JPA Transactional | 조회 메서드의 읽기 전용 트랜잭션 | 단순 조회에도 명시적 트랜잭션이 필요한가? |
| PreparedStatement | 서버 측 SQL 준비 및 캐싱 | 실제로 Statement가 재사용되는가? |

성능 개선은 반드시 새로운 기술을 도입하거나 더 빠른 쿼리를 작성하는 방식으로 이루어지는 것은 아니다.

기존 구조가 만들어 내는 불필요한 작업을 제거하거나, 추상화 계층 내부에서 발생하는 비용을 이해하는 것만으로도 개선의 여지를 찾을 수 있다.

> 참고: 각 절의 원문 링크에 나온 사례 및 측정 결과를 바탕으로 정리했다. 포인트 조회 예시, SQL 예제, 캐시 규모 계산, 작업 큐의 일반적인 설계 원칙은 이해를 돕기 위한 추가 설명이며 직접 재현한 실험은 아니다.
