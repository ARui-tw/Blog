---
title: "[OS] Chapter 7 — Deadlocks"
date: 2022-01-31 18:07:24
tags:
    - Operating System
    - Notes
categories: Operating System Notes
---
# Deadlock Characterization

{% note default %}
[Notion 好讀版](https://www.notion.so/arui-tw/Chapter-7-Deadlocks-b345b44b6ac14b28a37bdc7bb497edb9)
{% endnote %}

<aside>
📖 程式之間彼此等待，無法進行了

</aside>

- Example
    - 2 processes and 2 tape drivers
        - 一人各拿一個，然後等對方手上那個
    - 2 processes, and semaphores A & B
        - P1 (hold B, wait A): wait(A), signal(B)
        - P2 (hold A, wait B): wait(B) , signal(A)

**Necessary Condition** → 一定要**全部**的條件**同時**發生

- Mutual exclusion — 資源是沒有共享性的（暫時）
- Hold and Wait — 會 hold resource 然後 wait 別人
- No preemptive — 不能強制別人放開 resource
- Circular wait — 一定會有一個 circular 的 wait。
    
    $P_0 \rightarrow P_1  \rightarrow P_1 \rightarrow \dots\rightarrow P_n \rightarrow P_0$
    

![](/assets/[OS]Chapter-7/Untitled.png)

<!-- more -->

# System Model

- 要有某些資源 $R_1, R_2, \dots, R_m$
    - E.g. CPU, memory pages, I/O devices
- 每個資源 $R_i$ 可能會有複數的共 $W_i$ 個 instances
    - E.g. a computer has 2 CPUs
- 每一個 process 取得 resource 的方式：
    
    Request → use → release
    

## Resource-Allocation Graph

![圓形跟正方形不能改](/assets/[OS]Chapter-7/Untitled%201.png)

- 3 processes, P1 *~ P3*
- 4 resources, R1 *~ R4*
    - R1 and R3 each has one instance
    - R2 has two instances
    - R4 has three instances
- Request edges:
    - P1 → R1: P1 requests R1 （一直存在因為 resource 正在被用）
- Assignment edges:
    - R2 → P1: 那個 instance 正在被 P2 使用

⇒ P1 is hold on an instance of R2 and waiting for an instance of R1

- If the graph contains a cycle, a deadlock may exist
    - 如果有 multiple instances，那就要檢查每一條路線都是 cycle 才行
    - Example 1
        
        ![](/assets/[OS]Chapter-7/Untitled%202.png)
        
        1. P1 is waiting for P2
        2. P2 is waiting for P3 → P1 is waiting for P3
        3. P3 is waiting for P1 or P2, and they both waiting for P3
        
        ⇒ deadlock!
        
    - Example 2
        
        ![](/assets/[OS]Chapter-7/Untitled%203.png)
        
        1. P1 is waiting for P2 or P3
        2. P3 is waiting for P1 or P4
        3. Since P2 and P4 wait no one
        
        ⇒ no deadlock

### Deadlock Detection

- 沒 cycle → 沒 deadlock
- 有 cycle
    - one instance per resource type → 有 deadlock
    - multiple instances per resource type → 可能有 deadlock

### Handling Deadlocks

- 確保不會進去
    - deadlock prevention — 確保至少一個必要條件不會發生
    - deadlock avoidance — 在 runtime 去不斷的檢查
- 讓他進去再 recover
    - deadlock detection
    - deadlock recovery
- 完全不處理（ignore the problem）
    - 讓 user 去中斷他

## Review Slides ( I )

- deadlock necessary conditions?
    - mutual exclusion
    - hold & wait
    - no preemption
    - circular wait
- resource-allocation graph?
    - cycle in RAG → deadlock?
- deadlock handling types?
    - deadlock prevention
    - deadlock avoidance
    - deadlock recovery
    - ignore the problem


# Deadlock Prevention & Deadlock Avoidance

## Deadlock Prevention

- 每一種 resource 各拿掉一種條件。（每一個可以不一樣）
    - Mutual exclusion (ME): do not require ME on sharable resources
        - E.g. set file to read only → no mutual exclusion (can many read in same time)
        - 但不是所有都可以這樣解
        - （Memory 是 support concurrent access 的，是 programer 的行為讓他有 ME 的條件出來的）
    - Hold & Wait
        - 規定他要 free 掉手上的 resource 才能去要其他 resource。或是一就要把全部的 resource 要齊（只要要等就要把拿到的 free 掉，可以等，但是手上不能有東西）。
        - All or nothing allocation
        - 去改變 mutex_wait 的實作
        
        ☹️ Resource 使用率降低，可能會有 starvation
        
    - No preemption
        - 改成可以作 preemptive
        - E.g. preemptive 的 scheduler → 有 timer 去讓 OS always control CPU
        
        ☹️ Overhead，要多一塊 memory 去暫存資料
        
        → Printer 的代價就太高了（要重印）
        
    - Circular wait
        - 在 request resource 的時候，如果你要 request $R_k$，那就要 release $R_i, \forall i \geq k$。（小於 $k$ 的可以繼續 hold）
            
            手上不能握比較大的。
            
        - Example
            - F(*tape drive*) = 1, F(*disk drive*) = 5, F(*printer*) = 12
            - 如果 hold disk，可以要 printer
            - 如果 hold printer 和 tape，那 request drive 就要 release printer，tape 可以留著
        - Proof
            - 證明有 deadlock 但是不可能成立
            - $P_0(R_0) \rightarrow R_1$($P_0$ hold $R_0$ and request $R_1$)
                
                $P_1(R_1) \rightarrow R_2, \dots, P_N(R_N) \rightarrow \red {R_0}$
                
            - 但是最後一個不可能成立，因為 $N>0$

## Avoidance Algorithms

- Single instance of a resource type
    
    → 用 resource-allocation graph (RAG) algorithm ****直接偵測 circle
    
- Multiple instances of a resource type
    
    → banker’s algorithm
    

### Resource-Allocation Graph (RAG) Algorithm

![](/assets/[OS]Chapter-7/Untitled%204.png)

- Request edges:
    - $P_i \rightarrow R_j$: $P_i$ waiting for $R_j$
- Assignment edges:
    - $R_j → P_i$: $R_j$ is allocated and held by $P_i$
    - Resource 被 release 可能轉回 claim edge
- Claim edge:
    - $P_i \rightarrow R_j$: $P_i$ may request $R_j$ **in the future** (worst case)
    - 真的發生就會變成 request edges
- Example
    - 這不會出事
        
        ![](/assets/[OS]Chapter-7/Untitled%205.png)
        
    - 當 P2 request for R1 的時候，他會被擋掉。因為我們會假設 claim 的情形會在未來 somehow 發生。
        
        ![](/assets/[OS]Chapter-7/Untitled%206.png)
        
- Cycle-detection algorithm ⇒ $O(n^2)$

### **Banker’s algorithm**

去考慮 worse case。

![](/assets/[OS]Chapter-7/Untitled%207.png)

- Safe state → 可以找到一個 safe sequence 讓程式照著跑，結束前都不會遇到 deadlock
- Unsafe state → 有機率會 deadlock
- Deadlock avoidance → 不要進到 unsafe

**Safe Sequence**

- Assume 有 12 個 tape drives
- Safe state:
    - At t0:
        
        
        |  | Max Needs | Current Holding |
        | --- | --- | --- |
        | P0 | 10 | 5 |
        | P1 | 4 | 2 |
        | P2 | 9 | 2 |
        - Max need is hinted from processes
        - Max need is worst case
        
        → <P1, P0, P2> is a safe sequence
        
        1. 給 P1 3 個（有的都給他）
            
            
            |  | Max Needs | Current Holding | Available |
            | --- | --- | --- | --- |
            | P0 | 10 | 5 |  |
            | P1 | 4 | 2 | 3 |
            | P2 | 9 | 2 |  |
        2. 給 P0 5 個
            
            
            |  | Max Needs | Current Holding | Available |
            | --- | --- | --- | --- |
            | P0 | 10 | 5 | 5 |
            | P1 | 4 | 0 | 0 |
            | P2 | 9 | 2 |  |
        3. 給 P2 10 個
            
            
            |  | Max Needs | Current Holding | Available |
            | --- | --- | --- | --- |
            | P0 | 10 | 0 |  |
            | P1 | 4 | 0 |  |
            | P2 | 9 | 2 | 10 |
- Unsafe state:
    - At t1:
        
        假設這時候 OS 拿到的一個 request 說 P2 想要多一個 tape drive。
        
        |  | Max Needs | Current Holding | Available |
        | --- | --- | --- | --- |
        | P0 | 10 | 5 |  |
        | P1 | 4 | 2 | 3 → 2 |
        | P2 | 9 | 2 → 3 |  |
        - 跑一次上面的流程，然後我們就會發現，如果我們同意了（P2 的 current holding 變成 3 了），我們就會發現他可能會 deadlock （i.e. 進到 unsafe state）。
        - 所以 OS 就會擋掉這個 request。
- Banker algorithm:
    1. 拿到 request，先假設同意了
    2. 然後去跑 safe sequence
        
        選 sequence 的順序隨便都沒差。
        
    3. 如果 safe sequence 存在，我們就 grant request，反之就不同意
        
        如果有數個，那就都可以，反正至少一個就好了
        
- Banker’s Algorithm Example 1
    
    我懶得貼.. P.27
    
- Programming Example
    
    A, B, C are semaphores initialized with the values of 2, 1, 1.
    
    用 round-robin 跑。
    
    - 原本
        
        ![](/assets/[OS]Chapter-7/Untitled%208.png)
        
    - Use avoidance
        
        ![](/assets/[OS]Chapter-7/Untitled%209.png)
        
        - P4 的 wait(A) 如果成功了，加上 claim edge 就會 dead lock，所以 OS 不會讓他拿
        - 接下來 P2 就會搶到 A，然後 eventually 他就會 signal A and C，剩下的就會成功結束了

## Review Slides (II)

- deadlock prevention methods?
    - mutual exclusion
    - hold & wait
    - no preemption
    - circular wait
- deadlock avoidance methods?
    - safe state definition?
    - safe sequence?
    - claim edge?

# Deadlock Detection & Recovery from Deadlock

- deadlock 了再解
    
    （就算卡住了，也只是 user program，OS 一定不會卡住的啦，要信任 OS）
    
- 所以就就當下的情況去跑上面那些 check
- Single instance
    
    ![](/assets/[OS]Chapter-7/Untitled%2010.png)
    
- Multiple-Instance → Banker’s algorithm
    - 可以把 Max Needs 拿掉，沒用了
    - 去檢查有沒有 safe sequence 存在，有就當作沒事。沒有就代表進入 deadlock 了。
    
    ![](/assets/[OS]Chapter-7/Untitled%2011.png)
    
- Deadlock Recovery
    
    → Process termination
    
    - 但是要 kill 誰？（可能可以 kill priority 低的）（但是一直 kill priority 低的可能會 starvation ）
    - Kill 掉之後還可以 roll back 嗎？

# Textbook Problem Set

![](/assets/[OS]Chapter-7/Untitled%2012.png)

![](/assets/[OS]Chapter-7/Untitled%2013.png)

![](/assets/[OS]Chapter-7/Untitled%2014.png)

![](/assets/[OS]Chapter-7/Untitled%2015.png)

{% note warning %}  
此筆記為清華大學周志遠教授作業系統之課堂筆記，所有內容及圖片皆取材於課堂內容。<br />
如內容有誤，歡迎來信 [mail@arui.dev](mailto:mail@arui.dev)。
{% endnote %}