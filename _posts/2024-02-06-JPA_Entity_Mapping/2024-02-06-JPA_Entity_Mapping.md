---
categories: [JPA, Entity]
tags: [entity, table]
---


# @Entity
JPA를 사용해서 테이블과 매핑할 클래스는 `@Entity`어노테이션이 필수로 있어야한다. 

> **속성**

| 옵션명  | Description                      | 기본값    |
| ---- | -------------------------------- | ------ |
| name | JPA에서 사용할 엔티티이름 | 클래스 이름 |


⚠ 사용 주의사항
 - 기본 생성자는 필수
 - final 클래스, enum, interface, inner 클래스에는 사용할 수 없다
 - 저장할 필드에 final을 사용하면 안된다


# @Table
엔티티와 매핑할 테이블을 지정한다

> **속성**

| 옵션명  | Description                      | 기본값    |
| ---- | -------------------------------- | ------ |
| name | 맵핑할 테이블 이름 | 엔티티 이름을 사용 |
| catalog | catalog 기능이 있는 DB에서 catalog를 제공한다 | 
| schema | schema 기능이 있는 DB에서 schema를 제공한다 |
| uniqueConstraints | DDL 작성시 유니크 제약조건을 만든다. |

**uniqueContraints 옵션**
자동 생성 DDL에서 테이블 생성시에 UNIQUE 제약조건을 추가할 수 있는 옵션
DDL 생성시에만 동작하고 JPA의 실질적 동작이랑은 아무관련이 없다
즉 DDL 자동생성 옵션이 none이면 사실상 필요없는 옵션
***하지만 Entity 코드상 명시용으로 확인하기 용이하다***

사용법)
```java
@Table(name="MEMBER", uniqueConstraints = { @UniqueConstraint(
	name = "NAME_AGE_UNIQUE",
	columnNames = {"NAME", "AGE"}
)})
```

SQL에서 적용되는 쿼리)
```sql
alter table MEMBER 
    add constraint NAME_AGE_UNIQUe unique (NAME, AGE)
```


# 데이터베이스 스키마 자동생성

JPA는 데이터베이스 스키마를 자동으로 생성하는 기술을 지원한다.
application.properties 파일을 JPA 및 RDB 연동을 위해 작성했는데 그 중
```xml
spring.jpa.hibernate.ddl-auto=update
```
여기서 `spring.jpa.hibernate.ddl-auto`의 속성을 가지고 어플리케이션 실행 시점에 DB 테이블을 자동으로 만들거나 변경 점을 업데이트할 수 있다. 

> **속성**

| 옵션 | 설명 | 
| --- | --- | 
| create | 기존 테이블을 삭제하고 새로 생성한다. DROP + CREATE | 
| create-drop | create 속성에 추가로 어플리케이션을 종료할 때 생성한 DDL 을 제거한다. DROP + CREATE + DROP | 
| update | 데이터베이스 테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 애플리케이션을 실행하지 않는다. DDL을 수정하지 않는다. |
| none | 그냥 자동 생성기능을 사용하지 않는다. |


<blockquote class="prompt-danger">
    <dt>
        <code class="language-plaintext highlighter-rouge">
            실제 운영할땐 어떻게 사용할까?
        </code>
    </dt>
    create, create-drop, update 기능은 사용하면 운영중인 DB의 데이터를 날릴수도 있어서 사용하지 않는 기능이라고 한다.  
</blockquote>

JPA 2.1부터는 스키마 자동생성 기능을 표준으로 지원하지만 updtae, validate 속성을 지원 X 

JAVA에선 CamelCase를 주로 사용하지만 DB에서는 snake_case를 많이 사용한다 이 둘의 차이를 극복하기 위한 JPA 기술이 있다. 

```xml
# 자동 변경
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy

# 변경없이 그대로 쓰기
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

`CamelCaseToUnderscoresNamingStrategy`옵션을 application.properties에 적용하면 자동으로 Camel케이스가 `_(underscore)`를 사용하는 snake 케이스로 자동 변환된다. 

# 기본키 맵핑 
