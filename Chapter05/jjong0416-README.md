# 5.0 서론

이번 장에 알아볼 것은 아래와 같습니다. 하나씩 살펴봅시다.

- 잠금 vs 트랜잭션
- +) 트랜잭션 ACID
- 잠금(Lock) 종료
- 트랜잭션(Transaction)
- 트랜잭션 격리 수준(Isolation Level)

## 5.0.1 트랜잭션 vs 잠금

### 트랜잭션

> 데이터의 정합성을 보장하기 위한 기능으로, **작업의 완전성을 보장**하여 일부만 적용되는 현상(Partial update)이 발생하지 않게 만들어주는 기능
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/c9e02af3-6fc1-44f3-ba5e-78609e061a85/Untitled.png)

### 잠금

> 동시성을 제어하기 위한 기능으로, **여러 커넥션에서 동시에 동일한 자원을 요청할 경우 한 시점에 하나의 커넥션만 변경할 수 있게 해주는 역할**
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/767c399d-a87d-4aed-9d2a-2a1600793366/Untitled.png)

## 5.0.2 트랜잭션의 ACID

ACID는 데이터베이스에서 트랜잭션의 무결성을 보장하기 위해 꼭 필요한 4가지 요소를 의미한다.

1. 원자성 ( Atomicity )
2. 일관성 ( Consistency )
3. 독립성 ( Isolation )
4. 지속성 ( Durability )

### Atomicity 원자성

> 작업의 일부만 적용되는(Partial update) 현상이 발생 하지 않고, 작업의 완전성을 보장해 주는 것
>

트랜잭션 내 여러 작업들은 하나의 작업처럼 동작한다. 하나라도 실패하면 전체가 다 실패한 결과를, 전체가 다 성공해야 트랜잭션이 성공한 결과를 내보내야 한다.

### Consistency 일관성

> 트랜잭션이 성공적으로 완료되면 일관적인 DB상태를 유지해야한다.
>

데이터가 제약조건들을 다 만족하면서 모순되지 않은 상태를 유지하고있는것을 말한다.

예를들어 내 계좌가 마이너스 잔액을 허용하지 않는다고 가정하면, 여기서 제약 조건은 "마이너스 잔액을 허용하지 않음"이 된다. 즉, 잔액은 0원 미만이 될수 없다. 만약 계좌 잔액이 200만원인데 결제 트랜잭션에서 400만원을 결제 한다면, 그 결제 트랜잭션은 제약조건을 위배하고 모순된 상황을 만들어 내므로 실패해야 할것이다.

### Isolation 고립성

> 고립성은 동시 트랜잭션을 처리하는데 사용
>

동일한 테이블의 데이터를 수정하는 두 개의 트랜잭션이 동시에 발생하면 문제가 발생할 수 있기 때문에 하나의 트랜잭션이 수행되는 동안 다른 트랜잭션을 끼어들지 못하는 특성을 말한다.

### Durability 지속성

> 트랜잭션이 정상적으로 완료되었다면 이 트랜잭션으로 인한 결과는 영구적으로 보존 되야 하는 특성
>

### ACID 속성의 예시

쿠팡과 같은 예약구매 사이트에서 새로 출시한 아이폰을 사는 과정을 생각해보면

- 사용자가 결제와 함께 전체 결제 프로세스를 모두 완료한 경우에만 주문이 성공한 것으로 간주되야 한다. 만약 프로세스 중 하나라도 실패하게 된다면 전체 결제 과정이 실패한 것으로 생각하고 처음으로 되돌아 가야한다. (**A**tomicity, 원자성)
- 사용자가 아이폰의 예약 주문을 성공적으로 완료하면 웹사이트에서는 잔여 수량이 1개 줄어야 한다. 만약 안 줄고 그대로면 일관성이 없어진것으로 볼 수 있다. (**C**onsistency, 일관성)
- 구매 가능 수량이 1개 남은 시점에서 동시에 2명의 유저가 동시에 결제를 한다고해도 1명의 유저만 결제를 성공해야 하고 나머지 1명은 결제를 실패해야 한다.(**I**solation, 고립성)
- 만약 사용자가 아이폰을 구매한 후, 며칠 있다가 갑자기 구매 취소가 되거나, 주문 취소가 되면 안된다.(**D**urability, 지속성)

## 5.1 트랜잭션

- MySQL에서 MyISAM이나 MEMORY 스토리지가 더 빠르다고 생각하고 InnoDB 스토리지 엔진은 사용하기 복잡하고 번거롭다고 여전히 생각하는 사람이 많다.
- 하지만 사실은 MyISAM이나 MEMORY 같은 엔진은 트랜잭션을 지원하지 않는다.

### 5.1.1 MySQL 트랜잭션

**트랜잭션 관점**에서 InnoDB테이블과 MyISAM 테이블의 차이를 알아보자.

```sql
CREATE TABLE tab_myisam (fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=MyISAM;
INSERT INTO tab_myisam (fdpk) VALUES (3);

CREATE TABLE tab_innodb (fdpk INT NOT NULL, PRIMARY KEY (fdpk) ) ENGINE=INNODB;
INSERT INTO tab_innodb (fdpk) VALUES (3);

SELECT * FROM tab_innodb, tab_myisam;
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/bdcf5066-6d0e-430b-9e9c-65d51445aa6d/Untitled.png)

그 후, AUTO-COMMIT 모드에서 다음 쿼리를 실행해본다.

```sql
# AUTO-COMMIT 활성화
SET autocommit=ON;

INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);
INSERT INTO tab_innodb (fdpk) VALUES (1), (2), (3);
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/76b38ee9-0554-4aab-a9e2-271341a00188/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/931d9793-84c9-40eb-9cef-111e6a0f1452/Untitled.png)

```sql
SELECT * FROM tab_myisam;
+------+
| fdpk |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)

---

SELECT * FROM tab_innodb;
+------+
| fdpk |
+------+
|    3 |
+------+
1 row in set (0.00 sec)
```

- 일단 두 INSERT 구문은 모두 PK 키 중복 오류로 쿼리가 실패
- MyISAM
    - 1, 2가 저장된 이후 중복 키 발생 → 이미 1, 2가 저장됨 → 부분 업데이트(Partial Update)
        - 테이블 데이터의 정합성을 맞추는데 상당히 어려움
- InnoDB
    - 쿼리 일부라도 오류가 발생 → Insert 구문이 실행되기 전 상태로 복구

즉, 부분 업데이트가 발생되면 실패한 쿼리로 인해 남은 레코드를 다시 삭제하는 재처리 작업이 필요할 수 있다.

### 5.1.2 주의사항

> 프로그램 코드에서 트랜잭션의 범위를 최소화하는 것이 좋다!
>

데이터베이스의 **커넥션 갯수가 제한적**이기 때문이다.
각 프로그램이 커넥션을 소유하는 시간이 길어질수록 사용가능한 여유 커넥션 의 개수가 줄어들어, 어느순간부터는 각 프로그램에서 커넥션을 가져가기 위해 기다리는 상황이 발생할 수 있기 때문이다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/5e6cdbbb-be09-4a36-bab9-d2a49f71caf3/Untitled.png)

그렇다면 최적의 트랜잭션 설계는 어떻게 해야할까?

- **데이터베이스 커넥션을 가지고 있는 범위와 트랜잭션이 활성화 되어 있는 범위를 최소화**
- **네트워크 작업이 있을 경우, 반드시 트랜잭션에서 배제**
    - DBMS 서버가 높은 부하 상태에 빠지거나 위험한 상태에 빠지는 경우가 빈번히 발생하기 때문

## 5.2 MySQL 엔진의 잠금

MySQL에서 사용되는 잠금은 크게 아래와 같이 나눌 수 있다.

- MySQL 엔진 레벨
- 스토리지 엔진 레벨

MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨 잠금은 스토리지 엔진 간 상호 영향을 미치지 않는다.

### 5.2.1 글로벌 락(Global Lock)

- `FLUSH TABLES WITH READ LOCK` 명령을 통해 락을 획득할 수 있다.
- MySQL 제공하는 락 중 가장 범위가 크다.
- 한 세션에서 글로벌 락을 획득하면 다른 세션에서 `SELECT` 를 제외한 대부분의 DDL, DML 문장은 글로벌 락이 해제될 때 까지 `대기 상태`로 남는다.
- 글로벌 락은 MySQL 서버 전체 테이블에 영향을 주는 만큼 웹 서비스에서 가급적이면 사용하지 않아야 한다.
    - 여러 데이터베이스에 존재하는 InnoDB, MyISAM, MEMORY 테이블에 `mysqldump` 로 일관된 백업을 받아야 할 때 사용
- MySQL 8.0부터는 조금 더 가벼운 글로벌 락의 필요성(InnoDB 기본 채택)이 생겨 **[백업 락](https://steady-coding.tistory.com/545)**이 도입되었다.

```sql
FLUSH TABLES WITH READ LOCK;
UNLOCK TABLES;
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/477e9dbb-9de7-438c-9cbc-983dda8f9518/Untitled.png)

<aside>
💡 커넥션 vs 세션
커넥션은 **클라이언트 프로세스와 데이터베이스 인스턴스 간의 물리적 경로**을 말한다. 쉽게 말해 클라이언트와 인스턴스 간의 네트워트 커넥션을 의미

세션은 인스턴스 안에 있는 논리적인 실체로 현재 유저의 로그인 상태를 나타낸다.
ex) `mysql -u root -p`  를 통해 로그인 한다면 세션을 얻는 것

</aside>

### 5.2.2 테이블 락

개별 테이블 단위로 설정되는 잠금으로, 명시적 또는 묵시적으로 특정 테이블의 락을 획득할 수 있다.

**명시적 테이블 락**

특별한 상황이 아니면 애플리케이션에서 사용할 필요가 거의 없다. 명시적으로 테이블을 잠그는 작업은 글로벌 락과 동일하게 온라인 작업에 상당한 영향을 미치기 때문이다.

```sql
lock table table_name [read | write];
# ---
lock table tab_innodb read;
INSERT INTO tab_innodb (fdpk) VALUES (5);
UNLOCK TABLES;
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/a3859178-55d6-4581-bfc2-5734f8f93057/Untitled.png)

**묵시적 테이블 락**

> InnoDB의 경우, 레코드 기반의 잠금을 제공하고 있기 때문에 단순 데이터 변경 쿼리(DML)로 인해 테이블 자체에 락을 획득하지는 않고, 스키마를 변경하는 쿼리(DDL)의 경우에만 테이블 락이 설정된다
>

MyISAM 이나 MEMORY에서는 테이블에 데이터를 변경하는 쿼리(DML)를 실행하면 자동으로 테이블에 락을 획득한다. 쿼리 완료 후 테이블 락은 자동으로 반환된다.

### 5.2.3 네임드 락

테이블이나 레코드가 아닌 **사용자가 지정한 문자열에 대해 잠금을 설정** 할 수 있다.

```sql
// 네임드락 획득 : 2초간 mylock이라는 락을 생성
// 성공 일 경우 1을 반환, 아닐 경우 0 이나 NULL 반환
SELECT GET_LOCK('mylock', 2); 

// 네임드락 상태 확인
SELECT IS_FREE_LOCK('mylock');

// 네임드락 반환
SELECT RELEASE_LOCK('mylock');
```

네임드 락은 자주 사용되지는 않지만,

- 데이터베이스 서버 1대에 5대의 웹 서버가 접속해 서비스하는 상황에서 웹 서버가 특정 정보를 동기화 해야 하는 경우. **즉, 여러 클라이언트가 상호 동기화를 처리해야 할 때**
- 많은 레코드에 대해 복잡한 요건으로 레코드를 변경할 때
    - 마치 배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 쿼리
- 위와 같이 네임드 락을 사용하면  데드락과 같은 상황이 발생하는 것을 막을 수 있다.

MySQL 8.0부터는 다음과 같이 네임드 락을 중첩해서 사용할 수 있게 되었고, 현재 세션에서 획득한 락을 한번에 해재할 수 도 있다.

```sql
SELECT GET_LOCK('user_define_lock1', 10);
# -> 여기에서 user_define_lock1에 대한 작업들 수행

SELECT GET_LOCK('user_define_lock2', 10);

# -> 여기에서는 user_define_lock2에 대한 작업들 수행

SELECT RELEASE_LOCK('user_define_lock2'); # user_define_lock2 반납
SELECT RELEASE_LOCK('user_define_lock1'); # user_define_lock1 반납

# or

SELECT RELEASE_ALL_LOCKS(); # 전체 네임드락 반납
```

## +) JPA에서는 어떻게 사용할까?

### A) 일반적으로 DBMS의 기본 Isolation Level을 따라간다.

- Mysql은 REPEATABLE READ 가 기본 설정이다.
- Oracle은 READ_COMMIT 이 기본 설정이다.
- SQL Server는 READ_COMMIT 이 기본 설정이다.
- Postgres는 READ_COMMIT 이 기본 설정이다.

### B) JPA에서는 네임드 락을 어떻게 사용할 수 있을까?

```java
// 락 획득
@Query(value = "select get_lock(:key, 3000)", nativeQuery = true)
void getLock(String key);

// 락 해제
@Query(value = "select release_lock(:key, key)", nativeQuery = true)
void releaseLock(String key);
```

- 이 때, 중요한 것은 **해당 Lock은 스레드가 종료된다고 해서 자동으로 락이 반납되지 않는다**.
    - 어떤 상황일까? → Lock의 timeout 시간보다 스레드가 더 빨리 종료되었을 때

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/bd8f7a13-b2f8-41fd-aa33-e60cb9e55841/Untitled.png)

### C) 그 외 관련있는 함수들

```java
RELEASEALLLOCKS() : 현재 세션에서 유지되고 있는 모든 잠금을 해제하고 해제한 잠금 갯수를 반환
ISFREELOCK(str) : 입력한 이름(str)에 해당하는 잠금이 획득 가능한지 확인한다.
ISUSEDLOCK(str) : 입력한 이름(str)의 잠금이 사용중인지 확인한다.
```

- 관련 레퍼런스
  https://velog.io/@hyojhand/named-lock-distributed-lock-with-redis
  [https://jjongdev.notion.site/Database-Level-49db8279d417442f8a894fc36fdf6c39?pvs=4](https://www.notion.so/Database-Level-49db8279d417442f8a894fc36fdf6c39?pvs=21)

### 5.2.4 메타데이터 락

> DB 객체(대표적으로 테이블이나 뷰, 리네임 등)의 **이름이나 구조를 변경하는 경우에 자동으로 획득하는 잠금**
>

### RENAME TABLE 명령의 경우

> 원본 이름과 변경될 이름 모두 한꺼번에 잠금을 설정
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/7759a0b8-5f34-4034-9f79-8d85d01f5f28/Untitled.png)

- 2개로 나눠서 실행하면 아주 짧은 시간이지만 테이블이 존재하지 않는 순간이 생겨 오류를 발생

```sql
RENAME TABLE ignite.table_new TO ignite.table_old2;
RENAME TABLE ignite.table_old1 TO ignite.table_new;
```

```java
Caused by: java.sql.SQLSyntaxErrorException: (conn=7732136) 
Table 'ignite.table_new' doesn't exist
```

```sql
RENAME TABLE ignite.table_new to ignite.table_old, ignite.table_old to ignite.table_new;
```

### 로그 테이블 구조를 변경해야 하는 경우

> **테이블 락**과 **메타데이터 락**을 함께 사용하는 케이스
>

로그 테이블과 같이 UPDATE, DELETE가 없이 INSERT만 있는 테이블의 구조를 변경하는 경우 다음과 같은 방법들을 생각할 수 있다.

A) DDL로 구조 변경하기

- DDL은 단일 스레드로 동작하기 때문에 상당히 많은 시간이 소요될 수 있다.
- 시간이 너무 오래 걸리는 경우 언두 로그도 함께 증가하게 되고, 이에 따라 MySQL 서버 성능이 저하

B) 새로운 구조의 임시 테이블을 만들어서 복사하는 방법

1. 새로운 구조의 임시 테이블을 생성하고 최근 데이터까지 PK 범위 별로 나누어 멀티 쓰레드로 빠르게 복사
2. 나머지 데이터의 경우 테이블 락을 걸고 복수 작업 수행, 복사 작업 수행 후 임시 테이블을 실제 운용 테이블로 이름을 바꾼다
3. 이 때 작업2에서 테이블락과 메타데이터 락이 함께 사용된다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/997fb5cf-6970-47ea-9253-a7fbbe7df750/Untitled.png)

새로운 구조의 테이블 생성

```sql
# KEY_BLOCK_SIZE=4 의 경우, 테이블 압축을 사용하겠다는 것인데 6장에서 자세하게 다룬다.
CREATE TABLE access_log_new (
	id BIGINT NOT NULL AUTO_INCREMENT,
	client_ip INT UNSIGNED,
	access_dttm TIMESTAMP,
	...
	PRIMARY KEY(id)
) KEY_BLOCK_SIZE=4;
```

### 작업 1. 오래된 로그 새 테이블로 복사하기

PK 기준으로 10000개씩 짤라서 멀티 쓰레드로 동시에 복사 작업 수행

```sql
mysql_thread1> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=0 AND id<10000;
mysql_thread2> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=10000 AND id<20000;
mysql_thread3> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=20000 AND id<30000;
mysql_thread4> INSERT INTO access_log_new SELECT * FROM access_log WHERE id>=30000 AND id<40000;
```

### 작업 2. 남은 데이터 복사하기

복사하는 도중에 새로운 로그가 INSERT 된다면 문제가 될 수 있다. 이를 막기 위해 테이블 잠금을 설정하고 작업을 수행한다. 테이블 잠금으로 인해 복사하는 동안 새로운 로그가 INSERT 될 수 없게 된다.

최대한 작업1에서 최근 데이터까지 미리 복사해 두어야 **잠금 시간을 최소화해서 서비스에 미치는 영향을 줄일 수 있다( 즉, 다운 타임을 최소로 줄일 수 있다)**.

```sql
# auto-commit을 꺼둔다.
SET autocommit=0;

# 작업 대상 테이블 2개에 대해 테이블 쓰기 락을 획득
LOCK TABLES access_log WRITE, access_log_new WRITE;

# 남은 데이터 복사
SELECT MAX(id) as @MAX_ID FROM access_log_new;
INSERT INTO access_log_new SELECT * FROM access_log WHERE pk>@MAX_ID;
COMMIT;

# 리네임
RENAME TABLE access_log TO access_log_old, access_log_new TO access_log;
UNLOCK TABLES;

# 기존의 테이블을 지울건진 비지니스 상황보고 판단(백업 테이블로 사용할 수 있잖아?)
# DROP TABLE access_log_old;
```

## 5.3 InnoDB 스토리지 엔진 잠금

### 5.3.1 InnoDB 스토리지 엔진의 잠금

InnoDB 스토리지 엔진은 MySQL에서 제공하는 잠금과는 별개로, **스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재**하고 있다. 이 덕분에 **뛰어난 동시성 처리를 제공**한다.

InnoDB 스토리지 엔진에서는 잠금 정보가 상당히 적은 공간으로 관리되는 레코드 기반의 잠금 기능을 제공하기 때문에 레코드락이 페이지락, 테이블 락으로 레벨업 되는 **락 에스컬레이션**이 없다.

<aside>
💡 **락 에스컬레이터** 이란?
레코드 락이 페이지 락으로, 또는 테이블 락으로 레벨업 되는 경우처럼 많은 수의 작은 잠금을 더 적은 수의 큰 잠금으로 올리는 프로세스를 뜻한다. 락 에스컬레이션이 발생하면 동시성 경합 가능성이 올라가므로 동시성 처리 성능이 떨어진다.

</aside>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/56d43201-0fbd-4592-b7d0-9dc35b9de718/Untitled.png)

### 5.3.1 레코드 잠금

> 레코드 자체만을 잠그는 것
>
- 다른 상용 DBMS는 레코드 락과 동일한 역할
- **중요한 것은 InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다는 것**
    - 인덱스가 하나도 없는 테이블은 내부적으로 자동 생성된 클러스터 인덱스(PK)를 활용해 잠금을 설정
- 레코드 자체를 잠그는 것 / 인덱스를 이용해 잠그는 것에는 상당히 크고 중요한 차이가 난다
- 프라이머리 키 또는 유니크 인덱스에 의한 변경 작업 → 레코드 자체에만 락을 건다.
    - 보조 인덱스를 이용한 변경 작업 → 넥스트 키 락, 갭 락 이용

### 5.3.2 갭 락

> 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것
>
- 갭 락의 역할은 레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT)되는 것을 제어
- 넥스트 키 락의 일부로 자주 사용된다.

### 너무 간결해서 + 시켜보자.

```sql
CREATE TABLE tb_gaplock (
id INT NOT NULL,
name VARCHAR(50) DEFAULT NULL,
PRIMARY KEY (id)
);

INSERT INTO tb_gaplock(id, `name`)
VALUES (1, 'Matt'), (3, 'Esther'), (6, 'Peter');
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/aa453a90-3bc9-4a9a-86bc-abefd9a9cc33/Untitled.png)

- 3건의 레코드(Matt, Esther, Peter)가 있고, 빈 공간의 레코드에는 언제든지 Insert가 될 수 있다.
- MySQL에서는 이런 상태의 테이블에서 레코드가 없는 빈 공간에 락을 걸 수 있다.
- PK 뿐만 아니라 Secondary Index에도 동일하게 사용할 수 있다.
- 갭 락을 사용하게 되면, 조회 쿼리를 두 번 실행했을 때 **다른 트랜잭션에서 수정이 발생하더라도 같은 결과가 리턴되도록 보장**할 수 있다.
    - 즉, Phantom Read를 방지하는 효과를 가진다.

### +) Where 절을 통해 데이터를 조회한 결과가 하나일 때, Record vs Gap

- 기본적으로 컬럼에 인덱스가 걸려 있다면, record lock을 사용한다.
- 인덱스가 없거나, unique 하지 않는 index라면 gap lock을 사용해야 한다.

인덱스가 있다면 쿼리 결과를 조회하기 위해 스캔한 인덱스 범위에 대해 gap lock이 적용되고, 인덱스가 없다면 테이블 전체를 스캔해야 하기 때문에 모든 레코드에 대해서 락이 걸린다.

### 5.3.3 넥스트 키 락(Next key lock)

> 레코드 락(Record Lock)과 갭 락(Gap Lock)을 합쳐 놓은 형태의 잠금
>

### 5.3.4 자동 증가 락

> AUTO_INCREMENT 컬럼이 사용된 테이블에서 사용하는 테이블 수준의 잠금
>
- 자동 증가가 사용된 테이블에 동시에 여러 레코드가 Insert 된 경우, 중복되지 않고 저장된 순서대로 증가하는 일련 번호 값을 가지도록 해주는 테이블 수준의 잠금
- `INSERT`나 `REPLACE` 쿼리 문장에만 사용되고, 트랜잭션과 관계없이 자동 증가 값을 가져오는 순간 **아주 짧은 시간**동안 락이 걸렸다가 즉시 해제된다.
    - `UPDATE`나 `DELETE`등 쿼리에는 걸리지 않는다.
    - 두 개의 INSERT 쿼리가 일어나면, 하나의 쿼리는 자동 증가 락을 기다려야 한다.
- 자동 증가 락은 하나만 존재하기 때문에 INSERT가 일어나는 경우에는 하나의 쿼리는 해당 잠금을 기다린다.

```sql
CREATE TABLE table1 (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    test INT AUTO_INCREMENT
);

# 13:51:07	
# Error Code: 1075. 
# Incorrect table definition; 
# there can be only one auto column and it must be defined as a key	0.0025 sec
```

- 자동 증가 락은 명시적으로 해제하거나 획득할 수 없다.
- MySQL 5.1부터는 innodb_autoinc_lock_mode 라는 시스템 변수를 이용해 자동 락의 작동방식을 변경할 수 있다.

```sql
innodb_autoinc_lock_mode = 0
# MySQL 5.0과 동일한 잠금 방식으로 모든 INSERT 문장은 자동 증가 락을 사용한다.

innodb_autoinc_lock_mode = 1
# MySQL 5.x 기본 값

innodb_autoinc_lock_mode = 2
# MySQL 8.0 기본값
# 자동 증가 락을 절대 사용하지 않고 무조건 Mutex를 사용한다. auto_increment 값의 
# 유니크성과 단조 증가성을 보장한다.
# Bulk Insert가 실행될 때 다른 트랜잭션에서도 INSERT 수행할 수 있기 때문에 동시 처리 성능이 높아진다.
# 하지만 단건 삽입의 경우 auto_increment 값 사이의 갭이 존재하지 않지만 
# 여러 개가 삽입될 경우 갭이 존재할 수 있다.
```

### 앗 내가 마주쳤던 문제가 있었던 것 같은데?

분명 나는 `INSERT`를 실패했는데 왜 auto_increment값이 올라갔던걸까?

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/bee3ae61-2f4d-4774-8fac-7eaa7788c468/Untitled.png)

바로 MySQL 8.0의 기본값이 2이기 때문이다. 자세히 알고 싶다면 아래 블로그를 통해 학습해보자.

> REFERENCE : https://stuffdrawers.tistory.com/11
>

### 5.3.5 인덱스와 잠금

> InnoDB의 잠금은 **레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식**으로 처리한다고 했다.
>

즉 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다는 것이다. 이와 같은 특징때문에 MySQL에서는 인덱스 설계가 굉장히 중요하다.

예시를 보자.

```sql
SELECT COUNT(*) FROM employess WHERE first_name='Jaecy';
+--------+
|   253  |
+--------+

SELECT COUNT(*) FROM employess WHERE first_name='Jaecy' AND last_name='Chan';
+--------+
|    1   |
+--------+

-- // 입사 일자를 오늘로 변경하는 쿼리를 실행해보자.
UPDATE employess SET hire_date=NOW()
WHERE first_name='Jaecy' AND last_name='Chan';
```

- 위 UPDATE 문장을 실행하면 인덱스를 이용할 수 있는 조건은 first_name='Jaecy'이며, last_name 칼럼은 인덱스에 없기 때문에 first_name='Jaecy'인 레코드 253건의 레코드가 모두 잠긴다.
- 위 예제에서는 몇 건 안 되는 레코드만 잠그지만 **UPDATE 문장을 위해 적절히 인덱스가 준비돼 있지 않다면 각 클라이언트 간의 동시성이 상당히 떨어져서 한 세션에서 UPDATE 작업을 하는 중에는 다른 클라이언트는 그 테이블을 업데이트하지 못하고 기다려야 하는 상황이 발생**

아래 이미지는 UPDATE 문장이 어떻게 변경 대상 레코드를 검색하고, 실제 변경이 수행되는지를 보여준다.

!https://blog.kakaocdn.net/dn/bitAdh/btrXlh4h1U3/d1zphiH87S3EmphaSGWoS0/img.png

만약 위 테이블에서 인덱스가 하나도 없다면 어떻게 될까? 이러한 경우에는 **테이블을 풀 스캔하면서 UPDATE 작업을 하는데, 이 과정에서 테이블의 모든 레코드를 잠그게 된다.** 이것이 MySQL의 방식이며, MySQL의 InnoDB에서 인덱스 설계가 중요한 이유이다.

## 5.4 MySQL 격리 수준

**트랜잭션의 격리 수준이란** 여러 **트랜잭션이 동시에 처리될 때 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정**하는 것이다.

격리 수준은 크게 4가지로 나뉜다.

- `READ-UNCOMMITTED`
- `READ-COMMITTED`
- `REAPEATABLE-READ`
- `SERIALIZABLE`

4개의 격리 수준에서 순서대로 뒤로 갈수록 각 트랜잭션 간의 **데이터 고립 정도가 높아지며**, **동시 처리 성능도 떨어**진다. 일반적인 온라인 서비스 용도의 데이터베이스는 READ-COMMITTED나 REAPEATABLE-READ 중 하나를 사용한다. 이 중 MySQL의 기본 격리 수준은 REAPEATABLE-READ이다.

책에서는 `READ-COMMITTED`,`REAPEATABLE-READ` 성능의 개선이나 저하는 **크게** 발생하지 않는다고 한다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/abd969ff-d50c-4421-baa2-7687c0c6958b/Untitled.png)

격리 수준에 항상 언급되는 세 가지 부정합의 문제점이 있다. 이 세 가지 부정합의 문제는 격리 수준의 레벨에 따라 발생할 수도, 발생하지 않을 수 있다.

### 5.4.1 READ UNCOMMITTED

> 각 트랜잭션에서 변경 내용이 COMMIT이나 ROLLBACK 여부와 상관없이 다른 트랜잭션에서 보이는 것을 말한다.
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/a43084b7-d348-4b29-87d9-71e435b6fe17/Untitled.png)

사용자 A가 `INSERT`를 수행한 후 `COMMIT` 하지 않은 상태에서 사용자 B가 해당 컬럼을 조회했을 경우, B는 정상적으로 해당 컬럼을 읽어올 수 있다.

### 어떤 문제가 발생할 수 있을까? : Dirty read

해당 격리 레벨은 dirty read, non-repeatable read, phantom read 모두 발생 가능하다.

해당 격리수준에서는 트랜잭션에서 처리 작업이 완료되지 않았음에도 다른 트랜잭션에서 볼 수 있다는 특징 때문에 데이터가 나타났다가 사라졌다가 하는 현상이 발생된다.

결론적으로 A가 INSERT 문을 롤백해도 사용자 B는 정상적으로 조회가 동작되는 **정합성에서 많은 문제가 발생**

### +) 어떻게 동작하는걸까?

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/0813ba9d-61a0-4610-a108-52ddd6efe1fb/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/768c317c-4187-41ac-9f85-342a0e09eae7/Untitled.png)

```sql
# 첫 번째 사진처럼 Insert 발생 시
INSERT INTO User (id, name, city) VALUES (1, 'Jay', 'Busan');
# 그 후, UPDATE 발생 시, undo log에 데이터 보관
UPDATE User SET city='Seoul' WHERE name = 'Jay'
```

### 5.4.2 READ COMMITTED

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/a7ae4b82-aa1a-4056-9a26-c3664c05cfe0/Untitled.png)

> `COMMIT`이 완료된 데이터만 다른 트랜잭션에서 조회
>

오라클 DBMS에서 사용하는 기본 격리수준이며, Dirty-read는 해결된 격리 수준이다. 마찬가지로 앞서 공부했던 언두 로그가 등장한다.

- 새로운 값(업데이트 값)은 테이블에 즉시 기록되고, 이전 값은 언두 영역에 백업이 된다.
- 커밋 수행 전에 SELECT → Undo Log 영역에 백업된 레코드의 값을 가져온다.

사용자 A가 데이터를 변경한 후 사용자 B가 해당 컬럼을 조회하면 언두 로그에 백업된 레코드를 조회하게 된다.

결론적으로 READ-COMMITTED 격리 수준에서는 어떤 트랜잭션에서 변경한 내용이 커밋되기 전까지는 다른 트랜잭션에서 그러한 변경 내역을 조회할 수 없다.

### 어떤 문제가 있을까? : 부정합의 문제(NON-REPETABLE READ)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/d1f8cf5d-e453-418c-b270-342410545899/Untitled.png)

> 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때 항상 같은 결과를 가져오지 못한다.
>

### 5.4.3 REPEATABLE READ

> REFERENCE : https://cl8d.tistory.com/109
>

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/a44306bb-39a8-4850-ad63-7f4d71755e91/Untitled.png)

> **트랜잭션 ID를 기준**으로 자신 이후에 발생한 트랜잭션에서의 변경사항을 읽지 않는 레벨
****MVCC를 위해 언두 영역에 백업된 데이터를 이용해 **동일 트랜잭션 내에서는 동일한 결과를 보장**
>
- 해당 격리 수준는 InnoDB의 기본으로 사용하는 격리수준이다
- 해당 격리 수준부터는 `NON-REPEATABLE READ` 현상이 발생하지 않는다.
- 트랜잭션 내부에서 실행되는 `SELECT`와 외부에서 실행되는 `SELECT`의 차이가 없는 `READ-COMMITTED` 격리 수준과는 다르게 REPEATABLE-READ 격리 수준은 기본적으로 `SELECT` 쿼리 문장도 트랜잭션 범위 내에서 동작한다.
- `READ-COMMITTED`와 `REPEATABLE-READ`의 가장 큰 차이는 언두 영역에 백업된 레코드의 여러 버전 가운데 몇 번째 이전 버전까지 찾아 들어가야 하느냐이다.
- InnoDB의 트랜잭션은 고유한 트랜잭션 번호를 가지며, 언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션 번호가 포함되어 있다.
- 언두 영역의 백업된 데이터는 InnoDB에서 불필요하다고 판단할 때, 주기적으로 삭제한다.
- 가장 오래된 트랜잭션 번호보다 현재 트랜잭션 번호가 앞선 언두 영역의 데이터는 삭제할 수 없다.
- 트랜잭션이 길어지면 길어질수록 언두 영역의 데이터 역시 계속 늘어날 수 있기 때문에 성능에도 영향을 끼친다.

### 이렇게 완벽한 격리 수준에서도 부정합이 발생할 수 있다 : Phantom Read

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/a7812e84-4697-40a0-8127-3e3229a9a64b/c99fdbec-bd10-496b-acfb-487c301acc6c/Untitled.png)

`SELECT .. FOR UPDATE` 쿼리나 `SELECT .. FOR SHARE` 쿼리는 undo log가 아닌 **현재 테이블에서 데이터를 읽기 때문**에 팬텀 리드가 발생된다.

기본적으로 select .. for update는 쓰기 락을 걸어야 하는데, 언두 로그에는 락을 걸 수 없기 때문에 위 두 쿼리로 조회되는 레코드는 변경 전 데이터가 아닌 현제 레코드의 값을 가지고 오게 됩니다.

- `select for update`는 특정 레코드에 대해서 X-Lock을 거는 구문 (업데이트를 위한 베타 락)
- `select lock in share mode`는 S-Lock을 거는 구문이다.

<aside>
💡 Non-Repeatable Read랑 같은거 아니야?

여기서 '결과의 집합이 다르게 나왔다'는 점에 포커스를 맞춰야 한다. non-repeatable read의 경우 1개의 결과가 나오지만, 업데이트로 인해서 hello, hi라는 2개의 결과가 나왔고, 여기에서는 journey와 journey, cat이라는 2개의 결과가 나왔다. **동일한 트랜잭션 내에서 동일한 쿼리를 실행하더라도 다른 결과를 얻는다는 게 non-repeatable read**이고, **다른 결과 집합을 얻을 수 있다는 게 phantom-read**이다. non-repeatable read의 경우 수정된 데이터에 대한 일관성 문제를, phantom read의 경우 삽입이나 삭제된 데이터에 대한 일관성 문제를 가진다.

</aside>

### 5.4.4 SERIALIZABLE

> `InnoDB`에서는 `REPEATABLE-READ` 격리 수준에서도 PHANTOM READ가 발생하지 않으므로 굳이 SERIALIZABLE 격리 수준을 사용할 이유가 없다.
>

SERIALIZABLE 격리 수준은 가장 단순하면서도 가장 엄격한 격리 수준인 만큼 동시 처리 성능이 다른 트랜잭션 격리 수준보다 떨어진다.

읽기 작업에도 공유 잠금(읽기 잠금)을 획득해야만 하며, 동시에 다른 트랜잭션은 그 레코드를 변경할 수 없다. 즉, 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서 절대 접근할 수 없는 것이다.