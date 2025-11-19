# Transaction

데이터베이스 작업의 안전성과 일관성 제공

MyISAM과 MEMORY 스토리지 엔진이 InnoDB보다 더 빠르다 여겨지지만, 트랜잭션을 지원하지 않아서

데이터 무결성과 관리의 복잡성에 대한 문제를 야기할 수 있다.

하나의 논리적 데이터베이스 작업에 쿼리가 하나든 아니든 상관없이 100% 적용(Commit)되거나 아무것도 적용되지 않아야 함(Rollback)을 보장한다.

### ACID

데이터베이스의 핵심은 신뢰성 있는 데이터 처리

신뢰성을 보장하기 위한 기본 원칙이 ACID

**원자성 (Atomicity)**

> All or Nothing — 전부 수행되거나, 전혀 수행되지 않거나
> 

트랜잭션은 **더 이상 쪼갤 수 없는 최소 단위의 작업**이다.

즉, 트랜잭션 내의 모든 SQL문이 **전부 성공해야만** 커밋되고, 그중 하나라도 실패하면 **전체를 롤백**해야 한다.

일부만 반영되는 부분 성공 상태는 존재해서는 안 된다.

```sql
BEGIN;
UPDATE account SET balance = balance - 1000 WHERE id = 'A';
UPDATE account SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```

- 위 두 쿼리 중 하나라도 실패하면, 전체 트랜잭션은 롤백된다.
- A의 잔액만 줄거나, B의 잔액만 늘어나는 상태는 허용되지 않는다.

**개발자가 해야 할 일**

- **어디까지를 하나의 트랜잭션으로 묶을지** 명확히 정의해야 한다.
- **언제 커밋하고, 언제 롤백할지**를 코드 레벨에서 설계해야 한다.
- 커밋(Commit)과 롤백(Rollback)은 DBMS가 처리하지만, **트랜잭션의 경계는 개발자가 책임진다.**

**일관성 (Consistency)**

> 트랜잭션 전후의 DB 상태는 항상 일관적(Consistent)이어야 한다
> 

트랜잭션 수행 전후로 데이터베이스는 항상 **유효한 상태**여야 한다.

즉, DB에 정의된 **제약 조건(Constraints)**, **트리거(Triggers)**, **규칙(Rules)** 등을 위반해서는 안 된다.

트랜잭션은 **하나의 일관된 상태 → 또 다른 일관된 상태**로만 변해야 한다.

은행 계좌의 잔액이 음수가 되면 안 된다고 하자.

```sql
UPDATE account SET balance = balance - 1000000 WHERE id = 'A';
```

이 쿼리의 결과로 balance가 음수가 된다면,

DB는 이를 **일관성 위반**으로 판단하고 트랜잭션을 롤백해야 한다.

**개발자가 해야 할 일**

- DBMS는 커밋 전에 제약 조건 위반 여부를 자동으로 확인한다.
- 하지만 애플리케이션 수준에서도 **논리적 일관성**을 체크해야 한다.
    
    (ex. 총합 불일치, 중복 데이터, 잘못된 상태 전이 등)
    

**고립성 (Isolation)**

> 여러 트랜잭션이 동시에 실행되더라도, 마치 혼자 실행되는 것처럼 보여야 한다
> 

트랜잭션은 동시에 실행될 수 있지만, 서로의 중간 상태를 볼 수 없어야 한다.

즉, 다른 트랜잭션이 작업 중인 데이터를 읽거나 변경해서는 안 된다.

이러한 고립성(Isolation)을 유지하기 위해 DBMS는 여러 수준의 **격리 수준(Isolation Level)**을 제공한다.

A 트랜잭션이 상품 재고를 수정하는 중인데,

B 트랜잭션이 그 중간 값을 읽는다면? → 데이터 불일치 발생!

이런 문제를 막기 위해 DB는 **락(Lock)** 과 **MVCC (Multi-Version Concurrency Control)** 같은 메커니즘을 사용한다.

**영속성 (Durability)**

> 커밋된 트랜잭션의 결과는 영원히 사라지지 않아야 한다
> 

트랜잭션이 커밋되면, 그 결과는 **비휘발성 저장장치(HDD, SSD)** 에 영구적으로 저장되어야 한다.

즉, 전원이 꺼지거나 서버가 다운되어도, 커밋된 데이터는 유지되어야 한다.

**영속성을 보장하는 기술**

- **Redo Log** : 커밋된 변경 내용을 재기록하여 복구 가능
- **Write-Ahead Logging (WAL)** : 실제 데이터보다 로그를 먼저 기록
- **Crash Recovery** : 비정상 종료 후 트랜잭션 상태를 복원

### 격리 수준

여러 트랜잭션이 동시에 실행될 때, 하나의 트랜잭션이 다른 트랜잭션의 중간 상태를 얼마나 볼 수 있느냐를 결정하는 단계

DBMS는 여러 트랜잭션이 동시에 실행될 때의 간섭 정도를 제어하기 위해 4가지 격리 수준을 제공

| Isolation Level | 허용되는 현상 | 특징 |
| --- | --- | --- |
| **READ UNCOMMITTED** | Dirty Read 허용 | 가장 빠르지만 가장 위험 |
| **READ COMMITTED** | Non-repeatable Read 허용 | 대부분의 DB 기본값 |
| **REPEATABLE READ** | Phantom Read 허용 | InnoDB 기본 설정 |
| **SERIALIZABLE** | 없음 | 가장 안전하지만 성능 저하 |

**READ UNCOMMITTED (커밋되지 않은 읽기)**

> 가장 낮은 수준의 격리 단계로, 다른 트랜잭션에서 **아직 커밋되지 않은 데이터**까지 읽을 수 있다.
> 

**예시**

- 트랜잭션 A: `UPDATE user SET name='박경' WHERE id=10;` (아직 커밋 안 함)
- 트랜잭션 B: `SELECT name FROM user WHERE id=10;`
    
    → **‘박경’** 이 조회됨(커밋 전 데이터를 읽음)
    

만약 이후 A 트랜잭션이 **ROLLBACK** 한다면, B 트랜잭션은 존재하지 않는 값을 읽은 셈이 된다.

- 이런 현상을 **Dirty Read (더티 리드)** 라고 한다.

**특징**

- 데이터 부정합 가능성이 매우 높음
- 실제 서비스에서는 거의 사용하지 않음
- Oracle에서는 이 수준을 **지원하지 않음**

**READ COMMITTED (커밋된 읽기)**

> 커밋된 데이터만 읽을 수 있도록 하는 격리 수준
> 
> 
> 대부분의 DBMS(Oracle, PostgreSQL 등)에서 기본값
> 

**예시**

- 트랜잭션 A: `UPDATE user SET name='박경' WHERE id=10;` (아직 커밋 안 함)
- 트랜잭션 B: `SELECT name FROM user WHERE id=10;`
    
    → **‘박기영’** 조회됨 (커밋 전 데이터는 읽지 않음)
    

즉, **Dirty Read는 발생하지 않는다.**

**Undo 영역의 역할**

그렇다면 커밋 전 값을 어떻게 읽을 수 있을까?

그 비밀은 **Undo 영역(Undo Log)** 에 있다.

Undo Log에는 트랜잭션이 변경하기 전의 **이전 값(Old Value)** 과 **PK 정보**가 저장된다.

트랜잭션이 커밋되지 않았을 때, 다른 트랜잭션이 데이터를 읽으면 

Undo 영역의 이전 값을 읽어서 **일관된 데이터 조회를 보장**한다.

**Undo Log의 구조**

- **Redo Log** : 커밋된 트랜잭션 복구용
- **Undo Log** : 롤백을 위한 이전 데이터 저장용
- **Undo Log Buffer → Undo Log File**
    
    메모리 → 디스크 순으로 저장되어, 성능과 안정성을 모두 확보
    

**Non-Repeatable Read 문제**

같은 SELECT 쿼리를 두 번 실행했을 때 결과가 달라지는 현상

```sql
-- 트랜잭션 B
SELECT name FROM user WHERE id=10;  -- ‘박기영’
-- 트랜잭션 A: UPDATE user SET name='박경' WHERE id=10; COMMIT;
SELECT name FROM user WHERE id=10;  -- ‘박경’
```

**REPEATABLE READ (반복 가능한 읽기)**

> 같은 SELECT 쿼리를 여러 번 실행해도 항상 같은 결과를 보장하는 격리 수준
> 
> 
> MySQL(InnoDB)의 기본 설정
> 

READ COMMITTED보다 한 단계 높은 수준으로, **Non-Repeatable Read 문제를 해결**한다.

**원리: MVCC (Multi-Version Concurrency Control)**

REPEATABLE READ는 **Undo Log** 와 **트랜잭션 ID(Transaction ID)** 를 이용해 데이터의 스냅샷(Snapshot)을 관리한다.

즉, 트랜잭션이 시작된 시점에 존재하던 데이터 버전만 읽는다.

```sql
-- 트랜잭션 A (TID=10)
SELECT * FROM user;  -- ‘박기영’ 조회

-- 트랜잭션 B (TID=13)
UPDATE user SET name='박경' WHERE id=10; COMMIT;

-- 트랜잭션 A
SELECT * FROM user;  -- 여전히 ‘박기영’ 조회
```

즉, 트랜잭션 10번은 자신보다 **작은 TID에서 커밋된 데이터만 읽는다.**

**오라클에서는 왜 REPEATABLE READ가 없을까?**

Oracle은 REPEATABLE READ 수준을 직접 지원하지 않는다.

하지만 동일 효과를 얻기 위해 **Exclusive Lock (배타적 잠금)** 을 사용한다.

**Exclusive Lock**

- 다른 트랜잭션이 **읽기·쓰기 모두 불가능**하게 막는다.
- `SELECT ... FOR UPDATE` 구문을 통해 설정 가능.

```sql
SELECT * FROM user WHERE id=10 FOR UPDATE;
```

해당 레코드는 Lock이 풀릴 때까지 다른 트랜잭션이 UPDATE / DELETE / INSERT 불가 상태로 대기하게 된다.

단, **읽기(SELECT)** 는 여전히 가능

InnoDB의 **MVCC 기술**을 통해 Undo 영역에서 이전 버전을 읽기 때문

**Phantom Read (유령 읽기)**

Repeatable Read에서도 발생할 수 있는 또 다른 문제.

하나의 트랜잭션 내에서 동일한 SELECT 쿼리를 여러 번 실행했을 때 결과의 **레코드 개수**가 달라지는 현상

```sql
-- 트랜잭션 A
SELECT * FROM user WHERE dept='EE';  -- 10개 행

-- 트랜잭션 B
INSERT INTO user VALUES(... dept='EE' ...); COMMIT;

-- 트랜잭션 A
SELECT * FROM user WHERE dept='EE';  -- 11개 행 (유령 등장!)
```

Exclusive Lock은 조회된 레코드에만 락을 걸기 때문에,

**새로운 INSERT에 대해서는 제어할 수 없다.**

**MySQL에서는 왜 Phantom Read가 발생하지 않을까?**

InnoDB에서는 `SELECT ... FOR UPDATE` 시 **Next-Key Lock**을 사용하기 때문이다.

**SERIALIZABLE (직렬화)**

> 가장 높은 수준의 격리 단계 — 모든 트랜잭션을 순차적으로 실행시킨다.
> 

모든 트랜잭션이 **직렬(Serial)** 로 실행되는 것처럼 처리되므로

Dirty Read, Non-Repeatable Read, Phantom Read 모두 발생하지 않는다.

**내부 동작**

- `SELECT` 실행 시 → **Shared Lock (공유 락)**
- `INSERT / UPDATE / DELETE` 실행 시 → **Exclusive Lock (배타적 락)**
    
    (MySQL은 Next-Key Lock으로 구현)
    

**Shared Lock**

- 여러 트랜잭션이 동시에 읽기는 가능하지만
    
    **쓰기 작업은 불가능**
    
- `SELECT ... FOR SHARE` 문법으로 사용

```sql
SELECT * FROM user WHERE id=10 FOR SHARE;
```

다른 트랜잭션이 Exclusive Lock을 걸려고 하면 대기하게 된다.

따라서 트랜잭션들이 서로 끼어들 수 없고 **완벽한 순차 처리**가 보장된다.

### MyISAM과 InnoDB 비교

```sql
INSERT INTO tab_myisam (fdpk) VALUES (3);
INSERT INTO tab_innodb (fdpk) VALUES (3);

INSERT INTO tab_myisam (fdpk) VALUES (1),(2),(3);
-- Error Code: 1062. Duplicate entry '3' for key 'tab_myisam.PRIMARY'

INSERT INTO tab_innodb (fdpk) VALUES (1),(2),(3);
-- Error Code: 1062. Duplicate entry '3' for key 'tab_innodb.PRIMARY'
```

두 쿼리문 모두 프라이머리 키 중복 오류로 쿼리가 실패한다.

- 하지만 MyISAM 테이블에는 오류가 발생했는데도 1과 2가 INSERT된 상태로 남아 있다.
    
    데이터의 무결성을 보장하지 못하는 증거다.
    
    InnoDB 테이블은 트랜잭션을 지원하기 때문에 쿼리문을 실행하기 전의 상태로 복구한다.
    

MyISAM처럼 Partial Update가 일어나면 데이터의 정합성을 맞추기 위해서 실패한 쿼리로 인해 남은 레코드를 다시 삭제하는 이중 작업이 필요하게 된다. 

이런 경우에 대해서 굉장히 비효율적!

- 쿼리가 실패했을 때 생긴 부분 업데이트를 어떻게 다시 복구할지에 대한 처리가 필요하기 때문
    
    

### Transaction은 꼭 필요한 최소 범위로만

많은 개발자들이 실수하는 부분 중 하나가 바로 **트랜잭션의 범위를 불필요하게 넓게 잡는 것**이다.

트랜잭션은 데이터 무결성을 보장하기 위한 필수 장치이지만, 잘못 사용하면 오히려 DB 부하를 증가시키고 서버 전체의 성능을 떨어뜨릴 수 있다.

### 왜 최소 범위로?

DBMS의 트랜잭션은 기본적으로 **데이터베이스 커넥션(Connection)** 위에서 동작한다.

- 트랜잭션이 유지되는 동안 해당 커넥션은 다른 요청이 사용할 수 없다.

만약 프로그램이 트랜잭션을 길게 유지한다면,

- **커넥션 풀의 여유 커넥션이 고갈되고**,
- 결국 다른 요청들이 **대기 상태**로 빠질 수 있다.
    
    이는 웹 서버뿐만 아니라 **DB 서버에도 큰 부하**를 주는 위험한 상황이다.
    

**예시 시나리오 : 주문 등록 처리 흐름**

1. 요청 수신
    
    → **DB 커넥션 생성 + 트랜잭션 시작**
    
2. 사용자 로그인 여부 확인
3. 장바구니 검증
4. 결제 서버와 통신
5. 주문 정보 DB 저장
6. 결제 정보 DB 저장
7. 주문 확인 메일 발송
8. 메일 발송 이력 DB 저장
    
    → **트랜잭션 커밋 + 커넥션 반납**
    

이 경우, **실제로 DB에 데이터를 저장하는 작업은 5~6번뿐**인데,

1~4, 7번처럼 DB와 직접 관련 없는 작업들까지 트랜잭션이 유지된다.

특히 4번(결제 API 통신)이나 7번(메일 발송)처럼

네트워크 지연이 발생할 수 있는 구간을 트랜잭션 안에 넣는 것은 매우 위험하다.

만약 외부 서버가 느리거나 오류가 난다면, DB의 트랜잭션까지 지연되어 전체 서비스가 막힐 수 있다.

1. 요청 수신
2. 로그인 여부 확인
3. 장바구니 및 결제 검증
4. 결제 서버와 통신 (외부 API 호출)
    
    → **DB 커넥션 생성 + 트랜잭션 시작**
    
5. 주문 정보 DB 저장
6. 결제 정보 DB 저장
    
    → **트랜잭션 커밋 + 커넥션 반납**
    
7. 주문 확인 메일 발송
    
    → **새 트랜잭션 시작**
    
8. 메일 발송 이력 DB 저장
    
    → **트랜잭션 커밋 + 커넥션 반납**
    

이렇게 하면

- 트랜잭션은 **DB 작업에만 집중**되고,
- 네트워크 통신이나 외부 서비스 호출은 **트랜잭션 밖에서 처리**된다.
    
    결과적으로 **DB 자원 점유 시간이 줄어들고**, **트랜잭션 충돌 위험과 서버 부하가 감소**한다.
    

# Lock

여러 세션이 동시에 데이터에 접근할 때 **데이터 정합성(Consistency)**를 유지하기 위한 핵심 메커니즘

잠금의 적용 범위와 종류에 따라서 성능에 미치는 영향이 크게 달라질 수 있다.

### Lock의 2가지 Level

1. **MySQL Engine Lv**
    - 스토리지 엔진(MyISAM, InnoDB 등)과는 독립적으로 동작.
    - MySQL 서버 전체나 테이블 구조 수준에 영향을 미침.
2. **Storage Engine Lv**
    - 엔진 내부에서 작동하며, InnoDB의 경우 레코드 단위 잠금(Record Lock)까지 세밀하게 제어 가능.
    - 다른 스토리지 엔진에는 영향을 미치지 않음.

### MySQL Engine에서 제공하는 대표 Lock 종류

| 종류 | 잠금 대상 | 주요 사용 목적 |
| --- | --- | --- |
| **글로벌 락 (Global Lock)** | 서버 전체 | 전체 DB 백업이나 유지보수 시 데이터 일관성 확보 |
| **테이블 락 (Table Lock)** | 특정 테이블 | 테이블 단위 데이터 변경 보호 |
| **네임드 락 (Named Lock)** | 임의의 문자열 | 사용자 정의 동기화 제어 |
| **메타데이터 락 (Metadata Lock)** | 테이블 구조 및 객체 | 스키마 변경 시 구조 안정성 보장 |

### Global Lock

> 명령어:
> 
> 
> `FLUSH TABLES WITH READ LOCK;`
> 
> `UNLOCK TABLES;`
> 

글로벌 락은 **MySQL 전체 서버를 잠그는 가장 강력한 잠금**이다.

한 세션에서 글로벌 락을 획득하면, 다른 세션에서는 SELECT 외 대부분의 쿼리(INSERT, UPDATE, DELETE, DDL)가 모두 대기 상태로 들어간다.

즉, **서버 전체를 읽기 전용(Read-Only)** 상태로 만든다.

**사용 예시**

- 여러 DB에 있는 MyISAM 테이블을 **mysqldump**로 백업할 때
    
    **일관성 있는 스냅샷**(특정 시점에서 데이터베이스 또는 스토리지 시스템의 데이터가 트랜잭션적으로 완전하고 유효한 상태로 캡처된 복사본)을 확보하기 위해 사용.
    

**주의사항**

- 락이 걸리면 MySQL 서버 전체에 영향을 주므로,
    
    **운영 중인 웹 서비스 환경에서는 사용하지 않는 것이 좋다.**
    
- 긴 SELECT 쿼리가 실행 중이면, 락을 획득할 때까지 대기해야 하므로
    
    **INSERT/UPDATE 작업이 모두 지연될 수 있다.**
    

**대안: 백업 락 (Backup Lock)**

MySQL 8.0부터는 더 가벼운 백업 전용 락이 도입됐다.

```sql
LOCK INSTANCE FOR BACKUP;
-- 백업 실행
UNLOCK INSTANCE;
```

이 락은 다음 변경 작업만 막고, 일반적인 DML(데이터 수정)은 허용한다.

- DDL (CREATE, ALTER, DROP)
- 사용자 계정 변경, 비밀번호 변경
- REPAIR / OPTIMIZE TABLE

즉, **데이터는 계속 수정 가능하지만**,

**스키마 변경으로 인한 백업 실패를 방지**하기 위해 필요한 락이다.

### Table Lock

> 명령어:
> 
> 
> `LOCK TABLES table_name [READ | WRITE];`
> 
> `UNLOCK TABLES;`
> 

테이블 락은 **특정 테이블 단위로 걸리는 잠금**이다.

한 세션이 테이블을 잠그면 다른 세션은 해당 테이블에 접근할 수 없다.

**명시적 테이블 락**

직접 `LOCK TABLES` 명령으로 획득하는 락으로,

**MyISAM, MEMORY, InnoDB 테이블 모두**에 적용할 수 있다.

하지만 **글로벌 락과 마찬가지로 온라인 서비스에는 부적합**하다.

잠금 시간 동안 해당 테이블에 대한 접근이 완전히 막히기 때문이다.

**묵시적 테이블 락**

쿼리 실행 시 MySQL이 자동으로 잠그는 형태다.

예를 들어 MyISAM 테이블에 `UPDATE`를 실행하면 MySQL이 알아서 테이블을 잠그고,

작업이 끝나면 자동으로 해제한다.

단, InnoDB의 경우 레코드 단위 잠금(Record Lock)을 사용하므로

이러한 묵시적 테이블 락은 DDL(스키마 변경) 시에만 영향이 있다.

### Named Lock

> 명령어 예시:
> 

```sql
SELECT GET_LOCK('task_sync', 10);   -- 잠금 획득 (최대 10초 대기)
SELECT IS_FREE_LOCK('task_sync');   -- 잠금 여부 확인
SELECT RELEASE_LOCK('task_sync');   -- 잠금 해제
SELECT RELEASE_ALL_LOCKS();         -- 모든 네임드 락 해제
```

네임드 락은 임의의 문자열(String)을 기준으로 잠금을 설정하는 기능이다.

즉, 테이블이나 레코드가 아니라, 이름(name) 자체에 대한 잠금이다.

**언제 사용하나?**

- 여러 웹 서버가 동시에 같은 데이터를 갱신해야 하는 **동기화 작업**
- 배치 프로그램처럼 대규모 UPDATE 작업을 수행할 때 **데드락 방지용 제어**

예를 들어 5대의 웹 서버가 하나의 설정값을 갱신해야 한다면,

`GET_LOCK('config_update', 5)` 로 간단히 동기화를 걸 수 있다.

**MySQL 8.0 이후 개선점**

- 네임드 락을 **중첩 획득 가능**
- `RELEASE_ALL_LOCKS()` 로 한 번에 해제 가능

### Metadata Lock

> 자동으로 발생하며, 직접 제어할 수 없음.
> 

메타데이터 락은 테이블이나 뷰의 **이름 또는 구조를 변경할 때 자동으로 획득**된다.

예를 들어 아래 명령은 RENAME 과정에서 두 테이블 모두에 락이 걸린다.

```sql
RENAME TABLE rank TO rank_backup, rank_new TO rank;
```

이 방식은 원본 테이블과 새 테이블이 **한 번에 교체**되므로,

실시간 트래픽 환경에서도 **“Table not found” 오류 없이** 안전하게 교체할 수 있다.

하지만 아래처럼 분리해서 실행하면

`rank` 테이블이 일시적으로 사라지는 구간이 생긴다 👇

```sql
RENAME TABLE rank TO rank_backup;
RENAME TABLE rank_new TO rank;
```

### InnoDB 스토리지 엔진 잠금의 장점

InnoDB는 MySQL 엔진 레벨의 잠금과 별개로, 스토리지 엔진 내부에서 자체적인 잠금 메커니즘

- **레코드 단위 잠금(Record-level Lock)으로** MyISAM의 테이블 락보다 훨씬 높은 동시성을 제공
- 잠금 정보가 **작은 메모리 공간**에 효율적으로 관리
- 다른 DBMS에서 발생할 수 있는 **락 에스컬레이션(Lock Escalation)**
    
    즉, 레코드 락이 페이지 락이나 테이블 락으로 격상되는 현상이 없음
    

### InnoDB 잠금의 종류

InnoDB는 단순히 레코드 자체뿐 아니라,**레코드 사이의 간격(Gap)** 도 잠글 수 있다.

이로 인해 총 네 가지 형태의 잠금이 존재한다.

| 종류 | 설명 | 특징 |
| --- | --- | --- |
| **레코드 락 (Record Lock)** | 특정 레코드 자체를 잠금 | 인덱스 기반 |
| **갭 락 (Gap Lock)** | 레코드 사이의 빈 간격을 잠금 | 새로운 레코드 삽입 방지 |
| **넥스트 키 락 (Next-Key Lock)** | 레코드 락 + 갭 락의 결합 | InnoDB 기본 잠금 형태 |
| **자동 증가 락 (AUTO_INCREMENT Lock)** | 자동 증가 칼럼 보호용 | 짧은 테이블 수준 락 |

![image.png](attachment:930e0592-7e53-4bc0-8076-0a7491812072:image.png)

### Record Lock

> 특정 레코드만 잠그는 잠금 방식
> 

InnoDB는 실제 테이블의 행(row)을 잠그는 것이 아니라, **인덱스의 레코드(record)** 를 잠근다.

따라서 인덱스가 하나도 없는 테이블이라도 InnoDB는 내부적으로 생성된 **클러스터 인덱스(Primary Key)** 를 기준으로 잠금을 건다.

- **Primary Key 또는 Unique Key** 기반의 변경 시에는
    
    갭(Gap)은 잠그지 않고 해당 레코드만 잠금
    
- **보조 인덱스(Secondary Index)** 기반의 변경 시에는
    
    갭 락(Gap Lock)이나 넥스트 키 락(Next-Key Lock)이 함께 적용됨
    

### Gap Lock

> 인덱스 레코드 사이의 “빈 공간”을 잠그는 락
> 

예를 들어 다음과 같은 테이블이 있다고 하자.

id

3

7

id 컬럼에 인덱스가 걸려 있다면,

`id = 4, 5, 6` 구간은 실제 레코드가 없는 **갭(Gap)** 구간이다.

이 구간에 갭 락이 걸리면, 다른 트랜잭션이

`id = 4~6` 사이에 **새로운 레코드 INSERT**를 할 수 없게 된다.

**즉, 갭 락은 새로운 데이터 삽입을 제어하는 역할**을 한다.

단독으로 사용되기보다는 **넥스트 키 락의 일부**로 자주 등장한다.

### Next-Key Lock

> 레코드 락 + 갭 락을 결합한 형태
> 
> 
> InnoDB의 기본 잠금 방식
> 

```sql
SELECT * FROM orders WHERE id > 99 FOR UPDATE;
```

이때 InnoDB 내부에서는 다음과 같은 일이 일어난다.

1. `id > 99` 조건을 만족하는 첫 번째 레코드(`id = 101`) 탐색
2. 직전 레코드(`id = 97`) ~ `id = 101` 사이 구간에 **갭 락** 설정
3. 이후 `id > 99`에 해당하는 모든 레코드 구간에도 **갭 락** 설정
4. `id > 99` 레코드들에 **레코드 락** 설정

즉, `id = 99 이상`의 **레코드와 그 사이 간격 모두 잠금**이 걸린다.

이러한 복합 형태의 잠금을 “넥스트 키 락”이라 부른다.

**넥스트 키 락의 목적**

**복제(Replication)** 환경에서 바이너리 로그(binlog)의 쿼리 결과를 **정확히 동일하게 유지**하기 위함.

하지만 넥스트 키 락은 **데드락(Deadlock)** 발생의 주요 원인 중 하나이기도 하다.

가능하다면 **바이너리 로그 포맷을 `ROW` 형태로 변경**하여 넥스트 키 락을 최소화하는 것이 좋다.

(MySQL 8.0은 기본적으로 ROW 포맷을 사용한다.)

### AUTO_INCREMENT Lock

> AUTO_INCREMENT 컬럼의 일관성을 유지하기 위한 테이블 단위 락
> 

MySQL에서는 자동 증가 컬럼을 위해 `AUTO_INCREMENT` 속성을 제공한다.

여러 세션이 동시에 INSERT를 수행하더라도 중복되지 않는 연속된 숫자가 저장되어야 한다.

이때 InnoDB는 내부적으로 **AUTO_INCREMENT 락**을 사용한다.

- **INSERT / REPLACE** 쿼리에서만 발생
- **UPDATE / DELETE** 에서는 걸리지 않음
- **트랜잭션과는 무관**하게, AUTO_INCREMENT 값을 할당하는 순간에만 잠금이 걸렸다가 즉시 해제됨

즉, 아주 짧은 시간만 잠기기 때문에 일반적으로 문제되지 않는다.

명시적으로 획득하거나 해제할 수도 없다.

**잠금 방식 변경**

MySQL 5.1 이상에서는 `innodb_autoinc_lock_mode` 시스템 변수를 통해

자동 증가 락의 동작 방식을 조정할 수 있다.

| 모드 | 설명 |
| --- | --- |
| **0 (전통 모드)** | 모든 INSERT가 AUTO_INCREMENT 락을 사용 |
| **1 (연속 모드, 기본값)** | 일반적인 INSERT는 락 최소화 |
| **2 (간헐적 모드)** | 대량 INSERT 시에도 락 최소 |