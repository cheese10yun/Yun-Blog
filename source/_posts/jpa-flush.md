---
layout: post
title: JPA 플러시 정리
catalog: true
header-img: 'https://i.imgur.com/avC1Xor.jpg'
tags:
  - JPA
date: 2020-01-29
subtitle:
---

> [자바 ORM 표준 JPA 프로그래밍](http://acornpub.co.kr/book/jpa-programmig)을 보고 플러시 관련 내요을 정리한 정리한 내용 입니다.

## 플러시 란?
JPA는 엔티티를 영속성 컨텍스트에서 관리합니다. 영속성 컨텍스트에 있는 내용을 데이터베이스에 반영하는 것을 플러시라고 합니다. 보통 트랜잭션을 커밋하면 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화(등록, 수정, 삭제) 작업을 진행하게 됩니다.


## 엔티티 등록
```java
EntityMaanger em  = emf.createEnttiyManager();
ENtityTranscation transaction = em.getTransaction();
// 엔티티 매니저는 데이터 변경 시 트랜잭션을 시작해야한다.

transaction.begin();

em.persist(memberA);
em.persist(memberB);

// 여기까지 Insert SQL을 데이터베이스에 보내지 않는다.

// Commit을 하는 순간 데이터베이스에 Insert SQL을 보낸다
transaction.commit();
```
엔티티 **매니저는 트랜잭션을 커밋하기 직전까지 데이터베이스에 엔티티를 저장하지 않고 내부 쿼리 저장소에 INSERT SQL을 모아둔다.** 그리고 트랜잭션을 커밋할 때 모아둔 쿼리를 데이터베이스에 보내느데 이것을 **트랜잭션을 지원하는 쓰기 지연** 이라 한다.

![](https://github.com/cheese10yun/TIL/raw/master/assets/jpa-insert-persistent.png)
회원 A를 영속화 했다. 영속성 컨텍스트는 1차 캐시에 회원 엔티티를 저장하면서 동시에 회원 엔티티 정보로 등록 쿼리를 만든다. 그리고 만들어진 등록 쿼리를 쓰기 지연 SQL 저장소에 보관한다.

![](https://github.com/cheese10yun/TIL/raw/master/assets/jpa-insert-persistent-2.png)
다음으로 회원 B를 영속화했다. 마찬가지로 회원 엔티티 정보로 등록 쿼리를 생성해서 쓰지 지연 SQL 저장소에 보관한다. 현재 쓰기 지연 SQL저장소 에는 등록 쿼리가 2건이 저장되어 있다.

![](https://github.com/cheese10yun/TIL/raw/master/assets/jpa-insert-persistent-3.png)

마지막으로 트랜잭션을 커밋했다. **트랜잭션을 커밋하면 엔티티 매니저는 우선 영속성 컨텍스트를 플러시한다. 플러시는 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화하는 작업인데 이때 등록, 수정, 삭제한 엔티티를 데이터베이스에 반영한다.**

즉, 쓰기 지연 SQL 저장소에 모인 쿼리를 데이터베이스에 보낸다. 이렇게 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화한 후에 실제 데이터베이스 트랜잭션을 커밋한다.
(flush가 먼저 동작하고 (데이터베이스에 동기화한 후에) 실제 데이터베이스 트랜잭션을 커밋한다.)

### 트랜잭션을 지원하는 쓰기 지연이 가능한 이유
```java
begin(); // 트랜잭션 시작

save(A);
save(B);
save(C);

commit(); // 트랜잭션 커밋
```
1. 데이터를 저장하는 즉시 등록 쿼리를 데이터베이스에 보낸다. 예제에서 save() 메서드를 호출할 때 마다 즉시 데이터베이스에 등록 쿼리를 보낸다. 그리고 마지막에 트랜잭션을 커밋한다.
2. 데이터를 저장하면 등록 쿼리를 **데이터베이스에 보내지 않고 메모리에 모아 둔다.** 그리고 **트랜잭션을 커밋할 때 모아둔 등록 쿼리를 데이터베이스에 보낸다.**

### 트랜잭션을 지원하는 쓰지 지연과 성능 최적화

#### 트랜잭션을 지원하는 쓰지 이연과 JDBC 배치
```java
insert(member1); // INSERT INTO ...
insert(member2); // INSERT INTO ...
insert(member3); // INSERT INTO ...
insert(member4); // INSERT INTO ...
insert(member5); // INSERT INTO ...

commit();
```
**네트워크 호출 한번은 단순한 메소드를 수만 번 호출하는 것보다 더 큰 비용이 든다.** 이 코드는 5번의 INSERT SQL과 1번의 커밋으로 총 6번 데이터 베이스와 통신한다. 이것을 최적화하라면 5번의 INSERT SQL을 모아서 한 번에 데이터베이스로 보내면 된다. **JDBC가 제공하는 SQL 배치 기능을 사용하면 SQL을 모아서 데이터베이스에 한 번에 보낼 수 있다.** 하지만 이 기능을 사용하라면 많은 코드를 수정해야한다. **JPA는 플러시 기능이 있이므로 SQL 배치 기능을 효과적으로 사용할 수 있다.**

**`hibernate.jdbc.batch_size` 속성의 값을 50으로 주면 최대 50건씩 모아서 SQL 배치를 실행한다. 하지만 SQL 배치는 같은 SQL일 때만 유효하다. 중간에 다른 처리가 들어가면 SQL 배치를 다시 시작한다.**
```java
em.persist(new Member()); // 1
em.persist(new Member()); // 2
em.persist(new Member()); // 3
em.persist(new Member()); // 4
em.persist(new Orders()); // 1-1, 다른 SQL이 추가 되었기 때문에  SQL 배치를 다시 시작 해야 한다
em.persist(new Member()); // 1
em.persist(new Member()); // 2
```
1,2,3,4를 모아서 하나의 SQL 배치를 실행하고 1-1를 한 번 실행하고 1,2을 모아서 실행한다. 따라서 총 3번의 SQL 배치를 실행한다.

모든 경우에 사용할 수 있는 것은 아니다. 엔티티가 영속 상태가 되려면 식별자가 꼭 필요하다. **그런데 IDENTITY 식별자 생성 전략은 엔티티를 데이터베이스에 저장해야 식별자를 구할 수 있으므로 em.persist()를 호출하는 즉시 INSERT SQL이 데이터베이스에 전달된다. 따라서 쓰지 지연을 활용한 성능 최적화를 할 수가 없다.**

#### 트랜잭션을 지원하는 쓰기 지연과 애플리케이션 확장성
**트랜잭션을 지원하는 쓰기 지연의 가장큰 장점은 데이터베이스 테이블 로우에 락이 걸리는 시간을 최소한다는 것이다. 이 기능은 트랜잭션을 커밋해서 영속성 컨텍스트를 플러시하기 전까지는 데이터베이스에 데이터를 등록, 수정, 삭제 하지 않는다. 따라서 커밋 전까지 데이터베이스 로우에 락을 걸지 않는다.**

```java
update(memberA); // UPDATE SQL Member A
비즈니스로직A(); // UPDATE SQL ...
비즈니스로직B(); // UPDATE SQL ...
commit();
```
**JPQL를 사용하지 않고 SQL 직접다루면 update(memberA)를 호출할 때 UPDATE SQL을 실행하면 데이터베이스 테이블 로우에 락을 건다. 이 락은 비즈니스 `로직A()`, `비즈니스 로직B()`를 모두 수행하고 `commit()`을 호출할 때까지 유지된다. 트랜잭션 격리 수준에 따라 다르지만 보통 많이 사용하는 커밋된 읽기 Read Committed 격리 수준이나 그 이상에는 데이터베이스에 현재 수정 중인 데이터(로우)를 수정하려는 다른 트랜잭션은 락이 풀릴 때까지 대기한다.**

**JPA는 커밋을 해야 플러시를 호출하고 데이터베이스에 수정 쿼리를 보낸다.** 예제에서 `commit()`을 호출할 때 UPDATE SQL을 실행하고 바로 데이터베이스 트랜잭션을 커밋한다. 쿼리를 보내고 보내고 바로 트랜잭션을 커밋하므로 결과적으로 데이터베이스에 락이 걸리는 시간을 최소화 한다. **이는 동시에 더 많은 트래잭션을 처리할 수 있다는 장점이 된다.**



## 엔티티 수정

### 변경 감지
```
EntityMaanger em  = emf.createEnttiyManager();
ENtityTranscation transaction = em.getTransaction();
transaction.begin(); // 트랜잭션 시작

// 영성속 텐티티 조회
Member memberA = em.find(Member.class, "memberA");

// 영속성 엔티티 데이터 수정

memberA.setUsername("hi");
memberA.setAge(10);

//em.update(member) 이런 코드가 있어야하지 않을까?

transaction.commit(); // 트랜잭션 커밋
```
**엔티티의 변경사항을 데이터베이스에 자동으로 반영하는 기능을 변경 감지(dirty checking) 이라 한다.**
![](https://github.com/cheese10yun/TIL/raw/master/assets/jpa-persistent-dirty-checking.png)
JPA는 엔티티를 영속성 컨텍스트에 보관할 때, **최초 상태를 복사해서 저장해두는데 이것을 스냅샷이리고 한다.** 그리고 플러시 시점에서 스냡샵과 엔티티를 비교해서 변경된 엔티티를 찾는다.

1. **트랜잭션을 커밋하면 엔티티 매니저 내부에서 먼저 플러시가 호출된다.**
2. 엔티티와 스냅샵을 비교해서 변경된 엔티티를 찾는다.
3. **변경된 엔티티가 있으면 수정 쿼리를 생성해서 쓰기 지연 SQL 저장소에 보낸다.**
4. 쓰기 지연 저장소의 SQL을 데이터베이스에 보낸다.
5. 데이터베이스 트랜잭션을 커밋한다.


### 읽기 전용 트랜잭션
스프링 프레임워크를 사용하면 트랜잭션을 읽기 전용 모드로 설정할 수 있다. `@Transactional(readOnly = true)` 옵션을 주면 스프링 프레임워크가 하이버네이트 세션의 플러시 모드를 MANUAL로 설정한다 **그렇게되면 강제로 플러시를 호출하지 않은 한 플러시가 일어나지 않는다.** 따라서 트랜잭션을 커밋해도 영속성 컨텍스트 플러시하지 않는다. 영속성 컨텍스트를 플러시하지 않으니 엔티티의 등록, 수정, 삭제는 당연히 동작하지 않는다. **플러시 할 때 일어나는 스냅샷비교와 같은 무거운 로직을 수행하지 않으므로 성능에 향상된다.**

**변경 감지는 영속성 컨텍스타 관리하는 영속 상태의 엔티티에만 적용된다.** 비영속, 준영속처럼 영속성 컨텍스트의 관리를받지 못하는 엔티티는 값을 변경해도 데이터베이스에 반영되지 않는다.

## 엔티티 삭제
```java
Member meberA = em.find(Member.class, "memberA"); // 삭제할 대상 엔티티 조회
em.remove(memberA); // 엔티티 삭제
```
**엔티티를 삭제하려면 먼저 삭제 대상 엔티티를 조회해야한다.** em.remove()에 삭제 대상 엔티티를 넘겨주면 엔티티를 삭제한다. 물론 엔티티를 즉시 삭제하는 것이 아니라 엔티티 등록과 비슷하게 삭제 쿼리를 쓰기 지연 데이터베이스에 삭제 쿼리를 전달한다.

## 영속성 컨텍스트를 플러시 하는 3 가지 방법
**플러시는 영속성 컨텍스의 변경 내용을 데이터베이스에 반영한다.** 플러시를 실행하면 구체적으로 다음과 같은 일이 일어난다

1. 변경 감자기 동작해서 영속성 컨텍스트에 있는 모든 엔티티를 스냅샷과 비교 해서 수정된 엔티티를 찾는다. 수정 엔티티는 수정 쿼리를 만들어 쓰기 지연 SQL 등록한다.
2. 쓰기 지연 SQL 의 저장소의 쿼리를 데이터베이스에 전송한다. (등록, 수정, 삭제 쿼리)

영속성 컨텍스트를 플러시 하는 방법은 3가지다.

1. em.flush()를 직접 호출한다.
2. JPQL 쿼리 실행 시 플러시가 자동 호출된다. 
3. 트랜잭션 커밋 시 플러시가 자동 호출된다.


### 플러시를 직접 호출하는 경우
엔티티 매니저의 flush() 메서드를 직접 호출해서 영속성 컨텍스트를 강제로 플러시 한다. **테스트나 다른 프레임워크와 JPA 함께 사용할 때는 제외하고 거의 사용하지 않는다.**

### 트랜잭션 커밋 시 플러시가 자동 호출
**데이터베이스에 변경 내용을 SQL로 전달하지 않고 트랜잭션만 커밋하면 어떤 데이터도 데이터베이스에 반영되지 않는다. 따라서 트랜잭션을 커밋하기 전에 꼭 플러시를 호출해서 영속성 컨텍스트의 변경 내용을 데이터베이스에 반영해야 한다.** JPA는 이런 문제를 예방하기 위해서 트랜잭션 커밋할 때 플러시를 자동으로 호출한다.

### JPQL 쿼리 실행시 플러시 자동 호출

```kotlin
@Test
internal fun `JPQL 쿼리 실행시 플러시 자동 호출`() {
    val teamA = Team("teamA")
    val teamB = Team("teamB")

    em.persist(teamA)
    em.persist(teamB)


    val teams = query.select(qTeam)
            .from(qTeam)
            .fetch()

    for (team in teams) {
        println("team : $team")
    }
}
```
JPQL이나 QueryDSL 같은 객체지향 **쿼리를 호출할 플러시가 실행된다.**

1. `teamA`, `teamB`를 영속성 컨텍스트에 저장한다.
2. QueryDSL으로 Team 전체를 조회 한다.
3. QueryDSL 쿼리 시점에 `teamA`, `teamB` 플러시를 이르켜 데이터베이스에 commit하지 않았다면 QueryDsl으로 조회한 값은 없을 것이다.
  
이런 결과가 나오기 때문에 쿼리를 실행하기 직전에 영속성 컨텍스트를 플러시해서 변경 내용을 데이터베이스에 반영해야 한다. JPA는 이런 문제를 예방하기 위해서 JPQL을 실행할 때도 플러시르 자동 호출한다. **참고로 식별자를 기준으로 조회하는 find() 메서드는 호출되지 않는다.**

## 참고
* [자바 ORM 표준 JPA 프로그래밍](http://acornpub.co.kr/book/jpa-programmig)