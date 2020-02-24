# Process Scheduling

[TOC]



## Process Scheduler란?

다음번에 실행될 `process`를 선택하는 kernel component 로써, 실행할 process에게 유한한 `processor(CPU) `시간을 분배한다. 

1. 어떤 process를 실행할 것인가?
2. 시스템 성능 최적화 (e.g. load balancing) 
3. 여러 process가 동시에 실행되는 것 처럼 보이도록 동작



이러한 scheduler는 리눅스와 같은 **multitasking 운영체제**의 기본 요소이다.

> Multitasking 운영체제에는 cooperative multitasking과 preemptive multitasking 두가지가 있다.
>
> **Cooperative multitasking**: process가 자발적으로 실행 중지, 특정 process가 processor를 독점할 수 있음.
>
> **Preemptive multitasking**: scheduler가 process의 실행/중지를 결정, process의 processor 독점을 막음.



**Scheduler의 목표: 빠른 응답 시간 + 높은 process 처리량**



## Policy

### Process의 priority

각 process가 processor의 시간을 얼마나 받을 가치가 있는지, processor를 얼마나 오래 사용해야 하는지에 따라 priority를 결정한다.

priority 기반의 policy는 다음과 같은 특징이 있다.

1. 높은 우선순위의 작업을 우선적으로 수행한다. 같은 우선순위 안에서는 `RR(round-robin)` 으로 실행된다.
2. priority는 시간에 따라 동적으로 달라질 수 있다.
3. 리눅스에서는 priority를 nice값으로 표현한다. (-20 <= nice < 20, 기본 값은 0)



### Time-slice

어떤 process(task)가 선점(preemptive)되기 전까지 실행되는 값(시간)이다.

Scheduler는 time-slice 값을 정해야 하는데, 그 값이 너무 크면 사용자가 느끼기에 응용 프로그램들이 동시에 실행되는 것 처럼 느껴지지 않는다. 반대로 그 값이 너무 작다면 작업을 switching 하기 위한 overhead가 커진다. (e.g. cache)

그렇기 때문에, time-slice 값을 task의 종류에 따라 다르게 설정한다. **이때, 한번에 time-slice 값을 모두 소비할 필요는 없다.** 즉, 100ms를 5번에 나누어 사용해도 상관없다. 기본적으로 interactive한 작업의 time-slice 값을 크게 한다. 



#### 실제 상황에서의 scheduling policy

2개의 process를 가정하자. 

1. text editor (interactive task, I/O-based task)
2. video encoder (processor-based task)

encoder 작업의 경우, 지금 시작하든 0.5초 뒤에 시작하던 사용자가 구별하지 못할 것 이다. 

editor의 경우, 사용자의 반응에 즉각 반응해야 한다. (= interactive task)

그러므로 editor에 높은 priority와 긴 time-slice를 제공한다. 이를 통해, editor는 사용자의 반응에 즉각 반응할 수 있다. 하지만 사용자의 입력 시간이 길지 않으므로,  test editor는 매우 짧은 시간만 실행되고 남는 시간에 video encoder작업을 수행할 수 있다.



## Scheduling algorithm

리눅스 scheduler는 kernel/sched.c에 정의되어 있다. scheduling algorithm과 관련된 코드는 리눅스 2.5부터 재작성되었다. 새로 작성된 algorithm은 다음과 같은 목표를 가지고 설계되었다.

1. O(1) scheduling: 실행중인 process의 수, 입력의 크기 등에 무관하게 상수 시간 안에 scheduling 할 수 있어야 한다. 

2. SMP (대칭형 multiprocessing) 확장성: 모든 processor는 자신만의 `lock`과 `run queue`를 갖고 있어야 한다.

3. SMP와의 상성 개선: 

   Task와 특정 CPU를 연계한다. 즉, 한 CPU에서 실행된 task는 계속 그 CPU에서 실행된다. (오직 run queue의 load balancer를 통해서만 task 이동)

   Interactive한 작업의 성능을 보장해야 한다. 즉, 시스템의 부하가 많아도 interactive한 task에 대해 즉각적인 반응을 보여야한다.



### Run queue

Processor에 있는 실행 가능한 process의 목록을 저장하고 있는 자료구조로써, 모든 processor에 하나씩 존재한다. 이때, 실행 가능한 각각의 process는 하나의 run queue에만 속할 수 있다.

Run queue를 .h가 아니라 kernel/sched.c에 정의함으로써, kernel의 다른 부분에 scheduler의 본체를 숨기고 실요한 interface만 노출한다.

아래는 run queue 자료 구조의 일부이다. 

```c++
struct runqueue{
    spinlock_t			lock;
    ...
    struct prio_array	*active;	// 활성 우선순위 배열을 가리키는 포인터
    struct prio_array	*expired;	// 비활성 우선순위 배열을 가리키는 포인터
    ...
}
```



#### struct prio_array

이 때, 활성/비활성 priority 배열을 가리키는 포인터의 자료구조인  `struct prio_array`은 O(1) scheduling의 핵심이다. `struct prio_array` 자료 구조는 아래와 같다.

```c++
struct prio_array{
    int					nr_active;				// task의 개수
    unsigned long		bitmap[BITMAP_SIZE];	// 우선순위 비트맵
    struct list_head 	queue[MAX_PRIO];		// 우선순위 queue
}
```

`MAX_PRIO`는 priority level의 개수이며, 기본값은 140이다.

`bitmap`의 각 bit는 하나의 priority level을 의미한다. 즉, 140개의 priority level이 있고 word의 크기가 32bit라면 `BITMAP_SIZE`는 5이다. (4 * 32 < 140 <= 5 * 32) 만약 특정 priority의 task가 running 가능하다면(`TASK_RUNNING`), 해당 level에 해당하는 bit는 1이 된다. (priority가 7인 task가 runnable하다면, bitmap의 7번째 bit가 1이다.) 

즉, processor에서 가장 priority가 높은 task를 찾는 일은, bitmap에서 가장 먼저 1인 bit를 찾는 일과 같다. System의 priority level이 정해져 있으므로, 실행 가능한 process 갯수에 영향 받지 않고 실행할 process를 찾을 수 있다. 또한 리눅스는 bitmap에서 bit가 1인 bit를 빠르게 찾아낼 수 있는 알고리즘을 제공한다.

같은 priority안에 있는 task들은 RR 방식으로 scheduling 된다.

 

#### Time-slice 재계산

Task의 time-slice가 0이 되면, priority와 time-slice를 다시 계산해야한다. 하지만, time-slice가 0이 되는 모든 작업에 대해 재 연산을 수행하는 것은 overhead가 클 뿐만 아니라, (task가 n개 라면, O(n)만큼의 시간이 필요)  lock overhead도 매우 크다.

이러한 문제를 해결하기 위해, time-slice가 0이 된 작업의  priority와 time-slice를 다시 계산하는 대신, task를 active 상태에서 expired 상태로 바꿔준다. 이 작업은 포인터만 교환하면 되므로, 매우 간단하다.

task는 expired queue에서 priority와 time-slice를 다시 계산한다.



### schedule()

Task의 최초 priority는 사용자에 의해 정해진다. (기본값은 0 이다.)

Scheduler는 bitmap에서 가장 먼저 1로 설정된 bit를 찾아, 해당 bit에 해당하는 priority task를 실행시킨다. task의 time-slice가 0이 되어,  priority와 time-slice를 다시 계산할 때 **interactive한 task의 priority를 높이고, time-slice를 늘린다**.

이때, 리눅스 scheduler는 Interactive한 작업을 찾기 위해 heuristics algorithm을 사용한다. scheduler는 실행 시간에 비해 대기 시간이 긴 task를 interactive task라고 판단한다. 이를 위해, 모든 task는 실행 시간 대비 대기 시간을 `sleep_avg`변수에 저장한다. task가 running 상태일 때는 `sleep_avg--`를 수행하고, task가 waiting 상태일 떄는 대기한 시간 만큼 `sleep_avg`의 값을 증가 시킨다. 즉, `sleep_avg`의 값이 클수록 interactive한 task라고 인식된다.

만약, 작업이 충분히 interactive 하다면, time-slice가 0이 되었을 때 task를 expired 상태로 바꾸지 않고 계속 active상태로 유지한다.



### Load balancer

Load balancer는 run queue 사이의 균형을 맞춰준다. load balancer는 현재 실행중인 processor의 run queue와 다른 시스템의 run queue를 비교한다. 만약 균형이 맞지 않다면, 바쁜 run queue에서 process를 빼와 현재 processor의 run queue에 삽입한다.

다른 processor의 run queue에서 process를 빼올 때, 다음의 우선 순위를 따른다.

1. 현재 expired 상태에 있는 task (상대적으로 긴 시간 수행되지 않았기 때문에, process cache가 없을 가능성이 큼)
2. priority가 높은 task (priority가 높은 작업을 공평하게 분배하는 일이 중요하기 때문)





## CFS?

container 할 때... to be continue...