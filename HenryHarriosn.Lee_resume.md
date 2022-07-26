## ✔ Info 

### Henry Harrison Lee 
![image](https://user-images.githubusercontent.com/51354965/172858000-9967e637-cf3f-43f7-9fb7-615a0dbd2799.png)
- Backend Developer based on Java & Spring (over 5years experienced)
- Bachelor's degree of Software Engineering
- Korean (Native), English (Fluent)

### Outlinks

- [LinkedIn](https://www.linkedin.com/in/heeyeon-lee-61763a190/)
- [Blog](https://velog.io/@recordsbeat) <br>
(All article is written in Korean, you may read with Chrome translator. I think it's working very well, but some of articles from my blog have been translated in English roughly in this repository and I attach them each of proper section of this article)
- [GitHub](https://github.com/recordsbeat)

### Contact
- beathuman@naver.com
- or LinkedIn Messenger!


### Experience
- **Back End Developer@NAVER Corp (Sep 2021 - Present)**
  - Backend development based on Spring boot within MSA, EDA
  
- **Catalog Data Development@Wemakeprice(Jul 2020 - Aug 2021 · 1 yr 2 mos)**
  - Search Data Operation with Spirng MVC, Batch in MSA
- **Software Engineer@Guiving(Aug 2019 - Jun 2020 · 11 mos)**
  - Backend-Web development
  - DevOps, Server Infra engineering
- **Project Manager@Drift Entertainment(Oct 2018 - Aug 2019 · 11 mos)**
  - Project management
  - Web development
  - AWS Infra engineering
- **Web Developer@DongeunIT(May 2017 - Jul 2018 · 1 yr 3 mos)**
  - EMR of Soonchunhyang University Hospital management
  - Emergency monitoring service
<br>

## 🔨 Development Environment 
- Framwork : Spring boot 2.5.x
- SDK : JDK 11 (modern Java)
- IDE : IntelliJ
- Infra : Mysql, Elastic Search, Redis, Kafka
- Monitoring : Grafana, Pinpoint, Spring Admin
- DevOps : Github, Bitbucket, Jenkins, Aws, Kubernetes 


## 🧱 Domain 
### Influencer Service(User Generated Content) @[Naver](https://influencercenter.naver.com) - current
- Collecting content through multiple platform (such as. naver blog, youtube, instagram, twitter)
- Expose contents to search engine
- Enablement to create user generated content in Influencer platform itself


### E-commerce @[Wemakeprice](https://wemakeprice.com)
- Categorizing & labeling item data
- Determining the lowest price of each categories
- Expose the lowest price to search engine


<br><br>



## 💪 What I’m good at

- Knowing JPA(hibernate) lifecycle to avoid unintended query & performance decline <br>
[Query tuning and solving N+1 using QueryDsl - from my blog](https://github.com/recordsbeat/resume-eng/blob/a275fd280ca51d9c9df37c48a34e57ebff157ff4/Query%20tuning%20and%20solving%20N+1%20using%20QueryDsl.md) <br>
[Resolving OutOfMemory caused by JPA hibernate Query Plan Cache - from my blog](https://github.com/recordsbeat/resume-eng/blob/a275fd280ca51d9c9df37c48a34e57ebff157ff4/Resolving%20OutOfMemory%20caused%20by%20JPA%20hibernate%20Query%20Plan%20Cache.md) 

- Enhancing transaction performance in Batch application by index optimization and decrease locking duration <br>
[Handling Transaction Concurrency Using JPA - from my blog](https://github.com/recordsbeat/resume-eng/blob/a275fd280ca51d9c9df37c48a34e57ebff157ff4/Handling%20Transaction%20Concurrency%20Using%20JPA.md) <br>

<br>


## 📝 What I have done

### Auto-Scheduler restart when it failed
- Logging result meta-data using Scheduler annotation 
- Success & fail alarm to Slack (chatOps & SRE oriented)

### MethodRouter based Spring AOP
- Implementation of strategy pattern 
- Routing method by comparing requested value to reserved value

### Domain driven oriented architecture design
- Define domain model as a POJO
- Isolate domain aggregate by package boundary
- Communication between domain package by using facade, inbound event


<br>


## 📑 What I’m working on

### Component Test within MSA
- Building testing environment with docker-compose 
- Describe test case by method fixture
- Enhance test performance by reusing container 

### Debezium(CDC)
- Establish Debezium instance in Kubernetes
- Planning database instance fail over strategy
- Adding on Kafka Connect (including debezium) management tool (Debezium UI)


<br><br>


## 👂 What I’m interested in

### Virtual thread (a.k.a project loom) 
- Lightweight thread(fiber) will set us free from thread performance so that we don’t need to deal with non-blocking operation for performance

### Shenandoah GC (a.k.a Low-pause)
- Load reference barrier enables not only concurrent marking swap but also compact(evacuation) 
- Expect it will shrink GC pause time dramatically at Server API application

[Low-Pause ! Shenandoah GC - from my blog](https://github.com/recordsbeat/resume-eng/blob/a275fd280ca51d9c9df37c48a34e57ebff157ff4/Low-Pause%20!%20Shenandoah%20GC.md) <br>

### Consumer Driven Contract Test (a.k.a Pact)
- Consumer generates Pact, so client side won't need to wait anymore until server gives Api spec(such as swagger)
- Pact broker shows commnication relcationship between each microservice
- Handling Pact's version and history easliy (living document)


