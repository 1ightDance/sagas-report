# sagas-report

想要写一个基于SAGAS理念的分布式事务框架，翻出来 1987 年的sagas论文看看，翻译DOING...  
[Princeton University Report ID: TR-070-87](https://www.cs.princeton.edu/research/techreps/TR-070-87)  
我是英语渣，有翻译错误或更好的翻译，请 issue 留言，或者给我发邮件 alex.sun.email@gmail.com 来一起完成这项工作

-----------------------

# SAGAS
Hector Garica-Molina  
Kenneth Salem  

CS-TR-070-87  
January 1987  

# SAGAS
Hector Garcia-Molina  
Kenneth Salem  

Department of Computer Science  
Princeton University  
Princeton, NJ.08544  


## 摘要

一个长时间事务会在相对较长的时间内占用数据库资源，明显的阻碍了较短的和公用的其他事务完成。为了缓解这些问题, 我们提出一个 saga的概念。它是由多个有序的事务组成、并且与其他事务可以交错的一个长时间事务（LLT），数据库管理系统保证成功完成 saga 中的所有事务, 或对部分进行事务补偿。saga的概念 和它的实施相对简单, 但它们有可能显著提高性能。我们分析了与 sagas 相关的各种实施问题，包括如何在不直接支持它们的现有系统上运行它们。我们进行了数据库和 LLT技术讨论, 使 sagas成为LLT解决方案的可能。

1987年1月7日
 

>## ABSTRACT
>
>Long lived transactions (LLTS) hold on to database resources for relatively long periods of time, significantly delaying the termination of shorter and more common transactions. To alleviate these problems we propose the notion of a saga. A LLT is a saga if it can be written as a sequence of transactions that can be interleaved with other transactions. The database management system guarantees that either all the transactions in a saga are successfully completed or compensating transactions are run to amend a partial execution. Both the concept of saga and its implementation are relatively simple but they have the potential to improve performance significantly. We analyze the various implementation issues related to sagas, including how they can be run on an existing system that does not directly support them. We also discuss techniques for database and LLT design that make it feasible to break up LLTs into sagas.
>
>January 7, 1987
 

## SAGAS
Hector Garcia-Molina  
Kenneth Salem  

Department of Computer Science  
Princeton University  
Princeton, N.J. 08544  

## 1 简介

顾名思义，一个执行长生命周期的事务，即使没有其他事务的干扰，也需要大量的时间，可能需要数小时或数天。一个长生命周期事务，或者说 LLT，与大多数其他事务相比, 持续时间较长, 因为它访问了许多数据库对象，它有很长的计算过程，或因用户输入的停顿，或者多种因素的组合。举一个TTL的例子，根据银行交易记录生成每月账目报表, 用来处理保险公司的索赔，这个事务需要收集整个数据库的信息[Gray81a]。

>## 1 INTRODUCTION

>As its name indicates, a long lived transaction is a transaction whose execution, even without interference from other transactions, takes a substantial amount of time, possibly on the order of hours or days. A long lived transaction, or LLT, has a long duration compared to the majority of other transactions either because it accesses many database objects, it has lengthy computations, it pauses for inputs from the users, or a combination of these factors. Examples of LLTs are transactions to produce monthly account statements at a bank transactions to process claims at an insurance company, and transactions to collect statistics over an entire database [Gray81a].

在大多数情况下, LLT存在严重的性能问题。由于它们是事务, 系统必须将它们作为原子操作执行, 从而保持数据库的一致性[date81a, ullm82a]。为了保持事务的原子性，系统通常会锁定事务访问的对象，直到事务它提交，而这通常发生在事务结束时。因此, 试图访问 LLT 锁住的对象的其他事务会被阻塞很长时间。

>In most cases, LLTs present serious performance problems. Since they are transactions, the system must execute them as atomic actions, thus preserving the consistency of the database [Date81a, Ullm82a]. To make a transaction atomic, the system usually locks the objects accessed by the transaction until it commits, and this typically occurs at the end of the transaction. As a consequence, other transactions wishing to access the LLT's objects are delayed for a substantial amount of time.

此外，LLTs可能还会导致事务终止的几率上升。正如 [Gray81b] 中所说的，死锁的频率对于事务的“大小”是非常敏感的，也就是说, 事务访问的对象数。(在 [Gray81b] 的分析中, 死锁频率随事务大小的四倍而增长)。因此, 由于 LLTs 访问了许多对象, 它们可能会导致许多死锁, 相应地，也会有许多中断。从系统崩溃的角度来看，LLTs遇到故障的概率较高 (因为它们的持续时间), 因此更有可能遇到更多的延迟, 更有可能自己被中止。

>Furthermore, the transaction abort rate can also be increased by LLTS. As discussed in [Gray81b], the frequency of deadlock is very sensitive to the "size" of transactions, that is, to how many objects transactions access. (In the analysis of [Gray81b] the deadlock frequency grows with the fourth power of the transaction size.) Hence, since LLTS access many objects, they may cause many deadlocks, and correspondingly, many abortions. From the point of view of system crashes, LLTS have a higher probability of encountering a failure (because of their duration), and are thus more likely to encounter yet more delays and more likely to be aborted themselves.

一般来说，没有解决办法可以消除 LLT带来的问题。即使我们使用不同的锁机制来确保 LLTs的原子性, 长时间的延迟和/高中止率仍然会存在：无论什么机制, 当一个事务需要访问的对象们，已经被 LLT 访问了，那么直到 LLT 提交这个事务才能被提交。

>In general there is no solution that eliminates the problems of LLTs. Even if we use a mechanism different from locking to ensure atomicity of the LLTS, the long delays and/or the high abort rate will remain: No matter how the mechanism operates, a transaction that needs to access the objects that were accessed by a LLT cannot commit until the LLT commits.

但是, 对于特定的应用程序, 或许可以通过放宽 LLT原子性要求，来缓解这些问题。换言之, 在不牺牲数据库一致性的情况下, 某些 LLT可以在完成之前释放其某些资源, 从而允许其他等待资源的事务可以继续进行。

>However, for specific applications it may be possible to alleviate the problems by relaxing the requirement that an LLT be executed as an atomic action. In other words, without sacrificing the consistency of the database, it may be possible for certain LLTs to release their resources before they complete, thus permitting other waiting transactions to proceed.

为了阐释这个想法, 请思考航空公司的订票系统。这个数据库（或这可能实际上是来自不同航空公司的数据库集合）包含航班预定，并且这个事务 T 希望有多个预定。对于本讨论, 让我们假设事务T 是一个 LLT (比如说, 每次预订后, 客户都会暂停输入)。在这个应用程序中，在T 完成之前可能并不需要保留其所有的资源,。例如，T在 F1 上保留一个座位后，它可以立即允许其他事务在同一架飞机上预定座位。换句话说，我们可以把T看做是许多“小事务的集合“，每个小事务T1, T2, ..., Tn 就是预留各个座位。
 
>To illustrate this idea, consider an airline reservation application. The database (or actually a collection of databases from different airlines) contains reservations for flights, and a transaction T wishes to make a number of reservations. For this discussion, let us assume that T is a LLT (say it pauses for customer input after each reservation). In this application it may not be necessary for T to hold on to all of its resources until it completes. For instance, after T reserves a seat on flight F1, it could immediately allow other transactions to reserve seats on the same flight. In other words, we can view T as a collection of “sub-transactions" T1, T2, ..., Tn, that reserve the individual seats.

但是, 我们不希望将 T 简单地作为多个独立事务的集合提交到数据库中 (dbms)，因为我们仍然希望 T 是一个单元, 要么全部成功或全部失败。我们不希望事务 T 在数据库中五个座位只保留了三个，然后（由于崩溃）而什么也做不了。另一方面，我们希望DBMS 能够保证 T 将所有预定都成功，或者如果 T 必须停止，那么也将取消所有已预定座位。

>However, we do not wish to submit T to the database management system (DBMS) simply as a collection of independent transactions because we still want T to be a unit that is either successfully completed or not done at all. We would not be satisfied with a DBMS that would allow T to reserve three out of five seats and then (due to a crash) do nothing more. On the other hand, we would be satisfied with a DBMS that guaranteed that T would make all of its reservations, or would cancel any reservations made if T had to be suspended.

这个例子表明了一种控制机制，可以不那么严格的执行事务的原子性，但仍然提供了一些保证措施，保证LLT是可以实现的。在本文中，我们将提出一个这样的机制。

>This example shows that a control mechanism that is less rigid than the conventional atomic transaction ones but still offers some guarantees regarding the execution of the components of an LLT would be useful. In this paper we will present such a mechanism. 



让我们用 saga 这一词，来形容一个 LLT ，这个 LLT 可以被分解为多个子事务的集合，这些子事务也可以和其他事务交织在一起。在这种情况下，每个子事务都是一个真正的事务，因为它保留了数据库的一致性。但是与其他事务不同的地方，对于 saga 中的事务彼此相关，并且要作为一个（非原子）单元去执行： ~当saga的任意部分如果发生了意外，那么必须要进行补偿。~

>Let us use the term saga to refer to a LLT that can be broken up into a collection of sub-transactions that can be interleaved in any way with other transactions. Each sub-transaction in this case is a real transaction in the sense that it preserves database consistency. However, unlike other transactions, the transactions in a saga are related to each other and should be executed as a (non-atomic) unit: any partial executions of the saga are undesirable, and if they occur, must be compensated for.


~若要对发生意外的部分进行修正，则每个事务 Ti 需要提供一个补偿事务 Ci。这个补偿的撤销操作，从语义上来看，补偿事务可以取消 Ti 所执行的任何操作，但不一定要在数据库层面返回到 Ti 执行前的状态。~ 在我们上面举的航公公司订票例子，如果 Ti 是在航班上预订座位，那么 Ci 可以取消这个预定（减少一个预留座位并进行一些检查）。但是 Ci 不能简单的只记录一个座位数量，因为在 Ti 预订座位到 Ci 取消预订座位这段时间内，还有其他事务进行着预订及取消，这可能会导致此航班座位的预订数量发生变化。

>To amend partial executions, each saga transaction Ti should be provided with a compensating transaction Ci. The compensating transaction undoes, from a semantic point of view, any of the actions performed by Ti, but does not necessarily return the database to the state that existed when the execution of Ti began. In our airline example, if Ti, reserves a seat on a flight, then Ci can cancel the reservation (say by subtracting one from the number of reservations and performing some other checks). But Ci cannot simply store in the database the number of seats that existed when Ti ran because other transactions could have run between the time Ti reserved the seat and Ci canceled the reservation, and could have changed the number of reservations for this flight.

一旦将 saga 的 T1, T2, ..., Tn 的取消补偿事务定义为 C1, C2, ..., Cn-1 ，那么 这个系统将会作出如下保证，任何一个序列  
　　　　　　　　T1, T2, ..., Tn　　
　　　　　　　　（最好是一个）或者这个序列　　
　　　　　　　　T1, T2, ..., Tj, Cj, ..., C2, C1　　
对于0 ≤ j < n 将被执行。

>Once compensating transactions C1, C2, ..., Cn-1 are defined for saga T1, T2, ..., Tn, then the system can make the following guarantee. Either the sequence
T1, T2, ..., Tn　　
>　　　　　　　　(which is the preferable one) or the sequence　　
>　　　　　　　　T1, T2, ..., Tj, Cj, ..., C2, C1　　
>　　　　　　　　for some 0 ≤ j < n will be executed.　　




