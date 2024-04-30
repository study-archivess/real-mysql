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

![5.1.png](hyunhwaoh-images%2F5.1.png)

1. 레코드 락
레코드 자체만을 잠근다.
InnoDB는 레코드 자체가 아니라 인덱스의 레코드를 잠근다.
이것이 큰 성능차이를 만들어낸다.

2. 갭락
레코드 자체가 아니라 레코드와 인접한 레코드 사이 간격을 잠그는것을 의미한다.
레코드 사이 간격에 새로운 레코드가 생기는 것을 제어하며, 넥스트 키 락의 일부로 사용된다.

3. 넥스트 키 락
레코드 락과 갭락을 합쳐놓은 형태의 잠금이다.
바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스서버에서 만들어 낸 결과와 동일한 결과를 만들어 내도록 보장하는것이 주목적이다.
버전이 8.0 되면서 ROW 포맷 바이너리 로그가 기본설정이 되었다.

4. 자동 증가 락
자동 증가 PK가 있는 테이블에 insert 시 증가하는 일련번호를 위해 사용된다.
AUTO_INCREMENT 락 이라고 하는 테이블 수준의 잠금을 사용한다.

### 인덱스와 잠금
InnoDB의 잠금과 인덱스는 중요한 부분이다.
InnoDB의 잠금은 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리된다.
변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드 모두에 락이 걸린다.

예를들어 employment 테이블에 first_name 컬럼에만 인덱스가 걸려있다고 하자.</br>
first_name 이 Georgi인 레코드가 100명이고 first_name 이 Georgi이면서 last_name 이 Klassen 인 사람은 1명이라 했을때,</br>
다음의 쿼리를 사용하여 업데이트를 한다면 몇개의 레코드에 락이 걸릴까?
```sql
UPDATE employment SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen';
```
쿼리 조건의 last_name 은 인덱스에 없기 때문에 first_name 이 Georgi 인 100개의 레코드가 모두 잠긴다.

만약 테이블에 인덱스가 하나도 없다면 테이블 풀 스캔하며 업데이트 작업을 하며 모든 레코드를 잠그게된다.
이러한 작동방식때문에 InnoDB에서 인덱스 설계가 중요하다.

### 레코드 수준의 잠금 확인 및 해제
테이블 잠금의 경우는 잠금의 범위가 커서 문제가 생겼을때 원인이 쉽게 발견될 수 있으나,
레코드 수준의 잠금은 사용되지 않으면 잘 발견되지 않는다.

쿼리를 실행해서 레코드 잠금과 잠금 대기에 대한 조회가 가능하다.

8.0 버전부터 performance_schema 의 data_locks 와 data_lock_waits 테이블로 확인할 수 있다.
```sql
SELECT * FROM performance_schema.data_locks;
```
data_locks 와 data_lock_waits 테이블, information_schema.innodb_trx 테이블을 사용해서 
잠금 대기중인 스레드와 대기 순서를 확인할 수 있고,
강제로 잠금을 해제하려면 스레드를 KILL 명령을 이용해 MySQL 서버의 프로세스를 강제로 종료할 수 있다.

## 4. MySQL 의 격리 수준 isolation level
트랜잭션 격리수준이란 여러 트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는것이다.
아래로 갈수록 각 트랜잭션 간의 데이터 격리 정도가 높아지며, 동시 처리 성능도 떨어진다.
REPEATABLE READ 과 SERIALIZABLE 격리 수준은 거의 사용되지 않는다.

|  | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ   |
|--|--------|--------------|----------------|
| READ UNCOMMITTED | 발생     | 발생     | 발생             |
| READ COMMITTED | 없음     | 발생     | 발생             |
| REPEATABLE READ | 없음     | 없음     | 발생<br/>(InnoDB는 없음) |
| SERIALIZABLE | 없음     | 없음      | 없음             |

### READ UNCOMMITTED
각 트랜잭션에서의 변경 내용이 커밋이나 롤백 여부에 상관없이 다른 트랜잭션에서 보인다.

![5.3.png](hyunhwaoh-images%2F5.3.png)

이처럼 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상을 더티 리드라고 한다.
READ UNCOMMITTED 격리수준은 더티 리드를 허용한다.

### READ COMMITTED
오라클 DBMS에서 기본으로 사용되는 격리수준이며 대부분의 서비스에서 많이 선택한다.

커밋이 완료된 데이터만 다른 트랜잭션에서 조회할 수 있기에 더티 리드가 발생하지 않는다.

![5.4.png](hyunhwaoh-images%2F5.4.png)

변경 발생 시 언두 로그 영역에 이전 레코드 정보가 백업된다. 

사용자 A가 데이터를 변경한 후 사용자 B가 해당 컬럼을 조회하면 언두 로그에 백업된 레코드를 조회하게 된다. 

하지만 사용자 B가 한 트랜잭션의 시작과 끝까지 같은 데이터 조회를 보장하지는 않는다.

이런 문제로 데이터의 정합성이 깨지고 어플리케이션에 찾기 힘든 버그를 유발할 가능성이 있다.

![5.5.png](hyunhwaoh-images%2F5.5.png)


### REPEATABLE READ
MySQL 의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리수준이다.
바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 한다.

REPEATABLE READ 는 MVCC(Multi Version Concurrency Control) 를 위해 언두 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있게 보장한다.

![5.6.png](hyunhwaoh-images%2F5.6.png)

REPEATABLE READ, READ COMMITTED 차이는 언두 영역에 백업된 레코드의 여러 버전 가운데 몇번째 이전 버전까지 찾아 들어가야 하는가다.
언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 번호가 포함되어 있다.

조회되는 레코드는 자신의 트랜젝션 번호보다 작은 트랜잭션 번호에서 변경한, 언두 영역의 데이터를 가져온다.

MVCC 를 보장하기 위해 실행중인 트랜잭션 가운데 가장 오래된 트랜잭션 번호보다 트랜잭션 번호가 앞선 언두 영역의 데이터는 삭제할 수 없다.

![5.7.png](hyunhwaoh-images%2F5.7.png)

SELECT FOR UPDATE 구문은 동시성 제어를 위해 배타적 잠금 기능을 수행하는 조회 쿼리다.</br>
그러나 이 쿼리는 언두 영역에 잠금기능을 수행할 수 없으므로 트랜잭션 번호와 상관없이 실제 데이터를 가져오게 되면서, 
초반 데이터가 보엿다 안보였다 하는 현상의 **PHANTOM READ** 정합성 오류가 발생한다.

### SERIALIZABLE
가장 단순하지만 가장 엄격한 수준의 격리이다. 동시 처리 성능이 떨어지고, 읽기 작업도 잠금을 획득해야한다.
한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서 절대 접근할 수 없다.</br>
그러나 InnoDB의 갭락과 넥스트키락으로 REPEATABLE READ 격리 수준에서도 PHANTOM READ 가 발생하지 않으므로 구지 사용하지 않는다.
