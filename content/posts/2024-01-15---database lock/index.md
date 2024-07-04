---
title: database lock
date: "2024-01-15T12:36:37.121Z"
template: "post"
draft: false
category: "mysql"
tags:
  - "mysql"

description: "database lock"
---

database lock은 말 그대로 잠금을 거는 것으로 동시에 여러 사용자가 하나의 데이터를 조작할 수 없도록 하는 기능이다. 즉 동시성을 제어할 수 있는 기능인데 MySQL에서의 lock에 대해 알아보자.

MySQL에서의 lock은 크게 MySQL엔진 lock과 스토리지 엔진 lock으로 나뉜다. MySQL엔진 lock은 모든 스토리지 엔진에 영향을 끼치지만 스토리지 엔진 lock은 스토리지 엔진 간 영향을 주지 않는다. lock의 종류로는 글로벌 락, 테이블 락, 메타데이터 락, 네임드 락이 있고, 스토리지 엔진 lock으로는 레코드 락, 갭 락, 넥스트 키 락, 자동 증가 락이 있다.

### 글로벌 락

글로벌 락은 가장 범위가 넓은 락으로 `FLUSH TABLES WITH READ LOCK`명령어를 통해 획득할 수 있다. 한 세션에서 글로벌 락을 획득하게 되면 다른 세션에서는 `SELECT`를 제외한 대부분 DDL,DML 문장은 글로벌 락이 해제될 때까지 대기상태로 남는다. 글로벌 락은 MySQL 서버 전체에 영향을 주며, 작업 대상 테이블이나 데이터베이스가 다르더라도 동일하게 영향을 준다.

`FLUSH TABLES WITH READ LOCK`를 통한 글로벌 락은 MySQL서버의 모든 변경 작업을 멈추지만 InnoDB의 사용이 일반화 되며, InnoDB의 트랜잭션 때문에 일관된 데이터 상태 유지를 위해 모든 데이터 변경 작업이 멈출 필요가 없어졌다.

```SQL
---read lock을 획득
mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.00 sec)
--- select query
mysql> select * from dept_emp limit 100;
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10001 | d005    | 1986-06-26 | 9999-01-01 |
|  10002 | d007    | 1996-08-03 | 9999-01-01 |
...
|  10089 | d007    | 1989-01-10 | 9999-01-01 |
|  10090 | d005    | 1986-03-14 | 1999-05-07 |
|  10091 | d005    | 1992-11-18 | 9999-01-01 |
+--------+---------+------------+------------+
```

위의 자료에서 select query는 성공함을 알 수 있다. 여기서 update query(dml)를 날려보면

```sql
---데이터 변경 dml
mysql> update dept_emp set from_data = 2024-02-02 where emp_no = 10091;
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock;
---테이블 생성 ddl
mysql> create table test;
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock;
```

읽기락을 획득했기 때문에 변경작업을 할 수 없다고 나온다. 추가로 테이블 생성(ddl) 또한 안되는 걸 확인할 수 있다.
ddl,dml에 해당하는 테이블 생성, 수정, 삭제와 데이터 삽입, 수정, 삭제가 불가하다.

### 테이블 락

테이블 락은 개별 테이블 단위로 설정되는 락이다. 명시적 또는 무시적으로 획득할 수 있으며, 명시적 획득은 `LOCK TABLES 테이블이름 [ READ | WRITE ]`로 할 수 있다. 명시적으로 획득한 락은 `UNLOCK TABLES`로 해제할 수 있다. 묵시적은 MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리 실행 시 발생한다. 하지만 InnoDB 테이블의 경우 레코드 락을 제공하기 때문에 단순 데이터 변경 쿼리로 묵시적 테이블 락이 설정되지 않는다. 단, 스키마를 변경하는 쿼리의 경우에는 영향을 미친다.
세션 1에서 락을 걸고 세션2에서 확인을 해보면

```SQL
세션 1
mysql> lock tables titles WRITE; -- titles 테이블에 잠금 설정
Query OK, 0 rows affected (0.00 sec)

세션 2
select * from titles; --대기중

세션 1
unlock tables; -- lock 해제

세션 2
...
| 499998 | Senior Staff       | 1998-12-27 | 9999-01-01 |
| 499998 | Staff              | 1993-12-27 | 1998-12-27 |
| 499999 | Engineer           | 1997-11-30 | 9999-01-01 |
+--------+--------------------+------------+------------+
443308 rows in set (1 min 29.07 sec) -- 쿼리 실행
```

위와 같이 됨을 확인할 수 있다.
