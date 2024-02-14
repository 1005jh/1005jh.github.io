---
title: database index 효율
date: "2024-01-08T12:36:37.121Z"
template: "post"
draft: false
category: "mysql"
tags:
  - "mysql"

description: "database index 효율"
---

index는 database의 검색속도를 향상시키는 도구이다.

index를 걸 때 우리는 중복도가 낮고, 자주 바뀌지 않는 column에 하게 된다. 여기서 중복도에 따라 인덱싱을 했을 때 차이를 알아보자.

database는 mysql에서 제공하는 sample employees를 이용했다.

```SQL
mysql> show tables;
+----------------------+
| Tables_in_employees  |
+----------------------+
| current_dept_emp     |
| departments          |
| dept_emp             |
| dept_emp_latest_date |
| dept_manager         |
| employees            |
| salaries             |
| titles               |
+----------------------+
8 rows in set (0.01 sec)
--
mysql> select COUNT(*) from salaries;
+----------+
| COUNT(*) |
+----------+
|  2844047 |
+----------+
1 row in set (0.08 sec)

```

여기서 가장 데이터가 많은 salaries 테이블을 이용해서 테스트를 해보자.

우선 `show create table salaries`를 하면 아래와 같은 결과를 볼 수 있다.

```SQL
CREATE TABLE `salaries` (
  `emp_no` int NOT NULL,
  `salary` int NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`,`from_date`),
  CONSTRAINT `salaries_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

pk로 emp_no과 from_date의 조합으로 되어 있고, emp_no에 인덱싱이 되어 있는 걸 볼 수 있다.(pk)
여기서 중복도를 체크해보자면

```SQL
mysql> select COUNT(DISTINCT(from_date)) from_date, COUNT(DISTINCT(to_date)) to_date, COUNT(DISTINCT(emp_no)) emp_no, COUNT(DISTINCT(salary)) salary from salaries;
+-----------+---------+--------+--------+
| from_date | to_date | emp_no | salary |
+-----------+---------+--------+--------+
|      6392 |    6120 | 300024 |  85814 |
+-----------+---------+--------+--------+
1 row in set (2.91 sec)
```

위와 같은 결과를 볼 수 있다. to_date에 중복도가 가장 높고, pk를 제외하면 salary가 중복도가 가장 낮은 걸 확인할 수 있다.

여기서 중복도가 가장 높고, 가장 낮은 to_date와 salary를 이용해 쿼리를 하나 날려보자.

```SQL
mysql> SELECT * FROM salaries  WHERE salary = 74447;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  16772 |  74447 | 2000-10-26 | 2001-10-26 |
...
| 483839 |  74447 | 1992-06-19 | 1993-06-19 |
+--------+--------+------------+------------+
38 rows in set (0.65 sec)
```

2844047개의 데이터 중 38개의 데이터가 나왔고, 시간은 0.65초가 걸렸다.

여기에 중복도가 가장 낮은 salary 컬럼에 인덱싱을 해보자.

```SQL
mysql> create index salary_index on salaries (salary)
    -> ;
Query OK, 0 rows affected (3.43 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

인덱스를 생성하는데 총 3.43초가 걸렸다. 그러면 인덱스가 생겼는지 확인해보자.

```SQL
mysql> show create table salaries;
CREATE TABLE `salaries` (
  `emp_no` int NOT NULL,
  `salary` int NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`,`from_date`),
  KEY `salary_index` (`salary`),
  CONSTRAINT `salaries_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

salary_index로 넣어준 인덱스가 제대로 걸려있는 걸 확인할 수 있다. 그러면 다시 아까 날렸던 쿼리를 날려보자.

```SQL
mysql> SELECT * FROM salaries USE INDEX (salary_index) WHERE salary = 74447;
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  16772 |  74447 | 2000-10-26 | 2001-10-26 |
...
| 483839 |  74447 | 1992-06-19 | 1993-06-19 |
+--------+--------+------------+------------+
38 rows in set (0.00 sec)
```

이전 0.65초가 걸린 것보다 훨씬 빠르게 데이터를 가져왔다. workbench로 확인해보면 0.0014초가 걸렸다고 한다.

그러면 이제 to_date에 걸어서 확인을 해보자.

```sql
mysql> SELECT * FROM salaries  WHERE to_date = '1993-11-04';
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  10273 |  88989 | 1992-11-04 | 1993-11-04 |
...
| 499285 |  45284 | 1992-11-04 | 1993-11-04 |
+--------+--------+------------+------------+
409 rows in set (0.66 sec)
```

총 409개의 데이터를 가져왔고 0.66초가 걸렸다.

이제 인덱스를 걸고 확인을 해보자.

```sql
mysql> create index td_idx on salaries (to_date);
Query OK, 0 rows affected (3.38 sec)
Records: 0  Duplicates: 0  Warnings: 0
--
mysql> SELECT * FROM salaries USE INDEX (td_idx) WHERE to_date = '1993-11-04';
+--------+--------+------------+------------+
| emp_no | salary | from_date  | to_date    |
+--------+--------+------------+------------+
|  10273 |  88989 | 1992-11-04 | 1993-11-04 |
...
| 499285 |  45284 | 1992-11-04 | 1993-11-04 |
+--------+--------+------------+------------+
409 rows in set (0.01 sec)
```

이 역시도 확연하게 줄어든 것을 확인할 수 있지만 중복도가 낮은 salary 컬럼에 비해 적은 걸 확인할 수 있다.
위의 쿼리는 workbench로 확인했을 때 0.0069초가 걸렸다.

중복도에 따라 인덱싱을 했을 때 효율의 차이를 알아볼 수 있었다. 더 많은 데이터에서 테스트를 했다면 더 확연한게 차이를 볼 수 있을 것 같다.
