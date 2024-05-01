# Chapter 5 트랜잭션과 잠금

## 잠금 / 트랜잭션 / 트랜잭션의 격리 수준의 차이
- 잠금 : 동시성 제어를 위한 기능
- 트랜잭션 : 데이터 정합성을 보장하기 위한 기능
- 트랜잭션의 격리 수준 : 하나 또는 여러 트랜잭션 간의 작업 내용을 공유하고 차단할 것인지 결정하는 레벨


## 5.1 트랜잭션
- InnoDB 스토리지 엔진은 트랜잭션을 지원하는 스토리지 엔진 (MyISAM,MEMORY 스토리지 엔진은 미지원)
- 꼭 여러 쿼리가 조합되야만 의미가 있는것은 아님
- 논리적인 작업 Set이 100% 적용 되거나 아예 적용되지 않음을 보장해줌

### 5.1.2 트랜잭션 주의사항
- 트랜잭션은 최소한의 코드에만 적용하는 것이 좋다
- 트랜잭션은 빠르게 처리되어야 한다
  - 커넥션은 개수가 제한적이기 때문에 소유 시간이 길어지면 성능에 영향을 준다.
- 메일 전송, FTP 파일 전송은 반드시 트랜잭션 밖에서 처리해야 한다.
  - 메일서버,FTP 서버와 통신할 수 없는 경우 장애가 DBMS까지 번지게 된다.


## 5.2 잠금

- 잠금의 종류
1) 스토리지 엔진 레벨의 잠금
   - 스토리지 엔진 간 상호 영향 주지 않음
2) MySQL 엔질 레벨의 잠금
   - 모든 스토리지 엔진에 영향을 줌


### 5.2.1. 글로벌 락
- 명령어 : FLUSH TABLES WITH READ LOCK
- 범위 : MySQL 서버 전체
  - 데이터베이스가 다르더라도 동일하게 영향을 미침
- 사용 목적 : MyISAM,MEMORY 테이블에 대해 mysqldump로 일관된 백업을 받아야 할 때 주로 사용
- 큰 영향을 미치기 때문에 가급적 사용하지 않는 것이 좋음
- 백업락 : MySQL 8.0 부터 Xtrabackup, Enterprise Backup 같은 백업 툴의 안정적인 실행을 위해 도입

#### 5.2.1.1 백업 락
- 일반적인 테이블의 데이터 변경은 허용 (DML)
- DDL 명령 실행 시 데이터 복제를 일시 중지


### 5.2.2 테이블 락
- 명령어 : LOCK TABLES table_name [READ | WRITE]
- 범위 : 테이블 단위
- 명시적으로 사용 가능
- 특별한 상황이 아니면 사용할 필요가 없다, 글로벌 락과 마찬가지로 온라인 작업에 상당한 영향을 미치기 때문
- 묵시적익 테이블 락
  - MyISAM,MEMORY 테이블의 데이터를 변경하는 쿼리 실행시 발생
  - InnoDB 테이블은 레코드 단위 락을 제공해 묵시적인 테이블락 사용 X
  - InnoDB 엔진에서 DML 쿼리의 경우 대부분 무시, DDL 쿼리의 경우 테이블 락이 설정 됨

### 5.2.3 네임드 락
- 기능 : 사용자가 지정한 문자열에 대해 잠금을 설정
- 용도 : 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용
- 락 설정 명령어 : SELECT GET_LOCK('lock_name',timeout) -> 이미 잠금 중인 경우 timeout만큼 대기하고 잠금 해제
- 락 확인 명령어 : SELECT IS_FREE_LOCK('lock_name') -> 락이 설정되어 있는지 확인
- 락 해제 명령어 : SELECT RELEASE_LOCK('lock_name') / SELECT REALEASE_ALL_LOCKS()

### 5.2.4 메타데이터 락
- 기능 : 데이터베이스 객체(테이블, 뷰 등 )의 이름이나 구조를 변경하는 경우에 획득하는 잠금
- 명시적으로 사용 불가능
- DDL 쿼리 실행시 자동으로 획득
- DDL 쿼리 실행 중에는 DML 쿼리가 대기
  - MySQL은 한 쓰레드에서 실행되기 때문에 느림
  - DDL 실행하기 보다 테이블을 여러 쿼리를 통해 여러 쓰레드에서 복제한 후 RENAME하는 것이 빠를 수 있음


## 5.3 InnoDB 스토리지 엔진 잠금

- InnoDB 스토리지 엔진은 트랜잭션을 지원하는 스토리지 엔진
- InnoDB 스토리지 엔진은 레코드 단위 락을 제공
- 트랜잭션 잠금, 잠금 대기 중인 트랜잭션 모니터링 기능 제공
  - infroamtion_schema.INNODB_TRX, INNODB_LOCKS, INNODB_LOCK_WAITS 테이블을 통해 확인 가능

### 5.3.1 InnoDB 스토리지 엔진의 잠금 종류
#### 5.3.1.1 레코드 락
- 레코드 단위로 잠금을 설정
- 실제 레코드를 잠그는 것이 아니라 레코드를 조회하기 위한 인덱스를 잠근다.
- 보조 인덱스를 활용한 잠금은 대부분 갭락, 넥스트 키 락을 사용한다.
- PK, 유니크 인덱스를 활용한 잠금은 레코드 락을 사용한다.

#### GAP 락
- 레코드가 아니라 레코드와 레코드의 인접한 간격을 잠그는 락
- 갭락 자체보다는 넥스트 키 락의 일부로 사용됨

#### 넥스트 키 락
- 레코드락 + 갭락
- 넥스트 키 락, 갭 락 으로 인한 데드락이 자주 발생하게 됨

#### 자동 증가 락 (AUTO_INCREMENT LOCK)
- AUTO_INCREMENT 컬럼에 대한 락
- INSERT,REPLACE 쿼리 같이 새로운 레코드 생성 쿼리에서만 사용
- 명시적 사용 X
- 아주 짧은 AUTO_INCREMENT 값을 조회하는 동안에만 락이 설정되어 있음
- mode에 따라 락을 사용하거나 락보다 가벼운 래치(뮤텍스)를 사용한다.
- 한 번 증가하면 줄지 않는다. (잠금을 최소화하기 위함)

### 5.3.2 인덱스와 잠금

- InnoDB 스토리지 엔진은 레코드 단위 락을 제공
- 보조 인덱스를 사용해 조회하는 경우 레코드가 아닌 인덱스에 잠금을 걺
ex)
- 상황1 : SELECT COUNT(*) FORM employees WHERE FIRST_NAME = 'SANGHOO' => 253
- 상황2 : SELECT COUNT(*) FORM employees WHERE FIRST_NAME = 'SANGHOO' AND LAST_NAME = 'PARK' => 1
- 상황3 : UPDATE employees SET SALARY = 100000 WHERE FIRST_NAME = 'SANGHOO' AND LAST_NAME = 'PARK'
- 상황3에서 업데이트를 위한 잠금을 얻는다.
- 이 때 한 개의 레코드에만 락이 걸리는 것이 아니라 FIRST_NAME을 조회하기 위한 **인덱스에** 락이 걸리게 된다.
- FIRST_NAME = 'SANGHOO'인 253개의 레코드에 전체 락이 걸림

### 5.3.3 레코드 수준의 잠금 확인 및 해제
- 레코드 수준의 잠금은 범위가 작기 때문에 잠금을 해제 하지 않은 경우 잘 발견되지 않는다.
- SHOW PROCESSLIST 명령어를 통해 pending된 trx 확인 가능
- KILL 명령어를 통해 강제로 해제 가능
- performance_schema.data_locks, data_lock_waits 테이블을 통해 확인 가능
ex)
- 17 trx가 커밋되지 않은 상황
- 18 trx,19 trx가 17 trx를 기다리고 있음
- performance_schema.data_lock_waits 확인 시 18은 17을 19는 17,18을 대기하고 있음

## 5.4 MySQL 격리 수준

### 5.4.0 Base 지식
#### 격리 수준
- 격리 수준 : 여러 트랜잭션이 동시 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지 결정하는 것
- READ UNCOMMITTED -> READ COMMITTED -> REPEATABLE READ -> SERIALIZABLE
- 격리 수준이 높아질수록 동시처리 성능 은 떨어지지만 데이터 정합성은 높아진다.
- SELIALIZABLE은 가장 높은 격리 수준이지만 가장 느리다. (나머지는 크게 성능 개선,차이가 발생하지 않음)

#### Dirty Read
- 어떤 트랜잭션에서 다른 트랜잭션의 변경되지 않은 데이터를 읽는 현상

#### Non-Repeatable Read
- REPEATABLE READ : 하나의 트랜잭션 내에서 동일한 SELECT 쿼리를 실행했을 때 항상 같은 결과를 가져와야한다
- REAPEATABLE READ 정합성을 보장하지 않는 현상

#### Phantom Read
- 하나의 트랜잭션 동안 동일한 쿼리를 여러 번 실행했을 때 결과로 나타나는 행의 집합이 변경되는 현상
- 발생원인 : 다른 트랜잭션이 쿼리가 실행되는 도중에 해당 쿼리의 결과에 영향을 줄 수 있는 새로운 행을 삽입하거나 삭제할 때 발생
- 언두 로그에는 잠금을 걸 수 없기 때문에 발생


### 5.4.1 READ UNCOMMITTED
- 다른 트랜잭션이 커밋하지 않은 데이터를 읽을 수 있음
- Dirty Read 발생 가능
- 최소한 READ COMMITTED 이상의 격리 수준을 사용할 것을 권장

### 5.4.2 READ COMMITTED
- 다른 트랜잭션이 커밋한 데이터만 읽을 수 있음
- Dirty Read 발생하지 않음
- 커밋되지 않은 TRX의 경우 언두 영역에 백업된 변경되기 전 데이터를 읽어옴
- Non-Repeatable Read 부정합이 발생

### 5.4.3 REPEATABLE READ

- InnoDB 스토리지 엔진의 기본 격리 수준
- 같은 트랜잭션 내에서는 항상 같은 조회 결과를 보장
- READ COMMITTED, REPEATABLE READ 모두 언두 로그를 이용해 MVCC를 구현
  - 둘의 차이 : 언두 영역에 백업된 레코드 중 몇 번째 이전 버전까지 찾아 들어가느냐
  - 언두 로그에는 변경을 발생시킨 트랜잭션 번호가 포함됨
  - REPEATABLE READ : 자신의 트랜잭션 번호보다 작은 트랜잭션 번호에서 변경된 것만 언두로그에서 읽음
- Phantom Read 부정합 발생 가능
  - MySQL InnoDB의 경우 갭락, 넥스트 키 락 덕분에 Phantom Read가 발생하지 않음


### 5.4.4 SERIALIZABLE
- 가장 높은 격리 수준
- 읽기 작업도 공유 잠금 권한을 획득해야함
- MySQL InnoDB에서는 REPEATABLE READ에서도 Phantom Read가 발생하지 않아 굳이 사용할 이유가 없음






