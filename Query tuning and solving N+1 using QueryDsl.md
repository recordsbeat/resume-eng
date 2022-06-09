

## Getting started

If you use JPA (spring-data), the ORM of the Spring camp, you will often run into limitations with queries. Because of this, if you are looking for a solution, you can see a method called QueryDsl without difficulty.

I wrote this article to summarize the trial and error experienced while using QueryDsl in the field and my own solution.

Let's talk about the conclusion earlier.

> 
**QueryDsl** 
- Used only for querying for listing and extracting complex queries Extracted
through dto or a separate object instead of domain model

_When using domain model, query cost (plan) or n+1 problem for select cannot be avoided
When using from clause as B in entity A (owned) - B (dependent) relationship (due to query cost problem) An n+1 problem for B arises. Therefore, it is applied only when you want to use it like mybatis or native query._





## What happend?
The situation was as follows.
1. I have a project that uses JPA.
2. I used a query to extract a large amount of data from the project.
3. The query was too slow, so I modified it according to the plan.
4. N+1 problem and persistence error occurred between QueryDsl and JPA.



## Let's see..

The following QueryDsl select statement was written.
RootEntity and RelatedEntity have a **1:1 relationship** and
Optional = True _(the RootEntity does not need to exist in the RelatedEntity table)_

**And the two entities are made up of one- way references** that are RootEntity -> RelatedEntity .

```
public Page<RootEntity> findAllRootEntityByUseYn(Pageable pageable)
{
    QRootEntity RootEntity = QRootEntity.RootEntity;
    QRelatedEntity RelatedEntity = QRelatedEntity.RelatedEntity;

    JPAQuery<RootEntity> query = queryFactory
        .select(RootEntity)
        .from(RootEntity)
        .innerJoin(RootEntity.RelatedEntity, RelatedEntity)
        .fetchJoin()
        .where(
            RelatedEntity.useYn.eq(UseYn.Y)
        );

    QueryResults<RootEntity> result = query
        .groupBy(RootEntity.RootEntitySeq)
        .orderBy(RootEntity.RootEntitySeq.desc())
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetchResults();


    return new PageImpl<>(result.getResults(), pageable, result.getTotal());
}

```
**_*This is a much simplified query than it really is._**


And the plan for the corresponding query statement was as follows.

![](https://images.velog.io/images/recordsbeat/post/555975c3-2b89-40da-bdca-5ec69e47a16b/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-12-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2012.34.45.png)

The rootEntity is doing a full index and there is a cost for temporary and filesort.
As a result, the query speed was more than doubled.

> **ALL**
A full scan is performed for a join with the previous table. <br>
If the first table (used in the join) is not static, it is inefficient, and in **most cases very slow performance.** The ALL type can be avoided by adding an index to extract a row with a normal constant value or a constant column value. <br>
To make the omitted query as fast as possible, pay attention to Using filesort or Using temporary in the Extra value. <br>
Reference link - How to interpret the query plan <br>
https://database.sarang.net/?inc=read&aid=24199&criteria=mysql



## trial and error

So I changed the query.

```
public Page<RootEntity> findAllRootEntityByUseYn(Pageable pageable)
{
    QRootEntity RootEntity = QRootEntity.RootEntity;
    QRelatedEntity RelatedEntity = QRelatedEntity.RelatedEntity;

    JPAQuery<RootEntity> query = queryFactory
        .select(RootEntity)
        // changed the target of from clue to RelatedEntity
        .from(RelatedEntity)
        .innerJoin(RootEntity.RelatedEntity, RelatedEntity)
        .on(RelatedEntity.RootEntitySeq.eq(RootEntity.RootEntitySeq))
        .where(
            RelatedEntity.useYn.eq(UseYn.Y)
        );

    QueryResults<RootEntity> result = query
	    // changed the target of groupBy and orderBy to RelatedEntity
        .groupBy(RelatedEntity.RootEntitySeq)
        .orderBy(RelatedEntity.RootEntitySeq.desc())
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetchResults();


    return new PageImpl<>(result.getResults(), pageable, result.getTotal());
}

```
To avoid full indexing, RelatedEntity, which is the target of where condition, was moved to the from clause, and groupBy and OrderBy were also performed.


![](https://images.velog.io/images/recordsbeat/post/65028a04-bc9c-477e-b5f3-9ad2ad82d332/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-12-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2012.57.32.png)

Looking at the query plan, it seems to be running properly.


And I looked at the query log.
![](https://images.velog.io/images/recordsbeat/post/3c9d1460-8681-479b-86f4-dff31c41284f/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-12-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%201.02.11.png)

...? How many logs are there..?

**~~_(N+1 welcome)_~~**


I put RootEntity in the select clause, but the fetchJoin to call the Entity Model did not work, so the RelatedEntity select action occurred several times.

In other words, N+1 cannot be avoided to search for RootEntity from the relationship between RootEntity and RelatedEntity to the join relationship for RelatedEntity.

Also, you might think that you can write a query by replacing RootEntity and RelatedEntity with bidirectional references, but in reality, there are many related objects of RootEntity besides RelatedEntity, which can cause a more complicated join.

_*If you have other opinions or errors on this part, please leave a comment!_



## Solutions

The trial and error above is written simply in writing, but after about two weeks, I would rather write mybatis! I thought that (at least because there is no n+1)

Then suddenly. Can't I use QueryDsl like mybatis? I came up with the idea and put it into practice.

```
public Page<QueryDto> findAllRootEntityByUseYn(Pageable pageable)
{
    QRootEntity RootEntity = QRootEntity.RootEntity;
    QRelatedEntity RelatedEntity = QRelatedEntity.RelatedEntity;

    JPAQuery<QueryDto> query = queryFactory
        .select(
                Projections.constructor(QueryDto.class,
                RootEntity.col1
                , RootEntity.col2
                , RelatedEntity.col1
                ....
            ))
       	// changed the target of from clue to RelatedEntity
        .from(RelatedEntity)
        .innerJoin(RootEntity.RelatedEntity, RelatedEntity)
        .on(RelatedEntity.RootEntitySeq.eq(RootEntity.RootEntitySeq))
        .where(
            RelatedEntity.useYn.eq(UseYn.Y)
        );

    QueryResults<QueryDto> result = query
	    // changed the target of groupBy and orderBy to RelatedEntity
        .groupBy(RelatedEntity.RootEntitySeq)
        .orderBy(RelatedEntity.RootEntitySeq.desc())
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetchResults();


    return new PageImpl<>(result.getResults(), pageable, result.getTotal());
}

```

I just made one more Dto for select.
In the first place, we want to eliminate the act of loading the hibernate persistent object called Entity object itself.
(Actually, this behavior was used a lot in other select statements, so I don't know why I tried so hard to load the Entity Model only in this query statement.)

If you run that query

![](https://images.velog.io/images/recordsbeat/post/a4038d3a-4b31-4d22-8cfb-3eb024886a59/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-12-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%201.23.57.png)

It loads fine without n+1 or any other errors.

Although the conclusion was stated above, it seems much more efficient to load the library called QueryDsl using a separate dto class when a complex select statement is used rather than when loading a persistent object.

However, since the use of QueryDsl and the JPA repository is divided in this way, **the reference relationship of the entity is not modified to use the query statement, so there is no effect between methods.**



## Wrap-up

Query performance issues and tuning that I experienced for the first time while extracting a large amount of data in the field.
And it was trial and error that I met while solving the issue through a library called QueryDsl. I wrote this down because I thought I would do it again later if I didn't write it down.

May everyone be free from n+1

![](https://images.velog.io/images/recordsbeat/post/c1890541-544b-4277-a7e8-8d8f7c9de6b7/image.png)
