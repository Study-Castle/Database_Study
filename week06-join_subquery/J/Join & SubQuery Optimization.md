## NL 조인(Nested Loop Join)

NL 조인은 MySQL이 기본적으로 사용하는 조인 방식이며, **인덱스를 활용한 조인**이라고 이해하면 가장 쉽다.

### 작동 방식

1. 드라이빙 테이블(먼저 읽는 테이블)의 row 하나를 가져옴
2. 이 row의 조인 조건을 이용해
    
    드리븐 테이블(나중에 읽는 테이블)에서 해당 row를 찾음
    
3. 이때 드리븐 테이블의 조인 컬럼에 **인덱스가 있으면 매우 빠름**
    
    (인덱스 = 랜덤 I/O 최소화)
    

### 인덱스의 중요성

드리븐 테이블에서 조인 조건을 만족하는 row를 찾는 과정이 핵심이기 때문에

- **드리븐 테이블의 조인 컬럼에 인덱스가 반드시 필요**
- 드라이빙 테이블은 1번만 full scan 하므로 인덱스 없어도 큰 문제 없음

**드리븐 테이블 인덱스 없음?**

→ 드라이빙 테이블 row 수만큼 **full table scan 반복**

→ 최악의 경우 **O(N × M)** 발생 → 지옥의 성능

## 예시로 이해하는 NL 조인

```sql
SELECT post.title, post.content
FROM user
JOIN post ON user.id = post.user
WHERE user.nickname = 'User920508';
```

```sql
-> Nested loop inner join  (cost=4315.18 rows=9444) (actual time=0.099..5.375 rows=15 loops=1)
    -> Filter: (`user`.nickname = 'User920508')  (cost=1009.65 rows=985) (actual time=0.067..5.257 rows=1 loops=1)
        -> Table scan on user  (cost=1009.65 rows=9854) (actual time=0.063..3.520 rows=10000 loops=1)
    -> Index lookup on post using user (user=`user`.id)  (cost=2.40 rows=10) (actual time=0.031..0.116 rows=15 loops=1)
```

### 실행 계획 해석

- `user` 테이블(드라이빙)을 먼저 full scan하여 nickname 조건 만족 row 찾음
- 찾은 user.id 값으로
- `post` 테이블(드리븐)에서
    
    **post.user 인덱스를 이용한 index lookup 수행**
    
- 드리븐 테이블에서 빠르게 15개의 row 검색 완료

→ **인덱스 덕분에 드리븐 테이블 접근 비용이 최소화됨**

## 드라이빙 테이블 / 드리븐 테이블 개념

| 역할 | 의미 |
| --- | --- |
| **드라이빙 테이블** | 조인에서 먼저 읽는 테이블 |
| **드리븐 테이블** | 드라이빙 테이블의 row로 조건 검색되는 테이블 |

MySQL 옵티마이저는

- **드라이빙 = 적은 row가 나오는 테이블**
- **드리븐 = 조인 조건 컬럼에 인덱스가 있는 테이블**
    
    로 선택하는 경향이 강함.
    

## Inner Join vs Outer Join

### Inner Join

- 두 테이블에서 **조건에 부합하는 row만** 결과 반환
- 조인 조건 매칭 안 되면 결과에서 제외

### Outer Join

- **드라이빙 테이블의 모든 row는 반드시 결과에 포함**
- 드리븐 테이블에 매칭 row가 없으면 NULL 로 반환

→ Outer Join은 NL 조인의 구조는 동일하지만

**드리븐에서 매칭 실패해도 드라이빙 row를 버리지 않음**

## MySQL의 NL 조인 특징 (InnoDB 기준)

- 인덱스 기반 조인에서 가장 빠른 방식
- 하지만 조인 컬럼에 인덱스가 없으면 성능이 급격히 나빠짐
- 인덱스 없는 조인을 위해 다른 조인 알고리즘도 존재

그중 하나가 바로 **BNL 조인**이다.

## BNL 조인(Block Nested Loop Join)

MySQL의 예전 버전(8.0.18 이전)에서 사용됨.

### 언제 사용되나?

- 드리븐 테이블에 인덱스가 없음
- 조인 조건 컬럼이 인덱스를 타지 못함
- NL 조인을 사용할 수 없는 상황

### 작동 방식

1. 드라이빙 테이블의 일부 row 블록을 **조인 버퍼**에 올림
2. 드리븐 테이블을 full scan 하면서 버퍼에 있는 값과 비교
3. 매칭되는 row만 결과로 생성

→ 메모리 버퍼를 사용해 full scan 비용을 줄이는 조인 방식

## MySQL 8.0.18 이후

BNL 조인은 더 이상 기본 사용 X

→ **더 빠르고 효율적인 Hash Join으로 대체**

따라서 BNL 조인은

옛날 MySQL이 인덱스 없는 조인을 처리하던 방식 정도로 기억하면 됨.

## NL 조인의 튜닝 포인트

NL 조인의 성능은 드리븐 테이블 접근 비용에 의해 전부 결정

- 드라이빙 테이블에서 row 수 줄이기
- 드리븐 테이블에서 인덱스 기반으로 빠르게 찾게 만들기

### 드리븐 테이블의 조인 컬럼에 반드시 인덱스 생성

**NL 조인의 핵심 규칙**

드리븐 테이블(나중에 읽는 테이블)의 조인 조건 컬럼에 인덱스가 없으면 지옥

```sql
SELECT *
FROM A
JOIN B ON A.id = B.a_id
```

여기서 **B.a_id에 인덱스 없으면**

→ A의 row 수만큼 B 테이블 full scan 반복

→ 최악의 경우 O(N×M) 발생 = 최악

**가장 중요한 튜닝 포인트**

- 드리븐 테이블 조인 키에 반드시 인덱스 있어야 함
- FK면 보통 자동으로 인덱스가 걸림

### 드라이빙 테이블은 가능한 row 수가 적게 나오는 테이블

드라이빙 테이블에서 많이 나올수록

→ 드리븐 테이블 접근 횟수도 증가

즉, **WHERE 조건 절로 많이 걸러지는 테이블이 드라이빙이 되어야 한다.**

옵티마이저가 자동으로 결정하지만 잘못 결정될 때도 있음.

**드라이빙 테이블 row 수 줄이는 방법**

- WHERE 절에 카디널리티 높은 컬럼 조건 추가
- LIMIT 사용 시 강제로 드라이빙 유도 가능
- 쿼리 구조 바꿔서 조인 순서 변경 (`STRAIGHT_JOIN` 힌트)

```sql
SELECT /*+ STRAIGHT_JOIN */ ...
FROM small_table s
JOIN big_table b ON s.id = b.s_id
```

### JOIN 조건에서 함수 사용 금지

MySQL은 조인 조건에서 컬럼에 함수가 적용되면

**절대 인덱스를 못 탐**

```sql
JOIN post p ON DATE(p.created_at) = u.created_date
```

→ p.created_at 인덱스 못 사용 → full scan 발생

**해결**

- 조인 조건에서 함수 제거
- 파생 컬럼 저장(예: created_date 컬럼 추가)
- 인덱스 컬럼 건드리지 말기

### 데이터 타입, 문자셋 동일하게 맞추기

조인 조건 양쪽 컬럼의 타입이 다르면 인덱스 못 탐

```sql
user.id (INT)
post.user_id (VARCHAR)
```

→ 비교 전 타입 변환 → 인덱스 완전 무효화 → full scan 발생

**반드시 데이터 타입을 통일하자.**

- 숫자 ↔ 문자열 비교 금지
- utf8mb4 ↔ latin1 조인 금지 (Collation 변경해야 함)

### 카디널리티 높은(= 중복 낮은) 컬럼을 조인 키로 사용

카디널리티 낮으면 조인 후 row 수가 크게 늘어나므로

드리븐 테이블 접근 횟수도 커짐

예) 성별(gender), 상태(status) 같은 컬럼으로 조인 하면 안됨

조인 키는 가능한 **고유성이 높은 값**이 유리함

(예: id, email, unique key 등)

### 필요한 컬럼만 SELECT (Covering Index 활용)

NL 조인에서는 드리븐 테이블 접근이 비용의 대부분

만약 **커버링 인덱스**가 가능하면 PK lookup을 하지 않아도 됨.

```sql
SELECT id, name
FROM user
JOIN post ON user.id = post.user_id;
```

→ `post (user_id, id, name)` 형태로 인덱스를 구성하면

→ post 테이블 데이터 페이지 읽을 필요 없음

→ 인덱스에서 바로 끝남 → 압도적으로 빠름

**커버링 인덱스 활용 시**

- 인덱스 읽기만으로 쿼리 끝남
- 랜덤 I/O 대폭 감소

### 드라이빙 테이블 축소를 위한 적절한 WHERE 절 추가

```sql
WHERE created_at >= NOW() - INTERVAL 1 DAY
```

→ 최근 데이터만 읽게 하여 드리븐 접근 횟수 감소

**조인하기 전에 먼저 row 수를 줄이는 것**

= NL 조인 튜닝의 핵심

### 인덱스가 있어도 선행 컬럼이 맞지 않으면 못 탄다

다중 인덱스

```sql
INDEX idx (dept_no, emp_no)
```

WHERE 절이

```sql
WHERE emp_no = 10
```

만 쓰면 인덱스 못 탐

→ NL 조인에서도 동일

반드시 인덱스 선행 컬럼을 조인 조건 또는 WHERE에서 사용해야 한다.

### 조인 순서를 강제로 고정(STRAIGHT_JOIN)

옵티마이저가 잘못된 순서를 선택해 NL 조인이 폭발할 때 사용.

```sql
SELECT /*+ STRAIGHT_JOIN */
FROM A
JOIN B ON A.id = B.a_id
```

→ A를 강제로 드라이빙 테이블로 만든다.

### 조인에 불필요한 ORDER BY / GROUP BY 제거

ORDER BY, GROUP BY가 조인보다 먼저 적용되면

→ 정렬을 위한 임시 테이블 발생

→ 조인 순서가 뒤틀릴 수 있음

가능하면 조인 후 정렬하도록 쿼리 구조 조정

## Sort-Merge Join (MySQL은 미지원)

NL 조인의 단점을 해결하려고 만들어진 조인 방식

### 개념

- 양쪽 테이블을 조인 컬럼 기준으로 정렬(Sort)
- 정렬된 두 스트림을 순차적으로 Merge하며 조인

### 장점

- 랜덤 I/O 거의 없음
- 대량 데이터 조인에 매우 효율적
- 인덱스 없어도 정렬로 해결 가능

### 단점

- 정렬 비용이 크다
- 메모리 사용 많음

**MySQL에서는 지원 안함**

→ MySQL은 대신 **Hash Join**으로 대체했다.

## Hash Join (MySQL 8.0.18+)

NL 조인의 **대량 탐색 시 랜덤 I/O 폭증 문제**와

Sort-Merge Join의 **정렬 오버헤드** 문제를 동시에 개선하는 조인 방식

### Hash Join 동작 원리

**Build 단계 (해시 테이블 생성)**

- 작은 테이블의 조인 컬럼을 기준으로 해시 테이블 생성
- 메모리에 저장(join buffer)

**Probe 단계**

- 큰 테이블을 스캔하면서
    
    조인 컬럼 값을 이용해 해시 테이블을 탐색
    

### Hash Join의 장점

| 문제점 | Hash Join의 해결 방식 |
| --- | --- |
| NL 조인: 랜덤 I/O 폭탄 | 해시 테이블은 메모리 기반 → 랜덤 I/O 제거 |
| Sort-Merge 조인: 정렬 비용 | 정렬 필요 없음 |
| 인덱스 필수 조건 | 인덱스 없이도 빠른 조인 가능 |

**대량 데이터 + 인덱스 없는 조인** 상황에서 최고의 성능

### Hash Join 제약

- 해시 테이블은 **join_buffer_size**를 넘을 경우 디스크로 spill → 성능 저하
- 매우 큰 테이블 조인 시 메모리 사용량 많아짐

## Hash Join 사용 조건(MySQL)

### "=" 조건이 있는 조인 (가장 일반적인 케이스)

```sql
SELECT * FROM t1
JOIN t2 ON t1.c1 = t2.c1;
```

단, **조인 컬럼에 인덱스가 없을 때** Hash Join 자동 사용

→ 인덱스가 있으면 대부분 NL Join 선택

### "=" 조건이 없어도 가능 (8.0.20+)

```sql
t1.c1 > t2.c1
```

등호가 없어도 Hash Join 가능하지만

인덱스가 있으면 MySQL은 여전히 NL 조인을 더 선호

### 카테시안 곱(조인 조건 없음)

```sql
SELECT * FROM post JOIN comment;
```

조건 없이 조인하면 Hash Join 사용.

## Hash Join 강제 / 차단 힌트

### Hash Join 유도(강제)

```sql
/*+ BNL(t2) */
```

BNL은 이전 블록 NL Join이었지만 이제 Hash Join 힌트 역할도 가능

### Hash Join 금지

```sql
/*+ NO_BNL(t2) */
```

## 예제로 보는 MySQL 동작

### "=" 조건 & 인덱스 없음 → Hash Join

```
post.createdAt = comment.created_at
```

인덱스 없으면 Hash Join 자동 선택

### "=" 조건 & 양쪽 인덱스 있음 → NL Join

MySQL은 인덱스가 있는 조인에서는 무조건 NL Join을 우선 선택

이게 더 빠르기 때문

### “>” 조건(부등호) & 인덱스 없음 → Hash Join 가능

하지만 조인 건수가 매우 많으면 Hash Join 성능이 안 좋을 수 있음

### 조인 조건 없음 → 무조건 Hash Join

## 전체 조인 방식 비교

| 조인 방식 | 장점 | 단점 | MySQL 지원 |
| --- | --- | --- | --- |
| NL Join | 인덱스 기반 소량 조인 최강 | 인덱스 없으면 성능 폭망 | ✔ |
| Sort-Merge Join | 대용량 조인에 강함, 정렬 기반 | 정렬 오버헤드 큼 | ❌ |
| Hash Join | 인덱스 없어도 OK, 대량 조인 강함 | 메모리 많이 씀, spill 시 느림 | ✔ (8.0.18+) |

## NL 조인 vs Hash Join 성능 요약

- 인덱스가 있다 → NL Join이 더 빠름
- 인덱스가 없다 → Hash Join이 훨씬 유리
- 대량 데이터 조인 → Hash Join 우세
- 조인 조건 없음 → Hash Join 사용

## Hash Join 튜닝 포인트

Hash Join은 인덱스 없는 대량 조인에서 NL 조인 보다 뛰어난 성능을 보이지만

세팅이나 테이블 크기에 따라 성능이 극단적으로 좋아지거나 망가질 수 있음

### join_buffer_size(가장 중요)

**핵심 개념**

- Hash Join은 driving table(더 작은 테이블)을 해시 테이블로 만들어 메모리에 올린다.
- 이 메모리 공간이 **join_buffer_size**다.
- 이 버퍼를 넘으면 디스크로 spill → Hash Join 속도가 폭망함

**튜닝 방법**

- 기본값: 매우 낮음 (256KB, 512KB 등)
- 해시 조인이 자주 쓰이는 환경이라면 64MB~512MB 정도로 조정 가능

```sql
SET GLOBAL join_buffer_size = 128 * 1024 * 1024;  -- 128MB
```

**주의사항**

**세션 단위 설정이 아님 → 모든 연결에 적용됨 → 너무 크게 잡으면 메모리 터짐**

Hash Join 가능성이 높은 쿼리를 파악한 후 설정

### 해시 테이블 크기 줄이기 (Build side 최적화)

Hash Join의 핵심은 **작은 테이블을 Build side**로 두는 것

**최적화 규칙**

1. Build side는 반드시 row 수가 작은 테이블
2. Build side는 가능한 한 선택도가 높은 조건을 먼저 걸어 row 수를 줄여야 함
3. Build side에 불필요한 컬럼까지 해시 만들지 않도록 SELECT 컬럼 최소화

**예시 — 나쁜 경우**

```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

orders가 10M rows이고, customers가 100K rows인데

orders가 build side가 되면 해시 테이블이 10M rows → 버퍼 초과 → 디스크 spill → 성능 최악

**개선**

```sql
SELECT /*+ JOIN_ORDER(customers, orders) */ ...
```

또는 WHERE 조건으로 build 쪽을 줄이는 전략

### "=" 조인 조건을 사용하는지 확인

Hash Join은 기본적으로 **equi-join**에서 가장 빠르게 동작한다.

해시 기반은 "=" 기반으로만 해시 테이블을 직접 조회할 수 있음

“>”, “<”, “>=”, “<=” 같은 range join은 해시 테이블 전체 탐색이 필요 → 느려짐

가능하면 "=" 조건으로 rewriting 하기

### 불필요한 인덱스 제거 (InnoDB 특성)

다소 counter-intuitive(직관적이지 않다)

**인덱스가 있으면 MySQL은 Hash Join 대신 NL 조인을 강제 선택하게 된다.**

**왜?**

- MySQL 옵티마이저는 **인덱스가 있으면 NL 조인이 더 빠르다라고 가정**
- 그래서 Hash Join을 사용하지 않음

**해결 방법**

- 정말 인덱스 없는 조인이 더 빠른 상황이라면
    
    힌트를 이용하거나 인덱스를 제거해 강제로 Hash Join을 유도
    

### Hash Join 강제 힌트(BNL 사용)

```sql
SELECT /*+ BNL(t2) */ ...
```

BNL은 원래 Block Nested Loop Join을 의미했지만

MySQL 8.0.20+에서는 사실상 **Hash Join 유도 힌트 역할**

**Hash Join 못 쓰는 이유 분석에도 유용**

특정 조인이 Hash Join이 걸리지 않을 때

- 인덱스가 있어서 안 쓰는지
- join_buffer_size가 부족한지
- equi-join 조건이 아닌지
- cardinality가 너무 큰지

등을 판단하는 데 힌트가 매우 유용

### join_buffer를 더 크게 쓰기 위한 server-side 튜닝

Hash Join 성능이 밀리면 아래도 함께 최적화

**join_buffer_size**

해시 테이블 저장 버퍼

**tmp_table_size**

Hash Join spill 시 임시 테이블로 사용됨

**max_heap_table_size**

메모리 기반 임시 테이블 크기

이 둘도 같이 늘리는 것이 좋음

```sql
SET GLOBAL tmp_table_size = 256M;
SET GLOBAL max_heap_table_size = 256M;
```

### 카디널리티가 낮은 컬럼 조인은 Hash Join 성능 급상승

Hash Join은 **카디널리티(Unique 값 개수)가 낮은 컬럼 조인**에서 효과가 큼

- gender (M/F)
- region code
- category id

이런 컬럼으로 조인하면 해시 테이블의 버킷 충돌이 적고 매우 빠르다.

반대로 카디널리티 높은 컬럼(예: UUID)은 해시 테이블이 비대해질 수 있음

### 페이지 크기, row_size, 데이터 타입 최적화

해시 테이블에 들어가는 실제 데이터 크기를 줄이면

- join_buffer_size 초과 가능성 ↓
- 캐싱 효율 ↑
- 전체 메모리 사용량 ↓
- hash bucket 충돌 ↓

**추천 전략**

- 조인 키는 가능한 INT, BIGINT 사용하기
- UUID(String 기반)는 가능하면 surrogate key로 변환
- TEXT/LONGTEXT 조인은 절대 금지
- SELECT * 대신 필요한 컬럼만 SELECT

### 해시 조인이 느릴 때 체크리스트

아래 항목을 하나씩 체크

**해시 조인이 느린 이유 Top 6**

| 원인 | 설명 |
| --- | --- |
| join_buffer_size 초과 | 해시 테이블이 메모리 대신 디스크로 spill |
| build side 선택이 잘못됨 | 큰 테이블을 build side로 설정 |
| 조인 조건이 "="이 아님 | full probe 필요 |
| 조인 키 타입이 무겁다 | VARCHAR, TEXT → 해시 충돌 증가 |
| select * 사용 | 불필요하게 큰 row가 해시 테이블에 들어감 |
| 인덱스가 있어 optimizer가 NL Join 선택 | 부적절한 실행 계획 |

## 서브쿼리 변환이 필요한 이유

DB 옵티마이저는 **쿼리 블록 단위**로 동작

즉, 어떤 서브쿼리든 하나의 **독립적인 쿼리 블록**으로 판단하고 최적화 수행

하지만 **각 블록을 따로 최적화했다고 해서 전체 쿼리가 최적화되는 건 아님**

서브쿼리는 메인쿼리에 종속되므로, 전체 관점에서 최적화하려면 **서브쿼리 계층을 풀어내는(unnest)** 작업이 필요

## 서브쿼리 종류와 특징

### 인라인 뷰 (FROM 서브쿼리)

- FROM 절에 오는 서브쿼리
- 일종의 **가상 테이블**처럼 작동
- 옵티마이저가 view merging을 못하면 서브쿼리 전체를 먼저 실행하는 비효율이 발생

### 중첩 서브쿼리 (WHERE 서브쿼리)

- 필터링 목적으로 WHERE / HAVING 절에 사용
- 메인쿼리 컬럼을 참조하면 → **상관 서브쿼리(Correlated Subquery)**
    
    → 메인쿼리 row마다 서브쿼리 1회 실행 → 매우 느릴 수 있음
    

### 스칼라 서브쿼리 (SELECT 서브쿼리)

- 하나의 값만 반환하는 서브쿼리
- 메인쿼리의 각 row마다 실행되지만, 내부적으로 **결과 캐싱**을 수행해 최적화 가능

## 서브쿼리 조인의 내부 동작

상관 서브쿼리는 내부적으로 **Nested Loop(Filter 방식)** 으로 수행

### 동작 방식

1. 메인쿼리 row 1개 읽음
2. 서브쿼리에 값을 전달
3. 서브쿼리를 실행해 존재 여부(exists) 확인
4. TRUE이면 즉시 멈추고 다음 메인 row로 이동
    
    (Exists 조건일 경우)
    

### 특징

- **NL 조인과 거의 동일**
- **부분 범위 처리 가능**
- 서브쿼리 결과를 **입력값 → 결과값** 형태로 **캐싱**
    
    → 동일 조건이면 서브쿼리 재실행하지 않음
    
- 단점
    
    **조인 순서 고정 (메인 = 드라이빙 테이블 고정)**
    
    → 최적화 선택 폭이 좁음
    

## Unnesting (서브쿼리 → 조인 형태로 변환)

서브쿼리를 메인 쿼리와 동일 레벨로 펼쳐서

**조인 형태로 재구성하는 것**

### Unnesting의 장점

- 더 다양한 조인 전략 사용 가능 (NL / Hash / Sort-Merge 등)
- 서브쿼리를 먼저 실행하여 필터링 가능
- 병렬 처리, 조인 reorder 가능
- 상관 서브쿼리의 **계층 종속성**을 제거하여 풀 스캔을 피할 수 있음

### Unnest 힌트 사용

```
UNNEST
NO_UNNEST  (반대로 unnest 방지)
```

### Unnest 시 사용하는 추가 최적화

- Sort Unique 연산
    
    (중복 조인 키 제거 → 메인쿼리 결과 폭발 방지)
    
- Pushdown 조건 사용 가능

## 서브쿼리 Pushdown (Push_Subq)

중첩 서브쿼리를 **더 먼저 실행**해서

메인쿼리로 전달되는 데이터 수를 줄이는 기법

**아래의 문제를 해결**

서브쿼리는 항상 메인쿼리의 드리븐 역할만 할 수 있다는 제약 극복

### Pushdown 조건

- 중첩 서브쿼리 형태여야 함
- 힌트 필요

```
NO_UNNEST
PUSH_SUBQ
```

- 이미 unnest된 서브쿼리에는 push_subq가 작동하지 않음

### 효과

- 조인하기 전 서브쿼리에서 먼저 filtering
    
    → 이후 단계 전체 처리 비용 크게 감소
    
    → 특히 multi-join 환경에서 매우 효과적
    

## 인라인 뷰 (Inline View) 최적화: MERGE vs Materialize

### 뷰 머징(Merge)

인라인 뷰를 메인쿼리와 합쳐서 **NL 조인 형태로 변경**

머징되면 메인쿼리 조건을 view 안에 pushdown할 수 있음

**필요한 데이터만 읽게 됨 → 성능 개선 폭 큼**

**뷰에 MERGE 힌트**

```
SELECT /*+ MERGE(v) */ ...
FROM (SELECT ... FROM ...) v
```

### 머징이 안 되는 경우

- 인라인 뷰에 GROUP BY / ORDER BY / DISTINCT 등이 존재
    
    → 부분 범위 처리 불가능
    
    → NL 조인은 비효율 → Hash Join이 더 적합함
    

### 해결

- HASH 조인 힌트로 유도
- 조인 조건 pushdown을 이용해 인라인뷰의 처리량을 줄임

## 스칼라 서브쿼리 최적화

스칼라 서브쿼리는 SELECT 절에서 자주 사용되며,

실제론 **NL 조인** + **캐싱**으로 동작

### 내부 동작 구조

1. 메인쿼리 row 1개에 대해 서브쿼리 1회 실행
2. **입력값 → 반환값** 형태로 캐싱 (PGA 내부)
3. 다음 row가 같은 값을 가진 경우 조인 생략
    
    → 캐시 재사용
    
    → 실행 횟수 급감
    

### 스칼라 서브쿼리 튜닝 포인트

- **입력값 종류(category 수)가 적을수록 캐싱 효율 ↑**
- 입력 값 종류가 많으면
    
    → 캐시 충돌 증가
    
    → 캐시 탐색 비용 증가
    
    → 오히려 성능이 나빠짐
    

### 이런 경우 스칼라 서브쿼리 사용 권장

- 메인쿼리 row가 많음
- 서브쿼리가 매우 빠름
- 입력 컬럼의 distinct 값이 적음
- 서브쿼리 결과를 여러 번 재사용 가능

### 이런 경우는 사용하면 안 됨

- 입력값 종류가 매우 많음
- Join 자체가 heavy함
- 캐시 메모리가 작은 환경

### 개선 방안

- 스칼라 서브쿼리 → 인라인 뷰로 변경
- 혹은 Unnesting하여 조인 형태로 재구성
- Oracle 12c 이상에서는 옵티마이저가 자동으로 unnesting 여부 판단