
## Shenandoah ?

Shenandoah GC is a Garbage Collector developed by Red Hat and released in OpenJDK 12. It is called **low-pause GC** , 
and the concept is 'It is better to **perform a small GC operation several times** than to perform a large GC operation a few times'. Shenandoahs guarantees concurrency to perform small-scale GC frequently. This means that the GC will reduce the pause (Stop-The-World) time at the cost of more CPU usage.


### Support

Another next-generation low-pause GC is Oracle's ZGC. Although it has almost the same mechanism as Shenandoah,
**Shenandoah appeared in OpenJdk 12 and supported backporting up to jdk11 and jdk8** . On the other hand, ZGC only supports jdk11 or higher versions.

> **JDK Support**
![](https://images.velog.io/images/recordsbeat/post/547f4b1d-6528-46d5-a8d2-f1a57e535a32/image.png)
https://wiki.openjdk.java.net/display/shenandoah/Main
The document marked openJdk12 upstream, but for some reason, the JDK 12 version is not supported as of the date of writing (2021.07.13) for some reason.

### Performance

Shenandoah coolly solved the issues of Fragmentation of the existing CMS and Stop-The-World of G1. With strong concurrency and light GC logic, **it takes a certain pause time without being affected by the heap size .**
Let's see how we solved it below.


![](https://images.velog.io/images/recordsbeat/post/3c3dc1cb-1eef-4ea8-ae31-3fc9f557ecbe/image.png)
_( Latency graph by GC implementation method. As you can see, it is very fast. )_

### Single-Generational
As mentioned above, the Shenandoah is based on the G1.
Divide Jvm heap by region and perform mark-sweep & evacuation.
However, unlike G1, Shenandoah does not divide the area into Generations.


> ![](https://images.velog.io/images/recordsbeat/post/efd025ba-33c7-42e8-9070-fc03a6163380/image.png)
"Everyone who is familiar with GC says that generational GC is the best. 'The generational hypothesis predicts that young objects are more likely to die than old objects in most situations.' That was true in 2008. But now it's less convincing."
Red Hat Developer Christine Flood - DevNation 2016 中
Shenandoah GC: Java Without The Garbage Collection Hiccups (Christine Flood)
[Shenandoah GC: Java Without The Garbage Collection Hiccups (Christine Flood)](https://www.youtube.com/watch?v=N0JTvyCxiv8)


![](https://images.velog.io/images/recordsbeat/post/c1d8abd1-9191-48a3-879e-4f057d6f3cb6/image.png)

_(You can also see Shenandoah VS G1 BenchMark, Shenandoah shows overwhelming performance.)_

The area is divided according to generation, and memory copy continues to occur during the evacuation process. Shenandoah said that he would rather eliminate the cost of generational copies because the generational hypothesis is no longer valid.
So Shenandoah is **Single-Generational .**



> **Generational Shenandoah**
Generation Shenandoah came into existence for reasons such as sustainable throughput, load spike resiliency, and improved memory utilization.
It seems that the most recent update is on 2021.07.12, so it is under development.
Still, the Single-Generational Shenandoah is maintained.
"The goal is not to make Generational mandatory. The option to use a single-generation Shenandoah without compromising performance or functionality **remains** ."
JEP 404: Generational Shenandoah


### Low-Pause

![](https://images.velog.io/images/recordsbeat/post/aa2ae474-c151-421a-b4d8-58863d392cc0/image.png)

Let's look at the picture above.
There are low-pause representatives ZGC and Shenandoah, and these two show a clear difference from previous GCs. As mentioned earlier, the pause caused by copying from Young Generation -> Old Generation has already been removed. In addition, the pause in the compact stage, which could not be solved in the existing G1, was also solved by the concurrent compact. In the end , **all the advantages of Mark-Sweep of CMS and Compact of G1 were implemented concurrently to take advantage of them .**


### GC Cycle
![](https://images.velog.io/images/recordsbeat/post/31092e23-6d20-4e6f-bb6d-c729bc740379/image.png)
```

GC(3) Pause Init Mark 0.771ms
GC(3) Concurrent marking 76480M->77212M(102400M) 633.213ms
GC(3) Pause Final Mark 1.821ms
GC(3) Concurrent cleanup 77224M->66592M(102400M) 3.112ms
GC(3) Concurrent evacuation 66592M->75640M(102400M) 405.312ms
GC(3) Pause Init Update Refs 0.084ms
GC(3) Concurrent update references  75700M->76424M(102400M) 354.341ms
GC(3) Pause Final Update Refs 0.409ms
GC(3) Concurrent cleanup 76244M->56620M(102400M) 12.242ms
```

1. **Init Mark** starts concurrent marking. The heap and threads prepare for concurrent marking and examine the next root set. This is the **first pause** between cycles , and the root set scan is the biggest load. So **the size of the root set determines** the pause time.

2. **Concurrent Makring tracks Reachable objects** across the Heap . This step **runs without stopping the application , and the amount of time it takes depends on the number of live objects and the heap object graph structure**. At this stage, applications are free to allocate new data, increasing the heap share between concurrent machining.

3. **Final Mark** flushes all pending marking/update queues and rediscovers the root set to complete concurrent marking. Also , **it identifies the movable region, initializes Evacuation and moves some roots** . It usually prepares the runtime for the next step. Some of this work can be done concurrently in the Concurrent Pre-cleaning phase. This causes a **second pause between cycles. Most of the time is spent on emptying the queue and doing root scanning** .

4. **Concurrent Cleanup** immediately flushes the **garbage area , i.e. the area with no live objects after concurrent mark** .

5. **Concurrent Evacuation copies objects** out of a collection set to another region . Since this step is executed again with the application, the application can freely allocate memory, which is **the biggest difference from other GCs** . The collection set size setting determines the required time . _(Collection Set (CSet): A data structure in which the region where GC is to be performed is stored)_

6. **Init Update Refs** initializes the Update References phase. It does almost nothing other than checking that all GCs, threads have exited evcuation and preparing for the next step. This is the **third pause** in the cycle and **the shortest.**

7. **Concurrent Update References update concurrently evacuated Object references** across the heap . This is also **the biggest difference** from GC. **The time required depends on the number of objects in the heap**, but the object graph structure is not affected because the heap is continuously searched. This step runs concurrently with the application.

8. **Concurrent Cleanup** reclaims **regions that have no references** .

---

## Concurrency

As such, Shenandoah aims to execute the GC algorithm in the background at the same time as the application is executed. However, it seems that it will not be easy to guarantee heap concurrency between GC algorithms. Shenandoah solved this problem as follows.



### Snapshot at the beginning (SATB)

SATB is a method that enables concurrent marking. This method is also used in CMS and G1, but Red Hat developer Christine Flood says that the code used in G1 is directly used as it is.


![](https://images.velog.io/images/recordsbeat/post/ae1bde70-a6d3-43fa-b58b-db38fbaf4490/image.png)

Basically, the marking algorithm divides objects into three colors.
1. White: An object that has not yet been reached
2. Gray: An object that has been reached but the reference content of the object has not been searched
3. Black: An object that has been reached and the reference content has been searched for


![](https://images.velog.io/images/recordsbeat/post/725d953f-48ce-42dd-9537-1d137eac39e1/image.png)

Let's look at the above situation.
The grayed out object has been reached and reference scanning is in progress.
But what if you delete the grayed out object from the application and change it to refer to the black object?

![](https://images.velog.io/images/recordsbeat/post/91db1609-e9e8-462c-8495-a13ce2177d37/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-07-15%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%203.44.51.png)
Or if we change the reference of the object to be referenced (circled in red) so that the black object directly refers to it as above?

![](https://images.velog.io/images/recordsbeat/post/69b53a96-c755-40af-b16a-edb7370b75ac/image.png)

If left as it is, the black objects marked in the graph above will survive, and the white objects will be removed by GC. It looks like it's completely ruined.

To prevent this, Snapshot at the beginning is necessary.
Literally, it is to take a snapshot of the object graph for the first time, and compare the graph after marking with the graph before marking to compensate for the above situation.

> **How does SATB work? - How does SATB work between Markings by Shenandoah developer Roman Kennke**
We use write barriers.
This is a separate hook that is called whenever the mutator thread writes an object's reference field. (Not to be confused with memory fences, also called barriers otherwise).
The write barrier writes the original contents to the linked list before the reference is modified. When all concurrent marking steps are completed and Stop The World occurs, all objects in the linked-list are marked and SATB-list is created. When all previous work is finished and the application resumes operation, you can check that all objects are marked as live at the start of the marking phase. Also, by counting the number of live objects during marking, it is possible to know how many objects exist in each region.
[Shenandoah GC: Concurrent parallel marking](https://rkennke.wordpress.com/2013/06/19/shenandoah-gc-concurrent-parallel-marking/) 


Additional marking is carried out with the information of the SATB created in this way.

![](https://images.velog.io/images/recordsbeat/post/757bb07d-c71b-4154-8b2c-98a5bdb18487/image.png)
First all references removed are grayed out.

![](https://images.velog.io/images/recordsbeat/post/0c9f32b1-98fe-4c00-a17b-459f95982a04/image.png)
And all newly allocated objects are painted black. As shown in the figure above, you can check that all references are connected normally.

> **Learn more about SATB - Write Barriers in G1 GC - [Write Barriers in G1 GC](https://www.jfokus.se/jfokus17/preso/Write-Barriers-in-Garbage-First-Garbage-Collector.pdf)**
![](https://images.velog.io/images/recordsbeat/post/33844190-0a8c-4ec1-ac8d-8f9f2cd0e835/image.png)



### Load Reference Barrier (LRB)

Shenandoah's Concurrent Evacuation and Concurrent Update References steps are said to be the biggest differences between other GCs. It is the Load Reference Barrier that makes this possible. **It can be seen as a pointer that allows the change to be maintained consistently** when moving an object to a new region between evacuations .

_( Load Reference Barrier is an integrated improvement of the Read & Write barrier function in Shenandoah 1.0. Improvements will be discussed in detail below. )_


![](https://images.velog.io/images/recordsbeat/post/2c19bc13-6db8-49ac-8095-62a62edf9c40/image.png)

Existing Evacutaion has the same conccrency problem as above.
There is an old object that occurs in from space (previously allocated region area) and a new object that is copied to space (newly allocated region region).

When looking at the four squares at the top of the figure as root objects of the thread stack, the two root objects on the right were safely updated to space. On the other hand, the two root objects on the left have not been able to perform reference update.

At this moment, thread A changes the value of the from space area, and thread B changes the value of the to space area. **In this case, the value of y=4 is lost** when reference update of all root objects is completed .


![](https://images.velog.io/images/recordsbeat/post/3c5f415e-c8e3-4012-93db-dc63a39c0d9f/image.png)

The concept is simple. When an object copy occurs in the to space area, the from space area object header faces the to space area. When a copy occurs, the old object is no longer an object, but a pointer.

This ensures that no matter which thread is **referencing & changing the to space, it will see the to space** .



### Shenandoah 1.0 -> 2.0

We talked about LRB like a simple story pointer, but if you look inside, you can see that it is divided into separate areas.


> ![](https://images.velog.io/images/recordsbeat/post/17c63630-d92b-4a51-b815-03ce8197c8cc/image.png)
This is the object layout of HopSpot jvm. A noteworthy point is the Mark Word, where various information such as hashcode, GC marking, and thread lock are stored.


#### Shenandoah 1.0

![](https://images.velog.io/images/recordsbeat/post/fe9223bf-f7b3-4835-90f2-f38fd65a0d80/image.png)

In Shenandoah 1.0, complex barriers such as Read, Write, and Equal existed before LRB. To implement this, a forward pointer is placed in front of the object header. And for every reference, it moves 8 bytes back from the mark word point and refers to the field pointed to by the forward pointer. Of course, most go to a mark word that points to itself. Because of this forward pointer, there was an 
issue of 50% in total, generally 5~10% in the worst case.

![](https://images.velog.io/images/recordsbeat/post/f26a3167-c630-46ea-801a-e9995478d8dd/image.png)

Also, re-referencing did not mean that concurrency could be resolved unconditionally. In the case of a read barrier, there is a case where the old object can be referred to as the worst case.


#### Shenandoah 2.0


![](https://images.velog.io/images/recordsbeat/post/548723ca-e515-46a7-8545-651f342144ba/image.png)

In the previous version of Shenandoah 1.0, it had problems such as the need for **additional memory allocation** due to the forward pointer , **a complicated barrier** mechanism, and **insufficient optimization** .


![](https://images.velog.io/images/recordsbeat/post/14f5b38d-69ff-49e1-a53f-973e0e916c99/image.png)

Shenandoah 2.0 simply solved the problem of 1.0, which is the concept that when an object is copied to to space, **the old object is no longer used** . Therefore, the mark word of the old object was replaced with a forward pointer.
In version 1.0, it checks whether the copied object exists in the to space while re-referencing itself every time, but now **it only checks whether the forward pointer exists** . This is the **Load Reference Barrier** . This ensures that all references unconditionally point to the object of to space.
_(If the same object as GC and application is copied, the object copied by the application is treated as to space)_


![](https://images.velog.io/images/recordsbeat/post/23120221-c027-4a1d-bede-4e2fd1861b0e/image.png)

It is no longer necessary to use barriers such as read and write with a simple forward pointer reference.

![](https://images.velog.io/images/recordsbeat/post/6d2d095d-0206-42d6-861a-3876cd0820ce/image.png)


Naturally, additional memory allocation was resolved, and fewer GC cycles were performed for the same amount of allocation. This confirms that all the problems in Shenandoah 1.0 have been resolved.


> **For detailed differences between Shenandoah 1.0 and 2.0, you can refer to the two videos below.**
[Shenandoah 2.0 (2020.08.06)](https://www.youtube.com/watch?v=o4Qf-uMrDBI)
[[VDT19] Concurrent Garbage Collectors: ZGC & Shenandoah by Simone Bordet [IT](2019.10.12)](https://www.youtube.com/watch?v=WU_mqNBEacw)


---

## Benchmark

see here!
> https://ionutbalosin.com/2019/12/jvm-garbage-collectors-benchmarks-report-19-12/

---

## 마치며

I got to know Shenandoah GC while studying GC, and there are not many Korean documents compared to CMS or G1, so I wrote this after doing some research.
There is a sense of disappointment because the explanations may be very poor compared to the research material, but I think you will be able to understand it easily if you refer to the contents attached below slowly.

In the process of acquiring the content, there may be parts that I did not understand well. We would appreciate it if you could leave a comment on this subject.

---

## Rereferneces
### OpenJDK
> 
https://wiki.openjdk.java.net/display/shenandoah/Main
https://openjdk.java.net/jeps/404
https://openjdk.java.net/jeps/189


### Shenandoah GC 2.0
> [Shenandoah 2.0 (2020.08.06)](https://www.youtube.com/watch?v=o4Qf-uMrDBI)

> [MoreVMs’20: “Shenandoah GC 2.0” by Roman Kennke (2020. 4. 15)](https://youtu.be/mViMtD0DeTs)

> [OpenJDK_in_the_new_Age_of_concurrent_Garbage_Collectors.pdf (2019.11)](https://jaxlondon.com/wp-content/uploads/2019/11/OpenJDK_-_in_the_new_Age_of_concurrent_Garbage_Collectors.pdf)

> Page  : [[VDT19] Concurrent Garbage Collectors: ZGC & Shenandoah by Simone Bordet [IT](2019.10.12)](https://www.youtube.com/watch?v=WU_mqNBEacw)
Download  : [Simone_Bordet_Concurrent_Garbage_collectors_ZGC__Shenandoah.pdf](https://assets.ctfassets.net/oxjq45e8ilak/709UsobBpBGHxaZ0z6MNvH/1d75677b26f1b7c9a71150c372645ad8/100746_367617808_Simone_Bordet_Concurrent_Garbage_collectors_ZGC__Shenandoah.pdf)

> [Shenandoah GC Version 2.0 (2019): The Great Revolution](https://shipilev.net/talks/jugbb-Sep2019-shenandoah.pdf)


### Shenandoah GC 1.0

> [Shenandoah GC - Part I: The Garbage Collector That Could (2018)](https://player.vimeo.com/video/289626122)
Download : [Shenandoah GC - Part I: The Garbage Collector That Could](https://shipilev.net/talks/javazone-Sep2018-shenandoah.pdf)

> [Shenandoah GC: The Next Generation(2018. 10. 24)](https://www.youtube.com/watch?v=E1M3hNlhQCg)
Download : [Shenandoah GC - Part I: The Garbage Collector That Could](https://shipilev.net/talks/javazone-Sep2018-shenandoah.pdf)

> [Shenandoah: The Garbage Collector That Could by Aleksey Shipilev (2017. 11. 10)](https://www.youtube.com/watch?v=VCeHkcwfF9Q)
Download : [Shenandoah GC - Part I: The Garbage Collector That Could](https://shipilev.net/talks/devoxx-Nov2017-shenandoah.pdf)

> [Shenandoah 2 0: Now That We’ve Gotten the GC Pause Times Under Control, What’s Next? (2017. 10. 3)](https://youtu.be/AAiB3fDwyRM)

> [Shenandoah GC: Java Without The Garbage Collection Hiccups (Christine Flood) (2016. 10. 18)](https://www.youtube.com/watch?v=N0JTvyCxiv8)


### Etc
> 
https://dev-punxism.tistory.com/entry/Shenandoah-gc
https://dzone.com/articles/java-garbage-collection-3
https://ichi.pro/ko/7-gaji-yuhyeong-ui-java-gabiji-sujibgi-108403385011842
https://www.baeldung.com/jvm-experimental-garbage-collectors
