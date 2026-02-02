---
layout: post
title: (DB) 다 같은 Count가 아니에요..!
date: 2026-01-31 24:00
category: [Database]
author: greensnapback0229
tags: [RDB, MySQL, count]
summary: Count의 동작 방식에 대한 내용입니다.

---



# 아 Count 운동 많이된다..  



> Database 대해서 공부한 내용을 정리한 글입니다.
> Count의 동작 방식에 대한 내용입니다.



## Count 쿼리

현재 테이블에 존재하는 행의 개수를 세고 싶을때 `count`함수를 사용합니다. 

- NULL을 제외하고 셉니다. 



### 사용법 

Count 함수를 사용해서 셀 수 있는 것들입니다. 

1. Column 

   ```sql
   SELECT COUNT(age) FROM user;
   ```

   - `age`가 **NULL이 아닌 행만** 셉니다.



2. `*` 아스타 링크

```sql
SELECT COUNT(*) FROM table;
```

- **행 개수 자체**를 셉니다 (NULL 상관없음)
- MySQL에서 가장 많이 쓰고, 최적화도 잘 됩니다.



3. 식(expression)

   ```sql
   SELECT COUNT(age + 1) FROM user;
   SELECT COUNT(DISTINCT email) FROM user;
   ```
   
   - 내부적으로는 **계산 결과가 NULL이냐 아니냐**만 봅니다.



4. 함수 

   ```sql
   SELECT COUNT(LENGTH(name)) FROM user;
   SELECT COUNT(IF(age > 20, 1, NULL)) FROM user;
   ```

   - 함수 결과가 **NULL이 아니면 카운트됨**



그렇다면 이 Count 함수들이 모두 같은 동작을 할까요? 

확인해보겠습니다. 

```
mysql> EXPLAIN SELECT COUNT(age) FROM count_test;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | count_test | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql>
mysql> EXPLAIN SELECT COUNT(*) FROM count_test;
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | count_test | NULL       | index | NULL          | PRIMARY | 4       | NULL |    5 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT COUNT(age + 1) FROM count_test;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | count_test | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql> EXPLAIN SELECT COUNT(DISTINCT email) FROM count_test;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | count_test | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT COUNT(LENGTH(name)) FROM count_test;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | count_test | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```



실행 계획을 확인해보면 type이 ALL인것도 있고 index인것도 있는것을 볼 수 있습니다. 

모두 같은 count 함수인데 실행계확이 다른 이유는 MySQL(InnoDB)는 row를 세지 않습니다. 

그 이유는 InnoDB는 MVCC(Multi-Version Concurrency Control) 기반 스토리지 엔진이기 때문입니다.

> **InnoDB는 정확한 row count보다
>  트랜잭션 일관성(MVCC)과 동시성을 우선했기 때문에
>  row count를 저장하지 않는다.**



## 만약 row count를 저장하면 생기는 지옥

row count를 저장하지 않는다는 것은 알겠는데 어떤 이유로 row를 세지 않을까요? 

간단하게 삽입삭제가 일어날때 수만 증감연산을 수행하면 훨씬 간단한 count가 가능할거라는 생각은 아직 지나지 않습니다. 



만약 row count를 센다고 가정해보겠습니다.

아래와 같은 메타데이터로 table의 row count를 수행한다고 생각해봅시다. 

```
table_meta.row_count = 1,000,000
```

### 동시에 발생하는 상황

- 동시에 INSERT 100개 / DELETE 50개 / ROLLBACK 20개 / 다른 트랜잭션은 아직 커밋 안 됨

  이런 상황이 지속적으로 발생한다면,

- 언제 증가?
- 언제 감소?
- 롤백 시 어떻게 복구?
- 트랜잭션별 count는?

이렇게 되면 **모든 DML마다 전역 락 필요**하게 되고 DB의 병목이 증가할 것입니다. 



## Count의 유형별 동작 및 실습



### 목데이터 삽입

1. 테이블 생성

실습을 위해 다음과 같은 Table을 생성하고 목데이터 100만개를 만드는 procedure를 실행했습니다. 

```sql
-- auto-generated definition
create table count_test
(
    id    int auto_increment
        primary key,
    age   int         null,
    email varchar(50) null,
    name  varchar(50) null
);

CREATE INDEX idx_age ON count_test(age);
CREATE INDEX idx_email ON count_test(email);
CREATE INDEX idx_name ON count_test(name);
```



인덱싱을 한 이유 

> 인덱스를 추가한 이유는 성능을 높이기 위함이 아니라,  COUNT 연산이 어떤 스캔 경로를 선택하는지 확인해보고자 추가했습니다.
>  인덱스가 있을 때와 없을 때의 실행계획 차이는 COUNT 성능의 본질이 ‘연산’이 아니라 ‘접근 경로’임을 보여줍니다.



2. 프로시저 실행

- age - NULL을 33%로 생성
- email - 모두 다르도록 unique하게 생성
- name - 100개씩 중복되도록 생성

```sql
DELIMITER $$

CREATE PROCEDURE insert_count_test(IN max_rows INT)
BEGIN
    DECLARE i INT DEFAULT 1;

    START TRANSACTION;

    WHILE i <= max_rows DO
        INSERT INTO count_test (age, email, name)
        VALUES (
            IF(i % 3 = 0, NULL, i % 80),           -- age: NULL 33%
            CONCAT('user', i, '@test.com'),        -- email: 거의 unique
            CONCAT('name', i % 100)                -- name: 중복 100개
        );

        SET i = i + 1;

    END WHILE;

    COMMIT;
END$$

DELIMITER ;
```



1. count(*)

```
mysql> EXPLAIN ANALYZE
    -> SELECT COUNT(*) FROM count_test;
+-----------------------------------------------------------------------+
| EXPLAIN                                                               |
+-----------------------------------------------------------------------+
| -> Count rows in count_test  (actual time=96.9..96.9 rows=1 loops=1)  |
+-----------------------------------------------------------------------+
```

위에서도 말했듯이 `*`를 활용한 count는 Aggregate / Scan 단계가 없는것을 볼 수 있습니다. 





2. count(age)

```
mysql> EXPLAIN ANALYZE
    -> SELECT COUNT(age) FROM count_test;
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                      |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Aggregate: count(count_test.age)  (cost=200307 rows=1) (actual time=143..143 rows=1 loops=1)
    -> Covering index scan on count_test using idx_age  (cost=100590 rows=997170) (actual time=5.62..110 rows=1e+6 loops=1)  |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.14 sec)
```

`count(*)`과는 다르게 탐색 과정에서 Aggrecate 과정이 추가된 것을 알 수 있습니다. 



#### 

- Aggregate 노드가 생겨 연산과정이 추가된것을 볼 수 있습니다.

- 또한 idx_age를 “끝까지” 스캔합니다. 스캔한 rows 수가 1,000,000개에 가깝습니다. 

  모든 row가 NULL인지 확인하기 위해서 하나하나 다 확인한 것을 알 수 있습니다.



3. Count(email)

그렇다면 NULL 없으면 빠를까요? 

```
mysql> EXPLAIN ANALYZE
    -> SELECT COUNT(email) FROM count_test;
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                           |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Aggregate: count(count_test.email)  (cost=200307 rows=1) (actual time=213..213 rows=1 loops=1)
    -> Covering index scan on count_test using idx_email  (cost=100590 rows=997170) (actual time=0.149..176 rows=1e+6 loops=1)						 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

- 마찬가지로 column을 기반으로 스캔하기 때문에 Aggregate 과정을 거칩니다. 
- NULL이 없어도 이 값에 NULL이 없는게 맞는지 확인하는 과정을 거치기 때문에 동이하게 모든 rows를 스캔하는 모습을 볼 수 있습니다. 



4. distinct를 통



### 

















