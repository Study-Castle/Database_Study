## **소트 연산 이해**

정렬(Sort)은 DB에서 **가장 비용이 큰 연산 중 하나**다.

![image.png](attachment:468c3f15-614f-4abc-9b44-783d899570f7:image.png)

**이유**

- **정렬은 기본적으로 PGA Memory(Local Memory)의 Sort Area를 사용**
- 메모리가 부족하면 → **Temporary 테이블스페이스(디스크)까지 사용**
    
    → 디스크 Sort 발생
    
    → 성능 *폭발적으로 저하*
    

**결론**

> 소트는 가능하면 발생하지 않도록,
> 
> 
> 발생하더라도 메모리 안에서 끝나도록 해야 한다.
> 

## **소트 오퍼레이션 종류 정리**

실행 계획에 표시되는 주요 Sort 연산들

### **Sort Aggregate**

**전체 집계를 수행할 때 발생**

(MIN, MAX, SUM, AVG 등)

```sql
SELECT MAX(sal) FROM emp;
```

- 실제로 정렬을 하는 것이 아니라
    
    Sort Area에서 값을 하나씩 비교하면서 집계를 수행함
    
- 큰 비용은 아님

### **Sort Order By**

**ORDER BY가 있을 때 발생하는 정렬**

```sql
SELECT * FROM emp ORDER BY sal DESC;
```

- 가장 기본적인 Sort
- 인덱스로 해결 가능하면 이 오퍼레이션은 사라진다.

### **Sort Group By**

GROUP BY가 있을 때 발생

```sql
SELECT deptno, MAX(sal)
FROM emp
GROUP BY deptno
ORDER BY deptno;
```

- 그룹핑을 위해 Sort 필요
- ORDER BY까지 있으면 Sort 비용 증가
- ORDER BY가 없으면 **Hash Group By** 사용 가능 → 정렬 생략

**ORDER BY가 없으면 Sort Group By는 발생하지 않을 수 있다.**

→ Hash Group By로 최적화됨

**Hash Group By 동작**

- Hash Table 생성 (메모리 기반)
- 레코드 하나씩 읽으며 deptno를 해시 키로 사용
- 같은 deptno는 같은 bucket에 모임
- bucket마다 집계 (SUM, COUNT 등)
- 최종 결과 반환

### **Sort Unique**

중복 제거 시 발생하는 소트

**IN 서브쿼리 중복 제거**

```sql
SELECT * FROM dept
WHERE deptno IN (SELECT deptno FROM emp);
```

**UNION**

```sql
SELECT job FROM emp WHERE deptno=10
UNION
SELECT job FROM emp WHERE deptno=20;
```

- UNIQUE 의 본질 = 중복 제거 → 정렬 필요
- PK / Unique 인덱스가 보장하는 경우에는 발생하지 않음 (이미 정렬 의미 있음)
- Distinct 연산자에서도 사용

**중복 제거 시 정렬이 필요한 이유**

데이터가 멀리 떨어져 있는 경우 모든 값을 기억해야 중복인지 판단할 수 있음

→ 메모리 한번 스캔으로는 안됨.

→ 메모리 폭발

정렬하면?

→ 직전 값과 같은지 비교만 하면 중복 판단 가능

→ 정렬하면 중복 판단이 O(1) 공간으로 가능

→ 정렬이 없으면 O(N) 메모리 필요

### **Sort Join (Sort Merge Join)**

Sort Merge Join에서 발생하는 정렬

각 테이블을 조인키 기준으로 정렬한 뒤 Merge

```
TABLE A SORT (JOIN)
TABLE B SORT (JOIN)
MERGE JOIN
```

- MySQL은 Sort Merge Join을 사용하지 않음
    
    (Oracle, PostgreSQL 등에서 주로 사용)
    

### **Window Sort**

윈도우 함수(행과 행 간의 관계를 쉽게 정의 및 분석하는 함수) 실행 시 발생

```sql
SELECT mgr, ename, sal,
       SUM(sal) OVER (PARTITION BY mgr) mgrsum
FROM emp;
```

- PARTITION BY + ORDER BY가 있으면 소트 발생
- PARTITION 크기가 크면 비용 폭증

## **소트 발생하지 않도록 SQL 작성 (튜닝 핵심)**

### UNION 대신 UNION ALL 사용

**비효율**

```sql
SELECT ...
UNION
SELECT ...
```

- UNION 은 *중복 제거* → Sort 발생

**최적화**

```sql
SELECT ...
UNION ALL
SELECT ...
```

**핵심**

- 데이터가 중복되지 않는다는 확신이 있으면 반드시 UNION ALL 사용
- 소트 오퍼레이션을 완전히 제거 가능

### DISTINCT 대신 EXISTS로 변경

**비효율적 DISTINCT**

```sql
SELECT DISTINCT a.*
FROM A a, B b
WHERE a.x = :x
AND b.y = a.y
AND b.z BETWEEN :dt1 and :dt2;
```

→ B 를 모두 읽고 → DISTINCT로 중복 제거 → Sort Unique 발생

**최적화 EXISTS 활용**

```sql
SELECT a.*
FROM A a
WHERE a.x = :x
AND EXISTS (
    SELECT 1 FROM B b
    WHERE b.y = a.y
    AND b.z BETWEEN :dt1 AND :dt2
);
```

중복 제거 필요 없음

EXISTS 는 존재 여부만 판단 → 매우 효율적

소트 제거 가능

### Hash Join 대신 NL Join 유도

Hash Join = 내부적으로 소트 & 버퍼 사용

NL Join = 인덱스 기반 랜덤 I/O

- 대량 데이터 조인에서는 Hash Join이 유리
- **인덱스가 매우 잘 구성되어 있고, 소트 비용이 크면 NL Join이 유리**

## **인덱스를 이용한 소트 생략**

정렬 없이 ORDER BY / GROUP BY / MIN / MAX를 처리할 수 있다면

**인덱스를 이용한 최적화**가 가능하다는 뜻이다.

### Top-N 쿼리 최적화 (Top-N Stopkey)

```sql
SELECT *
FROM emp
ORDER BY sal DESC
FETCH FIRST 10 ROWS ONLY;
```

**인덱스가 있다면**

정렬 없이 인덱스 정순/역순 스캔으로 10개만 읽고 중단

→ **STOPKEY 알고리즘**

→ 실행 계획: COUNT(STOPKEY)

### MIN / MAX 최적화 (First Row Stopkey)

```sql
SELECT MIN(sal) FROM emp;
```

인덱스(sal asc) 가 있다면

- 첫 번째 leaf node만 읽으면 끝
- Sort Aggregate 발생하지 않음

조건절 컬럼 + MIN/MAX 대상 컬럼 모두 인덱스에 포함되어 있어야 함

### GROUP BY 인덱스 활용 (Sort Group By 생략)

```sql
SELECT region, AVG(age)
FROM customer
GROUP BY region;
```

region 컬럼이 선두 컬럼 인덱스면

GROUP BY 수행 시 이미 정렬되어있음

Sort Group By → **Sort Group By (NOSORT)**로 변경됨

정렬 사라짐

## **Sort Area 사용량 줄이는 SQL 작성**

소트가 불가피하다면

> Sort Area에 최대한 적은 데이터를 넣게 만들어서 Temporary Table 사용을 줄여라
> 

### SELECT 절 항목을 줄여라

**비효율**

```sql
SELECT *
FROM emp
ORDER BY age;
```

Sort Area 에 전체 컬럼이 들어가므로 메모리 사용량 ↑

**효율적**

```sql
SELECT name
FROM emp
ORDER BY age;
```

필요한 컬럼만 Sort → Sort Area 사용량 ↓

메모리 안에서 끝날 확률 ↑

### TOP-N 쿼리는 Sort Area 사용량을 크게 줄인다

TOP-N 쿼리는 소트가 필요하더라도

- Sort Area에 전체 데이터를 넣지 않음
- 상위 N개만 유지하는 방식으로 Sort 수행
    
    → 메모리 사용량 최소화
    

> Top-N은 인덱스를 못 타더라도 **메모리 친화적**
>