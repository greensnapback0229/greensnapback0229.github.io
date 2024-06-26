---
categories: [JPA, Entity]
tags: [entity, table]
---

# 키맵핑 전략
JPA가 제공하는 3가지 기본 키 맵핑

### `IDENTITY` : 기본키 생성을 데이터베이스에 위임한다
MySQL, PostgreSQL 등에서 사용한다. MySQL의 경우에는 `AUTO_INCREMENT`기능으로 데이터 베이스가 기본 키를 자동으로 생성한다.

`AUTO_INCREMENT`  
데이터를 `insert`할 때마다 마지막에 저장되어있는 인덱스보다 1씩 점점 커지는거다.
데이터를 삭제했을때 mysql은 계속 수를 이어나가지만 안그런 DB도 있으니 주의해야한다.

