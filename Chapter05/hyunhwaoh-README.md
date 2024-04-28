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
```

MyISAM

### 주의사항


## 2. MySQL 엔진의 잠금

### 글로벌 락

### 테이블 락


### 네임드 락


### 메타데이터 락


## 3. InnoDB 스토리지 엔진 잠금

### InnoDB 스토리지 엔진의 잠금


### 인덱스와 잠금


### 레코드 수준의 잠금 확인 및 해제


## 4. MySQL 의 격리 수준

### READ UNCOMMITTED


### READ COMMITTED


### REPEATABLE READ


### SERIALIZABLE

