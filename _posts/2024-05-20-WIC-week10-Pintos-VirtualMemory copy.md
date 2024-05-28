---
title: WIC week10/pintos-VirtualMemory
author: eunsik-kim
date: 2024-05-20 17:00:00 +0900
categories: [jungle, WIC]
tags: code, VA
render_with_liquid: false
---

[정글끝까지-chapter3-VirtualMemory] 열심히 coding한 내용을 잊지 않기 위해 기록으로 남기겠습니다!


### last week review

1, 2 project를 지나 새팀에 오게되면서 저의 코드를 받아 이어쓰게 되었습니다. 다음 팀원에게 설명하다 보니, 버그에 대한 걱정도 있었고 이전에 작성한 삽입, 삭제 코드가 복잡하게 느껴져 refactoring 과정을 거쳤습니다.   

[이전 코드](https://github.com/eunsik-kim/pintos11/commit/4005e550aea145ea87f58cdc649f3838f4fb009b)

| fucntion           | length   | summary             
| :---------         |:-----    | :-------------------------------: |
| set_entity_in_page | 75   	| open or close 따라 fet_list와 fdt_list를 순회하면서 file과 file_entry를 제거 or 추가, 성공여부 return |
| add_page_to_list   | 6		| list에 빈 page가 없는경우 사용, page를 ls에 추가한 뒤 0으로 초기화 |
| dup2   	 		 | 38		| newfd가 가리키는 file을 닫고 oldfd의 fety을 newfd의 fety가 가리키도록 바꿈  |


코드를 최대한 묶어서 작성하고 싶은 마음과 debugging의 효율성을 고려하여 최대한 하나에 몰아서 작성했습니다. 그러다 보니 삽입과 삭제를 하나의 함수 set_entity_in_page에 하게 되면서 가독성과 이해가 힘들어 졌기에 코드를 나누어 작성하였습니다.

[refactoring 코드](https://github.com/eunsik-kim/pintos11/blob/eunsik/syscall_extra/userprog/syscall.c)

| fucntion               | 	length	 | summary             
| :---------             | :-------- | :-------------------------------: |
| open_fety_fdt_in_page  | 29		| fet_list, fdt_list page를 순회하며 빈자리에 fety와 fdt를 page에 추가, 성공여부 return |
| open_fdt_in_page       | 24		| fdt_list page를 순회하며 빈자리에 fdt를 page에 추가, 성공여부 return |
| delete_fety_fdt_in_page| 33		| fet_list, fdt_list page를 순회하며 조건에 맞는 해당 entity 삭제, 성공여부 return |
| add_page_to_list   	 | 8		| list에 빈 page가 없는지 check, 없다면 새 page를 ls에 추가한 뒤 0으로 초기화, 성공여부에 따라 page를 return  |
| dup2   	 			 | 25		| newfd가 가리키는 file을 닫고 oldfd의 fety을 newfd의 fety가 가리키도록 바꿈  |

fdt와 fet를 삽입과 삭제하는 각 부분, page 추가하는 부분을 수정하고 나눠 함수화 하였더니 분산 되었지만 더 읽기 좋아졌음을 알 수 있었습니다. result struct를 이용하여 함수의 결과를 받는 방식에 대해선 오용의 위험이 있지만 범용성을 고려하여 그대로 사용했습니다. (이전 > 이후 길이 총합) 119 > 119 총 길이는 같지만 더 가독성이 좋은 코드로 개선하여서 기분이가 좋았습니다!

### Virtual Address

pintos와 os의 kernel에 대해 더 깊이 이해할 수 있는 좋은 project 인것 같다고 생각이 되었습니다. Userprog와 scheduler에선 친절하게 func들이 일부분이(?) 채워져 있었습니다. 그 덕분에 vm.c를 처음 여는 순간 횡한 skeleton을 보고 당황할 수 있었습니다. syscall.c는 그나마 사용법을 아는 함수를 채우는 일이여서 덜 힘들었지만, 이번엔 제가 만들지도 않았고, 전혀 처음 보는 함수들을 채우려니 어려웠습니다. skeleton을 최대한 수정하지 않고 TODO를 따라 구현하도록 노력했습니다.

#### hash

[code](https://github.com/eunsik-kim/pintos11/blob/main/lib/kernel/hash.c)

저번주에 많은 *.c파일을 분석해둔 덕분에 hash.c 파일과 process.c 파일 하단 부분만 이해를 하면 이번 project를 할 수 있었습니다. 학부시절에 잠깐 배운(기억이 안날정도로 복잡한) hash func과 실제로 적용되는 hash의 차이를 보니 재밌었습니다.

정말 신기했던 점은 mechanism과 policy의 차이을 고려하여 구현이 되어있었습니다. 그렇기에 policy에 해당하는 hash func은 직접 구현해야 되는줄 알고 정말 걱정했습니다만 [hash_byte](https://github.com/eunsik-kim/pintos11/blob/main/lib/kernel/hash.c#L244)가 미리 구현이 되어있어서 다행이였습니다. 간단히 설명하면 그냥 입력되는 길이에 맞춰 큰숫자를 곱하며 xor하는 함수 였습니다. 중복이 될 수 있다고 생각했지만 bucket_index와 rehash를 사용하였기에 중복되지 않고 빠르게 찾을 수 있습니다. 

일반적으로 원소를 찾는 과정을 계산해보겠습니다.

1. 이상적으로 hash에는 size의 절반 갯수로 bucket이 존재합니다. (2차원 배열 생각하면 됩니다.) 
2. hash elem이 담긴 bucket을 찾기위해, 즉 buket_idx를 탐색하기 위해 hash fuc이 사용됩니다. 
3. 각 bucket안에서 elem을 찾을 땐 각 elem의 va(고유값)을 가지고 찾습니다.(각 bucket은 최대 n/log(n)개 미만의 원소를 가집니다.)
4. 총 시간이 O(n/log(n) * 24 = 최대 갯수 * idx 계산 시간[8bytes * 3(길이 * 연산횟수)])의 복잡도를 가집니다.  

```
rehash (struct hash *h) {

	/* 새롭게 bucket 갯수 계산 */
	new_bucket_cnt = h->elem_cnt / BEST_ELEMS_PER_BUCKET;
	if (new_bucket_cnt < 4)
		new_bucket_cnt = 4;
	while (!is_power_of_2 (new_bucket_cnt))
		new_bucket_cnt = turn_off_least_1bit (new_bucket_cnt);

	/* 대충 2의 지수승 될때만 갱신함 */
	if (new_bucket_cnt == old_bucket_cnt)
		return;

	/* 새롭게 메모리 만들기. */
	new_buckets = malloc (sizeof *new_buckets * new_bucket_cnt);
	if (new_buckets == NULL) {
		return;
	}
	for (i = 0; i < new_bucket_cnt; i++)
		list_init (&new_buckets[i]);

	/* 전부 순회하면서 삽입 하는 부분 */
	for (i = 0; i < old_bucket_cnt; i++) {
		struct list *old_bucket;
		struct list_elem *elem, *next;

		old_bucket = &old_buckets[i];
		for (elem = list_begin (old_bucket);
				elem != list_end (old_bucket); elem = next) {
			struct list *new_bucket
				= find_bucket (h, list_elem_to_hash_elem (elem));
			next = list_next (elem);
			list_remove (elem);
			list_push_front (new_bucket, elem);
		}
	}
}

```
{: file='lib/kernel/hash.c'}

그렇다면 bucket을 초기화 하여 중복이 되지 않고 빠르게 원소를 찾을 수 있도록 하는 rehash함수를 분석해 보겠습니다.

1. 원소의 갯수가 2의 지수승임을 확인합니다.
2. buckets전부를 순회하며 기존 bucket을 지우고 새롭게 만든 bucket에 넣어줍니다.
3. rehash의 평균 실행속도는 O(sum((t + malloc, free 속도) * log(t))) [t = 4, 8, ... , log(n)] 입니다.
4. 근사하면 log((1 + log(n)) * 2 ^ log(n)) 입니다.  (제 계산이 틀리지 않았다면 말이죠) 

삽입 삭제시 rehash로 갱신해줘야 되기에 빈번하게 일어나는 경우 overhead가 많이 생긴다고 볼 수 있습니다. 그래서 memory eviction을 잘하는것이 전반적인 프로그램 실행속도와 연관됨을 알 수 있습니다. (적고나니 너무 당연한 말입니다.) 그래도 linked list보단 훨신 빠른것 같아 편---안합니다.


#### Memory Management

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/vm/vm/vm.c#L97)

hash 자료구조를 활용하여 spt(supplemental_page_table)와 fbt(frame_table)를 구현하였습니다. 그리고 삽입과 삭제하는 부분을 TODO에 명시된 대로 넣으면 되었습니다. 근데 이게 lock을 매번적는게 귀찮아서 hash.c 파일을 수정하여 넣어 버렸습니다. (filesys 파일도 그렇게 수정할걸 그랬습니다.) 그렇게 하니 그냥 syscall에서 작성하는것 처럼 그냥 적절한 위치에 구현된 적절한 함수를 넣으면 되는 부분이라 쉽게 할 수 있습니다. 하지만 이함수를 언제 사용하는지 아직 모르기 때문에 예외처리를 어떻게 해야될지 몰라 감을 잡기가 어려웠습니다.


#### Anonymous Page & Memory Mapped Files

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/vm/userprog/process.c#L827)

> paging은 다행히 미리 class간의 상속을 기반을 통해 편하게 구현할 수 있도록 struct가 구현되어 있었습니다. 하지만 그 덕분에 처음에 이해하기 더 어려웠고 copy를 하는데 한가지 더 고려해야되는 불편함이 존재하는 양날의 검인것 같다고 생각했습니다. 그래도 주어진대로 따라 구현하였습니다.  

`VM_ANON(anonymous page)`은, 즉 0으로 초기화 되어야 되는 page, 약간씩 다르지만 heap과 stack과 bss영역에 해당됩니다. `VM_FILE(file-backed page)`는, disk에서 온 page, mmap에 할당된 메모리나 code영역에 해당합니다.   

##### strange initialization

하지만 저의 지식부족으로 처음에 bss영역은 VM_ANON이 아니라 code영역과 합쳐 VM_FILE로 구현하였습니다. (그래서 추가적으로 VM_MMAP으로 mmap data와 구분하였습니다.) pintos는 malloc과 같은 동적할당을 하지 않기 때문에 stack을 제외하고 VM_ANON 페이지를 할당하는 함수를 만들 필요가 없다고 생각하였기 때문입니다. 그래서 load_segment함수를 최대한 활용하여 대부분 pageing를 초기화 하도록 설계하였습니다. (사실 구현시 do_mmap과 do_munmap의 존재를 몰랐습니다.)

```bash
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  LOAD           0x000000 0x00000000003ff000 0x00000000003ff000 0x000190 0x000190 R   0x1000
  LOAD           0x001000 0x0000000000400000 0x0000000000400000 0x004ef0 0x004ef0 R E 0x1000
  LOAD           0x000f00 0x0000000000405f00 0x0000000000405f00 0x000000 0x000524 RW  0x1000
  NOTE           0x005ed0 0x0000000000404ed0 0x0000000000404ed0 0x000020 0x000020 R   0x8
  GNU_PROPERTY   0x005ed0 0x0000000000404ed0 0x0000000000404ed0 0x000020 0x000020 R   0x8
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10

  $ readelf -Wl args-none 
```

load segment를 구현하면서 제가 한 큰 실수(offset)를 하여서 처음에 loading을 못하였습니다. 그러면서 load segment의 header의 세부내용을 위의 명령어로 알 수 있었습니다. 신기하게 elf파일은 va와 코드영역이 분리가 되어 각 segment의 내용을 인자로 load_segment 함수로 들어옴을 알 수가 있었습니다.   

그렇기에 사실 분리하여 구현할 수 있었지만 구현의 용이함을 (귀찮아서) 위해 한꺼번에 VM_FILE로 구현하였습니다. 이 선택 때문에 mmap과 code, bss간의 구분하여 초기화 하기위해 어쩔 수 없이 함수 인자 type을 수정 하여 구현하 였습니다.

```
bool
load_segment(struct file *file, off_t ofs, uint8_t *upage,
			 uint32_t read_bytes, uint32_t zero_bytes, uint64_t writable)

```
{: file='userprog/process.c'}

```
bool
vm_alloc_page_with_initializer (enum vm_type type, void *upage, uint64_t writable,
		vm_initializer *init, void *aux) {

	/* Check wheter the upage is already occupied or not. */
	if (spt_find_page (spt, upage) == NULL) {


		if (writable >= VM_MMAP) {
			struct lazy_load_data *data = aux;
			data->mmap_list = writable & ~1;
			type = type | VM_MMAP;
		}
```
{: file='vm/vm.c'}

syscall mmap에서 활용할 수 있도록 static을 bool로 바꿨고 writable type을 bool이 였지만 uint64_t로 변경하였습니다. 추후 vm_alloc_page_with_initializer에서 page 초기화 할 때 mmap_list를 넣을 수 있도록 하였습니다. 이렇게 구현해도 되는지 모르겠지만 그냥 함수 구조를 바꿔가며 여럿 사용하기가 싫어서 이렇게 하였습니다.

##### stack-growth

```
/* Growing the stack. 
커널에서 user stack에 page fault가 나는 경우는 어떤경우? */
bool vm_stack_growth (void *addr) {
	if ((USER_STACK - (uint64_t)addr) >= (1 << 20))	 // out of bound
		return false;

	void *stack_bottom = pg_round_down(addr);
	if (!vm_alloc_page(VM_ANON | VM_STACK, stack_bottom, true))
		return false;

	if (!vm_claim_page(stack_bottom))
		return false;

	thread_current()->stack_bottom = stack_bottom;
	return true;
}
```
{: file='vm/vm.c'}

stack 영역에 메모리를 추가하는 함수는 위와 같이 구현하여 stack init과 stack growth시에 사용했습니다.  

 kernel에 stack-growth 요청임을 checking하는것이 rsp기준 -8이거나 0이라고 되어 있었습니다. 하지만 예전 FQA에는 fault난 rsp가 그렇지 않은 경우(arr를 초기화하는)도 존재한다고 적혀있었습니다. 따라서 모호하므로 그냥 last stack bottom기준 PGSIZE이내로 요청이 들어오는 경우 증가하도록 설정하였습니다.   

이외에 kernel 영역에서 fault나는 경우 stack growth를 고려해야 한다고 하였는데 사실 저희가 구현한 pintos에는 그런 경우가 없기에 따로 정보를 저장하여 예외 처리하지 않았습니다.

##### mmap & munmap

[완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/vm/userprog/syscall.c#L623)

file-backed page를 생성하는 syscall을 구현해야합니다. 이 개념에 대해 git book에 명확히 안적혀 있어서 구현을 하기에 혼란스러웠습니다. (제가 영어를 잘 못해서 그런것일 수 도 있습니다.) 메모리 mmap된 데이터를 공유가 되어야 할지 안되어야 할지 몰라 감을 못잡아서 일단 (msync가 없기 때문에) private로 생성하였습니다. 그리고 process가 exec할 때 mmap page는 삭제되지 않고 exit할 때만 삭제 되도록 구현하였습니다. 

그리고 destroy할 때만 dirty유무를 check하여 disk에 write하도록 하였습니다. 사실 항상 적도록 할 수 있습니다만 일단 page fault에서 write 요청이 들어가는경우 항상 frame에 check 하도록 하였습니다.
munmap에는 addr에 해당하는 page를 전부 삭제해야 되었기에 mmap 호출시 mmaplist를 만들어 관리하였습니다.

#### Copy-on-Write

[미완성본](https://github.com/eunsik-kim/pintos11/blob/eunsik/vm/vm/vm.c#L182)

> 생각해보니 fdt&fet 역시 모두 user pool에서 생성하여 managing하는것이 맞다고 생각이 되었습니다. (현재는 kernel pool에서 생성 합니다.) 다만 ANON으로 생성하여 관리하는 것은 상관 없지만 cpy하는 경우 고려해야될 경우가 너무 많아 시간이 없기에 추후 수정하기로 결정하였습니다. (코치님 시간이 너무 부족 합니다!!)

2주차 test까지 전부 통과하려면 spt copy를 구현해야 합니다. (저번 userprog에서 처음에 fet를 고려하여 구현하였다가 불필요하여 폐기하였는데, dup2 때문에 다시 부활 시킨 경험이 상기되었기에) 처음부터 cpwrt를 고려하여 구현하는것이 맞다고 생각했습니다. 그래서 hash action함수를 cpwrt기반으로 작동하도록 기초적으로 구현하였습니다. (simple만 통과하기에 기초라고 표시하였습니다.)

page를 복사할 때 고려해야될 경우는 5가지 입니다. frame에 존재하거나 존재하지 않을때, 각각 file이거나 anon인 경우, 총 4가지 이며 그 전 상태인 uninit을 포함하면 5가지 입니다.
hash elem을 돌면서 해당 함수를 사용하여 복사를 하였습니다. 

```
/*
 * 1. uninit인 경우 2. frame에 존재하는 경우, 3.disk에 존재하는 경우, 4.SWAP에 존재하는경우
 * 각 page 복사, frame은 pagefault시 복사, lazy_load_data, mmap_list복사,  
 */
bool hash_copy_action(struct hash_elem *e, void *aux)
{	
	struct thread *cur = thread_current();
	struct page *src_page = hash_entry(e, struct page, hash_elem);
	struct page *dst_page = (struct page *) calloc(1, sizeof(struct page));
	struct lazy_load_data *cp_aux;
	struct list *mmap_list;
	memcpy(dst_page, src_page, sizeof(struct page));

	if (src_page->operations->type == VM_UNINIT){
		// aux copy
		if (!(cp_aux = (struct lazy_load_data *)malloc(sizeof(struct lazy_load_data))))
			return false;
		memcpy(cp_aux, src_page->uninit.aux, sizeof(struct lazy_load_data));
		list_push_back(&cur->spt.lazy_list, &cp_aux->elem);
		dst_page->uninit.aux = cp_aux;	

		// cp mmap_list
		if (src_page->type & VM_MMAP) {
			if ((mmap_list = find_new_mmap_list(src_page)) == NULL)
				return false;
			cp_aux->mmap_list = mmap_list;
		}
		goto end;
	}

	switch (VM_TYPE(src_page->type))
	{
	case (VM_FRAME + VM_FILE):
		// aux copy
		if (!(cp_aux = (struct lazy_load_data *)malloc(sizeof(struct lazy_load_data))))
			return false;
		memcpy(cp_aux, src_page->file.data, sizeof(struct lazy_load_data));
		list_push_back(&cur->spt.lazy_list, &cp_aux->elem);
		dst_page->file.data = cp_aux;		

		// cp mmap_list
		if (src_page->type & VM_MMAP) {
			if ((mmap_list = find_new_mmap_list(src_page)) == NULL)
				return false;
			cp_aux->mmap_list = mmap_list;
			list_push_back(mmap_list, &dst_page->file.mmap_elem);
		}

	case (VM_FRAME + VM_ANON):
		// for cpwrite, make page unwritable
		if (src_page->type & VM_WRITABLE)
			ASSERT(pml4_set_page((uint64_t *)aux, src_page->va, src_page->frame->kva, 0));			
		if (!pml4_set_page(cur->pml4, dst_page->va, dst_page->frame->kva, 0))
			return false;
			
		src_page->type |= VM_CPWRITE;
		dst_page->type |= VM_CPWRITE;
		ASSERT(dst_page->frame == src_page->frame);
		if (pml4_is_dirty(aux, src_page->va))
			src_page->frame->dirty = true;
		src_page->frame->unwritable++;
		goto end;

	case VM_ANON:	
	case VM_FILE:
		goto end;

	default:
		PANIC("wrong access");
	}
end:
	return spt_insert_page(&cur->spt, dst_page);
}
```
{: file='vm/vm.c'}

swap-out&swap-in을 구현하지 못하였음에도 상당히 깁니다. 이유는 page 복사뿐만 아니라 mmap_list나 lazy_load_data구조체를 전부 똑같이 복사해야 되기 때문입니다. 이외에는 삭제시에 충돌이 너무 많이 발생하여 복사는 위와 같이 frame만 sharing하도록 처리하였습니다. 나중에 page fault시에 type을 checking하여서 cpwrt시에 page를 cp하도록 하였습니다.  

> 위와 같이 구현하였을 시에 복사는 잘되었지만 사실 가장 까다로웠던 부분은 언급 했듯이 delete하는 부분이었습니다. 구현을 잘한다면 lazy load data는 parent의 data로 sharing할 수 있다고 생각합니다. 하지만 삭제를하면서 중복 삭제가 되어서 data가 사라지고 panic에 자주 빠져 어쩔 수 없이 전부 복사를 실시하였습니다. 이 부분은 추후에 ref cnt를 넣어 중복 참조 할 수 있도록 수정하도록 하겠습니다.

#### etc

parallel test를 통해 여러 임계영역에서 shared data로 인한 문제가 많이 발생함을 알 수 있었습니다. 그래서 cpwrt할 때 lock을 걸었고 1주차에 작성한 donation시에 for문 안에 예외처리를 추가하였습니다. 아직은 error가 눈에 보이지 않지만 이외에도 여러 에러가 발생할 수 있습니다.   

특히 pml4 destroy 할때 발생하는 에러를 막기 위해 pml4_clear_page를 적절한 위치에 해줘야 되는것이 까다로웠습니다.  

그리고 disk write를 실행파일에 잘못 하는 경우 exec한뒤 함수를 실행못하고 갑자기 죽는 에러도 있었는데 지금생각하면 어떻게 해결했는지 신기합니다.  

### Summary

아직도 코드 설계실력이 많이 부족한것 같습니다. 작성한 함수가 독립성을 지키는지 기능에 맞게 구현 했는지 확실치 못하는것 같습니다.   

저번에 배운대로 mechanism과 policy의 차이와 layer astraction을 고려하여 구현하고 싶었지만 시간 관계상 구체적으로 설계를 하지 못하고 구현하였습니다.   

추 후 1. swap-in, out을 마저 구현하고 2. lazy load data를 중복 참조 3. BSS data를 VM_ANON으로 초기화 하도록 코드를 개선해야 겠습니다. (글을 정리하고 보니 문제가 되는것 같습니다.) 그리고 빨리 file system을 가능한곳 까지 구현 하도록 정말 최선을 다해야 겠습니다.

process간의 isolation을 지키며 하나의 computer를 사용하도록 illusion을 주는 VA를 설계하는것이 굉장히 어려움을 알 수 있습니다.  

VA와 더불어 메모리를 효율적으로 사용하기 위해 spt나 ftb를 유지하는 overhead를 줄이도록 고민을 많이하게 되었습니다.  

이번 프로젝트를 하면서 구현된 hash에 대한 사용방법을 알 수 있어서 좋았습니다. 추가적으로 bitmap 사용방법에 대해서도 알고 싶다고 느껴졌습니다.  


[palloc 발표자료](https://docs.google.com/presentation/d/13LtASbmnk7oyyRr1147jOyGWciSPvu0qsPH-hVNFxdE/edit?usp=sharing)


