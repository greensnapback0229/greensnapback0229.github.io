---
categories: [JPA, Persistence_Manage]
tags: [Persistence, 영속성]
---

**Persistence Unit(영속성 유닛)**  
응용프로그램에서 EntityMnager 인스턴스에 의해서 관리되는 모든 엔티티 클래스의 집합. 

### EntityManagerFactory
`EntityManager`를 만드는 공장인데 EntityManager에서 여러 작업이 이루어지기 때문에 공장을 만드는 것은 비용이 꽤나 크다 .
application.properties 파일을 참고하여 영속성 유닛을 지정해서 EntityManagerFactory를 생성해야한다. `Persistence`클래스를 사용한다.  
얘가 설정 정보를 읽어서 JPA를 동작시키기 위한 기반 객체를 만든다. 어플리케이션 전체에서 한 번만 생성하고 공유해서 사용해야 한다.

### EntityManager
JPA 기능 대부분을 제공하는 클래스다. 내부에 데이터 소스를 유지하면서 DB와 통신하낟. 엔티티 메니저를 가상의 DB로 생각할 수 있다. 
DB 커넥션과 밀접한 관계가 있어서 스레드 간에 공유하거나 재사용하면 안된다. 

### 얘네 둘이 동작하는 방식
`EntityMangerFactory`를 만들고 Factory에서 `EntityManager`를 만든다.
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("persistence_unit_name");
EntityManager em = emf.createEntityManager();
```
JPA구현체들은 `EntityManagerFactory`를 만들때 DB 커넥션 풀을 만든다.
생성된 `EntityManger`는 데이터베이스를 연결할 시점이 오지 않는이상 커넥션 풀에서 커넥션을 얻지 않는다.


### Persistence Context
영속성 컨텍스는 책에서는 엔티티를 영구 저장하는 환경이라고 말했다. 나는 왜 Context라는 워딩을 했을까를 생각하며 이해를 해봤는데 맥락, 문맥이라고 생각해보면 이 데이터(`Entity`)가 어떤 함수(persist(), find() 등)를 만났을때 어떤 맥락에 있는지를 관리해주는 환경정도로 생각했다.  
결국 주요 역할은 **엔티티 메니저로 엔티티를 저장하거나 조회하면 엔티티 매니저는 영속성 컨택스르에 엔티티를 저장하고 보관한다.**
```java
em.persist(member);
```
영속성 컨택스트를 이야기해보면 결국 위 코드는 저장하는게 아니라 `EntityManger`를 사용해서 엔티티를 영속성 컨택스트에 저장하는 것이다. 



### Entity의 생명주기
> **비영속(new/transient)**

영속성 컨택스트와 전화 관계가 없는 상태로 아직 EntityManager를 통해서 저장을 하지 않고 엔티티 자체에 데이터만 있는 상태이다.

> **영속(managed)**  

영속성 컨택스트에 저장된 상태다.   
//이때 주의할 것은 영속성 컨택스트에 저장되었다고해서 DB에 저장된 것은 아니다. 

> **준영속(detached)**  

영속성 컨택스트에 저장되었다가 분리된 상태, `em.detach()`를 호출하면 엔티티를 비영속 상태로 만들 수 있다. `em.close()`를 호출해서 영속성 컨택스트를 닫거나 `em.clear()`로 초기화해도 준영속 상태가 된다.

> **삭제(removed)**  
엔티티를 영속성 컨택스트와 DB에서 삭제한다.

```java
//비영속 상태
Memeber member = new member();
member.setId(1);
member.setUserName("name")

//영속성 컨택스트에 저장
em.persist(member);

//준영속 상태
//close clear도 마찬가지
em.detch(member); 
```

### 영속성 컨텍스트 동작
영속성 컨택스트는 1차 캐시를 가지고 있다. 영속상태의 엔티티는 여기에 저장된다. 이 캐시 내부에는 Map이 있는데 이 Map의 식별은 Entity를 만들때 정의한 `@Id`로 식별한다. 그리고 이 식별자는 DB의 기본키(Primary Key)와 맵핑되어 있다. 때문에 Entity는 `@Id`식별자 없이는 유효하지 않다.

> **조회**
```java
member.setId(1);
em.persist(member);
```
위 코드를 실행하면 1차 캐시에 `@Id`가 `1`인 엔티티가 1차 캐시에 저장되고 영속상태가 된다.  
1차 캐시에 저장된 값이 조회하려는 값이면 DB에 접근할 필요 없이 값을 1차캐시에서 가져올 수 있는 성능적 이점이 있다. 또한 같은 값을 복수 조회했을때 같은 인스턴스를 반환하여 동일성(==가 참임)을 보장한다.


> **엔티티 등록**

엔티티 매니저는 커밋(`commit()`)직전까지 Insert SQL을 쓰기지연 SQL 저장소에 모아둔다 그리고 커밋할때 DB에 모두 보내는데 이를 **쓰기 지연(Transactional write-behind)라고**한다. 

[쓰기지연 과정]
![image](/assets/image.png)

이렇게 쓰기지연 저장소에 있던 Insert SQL들이 `commit()`이 실행되면 DB로 보내지고 그 후에 커밋된다.

**엔티티 수정**

(SQL의 문제점)
SQL 수정 쿼리는 매번 수정할때마다 SQL을 날려야하기에 쿼리가 많아지고 비즈니스 로직을 분석하기 위해 SQL을 계속 확인해야 한다. 이는 비즈니스 로직이 SQL에 의존적일 수 밖에 없다.

엔티티를 영속성 컨택스트에 보관할때 영속성 컨택스트의 내부에는 엔티티 뿐만 아니라 최초 상태를 복사해서 저장하는 스냅샷이 있다. 그리고 `flush()`시점에 스냅샷과 엔티티를 비교해서 변경된 엔티티를 찾는다. 이를 `변경감지(dirty checking)`라고 한다. 

<blockquote class="prompt-tip">
`궁금한점` DB에 저장된 데이터는 변경감지가 안되는 것인가?   
</blockquote>


> **데이터를 변경할때 결국 우리는 해당 엔티티를 조회해야한다. 
그때 영속성 컨택스트에 엔티티가 영속되고 같은 트랜잭션 내에서 수정한다면 변경감지가 가능하다.**



[변경감지 상세과정]
1. 트랜재견을 커밋하면 엔티티 매니저 내부에서 먼저 `flush()`가 호출된다. 
2. 엔티티와 스냅샷을 비교해서 변경된 엔티티를 찾는다.
3. 변경된 엔티티가 있으면 수정 쿼리를 생성해서 쓰기 지연인 SQL저장소에 보낸다. 
4. 쓰기 지연 저장소의 SQL을 데이터베이스에 보낸다. 
5. 데이터베이스 트랜잭션을 커밋한다. 

<blockquote class="prompt-info">
변경감지는 영속성 컨택스트가 영속 상태의 엔티티에만 적용된다.
</blockquote>

[변경감지시 실제 Update SQL 동작]
변경 감지시에 해당 엔티티의 수정된 부분만 `Update`하는 것이 아니라 모든 컬럼에대해 적용된다. 전송량이 증가하는 단점을 안고 가더라도 얻는 이점은 다음과 같다.
 - 모든 필드를 사용하면 수정쿼리가 항상 같다
 - 데이터베이스에 동일한 쿼리를 보내면 DB는 이전에 한 번 파싱된 쿼리를 재사용 할 수 있다.

 ***하지만 컬럼수가 (평균 30개이상)많고 양이 많아지면 동적으로 하는 방법도 있다***  
 **`@DynamicUpdate` 참고**

 **엔티티 삭제**
```java
    em.remove(memberA);
```
이렇게 `remove()`를 호출하면 영속성 컨택스트에서 엔티티를 삭제하고 삭제쿼리를 쓰기지연 SQL 저장소에 등록하고 `commit()`을 호출하면 `flush()`를 호출하면서 실제 DB에 반영한다. 