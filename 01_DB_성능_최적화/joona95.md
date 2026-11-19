# SELECT에서는 빨랐는데 UPDATE에서는 느려졌다 — MySQL Tuple IN과 Optimizer

포인트 사용·발행 API의 여러 성능 문제를 개선하던 중, 다수의 포인트 발행 내역을 한 번에 갱신하는 UPDATE 쿼리도 살펴보게 되었다.

기존 쿼리는 `IssueKey`와 `Version`을 비교하는 조건을 OR로 연결하고 있었다. 당시 약 2,000개의 OR 조건이 생성되는 상황에서 대부분의 쿼리는 1초 이내에 처리되었지만, 일부 쿼리에서는 600초 이상 소요되는 극단적인 지연도 발생했다.

이를 개선하기 위해 OR 조건을 Tuple IN으로 변경했다.

MySQL은 특정 조건에서 IN 목록을 이진 탐색으로 평가할 수 있으므로, 다수의 OR 조건을 반복해서 비교하는 것보다 효율적일 수 있다고 판단했다. 실제로 SELECT로 `EXPLAIN ANALYZE`를 수행했을 때 성능이 개선되는 것도 확인했다.

하지만 실제 UPDATE에 적용한 결과는 예상과 달랐다.

2~25초가 소요되는 쿼리의 발생 빈도가 증가했고, 운영 Slow Query 로그에서는 약 24초가 소요되면서 1,400만 행 이상을 검사하는 사례가 확인되었다.

DBA와 실행 계획을 확인한 결과, Tuple IN을 사용한 UPDATE가 PK 인덱스의 특정 범위를 탐색하지 않고 인덱스 전체를 스캔하고 있었다.

결국 **Tuple IN은 유지하되, 대상 데이터를 SELECT로 먼저 조회한 뒤 JOIN을 통해 UPDATE하는 구조**로 변경했다.

그 결과 동일한 1,841건을 갱신하는 DBA 측정에서 수행 시간이 53.87초에서 0.90초로 단축되었다.

그렇다면 MySQL 내부에서는 어떤 차이가 있었던 것일까?

이번 글에서는 이 경험을 바탕으로 조건 평가 최적화와 인덱스 접근 최적화의 차이, 그리고 SELECT와 UPDATE에서 서로 다른 실행 계획이 선택될 수 있는 이유를 살펴보려고 한다.

## 1. 기존 OR 조건에서 Tuple IN으로

기존 UPDATE 쿼리는 다음과 같은 구조였다.

```sql
UPDATE PointIssue
SET AvailableAmount = 0,
    Updated = NOW(),
    Version = Version + 1
WHERE (IssueKey = 101 AND Version = 1)
   OR (IssueKey = 102 AND Version = 3)
   OR (IssueKey = 103 AND Version = 2);
-- 실제로는 더 많은 OR 조건이 생성됨
```

`IssueKey`는 PK이며, `Version`은 기존 갱신 조건에서 함께 검증하던 컬럼이다.

두 값이 모두 일치하는 레코드를 갱신하며, 갱신 시 Version을 1 증가시키는 구조다.

이를 다음과 같이 변경했다.

```sql
UPDATE PointIssue
SET AvailableAmount = 0,
    Updated = NOW(),
    Version = Version + 1
WHERE (IssueKey, Version) IN (
    (101, 1),
    (102, 3),
    (103, 2)
);
```

이처럼 여러 컬럼을 하나의 값 조합으로 묶어 비교하는 문법을 **Row Constructor**라고 한다.

```sql
(IssueKey, Version) = (101, 1)
```

이는 다음 조건과 동일한 의미다.

```sql
IssueKey = 101 AND Version = 1
```

여러 Row Constructor를 IN으로 비교하는 구문을 Row Constructor IN 또는 Tuple IN이라고 한다.

Row Constructor는 여러 컬럼을 하나의 조합으로 비교하기 위한 SQL 표현이며, 성능 최적화만을 위해 만들어진 문법은 아니다.

그렇다면 기존 OR와 Tuple IN은 실제로 어떻게 다르게 동작할까?

## 2. IN이 OR보다 효율적일 수 있는 이유

먼저 인덱스 접근을 제외하고, 이미 읽어 온 행이 WHERE 조건을 만족하는지 평가하는 상황을 생각해 보자.

### OR 조건의 순차 평가

```sql
WHERE (IssueKey = 101 AND Version = 1)
   OR (IssueKey = 102 AND Version = 3)
   OR (IssueKey = 103 AND Version = 2)
```

OR 조건을 순차적으로 평가한다면 첫 번째 조건부터 확인하고, 일치하지 않을 때 다음 조건으로 넘어간다.

조건이 2,000개라면 최악의 경우 한 행마다 2,000개의 조건을 확인해야 할 수 있다.

이때 조건 평가에 필요한 비교 횟수는 O(n)에 비례한다.

물론 MySQL이 OR 조건을 항상 순차적으로 평가하는 것은 아니다. Optimizer는 조건을 분석해 인덱스를 이용하거나 다른 방식으로 최적화할 수 있다.

### IN 조건의 이진 탐색

MySQL은 비교 가능한 상수들로 구성된 IN 목록을 정렬한 후 이진 탐색으로 일치 여부를 판단하는 최적화를 지원한다.

이진 탐색은 정렬된 목록의 중간값부터 비교하면서 탐색 범위를 절반씩 줄이는 방법이다.

2,000개의 값이 있다면 이론적으로 약 11번의 비교 단계로 원하는 값을 찾을 수 있다.

이때 탐색 비용은 O(log n)이다.

따라서 OR를 순차적으로 평가하는 상황이라면, IN 목록의 이진 탐색이 유리할 수 있다.

다만 실제로 이진 탐색이 적용되는지는 값의 타입과 표현식 등의 조건에 따라 달라진다. 특히 Tuple IN이라고 해서 모든 상황에서 O(log n)의 비교 비용이 보장되는 것은 아니다.

그리고 여기서 중요한 점이 있다.

**지금까지 설명한 것은 이미 읽은 행이 조건을 만족하는지 확인하는 방법이지, 테이블에서 해당 행을 찾아가는 방법이 아니다.**

데이터베이스는 인덱스를 통해 애초에 필요한 행만 찾아갈 수도 있다.

## 3. 조건 평가와 인덱스 접근은 서로 다르다

다음 쿼리를 살펴보자.

```sql
SELECT *
FROM PointIssue
WHERE IssueKey IN (101, 103, 105);
```

인덱스를 사용하지 못한다면 MySQL은 테이블의 여러 행을 읽으면서 IssueKey가 IN 목록에 포함되는지 검사해야 할 수 있다.

이때는 조건 평가 비용이 중요해진다.

하지만 IssueKey가 PK라면 어떨까?

MySQL은 B+Tree 인덱스를 이용해 101, 103, 105에 해당하는 레코드를 직접 찾아갈 수 있다.

모든 행을 순회하면서 IN 목록과 비교할 필요가 없는 것이다.

여기서 두 가지 최적화를 구분할 수 있다.

| 구분 | 조건 평가 최적화 | 인덱스 접근 최적화 |
|---|---|---|
| 목적 | 읽은 행의 조건 일치 여부를 빠르게 판단 | 읽어야 할 행 자체를 줄임 |
| 예시 | IN 목록의 이진 탐색 | B+Tree Range Scan |
| 주요 비용 | 조건 비교 | 인덱스 탐색과 데이터 읽기 |

이 두 최적화는 서로 다른 역할을 한다.

IN이 이진 탐색을 지원한다는 이유만으로 OR보다 항상 빠른 것은 아니다.

**인덱스를 통해 필요한 레코드만 찾아갈 수 있다면 행마다 많은 조건을 비교하는 과정 자체가 줄어들기 때문이다.**

그렇다면 MySQL은 WHERE 조건을 보고 어떤 인덱스를 사용할지, 해당 인덱스의 어느 범위를 읽어야 할지 어떻게 판단할까?

이를 이해하기 위해 Optimizer와 Range Scan의 동작을 살펴보자.

## 4. MySQL은 WHERE 조건을 어떻게 인덱스 검색으로 바꿀까?

SQL을 작성할 때 우리는 어떤 데이터를 조회할지 조건을 지정한다.

```sql
SELECT *
FROM PointIssue
WHERE IssueKey IN (101, 103, 105);
```

하지만 이 SQL에는 해당 데이터를 **어떻게 찾아야 하는지**까지 명시되어 있지는 않다.

MySQL은 모든 레코드를 읽고 WHERE 조건을 검사할 수도 있고, 인덱스를 이용해 필요한 레코드만 찾아갈 수도 있다.

어떤 방법이 더 효율적인지 판단하는 것이 **Optimizer**의 역할이다.

Optimizer는 가능한 데이터 접근 방법들을 검토하고, 테이블 통계와 인덱스 정보, 예상 조회 행 수 등을 바탕으로 비용이 낮을 것으로 예상되는 실행 계획을 선택한다.

이 과정에서 인덱스를 이용한 범위 검색 방법을 검토하는 작업이 Range Optimization이다.

### Range Scan이란?

InnoDB의 B+Tree 인덱스는 키를 정렬된 순서로 관리한다.

예를 들어 다음과 같은 PK 값들이 있다고 가정해 보자.

```text
IssueKey
--------
100
101
102
103
104
105
106
```

다음 조건으로 조회한다면 어떨까?

```sql
WHERE IssueKey BETWEEN 102 AND 104
```

인덱스를 사용하지 못한다면 모든 레코드를 읽으면서 조건을 검사해야 할 수 있다.

반면 PK 인덱스를 사용한다면 102가 저장된 위치를 찾고, 인덱스 순서대로 104까지 읽을 수 있다.

```text
100
101
102  ← 탐색 시작
103
104  ← 탐색 종료
105
106
```

이처럼 **인덱스에서 필요한 키 구간을 찾아 해당 범위의 데이터를 읽는 접근 방식**을 Range Scan이라고 한다.

### 여러 값도 Range Scan이 가능하다

그렇다면 다음처럼 연속되지 않은 값을 조회할 때는 어떨까?

```sql
WHERE IssueKey IN (101, 103, 105)
```

이 조건은 IssueKey가 101, 103, 105 중 하나인 레코드만 조회한다는 의미다.

인덱스는 정렬된 순서로 저장되어 있으므로, 가장 작은 값인 101부터 가장 큰 값인 105까지 하나의 범위로 읽는 방법을 생각할 수도 있다.

```text
[101, 105]

101  ← 조회 대상
102  ← 조회 대상 아님
103  ← 조회 대상
104  ← 조회 대상 아님
105  ← 조회 대상
```

하지만 이렇게 접근한다면 실제로 필요하지 않은 102와 104까지 읽은 후 WHERE 조건으로 걸러내야 한다.

조회 대상의 수가 적고 값들이 넓게 분포되어 있다면 불필요하게 읽는 데이터가 크게 늘어날 수 있다.

그렇다면 필요한 값에만 접근할 수는 없을까?

MySQL의 Range Optimizer는 해당 조건을 다음과 같은 여러 검색 구간으로 표현할 수 있다.

```text
[101, 101]
[103, 103]
[105, 105]
```

각 구간은 시작값과 끝값이 동일하다.

예를 들어 `[101, 101]`은 101부터 101까지, 즉 101이라는 값 하나만 포함하는 검색 구간을 의미한다.

이처럼 **하나의 값만 포함하는 검색 구간을 Equality Range**라고 한다.

이러한 범위를 활용하면 MySQL은 인덱스에서 필요한 키 위치를 찾아가며, 102나 104처럼 조건에 해당하지 않는 인덱스 엔트리를 검사하는 작업을 줄일 수 있다.

물론 여러 검색 구간을 각각 탐색하는 데도 비용이 발생한다. 따라서 모든 상황에서 검색 범위를 잘게 나누는 것이 더 효율적이라는 의미는 아니다.

중요한 것은 **Range Optimizer가 WHERE 조건을 인덱스 검색에 활용할 수 있는 구간으로 표현하고, 이를 바탕으로 효율적인 접근 방법을 검토한다는 점**이다.

참고로 `IssueKey = 101`처럼 PK에 대한 단일 동등 조건은 SELECT 실행 계획에서 `const` 접근으로 나타날 수 있다.

Equality Range는 조건을 검색 구간으로 표현하는 개념이고, `const`와 `range`는 실제로 선택된 테이블 접근 방식이라는 차이가 있다.

### 그렇다면 Range Optimizer는 무슨 일을 할까?

앞의 예시에서 우리는 `IssueKey IN (101, 103, 105)`라는 조건이 세 개의 인덱스 검색 구간에 해당한다는 것을 알 수 있었다.

하지만 실제 WHERE 조건이 항상 이렇게 간단한 것은 아니다.

```sql
WHERE (IssueKey = 101 AND Version = 1)
   OR (IssueKey = 103 AND Version = 2)
```

이처럼 여러 AND와 OR 조건이 조합되어 있다면 어떤 부분을 인덱스 검색에 활용할 수 있을까?

Range Optimizer는 이러한 조건을 분석해 **인덱스를 이용해 검색할 수 있는 키 구간을 구성하는 작업**을 수행한다.

예를 들어 `IssueKey`가 PK이고, `Version`은 해당 인덱스에 포함되지 않는다고 가정해 보자.

이 경우 다음 두 구간을 PK 인덱스 검색 후보로 구성할 수 있다.

```text
[101, 101]
[103, 103]
```

MySQL은 PK 인덱스로 해당 IssueKey를 가진 레코드를 찾은 뒤 Version 조건을 추가로 평가할 수 있다.

즉, WHERE 조건 전체를 인덱스 탐색에 사용하는 것이 아니라, **인덱스로 검색할 수 있는 조건과 이후에 평가해야 하는 조건을 구분하는 것**이다.

이 과정이 필요한 이유는 단순하다.

**WHERE 조건은 SQL의 논리적인 표현이고, 인덱스 검색에는 구체적인 검색 구간이 필요하기 때문이다.**

### 검색 구간을 왜 별도의 자료구조로 관리할까?

여기서 한 가지 의문이 생길 수 있다.

실제 데이터는 이미 B+Tree 인덱스에 저장되어 있는데, 왜 검색 구간을 별도의 자료구조로 구성할까?

두 구조의 역할이 다르기 때문이다.

**B+Tree는 실제 인덱스 데이터를 저장하고 탐색하는 자료구조다.**

반면 Range Optimizer가 구성하는 검색 구간은 해당 인덱스를 *어떻게 탐색할지 판단하기 위한 정보*다.

예를 들어 다음과 같이 구분할 수 있다.

```text
[실행 계획을 만드는 과정]

WHERE 조건
     ↓
인덱스로 검색 가능한 조건 분석
     ↓
검색할 키 구간 구성
[101,101], [103,103]
     ↓
예상 비용 계산
     ↓
Range Scan 등 실행 방법 선택


[실제 쿼리 실행 과정]

선택된 실행 계획
     ↓
InnoDB B+Tree 탐색
     ↓
실제 레코드 조회
```

MySQL은 이러한 검색 구간을 내부적으로 표현하기 위해 `SEL_TREE`, `SEL_ARG` 등의 자료구조를 사용한다.

- `SEL_TREE`는 인덱스별 검색 조건과 범위 조합을 표현하는 구조다.
- `SEL_ARG`는 인덱스 키의 검색 구간과 구간 사이의 관계를 표현하는 데 사용된다.

이들은 실제 데이터를 저장하는 B+Tree와 다르다.

B+Tree는 실제 테이블의 인덱스로 존재하지만, Range Optimizer의 내부 자료구조는 SQL의 조건을 분석해 검색 계획을 세우는 과정에서 사용된다.

이렇게 별도의 표현을 구성하면 MySQL은 복잡한 AND/OR 조건에서도 검색 가능한 구간을 관리하고, 여러 인덱스 접근 방법의 비용을 비교할 수 있다.

즉, 실제 인덱스를 새로 만드는 것이 아니라 **기존 인덱스의 어떤 부분을 검색하면 되는지 표현하는 것**이다.

### 검색 구간을 만들었다면 Range Scan을 실행할까?

그렇지는 않다.

Range Optimizer가 인덱스로 검색 가능한 구간을 구성했다고 해서 반드시 Range Scan이 실행되는 것은 아니다.

Optimizer는 해당 Range Scan의 예상 비용을 다른 접근 방법과 비교한다.

경우에 따라 필요한 구간을 찾아가는 것보다 전체 인덱스나 테이블을 스캔하는 방법이 더 저렴하다고 판단할 수도 있다.

따라서 전체 과정을 정리하면 다음과 같다.

```text
WHERE 조건 분석
       ↓
인덱스 검색에 사용할 수 있는 조건 추출
       ↓
검색 구간 구성
       ↓
예상 접근 비용 계산
       ↓
다른 접근 방법과 비용 비교
       ↓
최종 실행 계획 선택
       ↓
실제 데이터 조회
```

여기서 핵심은 **Range Scan을 사용할 수 있는 조건인지 판단하는 것과 실제 Range Scan을 선택하는 것은 별개의 문제**라는 점이다.

이 차이는 이후 실제 UPDATE에서 발생한 성능 문제와도 연결된다.

## 5. Row Constructor IN은 어떻게 Range Scan으로 최적화될까?

이제 처음의 Tuple IN 조건으로 돌아가 보자.

```sql
WHERE (IssueKey, Version) IN (
    (101, 1),
    (102, 3),
    (103, 2)
)
```

이는 논리적으로 다음과 동일하다.

```sql
WHERE (IssueKey = 101 AND Version = 1)
   OR (IssueKey = 102 AND Version = 3)
   OR (IssueKey = 103 AND Version = 2)
```

앞에서 살펴본 것처럼 MySQL은 AND와 OR로 조합된 조건을 분석해 인덱스 검색 구간을 구성할 수 있다.

그렇다면 Tuple IN도 같은 방식으로 처리할 수 있을까?

### 인덱스 구성에 따라 검색 구간이 달라진다

먼저 `(IssueKey, Version)` 복합 인덱스가 존재한다고 가정해 보자.

B+Tree에는 두 컬럼을 조합한 키가 인덱스 순서에 따라 정렬되어 있다.

```text
(100, 1)
(101, 1)  ← 검색 대상
(101, 2)
(102, 1)
(102, 3)  ← 검색 대상
(103, 1)
(103, 2)  ← 검색 대상
(104, 1)
```

Range Optimizer가 Tuple IN을 인덱스 검색 구간으로 변환할 수 있다면 다음과 같이 표현할 수 있다.

```text
[(101, 1), (101, 1)]
[(102, 3), (102, 3)]
[(103, 2), (103, 2)]
```

이 경우 B+Tree에서 두 컬럼의 조합에 해당하는 인덱스 엔트리를 탐색할 수 있다.

반면 `IssueKey`만 PK로 존재한다면 어떨까?

이 경우에는 IssueKey로 후보 레코드를 찾은 뒤 Version을 추가로 검사하는 접근이 가능하다.

```text
IssueKey = 101 → Version = 1 검사
IssueKey = 102 → Version = 3 검사
IssueKey = 103 → Version = 2 검사
```

즉, Tuple IN을 사용했다고 해서 반드시 두 컬럼으로 구성된 복합 인덱스가 필요한 것은 아니다.

다만 각 IssueKey와 Version의 조합은 유지되어야 한다.

단순히 `IssueKey IN (...) AND Version IN (...)`으로 변경하면 기존에 지정하지 않은 조합까지 허용할 수 있으므로 원래 조건과 의미가 달라진다.

### MySQL 5.7.3에서는 무엇이 개선되었을까?

Row Constructor 문법 자체는 MySQL 5.7 이전부터 존재했다.

하지만 문법을 사용할 수 있다는 것과 해당 조건을 효율적인 인덱스 검색으로 처리할 수 있다는 것은 별개의 문제였다.

MySQL 5.7.3 이전에는 다음 형태의 Row Constructor IN을 Range Optimizer가 인덱스 검색 구간으로 변환하지 못하는 경우가 있었다.

```sql
WHERE (a, b) IN (
    (1, 2),
    (3, 4)
)
```

반면 다음처럼 직접 AND/OR로 작성하면 Range Scan을 활용할 수 있었다.

```sql
WHERE (a = 1 AND b = 2)
   OR (a = 3 AND b = 4)
```

왜 논리적으로 동일한 조건인데 이러한 차이가 있었을까?

**Optimizer가 SQL 조건을 내부 검색 구간으로 변환할 수 있는 범위에 차이가 있었기 때문이다.**

기존 Range Optimizer는 AND/OR 형태의 조건을 인덱스 검색 구간으로 변환할 수 있었다.

반면 여러 Row Constructor를 포함하는 IN 조건은 동일한 논리적 의미를 가지더라도 Range Optimizer가 사용하는 내부 표현으로 변환하는 기능이 제한되어 있었다.

MySQL 5.7.3에서는 이러한 변환 기능이 개선되었다.

공식 Worklog WL#7019에서는 Row Constructor IN을 동등한 AND/OR 조건과 같은 Range Scan 실행 계획으로 처리할 수 있도록 하는 것을 목표로 명시하고 있다.

예를 들어 Worklog에서 소개한 기존 실행 계획은 다음과 같았다.

| 항목 | 최적화 이전 | 최적화 이후 |
|---|---|---|
| 접근 방식 | `index` | `range` |
| 예상 조회 행 수 | 4,098 | 2 |
| 검색 방식 | 전체 인덱스 스캔 | 필요한 범위 탐색 |

이 사례에서 개선된 것은 Row Constructor IN을 인덱스 범위 검색에 활용할 수 있게 되었다는 점이다.

즉, **Tuple IN이 OR보다 항상 빠르게 실행되도록 만든 것이 아니라, 기존 OR에서 가능했던 인덱스 검색 최적화를 Tuple IN에서도 적용할 수 있도록 한 것**이다.

다만 최종 실행 계획은 Range Optimizer 이외의 요소에도 영향을 받으므로, 두 구문이 언제나 동일한 실행 계획을 선택하는 것은 아니다.

### 그렇다면 OR를 Tuple IN으로 변경한 것은 의미가 없었을까?

그렇지는 않다.

앞에서 살펴본 것처럼 IN은 조건 평가 과정에서 이진 탐색의 이점을 얻을 수 있다.

따라서 다수의 OR 조건을 순차적으로 평가하는 상황이라면 Tuple IN이 더 효율적일 가능성은 있다.

하지만 인덱스 접근 관점에서는 기존 OR 조건도 Range Scan을 사용할 수 있다.

즉, **조건 평가 비용이 줄어드는 것과 실제 읽어야 하는 인덱스 범위가 줄어드는 것은 서로 다른 문제**다.

이번 사례에서도 처음에는 조건 평가 비용의 개선을 기대해 Tuple IN으로 변경했고, SELECT에서는 성능 개선을 확인했다.

하지만 실제 UPDATE에서는 예상과 다른 실행 계획이 선택되었다.

## 6. 운영에서 확인한 문제 — PK 전체 스캔

지금까지 살펴본 것처럼 MySQL은 OR와 Tuple IN 모두 인덱스 검색 구간으로 변환할 수 있다.

하지만 Range Scan이 가능한 조건이라고 해서 실제 실행에서도 반드시 Range Scan이 선택되는 것은 아니다.

이번 UPDATE에서도 바로 이러한 차이가 나타났다.

기존 OR UPDATE에서는 대부분 1초 이내에 처리되었지만, 일부 요청에서 600초가 넘는 극단적인 지연이 발생했다.

이를 Tuple IN으로 변경하고 SELECT에서 성능 개선을 확인했지만, 실제 UPDATE에서는 2~25초 구간의 쿼리가 빈번해졌다.

운영 Slow Query 로그와 실행 계획에서 확인한 한 사례는 다음과 같다.

| 항목 | 측정 결과 |
|---|---:|
| `Query_time` | 24.101초 |
| `Rows_examined` | 14,326,022 |
| 실행 계획 `type` | `index` |
| 선택된 인덱스 `key` | `PRIMARY` |
| 실행 계획 예상 `rows` | 14,265,292 |

중요한 것은 `key = PRIMARY`라는 사실이 아니다.

**`type = index`였다는 점이다.**

MySQL의 실행 계획에서 `type = index`는 인덱스 전체를 순회하는 Full Index Scan을 의미한다.

반면 `type = range`는 인덱스의 특정 검색 구간만 탐색할 수 있는 접근 방식이다.

이번 쿼리는 PK 인덱스를 사용하고 있었지만, 필요한 레코드만 찾는 대신 인덱스 전체를 순회하는 계획이 선택된 것이다.

운영 Slow Query 로그에서도 약 1,432만 행을 검사한 것으로 나타났다.

즉, 인덱스가 없어서 느린 것이 아니라 **인덱스를 사용하면서도 지나치게 많은 데이터를 읽는 것이 문제**였다.

### 왜 UPDATE에서는 Range Scan이 선택되지 않았을까?

MySQL은 Row Constructor IN의 Range Scan을 지원한다.

하지만 지원한다는 사실과 모든 UPDATE에서 해당 실행 계획을 선택한다는 것은 다르다.

MySQL은 SELECT와 UPDATE를 최적화할 때 서로 다른 실행 계획을 선택할 수 있다.

SELECT는 조건에 맞는 데이터를 읽는 것이 목적이지만, UPDATE는 실제 행을 수정해야 한다.

이 과정에서 행 잠금, Undo/Redo Log 기록, 인덱스 유지 관리 등의 작업도 발생한다.

또한 대량의 Tuple IN 조건을 처리할 때는 검색 구간을 구성하는 최적화 비용과 메모리 사용량도 고려할 필요가 있다.

MySQL은 `range_optimizer_max_mem_size`와 같은 제약으로 인해 Range Optimization을 포기하거나, 비용 추정에 따라 다른 접근 방법을 선택할 수 있다.

이러한 요소들은 실제 UPDATE의 실행 계획을 분석할 때 고려해야 할 부분이다.

다만 이번 사례에서 어떤 내부 판단이 Full Index Scan 선택으로 이어졌는지까지는 측정 결과만으로 특정할 수 없다.

확인된 것은 **실제 UPDATE에서 PK 인덱스 전체를 순회했고, 그로 인해 과도한 데이터 탐색이 발생했다는 사실**이다.

이 문제를 해결하기 위해 쿼리 구조를 변경했다.

## 7. SELECT 결과를 JOIN하는 방식으로 변경

DBA의 권고에 따라 최종적으로 UPDATE 대상 IssueKey를 SELECT로 조회한 뒤 JOIN하는 구조로 변경했다.

실제 개선 SQL의 구조는 다음과 같다.

```sql
UPDATE PointIssue A
INNER JOIN (
    SELECT P.IssueKey
    FROM PointIssue P
    WHERE (P.IssueKey, P.Version) IN (
        (101, 1),
        (102, 3),
        (103, 2)
    )
) B ON B.IssueKey = A.IssueKey
SET A.AvailableAmount = 0,
    A.Updated = NOW(),
    A.Version = A.Version + 1;
```

실제 코드에서는 IN 목록을 동적으로 구성했지만, 여기서는 이해를 돕기 위해 일부 값만 표현했다.

핵심은 **기존 Tuple IN을 없앤 것이 아니라 SELECT 내부로 옮겼다는 것**이다.

### 실행 계획은 어떻게 달라졌을까?

변경 전에는 단일 UPDATE에서 Tuple IN 조건을 평가하면서 Full Index Scan이 발생했다.

반면 JOIN 구조로 변경한 이후에는 DBA 측정에서 `range + eq_ref` 접근이 확인되었다.

개념적인 흐름은 다음과 같다.

```text
[내부 SELECT]
(IssueKey, Version) IN (...)
              │
              ▼
Range Scan
대상 IssueKey 조회
              │
              ▼
[외부 UPDATE JOIN]
A.IssueKey = B.IssueKey
              │
              ▼
eq_ref
PK로 갱신 대상 조회
              │
              ▼
AvailableAmount = 0
Version = Version + 1
```

`range`는 인덱스의 특정 검색 구간을 읽는 접근 방식이다.

`eq_ref`는 JOIN에서 앞서 읽은 행마다 PK 또는 UNIQUE 인덱스를 이용해 최대 한 행을 찾는 접근 방식이다.

즉, 내부 SELECT에서는 조건에 해당하는 IssueKey를 효율적으로 찾고, 외부 UPDATE에서는 그 결과를 이용해 PK로 갱신 대상에 접근할 수 있게 된 것이다.

이 구조에서는 갱신 대상을 선정하는 작업과 실제 레코드를 갱신하는 작업이 서로 다른 쿼리 블록으로 표현된다.

### 실제 개선 결과

DBA가 1,841건의 갱신을 기준으로 측정한 결과는 다음과 같았다.

| 구분 | 기존 Tuple IN UPDATE | UPDATE JOIN |
|---|---:|---:|
| 수행 시간 | 53.87초 | 0.90초 |
| 변경된 행 수 | 1,841건 | 1,841건 |
| 실행 계획 | PK 전체 스캔 | `range + eq_ref` |

**약 59.9배의 성능 개선이 확인되었다.**

기존에는 대상 레코드를 찾기 위해 PK 인덱스 전체를 순회했지만, 개선 후에는 Range Scan과 PK 기반 JOIN으로 필요한 레코드에 접근할 수 있었다.

이처럼 실행 계획에서 데이터를 탐색하는 방식이 달라지면서 수행 시간이 크게 단축되었다.

### 추가로, Derived Table은 항상 별도로 실행될까?

이번 개선처럼 SELECT 결과를 테이블처럼 사용하는 구조를 Derived Table이라고 한다.

MySQL은 Derived Table을 외부 쿼리에 병합하는 **Derived Table Merge** 최적화를 수행할 수 있다.

따라서 SELECT를 JOIN으로 감쌌다고 해서 반드시 SELECT 결과가 별도의 중간 결과 집합으로 생성되는 것은 아니다.

이러한 병합을 방지하는 방법 중 하나가 `LIMIT`이다. MySQL은 Derived Table 내부에 `LIMIT`이 존재하면 외부 쿼리와 병합하지 않는다.

다만 `LIMIT`은 실제 조회 대상의 수까지 제한하므로, 병합 방지만이 목적이라면 `NO_MERGE` 힌트를 활용하는 방법도 있다.

이번 개선 SQL에는 이러한 힌트나 LIMIT을 추가하지 않았지만, 실제 실행 계획에서 `range + eq_ref`를 확인했다.

중요한 것은 Derived Table이라는 문법 자체가 아니라, **Optimizer가 실제로 어떤 접근 경로와 JOIN 순서를 선택했는지**다.

## 마무리

처음에는 많은 OR 조건을 Tuple IN으로 변경하면 조건 평가 비용을 줄일 수 있을 것이라고 생각했다.

MySQL은 특정 조건에서 IN 목록을 이진 탐색으로 평가할 수 있고, 실제 SELECT 실행 결과에서도 개선을 확인했기 때문이다.

하지만 실제 UPDATE에서는 다른 결과가 나타났다.

운영 실행 계획에서 `type = index`, `key = PRIMARY`가 선택되었고, 약 1,426만 행을 읽는 Full Index Scan이 발생했다.

이후 Tuple IN을 SELECT 내부로 옮기고 JOIN을 통해 UPDATE하도록 변경하면서 실행 계획은 `range + eq_ref`로 개선되었다.

동일한 1,841건을 갱신하는 데 걸리는 시간도 53.87초에서 0.90초로 단축되었다.

이번 경험을 통해 세 가지를 알게 되었다.

**첫째, 조건 평가 최적화와 인덱스 접근 최적화는 서로 다르다.**

IN 목록의 이진 탐색이 효율적이라고 해서 실제 레코드를 찾는 과정까지 효율적인 것은 아니다.

**둘째, Row Constructor IN과 OR 조건은 모두 Range Scan을 활용할 수 있지만, 실제로 어떤 접근 방식이 선택될지는 별개의 문제다.**

**셋째, SELECT에서 확인한 실행 계획과 성능 개선을 UPDATE에서도 그대로 기대해서는 안 된다.**

이번 사례에서는 Tuple IN을 유지하면서도 SELECT와 UPDATE의 실행 구조를 분리함으로써 문제를 개선할 수 있었다.

결국 SQL 튜닝에서 중요한 것은 단순히 조건을 어떻게 표현했는지가 아니라, **Optimizer가 그 조건을 어떤 실행 계획으로 변환하고 실제로 얼마나 많은 데이터를 읽는지 확인하는 것**이라는 생각이 들었다.

---

## 참고 자료

- [MySQL — Comparison Functions and Operators](https://dev.mysql.com/doc/refman/8.4/en/comparison-operators.html)
- [MySQL — Range Optimization](https://dev.mysql.com/doc/refman/8.4/en/range-optimization.html)
- [MySQL WL#7019 — Row Constructor IN Range Optimization](https://dev.mysql.com/worklog/task/?id=7019)
- [MySQL — Row Constructor Expression Optimization](https://dev.mysql.com/doc/refman/8.4/en/row-constructor-optimization.html)
- [MySQL — Optimizing UPDATE Statements](https://dev.mysql.com/doc/refman/8.4/en/update-optimization.html)
- [MySQL — Derived Table Optimization](https://dev.mysql.com/doc/refman/8.4/en/derived-table-optimization.html)