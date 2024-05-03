---
title: /WIC/week7/pintos-threads
author: eunsik-kim
date: 2024-05-01 17:00:00 +0900
categories: [jungle, WIC]
tags: code, concurrency
render_with_liquid: false
---

[정글끝까지-chapter1-threads] 열심히 coding한 내용을 잊지 않기 위해 기록으로 남기겠습니다!

### Pintos kaist

pintos는 간단한(?) 80x86 architecture기반의 operating system입니다. 정말정말 다양한 기능이 구현이 되어 있습니다. OS안에서
kernel threads, loading, runing user program, 그리고 file system까지의 내용을 다룹니다. 각 topic은 1 project에 대응되며 세부 목표들로 이루어져 있습니다.   


저희는 stanford에서 만든 pintos를 kaist 권영진 교수님께서 수정한 버전으로 과제를 진행하였습니다.
advanced scheduler의 [Kaist](https://casys-kaist.github.io/pintos-kaist/project1/advanced_scheduler.html)과제 설명서를 본다면 [Stanford](https://web.stanford.edu/class/cs140/projects/pintos/pintos_2.html#SEC15)의 버전에 비해 굉장히 친절하게 설명이 되어 있음을 알 수 있습니다. 과제 설명서만으로도 많은 부분을 이해하고 구현할 수 있도록 해주신 권영진 교수님께 감사의 말씀을 전해드립니다.(최고)

> 가상환경을 "source `./activate`{: .filepath}." 명령어를 통해 실행 할 수 있습니다.   
("code `~/.bashrc`{: .filepath}."에 적으면 더 좋습니다.) `threads/build`{: .filepath}. 폴더에서 "pintos [opt] -- [opt] run [test name]" 명령어를 통해 test를 실행 할 수 있습니다. `threads`{: .filepath}. 폴더에서 make를 하면 testcode 전체를 compile 할 수 있고 make-check를 통해 모든 test를 실행할 수 있습니다.(굉장히 오래걸립니다.)  
{: .prompt-info }

### Threads

이번주는 threads에 대해 다뤘습니다. proxy server 과제를 하며 이해했다고 생각했던 threads는 정말 user level의 간단한 부분만을 이해했음을 직접 구현이 되어있는 thread.c code를 보며 알 수 있었습니다. thread join과 같이 많은 부분을 다루지 않았지만 context switching을 assembly로 구현된 부분은 정말 신기했습니다. 하나도 이해가 안되었지만 실제로 asm을 사용한것을 처음보았기에 asm에 대해 공부가 필요함을 다시한번 느낄 수 있었습니다.. (이부분도 교수님께서 추가하신 부분이라고 합니다.)

```
static void
thread_launch (struct thread *th) {
	uint64_t tf_cur = (uint64_t) &running_thread ()->tf;
	uint64_t tf = (uint64_t) &th->tf;
	ASSERT (intr_get_level () == INTR_OFF);
    __asm __volatile (
                /* Store registers that will be used. */
                "push %%rax\n"
                "push %%rbx\n"
                "push %%rcx\n"
                /* Fetch input once */
                "movq %0, %%rax\n"
                "movq %1, %%rcx\n"
                "movq %%r15, 0(%%rax)\n"
                "movq %%r14, 8(%%rax)\n"
                "movq %%r13, 16(%%rax)\n"
                "movq %%r12, 24(%%rax)\n"
                "movq %%r11, 32(%%rax)\n"
                "movq %%r10, 40(%%rax)\n"
                "movq %%r9, 48(%%rax)\n"
                ....
    }
```
{: file='threads/threads.c'}

#### Alarm-clock

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/alarm-multiple/devices/timer.c#L90)

첫번째 과제는 timer-sleep함수내의 `busy-waiting` 방식을 개선하는것입니다. 단순한 방법을 적용하면 해결 할 수 있습니다. 그 과정 속에서 timer.c의 코드와 thread.c의 ready list의 코드를 이해할 수 있기에 좋은 에피타이저였습니다. timer.c는 특히 calibrate함수에서 1초 미만의 시간의 정밀도를 tick으로 계산하는 과정이 인상깊었고 program 내의 시간인 tick을 interrupt로 측량하는 방법도 자세히 이해는 못하였지만 신기하였습니다. 


| fucntion          | summary             
| :---------------- | :-------------------------------: |
| thread_block      | running thread를 block 상태로 만듬 |
| thread_unblock    | block thread를 ready 상태로 만들고 ready list에 넣음 |
| thread_yield      | runnging thread를 ready 상태로 만들고 ready list에 넣음 |
| thread_create     | 새로운 thread를 만들어 ready list에 넣음 |

thread.c의 주요 함수를 훝어보면 위와 같습니다. 기존의 함수들은 상태 변화에 초점을 맞춰 함수를 만들었습니다. 
그렇기에 원본 코드를 따르도록 노력하였습니다. 가령 ready list에 삽입할 경우 unblock과 yield함수를 통해서만 삽입할 수 있도록 하였습니다.

```
/* Suspends execution for approximately TICKS timer ticks. */
void
timer_sleep (int64_t ticks) {
	int64_t start = timer_ticks ();

	ASSERT (intr_get_level () == INTR_ON);
	while (timer_elapsed (start) < ticks)
		thread_yield ();
}
```
{: file='devices/timer.c'}

위에서 thread_yield()를 저희가 개선해야 될 부분입니다. 직관적으로 생각해보면 yield를 호출할 이유가 없습니다. sleep은 일정시간을 대기해야 하므로 thread_block()으로 대체하면 쉽게 해결 할 수 있습니다. 심지어 block된 list를 저장하기 위해 `doubly linked list` 자료형 역시 구현이 되어 있어 sleep하는 thread 저장 공간 역시 쉽게 만들 수 있었습니다.


| fucntion          | summary             
| :---------------- | :-------------------------------: |
| record_sleeptick  | 자야되는 tick 시간을 저장 후 sleep list에 삽입 |
| thread_wake       | 매 시각, 기상 시간이 지난 sleep list의 thread을 찾아 ready list에 삽입 |

추가로 두가지 함수 [record_sleeptick(ticks)](https://github.com/eunsik-kim/pintos11/blob/eunsik/alarm-multiple/threads/thread.c#L597)와 thread_wake(ticks)를 구현하여 해결하였습니다. list_insert_ordered의 존재를 나중에 알게되었지만 순회하는 방식을 통해 오름 차순으로 삽입 하여 inturrupt시 소모되는 시간을 단축하였습니다. (기상 여부를 timer inturrupt 안에서 측정하게 하였습니다.)

#### Priority scheduling

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/prioriy-scheduling/threads/synch.c)


> 역시 악명이 높은 pintos 과제를 시작하였음을 알게하는 과제였습니다. 복잡성은 이전의 rb-tree를 고려하면 비슷한 난이도 였지만 직접 thread간의 발생하는 모든 경우를 생각하기엔 너무 많아 어려웠습니다. test code가 없었다면 결코 모든 예외나 상황을 고려하지 못했을것 입니다. (물론 test case에도 없는 예외도 있습니다.) 이 목표를 통해 동기화 과정의 내부 구조를 배울 수 있어서 좋았습니다. (semaphore가 신이다...)

`priority scheduling`은 priority가 높은 thread를 가장 먼저 실행하면 되는 간단하고 직관적인 greedy한 방법입니다. 새롭게 ready list에 높은 priority를 가지는 thread가 등장하거나 thread_create를 통해 생성, thread_set_priority 함수를 통해 상승하는 상황에 따라 현재 running thread가 변경됩니다. 그래서 blocked thread가 아닌 ready list만 고려하면 됩니다. 현재 깨어있는 thread가 최강 thread가 되는, 현실에서도 볼수 있는 상황입니다.

```text
// list insert 와 pop 은 list func을 통해 간단히 할 수 있습니다.
list_insert_ordered(&ready_list, &curr->elem, priority_larger, READY_LIST); 
list_entry (list_pop_front (&ready_list), struct thread, elem);
```

하지만 blocked thread 때문에 문제가 발생할 수 있습니다. 특히 lock에 의해 block이 되면 상황에서 발생하는 `priority inversion`(낮은 우선순위의 thread의 lock에 의해 block되어 자기보다 낮은 우선순위에 밀리는 현상)을 막기 위한 `priority donation`과 `priority return`을 구현하는게 어려웠습니다.

multiple donation > nested donation > donation-chain 순서로 test code과 .ck파일을 분석하여서 덕분에 모든 상황을 이해 할 수 있었습니다. donation-chain에 발생하는 상황을 간단하게 보면 아래와 같습니다.

```
void
test_priority_donate_chain (void) 
{
  int i;  
  struct lock locks[NESTING_DEPTH - 1];
  struct lock_pair lock_pairs[NESTING_DEPTH];

  lock_acquire (&locks[0]);

  for (i = 1; i < NESTING_DEPTH; i++)
    {
      int thread_priority;

      thread_priority = PRI_MIN + i * 3;
      lock_pairs[i].first = i < NESTING_DEPTH - 1 ? locks + i: NULL;
      lock_pairs[i].second = locks + i - 1;

      thread_create (name, thread_priority, donor_thread_func, lock_pairs + i);
      thread_create (name, thread_priority - 1, interloper_thread_func, NULL);
    }
  lock_release (&locks[0]); // 7번부터 priority return이 시작되는 부분

}

donor_thread_func (void *locks_) 
{
  struct lock_pair *locks = locks_;

  if (locks->first)
    lock_acquire (locks->first);

  lock_acquire (locks->second);     // block이 되며 priority donation 하는 부분
  lock_release (locks->second);

  if (locks->first)
    lock_release (locks->first);

}
```
{: file='tests/threads/priority-donation-chain.c'}

1. priority가 가장 낮은 main thread가 lock 0을 얻습니다.
2. main thread가 점진적으로 priority가 높아지는 thread를 create합니다. 
3. 생성된 thread는 자신보다 낮은 thread에 의해 block이 됩니다. (1번 > main, 2번 > 1번 ... 7번 > 6번)
4. 각 thread는 priority donation을 받지만 0 main의 thread에 의해 block이 됩니다.
5. main thread가 lock를 해제하면 순차적으로 lock을 해제 하고 blocked됩니다. (lock1 > lock2 > ... > lock7)
6. 7번 thread부터 1번까지 donation return을 순차적으로 하며 종료합니다.

> Trial and error : 위의 case를 해결하기 위해 많은 시간을 쏟았습니다. 하지만 해결을 못하였고 다른 easy case를 해결하다 우연히 해결되었습니다. donate가 없는 파일 부터 순차적으로 해결을 하면 복잡성이 짙은 case를 debugging하는데 에너지를 아낄 수 있습니다. 심지어 가장 좋은 점은 개념을 실례를 통해 쉽게 이해할 수 있다는 사실입니다. (역시 TDD)

| fucntion          | summary             
| :---------------- | :-------------------------------: |
| priority_larger   | list_insert_ordered 에 사용된 list 정렬 기준값 비교 함수 |
| priority_donation | 재귀적으로 우선순위를 해제할 lock을 가진 대상에게 전달 |
| thread_reschedule | donation 후 thread를 list에서 위치를 재정렬하는 함수|
| donation_withdraw | 자기가 기부한 donation을 회수하는 함수 |
| print_list        | debugging용 list traverse하며 thread info 출력 함수(사용 주의) |

크게 4가지의 함수를 통해 우선순위 정렬, 우선순위 기부, 우선순위 회수를 구현할 수 있었습니다. test case를 고려하여 필수 정보를 저장하였고 donation과 withdraw 함수에서 사용하였습니다.

- multiple donation : 한 대상에게 여러 donation 이 오기 때문에 대상별 donation을 기억해야 합니다.
- nested donation : lock 별로 한사람에게 lock release할 수 있기 때문에 lock 별로 donation이 중첩되면 안됩니다.
- donation chain : 연쇄적으로 donation이 전달되며 중첩되는 상황에서 값을 정확히 회수 할 수 있도록 기부자나 수혜자는 기억해야합니다.

따라서 donation connection을 특정짓기 위해 donation시 발생하는 모든 정보를 기록하였습니다. donation한 사람 기준으로 1.1 donation시 막히게된 lock 목록, 1.2 donation한 우선순위 목록 그리고 donation 받은 사람기준으로 2.1 기부한 thread 목록, 2.2 기부받은 값 목록 각각에 대한 정보를 array로 구성하여 활용하였습니다. array로 구성하였기에 삽입과 삭제가 불편하여 trial and error가 많이 존재하였습니다. (list를 사용해 구현하는게 훨신 쉽습니다.) 간단하게 priority_donation의 구현을 보면 다음과 같습니다.

```
/* 현재 blocked holder의 우선순위가 낮다면 priority donation이 발생함 */
void priority_donation(struct lock *lock, struct thread *donator)
{	
	struct thread *beneficiary = lock->holder;
	if (beneficiary->priority < donator->priority)
	{	
		int idx;	// 같은 lock에 대해 donation한 save priority를 찾아 지움
		for (idx =0; idx<=beneficiary->nested_depth; idx++)
			if (lock == beneficiary->saved_lock[idx]) {
				if (idx == beneficiary->nested_depth)
					beneficiary->priority = beneficiary->saved_priority[idx];	// origin priority 값을 다시 불러옴
				for (; idx < beneficiary->nested_depth; idx++) {
					beneficiary->saved_priority[idx+1] = beneficiary->saved_priority[idx+2];
					beneficiary->saved_lock[idx] = beneficiary->saved_lock[idx+1];
				}
				beneficiary->nested_depth--;
				break;
			} 
		beneficiary->saved_priority[++beneficiary->nested_depth] = beneficiary->priority;	 // 기부 받기전 우선순위 저장
		beneficiary->saved_lock[beneficiary->nested_depth] = lock;					// 기부 받은 lock 저장
		donator->beneficiary_list[++donator->donation_depth] = beneficiary;			// donation return을 위해 후원자 목록 저장
		donator->donation_list[donator->donation_depth] = donator->priority;		// donation return을 위해 후원한 우선순위 값 저장
		beneficiary->priority = donator->priority;			        // 우선순위 기부
											
		// ready list에 넣어 마지막 수혜자부터 순서대로 lock_release 시도
		if ((beneficiary->nested_depth >= MAX_DONATION_LEVEL) || 
			(beneficiary->blocked_lock == NULL)) 
		{	// 우선순위 크기순으로 내림차순 재정렬 
			if (beneficiary->status == THREAD_BLOCKED) {
				list_remove(&beneficiary->elem);
				thread_unblock(beneficiary);
			}
			else if (beneficiary->status == THREAD_READY)
				thread_readylist_reorder(beneficiary);
			return thread_yield();		
		}
		else // MAX_DONATION_LEVEL 초과 혹은 lock-holder가 없을 때 까지, 재귀적으로 전달
		{
			thread_reschedule(beneficiary);
			priority_donation(beneficiary->blocked_lock, beneficiary);
		}
	}	
}
```
{: file='threads/synch.c'}
연쇄 기부를 위해 재귀를 사용하였습니다. 마지막 beneficiary가 running하도록 block되기 전 yield하였고 그 thread가 순차적으로 lock_release하며 top donator의 lock을 해제할 수 있도록 lock_release을 수정 하였습니다. 그 과정에서 4가지의 정보를 저장하는 방식으로 인해 thread의 struct의 크기가 불필요하게 증가하는 문제가 발생 하였습니다. 가독성을 위해 array를 pop하기 위해 traverse 과정을 모듈화하지 못한것이 아쉽습니다. (전반적으로 설계가 부족한 코드라고 생각이 됩니다.)

#### Advanced Scheduler

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/advanced-scheduler/threads/thread.c)

> 권영진 교수님께서 친절하게 설명서를 적어주신 덕분에 부동소수점 macro와 priority 수식을 쉽게 이해함과 동시에 구현을 할 수 있었습니다. (감사합니다!!) 이름은 더어려워 보이지만 priority scheduler보다 간결하다고 생각됩니다.

Scheduler의 종류중 하나인 MLFQ(multi level feedback queue)에 대해 구현을 하는 과제입니다. 수식 계산으로 우선순위를 조절하기 때문에 간단하게 구현을 할 수 있고 우선순위 scheduler와 비교를 하면 starvation과 response time의 문제점을 개선한 방법론입니다.
크게 3가지 공식을 구현을 목표로 합니다.

> priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)  

우선 순위는 4tick마다 갱신이 됩니다. 각 thread가 얼마나 남들에게 잘보이는지(nice), 얼마나 최근에 사용했는지(recent_cpu)에 따라 우선순위가 바뀝니다.

> recent_cpu = (2 * load_avg) / (2 * load_avg + 1) * recent_cpu + nice  

최근에 많이 사용한경우 우선순위가 내려갑니다. 각 thread는 1초당 recent_cpu 값이 1씩 증가합니다.

> load_avg = (59/60) * load_avg + (1/60) * ready_threads

평균적으로 ready list에서 대기하는 thread수가 많아짐에 따라 우선순위가 내려갑니다. 

즉, 종합해보면 최근에 많이 사용하며, 대기하는 thread가 많을 수록 priority는 내려가고 사용자가 지정한 nice 값에 따라 양보를 많이해 우선순위가 내려가게 됩니다. 이 후 우선순위 값이 변경됨에 따라 가장 높은 값을 가지는 우선순위를 먼저 실행하면 됩니다. 추가적으로 recent_cpu와 load_avg는 실수 값이므로 해당 내용을 macro로 구현한뒤 코드를 작성하면 됩니다. 

> 저희 pintos는 복소수 계산을 지원하지 않습니다. 복소수 계산을 하는 register 값을 저장과 이동을 thread launch시 적용하지 않기 때문입니다. 그래서 실수값을 표현하기 위해서는 부동소수점 17.14 표현법을 따라 정수로 표현하여 사용하면 됩니다.


| fucntion          | summary             
| :---------------- | :-------------------------------: |
| mlfq_cal_priority   | 4tick마다 우선순위를 계산하는 함수 |
| thread_cal_recent_cpu | 100tick마다 recent_cpu를 계산하는 함수 |
| thread_cal_load_avg | 100tick마다 load_avg를 계산하는 함수|
| thread_set_nice | nice값을 갱신하는 함수 |
| mlfq_scheduler  | 일정 tick 값에 따라 위의 값들을 갱신하는 함수 |

그래서 우선순위를 계산하기 위한 함수들을 생성하면 간단하게 구현할 수 있습니다. 다만 수식이 timer_intrrupt시에 발생하기 때문에 계산을 느리게 하면 Timeout error 혹은 알수없는 interrupt 에러가 발생할 수 있으므로 주의해야 합니다.

> 수식을 정확히 따라 적고, 정확한 시점에 초기화와 갱신을 하면 문제없이 통과를 할 수 있지만, 실수를 한다면 찾기가 매우 어렵습니다.  왜냐하면 test가 전부 성능 test이기 때문입니다. 그래서 코드를 작성하실 때 신중하게 작성하시길 바랍니다.  
혹시 TEST 시에 무한 loop에 빠지거나 insert시 발생하는 에러는 대부분 (경험상) list 중복삽입 문제로 추측이 되니 함수 호출 전 확인 한번 더 하시길 바랍니다.
{: .prompt-warning }

```
void mlfq_scheduler(struct thread *t)
{
	enum intr_level old_level;
	struct thread *tar_t;
	struct list_elem *tar_elem;

	old_level = intr_disable();	
	int now_tick = timer_ticks();

	if (t != idle_thread) {
		t->recent_cpu += N_to_FP(1);
		// 중복 제거하여 삽mlfq_scheduler입
		tar_elem = list_head(&ch_prior_list);
		while ((tar_elem = tar_elem->next) != &t->check_prior_elem)
			if (tar_elem == &ch_prior_list.tail){
				list_push_back(&ch_prior_list, &t->check_prior_elem);
				break;
		}
	}

	// TIMER_FREQ(100) 마다 전체 thread를 순회하며 recent_cpu와 load_avg 갱신
	if (now_tick % TIMER_FREQ == 0){
		thread_cal_load_avg();
		tar_elem = list_head(&thread_list);
		while ((tar_elem = tar_elem->next) != &thread_list.tail){
			tar_t = list_entry(tar_elem, struct thread, thread_elem);
			thread_cal_recent_cpu(tar_t);
			if (t->status == THREAD_BLOCKED)
				thread_reschedule(tar_t);
		}
		list_sort(&ready_list, priority_larger, READY_LIST);
	}

	// 4 tick 마다 recent_cpu가 갱신된 thread만 우선순위 갱신
	if (now_tick % 4 == 0) {
		while (!list_empty(&ch_prior_list)) {
			tar_elem = list_pop_front(&ch_prior_list);
			tar_t = list_entry(tar_elem, struct thread, check_prior_elem);
			mlfq_cal_priority(tar_t);
			thread_reschedule(tar_t);
		} 
		list_init(&ch_prior_list);	
	}	
	intr_set_level(old_level);
}
```
{: file='threads/thread.c'}
시간을 줄이기 위해 ch_prior list에 recent_cpu 값이 변경된 thread만 계산하는 방법을 도입했습니다. 하지만 알 수 없는 list삽입을 시도할 때 알수없는 interrupt로 인해 무작위로 실패하는 현상을 확인했습니다. 이후 thread_list를 탐색해 값을 갱신해주는 방식으로 변경한뒤 다시 통과하였습니다. 시간복잡도 측면에서 더 빠르다고 생각한 방법이 함수의 stack을 쌓는 overhead에 의해 실패할 수 있으니 주의하시기 바랍니다.

#### summary

timer sleep,  thread의 문맥 전환, lock 작동원리, thread scheduling의 과정을 직접 구현하며 debugging을 해봤습니다. OS입장에서 system을 관리하는것이 굉장한 어려움이 존재한다는 사실을 깨달을 수 있었습니다. interrupt와 assembly에 대한 추가 공부를 통해 다음 project에 더 깊이 있는 코드를 작성하도록 준비해야겠습니다.