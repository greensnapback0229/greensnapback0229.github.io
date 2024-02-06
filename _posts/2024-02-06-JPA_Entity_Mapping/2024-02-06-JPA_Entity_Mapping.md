---
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [entity, table]
---


# `@Entity`
JPA를 사용해서 테이블과 매핑할 클래스는 `@Entity`어노테이션이 필수로 있어야한다. 

> **속성**

| 옵션명  | Description                      | 기본값    |
| ---- | -------------------------------- | ------ |
| name | JPA에서 사용할 엔티티이름 | 클래스 이름 |


⚠ 사용 주의사항
 - 기본 생성자는 필수
 - final 클래스, enum, interface, inner 클래스에는 사용할 수 없다
 - 저장할 필드에 final을 사용하면 안된다


# `@Table`
엔티티와 매핑할 테이블을 지정한다

> **속성**
