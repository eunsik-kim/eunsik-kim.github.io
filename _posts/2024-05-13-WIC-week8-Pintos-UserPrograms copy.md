---
title: WIC week8/pintos-UserPrograms
author: eunsik-kim
date: 2024-05-13 17:00:00 +0900
categories: [jungle, WIC]
tags: code, sysmtemcall
render_with_liquid: false
---

[정글끝까지-chapter2-UserPrograms] 열심히 coding한 내용을 잊지 않기 위해 기록으로 남기겠습니다!


### last week review

저번주에 asm에 대해 이해가 부족하였습니다. 그럼에도 이번주 역시 user prog로 전환을 위해 do_iret을 사용하기에 복습으로 코드를 분석하였습니다.

```
/* Use iretq to launch the thread */
/* 새로운 스레드의 context를 레지스터에 저장 */
void do_iret(struct intr_frame *tf)
{
	__asm __volatile(
		"movq %0, %%rsp\n"
		"movq 0(%%rsp),%%r15\n"
		"movq 8(%%rsp),%%r14\n"
		"movq 16(%%rsp),%%r13\n"
		"movq 24(%%rsp),%%r12\n"
		"movq 32(%%rsp),%%r11\n"
		"movq 40(%%rsp),%%r10\n"
		"movq 48(%%rsp),%%r9\n"
		"movq 56(%%rsp),%%r8\n"
		"movq 64(%%rsp),%%rsi\n"
		"movq 72(%%rsp),%%rdi\n"
		"movq 80(%%rsp),%%rbp\n"
		"movq 88(%%rsp),%%rdx\n"
		"movq 96(%%rsp),%%rcx\n"
		"movq 104(%%rsp),%%rbx\n"
		"movq 112(%%rsp),%%rax\n"
		"addq $120,%%rsp\n"
		"movw 8(%%rsp),%%ds\n"
		"movw (%%rsp),%%es\n"
		"addq $32, %%rsp\n" // 다시 한 번 스택 포인터를 조정하여, iretq 명령어를 실행하기 전에 필요한 스택의 위치로 이동시킵니다. iretq는 'interrupt return'의 약자로, 이 명령어는 인터럽트 또는 예외가 처리된 후 초기 상태로 복귀하는데 사용됩니다.
		"iretq"				// 최종적으로 iretq 명령어를 실행하여, 이전 상태로 복귀합니다. 이 과정에는 코드 세그먼트 레지스터(CS), 인스트럭션 포인터(RIP), 그리고 프로그램 상태 레지스터(RFLAGS)가 스택에서 복원되어, 원래 실행하던 프로그램의 지점으로 정확히 돌아가 계속 실행될 수 있게 합니다.
		: : "g"((uint64_t)tf) : "memory");
}

/* 현재 스레드의 context를 스레드의 interrupt frame에 옮기는 작업 그리고 do_iret함수를 호출 register rdi에 tf포인터를 인자로 넘기며 */
static void
thread_launch(struct thread *th)
{
	uint64_t tf_cur = (uint64_t)&running_thread()->tf;
	uint64_t tf = (uint64_t)&th->tf;
	ASSERT(intr_get_level() == INTR_OFF);

	/* The main switching logic.
	 * We first restore the whole execution context into the intr_frame
	 * and then switching to the next thread by calling do_iret.
	 * Note that, we SHOULD NOT use any stack from here
	 * until switching is done. */
	__asm __volatile(
		/* Store registers that will be used. */
		/* register rax, rbx, rcx의 값을 stack에 저장한다(다른값(tf_cut과 tf의 interrupt frame을 가리키는 포인터(8바이트)))으로
		채울것이기 때문에, 스택에 저장해놓는것) */
		"push %%rax\n"
		"push %%rbx\n"
		"push %%rcx\n"
		/* Fetch input once */
		/* movq %0, %%rax : 첫 번째 입력(%0, 여기서는 tf_cur)을 rax 레지스터로 이동합니다.
		   movq는 64비트 값을 이동하는 명령어로, 여기서는 tf_cur 포인터를 rax에 저장합니다.
		   movq %1, %%rcx : 두 번째 입력(%1, 여기서는 tf)를 rcx 레지스터로 이동합니다.
		   이 작업도 포인터를 레지스터로 옮기는 작업입니다. */
		"movq %0, %%rax\n"
		"movq %1, %%rcx\n"
		/* rax가 가키리는 주소값+x에 레지스터들의 값들을 저장한다.(tf_cur의 interrupt frame에 다음에 실행할때의 context를 저장) */
		"movq %%r15, 0(%%rax)\n"
		"movq %%r14, 8(%%rax)\n"
		"movq %%r13, 16(%%rax)\n"
		"movq %%r12, 24(%%rax)\n"
		"movq %%r11, 32(%%rax)\n"
		"movq %%r10, 40(%%rax)\n"
		"movq %%r9, 48(%%rax)\n"
		"movq %%r8, 56(%%rax)\n"
		"movq %%rsi, 64(%%rax)\n"
		"movq %%rdi, 72(%%rax)\n"
		"movq %%rbp, 80(%%rax)\n"
		"movq %%rdx, 88(%%rax)\n"
		/* 스택에서 pop해서 register rbx에 저장(a, b, c순으로 넣었으므로 c가 나왔을것임) */
		"pop %%rbx\n" // Saved rcx
		/* 원래 rcx에 있던값을 tf_cur의 interrupt frame에 저장 */
		"movq %%rbx, 96(%%rax)\n"
		"pop %%rbx\n" // Saved rbx
		"movq %%rbx, 104(%%rax)\n"
		"pop %%rbx\n" // Saved rax
		"movq %%rbx, 112(%%rax)\n"
		/* addq $120, %%rax: rax 레지스터의 값에 120을 더합니다.
		이는 rax가 가리키는 위치를 조정하여 특정 저장 공간(예: 인터럽트 프레임)에 접근하기 위한 준비 작업입니다. */
		"addq $120, %%rax\n"
		"movw %%es, (%%rax)\n"
		"movw %%ds, 8(%%rax)\n"
		"addq $32, %%rax\n"
		/* 현재의 명령 포인터(rip) 값을 스택에 저장하고 __next 라벨로 점프합니다. 이는 rip 값을 얻기 위한 트릭입니다. */
		"call __next\n"						  // read the current rip.
		"__next:\n"							  // rip 값을 얻는 목적지 라벨입니다.
		"pop %%rbx\n"						  // 이전 명령(call __next)에 의해 스택에 저장되었던 rip 값을 rbx 레지스터로 가져옵니다.
		"addq $(out_iret -  __next), %%rbx\n" // rbx의 값(현재 rip의 값)에 (out_iret - __next)를 더합니다. 이는 인터럽트 후에 실행을 계속할 위치를 계산하기 위함입니다.
		"movq %%rbx, 0(%%rax)\n"			  // rip.  gpt의 설명 : 계산된 rip 값을 rax가 가리키는 위치에 저장합니다.
		"movw %%cs, 8(%%rax)\n"				  // cs.   gpt의 설명 : 코드 세그먼트 레지스터 cs의 값을 rax가 가리키는 위치로 부터 8바이트 떨어진 곳에 저장합니다.
		"pushfq\n"							  // 플래그 레지스터의 현재 상태를 스택에 저장합니다.
		"popq %%rbx\n"						  // 스택에서 플래그 레지스터 값을 꺼내어 rbx에 저장합니다.
		"mov %%rbx, 16(%%rax)\n"			  // eflags
		"mov %%rsp, 24(%%rax)\n"			  // rsp
		"movw %%ss, 32(%%rax)\n"			  // 스택 세그먼트 레지스터 ss의 값을 rax가 가리키는 위치로부터 32바이트 떨어진 곳에 저장합니다.
		"mov %%rcx, %%rdi\n"				  // 변수 혹은 포인터를 함수 do_iret에 전달하기 위해 rcx의 값을 rdi에 복사합니다. x86_64 호출 규약에서 첫 번째 인자는 rdi를 통해 전달됩니다.
		"call do_iret\n"					  // do_iret 함수를 호출합니다. 이 함수는 인터럽트 후에 처리를 진행합니다.
		"out_iret:\n"						  // addq 명령에서 사용된 라벨로, rip 위치 계산에 필요합니다.
		: : "g"(tf_cur), "g"(tf) : "memory");
}
```
{: file='threads/threads.c'}

thread_launch는 rax, rbx, rcx를 사용하여 모든 register에 있는 값을 intr_frame안으로 옮기는 함수 입니다. rax에 struct를 넣고 byte단위로 이동하며 값을 넣기에 꽤나 수동적으로 복사를 한다고 생각했습니다.
launch와 대비하여 do_iret은 thread launch의 반대 과정임을 쉽게 알 수 있습니다. 예외로 특이하게 iretq함수가 존재하는데 왜 이런걸 만들었는지에 대해선 이해를 하지 못했습니다. (나머지 reg cpy는 빠르게 하기 위해?) 

### UserPrograms

1주차에선 data segment에 대한 이해 없이, scheduling방법에 초점을 두고 코드를 작성하였습니다. 그렇기에 기본지식 없이 작성한 대로 코드가 돌아가기만 하면 되었습니다. 하지만 이번주는 kernel이 하는 일에 대해 사전 지식이 필요하였습니다. 그리고 어떤 register에 무슨 값을 넣어야 program이 실행되는지에 대한 (1주차는 정말 연습이었습니다.) 이해가 필요하였습니다. 대표적으로 do iret이 왜 안되는지 분석하거나, 코드가 정상적으로 돌아가지 않을 때 rax와 정수형 reg의 인자들과, rip값을 확인하며 debugging하였습니다. 

```
int open(const char *file)
{
	check_address(file);
	struct file *file_entity = filesys_open(file);
	if (file_entity == NULL) 	// wrong file name or oom or not in disk (initialized from arg -p)
		return -1;
	// find file_entry
	struct func_params params;
    ...
```
{: file='threads/threads.c'}
```text
int open(const char *file)
{
   0x000000800421e92d <+23>:    movabs $0x800421e769,%rax
   0x000000800421e937 <+33>:    call   *%rax
   0x000000800421e939 <+35>:    mov    -0x28(%rbp),%rax
   0x000000800421e93d <+39>:    mov    %rax,%rdi
   0x000000800421e940 <+42>:    mov    $0x0,%eax
    ...
```

예를 들면, filesys_open(file)값을 return하는 과정에서 8byte가 4byte로 잘리는 현상도 목격 할 수 있었습니다. 물론 asm으로 분석한다고 해서 원인을 분석해내진 못하였지만 ([header 문제로 해결하였습니다.](https://stackoverflow.com/questions/5974300/why-am-i-able-to-compile-and-run-c-programs-without-including-header-files)) 결국 c프로그램이 돌아가는 원리에 대해 알 수 있었습니다.  

그리고 13개의 system call을 구현을 하기 위해 다양한 코드를 분석해야 했습니다.  fork를 구현하기 위해 mmu.c, palloc.c을 분석하였습니다. 그리고 file과 관련한 call을 구현하기 위해 filesys.c, file.c, inode.c에 대해 코드를 분석하였습니다. 추가적으로 더 나아가 bitmap.c와 disk.c에 대해 이해를 못한점은 아쉬웠습니다. 

os에 대한 이해 없이, userprog에 작동과정에 대한 이해 없이, 대응하는 test code를 찾지못하였고 마지막으로 printf 사용역시 불가능 하였기에 여러모로 시작하기 굉장히 어렵고 구현 여부를 확인하기가 굉장히 난해하였습니다. 

#### Argument Passing

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/syscall_extra/userprog/process.c#L194) (팀원분이 작성해주셨습니다.)

user program이 시작하기전 user stack에 data를 적재하는 과정입니다. 즉 unix에서 파일 실행 명령어를 칠때, file을 찾아 실행하는 과정에서 전처리를 하는 부분이였습니다. 그렇기에 여러 인자들이 합쳐서 들어온다면 해당하는 부분을 parsing 하여 순서대로 [calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions#System_V_AMD64_ABI)에 따라 stack에 쌓아 주면 됩니다. 간단히 요약하면 argc와 argv를 접급할 수 있게 intr_frame의 [rsp](https://casys-kaist.github.io/pintos-kaist/project2/argument_passing.html)에 넣어주면 됩니다.  

그렇기에 ptr배열이나 이중 ptr를 사용해 순차대로 넣어주면 됩니다. 혹여나 힘들어 할 학생을 위해 strtok_r함수 역시 구현되어 있어서 쉽게 할 수 있습니다. 현재 코드를 개선 한다면 반복문 1번으로도 구현할 수 있지만 속도차이는 별로 나지않다고 판단하여 for을 2번 돌면서 각각 data와 ptr그리고 padding을 순차대로 넣었습니다.

#### User Memory

2주차 과제를 하며 가장 어려웠던 순간이였습니다. 어떤걸 지시하는지 이해할 수는 있었지만 무엇을 해야될지 이해를 할 수가 없었습니다. 근본적인 원인은 pintos에서 userprog를 실행하는 원리를 이해하지 못했기 때문이었습니다. 특히 syscall이 호출되는 syscall-entry.S에 대해 알지 못하였고 virtual address에 대해 몰랐기에 다른 블로그를 참조하여 정답을 보고 이해를 할 수 있었습니다.   

사실 정답은 단순하였습니다. syscall에서 호출될 때 허용되지 않는 ptr인지 판단하는 함수인 check_address를 작성하여 추후 syscall 구현시 사용하면 됩니다. 그래서 무엇을 해야될지 모르는게 자연스러웠습니다.

#### System calls

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/syscall_extra/userprog/syscall.c) 

사실상 system call 구현이 argument passing을 제외한 2project 전부를 포함하는 내용이라고 생각합니다. 위에서 얘기하였듯이 여러.c 파일을 보고 해당하는 함수를 사용한다면 의외로 간단하게 구현 할 수 있게끔 되어 있었습니다. (wait과 fork를 제외하고)   

이외에 고려해야 될 부분은 fdt의 정보를 저장하기 위한 자료구조 선택이였습니다. 처음엔 list_elem을 관리 하기 위해 page를 전체를 할당받는 줄 알고 메모리 내부 단편화 때문에 선택하지 않았습니다. list보단 속도와 메모리 관리가 쉽다고 판단하여 page에 array를 적재하여 사용하기로 생각했습니다.

> 하지만 malloc구현을 나중에 보니 free block(16 byte씩)을 저장하여 slicing해서 주기에 단편화는 거의 없었습니다. 더구나 array에 비해 메모리 leak관리에서 훨신 유리할것이라 생각이 됩니다.

| fucntion  | summary             
| :---------| :-------------------------------: |
| halt      | power_off로 kenel process(qemu)종료  |
| exit   	| kernel이 종료한 경우를 제외하고 종료 메시지 출력 후 thread_exit() |
| create    | initial_size의 file을 생성 후 성공여부 return, 이미 존재하거나 메모리 부족 시 fail |
| remove    | file을 삭제 후 성공여부 return, file이 없거나 inode 생성에 실패시 fail |
| open      | file을 열고 file entry와 fd를 만들어 저장후 fd를 return |
| filesize  | fd에 해당하는 file의 크기 return |
| read      | fd값에 따라 읽은 만큼 byte(<=length)값 return |
| write    	| fd값에 따라 적은 만큼 byte(<=length)값 반환, 못 적는 경우 -1 반환 |
| seek   	| 현재 파일의 읽는 pos를 변경 |
| tell      | 현재 파일을 읽는 위치 return |
| close  	| fd에 해당하는 file를 close, file_entry를 NULL로 초기화 |
| dup2  	| newfd가 가리키는 file을 닫고 oldfd의 fety을 newfd의 fety가 가리키도록 바꿈 |

fork와 exec, wait을 제외하고 filesys.c를 참고하여 나머지 call을 먼저 구현하였습니다. file 제어 함수는 다 구현이 되어 있어서 위에서 말한 header file issue를 제외하고 쉽게 할 수 있었습니다. 하지만 결함이 존해 하였는데 write stderr같이 제공된 함수가 없는 경우가 해당 됩니다. (test에 해당하지 않아서 따로 구현(할 줄 몰라서)하지 않았습니다.)   

test갯수가 많은 만큼 평가가 몇몇 test를 제외하곤 scheduler test만큼 어렵지 않았습니다. 그래서 기본적인 부분을 동일하게 구현하면 여러 test를 한번에 통과할 수 있었습니다.   

다만 까다로운 부분은 역시 fdt였습니다. fd에 대해 이해가 부족하여 file entry를 구현해야될지 말아야 될지 여러번 번복하였습니다. (나중에 dup2를 위해 다시 구현하긴 하였지만) 제가 구현한 방식은 실제 linux system call과 pintos가 평가하는 system call은 약간의 차이가 있다고 생각이 됩니다. 


| fucntion               | summary             
| :---------             | :-------------------------------: |
| find_file_in_page      | 모든 page를 traverse하며, table arr에서 file을 찾으면 true return, 못찾으면 false return  |
| set_entity_in_page     | open or close 따라 fet_list와 fdt_list를 순회하면서 file과 file_entry를 제거 or 추가, 성공여부 return |
| add_page_to_list   	 | list에 빈 page가 없는경우 사용, page를 ls에 추가한 뒤 0으로 초기화 |
| update_offset          | page search를 빠르게 하기 위해 offset update하는 함수 |

처음엔 open갯수를 제한하여 구현을 많이 하였지만 (test에 문제없이 통과됩니다.) 실제로는 무제한으로 메모리를 증가시키는것이 맞다고 생각되어 page list를 통해 open된 file ptr을 저장하였습니다. 즉 fdt_list (file discripter table)과 fet_list(file entry table)을 통해 데이터를 저장하고 필요시 page를 추가로 할당받도록 구현하였습니다. page에 특정 offset부터 접근하여 탐색시 약간의 속도를 높이도록 설정하였습니다.  

> 특정 offset이라 함은 file 삭제와 삽입시 탐색 시작 위치와 종료 위치를 기억해뒀다가 다시 사용하는 방법입니다.

제가 선택한 자료구조에서도 문제가 존재하였습니다. 바로 가장 낮은 fd값을 찾기 위해서는 O(n)만큼 소모해야 합니다. (물론 offset으로 접근하더라도 page를 search하는 시간이 최악의 경우 O(n)입니다.) 그렇기에 속도를 위해 dup2호출을 제외하고 open시에는 index에 해당하는 fd를 사용하도록 편법을 사용하였습니다.  

어떻게 해야 효과적으로 fd를 찾을 수 있을지는 추가적인 고민이 필요한것 같습니다. 그리고 open과 close에서 사용되는 page의 값을 변경하기 위한 함수와, 그 외 나머지 함수에서 사용하는 file을 찾는 함수를 만들어 사용하였습니다.

```
struct fdt 
{
	struct file_entry *fety;
	uint64_t fd;
};

struct file_entry
{
	struct file *file;
	uint64_t refc;
};

struct fpage {
    struct list_elem elem;
    int s_ety;          // first empty offset
    int s_elem;         // first elem offset
    int e_elem;         // last elem offset + 1
    uint64_t page_diff;       // ptr diff new fetpage which has been duped
    union data {
        struct fdt fdt[MAX_FETY];
        struct file_entry fet[MAX_FETY];
    } d;
};
```
{: file='threads/threads.h'}

| fucntion  | summary             
| :---------| :-------------------------------: |
| fork      | parent process의 pml4, intr_frame, fd copy 후 return tid, 실패 시 return TID_ERROR |
| exec    	| file name을 parsing하여 해당 code를 적재한 intr_frame, pml4를 만들고 해당 process로 변경 |
| wait      | fork한 pid를 대기 fork한 process의 exit의 status return |

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/syscall_extra/userprog/process.c) 

위의 함수를 구현하며 process.c의 duplicate_pte, __do_fork fork를 추가적으로 구현하였습니다. fork exec 그리고 wait를 구현하면서 page_fault exception을 너무 많이 겪어 debugging할 때 힘들었습니다. 기본적인 TODO들이 process.c에 지시되어 있었습니다. 하지만 syscall 호출 시 current tf에 저장이 안되는 현상 즉 syscall-entry.S를 이해하지 못하여 발생하는 상황 때문에 어려웠습니다. 

> 과제 설명서엔 You don't need to clone the value of the registers except %RBX, %RSP, %RBP, and %R12 - %R15 라고 적혀있지만 register 전부를 복사하지 않으면 page fault 에러가 발생합니다. 저는 복사를 해야되는 필수 register를 찾지 못하였기에 일괄로 전부 복사하였습니다.    
{: .prompt-warning }

그리고 pml4의 작동원리를 이해하지 못하여서 duplicate_pte를 작성하는데 많은 에러를 겪었습니다. ([va가 kerenl addr인지 판별하는 구문](https://github.com/eunsik-kim/pintos11/blob/eunsik/syscall_extra/userprog/process.c#L103)에서 true를 return해야 되지만 va를 습관적으로 false로 return 해버렸습니다.) pml4는 [다음블로그](https://sean.tistory.com/146#google_vignette)에서 많은 정보를 얻었습니다. 현재 pintos의 구현과 다른것 같지만 잘 정리가 되어 있습니다. 

1. pml4 는 paging 기법 중 하나입니다. 현재 pintos에서는 4단계 table로 구현되어 있습니다.
2. 각 table은 512개의 entry를 가지며 각 entry가 virtual address의 index에 해당합니다. 마지막 table안에 physical address가 존재합니다. 
3. pintos에서는 추가로 init.c에서 모든 pml4의 기초가 되는 base_pml4를 먼저 생성하고 이후 pml4 create할 때 base_pml4를 복사하여 생성합니다. 이는 base_pml4에는 kernel영역에 접근하는 첫번째 index갑이 존재하는 page입니다.
4. 즉 모든 thread는 kernel영역에 접근 가능해야하고 그렇기에 생성할때마다 복사를하며 따라서 제가 틀린부분(duplicate_pte)에서 true를 return 해야합니다. (한번 더 복사할 필요 없음)

fork를 하면서 parent는 child가 온전히 do_fork를 수행하였는지 확인해야 합니다. 그렇기에 signal 대신에 sema 한개가 필수적으로 사용됩니다. 이외에 wait을 할 때, parent나 child는 exit status를 전달하기 위해 서로를 기다립니다. 이 부분에선 sema는 최소 2개가 필요합니다.  

그리고 exit_status를 전달 받기 위해 child_elem을 parent_list에 저장하여 전달 받도록 하였습니다. sema를 잘못사용하거나 실패시 예외 처리를 잘못하면 zombie process가 생성되어 oom test를 통과하기 위해 불가피한 코드를 삽입해야할 수 있습니다. (팀원분들은 parent가 죽기전 child를 깨우는 `전설의 코드`를 삽입하면 대부분 memory leak를 해결 할 수 있었지만 불필요하다고 생각이 됩니다.)   
 
추가적으로 memory leak이나 에러를 겪었던 부분들입니다.
1. _do_fork 실패시 TID_ERROR 반환하며 sema down되면 안됩니다.
2. open하며 open실패한 file 메모리 해제
3. process_exec와 thread_create내에 실패시 실패 전까지 할당 한 메모리 해제
4. deadlock을 피하기 위해 file_lock을 죽기전 해제
5. process cleanup 할 땐 fdt를 삭제하면 pml4를 destroy 할 때 에러가 발생

| fucntion  | summary             
| :---------| :-------------------------------: |
| process_init_fdt    	  |  stdin, stdout, stderr를 기본 생성 |
| process_duplicate_fdt   | fdt와 관련된 fdt_list와 fet_list안의 page들과 file들을 자식에게 cpy  |
| process_delete_fdt      |  fdt와 관련된 fdt_list와 fet_list안의 page들과 file들을 free |

마지막으로 fdt table을 유지하기 위해 3가지 함수를 만들어 사용했습니다. 특히 process_duplicate_fdt는 약간 dirty하지만 n개를 전부 복사하려면 어쩔수 없었습니다... 

1. fet와 fdt모두 page단위로 memcpy를 하였습니다. 
2. new fdt에서 fet ptr는 예전 parent의것과는 다른 새로운 ptr를 할당해야 합니다. fet page에서 old page와 new page간의 차이를 찾아 page_dff에 저장합니다.
3. new fdt를 순회하며 이전 fet page의 page_dff를 보고 새롭게 초기화 합니다.
3. new fet에선 file을 새롭게 복사하여 new offset을 가지도록 했습니다. 

위와 같은 방법으로 O(n)의 속도를 가지는 함수를 만들었습니다. 다만 process_delete_fdt는 비교적 list 자료형 보다 빠르게 할 수 있다고 생각합니다.

#### etc..

rox는 load할 때 file_close를 안하고 thread_exit할 때 해결 할 수 있습니다. bad-*** 류의 test는 page fault 발생시 intr_frame의 rax값을 조회함으로 한번에 모두 해결 할 수 있었습니다.  disasm을 통해 알아내었는데 rax를 주로 값을 저장하는 register로 사용하는 이유는 잘 모르겠습니다.

### summary

다양한 syscall을 구현하므로 실제로 사용자의 입장이 아닌 제작자의 입장에서 함수내부를 고민해볼 수 있는 시간이였습니다. 그러기 위해 새로운 library 사용법을 익힌다는 마음가짐으로 syscall를 구현하기 위해 다양한 함수들의 구성들을 알아보고 사용할 수 있어서 좋았습니다.   

깊게 들어가보면 pintos의 세계의 끝에는 전부 asm으로 구성되어 있어서 system 개발을 위해선 더욱이 asm을 배울 필요가 있다는 생각을 하게 되었습니다.   

다양한 test덕분에 예외처리의 중요성에 대해 깨달을 수 있었습니다. 특히 oom test를 통과 하기 위해 memory관리를 효율적으로 하기 위해 고민을 많이 했어야 했습니다.  

이전 project에서 와 동일한 실수를 범하지 않게 하기 위해 충분한 고민을 통해 fdt를 유지하도록 자료구조를 선택하였습니다. 여러번 엎었지만 그래도 괜찮은 성능을 보여 만족스럽다고 생각이 됩니다.  
 
앞으로 3주차 프로젝트에선 권영진교수님께서 해주신 강의 내용 중 mechanism과 policy의 차이와 layer astraction을 고려한 코드를 작성할 수 있도록 노력해야될것 같습니다.  

[disk 발표 자료](https://docs.google.com/presentation/d/13VVCnRquPySW4SSA34tnzuBQKqWJspWuVdaE-25xxvw/edit#slide=id.p)