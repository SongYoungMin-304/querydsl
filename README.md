# querydsl

# 2023.01.19

### queryDsl 이란?

→ compileQueryDsl 을 통해서 entity 의 Q파일을 생성하여서 사용하는 방식

→ 자바의 방식으로 쿼리를 실행하기 때문에 컴파일 오류를 잡을 수 있다.

→ 동적 쿼리 등의 사용이 편리하다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/86529641-8472-4bac-95fc-296fe6a730e5/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/50a66586-a423-4c93-a900-fb8208e57875/Untitled.png)

```java
package study.querydsl.entity;
import lombok.*;
import javax.persistence.*;
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {"id", "username", "age"})
public class Member {
 @Id
 @GeneratedValue
 @Column(name = "member_id")
 private Long id;
 private String username;
 private int age;
 @ManyToOne(fetch = FetchType.LAZY)
 @JoinColumn(name = "team_id")
 private Team team;
 public Member(String username) {
      this(username, 0);
 }
 public Member(String username, int age) {
      this(username, age, null);
 }
 public Member(String username, int age, Team team) {
      this.username = username;
      this.age = age;
 if (team != null) {
      changeTeam(team);
 }
 }
 public void changeTeam(Team team) {
      this.team = team;
      team.getMembers().add(this);
 }
}
```

```java
package study.querydsl.entity;
import lombok.*;
import javax.persistence.*;
import java.util.ArrayList;
import java.util.List;
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = {"id", "name"})
public class Team {

 @Id @GeneratedValue
 @Column(name = "team_id")
 private Long id;
 private String name;
 
 @OneToMany(mappedBy = "team")
 private List<Member> members = new ArrayList<>();
 public Team(String name) {
      this.name = name;
 }
}
```

→ changeTeam 관련해서 team 세팅 뿐만 아니라 team에 있는 멤버값도 세팅 하는 이유!

→ member 엔티티를 가져와서 team을 세팅한 이후에 team에 있는 멤버를 가져오는 경우..

team을 이미 entity 로써 가져왔기 때문에 쿼리 재 실행을 안하고 해당 이유로 인해서 제대로 관리가 안됨.  flush를 해주던지 양방향 관계를 세팅해줘야 함

### queryDsl 예제

```java
@BeforeEach
    public void before(){
        jpaQueryFactory  = new JPAQueryFactory(em);
        Team teamA = new Team("teamA");
        Team teamB = new Team("teamB");

        em.persist(teamA);
        em.persist(teamB);

        Member member1 = new Member("member1", 10, teamA);
        Member member2 = new Member("member2", 20, teamA);
        Member member3 = new Member("member3", 30, teamA);
        Member member4 = new Member("member4", 40, teamA);

        em.persist(member1);
        em.persist(member2);
        em.persist(member3);
        em.persist(member4);

    }
```

- 샘플 데이터 적재

```java
@Test
    public void startJPQL(){
        //member1을 찾아라
        String qlString = "select m from Member m " +
                "where m.username =:username";

        Member findMember = em.createQuery(qlString, Member.class)
                .setParameter("username", "member1")
                .getSingleResult();

        assertThat(findMember.getUsername()).isEqualTo("member1");
    }
```

→ 일반적인 JPQL

```java
@Test
public void startQuerydsl(){
    QMember m = new QMember("m");

    Member member = jpaQueryFactory
            .select(m)
            .from(m)
            .where(m.username.eq("member1"))
            .fetchOne();

assertThat(member.getUsername()).isEqualTo("member1");
}
```

→ 생성된 Q파일을 통한 데이터 가져오기

```java
@Test
    public void startQuerydsl2(){
        // 같은 테이블을 조인하는 경우에는 alias 사용 필요
        //QMember m = new QMember("m");

        Member findMember = jpaQueryFactory
                .select(member)
                .from(member)
                .where(member.username.eq("member1"))
                .fetchOne();

        assertThat(findMember.getUsername()).isEqualTo("member1");
    }
```

→ static Import를 통한 member 사용

```java
@Test
    public void search(){
        Member findMember = jpaQueryFactory
                .selectFrom(member)
                .where(
                        member.username.eq("member1")
                        .and(member.age.eq(10))
                )
                /*.where(
                        member.username.eq("member1")
                                , (member.age.eq(10))
                )*/
                .fetchOne();

        assertThat(findMember.getUsername()).isEqualTo("member1");
    }
```

→ where 등을 and 대신 ,로 사용 가능

→ selectFrom 합쳐서 사용 가능

```java
public void resultFetch(){
        List<Member> fetch = jpaQueryFactory
                .selectFrom(member)
                .fetch();

        Member fetchOne = jpaQueryFactory
                .selectFrom(QMember.member)
                .fetchOne();

        Member fetchFirst = jpaQueryFactory
                .selectFrom(QMember.member)
                .fetchFirst();

        QueryResults<Member> result = jpaQueryFactory
                .selectFrom(member)
                .fetchResults();

        result.getTotal();
        List<Member> results = result.getResults();

        long l = jpaQueryFactory
                .selectFrom(member)
                .fetchCount();

        System.out.println(l);
    }
```

→ fetch → list 반화

→ fetchOne → 객체반환

→ fetchFirst → 첫번째 객체 반환

→ fetchResult → 페이징 관련 가져올 수 있게 처리
