# 3주차

날짜: 2023년 1월 29일

# 5.1 트랜잭션

---

> **Transaction(트랜잭션)의 4가지 성질 (ACID)**
> 
> - **Atomicity(원자성)**
>     - all or nothing
>     - 트랜잭션의 모든 연산들이 정상 완료되거나 어떠한 연산도 수행되지 않은 상태를 보장
> - **Consistency(일관성)**
>     - 공적으로 수행된 트랜잭션은 정당한 데이터들이 데이터베이스에 반영되었음을 의미
>     - DB에서 데이터 변경 시 사전에 설정되어 있는 룰에 맞지 않는 데이터가 들어가는 것을 방지
> - **Isolation(독립성/고립성)**
>     - 여러 트랜잭션이 동시에 수행 되더라도 각각의 트랜잭션은 다른 트랜잭션의 수행에 영향을 받지 않고 독립적으로 수행되어야 함
>     - 다수의 세션 또는 유저가 같은 시간에 같은 데이터에 접근하고 처리 중일 때 수행 중인 트랜잭션이 완료 될 때 까지 다른 트랜잭션이 끼어 들지 못하게 함으로써 데이터의 누락이나 잘못된 데이터에 대한 방지
> - **Durability(지속성)**
>     - 트랜잭션이 성공적으로 완료되어 커밋되었다면, 해당 트랜잭션에 의한 반영된 모든 변경은 향후에 어떤 소프트웨어나 하드웨어 장애가 발생 되더라도 보존되어야 함

## 5.1.1 MySQL에서의 트랜잭션

---

### **Transaction(트랜잭션)**

- 작업의 완전성을 보장해 주는 것
- 논리적인 작업 셋을 모두 완벽하게 처리해야함
    - 완벽하게 처리하지 못한다면, 원 상태로 복구해야함
    - 일부만 적용되는 현상(Partial update)이 발생하지 않아야함
- 쿼리의 개수와 관계없이 논리적인 작업 자체가 100% 또는 0% 적용되야함
    - 100% : COMMIT 실행 시
    - 0% : ROLLBACK 시
- 스토리지 엔진에 따른 트랜잭션 오류 시, 실행 결과
    - MyISAM : 트랜잭션의 오류에도 부분 업데이트 실행 → **데이터 정합성(모순 없이 일치함) 문제**
    - InnoDB : 트랜잭션에 일부라도 오류 시, 복구

```jsx
mysql> create table tb_tab_myisam(col int primary key) ENGINE=MyISAM;
mysql> insert into tb_tab_myisam values (3);

mysql> create table tb_tab_innodb(col int primary key) ENGINE=InnoDB;
mysql> insert into tb_tab_innodb values (3);

mysql> insert into tb_tab_myisam values(1),(2),(3);
Error Code: 1062. Duplicate entry '3' for key 'tb_tab_myisam.PRIMARY'

mysql> insert into tb_tab_innodb values(1),(2),(3);
Error Code: 1062. Duplicate entry '3' for key 'tb_tab_innodb.PRIMARY'

-- MyISAM
mysql> select * from tb_tab_myisam;
+-----+
| col |
+-----+
|   1 |
|   2 |
|   3 |
+-----+

-- InnoDB
mysql> select * from tb_tab_innodb;
+-----+
| col |
+-----+
|   3 |
+-----+
```

## 5.2.2 주의사항

---

> 트랜잭션의 범위는 최소화하라.
> 
> - 꼭 필요한 최소의 코드에만 적용하는 것이 좋다.

<aside>
<img src="https://www.notion.so/icons/report_blue.svg" alt="https://www.notion.so/icons/report_blue.svg" width="40px" /> 처리시작 - > 
**트랜잭션 시작** ->사용자 로그인 여부 확인 -> 사용자 글쓰기 내용 오류 여부 확인 -> 
첨부된 파일의 업로드 가능 확장자 및 용량 확인 , 저장 -> 
**사용자 입력 내용(글 과 첨부파일 정보) 을 DB에 저장** -> 
저장 된 내용 조회 및 추가 정보 DB에서 조회 -> 게시물에 대한 알림 메일 발송 -> 
**메일 발송 이력 DB에 저장** -> **트랜잭션 종료** -> 
처리 완료

</aside>

- 트랜잭션 시작부터 종료까지 직접 DB에 저장하는 행동은 2번 밖에 없음
    - 1) 사용자 입력 내용 DB 저장, 2) 메일 발송 이력 DB 저장
    - 위 2개의 행동에만 트랜잭션을 시작 및 종료하면 문제를 피할 수 있다.
- **외부 로직의 문제로 장시간 소요될 경우,** 전체적인 처리가 매우 늦어지거나 문제 가능성 높음
    - 특히 ,네트워크 작업이 있는 경우 반드시 트랜잭션에서 배제
    - DBMS 서버가 높은 부하상태로 빠지거나 위험한 상태로 빠질 가능성 높음

# 5.2 MySQL 엔진의 잠금

---

- 잠금의 레벨
    - MySQL 레벨의 잠금
        - 모든 스토리지 엔진에 영향을 미침
        - 테이블 락 외에도 메타데이터 락(테이블 구조 잠금), 네임드 락 등 기능 제공
    - 스토리지 엔진 레벨의 잠금
        - 스토리지 엔진 간 상호 영향 없음

## 5.2.1 Global Lock(글로벌 락)

---

> **DML(Data Manipulation Language, 데이터 조작어)**란?
> 
> - 정의된 데이터베이스에 입력된 레코드를 조회, 수정, 삭제 등의 역할을 하는 언어
> - SELECT INSERT UPDATE DELETE

> **DDL(Data Definition Language, 데이터 정의어)**란?
> 
> - 데이터베이스를 정의하는 언어 (데이터 전체의 골격을 결정)
> - CREATE ALTER DROP TRUNCATE
- `FLUSH TABLES WITH READ LOCK` 명령으로 획득
- MySQL에서 제공하는 잠금 가운데 가장 범위가 큼
- 한 세션에서 글로벌 락 획득 시,
    - 다른 세션에서 SELECT를 제외한 대부분의 DDL, DML 문장 실행이 대기상태로 남음
- 글로벌 락의 영향 범위 : MySQL 서버 전체
    - 작업 대상 테이블이나 DB가 다르더라도 동일하게 영향
- MySQL 8.0 이후
    - InnoDB 기본 스트리지 엔진 채택
    - Backup Lock(백업 락) 도입
        - Xtrabackup이나 Enterprise Backup과 같은 백업 툴들의 안정적 실행을 위한 백업 락 도입
    - 코드
    
    ```sql
    mysql> LOCK INSTANCE FOR BACKUP;
    -- // 백업 실행
    mysql> UNLOCK INSTANCE;
    ```
    
    - 특정 세션에서 백업 락 획득 시, 모든 세션에서 다음 정보는 변경 불가
        - 데이터베이스 및 테이블 들 모든 객체 생성 및 변경, 삭제
        - REPAIR TABLE과 OPTIMIZE TABLE 명령
        - 사용자 관리 및 비밀번호 변경
    - 하지만 일반적인 테이블의 데이터 변경은 허용
    - 백업의 실패를 막기 위해서 DDL 명령이 실행되면 복제를 일시 중지하는 역할

## 5.2.2 Table Lock(테이블 락)

---

- 개별 테이블 단위로 설정되는 잠금
- 명시적 또는 묵시적으로 특정 테이블 락 획득 가능
- 명시적으로 획득한 테이블 락
    - `LOCK TABLES table_name [READ | WRITE]` 명령으로 특정 테이블 락 획득
    - `UNLOCK TABLES` : 잠금 반납(해제)
    - 글로벌 락과 동일하게 온라인 작업에 상당한 영향을 미침
- 묵시적인 테이블 락
    - 쿼리가 실행되는 동안 자동으로 획득됐다가 쿼리가 완료되고 자동 해제
    - 하지만 InnoDB 테이블은 스토리지 엔진 차원에서 레코드 기반의 잠금 제공
        - 단순 데이터 변경 쿼리로 인해 묵시적인 테이블 락이 설정되지 않음
        - 테이블 락이 설정은 되지만 DDL의 경우에만 영향

## 5.2.3 Named Lock(네임드 락)

---

- 단순히 사용자가 지정한 **문자열**에 대해 획득하고 반납하는 잠금
- 사용처
    - 여러 클라이언트가 상호 동기화를 처리해야 하는 경우
    - 많은 레코드에 대해 복잡한 요건으로 레코드를 변경하는 트랜잭션일 경우
- MySQL 8.0 이후, 네임드 락 중첩 사용 가능
    - 한 번에 락 해제도 가능
- 코드

```jsx
// mylock 이라는 문자열 잠금 관련 코드
mysql> SELECT GET_LOCK('mylock', 2); // 잠금 사용중일 경우, 2초 대기
mysql> SELECT IS_FREE_LOCK('mylock'); // 해당 문자열의 잠금 설정 확인
mysql> SELECT RELEASE_LOCK('mylock'); // 잠금 반납(해제)
// 2개 이상의 네임드 락 사용 가능 (MySQL 8.0 이후)
mysql> SELECT GET_LOCK('mylock_1', 10);
mysql> SELECT GET_LOCK('mylock_2', 10);
mysql> SELECT RELEASE_ALL_LOCK();  // 모든 네임드 락 해제
```

## 5.2.4 Metadata Lock(메타데이터 락)

---

- 데이터베이스 객체(테이블, 뷰 등)의 이름이나 구조를 변경하는 경우에 잠금 획득
    - 명시적으로 획득하거나 해제 할 수 있는 것이 아님
    - RENAME TABLE 명령과 같이 테이블 이름 변경 시, 자동 획득
    - RENAME TABLE 명령은 원본 이름과 변경 이름 모두 잠금 설정
- 한 세션에서 트랜잭션을 명시적으로 시작하고 데이터를 조회 중
    - 다른 세션에서 테이블의 구조를 변경하게 되면 Metadata Lock 으로 대기를 하게 됩니다.

```sql
-- Session 1
select CONNECTION_ID();
+-----------------+
| CONNECTION_ID() |
+-----------------+
|               4 |
+-----------------+
<-- Session 1의 Process ID 는 4 로 확인됨

-- 트랜잭션 시작
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user_info;
+----+------+
| id | name |
+----+------+
|  1 | jade |
|  2 | Tom  |
+----+------+
2 rows in set (0.00 sec)

-- Session 2 : 컬럼 추가 시도
mysql> alter table user_info add column col2 varchar(100);
<!!--- 락 획득 실패 및 락에 의해 대기중

-- Session 1 : Full processlist
mysql> show full processlist;
+----+------------+------+------+---------------------------------+----------------------------------------------------+
| Id | User       | db   | Time | State                           | Info                                               |
+----+------------+------+------+---------------------------------+----------------------------------------------------+
|  2 | system user| NULL | 1321 | Connecting to master            | NULL                                               |
|  4 | root       | npm  |    0 | starting                        | show full processlist                              |
|  8 | root       | npm  |   10 | Waiting for table metadata lock | alter table user_info add column col2 varchar(100) |
+----+------------+------+------+---------------------------------+----------------------------------------------------+
<!!-- Metadata lock 에 의해 Session 2가 대기 하는 상황
```

## 참고사이트

---

- MySQL 트랜잭션과 잠금 참고

[MySQL - 트랜잭션 과 잠금 (1) - Real MySQL 8.0](https://hoing.io/archives/4182)

- DDL, DML, DCL 용어 참고

[DDL, DML, DCL 이란?](https://velog.io/@ksk5401/DDL-DML-DCL-%EC%9D%B4%EB%9E%80)