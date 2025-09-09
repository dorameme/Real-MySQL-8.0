# InnoDB 스토리지 엔진 vs MySQL 엔진의 잠금 수준 비교

## 핵심 차이 요약

| 구분 | InnoDB 스토리지 엔진 | MySQL 엔진 |
|------|---------------------|------------|
| 잠금 단위 | 행(레코드) 단위 | 테이블/전체 단위 |
| 잠금 범위 | 세밀한(fine-grained) | 광범위한(coarse-grained) |
| 동시성 | 높음 | 낮음 |
| 대상 | 데이터와 인덱스 | 메타데이터와 연결 |

## InnoDB 스토리지 엔진: 세밀한 잠금

### 행 수준 잠금 (Row-Level Locking)
```sql
-- InnoDB의 행 단위 잠금 예시
UPDATE employees SET salary = 5000 WHERE employee_id = 101;
-- employee_id가 101인 행만 잠금, 다른 행은 수정 가능
```

### 내부적 잠금 관리
- 인덱스 기반 잠금: 인덱스 레코드를 대상으로 잠금
- MVCC(Multi-Version Concurrency Control): 읽기/쓰기 충돌 최소화
- 자동 데드락 감지: 교착 상태 자동 해결

## MySQL 엔진: 광범위한 잠금

### 메타데이터 잠금 (Metadata Lock)
```sql
-- 테이블 구조 변경 시
ALTER TABLE employees ADD COLUMN bonus INT;
-- 전체 테이블에 대한 메타데이터 잠금 발생
-- 다른 세션의 DDL 작업 차단
```

### 글로벌 락 (Global Lock)
```sql
-- 전체 데이터베이스 잠금
FLUSH TABLES WITH READ LOCK;
-- 모든 테이블에 대한 읽기/쓰기 작업 차단
```

## 실제 동작 비교

### InnoDB: 동시성 우선
```sql
-- 세션 1: 특정 행 수정
UPDATE accounts SET balance = balance - 100 WHERE account_id = 123;

-- 세션 2: 다른 행 동시 수정 (가능)
UPDATE accounts SET balance = balance + 200 WHERE account_id = 456;
-- account_id 123과 456이 다른 행이므로 동시 실행 가능
```

### MySQL 엔진: 안정성 우선
```sql
-- 세션 1: 테이블 구조 변경
ALTER TABLE accounts ADD COLUMN last_updated TIMESTAMP;

-- 세션 2: 데이터 조회 (일시적 차단)
SELECT  FROM accounts WHERE account_id = 123;
-- 메타데이터 잠금으로 인해 ALTER 완료까지 대기
```

## 계층적 잠금 구조

### 1. MySQL 엔진 레벨
- 메타데이터 락: 테이블 구조 보호
- 글로벌 락: 전체 데이터베이스 보호
- 테이블 락: 개별 테이블 보호

### 2. InnoDB 스토리지 엔진 레벨
- 행 락: 개별 데이터 행 보호
- 갭 락: 범위 보호
- 넥스트 키 락: 행+갭 복합 보호

## 실무적 영향

### 동시성 처리 능력
```sql
-- InnoDB: 높은 동시성 (대규모 트래픽 가능)
-- 여러 사용자가 동시에 다른 행 수정 가능

-- MySQL 엔진 잠금: 동시성 제한
-- DDL 작업 시 일시적 서비스 중단 발생 가능
```

### 데이터베이스 설계 시 고려사항
```sql
-- 1. 트랜잭션 많이 사용 → InnoDB 선택
-- 2. 빈번한 DDL 작업 → 작업 시간 조정 필요
-- 3. 대용량 데이터 → InnoDB의 행 잠금이 유리
-- 4. 읽기 전용 데이터 → MyISAM 고려 (但 deprecated)
```

## 최적화 전략

### InnoDB 잠금 최적화
```sql
-- 1. 적절한 인덱스 설계
-- 2. 짧은 트랜잭션 사용
-- 3. 적절한 격리 수준 선택
-- 4. 잠금 타임아웃 설정
```

### MySQL 엔진 잠금 관리
```sql
-- 1. DDL 작업은 사용량 적은 시간에
-- 2. pt-online-schema-change 사용
-- 3. 메타데이터 잠금 모니터링
-- 4. 불필요한 글로벌 락 회피
```

## 결론
- InnoDB 스토리지 엔진: 세밀한 행 단위 잠금으로 높은 동시성 제공
- MySQL 엔진: 광범위한 테이블/메타데이터 잠금으로 구조적 안정성 제공

이러한 이중 잠금 체계로 MySQL은 데이터 무결성과 동시성 모두를 확보.      
현대적 애플리케이션에서는 InnoDB의 행 단위 잠금이 대부분의 작업을 처리하며,     
MySQL 엔진 잠금은 DDL 작업과 같은 특수 상황에서만 사용.
