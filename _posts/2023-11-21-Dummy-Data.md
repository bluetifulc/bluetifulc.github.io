---
title: "DB에 더미데이터를 넣어보자"
author: vectorch9
date: 2023-11-21 00:18:00 +09:00
categories: [DB]
tags: [DB]
pin: false
img_path: '/assets/img/posts/'
---
현재 진행 중인 프로젝트에서 개발 서버 DB에 더미데이터를 넣어달라는 프론트단의 요청이 있었다. 이전까진 매우 소량(약 1~10개)의 데이터를 직접 DB에 삽입해주었는데, 이번엔 *더미 데이터*를 대량으로 만들어 넣어보기로 했다. DB에 좀 더 유의미한 데이터의 수를 유지하고, 추후에 성능 테스트나 쿼리 개선 시에도 활용할 수 있기 때문이다.

> **MariaDB** 10.6 버전을 기준으로 작성했다. 아마 **Mysql**도 거의 동일하게 동작할 것이다.

```sql
SELECT SUM(data_length+index_length) / 1024 / 1024 "Used" from information_schema.tables;
```

쿼리를 통해 현재 사용 중인 데이터와 인덱스가 차지하는 용량을 `MB`단위로 확인할 수 있다.

```sql
SELECT table_schema, SUM(data_length+index_length) / 1024 / 1024 "Used" from information_schema.tables group by table_schema;
```

쿼리를 통해 아래와 같이 각 테이블별 용량도 확인할 수 있다.

```
+---------------------+------------+
| table_schema        | Used       |
+---------------------+------------+
| mysql               | 2.23437500 |
| performance_schema  | 0.00000000 |
| springboard         | 0.96875000 |
+---------------------+------------+
```

## 프로시저를 활용한 방법
먼저 데이터를 삽입할 테이블은 아래와 같다.

```
MariaDB [test]> desc user;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | bigint(20)   | NO   | PRI | NULL    | auto_increment |
| username   | varchar(15)  | NO   | UNI | NULL    |                |
| name       | varchar(20)  | NO   |     | NULL    |                |
| email      | varchar(30)  | NO   | UNI | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

DB에서 지원하는 프로시저를 사용하면 간단하게 더미 데이터를 생성하고 삽입할 수 있다.

```sql
DELIMITER $$
DROP PROCEDURE IF EXISTS insertDummy$$
 
CREATE PROCEDURE insertDummy()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 10000 DO
        INSERT INTO test.user(username, name, email) 
        VALUES (concat("username", i), concat("name", i), concat("email", i, "@email.com"));
        SET i = i + 1;
    END WHILE;
END$$
DELIMITER ;

CALL insertDummy;
```

```
MariaDB [test]> CALL insertDummy;
Query OK, 10000 rows affected (10.489 sec)
```

약 10초의 시간이 소요되어 만건의 데이터가 추가되었다. 실제로 살펴보면 아래와 같이 의도한 대로 잘 삽입되었다.

```

MariaDB [test]> select * from user where id < 100;
+--------+------------+-------------------+----+
| name   | username   | email             | id |
+--------+------------+-------------------+----+
| name1  | username1  | email1@email.com  |  1 |
| name2  | username2  | email2@email.com  |  2 |
| name3  | username3  | email3@email.com  |  3 |
| name4  | username4  | email4@email.com  |  4 |
| name5  | username5  | email5@email.com  |  5 |
| name6  | username6  | email6@email.com  |  6 |
| name7  | username7  | email7@email.com  |  7 |
| name8  | username8  | email8@email.com  |  8 |
| name9  | username9  | email9@email.com  |  9 |
| name10 | username10 | email10@email.com | 10 |
| name11 | username11 | email11@email.com | 11 |
| name12 | username12 | email12@email.com | 12 |
| name13 | username13 | email13@email.com | 13 |
| name14 | username14 | email14@email.com | 14 |
| name15 | username15 | email15@email.com | 15 |
| name16 | username16 | email16@email.com | 16 |
| name17 | username17 | email17@email.com | 17 |
| name18 | username18 | email18@email.com | 18 |
| name19 | username19 | email19@email.com | 19 |
...
```

하지만 이는 이름이 `name` + `i`의 형태가 되는 상당히 규칙적인 형태로 생성된다. `rand()`와 같은 형태를 사용할 수도 있겠으나, 이는 너무 불규칙해진다.

## Faker
좀 더 현실적인 더미 데이터를 만들기 위해 프로시저가 아닌 외부 툴을 이용해보자. 국내 블로그는 `Mockaroo`라는 사이트를 많이 사용하는데, 무료 이용자는 최대 `1000 rows`라는 제한이 있다보니 상당히 번거롭다. `filldb`라는 사이트도 찾아보았는데 최대 `10000 rows`라 살짝 아쉬웠다.

그렇게 찾은 방법이 **파이썬의 Faker 라이브러리**를 이용하는 방식이다. `Faker`는 이름대로 가짜 데이터를 만들어주는 라이브러리다. (뭔가 이름이 익숙하다면 기분 탓이다..)

```python
from faker import Faker
import pandas as pd

Faker.seed(1234) # 시드 설정이 가능하다.
fake = Faker()

rows = 1000
sqls = []

for _ in range(rows):
	name = fake.name()
	username = fake.unique.user_name() # unique를 통해 고유함을 보장할 수 있다.
	email = fake.unique.free_email()
  
	sql = f'INSERT INTO `user` (name, username, email) VALUES (\'{name}\' ,\'{username}\', \'{email}\');'
	sqls.append(sql)


with open(r'./test.sql', 'w') as file:
	for sql in sqls:
		file.write(sql + "\n")
```

사용법은 직관적이어서 따로 설명하진 않겠다. 코드가 깔끔하진 않으나 더미 데이터를 생성하기 위한 일회용 코드이므로 크게 신경쓰지 않았다.

> 더 자세한 사항과 API들에 대해선 [공식 문서](https://faker.readthedocs.io/en/master/index.html)를 찾아보자. 정말 다양한 API를 지원한다.

한가지만 짚고 넘어가면, DB상에 `enum` 클래스를 저장하고, 각 `enum` 값의 비율이 서로 다른 경우가 있다. 

예를 들어, 성별을 저장하면 `MALE`, `FEMALE`의 값이 존재할 것이다. 만약 서비스 특성상 남성 회원이 많다면, 남성: 여성 = 9: 1 의 비율을 갖는 데이터를 원할 수 있다.

```python
enum = OrderedDict([("MALE", 0.90), ("FEMALE", 0.10)])
grade = [fake.random_element(elements=enum) for _ in range(rows)]
```

이는 위처럼 `OrderedDict`를 활용하여 각 값이 나올 가중치를 정해줄 수 있다.

```
(array(['FEMALE', 'MALE']), array([104, 896]))
```

1000개의 데이터로 테스트 해보면 위처럼 9대1의 비율이 유지되어 샘플링된다.

저장된 sql 파일은 아래와 같이 실행시킬 수 있다.

```bash
mysql -u {username} -p {db_name} < test.sql
```

```
MariaDB [test]> select id, username, name from user;
+-------+-----------------+----------------------+
| id    | username        | name                 |
+-------+-----------------+----------------------+
| 11506 | ywilliams       | Clifford Price       |
| 11507 | laurawu         | William Mitchell     |
| 11508 | johnholt        | Julie Osborne        |
| 11509 | andrewtownsend  | Albert Poole         |
| 11510 | butlerbrenda    | Javier Brandt        |
| 11511 | greenapril      | Dominic Delgado      |
| 11512 | brian82         | Allison Ingram       |
| 11513 | kirsten44       | Kyle Wright          |
| 11514 | lydia13         | Joshua Bates         |
| 11515 | cummingslaura   | James Estes          |
| 11516 | kwilson         | Jordan Perkins       |
...
```

잘 입력된 모습이다. (`id`는 여러번 테스트 해보느라 auto increment 값이 커졌다.)

직접 재보진 않았으나 체감 상 프로시저를 이용한 경우보다 약간 느렸다. 대신 실제에 더 가까운 형태의 더미 데이터를 만들어 냈다.

### 장단점
이 방식의 가장 큰 장점은 데이터 수가 제한되어 있지 않으며, 원하는 만큼 커스텀이 가능하단 점이다. 이 글에서 소개하진 않았으나 `locale`설정을 통해 해당 나라에 알맞는 데이터를 생성할 수도 있다. (한국의 경우도 완벽하진 않으나 어느정도 동작한다.) 정말 다양한 형태의 데이터를 지원하니, 토이 프로젝트 수준에선 거의 모든 데이터를 생성할 수 있을 것이다.

단점으론, 직접 코드로 짜주어야 하기 때문에 꽤 번거롭다. 다만 '더미 데이터 생성'은 일회성 작업이기 때문에 큰 문제는 되지 않는다고 생각한다. 

파이썬을 모르더라도 Js, Ruby, Java를 비롯하여 다양한 생태계의 `Faker` 라이브러리가 존재하므로 필요하다면 찾아보자.