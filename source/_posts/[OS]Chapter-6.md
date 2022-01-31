---
title: "[OS] Chapter 6 — Process Synchronization"
date: 2022-01-31 18:06:24
tags:
    - Operating System
    - Notes
categories: Notes
---

{% note default %}
[Notion 好讀版](https://arui-tw.notion.site/Chapter-6-Process-Synchronization-dc2fdd2951b047f78a39377759ebe90c)
{% endnote %}

# Background

- 因為有 share 的 data，會被多個 content 去用他，所以需要
- 就算只有一個 core，也會有 Concurrent 的問題（因為會有 content switch）
- 重點就是他們 execute 的 order
- Consumer & Producer Problem
    - 用 counter 去記還有幾個空位
    - Producer
        
        ```cpp
        while (1) {
        	nextItem = getItem( );
        	while (counter == BUFFER_SIZE);
        	buffer[in] = nextItem;
        	in = (in + 1) % BUFFER_SIZE;
        	counter++;
        }
        ```
        
    - Consumer
        
        ```cpp
        while (1) {
        	while (counter == 0) ;
        	item = buffer[out];
        	out = (out + 1) % BUFFER_SIZE;
        	counter--;
        }
        ```
        
    - 但是 counter 會是這兩隻 thread 共用的東東，所以會出現不一致的問題：
        - `counter++`
            
            ```cpp
            move ax, counter
            add  ax, 1
            move counter, ax
            ```
            
        - `counter--`
            
            ```cpp
            move bx, counter
            sub  bx, 1
            move counter, bx
            ```
            
        - 但是…如果是這樣呢？（counter initially is 5）
            
            ```cpp
            producer: move ax, counter   // ax = 5
            producer: add ax, 1          // ax = 6
            context switch
            consumer: move bx, counter   // bx = 5
            consumer: sub bx, 1          // bx = 4
            context switch
            producer: move counter, ax   // counter = 6
            context switch
            consumer: move counter, bx   // counter = 4
            ```
            
            不過如果不是 preemptive 就不會有這個問題了。
            

## Race Condition

<aside>
📖 多個 process 操作 shared data concurrently，結果會取決於最後完成的人。（這是描述一個問題點，所以是因為有 race condition，要用同步化解決）

</aside>

- Single processor 的可以用 disable interrupt 或 non-preemptive CPU 來解決。

<!-- more -->

# Critical Section

- ~~一個 process 之間溝通的 protocol~~
- Critical section: 會產生 race condition 的那段程式碼。
- Mutually exclusive: 會有相同 critical section 的 process。他們要在那段 critical section 被迫去 serialize。
    
    ![Remainder section: 剩下的](/assets/[OS]Chapter-6/Untitled.png)

## Critical Section Requirements

1. Mutual Exclusion: if process P is executing in its CS, no other processes can be executing in their CS
2. Progress: 有 process 想要進去空的 critical section，就應該要讓他馬上進去。（有效性）
3. Bounded Waiting: 不能讓等的人一直等，要 bound 他的 waiting time。（公平性）
- 2 和 3 是可捨取的

## Review Slide

- Race condition?
- Critical-Section (CS) problem? 4 sections?
    - entry, CS, exit, remainder
- 3 requirements for solutions to CS problems?
    - mutual exclusion
    - progress
    - bounded waiting

# Software Solution

- Assume only 2 processes, $P_0$ and $P_1$
- 用 `turn` 決定輪到誰
    
    ![](/assets/[OS]Chapter-6/Untitled%201.png)
    
    - 因為是用 single instruction 去 set，所以不會被中斷詠唱
    - Requirements
        1. Mutual Exclusion: Yes
        2. Progress: **Nope**，不符合 Progress 的條件， 因為 $P_0$ 可能已經繞一圈想上去了，但是因為 $P_1$ 還沒進過 CS，所以他進不去
        3. Bounded Waiting: Yes，因為是 Round robin.

## Peterson’s Solution for Two Processes

- 跟剛剛那個很像，但是多了一個 flag，用來表示他是不是想要進去 critical section（True 就是他準備好進去了）
- 對方可以看到別人的 flag，但是只有自己可以改自己的
- 所以進去的時候就是別人不想進去，或是你拿到 token 了（i.e. 對方沒拿到 token）
    
    ![注意是要在**進去前**把 turn 值給對方](/assets/[OS]Chapter-6/Untitled%202.png)
    

### Proof

- Mutual exclusion: 用反證
    1. 假設他們可以同時在裡面
    2. 那下面的條件就要同時存在。
        
        ![](/assets/[OS]Chapter-6/Untitled%203.png)
        
    3. 因為他們都在裡面，所以 flag 都是 true
    4. 但是 turn 不可能同時是 0 且 1
    5. 所以不可能同時進去
- Progress
    1. 假設 $P_0$ 想要進去，且 $P_1$ 不想要進去，要證這時候 $P_0$ 應該要要可以進去 ⇒ 好像也不用證，反正就可以啊
    2. 兩個都想進去的時候，那就看 `turn`，一定有其中一個可以進去
- Bounded Waiting
    - 假設當 $P_1$ 想要進去，但是 $P_0$ 在裡面。
    1. $P_0$ 一離開 CS 就把 `flag` 設成 False 了，所以馬上就輪到 $P_1$ 了
    2. $P_0$ 如果一離開 CS 就又想進去（可能因為剩下的都是 CPU burst，所以就沒有被 content switch 掉）
        
        ⇒ 但是因為會在最一開始把 token 給別人，所以會被讓給別人作
        

## Producer/Consumer Problem

- 如果先進 Consumer 就會 deadlock（i.e. 程式沒有在前進，但是 CPU 在燒）。
    
    ![](/assets/[OS]Chapter-6/Untitled%204.png)
    
- 正確但是低效能。
    
    ![](/assets/[OS]Chapter-6/Untitled%205.png)
    
- 正確且優質。
    
    ![](/assets/[OS]Chapter-6/Untitled%206.png)
    

## Bakery Algorithm (n processes)

- 像去麵包店
1. 在每次進去 CS 前，要抽號碼牌
2. 號碼牌最小的先進（因為 counter 是用 `++` 的，所以**只能**保證是 non-decrease order 的，e.g. 1, 2, 3, 3, 4, 4, 4, 5）
3. 如果號碼牌一樣，就根據 process ID，小的先

```cpp
// Process i
do {
	choosing[i] = true;  // 開始抽
	num[i] = max(num[0], num[1], ..., num[n-1]) + 1; // get ticket
	choosing[i] = false; // 抽完了
	for (j = 0; j < n; j++) {
		while (choosing[j]);  // 確定他抽完了才可以比
		while ((num[j] != 0) &&     // num 是 0 就是不想進去
					 ((num[j], j) < (num[i], i))); // 比較每一個 j 確定我是最小的
	}
		**critical section**
	num[i] = 0;
		**reminder section**
} while(1);
```

- 最小號碼一定先做 → first come first serve → bounded-waiting time
- 如果沒有 choosing
    1. 假設目前最大是 5
    2. P4 抽的比較快，抽到 6，這時候 P1 還是 0
    3. 然後 P4 會進去 CS（他這時候會以為 P1 沒有要進去，因為他是 0）
    4. 接下來 P1 抽完了，抽到 6，他就去跟 P4 比，發現跟他一樣，但是他的 pid 比較小，所以他就開開心心的進去了。然後就沒有然後了。(race condition by355)
    - 如果有 locking，P4 就會等到 P1 先抽完再跟他比

# Synchronization Hardware

只要用到 Hardware 的就算是（包括之前的 disable interrupt）

## Atomic instruction

一個 CPU 的 instruction，所以一定會一次做完。

不需要 library 的 support，內建就有了。

### `TestAndSet()`

```cpp
boolean TestAndSet (bool &lock) {
	bool value = lock;
	lock = TRUE;   // 不論結果，都會鎖起來
	return value;  // return 的是原本的值
}
```

![](/assets/[OS]Chapter-6/Untitled%207.png)

- Mutual exclusion: Yes
    
    ⇒ 因為只有正在 CS 裡面的人把他設成 Flase 才可以放下一個進去
    
- Progress: Yes
    
    ⇒ 只要沒人在裡面，想進去就可以馬上進去
    
- Bounded-Wait: No
    
    ⇒ 因為先搶先贏，所以可能一直搶不到
    
    ⇒ P0 的 CS 結束了，但是 CPU burst 還沒完，所以他又搶到 lock，然後進去之後 CPU burst 結束了，content switch。換 P1，但是他被擋在外面，CPU burst 燒完之後都輪不到他 QQ，又回到 P0 繼續。
    

### `Swap()`

把值對調。

![一開始會把 key 設成 True，然後就在 while 裡面一直把 lock 跟 key 交換。如果 lock 在別的地方被設成 False 了，就可以進去 CS 了。](/assets/[OS]Chapter-6/Untitled%208.png)

- 特性也跟剛剛一樣

## Review Slide (2)

- Use software solution to solve CS?
    - Peterson’s and Bakery algorithms
- Use HW support to solve CS?
    - `TestAndSet()`, `Swap()`

# Semaphores

- Resource counting 的問題。
- 一個值 indicate 想要透過 semaphore 保護的資源的數量多寡。
    - If #record = 1 → binary semaphore, mutex lock
    - if #record > 1 → counting semaphore
- 用兩個 atomic operation：
    1. `wait`：搶資源
        
        ```cpp
        wait(S) {
        	while (S <= 0); // busy waiting
        }
        ```
        
        用的是 Spinlock → 等待還是會燒 CPU（因為一直在作指令）
        
    2. `signal`：放資源
        
        ```cpp
        signal(S) {
        	S++;
        }
        ```
        

## POSIX Semaphore

- 他是 OS 的 API
- Semaphore is part of POSIX standard BUT it is not belonged to Pthread
    - It can be used with or **without** thread
- 因為 lock 多是在 threads 之間，semaphore 多是在 process 之間。
- POSIX Semaphore routines:
    
    ```cpp
    sem_init(sem_t *sem, int pshared, unsigned int value)
    sem_wait(sem_t *sem)
    sem_post(sem_t *sem)
    sem_getvalue(sem_t *sem, int *valptr) // check
    sem_destory(sem_t *sem)
    ```
    
- Example:
    
    ```cpp
    #include <semaphore.h>
    sem_t sem;
    sem_init(&sem);
    sem_wait(&sem);
    	// critical section
    sem_post(&sem);
    sem_destory(&sem);
    ```
    
- Another example
    
    ```cpp
    do {
    	wait(mutex);   // pthread_mutex_lock(&mutex)
    		// critical section
    	signal(mutex); // pthread_mutex_unlock(&mutex)
    		// remainder section
    } while(1);
    ```
    
    - Progress: Yes
    - Bounded waiting: 取決於 `wait` 的實作

## Non-busy waiting Implementation

<aside>
✏️ Waiting 的應該要被 put to sleep（所以有要要去 wake up 他）

</aside>

- Semaphore is data struct with a queue （可以用各種  queue strategy）
    
    ```cpp
    typedef struct {
    	int value;        // init to 0
    	struct process *L;
    	// “PCB” queue 
    } semaphore;
    ```
    
    ![](/assets/[OS]Chapter-6/Untitled%209.png)
    
- `wait()` and `signal()`
    - 需要用到 `block()` and `wakeup()` 兩個 system calls。（明顯的缺點，需要多用到 system call）
    
    ```cpp
    void signal(semaphore S) { 
    	S.value++;
    	if (S.value <= 0) {         // 注意有 `=`，因為我們先 `++` 了 
    		remove a process P from S.L; 
    		wakeup(P);
    	}
    }
    ```
    
    ```cpp
    void wait(semaphore S) {
    	S.value--;  // subtract first
    	if (S.value < 0) {         // 注意沒有 `=`
    		add this process to S.L; // 要先放進去才能去睡覺
    		block();
    	}
    }
    ```
    
- 更多行加上更多 share data（那些 queue），所以要確保他有 atomic 的特性。可以用：
    - Single-processor: disable interrupts
    - Multi-processor:
        - HW support (e.g. Test-And-Set, Swap)
        - SW solution (Peterson’s solution, Bakery algorithm)
    
    ![注意不要包到 sleep 和 wakeup，不然會 deadlock。](/assets/[OS]Chapter-6/Untitled%2010.png)
    
- 如果等待時間短，不如用 busy waiting，不然要一直 call system call。

## Cooperation Synchronization

因為會有資料的 independency，所以也可以用這個來解。

- Example:
    - P1 executes S1; P2 executes S2 (S2 be executed only after S1 has completed)
    - `sync` 一開始設成 0
        
        ![](/assets/[OS]Chapter-6/Untitled%2011.png)
        
- A More Complicated Example (DAG)
    
    ![](/assets/[OS]Chapter-6/Untitled%2012.png)
    
    - (Initially, all semaphores are 0)
    
    ![](/assets/[OS]Chapter-6/Untitled%2013.png)
    

## Deadlocks & Starvation

- Deadlocks: 2 processes are waiting indefinitely for each other to release resources（程式之間彼此互卡）
- Starvation:
    - example: LIFO (Last In First Out) queue in semaphore process queue
    - 有人可以執行只是輪不到我

## Review Slide (3)

- What’s semaphore? 2 operations?
- What’s busy-waiting (Spinlock) semaphore?
- What’s non-busy-waiting (Non-Spinlock) semaphore?
- How to ensure atomic wait & signal ops?
- Deadlock? starvation?

# Classical Problems of Synchronization

解決方法很多，重點是去定義問題。

- Bounded-Buffer (Producer-Consumer) Problem
- Reader-Writers Problem
- Dining-Philosopher Problem

## Bounded-Buffer Problem

就上面那些問題。

- A pool of *n* buffers, each capable of holding one item
- Producer:
    - grab an empty buffer
    - place an item into the buffer
    - waits if no empty buffer is available
- Consumer:
    - grab a buffer and retracts the item
    - place the buffer back to the free pool
    - waits if all buffers are empty

## Readers-Writers Problem

- reader processes → 只會讀
- writer processes → 只會寫
- 如果都是 reader 那沒差，反正同時 read 也不會出事。但是 writer 就只能一個人寫，連 reader 都不能同時 read。（Exclusive access）
- Different variations involving priority
    - **First RW problem**: no reader will be kept waiting unless a writer is updating a shared object （只要符合上面的基本定義就好了）
    - **Second RW problem**: first 的條件加上 writer 不能 starvation。
        
        → Writer 要比較高的 priority
        
        → once a writer is ready, no new reader may start reading
        

### First Reader-Writer Algorithm

```cpp
// mutual exclusion for write
semaphore wrt = 1;

// mutual exclusion for readcount
semaphore mutex=1  // 保護 readcount
int readcount=0;   // 有多少人要 read
```

```cpp
Writer() { 
	while(true) {
		wait(wrt);
			**// Writer Code**
		signal(wrt);
	}
}
```

```cpp
Reader() {
	while(true) {
		wait(mutex);
		
		readcount++;
		// Acquire write lock if reads haven’t
		// 全部 reader 共用一個 write lock
		if (readcount == 1) // 代表原本沒有人在 read
			wait(wrt);        // 所以需要要過來

		signal(mutex);

			**// Reader Code**

		wait(mutex);

		readcount--;
		// release write lock if no more reads
		if (readcount == 0)  // 代表沒有人要 read 了
			signal(wrt);       // 就可以放掉了

		signal(mutex);
	}
}
```

- Writer 會有 starvation，因為 reader 共用一個 lock，所以可以一直輪流佔著
    
    ⇒ Second RW problem solve it.
    

### Dining-Philosophers Problem

![](/assets/[OS]Chapter-6/Untitled%2014.png)

- 每個人左邊跟右邊都有一根筷子（五個人五隻筷子）
    - 吃飯需要兩隻筷子
    - 一隻筷子只能一個人拿
    - 吃完就要把筷子放下
- Deadlock problem
    - 每個人拿左邊的筷子直到吃完飯 → 卡死
- Starvation problem
    - 不能讓人一直拿，一直吃

# Monitors

<aside>
💡 動機：因為前面很難，要在對的地方用對的東西，不能多，不能少。

</aside>

- 目的是要更抽象化（他是在 language level）
- 跟 OO 很像
- 像是由一個 class 定義的東西：有自己的 methods、variables、private variables。
- （Private variables 只能由自己的 methods 操作）
- 和 OO 不同的是他會確保他的 process 只會有一個在 **active** (running) 的狀態
- Schematic View
****
    
    ![](/assets/[OS]Chapter-6/Untitled%2015.png)
    
    - Shared data → private variable
    - Operations → methods
    - Initialization code → constructor
    - Entry queue → 想要用 monitors 的傢伙

## (Monitor) Condition Variables

- 他是一個獨立的同步化 tool，跟 monitor 沒關係
- Event driven
- To allow a process to wait **within** the monitor, a condition variable must be declared, as
    
    `condition x, y;`
    
- Can be used with `wait()` and `signal()` （但是跟上面的那些毫無關係，請不要搞混，定義完全不同）
    - `x.wait()`
        
        有一個 event queue，所以這像是 enqueue。（上面的那個是 counting）
        
    - `x.signal()`
        
        像是 dequeue，把裡面的 process 叫醒去工作。如果裡面沒有東西，就啥都不會發生。（如果是 semaphore，值會繼續加上去）
        

![](/assets/[OS]Chapter-6/Untitled%2016.png)

- Dining Philosophers Example
    - Thinking — 根本沒要吃
    - 假設大家都是看得到彼此的狀態的（i.e. 有 Global information）
    
    ```cpp
    monitor dp {
    	enum {thinking, hungry, eating} state[5]; // current state
    	condition self[5]; // delay eating if can’t obtain chopsticks
    	void pickup (int i)    // pickup chopsticks
    	void putdown (int i)   // putdown chopsticks
    	void test (int i)      // try to eat
    	void init() {
    		for (int i = 0; i < 5; i++) 
    			state[i] = thinking;
    	}
    }
    ```
    
    ```cpp
     void pickup(int i) {
    	state[i] = hungry;
    	test(i);               // try to eat (to prevent deadlock)
    	if (state[i] != eating)  // 沒成功吃到
    		self[i].wait();        // wait to eat
    }
    ```
    
    ```cpp
    void putdown(int i) {
      state[i] = thinking;
    
    	// check if neighbors are waiting to eat
    	// 叫他們起來吃飯
    	test((i+4) % 5);
    	test((i+1) % 5);
    }
    ```
    
    ```cpp
    void test(int i) {
    	if ((state[(i + 4) % 5] != eating) 
    		&& (state[(i + 1) % 5] != eating) // 左邊和右邊有沒有在吃
    		&& (state[i] == hungry)) {        // 和我想不想吃
    		// No neighbors are eating and Pi is hungry
    		state[i] = eating;
    
    		// 把他自己叫起（如果是 pickup call 的，這是沒有用的
    		// 但是如果是 putdown call 的，那就會搖醒睡著的）
    		self[i].signal(); 
    	}
    }
    ```
    
    - An illustration
        
        ![](/assets/[OS]Chapter-6/Untitled%2017.png)
        
        ![](/assets/[OS]Chapter-6/Untitled%2018.png)
        
        ![](/assets/[OS]Chapter-6/Untitled%2019.png)
        
        ![P2 test 失敗，所以去 waiting](/assets/[OS]Chapter-6/Untitled%2020.png)
        
        ![](/assets/[OS]Chapter-6/Untitled%2021.png)
        
        ![P2 被叫醒了](/assets/[OS]Chapter-6/Untitled%2022.png)
        
        ![](/assets/[OS]Chapter-6/Untitled%2023.png)
        

# Thread Programming

（考試比較不會考）

## Pthread Lock/Mutex Routines

To use mutex:

- Declared as of type `pthread_mutex_t`
- Initialized with `pthread_mutex_init()`
- Destroyed with `pthread_mutex_destory()`
- Use `pthread_mutex_lock()` and `pthread_mutex_unlock()`

```cpp
#include “pthread.h”
pthread_mutex mutex;
pthread_mutex_init (&mutex, NULL);  // NULL -> optional configuration
pthread_mutex_lock(&mutex);
	**// Critical Section**
pthread_mutex_unlock(&mutex);
pthread_mutex_destory(&mutex);
```

## Condition Variables (CV)

跟上面的一樣。

- In Pthread, CV type is a `pthread_cond_t`
    - Use `pthread_cond_init()` ****to initialize
    - `pthread_cond_wait (&theCV, &somelock)`
    - `pthread_cond_signal (&theCV)`
    - `pthread_cond_broadcast (&theCV)` → 把 queue 裡面全部的 thread 都叫醒（i.e. 對他們全部作 `signal`）
- Example
    - 一個 thread 在 x = 0 的時候做事
    - 另一個負責 x--
    
    ![](/assets/[OS]Chapter-6/Untitled%2024.png)
    
    - 一定要把 wait 和 signal 在 mutex 被 lock 的時候才能用（不然連 code 都不會跑）
        1. 因為 call wait 和 signal 的時候，大多會和其他 conditional variable 有相依性（In this case: `x`），所以 lock 起來才是合理的。
        2. 要確保 `cond` 不會有 concurrent write 的問題。
        
        ⇒ 所以至少要包到他本身和他的觸發條件。
        
    - What really happens...
        1. `action`: Lock `mutex`
            
            ![](/assets/[OS]Chapter-6/Untitled%2025.png)
            
        2. `action` call `wait`, put thread into sleep and **release the lock**. ⇒ 因為 `wait` 會直接 release `mutex` （所以不是下面那行做的，是 `wait` 就直接 release 掉了！）
            
            `counter` lock `mutex`
            
            ![](/assets/[OS]Chapter-6/Untitled%2026.png)
            
        3. `conter` go into CS and call `signal`.
            
            `action` is waked up, but thread is locked. （`unlock` 會先呼叫 `lock` 確定可以跑，但顯然不行，因為 lock 還在 counter 那邊 ）
            
            ![](/assets/[OS]Chapter-6/Untitled%2027.png)
            
        4. `counter` call unlock.
            
            `action` Re-acquire lock and resume execution
            
            ![](/assets/[OS]Chapter-6/Untitled%2028.png)
            
        5. `action` release lock
            
            ![](/assets/[OS]Chapter-6/Untitled%2029.png)
            
        - 另一個我們之所以要用 lock 包住的原因。

## ThreadPool Implementation

- Task structure
    
    ![](/assets/[OS]Chapter-6/Untitled%2030.png)
    
    - 存 function pointer 跟 argument
- ThreadPool structure
    
    ![](/assets/[OS]Chapter-6/Untitled%2031.png)
    
    - `thread_count` — 紀錄一共有多少 threads
    - `*thread` — threadPool
    - `head` — `*queue` 的 head
    - `tail` — `*queue` 的 tail
    - `shutdown` — threadPool 是不是要結束掉了（決定要不要 break while loop）
- Allocate thread and task queue
    
    ![](/assets/[OS]Chapter-6/Untitled%2032.png)
    
- Implement
    
    ![注意有個 `mutex_lock`
    也會去檢查 `count` 是不是 0，如果是，那目前沒有工作，先睡覺（notify is our condition variable）](/assets/[OS]Chapter-6/Untitled%2033.png)
    
    ![有事情做了，或是被 wake up 起來了，就可以做事了。 
    最後是 call function pointer。（注意是要放在 CS 外面，這樣才有平行度）](/assets/[OS]Chapter-6/Untitled%2034.png)
    

## Synchronized Tools in JAVA

- Synchronized Methods (Monitor)
    
    ```java
    public class SynchronizedCounter { 
    	private int c = 0;
    	public synchronized void increment() { c++; }
    	public synchronized void decrement() { c--; }
    	public synchronized int value() { return c; }
    }
    ```
    
    加上 synchronized 就可以保證只有一個才操作。所以可以看到有動到 `c` 的都有加。如果沒有動到 share data 的，可以不用加，增加平行度。
    
- Synchronized Statement (Mutex Lock)
    
    ```java
    public void run()
    {
    	synchronized(p1)
    	{
    		int i = 10; // statement without locking requirement
    		p1.display(s1);
    	}
    }
    ```
    

## Review Slides (4)

- Bounded-buffer problem?
- Reader-Writer problem?
- Dining Philosopher problem?
- What is monitor and why need monitor?

# Atomic Transactions

（考試不太會提）

- System Model
    - Transaction: 一系列的 operation
    - Atomic Transaction: 需要一次完成 transaction，all or nothing
    - DB 很容易需要
- File I/O Example
    - Transaction 是一連串的 read and write
    - 像是 version control 的工具，需要有 commit （成功）、abort（失敗）、rolled back（失敗後要 undo）
    - 所以重點是要有 log
        - 因為有 log 才能回推重建
        - 也叫做 checkpoint
        - 存在 secondary storage
    - 也會有 checkpoints
        - 還原點，上一個 commit 的點
        - 說不定會直接不 undo，然後 apply checkpoint

## Review Slides (5)

- What is atomic transaction?
- Purpose of commit, abort, rolled-back?
- How to use log and checkpoints?

# Textbook Problem Set

![](/assets/[OS]Chapter-6/Untitled%2035.png)

![](/assets/[OS]Chapter-6/Untitled%2036.png)

{% note warning %}  
此筆記為清華大學周志遠教授作業系統之課堂筆記，所有內容及圖片皆取材於課堂內容。<br />
如內容有誤，歡迎來信 [mail@arui.dev](mailto:mail@arui.dev)。
{% endnote %}