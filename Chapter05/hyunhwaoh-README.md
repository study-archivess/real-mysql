05_트랜잭션과 잠금
===

## 1. 트랜잭션
트랜잭션을 지원하지 않는 MyISAM 과 트랜잭션을 지원하는 InnoDB 의 처리 방식의 차이를 살펴보자.

### MySQL 에서의 트랜잭션
트랜잭션이란? 논리적인 작업 셋 자체가 100% 적용 (commit) 되거나 아무것도 적용되지 않아야(rollback) 함을 보장해 주는 것.

```sql
CREATE TABLE tab_myisam (
    id INT NOT NULL,
    PRIMARY KEY (id)
) ENGINE=MYISAM;

CREATE TABLE tab_innodb (
    id INT NOT NULL,
    PRIMARY KEY (id)
) ENGINE=innodb;

insert into tab_myisam (id) values (3);
insert into tab_innodb (id) values (3);

insert into tab_myisam (id) values (1), (2), (3);
insert into tab_innodb (id) values (1), (2), (3);
```

각 테이블에 pk 3인 데이터를 미리 넣어두고, 
쿼리로 pk 1,2,3 을 insert 하면 어떻게 될까?
MyISAM 테이블에서는 오류가 발생해도 1,2 가 insert 되어 데이터가 남아있다.
InnoDB 테이블은 쿼리 중 일부라도 오류가 발생하면 전체를 원상태로 만들어 데이터가 남아있지 않다.

MyISAM 테이블에서 발생하는 현상을 부분 업데이트(Partial Update)라고 하며, 정합성 오류를 만들어낸다.
부분 업데이트 현상이 필요하면 데이서 삭제, 재처리의 까다로운 작업이 필요하다.

### 주의사항 (트랜잭션 범위)
프로그팸 코드에서 트랜잭션은 최소한으로 유지하는 것이 좋다.
자원은 점유하고 오래 유지하면 교착상태에 빠지게 된다.

특히 이메일 발송같은 네트워크 통신이 필요한 작업은 자원을 오래 점유할 수 있으므로,
필요한 부분만 트랜잭션을 적용하고, 롤백이 필요하면 실패 허용 + 재처리 작업을 하거나
분산 트랜잭션을 적용하는것이 좋다.

## 2. MySQL 엔진의 잠금
잠금은 크게 MySQL Engine 잠금, Storage Engine 잠금 두가지로 나눌 수 있다.

MySQL Engine 잠금은 모든 스토리지 엔진에 영향을 미친다.

### 글로벌 락
- 잠금의 범위가 가장 크다. 실행과 동시에 모든 테이블을 닫고 잠근다.
- FLUSH TABLES WITH READ LOCK 
- mysqldump 같은 백업 프로그램은 이 명령을 내부적으로 실행하고 백업할 때도 있으므로 사용하는 옵션을 잘 살펴보고 사용하자.
- 글로벌 락을 얻은 세션을 제외한 다른 세션이 DDL, DML 문장을 실행하는 경우, 글로벌 락이 해제될때까지 대기상태로 남는다.


**백업락**</br>
 백업 툴들의 안정적 실행을 위해 도입되었다.
 특정 세션에서 백업락을 획득하면 모든 세션에서 다음의 테이블의 스키마나 사용자 인증 관련 정보의 변경이 불가하다. 
(데이터베이스 밍 테이블 등 모든 객체 생성 및 변경, 삭제/ REPAIR TABLE 과 OPTIMIZE TABLE 명령 /사용자 관리 및 비밀번호 변경)
백업락은 일반적인 테이블의 데이터 변경은 허용된다.</br>
 MySQL 서버가 소스 서버와 레플리카 서버로 구성되는데 백업은 레플리카 서버에서 실행된다.
정상적으로 복제는 실행되나 백업 실패를 막기위해 DDL 명령이 실행되었을때 복제를 일시 중지하는 역할을 한다.


### 테이블 락
- 개별 테이블 단위로 설정되는 잠금이다. 특정 테이블의 락을 횔득할 수 있다.
- Lock TABLES table_name [ READ | WRITE] 
- 명시적으로 획득한 락은 UNLOCK TABLES 명령으로 잠금 반납한다. 명시적 락은 사용성이 거의 없음.
- 묵시적 락은 테이블에 데이터를 변경하는 쿼리를 실행하면 발생한다. 쿼리가 실행되는 동안 자동은로 획득됬다가 완료된 후 자동 해제된다.

### 네임드 락 Named Lock
- GET_LOCK() 함수를 이용해 임의의 문자열에 대해 exclusive lock 을 얻는다.
- 동시성 제어가 필요할 떄 분산 락을 이용해 제어할 수도 있다.
  (분산락을 이용한 동시성 제어 예시: https://sungsan.oopy.io/5f46d024-dfea-4d10-992b-40cef9275999)
- docs : https://dev.mysql.com/doc/refman/8.0/en/locking-functions.html

### 메타데이터 락 Metadata Lock
- MySQL 5.5 부터 생긴 개념이다.
- 데이터베이스 객체의 이름이나 구조를 변경하는 경우에 획득하는 잠금.
- 명시적으로 획득하고 해재할 수는 없다.
- 메타데이터 락은 개체에 대한 변경작업을 시작하거나 마무리 할 떄 발생한다.
  pending 상황을 해소하기 위해 메타데이터 락을 유발하는 세션을 kill 해야 한다.
- docs : https://dev.mysql.com/doc/refman/8.0/en/metadata-locking.html
- 메타데이터 락 트러블 슈팅 예시 https://leezzangmin.tistory.com/51

## 3. InnoDB 스토리지 엔진 잠금
Storage Engine 잠금은 내부 레코드 기반 잠금 방식을 사용한다.
MyISAM 방식보다 뛰어난 동시성 처리를 제공한다.

### InnoDB 스토리지 엔진의 잠금
스토리지 엔진 내부에 레코드 기반 잠금 방식을 가지고 있으며, 뛰어난 동시성 처리를 제공한다.
최근 버전에는 트랜잭션, 잠금, 대기 중인 목록도 조회 가능하며, 장시간 잠금된 클라이언트를 종료시킬수도 있다.


### 인덱스와 잠금


### 레코드 수준의 잠금 확인 및 해제


## 4. MySQL 의 격리 수준

### READ UNCOMMITTED


### READ COMMITTED


### REPEATABLE READ


### SERIALIZABLE

