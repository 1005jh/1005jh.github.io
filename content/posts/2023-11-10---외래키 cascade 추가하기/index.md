---
title: 외래키 cascade 추가하기
date: "2023-11-10T12:36:37.121Z"
template: "post"
draft: false
category: "mysql"
tags:
  - "mysql"

description: "외래키 cascade 추가하기"
---

`cascade`는 mysql의 외래키 옵션 중 하나로 무결성을 유지하며 연결된 레코드의 작업을 자동으로 해주는 옵션이다.
부모에서 `update`나 `delete`작업이 자식에게도 반영이 된다.

이 옵션을 설정해주지 않는다면 부모에서의 `update`나 `delete` 작업에서 에러를 마주할 수 있다. 어떤 에러가 발생했고 어떻게 해결했는지 포스팅해보겠다.

aliy 프로젝트를 진행하며 카테고리를 매핑하는 테이블을 만들었었다.

그 과정에서 uniqueId를 외래키로 사용했었는데 이 과정에서 `cascade`옵션을 빠트려 문제가 발생했었다.

```
Cannot delete or update a parent row: a foreign key constraint fails (매핑테이블, CONSTRAINT ... FOREIGN KEY (mainCategoryId) REFERENCES 부모테이블 (uniqueId))
```

카테고리가 변경됨에 따라 카테고리의 uniqueId를 수정하며 생긴 이슈였다. `cascade`옵션이 없었기 때문에 수정하는 과정에서 자식 테이블에 적용이 되지 않았고, 외래키 제약조건에 실패했다며 위와 같은 에러를 뱉었다.

그래서 `show create table 테이블명`을 이용해 테이블을 확인해봤다.

```mysql
CREATE TABLE 테이블 (
id int NOT NULL AUTO_INCREMENT,
... 각 카테고리 외래키,
PRIMARY KEY (id),
KEY 인덱스 (외래키),
KEY 인덱스 (외래키),
KEY 인덱스 (외래키),
KEY 인덱스 (외래키),
...
CONSTRAINT 인덱스 FOREIGN KEY (외래키) REFERENCES 카테고리 테이블 (uniqueId),
) ENGINE=InnoDB AUTO_INCREMENT=70 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

들어가야 할 `cascade`옵션이 빠져 있는 걸 확인 할 수 있었다.

그러면 `cascade`옵션을 다시 추가하려면 어떻게 해야할까?

<a href="https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html#foreign-key-adding">mysql 홈페이지</a>에서 살펴보면 외래키를 추가하기 위해서는 외래키가 참조하는 컬럼에 인덱스를 생성해야 한다고 나와있다.

```
Adding Foreign Key Constraints
You can add a foreign key constraint to an existing table using the following ALTER TABLE syntax:

ALTER TABLE tbl_name
    ADD [CONSTRAINT [symbol]] FOREIGN KEY
    [index_name] (col_name, ...)
    REFERENCES tbl_name (col_name,...)
    [ON DELETE reference_option]
    [ON UPDATE reference_option]
The foreign key can be self referential (referring to the same table). When you add a foreign key constraint to a table using ALTER TABLE, remember to first create an index on the column(s) referenced by the foreign key.
```

우선 인덱스가 걸려있으니 기존 인덱스를 삭제하고 다시 걸어주며 걸어줄 때 `cascade`옵션을 넣기로 했다.

우선 `casecade`옵션이 빠진 외래키에서 인덱스를 제거해준다.

```SQL
ALTER TABLE tbl_name DROP FOREIGN KEY fk_symbol;
예시.
ALTER TABLE 테이블명 DROP FOREIGN KEY 인덱스;
```

위의 작업이 성공했으면 다시 인덱스를 걸어준다. 기존의 삭제한 인덱스를 다시 걸어줄건데 옵션을 넣어주면 된다.

```SQL
ALTER TABLE tbl_name
    ADD [CONSTRAINT [symbol]] FOREIGN KEY
    [index_name] (col_name, ...)
    REFERENCES tbl_name (col_name,...)
    [ON DELETE reference_option]
    [ON UPDATE reference_option]
예시.
ALTER TABLE 테이블명
ADD CONSTRAINT 인덱스
FOREIGN KEY (컬럼명)
REFERENCES 부모테이블 (uniqueId)
ON DELETE CASCADE;
ON UPDATE CASCADE;
```

위와 같은 방법으로 넣어주고 에러를 해결할 수 있다.
