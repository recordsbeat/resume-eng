

### I used Spring Batch.

When I need to create something by combining data from different DBs...
I usually write like this.

![](https://images.velog.io/images/recordsbeat/post/88c6590e-28dd-4c7b-b73c-e75d721faab4/image.png)

1. Sends the data selection result of db1 from ItemReader to ItemWriter between chunk-oriented step usage.
2. Select items passed from ItemReader from another table in db1 again
_(for performance reasons, use select-in to map objects to source code instead of using join for performance reasons)_
3. Using the above result as db2 query condition, select
4. Finally write to db


~~Am I the only one writing this..~~

(I set the ItemReader to return a list rather than a single item, and the processor tried processing.)





### Failed.

The famous and famous OutOfMemory Error
(replaced with the contents of the spring batch meta table as an error that occurred in an actual production environment)

![](https://images.velog.io/images/recordsbeat/post/9ecd5876-dcea-49a0-a82e-1c8e65f9ae10/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-04-25%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%203.49.27.png)

~~Countless failures !!~~

I thought it was a memory issue because it was large.
So, I tried all kinds of things such as batch insert update etc..
> [hibernate batch insert update transaction configuration](https://m.blog.naver.com/writer0713/221723210970)
[Batch insert optimization in Spring Data](https://homoefficio.github.io/2020/01/25/Spring-Data%EC%97%90%EC%84%9C-Batch-Insert-%EC%B5%9C%EC%A0%81%ED%99%94/)
..As an aside, I just gave the hibernate configuration value (?).


**When looking at the phenomenon, when**
monitoring while looking at the number of commits in the step_execution table in the meta table, it went at a reasonable
speed at the beginning, and then converges to 0 when it exceeds a certain commit. After that, an OOM error occurred.

It would have been nice to have checked the memory buffer earlier with an apm tool like pinpoint.




### In conclusion...

JPA is Hibernate's interface layer that provides user convenience, that is, Hibernate
is an implementation of JPA
![](https://images.velog.io/images/recordsbeat/post/504fcaef-216e-4f98-85f3-16779a23b06d/image.png)
Source - https://suhwan.dev/2019/02/24/jpa-vs-hibernate-vs-spring-data-jpa/


The problem was that a function called **QueryPlan Cache existed* inside hibernate .
> **Query Plan Cache**
Every JPQL query or Criteria query is parsed into an Abstract Syntax Tree (AST) prior to execution so that Hibernate can generate the SQL statement. Since query compilation takes time, Hibernate provides a QueryPlanCache for better performance.
> 
For native queries, Hibernate extracts information about the named parameters and query return type and stores it in the ParameterMetadata.
> 
For every execution, Hibernate first checks the plan cache, and only if there's no plan available, it generates a new plan and stores the execution plan in the cache for future reference.
>
[Hibernate Query Plan Cache](https://www.baeldung.com/hibernate-query-plan-cache#query-plan-cache)


All queries are converted to AST (abstract syntax tree) and hibernate generates SQL through it.
In this process, a function called QueryPlanCache is provided for better performance, which extracts parameter information and stores it in ParameterMetadata.
**Between every query execution, hibernate checks** the plan cache and creates a new plan if there is no matching plan.

![](https://images.velog.io/images/recordsbeat/post/67a27204-b154-42a7-bec4-a6b8330b0c41/StatementLifeCycle-1024x767.png)
Source - https://vladmihalcea.com/improve-statement-caching-efficiency-in-clause-parameter-padding/


But what about the default capacity of this QueryPlanCache and ParameterMetadata?
![](https://images.velog.io/images/recordsbeat/post/259a9377-2d02-4f1a-a204-380aa91e9a66/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-04-25%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%204.42.31.png)

_**Although the application memory size is 1024mb, the
default is 2048 for hibernate - query plan cache alone.**_

A situation where the navel is bigger than the stomach




### Let's look at the cause

You can see that hibernate uses the query plan cache for each query.
But why is this query plan cache in trouble?

There was a part that was overlooked in the logic I mentioned at the beginning.


![](https://images.velog.io/images/recordsbeat/post/884c6e65-7b32-40cc-aadc-6882b610593b/image.png)

Yes.

It is said that hibernate checks QueryPlanCache before executing each query and creates a new plan cache if there is no plan available.

Looking at the picture above, the size of the New selected ``List<item>`` is not constant, so
**the number of parameters in the select-in statement** is inconsistent . **Therefore , a QueryPlanCache was created for each size of the List used** for
non-fixed select-in, and this was accumulated in memory, causing OutOfMemory to burst.



### Solutions

Simple.

Developer Jung Ji-beom of nhn posted a clear solution.
> https://meetup.toast.com/posts/211
1) in_clause_parameter_padding
2) padding programmatically
3) adjust size of execution plan cache 

1 and 3 above were selected and set.
Number 1 (in_clause_parameter_padding) is
to fix the number of parameters to be entered in select-in to the power of 2.

```

    protected Map<String, Object> jpaProperties() {
        Map<String, Object> props = new HashMap<>();
        props.put("hibernate.session_factory.interceptor", interceptor);

        // execution query plan optimization
        props.put("hibernate.query.in_clause_parameter_padding", true);
        props.put("hibernate.query.plan_cache_max_size", 256);
        props.put("hibernate.query.plan_parameter_metadata_max_size", 16
        
        return props;
    }
```

The criteria for reducing ``plan_cache_max_size`` and ``plan_parameter_metadata_max_size`` are

>
...By default, the maximum number of entries in the plan cache is 2048. An HQLQueryPlan object occupies approximately 3MB. ~3 * 2048 = ~6GB, and **I the heap size is limited to 4GB**…
Finally! That must be the cause!
The solution is simple: decreasing the query plan cache size by setting the following properties:
>
spring.jpa.properties.hibernate.query.plan_cache_max_size: controls the maximum number of entries in the plan cache (defaults to 2048)
>
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size: manages the number of ParameterMetadata instances in the cache (defaults to 128)
>
**I set them to 1024 and 64 respectively**.
>
https://medium.com/quick-code/what-a-recurring-outofmemory-error-taught-me-11f2061063a1

In the above article, in the case of 4GB,
plan_cache_max_size = 1024
plan_parameter_metadata_max_size was set to 64
, so I also set it according to the ratio.

And with the above solution, the OutOfMemory error is said to be fixed.

### Wrapping up

_**In hibernate, you have to be very careful to use the select-in clause in which the size of the parameter is not fixed.**_


**Environment**
- Spring Batch
-  JPA - QueryDsl
- QuerydslItemReader - Dongwook Lee
( https://github.com/jojoldu/spring-batch-querydsl )
- JVM memory - max 1024



**reference link**

https://vladmihalcea.com/improve-statement-caching-efficiency-in-clause-parameter-padding/

https://medium.com/quick-code/what-a-recurring-outofmemory-error-taught-me-11f2061063a1

https://meetup.toast.com/posts/211

https://www.baeldung.com/hibernate-query-plan-cache
