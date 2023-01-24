# querydsl

# 2023.01.19

### queryDsl 이란?

→ compileQueryDsl 을 통해서 entity 의 Q파일을 생성하여서 사용하는 방식

→ 자바의 방식으로 쿼리를 실행하기 때문에 컴파일 오류를 잡을 수 있다.

→ 동적 쿼리 등의 사용이 편리하다.

![Untitled](https://user-images.githubusercontent.com/56577599/213719219-617705e9-7836-4fa0-85ed-2c19307731c4.png)

![Untitled (1)](https://user-images.githubusercontent.com/56577599/213719279-8214a165-415c-4758-b067-44975ffb36ce.png)


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

# 2023.01.21
1) QMember m = new QMember(”m”)

2) QMember.member

(`public static final QMember *member* = new QMember("member1");`)

3) `import static study.querydsl.entity.QMember.*member*;`

member

select(m)

from(m)~~ 

### 1) sort 처리

→ 정렬 처리

```java
@Test
    public void sort(){
        em.persist(new Member(null, 100));
        em.persist(new Member("member5", 100));
        em.persist(new Member("member6", 100));

        List<Member> fetch = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.eq(100))
                .orderBy(member.age.desc(), member.username.asc().nullsLast())
                .fetch();

        for (Member fetch1 : fetch) {
            System.out.println(fetch1);
        }

        assertThat(fetch.get(0).getUsername()).isEqualTo("member5");
        assertThat(fetch.get(1).getUsername()).isEqualTo("member6");
        assertThat(fetch.get(2).getUsername()).isNull();

    }

-------------------------------------------------------

실행 JPQL

select
        member1 
    from
        Member member1 
    where
        member1.age = ?1 
    order by
        member1.age desc,
        member1.username asc nulls last

실행 쿼리

select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        where
            member0_.age=? 
        order by
            member0_.age desc,
            member0_.username asc nulls last
```

### 2) 페이징처리

CASE1)

```java
@Test
    public void paging1(){
        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .orderBy(member.username.desc())
                .offset(1)
                .limit(2)
                .fetch();

        assertThat(result.size()).isEqualTo(2);
    }

-------------------------------------------------------

실행 JPQL
select
        member1 
    from
        Member member1 
    order by
        member1.username desc */

실행 SQL
select
            * 
        from
            ( select
                row_.*,
                rownum rownum_ 
            from
                ( select
                    member0_.member_id as member_id1_1_,
                    member0_.age as age2_1_,
                    member0_.team_id as team_id4_1_,
                    member0_.username as username3_1_ 
                from
                    member member0_ 
                order by
                    member0_.username desc ) row_ 
            where
                rownum <= ?
            ) 
        where
            rownum_ > ?

```

페이징 처리를 해도 JPQL은 변경되지 않는다. 내부적으로 JPQL을 쿼리로 변경할때 적용 될 뿐..

CASE2)

```java
@Test
    public void paging2(){
        QueryResults<Member> queryResults = jpaQueryFactory
                .selectFrom(member)
                .orderBy(member.username.desc())
                .offset(1)
                .limit(2)
                .fetchResults();

        assertThat(queryResults.getTotal()).isEqualTo(4);
        assertThat(queryResults.getLimit()).isEqualTo(2);
        assertThat(queryResults.getOffset()).isEqualTo(1);
        assertThat(queryResults.getResults().size()).isEqualTo(2);
    }

-------------------------------------------------------

실행 JPQL
/* select
        count(member1) 
    from
        Member member1 */

/* select
        member1 
    from
        Member member1 
    order by
        member1.username desc */

실행 SQL

select
            count(member0_.member_id) as col_0_0_ 
        from
            member member0_

select
            * 
        from
            ( select
                row_.*,
                rownum rownum_ 
            from
                ( select
                    member0_.member_id as member_id1_1_,
                    member0_.age as age2_1_,
                    member0_.team_id as team_id4_1_,
                    member0_.username as username3_1_ 
                from
                    member member0_ 
                order by
                    member0_.username desc ) row_ 
            where
                rownum <= ?
            ) 
        where
            rownum_ > ?
```

### 3) 집합

CASE1)

```java
public void aggregation(){
        List<Tuple> result = jpaQueryFactory
                .select(
                        member.count(),
                        member.age.sum(),
                        member.age.avg(),
                        member.age.max(),
                        member.age.min()
                )
                .from(member)
                .fetch();

        Tuple tuple = result.get(0);
        assertThat(tuple.get(member.count())).isEqualTo(4);
        assertThat(tuple.get(member.age.sum())).isEqualTo(100);
        assertThat(tuple.get(member.age.avg())).isEqualTo(25);
        assertThat(tuple.get(member.age.max())).isEqualTo(40);
        assertThat(tuple.get(member.age.min())).isEqualTo(10);
    }

-------------------------------------------------------

실행 JPQL

/* select
        count(member1),
        sum(member1.age),
        avg(member1.age),
        max(member1.age),
        min(member1.age) 
    from
        Member member1 */

실행 SQL
select
            count(member0_.member_id) as col_0_0_,
            sum(member0_.age) as col_1_0_,
            avg(member0_.age) as col_2_0_,
            max(member0_.age) as col_3_0_,
            min(member0_.age) as col_4_0_ 
        from
            member member0_
```

CASE2)

```java
@Test
    public void group() throws Exception{
        List<Tuple> fetch = jpaQueryFactory
                .select(team.name, member.age.avg())
                .from(member)
                .join(member.team, team)
                .groupBy(team.name)
                .fetch();

        Tuple teamA = fetch.get(0);
        Tuple teamB = fetch.get(1);

        assertThat(teamA.get(team.name)).isEqualTo("teamA");
        assertThat(teamA.get(member.age.avg())).isEqualTo(15);

        assertThat(teamB.get(team.name)).isEqualTo("teamB");
        assertThat(teamB.get(member.age.avg())).isEqualTo(35);

    }

-------------------------------------------------------

실행 JPQL
/* select
        team.name,
        avg(member1.age) 
    from
        Member member1   
    inner join
        member1.team as team 
    group by
        team.name */

실행 SQL

select
            team1_.name as col_0_0_,
            avg(member0_.age) as col_1_0_ 
        from
            member member0_ 
        inner join
            team team1_ 
                on member0_.team_id=team1_.id 
        group by
            team1_.name
```

→ MEMBER와 TEAM을 조인해서 GROUP BY 를 한다.

→ 연관관계 JOIN 이기 때문에 MEMBER.TEAM, TEAM 이런식으로 표현 가능하고

ON 절 사용이 필요 없다.(당연히 INNER JOIN 처리 됨)

### 4) 조인

1) 일반 INNER JOIN

```java
@Test
    public void join(){
        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .join(member.team, team)
                .where(team.name.eq("teamA"))
                .fetch();

        assertThat(result)
                .extracting("username")
                .containsExactly("member1","member2");
    }

```java
@Test
    public void join(){
        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .join(member.team, team)
                .where(team.name.eq("teamA"))
                .fetch();

        assertThat(result)
                .extracting("username")
                .containsExactly("member1","member2");
    }

-------------------------------------------------------

실행 JPQL
/* select
        member1 
    from
        Member member1   
    inner join
        member1.team as team 
    where
        team.name = ?1 */

실행 SQL
select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        inner join
            team team1_ 
                on member0_.team_id=team1_.id 
        where
            team1_.name=?
\
```

→ 연관관계 JOIN 이기 때문에 MEMBER.TEAM, TEAM 이런식으로 표현 가능하고

알아서 키 값으로 JOIN 이 된다.

2) 세타 조인(연관관계 없는 테이블? 연관관계 키 값을 사용하지 않은 조인)

```java
@Test
    public void theta_join(){
        em.persist(new Member("teamA"));
        em.persist(new Member("teamB"));
        em.persist(new Member("teamC"));

        List<Member> result = jpaQueryFactory
                .select(member)
                .from(member, team)
                .where(member.username.eq(team.name))
                .fetch();

        assertThat(result)
                .extracting("username")
                .containsExactly("teamA", "teamB");

    }

-------------------------------------------------------

실행 JPQL
/* select
        member1 
    from
        Member member1,
        Team team 
    where
        member1.username = team.name */

실행 SQL

select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ cross 
        join
            team team1_ 
        where
            member0_.username=team1_.name
```

→ 키 값으로 조인을 함..

→ 해당 부분은 외부 조인이 불가하다.(연관 키 값이 없는 테이블)

3) LEFT JOIN 방식(연관관계)

```java
@Test
    public void join_on_filtering(){

        List<Tuple> result = jpaQueryFactory
                .select(member, team)
                .from(member)
                .leftJoin(member.team, team)
                .on(team.name.eq("teamA"))
                //.where(team.name.eq("teamA"))
                .fetch();

        for (Tuple tuple : result) {
            System.out.println("tuple =" + tuple);
        }
    }

-------------------------------------------------------

실행 JPQL
/* select
        member1,
        team 
    from
        Member member1   
    left join
        member1.team as team with team.name = ?1 */

실행 SQL
select
            member0_.member_id as member_id1_1_0_,
            team1_.id as id1_2_1_,
            member0_.age as age2_1_0_,
            member0_.team_id as team_id4_1_0_,
            member0_.username as username3_1_0_,
            team1_.name as name2_2_1_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on member0_.team_id=team1_.id 
                and (
                    team1_.name=?
                )
```

4) LEFT JOIN 방식(연관관계 x)

```java
@Test
    public void join_on_no_releation(){
        em.persist(new Member("teamA"));
        em.persist(new Member("teamB"));
        em.persist(new Member("teamC"));

        List<Tuple> result = jpaQueryFactory
                .select(member, team)
                .from(member)
                .leftJoin(team)
                .on(member.username.eq(team.name))
                .fetch();

        for (Tuple tuple : result) {
            System.out.println(tuple);
        }

        /*assertThat(result)
                .extracting("username")
                .containsExactly("teamA", "teamB");*/

    }

-------------------------------------------------------

실행 JPQL
/* select
        member1,
        team 
    from
        Member member1   
    left join
        Team team with member1.username = team.name */

실행 SQL
select
            member0_.member_id as member_id1_1_0_,
            team1_.id as id1_2_1_,
            member0_.age as age2_1_0_,
            member0_.team_id as team_id4_1_0_,
            member0_.username as username3_1_0_,
            team1_.name as name2_2_1_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on (
                    member0_.username=team1_.name
                )
```

→ left join 에서 member.team, team 이런식으로 안씀(연관관계가 아니여서)

→ on 절을 통해서 해결

5) Fetch join 

fetch join 과 일반 join 의 차이는?

→ 영속성 관리의 여부.. 

```java
@Test
    public void fetchJoinUse(){
        em.flush();
        em.clear();

        Member findMmeber = jpaQueryFactory
                .selectFrom(member)
                .join(member.team, team).fetchJoin()
                .where(QMember.member.username.eq("member1"))
                .fetchOne();

        boolean loaded = emf.getPersistenceUnitUtil().isLoaded(findMmeber.getTeam());
        assertThat(loaded).as("패치 조인 미적용").isTrue();
    }

-------------------------------------------------------
case1) fetchJoin 미사용
실행 JPQL
/* select
        member1 
    from
        Member member1   
    inner join
        member1.team as team 
    where
        member1.username = ?1

실행 sql
select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        inner join
            team team1_ 
                on member0_.team_id=team1_.id 
        where
            member0_.username=?

case2) fetchJoin 사용
실행 JPQL

/* select
        member1 
    from
        Member member1   
    inner join
        fetch member1.team as team 
    where
        member1.username = ?1 */

실행 sql
select
            member0_.member_id as member_id1_1_0_,
            team1_.id as id1_2_1_,
            member0_.age as age2_1_0_,
            member0_.team_id as team_id4_1_0_,
            member0_.username as username3_1_0_,
            team1_.name as name2_2_1_ 
        from
            member member0_ 
        inner join
            team team1_ 
                on member0_.team_id=team1_.id 
        where
            member0_.username=?
```

→ 실행 JPQL은 다르고 실행 쿼리를 동일하면 FETCH JOIN 미사용 시 영속성 관리가 되지 않는다.  

예를 들어 아래의 코드를 실행 시 FETCH JOIN 이 안되어있으면 LAZY LOADING 을 통해서 

TEAM를 가져오는 쿼리가 재 실행된다.

```
Team team = findMmeber.getTeam();
team.getName();
```

6) 서브쿼리

`import static com.querydsl.jpa.JPAExpressions.*select*;`

→ select

case1)

```java
@Test
    public void subQuery() throws Exception{

        QMember memberSub = new QMember("memberSub");

        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.eq(
                        select(memberSub.age.max())
                                .from(memberSub)
                ))
                .fetch();

        assertThat(result).extracting("age")
                .containsExactly(40);
    }

-------------------------------------------------------
실행 JPQL
/* select
        member1 
    from
        Member member1 
    where
        member1.age = (
            select
                max(memberSub.age) 
            from
                Member memberSub
        ) */

실행 SQL
select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        where
            member0_.age=(
                select
                    max(member1_.age) 
                from
                    member member1_
            )
```

case2)

```java
@Test
    public void subQueryGoe() throws Exception{

        QMember memberSub = new QMember("memberSub");

        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.goe(
                        select(memberSub.age.avg())
                                .from(memberSub)
                ))
                .fetch();

        assertThat(result).extracting("age")
                .containsExactly(30, 40);
    }

-------------------------------------------------------
실행 JPQL
/* select
        member1 
    from
        Member member1 
    where
        member1.age >= (
            select
                avg(memberSub.age) 
            from
                Member memberSub
        ) */

실행 SQL
select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        where
            member0_.age>=(
                select
                    avg(member1_.age) 
                from
                    member member1_
            )
```

case3)

```java
@Test
    public void subQueryIn() throws Exception{

        QMember memberSub = new QMember("memberSub");

        List<Member> result = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.in(
                        select(memberSub.age)
                                .from(memberSub)
                                .where(memberSub.age.gt(10))
                ))
                .fetch();

        assertThat(result).extracting("age")
                .containsExactly(20, 30, 40);
    }

-------------------------------------------------------
실행 JPQL
/* select
        member1 
    from
        Member member1 
    where
        member1.age in (
            select
                memberSub.age 
            from
                Member memberSub 
            where
                memberSub.age > ?1
        ) */

실행 SQL
select
            member0_.member_id as member_id1_1_,
            member0_.age as age2_1_,
            member0_.team_id as team_id4_1_,
            member0_.username as username3_1_ 
        from
            member member0_ 
        where
            member0_.age in (
                select
                    member1_.age 
                from
                    member member1_ 
                where
                    member1_.age>?
            )
```

case4)

```java
@Test
    public void selectSubquery(){
        QMember memberSub = new QMember("memberSub");

        List<Tuple> fetch = jpaQueryFactory
                .select(member.username,
                        select(memberSub.age.avg())
                                .from(memberSub))
                .from(member)
                .fetch();

        for (Tuple tuple : fetch) {
            System.out.println("tuple = "+tuple);
        }
    }

-------------------------------------------------------
실행 JPQL
/* select
        member1.username,
        (select
            avg(memberSub.age) 
        from
            Member memberSub) 
    from
        Member member1 */

실행 SQL
select
            member0_.username as col_0_0_,
            (select
                avg(member1_.age) 
            from
                member member1_) as col_1_0_ 
        from
            member member0_
```

→ from 절안에 서브쿼리 기능은 없다. 

→ 최대한 no sql,, sql은 단지 데이터를 가져오는데만 사용 하자

7) case when

CASE1)

```java
@Test
    public void basicCase(){
        List<String> fetch = jpaQueryFactory
                .select(member.age
                        .when(10).then("열살")
                        .when(20).then("스무살")
                        .otherwise("기타"))
                .from(member)
                .fetch();

        for (String s : fetch) {
            System.out.println(s);
        }
    }

-------------------------------------------------------
실행 JPQL
/* select
        case 
            when member1.age = ?1 then ?2 
            when member1.age = ?3 then ?4 
            else '기타' 
        end 
    from
        Member member1 */

실행 SQL
select
            case 
                when member0_.age=? then ? 
                when member0_.age=? then ? 
                else '기타' 
            end as col_0_0_ 
        from
            member member0_
```

CASE2)

```java
@Test
    public void complexCase(){
        List<String> result = jpaQueryFactory
                .select(new CaseBuilder()
                        .when(member.age.between(0, 20)).then("0~20살")
                        .when(member.age.between(21, 30)).then("21~30살")
                        .otherwise("기타"))
                .from(member)
                .fetch();

        for (String s : result) {
            System.out.println(s);
        }
    }

-------------------------------------------------------
실행 JPQL
/* select
        case 
            when (member1.age between ?1 and ?2) then ?3 
            when (member1.age between ?4 and ?5) then ?6 
            else '기타' 
        end 
    from
        Member member1 */

실행 SQL
select
            case 
                when member0_.age between ? and ? then ? 
                when member0_.age between ? and ? then ? 
                else '기타' 
            end as col_0_0_ 
        from
            member member0_
```

8) CONSTANT

```java
@Test
    public void constant(){
        List<Tuple> result = jpaQueryFactory
                .select(member.username, Expressions.constant("A"))
                .from(member)
                .fetch();

        for (Tuple tuple : result) {
            System.out.println(tuple);
        }
    }

-------------------------------------------------------
실행 JPQL
/* select
        member1.username 
    from
        Member member1 */ 

실행 SQL
select
            member0_.username as col_0_0_ 
        from
            member member0_
```

→ 쿼리에는 constant 가 실제로 붙어서 실행되지는 않고 실행된 데이터에 붙인다.

9) CONCAT

```java
@Test
    public void concat(){
        List<String> result = jpaQueryFactory
                .select(member.username.concat("_").concat(member.age.stringValue()))
                .from(member)
                .where(member.username.eq("member1"))
                .fetch();

        for (String s : result) {
            System.out.println(s);
        }

    }

-------------------------------------------------------
실행 JPQL
/* select
        concat(concat(member1.username,
        ?1),
        str(member1.age)) 
    from
        Member member1 
    where
        member1.username = ?2 */ 

실행 SQL
select
            member0_.username||?||to_char(member0_.age) as col_0_0_ 
        from
            member member0_ 
        where
            member0_.username=?
```



# 2023.01.21

### 1) 프로젝션 결과 값 반환(기본)

```java
@Test
    public void simpleProjection(){
        List<String> result = jpaQueryFactory
                .select(member.username)
                .from(member)
                .fetch();

        for (String s : result) {
            System.out.println(s);
        }
    }

@Test
    public void tupleProjection(){
        List<Tuple> result = jpaQueryFactory
                .select(member.username, member.age)
                .from(member)
                .fetch();

        for (Tuple tuple : result) {
            String username = tuple.get(member.username);
            Integer age = tuple.get(member.age);

            System.out.println(username);
            System.out.println(age);
        }
    }
```

→ 단건은 String으로 반환, 여러 건은 Tuple로 반환해서 사용

`tuple 도 queryDsl 에 종속적이여서.. 서비스나 controller 등에서 호출하지 말자`

### 2) 프로젝션 결과 값 반환(DTO)

AS-IS

```java
@Test
    public void findDtoByJPQL(){
        List<MemberDto> resultList = 
em.createQuery("select new study.querydsl.dto.MemberDto(m.username, m.age) "+
"from Member m", MemberDto.class)
                .getResultList();

        for (MemberDto memberDto : resultList) {
            System.out.println(memberDto);
        }
    }
```

→ JPQL을 통해서 데이터를 생성자 방식으로 가져와야 하는 데 불편함 존재

TO-BE

CASE1) 프로퍼티 접근

```java
@Test
    public void findDtoBySetter(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.bean(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        for (MemberDto memberDto : result) {
            System.out.println(memberDto);
        }
    }
```

→ Getter Setter 필요함

CASE2) 필드 직접 주입

```java
@Test
    public void findDtoByField(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.fields(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        for (MemberDto memberDto : result) {
            System.out.println(memberDto);
        }
    }
```

→ Getter Setter 필요없음

CASE3) 생성자 주입

```java
@Test
    public void findDtoByConstructor(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.constructor(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        for (MemberDto memberDto : result) {
            System.out.println(memberDto);
        }
    }
```

→ 생성자 방식으로 처리

CASE4) SELECT 서브 쿼리

```java
@Test
    public void findUserDto(){

        QMember memberSub = new QMember("memberSub");

        List<UserDto> result = jpaQueryFactory
                .select(Projections.fields(UserDto.class,
                        member.username.as("name"),

                        ExpressionUtils.as(JPAExpressions
                                .select(memberSub.age.max()).
                                from(memberSub), "age")
                ))
                .from(member)
                .fetch();

        for (UserDto memberDto : result) {
            System.out.println(memberDto);
        }
    }
```

→ [ExpressionUtils.as](http://ExpressionUtils.as) 방식으로 alias를 주는 방식

→ .as 사용가능

CASE5) DTO를 큐로 만들기

```java
@QueryProjection
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }

@Test
    public void findDtoByQueryProjection(){
        List<MemberDto> result = jpaQueryFactory
                .select(new QMemberDto(member.username, member.age))
                .from(member)
                .fetch();

        for (MemberDto memberDto : result) {
            System.out.println(memberDto);
        }
    }
```

→ DTO에 @QueryProjection 을 하면 QMemberDto 파일이 생성된다.

→ 간단하게 사용 가능 하나.. queryDsl 종속적인 라이브러리를 dto에 써도 될지에 대한 의문이 존재

### 3) 동적 쿼리_BooleanBuilder

```java
@Test
    public void dynamicQuery_BooleanBuilder(){
        String usernameParam = "member1";
        Integer ageParam = null;

        List<Member> result = searchMember1(usernameParam, ageParam);
        assertThat(result.size()).isEqualTo(1);
    }

    private List<Member> searchMember1(String usernameParam, Integer ageParam) {

        BooleanBuilder builder = new BooleanBuilder();
        if(usernameParam != null){
            builder.and(member.username.eq(usernameParam));
        }
        if(ageParam != null){
            builder.and(member.age.eq(ageParam));
        }
        return jpaQueryFactory
                .selectFrom(member)
                .where(builder)
                .fetch();
    }
```

→ builder를 통한 동적쿼리 처리

### 4) 동적 쿼리_다중Param

```java
@Test
    public void dynamicQuery_WhereParam(){
        String usernameParam = "member1";
        Integer ageParam = null;

        List<Member> result = searchMember2(usernameParam, ageParam);
        assertThat(result.size()).isEqualTo(1);
    }

    private List<Member> searchMember2(String usernameParam, Integer ageParam) {
        return jpaQueryFactory
                .selectFrom(member)
//                .where(usernameEq(usernameParam), ageEq(ageParam))
                .where(allEq(usernameParam, ageParam))
                .fetch();
    }

    private BooleanExpression usernameEq(String usernameParam) {
        return usernameParam != null ? member.username.eq(usernameParam) : null;
    }

    private BooleanExpression ageEq(Integer ageParam) {
        return ageParam != null ? member.age.eq(ageParam) : null;
    }

    // 광소 상태 isValid, 날짜가 IN : isServicable

    private Predicate allEq(String usernameCond, Integer ageCond){
        return usernameEq(usernameCond).and(ageEq(ageCond));
    }
```

→ BooleanExpression 반환 값으로 where 값에 넘긴다.

→ 해당 조건들을 allEq 처럼 조합해서 사용가능

### 5) 벌크성 처리

```java
@Test
    public void bulkUpdate(){

        //member1 = 10 -> DB member1
        //member2 = 20 -> DB member2
        //member3 = 30 -> DB member3
        //member4 = 40 -> DB member4

        long count = jpaQueryFactory
                .update(member)
                .set(member.username, "비회원")
                .where(member.age.lt(28))
                .execute();

        // 벌크형 쿼리는 영속성 컨데이터를 건드리지 않는다.

        //1 member1 = 10 -> DB 비회원
        //2 member2 = 20 -> DB 비회원
        //3 member3 = 30 -> DB member3
        //4 member4 = 40 -> DB member4

        em.flush();
        em.clear();

        List<Member> fetch = jpaQueryFactory
                .selectFrom(member)
                .fetch();

        for (Member fetch1 : fetch) {
            System.out.println(fetch1);
        }

    }

    @Test
    public void bulkAdd(){
        long execute = jpaQueryFactory
                .update(member)
                .set(member.age, member.age.multiply(2))
                .execute();
    }

    @Test
    public void bulkDelete(){
        long execute = jpaQueryFactory
                .delete(member)
                .where(member.age.gt(10))
                .execute();
    }
```

→ 벌크성 처리는 영속성 관리가 되지 않은 상태이기 때문에 재 조회를 해도 변경 전 데이터를 가져오게 되는 상황이 있어서 수정 시

em.flush() em.clear() 를하자

### 6) 벌크성 처리

```java
@Test
    public void sqlFunction2(){
        List<String> fetch = jpaQueryFactory
                .select(member.username)
                .from(member)
/*
                .where(member.username.eq(
                        Expressions.stringTemplate("function('lower', {0})", member.username)))
*/
                .where(member.username.eq(member.username.lower()))
                .fetch();

        for (String s : fetch) {
            System.out.println(s);
        }
    }
```

Expressions.stringTemplate 방식으로 function 사용 가능하나 기본적인 함수는 lower 처럼 존재한다.


# 2023.01.23


### 1) QUERYDSL 정리

**case1) 일반**

AS-IS) 기본 JPA

```java
private final EntityManager em;

public List<Member> findAll(){
     return em.createQuery("select m from Member m", Member.class)
             .getResultList();
}
```

→ em을 사용해서 쿼리 작성

TO-BE) QUERYDSL

```java
@Bean
JPAQueryFactory jpaQueryFactory(EntityManager em){
     return new JPAQueryFactory(em);
}

public MemberJpaRepository(EntityManager em, JPAQueryFactory queryFactory) {
        this.em = em;
        this.queryFactory = queryFactory;
    }

// Bean 주입을 안하는 경우
public MemberJpaRepository(EntityManager em){
        this.em = em;
        this.queryFactory = new JPAQueryFactory(em);
}

public List<Member> findAll_Querydsl(){
     return queryFactory
              .selectFrom(member)
//              .selectFrom(QMember.member)  // staticImport
              .fetch();
```

- static import 란?

```java
public class staticTest {
    public static String song ="songTest";
}

~~~~~~~~

import static study.querydsl.controller.staticTest.song;

staticTest.song;

song; // static import를 하면 사용 가능

```

**case2) 일반(검색 조건 존재)**

AS-IS) 기본 JPA

```java
public List<Member> findByUsername(String username){
        return em.createQuery("select m from Member m " +
                "where m.username = :username", Member.class)
                .setParameter("username", username)
                .getResultList();
    }
```

TO-BE) QUERYDSL

```java
public List<Member> findByUsername_Querydsl(String username) {
        return queryFactory
                .selectFrom(member)
                .where(member.username.eq(username))
                .fetch();
    }
```

**case3) 조건 검색 - Builder 처리**

```java
package study.querydsl.dto;

import com.querydsl.core.annotations.QueryProjection;
import lombok.Data;

@Data
public class MemberTeamDto {
    private Long memberId;
    private String username;
    private int age;
    private Long teamId;
    private String teamName;

    @QueryProjection
    public MemberTeamDto(Long memberId, String username, int age, Long teamId, String teamName) {
        this.memberId = memberId;
        this.username = username;
        this.age = age;
        this.teamId = teamId;
        this.teamName = teamName;
    }
}
```

→ QueryProjection 을 통해서 Q파일 생성

```java
package study.querydsl.dto;

import lombok.Data;

@Data
public class MemberSearchCondition {
    //회원명, 팀명, 나이(ageCoe, ageLoe)

    private String username;
    private String teamName;
    private Integer ageGoe;
    private Integer ageLoe;
}
```

```java
BooleanBuilder builder = new BooleanBuilder();
if(hasText(condition.getUsername())) {
     builder.and(member.username.eq(condition.getUsername()));
}
if(hasText(condition.getTeamName())) {
     builder.and(team.name.eq(condition.getTeamName()));
}
if(condition.getAgeGoe()!=null){
     builder.and(member.age.goe(condition.getAgeGoe()));
}
if(condition.getAgeLoe()!=null){
     builder.and(member.age.loe(condition.getAgeLoe()));
}

public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition){

     return queryFactory
             .select(new QMemberTeamDto(
                    member.id.as("memberId")
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(builder)
            .fetch();
}

```

**case4) 조건 검색 - 매소드 방식**

```java
public List<MemberTeamDto> search(MemberSearchCondition condition){

        return queryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")
                ))
                .from(member)
                .leftJoin(member.team, team)
                .where(
                        usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe())
                )
                .fetch();
    }

private BooleanExpression usernameEq(String username) {
     return hasText(username) ? member.username.eq(username) : null;
}

private BooleanExpression teamNameEq(String teamName) {
     return hasText(teamName) ? team.name.eq(teamNmae) : null;
}

private BooleanExpression ageGoe(Integer ageGoe) {
     return ageGoe != null ? member.age.goe(ageGode) : null;
}

private BooleanExpression ageLoe(Integer ageLoe) {
     return ageLoe!= null ? member.age.loe(ageLoe) : null;
}

private BooleanExpression ageBetween(int a, int b){
        return ageGoe(a).and(ageLoe(b));
    }
// 이런식으로 합쳐서 사용 할 수 도 있다.
```
