---
title: 외래키 변경하기
date: "2023-11-20T12:36:37.121Z"
template: "post"
draft: false
category: "mysql"
tags:
  - "mysql"

description: "외래키 변경하기"
---

aliy 프로젝트 중 외래키를 변경해야 하는 일이 생겼다.

각 카테고리에 대한 uniqueid를 외래키로 사용중이었는데 방식이 바뀌며 uniqueid를 사용하지 않게되어 pk로 변경을 해야했다.

그래서 변경하는 과정을 포스팅을 해보겠다.

```SQL
CREATE TABLE 테이블 (
id int NOT NULL AUTO_INCREMENT,
... 각 카테고리 외래키,
PRIMARY KEY (id),
...
KEY 인덱스 (컬럼명),
...
CONSTRAINT 인덱스 FOREIGN KEY (mainCategoryId) REFERENCES 카테고리 테이블 (uniqueId) ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=216 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

우선 show create table 로 확인을 해보면 위처럼 나왔다.

각 카테고리는 pk를 가지고 있으며 uniqueid를 가지고 있었다.
그렇기 때문에 우선적으로 각 카테고리의 uniqueid에 대한 pk를 가지고 와야했다.

그렇기 때문에 우선적으로 컬럼을 추가해 uniqueid에 맞는 pk를 가져왔다.

```SQL
-- pk를 저장할 컬럼 생성
alter table 테이블 add column cate_main int
-- uniqueid로 pk 세팅
update 테이블 join mainCategory on 테이블.mainCategoryId = mainCategory.uniqueId set 테이블.cate_main = mainCategory.id
```

위 작업을 해준 후 외래키를 `cate_main`이 되게끔 해주기 위해 기존의 uniqueid에 걸려있는 외래키 인덱스를 제거해줬다.

```SQL
alter table 테이블 drop foreign key 인덱스
```

위 작업에 쓰인 인덱스는
`CONSTRAINT 인덱스 FOREIGN KEY (카테고리 외래키 컬럼명) REFERENCES 카테고리 테이블 (uniqueId) ON UPDATE CASCADE`에서의 인덱스이다.

그리고 기존의 컬럼명을 다시 사용하기 위해 기존의 mainCategoryId 컬럼을 삭제해준 후 새로 만든 `cate_main`의 컬럼명을 변경해줬다.

```SQL
-- 기존 uniqueid를 가지고 있는 외래키 삭제
ALTER TABLE 테이블
DROP COLUMN mainCategoryId;
-- cate_main 컬럼명 변경
alter table 테이블 change column cate_main mainCategoryId int not null
```

위 작업이 끝나면 기존의 uniqueid를 가지고 있던 mainCategroyId는 사라지고, pk를 저장한 cate_main컬럼이 mainCategroyId가 되어있다.
마지막으로 삭제를 한 외래키의 인덱스를 다시 걸어주면 된다.

```SQL
alter table 테이블
add constraint 인덱스
foreign key (mainCategoryId)
references 'mainCategory'(id)
on update cascade
```

위의 작업이 성공했다면 mainCategoryId에 대한 값이 uniqueId -> pk로 변경이 완료된다.

비즈니스 로직 변경도 필요하다.
