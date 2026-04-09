---
layout: post
title: (MagicOfConch) FCM 알림 배치 100만건 최적화기
date: 2026-04-06 23:00
category: [Magicofconch, Troubleshooting]
author: greensnapback0229
tags: [Spring Batch, RabbitMQ, Keyset Paging, Performance]
summary: Streak 알림 배치를 100만건까지 견디게 만드는 과정에 대한 글입니다.
image_dir: /assets/posts/streak-batch
---

# (MagicOfConch) FCM 알림 배치 100만건 최적화기(1)

<br>

> MagicOfConch(마법의 소라고동) 프로젝트의 Streak 알림 배치를 단계적으로 개선한 기록입니다.
> Stack: Spring Batch, JPA, RabbitMQ, Prometheus, Grafana
> 1분에 100만 명에게 알림을 보내야 한다면 이 배치는 살아남을 수 있을까? 라는 질문에서 시작합니다.



## 어떤 기능인가?

마법의 소라고동은 "회고 습관 형성 서비스"인 만큼 사용자가 설정한 시간에 회고 작성을 유도하는 Push 알림을 전송하는 기능이 필요합니다. 



구현 기준은 다음과 같았습니다. 



사용자가 "매일 저녁 9시에 알려줘" 라고 설정하면, 그 시간에 맞춰서 FCM으로 푸시를 보내주는 것입니다. 메시지는 사용자의 회고 작성 이력에 따라 달라집니다.



- 회고 0건: "오늘부터 회고 습관을 만들어볼까요?"
- 회고 있지만 스트릭 0: "3월 5일 첫 회고를 작성했어요. 오늘도 한 줄 적어볼까요?"
- 스트릭 1~3: "이제 곧 회고 시간이에요. 개발자에서 카페가 되어볼까요?"
- 스트릭 4 이상: "오늘 한문장 쓰면 연속 5일차, 개발자에 한걸음 더 가까워져요"

이 기능을 Spring Batch로 구현했습니다. 배치 Job이 1분 단위로 실행되면서, 해당 분에 알람이 설정된 사용자만 골라서 FCM 메시지를 만든 후 발송합니다.

처음 개발할 때는 사용자가 많지 않았기 때문에 단순한 구조로 빠르게 만들었습니다. 그런데 만약 사용자가 100만 명이 넘는 시점이 오면 이 배치가 견딜 수 있을지 궁금했습니다.

그래서 100만 건의 더미 데이터를 넣고 직접 부하를 줘봤습니다.



## 측정 환경

집에 있는 미니 PC에서 Docker로 환경을 구성했습니다.

| 항목 | 사양 |
|---|---|
| CPU | 4C/4T |
| RAM | 16GB |
| sora-admin 컨테이너 | 2 CPU / 2GB |
| MySQL 컨테이너 | 1~2 CPU / 2~3GB |
| RabbitMQ | 0.5 CPU / 1GB |
| Worker | 0.5 CPU / 512MB |
| 모니터링 | Prometheus + Grafana (호스트) |

테스트 데이터는 100만 명의 user, 100만 개의 streak_info, 약 1250만 개의 review로 구성했습니다. 사용자당 0~50개의 review를 랜덤하게 분포시켰고, `streak_info.review_at`은 70%를 저녁 시간대(18~22시)에 몰아 두었습니다.



## 1. 기존 구현 - JpaCursorItemReader

처음 작성한 Reader는 다음과 같습니다.

```java
@Bean
@StepScope
public JpaCursorItemReader<UserInfo> streakReader(
    EntityManagerFactory entityManagerFactory,
    @Value("#{jobExecutionContext['alertTime']}") LocalDateTime alertTime) {
    return new JpaCursorItemReaderBuilder<UserInfo>()
        .name("streakUserInfoReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT u FROM UserInfo u "
            + "LEFT JOIN FETCH u.osAuthInfo oi "
            + "LEFT JOIN FETCH u.streakInfo si "
            + "WHERE FUNCTION('HOUR', si.reviewAt) = :hour "
            + "AND FUNCTION('MINUTE', si.reviewAt) = :minute "
            + "ORDER BY u.id")
        .build();
}
```

`UserInfo`와 연관 엔티티를 한 번에 가져오고, Processor에서 메시지를 분기, Writer에서 FCM을 발송합니다. 단순한 chunk 기반 배치입니다.

이 상태에서 19시 15분에 4만 5천여 명을 대상으로 돌렸을 때 결과는 다음과 같습니다.

```text
Job: [streakAlertJob] completed in 1m6s716ms
```

![streak_batch_baseline]({{ page.image_dir }}/streak_batch_baseline.png)

44,945건에 약 1분 7초. 처음에는 "그래도 1분이면 괜찮네" 라고 생각했지만, 자세히 들여다보니 이상한 게 있었습니다.



## 2. N+1 발견과 해결

Hibernate statistics를 켜두고 메트릭을 찍어봤더니 이상한 값이 보였습니다.

```text
spring_data_repository_invocations_seconds_count{method="countByUserInfo"} = 44,945
spring_data_repository_invocations_seconds_sum                              = 35.01초
```

`countByUserInfo`가 **44,945번** 호출되고, 그 호출이 차지한 누적 시간이 **35초** 였습니다. 전체 배치 시간 1분 6초 중 67%가 이 메서드 호출이었습니다.

원인은 Processor 코드였습니다.

```java
// 회고 횟수 분기를 위해서 유저당 1번씩 count 쿼리 발생
int count = reviewRepository.countByUserInfo(userInfo);

if (count <= 0) {
    body = zeroReview;
} else if (streakInfo.getReviewStreakCount() == 0) {
    body = String.format(streakZeroReviewNonZero, ...);
} ...
```

Processor가 실행될 때마다 review 테이블을 한 번씩 더 조회하고 있었습니다. 흔한 N+1 문제입니다.

> **N+1 이란?**
> 1개의 메인 쿼리(N개의 row를 가져오는 쿼리)를 실행한 뒤, 각 row마다 추가 쿼리가 N번 발생하는 패턴입니다.
> JPA에서는 보통 lazy loading된 연관 관계나, Processor에서 row마다 Repository를 호출할 때 발생합니다.
> 쿼리 1번이면 끝날 일을 N+1번 호출하기 때문에 데이터가 많아질수록 치명적입니다.

해결은 단순합니다. count를 Processor에서 빼고 Reader 쿼리의 서브쿼리로 옮기면 됩니다.

```java
.queryString("SELECT u, "
    + "(SELECT MIN(r.reviewDate) FROM Review r WHERE r.userInfo = u), "
    + "(SELECT COUNT(r) FROM Review r WHERE r.userInfo = u) "
    + "FROM UserInfo u "
    + "LEFT JOIN FETCH u.osAuthInfo oi "
    + "LEFT JOIN FETCH u.streakInfo si "
    + "WHERE si.reviewAt >= :start "
    + "AND si.reviewAt < :end "
    + "ORDER BY u.id")
```

JPA에서 `SELECT u, ...` 처럼 여러 컬럼을 조회하면 `Object[]`로 결과가 반환되기 때문에, Reader에서 이를 받아 DTO로 매핑해줬습니다.

같은 4만 5천 건을 다시 돌려본 결과는 다음과 같습니다.

```text
Job: [streakAlertJob] completed in 22s489ms
```

**1분 6초 → 22초**, 약 65% 단축. 단순히 쿼리 1개를 줄였을 뿐인데 효과가 컸습니다.



## 3. 진짜 부하 - 100만 건

여기까지는 베이스라인이고, 진짜 궁금한 건 100만 건이었습니다. `UPDATE streak_info SET review_at = '22:00:00'` 으로 모든 사용자의 알람 시간을 한 시점에 몰아넣고 다시 돌려봤습니다.

결과는 OOM 이었습니다.

```text
Caused by: java.lang.OutOfMemoryError: Java heap space
    at com.mysql.cj.protocol.a.SimplePacketReader.readMessageLocal(...)
    ...
```

![streak_batch_memory_oom]({{ page.image_dir }}/streak_batch_memory_oom.png)

![streak_batch_db_connection]({{ page.image_dir }}/streak_batch_db_connection.png)

Grafana를 보니 배치가 돌아가는 내내 **DB 커넥션이 반납되지 않고 계속 점유**되고 있었습니다. Chunk 기반 배치인데 왜 메모리가 터질까? `JpaCursorItemReader`의 동작 원리를 다시 짚어볼 필요가 있었습니다.

### JpaCursorItemReader 동작 원리

`JpaCursorItemReader`는 이름 그대로 **DB의 커서를 열어두고 한 행씩 스트리밍**하는 Reader입니다.

- 배치가 시작되면 EntityManager를 하나 생성합니다.
- DB에 커서를 열고 첫 번째 행부터 한 행씩 받아옵니다.
- chunk 사이즈만큼 모이면 Processor → Writer → flush + commit.
- 다시 다음 행을 받아옵니다.
- 모든 행을 다 읽을 때까지 같은 EntityManager(=같은 영속성 컨텍스트)를 유지합니다.

여기서 함정이 있습니다. Spring Batch는 chunk 끝에서 **flush와 commit은 하지만 clear는 하지 않습니다**.

> **flush vs clear vs close**
> - `flush()` : 변경사항(dirty)을 DB에 반영합니다. 엔티티는 영속성 컨텍스트에 그대로 남아 있습니다.
> - `clear()` : 영속성 컨텍스트에서 모든 엔티티를 detach 시킵니다. 메모리에서 GC 대상이 됩니다.
> - `close()` : EntityManager 자체를 닫습니다.
>
> flush와 clear는 다릅니다. flush 만 한다고 메모리가 비워지지 않습니다.

즉 chunk마다 flush + commit이 일어나도, 읽어들인 엔티티는 계속 영속성 컨텍스트에 쌓입니다.

```text
chunk 1:    UserInfo 50개 누적
chunk 100:  5,000개 누적
chunk 1000: 50,000개 누적
...
chunk 20000: 1,000,000개 → OOM
```

연관 엔티티(`StreakInfo`, `OsAuthInfo`)까지 합치면 100만 건은 약 300만 개의 엔티티가 메모리에 상주합니다. 컨테이너 메모리 1GB 안에 들어갈 리가 없죠.

### 그러면 chunk 끝에 clear() 하면 되지 않나?

안 됩니다. **커서가 EntityManager에 의존하고 있기 때문**입니다.

```text
clear() 호출 → 영속성 컨텍스트가 비워짐
              → 커서가 참조하던 상태도 같이 사라짐
              → 다음 read() 시 NPE
```

이게 Cursor 방식의 구조적 딜레마입니다.

```text
커서를 유지하려면 → EntityManager를 열어둬야 함
메모리를 줄이려면 → clear()로 비워야 함
→ 둘을 동시에 할 수 없음
```

### Cursor의 한계 정리

이번 OOM을 계기로 정리한 `JpaCursorItemReader`의 단점은 다음과 같습니다.

1. **영속성 컨텍스트 누적으로 인한 OOM** - 100만 건이면 100만 + α 개의 엔티티가 쌓입니다.
2. **DB 커넥션 장시간 점유** - 커서를 유지하기 위해 배치 전체 시간 동안 커넥션 1개를 잡고 있습니다. 실측으로 `hikaricp_connections_usage_seconds_max = 50.4초` 가 찍혔습니다.
3. **Dirty Checking 비용 증가** - flush 시 영속성 컨텍스트의 모든 엔티티를 검사합니다. 뒤로 갈수록 검사 대상이 많아져서 느려집니다.
4. **Partition 확장 불가** - 여러 스레드가 각자 커서를 들고 있으면 커넥션이 그만큼 잡힙니다. HikariCP가 빠르게 고갈됩니다.

이 흐름을 그림으로 보면 한 눈에 들어옵니다.

![cursor-problem]({{ page.image_dir }}/cursor-problem.png)

배치가 시작되는 순간 HikariCP에서 커넥션을 받아오고, 그 커넥션 위에서 EntityManager 하나가 열립니다. 이후 `DECLARE CURSOR` 가 한 번 실행되면 chunk마다 데이터를 fetch 하는데, **그 사이에 EntityManager는 절대 닫히지 않습니다**. 그 결과 chunk 1에서 150개, chunk 100에서 15,000개, chunk 899에서 134,835개가 영속성 컨텍스트에 그대로 쌓이게 됩니다. 100만 건으로 가면 약 300만 개의 엔티티가 메모리에 상주하게 되고, 컨테이너 힙이 1GB 수준이라면 그대로 OOM 으로 떨어집니다.

여기서 흥미로운 건 빨간색으로 표시한 구간 전체 동안 HikariCP의 커넥션 1개가 계속 점유 상태로 머문다는 점입니다. 짧은 트랜잭션 여러 번이 아니라 **한 번 잡고 안 놓는** 구조입니다. 이게 partition 확장을 막는 결정적인 이유였습니다. 같은 구조로 4개 파티션을 띄우면 커넥션 4개가 똑같이 묶입니다.

이 한계 때문에 다른 방식이 필요했습니다.



## 4. Keyset Paging Reader 직접 구현

### Keyset Paging 이란?

페이지를 나누는 두 가지 방식이 있습니다.

**OFFSET 방식**

```sql
-- 1페이지
SELECT ... LIMIT 1000 OFFSET 0

-- 100페이지
SELECT ... LIMIT 1000 OFFSET 99000
```

OFFSET 방식은 DB가 매번 처음부터 데이터를 세서 99,000개를 건너뛴 뒤에 다음 1,000개를 가져옵니다. 뒤로 갈수록 비용이 점점 커집니다.

**Keyset 방식**

```sql
-- 1페이지
SELECT ... WHERE id > 0     ORDER BY id LIMIT 1000

-- 2페이지
SELECT ... WHERE id > 1000  ORDER BY id LIMIT 1000

-- 100페이지
SELECT ... WHERE id > 99000 ORDER BY id LIMIT 1000
```

Keyset은 직전 페이지의 마지막 id를 기억해 두었다가, 다음 페이지에서 `id > lastId` 조건으로 인덱스를 바로 점프합니다. 몇 페이지를 가도 비용이 일정합니다.

추가로 우리에게 더 중요한 효과가 있습니다. **각 페이지마다 EntityManager를 새로 열고 닫는 구조로 만들 수 있다**는 점입니다. 그러면 Cursor의 영속성 컨텍스트 누적 문제가 자연스럽게 해결됩니다.

### 직접 구현이 필요했던 이유

Spring Data JPA의 Repository 추상화로는 **EntityManager의 생명주기를 직접 제어할 수 없습니다**. Spring이 알아서 관리해주기 때문에 페이지마다 EntityManager를 열고 닫는 식의 제어가 불가능합니다.

QueryDSL을 써도 마찬가지입니다. QueryDSL은 결국 EntityManager 위에서 동작하기 때문에, EntityManager 생명주기 관리는 우리가 직접 해야 합니다.

그래서 `EntityManagerFactory`를 주입받아서 페이지마다 직접 EntityManager를 생성/소멸시키는 Reader를 만들었습니다.

```java
private List<UserStreakItem> fetchNextPage() {
    EntityManager em = entityManagerFactory.createEntityManager();
    try {
        // 1단계: ID만 조회
        List<Long> ids = em.createQuery(ID_JPQL, Long.class)
            .setParameter("start", start)
            .setParameter("end", end)
            .setParameter("lastId", lastId)
            .setParameter("maxId", maxId)
            .setMaxResults(PAGE_SIZE)
            .getResultList();

        if (ids.isEmpty()) return List.of();

        // 2단계: ID 목록으로 엔티티 + 연관관계 fetch
        List<Object[]> results = em.createQuery(FETCH_JPQL, Object[].class)
            .setParameter("ids", ids)
            .getResultList();

        return results.stream()
            .map(row -> {
                UserInfo userInfo = (UserInfo) row[0];
                LocalDate firstReviewDate = (LocalDate) row[1];
                Long reviewCount = (Long) row[2];
                lastId = userInfo.getId();
                return new UserStreakItem(userInfo, firstReviewDate, reviewCount);
            })
            .toList();
    } finally {
        em.close();
    }
}
```

페이지마다 새 EntityManager를 만들고, finally에서 닫습니다. 닫히는 순간 영속성 컨텍스트도 같이 비워지기 때문에 GC가 회수해갈 수 있습니다.

### LEFT JOIN FETCH + LIMIT 의 함정

여기서 한 번 막혔습니다. 처음에는 단일 쿼리로 작성했는데 이런 경고가 떴습니다.

```text
HHH90003004: firstResult/maxResults specified with collection fetch;
applying in memory
```

`LEFT JOIN FETCH u.osAuthInfo` 처럼 **컬렉션(OneToMany)에 대한 fetch join + LIMIT** 조합이 문제였습니다.

OneToMany는 결과가 행 단위로 펼쳐집니다.

```text
user_id=1, osAuthInfo_id=1   ← UserInfo 1개인데 2행으로 펼쳐짐
user_id=1, osAuthInfo_id=2
user_id=2, osAuthInfo_id=3
user_id=3, osAuthInfo_id=4
```

여기에 `LIMIT 1000`을 걸면 행 1,000개가 잘릴 뿐 엔티티 1,000개가 잘리는 것이 아닙니다. 그래서 Hibernate는 **LIMIT을 DB로 보내지 않고, 결과를 전부 가져온 뒤 메모리에서 자릅니다.**

```text
우리 의도:  DB에서 1,000건만 가져와
Hibernate: 100만건 다 가져온 후 → 메모리에서 1,000개 추출 → OOM
```

해결 방법은 2단계 쿼리입니다.

| | 1단계 | 2단계 |
|---|---|---|
| 컬렉션 JOIN FETCH | 없음 | 있음 |
| LIMIT | 있음 | 없음 |
| 쿼리 | `SELECT u.id ...` | `SELECT u ... WHERE u.id IN (:ids)` |

1단계에서는 컬렉션 fetch 없이 ID만 LIMIT으로 가져오고, 2단계에서는 LIMIT 없이 ID 목록으로 fetch join을 합니다. 이러면 둘 다 정상 동작합니다.

### 결과

100만 건으로 다시 돌렸습니다.

```text
Job: [streakAlertJob] completed in 1m7s003ms
```

![streak_batch_keyset_result]({{ page.image_dir }}/streak_batch_keyset_result.png)

OOM 없이 1분 7초로 완주했습니다. 같은 구조를 그림으로 비교해보면 차이가 분명합니다.

![keyset-solution]({{ page.image_dir }}/keyset-solution.png)

매 chunk 마다 새 EntityManager 를 열고, ID 1,000개를 받아 처리한 뒤, **flush + clear + close 를 거쳐 닫아버립니다**. 닫히는 순간 영속성 컨텍스트도 같이 사라지기 때문에 GC가 회수해갈 수 있고, HikariCP 커넥션도 즉시 풀로 반환됩니다. 1,000개를 처리하든 100만 개를 처리하든 어느 시점이든 메모리에 올라와 있는 엔티티 수는 거의 일정하게 유지됩니다.

Cursor 방식과 비교해서 두 가지가 핵심적으로 달라졌습니다.

| | Cursor | Keyset |
|---|---|---|
| EntityManager 수명 | 배치 전체 | chunk 단위 |
| 커넥션 점유 시간 | 50초 (배치 전체) | 0.5초 미만 (chunk 단위) |
| 영속성 컨텍스트 | 누적 | 매번 초기화 |
| GC 가능 시점 | 배치 종료 후 | chunk 종료 직후 |
| Partition 확장 | 불가 (커넥션 점유) | 가능 (짧게 잡았다 놓음) |

> "Spring Data JPA의 Repository 추상화로는 EntityManager 생명주기를 직접 제어할 수 없어서, JPA EntityManager를 직접 사용해 페이지마다 영속성 컨텍스트를 초기화하는 Keyset Reader를 구현했다."

이렇게 정리할 수 있게 되었습니다.



## 5. 더 큰 벽 - 외부 I/O

여기까지는 사실 "DB에서 읽기" 만 측정한 것입니다. 실제 배치는 마지막에 **FCM에 푸시 메시지를 보내야** 합니다. 그 시간을 어떻게 측정해야 할지 고민했습니다.

Firebase 공식 문서를 찾아봤습니다. ([FCM Status Dashboard](https://status.firebase.google.com/cloud-messaging/))

> "Latency represents the time to respond to a request to send a message and does **not** measure the latency to deliver the message to the device."
> p95 < 350 ms (30일 롤링 SLO)

FCM HTTP API의 응답 시간이 p95 기준 350ms 이하라는 SLO 입니다. 디바이스 도달 시간이 아니라, "메시지 접수했다" 응답까지의 시간입니다.

직접 서버에서 FCM API를 호출해봤습니다.

```bash
time curl -s -o /dev/null -w "HTTP %{http_code}, Time: %{time_total}s\n" \
  -X POST "https://fcm.googleapis.com/v1/projects/.../messages:send" \
  -H "Authorization: Bearer $FCM_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":{"token":"INVALID","notification":{"title":"latency","body":"test"}}}'
```

```text
real    0m0.203s
```

![streak_batch_fcm_latency]({{ page.image_dir }}/streak_batch_fcm_latency.png)

약 **203ms**. 공식 SLO 안쪽입니다.

이걸 100만 건에 곱하면 어떻게 될까요?

```text
1,000,000 × 200ms = 약 56시간
```

배치에서 직접 FCM을 호출하는 구조는 이 시점에 사형선고를 받았습니다.

`Thread.sleep(200)`으로 시뮬레이션할까 하다가, 노트북에 webflux 서버를 띄우고 Tailscale로 미니 PC에서 호출하도록 해서 진짜 네트워크 I/O를 측정해봤습니다.

Writer를 RestTemplate 호출로 바꾸고 chunk 1,000건짜리 배치를 돌렸을 때 chunk 하나가 약 50초 걸렸습니다. 100만건 환산하면 약 14시간. Thread.sleep 추정치와 비슷한 수준입니다.

![streak_batch_io_bottleneck]({{ page.image_dir }}/streak_batch_io_bottleneck.png)



## 6. 메시지 큐 도입 - RabbitMQ

### 왜 큐 분리인가

배치에서 하는 일을 다시 보면, 한 행마다 이런 흐름입니다.

```text
Reader (DB read) → Processor (메시지 분기) → Writer (FCM 발송)
```

Reader와 Processor는 빠릅니다. Writer가 외부 API를 호출하면서 200ms씩 블로킹되는 것이 전부입니다. 이걸 한 스레드에서 직렬로 하니까 14시간이 나오는 것입니다.

> **메시지 큐 패턴**
> Producer가 메시지를 큐에 던지고, Consumer가 큐에서 꺼내서 처리하는 구조입니다.
> Producer와 Consumer가 분리되어 있어서, Producer는 큐에 넣자마자 손을 떼고 다음 일을 합니다.
> Consumer가 느리더라도 Producer는 영향을 받지 않습니다.
> 외부 I/O가 들어가는 작업을 비동기로 분리할 때 가장 흔히 쓰는 패턴입니다.

배치를 Producer로, 발송 워커를 Consumer로 두면 됩니다. 배치는 메시지를 큐에 넣는 것까지만 책임지고, 실제 FCM 발송은 Worker가 비동기로 처리합니다.

```text
[배치]              [큐]                [Worker]
Reader →            ─────→              ─────→
Processor →    publish     consume        FCM 발송
Writer →            ─────→              ─────→
```

배치 입장에서는 외부 I/O를 0초로 보는 셈입니다.

### 구조

RabbitMQ 컨테이너를 띄우고, batch-worker 라는 별도 모듈을 만들었습니다.

| 컴포넌트 | 역할 |
|---|---|
| `notification-exchange` | direct exchange |
| `notification-queue` | 메인 큐 |
| `notification-queue.dlq` | retry 3회 실패 시 이동 |
| `notification.streak` | routing key |

소라어드민(Producer)에서는 Writer가 RabbitMQ에 publish 합니다.

```java
@Component
@RequiredArgsConstructor
public class StreakWriter implements ItemWriter<FcmRequest> {

    private final RabbitTemplate rabbitTemplate;

    @Value("${rabbitmq.notification.exchange}")
    private String exchange;

    @Value("${rabbitmq.notification.routing-key}")
    private String routingKey;

    @Override
    public void write(Chunk<? extends FcmRequest> chunk) throws Exception {
        for (FcmRequest request : chunk) {
            rabbitTemplate.convertAndSend(exchange, routingKey, request);
        }
    }
}
```

배치 워커 모듈에서는 `@RabbitListener`로 큐를 구독합니다.

```java
@RabbitListener(queues = "${rabbitmq.notification.queue}")
public void consumeAndSendFcm(FcmRequest request) {
    fcmUtil.send(request);
}
```

100만 건을 다시 돌렸을 때 배치가 publish를 끝내는 시간은 **약 2분** 이었습니다. Reader + Processor 시간이 거의 그대로 찍힌 셈입니다.

![streak_batch_rabbitmq_queue]({{ page.image_dir }}/streak_batch_rabbitmq_queue.png)

배치는 빨리 끝났지만, 실제 발송 책임은 Worker가 가져갑니다. 이제 Worker의 처리 속도를 어떻게 올릴지 가 다음 문제입니다.



## 7. Worker concurrency 튜닝

Worker가 단일 스레드로 메시지를 하나씩 가져와서 처리하면 결국 56시간이 됩니다. RabbitMQ Listener의 concurrency를 올려서 동시에 여러 건을 처리하게 만들어야 합니다.

> **prefetch 와 concurrency**
> - **prefetch** : Consumer가 한 번에 큐에서 가져오는 메시지 수입니다. 50이면 한 번에 50개를 받아 자기 메모리에 두고 처리합니다.
> - **concurrency** : Listener의 동시 처리 스레드 수입니다. 5로 설정하면 5개 스레드가 병렬로 메시지를 처리합니다.
>
> Worker의 작업이 CPU 연산이 아니라 **네트워크 I/O 대기**라면, CPU 코어 수보다 스레드를 더 올려도 효과가 있습니다. I/O 대기 중에는 스레드가 CPU를 안 쓰니까요.

### 처음엔 적용이 안 됐던 이슈

`application.yml`에서 concurrency 값을 설정했는데도 Worker는 스레드 1개로만 동작했습니다.

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 10
        max-concurrency: 15
```

원인은 `SimpleRabbitListenerContainerFactory`를 직접 Bean으로 정의해 두었기 때문이었습니다. Spring Boot의 자동 설정이 사용자 정의 Factory가 있으면 비활성화되면서 application.yml 설정이 무시되는 것이었습니다.

Spring AMQP 공식 문서를 찾아봤습니다.

> "If a `MessageConverter` bean is defined, it is associated automatically to the auto-configured `AmqpTemplate`."

Factory를 직접 만들 필요가 없었습니다. **`MessageConverter` Bean 하나만 정의해두면 Spring Boot가 자동으로 연결**해줍니다. JSON 역직렬화가 필요해서 Factory를 만들었던 것이었는데, 그건 MessageConverter Bean만으로 충분했던 것이었습니다.

Factory를 삭제하고 Bean을 정리했습니다.

```java
@Bean
public MessageConverter jsonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

이제 application.yml의 concurrency, prefetch, retry 설정이 자동으로 반영됩니다.

### 측정

같은 방식으로 1만 건짜리 데이터를 두고 concurrency 값만 바꿔가면서 측정했습니다. (Worker는 0.5 CPU / 512MB 컨테이너)

| concurrency | 소요 시간 | rate (msg/s) | 개선율 |
|---:|---:|---:|---:|
| 1 | 367초 | 27 | baseline |
| 5 | 95초 | 105 | 3.9배 |
| 7 | 79초 | 126 | 4.7배 |
| 10 | 65초 | 153 | 5.7배 |

![streak_batch_concurrency_10]({{ page.image_dir }}/streak_batch_concurrency_10.png)

스레드를 늘릴수록 빨라지지만 수확 체감(diminishing returns)이 보입니다.

```text
1 → 5  : +78 msg/s (스레드당 +15.6)
5 → 7  : +21 msg/s (스레드당 +10.5)
7 → 10 : +27 msg/s (스레드당 +9.0)
```

CPU 0.5코어인데도 concurrency 10에서 단일 스레드 대비 5.7배 빨라졌습니다. 작업이 I/O bound인 덕분입니다. 그 이상은 컨텍스트 스위칭 비용과 webflux 수신 서버 처리량이 따라주지 않아서 효과가 줄어듭니다.

10으로 고정했습니다.



## 8. 배치도 병렬로 - Spring Batch Partitioning

Worker는 빨라졌지만, 아직 배치 자체(=Producer)는 단일 스레드입니다. 100만 건 publish가 약 2분 걸리는데 이걸 더 줄이고 싶었습니다.

> **Spring Batch Partitioning 이란?**
> 하나의 Step을 N개의 Worker Step으로 쪼개어 병렬 실행하는 구조입니다.
> Manager Step이 Partitioner를 통해 데이터 범위를 N등분 한 뒤, TaskExecutor로 각 Worker Step을 동시에 실행합니다.
> 각 Worker Step은 자기 범위만 Reader → Processor → Writer 합니다.

```text
                    Manager Step
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
        Worker Step  Worker Step  Worker Step
        (id 1~250K) (250K~500K) (500K~750K)
```

Partitioner에서 user_id의 min/max를 구해서 N등분 했습니다.

```java
public class StreakPartitioner implements Partitioner {

    private final EntityManagerFactory entityManagerFactory;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long minId = ...;
        Long maxId = ...;
        long range = (maxId - minId) / gridSize + 1;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            context.putLong("minId", minId + (range * i));
            context.putLong("maxId", ...);
            partitions.put("partition" + i, context);
        }
        return partitions;
    }
}
```

Reader에서는 `stepExecutionContext`에서 minId, maxId를 받아서 자기 범위만 읽도록 했습니다.

### 효과가 없었다

기대했던 것과 다르게 파티션 1개와 3개의 시간이 거의 같았습니다.

```text
Partition 1 : 1m 49s
Partition 3 : 1m 41s
Partition 5 : 1m 41s
```

![streak_batch_partition_3]({{ page.image_dir }}/streak_batch_partition_3.png)

3개 이상 늘려도 거의 줄지 않았습니다. 왜일까?

### 첫 번째 원인 - RabbitMQ 커넥션 공유

Spring AMQP의 기본 `CachingConnectionFactory`는 **커넥션 1개를 공유하고 채널만 캐싱**하는 구조입니다.

> **AMQP connection vs channel**
> - **Connection** : RabbitMQ 브로커와의 TCP 연결입니다.
> - **Channel** : 하나의 connection 위에서 다중화되는 가상 통신 경로입니다.
>
> 여러 채널을 만들면 같은 TCP 연결을 공유하게 됩니다. 동시 publish가 많을 때는 채널이 아무리 많아도 결국 같은 TCP 소켓을 직렬로 통과해야 합니다.

```text
파티션 Thread 1 ─┐
파티션 Thread 2 ─┤→ Connection 1개 → RabbitMQ
파티션 Thread 3 ─┤   (채널만 여러 개)
파티션 Thread 4 ─┘
```

고속도로 1차선에 차 4대가 줄 서 있는 셈이었습니다.

`spring.rabbitmq.cache.connection.mode` 를 `connection` 으로 바꿔서 파티션 수만큼 별도 커넥션을 만들도록 했습니다.

```yaml
spring:
  rabbitmq:
    cache:
      connection:
        mode: connection
        size: ${BATCH_PARTITION_GRID_SIZE:1}
```

### 두 번째 원인 - 단일 MySQL CPU

Connection을 분리해도 시간은 거의 그대로였습니다. 파티션을 늘리면 chunk timing 로그에 가끔씩 1~5초 짜리 스파이크가 찍혔습니다.

```text
chunk=985, duration=701ms
chunk=974, duration=265ms
...
chunk=132, duration=5362ms  ← 5초 스파이크
```

![streak_batch_mysql_spike]({{ page.image_dir }}/streak_batch_mysql_spike.png)

원인은 MySQL이었습니다. 컨테이너 MySQL이 1 CPU 에 buffer pool 128MB 였고, 여러 파티션 스레드가 동시에 쿼리를 날리면서 CPU와 디스크 I/O 경합이 생겼습니다.

이 가설을 검증하려고 몇 가지를 시도했습니다.

- **MySQL 버퍼풀을 2GB로 증설** → 거의 차이 없음 (1m 37s → 1m 36s)
- **review 집계를 임시 테이블에 미리 저장** (서브쿼리 제거) → 거의 차이 없음 (1m 37s)

이 시점에 의문이 들었습니다. 정말 RabbitMQ publish가 문제인 걸까? 아니면 DB 읽기 자체가 한계인 걸까?

### 마지막 검증 - Writer 제거

Writer의 publish 코드를 통째로 주석 처리하고 Reader + Processor만 측정해봤습니다.

```java
@Override
public void write(Chunk<? extends FcmRequest> chunk) throws Exception {
    // for (FcmRequest request : chunk) {
    //     rabbitTemplate.convertAndSend(exchange, routingKey, request);
    // }
}
```

```text
Step: [workerStep] executed in 37s266ms
```

DB만 읽으면 37초인데, RabbitMQ에 publish까지 하면 1분 37초. 그 차이가 publish 비용입니다.

근데 결국 **DB를 그냥 다 읽는 것만 해도 37초**가 걸린다는 점에서 진짜 한계는 여기였습니다. 4 CPU 환경에서 단일 MySQL 인스턴스로 100만 건을 읽는 시간 자체가 이미 충분히 큰 것이었습니다.



## 9. 결론 - 환경의 한계

### 최종 아키텍처

이 모든 과정을 거쳐서 최종적으로 만들어진 구조는 다음과 같습니다.

![streak_batch_architecture]({{ page.image_dir }}/streak_batch_architecture.png)

배치(Producer)는 Keyset Paging으로 DB에서 읽어 RabbitMQ에 publish 까지만 책임지고, Worker(Consumer)가 큐에서 꺼내서 FCM 발송을 담당합니다. 배치는 파티셔닝으로 병렬화 가능하고, Worker는 concurrency 튜닝과 컨테이너 스케일 아웃으로 처리량을 늘릴 수 있습니다.

### 단계별 결과

각 단계의 결과를 정리하면 이렇습니다.

| 단계 | 대상 | 결과 |
|---|---|---|
| Cursor + N+1 | 44,945건 | 1분 6초 (baseline) |
| N+1 제거 (서브쿼리) | 44,945건 | 22초 |
| Cursor | 1,000,000건 | OOM |
| Keyset Paging | 1,000,000건 | 1분 7초 |
| Writer + 외부 I/O (가정) | 1,000,000건 | 약 14시간 |
| Queue 분리 (RabbitMQ) | 1,000,000건 | 약 2분 (publish 기준) |
| Worker concurrency 1 | 10,000건 | 367초 |
| Worker concurrency 10 | 10,000건 | 65초 |
| Partition 1 | 1,000,000건 | 1분 49초 |
| Partition 3 | 1,000,000건 | 1분 41초 |
| Partition 5 | 1,000,000건 | 1분 41초 |

코드 레벨에서 할 수 있는 것은 전부 한 것 같습니다.

- **N+1** : 서브쿼리로 해결
- **OOM** : Cursor → Keyset Paging
- **외부 I/O 병목** : Queue 분리
- **Worker 처리량** : concurrency 튜닝
- **Producer 처리량** : Partitioning + Connection 분리

마지막 한 단계, 단일 MySQL의 처리량을 넘어서는 부분만 남았습니다. 이건 코드가 아니라 **인프라 스케일업** 영역입니다. CPU를 더 주거나, Read Replica를 분리하거나, 클라우드 RDS처럼 충분한 리소스가 있는 환경으로 옮기는 것이 다음 단계입니다.



## 회고

이번 작업은 단순히 "배치를 빨리 만들었다" 는 결과가 아니라, **각 단계에서 병목이 어디인지를 측정으로 증명하는 과정**이었던 것 같습니다.

Cursor가 OOM이 났을 때도 "메모리를 늘려야지" 가 아니라 "왜 메모리에 쌓이는가?" 를 따라가서 영속성 컨텍스트와 커서의 구조적 딜레마까지 도달했습니다. Partition이 효과가 없었을 때도 "왜?" 를 계속 물어서 RabbitMQ 커넥션 공유 → 단일 MySQL 까지 내려갔습니다.

특히 마지막에 Writer를 제거하고 Reader만 측정해본 게 결정적이었던 것 같습니다. RabbitMQ 탓을 하다가 사실은 DB가 한계였다는 걸 측정 한 번으로 알게 됐으니까요. 측정 없이 추측만 했다면 며칠을 더 헤맸을지도 모릅니다.

또 한 가지 배운 게 있습니다. 처음엔 Spring AMQP의 `SimpleRabbitListenerContainerFactory`를 직접 Bean으로 만들었었는데, 그게 자동 설정을 가로채서 application.yml의 concurrency 설정이 무시되고 있었습니다. AI에게 물어봤을 때는 Factory에 `@Value`를 더 받으라고 했었는데, 직접 공식 문서를 찾아보니 `MessageConverter` Bean만 정의하면 자동으로 연결되는 거였습니다. Factory를 굳이 만들 필요가 없었던 것이었습니다. 처음으로 AI보다 공식 문서가 정답이었던 케이스였습니다 ㅋㅋ

지금은 4코어 미니 PC라서 한계가 있지만, 곧 데스크탑(5800X / 32GB)에 RDS급 환경을 흉내내서 다시 측정해볼 예정입니다. 그 때는 진짜로 100만 건을 1분 안에 끝낼 수 있을지 궁금합니다.



## 정리

- 코드 레벨 최적화는 N+1 → Cursor 한계 → I/O 병목 → 동시성 → 파티셔닝 순으로 풀 수 있습니다.
- 각 단계마다 **병목을 추측하지 말고 측정**해야 합니다. Prometheus + Grafana, 그리고 가끔은 Writer를 통째로 주석 처리하는 것 같은 단순한 실험이 가장 효과적입니다.
- Cursor 방식의 Reader는 영속성 컨텍스트 + 커서 의존이라는 구조적 딜레마 때문에 대용량에 부적합합니다. Keyset Paging이 정답이지만, JPA Repository로는 EntityManager 생명주기를 직접 제어할 수 없어서 Reader를 직접 만들어야 합니다.
- 외부 I/O가 들어가는 작업은 **배치에서 직접 호출하지 말고 큐로 분리**하는 게 정답입니다. 배치는 publish 까지만 책임지고, 발송은 Worker에게 맡깁니다.
- 분산 스레드/파티션 최적화는 그 다음 단계의 한계(브로커 커넥션, DB CPU)를 그대로 드러냅니다. 코드가 아니라 인프라가 답인 영역도 있다는 걸 인정해야 합니다.
