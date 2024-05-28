---
title: WIC week11/pintos-FileSystems
author: eunsik-kim
date: 2024-05-27 17:00:00 +0900
categories: [jungle, WIC]
tags: code, File
render_with_liquid: false
---

[정글끝까지-chapter4-FileSystem] 열심히 coding한 내용을 잊지 않기 위해 기록으로 남기겠습니다!

> swap in & out 을 구현하면서 드디어 각 함수들의 존재 유무에 대해 깨달을 수 있었습니다. 각 함수들의 기능을 잘 구현하면 프로그램이 동작할 수 밖에 없도록 되어 있습니다. 즉, 노력하여 layer astraction을 생각할 필요없이 해당 기능들을 정해진 위치에 사용한다면 코드 중복을 줄이고 최소한의 함수들로 구현을 할 수 있습니다. 

### last week review

[완성본](https://github.com/eunsik-kim/pintos11/tree/eunsik/vm/vm/vm.c)

발표를 준비하며 공부했던 disk write와 palloc 덕분에 swap out과 swap in을 쉽게 구현할 수 있었습니다. 주제 선정에 관한 운이 정말 좋았던것 같습니다. swap table은 extent기반의 bitmap으로 구현했습니다.  

저번주 부족했다고 생각했던점, 총 3가지 중 2번은 하지 않아도 된다고 판단하여 하지 않았습니다. (공유자원 관리의 비효율성)

1. swap-in, out을 마저 구현
2. lazy load data를 중복 참조 
3. BSS data를 VM_ANON으로 초기화

그리고 추가적으로 다음사항들을 수정하였습니다. 코드를 많이 수정한 순서대로 나열하면 다음과 같습니다.

1. copy write에서 reference count > circular linked list로 frame 중복 참조 구현  - swap out 때문에 불가피하였습니다.
2. munmap에서 mmap list > pg_cnt를 저장하여 va에서 pg_cnt 횟수만큼 page 삭제로 대체 - copy 할때 사용한 방식이 너무 어렵고 속도역시 느렸습니다.(디버깅 해주면서 알게 되었습니다.)
3. stack kernel에서 page fault 발생하는 경우 해결할 수 있도록 수정 > FAQ 게시판을 보고 가능하다고 판단하여 구현을 수정하였습니다.
4. 1주차 lock에서 발생하는 문제 해결... (parallel test가 정말 좋은 test임과 동시에 해결하기 너무 어려운 test라고 생각이 되었습니다.)

> ref count를 frame struct에서 관리를 하였습니다. 하지만 swap out되면 frame 객체가 사라진다는 사실을 마지막에서야 깨달았고 이것을 보존하기 위해 1. disk에 기록하거나 anon을 2. hashmap으로 다시 구현하거나 3. page에 기록하는 방식중 3번을 선택하여 다시 구현하기로 결정했습니다. 이유는 그게 list 자료구조가 이미 존재하여 구현이 용이하고 빠르기 때문입니다. 

위의 내용들을 제외하고 swap-in, out을 구현 하면서 전반적인 코드 구성을 수정하였습니다. 불필요한 내용을 줄이며 많은 내용을 수정하였지만 다행히 잘 돌아가는 제 코드를 보니 자책하던 마음을 조금 접어둬야겠다고 생각했습니다. 다만 디버깅 중에 cpwrt시 write 요청시 race condition에서 발생한 문제 상황에 대한 정확한 분석을 할 수 없었습니다. 
```
static bool vm_handle_wp (struct page *page) {
	struct thread *cur = thread_current();
	if (!(page->type & (VM_CPWRITE || VM_WRITABLE)))
		return false;
	page->type &= ~VM_CPWRITE;
	page->type |= VM_DIRTY;
	lock_acquire(&cp_lock);		<< race condition start >>
	// cp on wrt
	if (is_alone(&page->cp_elem)){		// use frame alone
		lock_release(&cp_lock);
		if (!pml4_set_page(cur->pml4, page->va, page->frame->kva, 1))
			return false;
		return true;
	}
	list_remove(&page->cp_elem); 	// delete redundant refer
	struct frame *new_frame = vm_get_frame();
	lock_release(&cp_lock);			<< race condition end>>
	memcpy(new_frame->kva, page->frame->kva, PGSIZE);
	page->frame = new_frame;
	new_frame->page = page;
	if (!pml4_set_page(cur->pml4, page->va, new_frame->kva, 1))
		return false;
	
	return true;	
}

```
{: file='vm/vm.c'}
이분탐색을 통해 경쟁상황이 발생하는 구역을 특정할 수 있었습니다만 vm_get_frame() 함수 이전에 걸면 안되는 이유를 찾지 못하였습니다.

> 번외로 공유자원에 대한 접근문제인지 확인하기 위해 다음과 같은 테스트를 해봤습니다. circular linked list가 공유자원이므로 공유 semaphore 를 통해 참조 및 삽입과 삭제시 접근을 제한하도록 하였습니다. 그럼에도 결과는 위에서 발생하는 문제를 해결하지 못하였습니다. (그리고 그에따른 속도는 느려지고 다른 문제가 생길소지가 있다고 판단하여 삭제하였습니다.) 
더구나 vm_get_frame() 에서는 공유자원이 사용되지 않음에도 발생하므로 이유를 찾을 수가 없었습니다.

위의 수정사항 외에도 swap out이 되더라도 frame을 공유할 수 있도록 구현을 하였습니다. eviction에는 처리하는 비용을 아끼기 위해 clock algorithm에 fork 유무를 포함하여 탐색하도록(아주 
약간 바꿈) 적용하였습니다.

마지막으로 아쉬운점은 lazy load data라는 aux에 적용하기 위해 불필요한 추가 매모리를 운용하였습니다. 시간이 있다면 적절히 압축하여 page에 적는방식으로 수정을 하는게 더 효과적일것이라 생각했습니다.

ubuntu 22.04 free tier로 test를 실시하였지만 swap-disk 공간 200m를 확보하지 못하여 swap-fork test를 통과할 수 없었습니다. 결국 다른 환경에서 실시함으로 결과를 확인할 수 있었습니다.(Make file을 수정할려고 노력했습니다만 개별 test에 대한 설정 위치를 찾을 수 없었습니다.)

pintos의 과정들은 전부 연결되어 있으므로 page_cache를 개발하는 과정에서 이전 문제로 겪지 않게 하기 위해(lock에서 문제가 생겼을 줄은 꿈에도 몰랐습니다.), 버그를 찾기 쉬운 코드를 위해 많은 번복을 하였습니다. 그럼에도 아직은 부족한, 버그가 또 생길수 있는 코드라고 생각했습니다.

project 3를 하면서 역시 gitbook에 적힌 순서대로 개발하며, 틈틈히 변경점을 추가하는것이 더 쉽다는 점과 처음부터 모든걸 고려해서 개발(하였다고 생각)하더라도 결국에 다시 수정해야된다는 점을 x고생하면서 깨달을 수 있었습니다.

### FileSystems

file system을 구현하면서 운영체제가 얼마나 사용자 친화 환경을 제공하는지 깨달을 수 있었습니다. window 와 linux에서 사용한 file system을 생각하여 따라 구현하였습니다.

project 1, 2, 3과 달리 file에관한 지식 없이 오로지 pintos의 코드에 대한 이해만으로 구현을 하려고 하니 너무 어려웠습니다.   

다른 project역시 시작은 힘들었지만 이후 구현은 과제라는 범주안에서 할 수 있었습니다. 하지만 마지막 file system은 불명확한 gitbook의 설명, guilde line이 없는 높은 자유도의 구현을 하려니 정말 난감하였습니다. 그래도 어려움에 비례한 성취감과 뿌듯함은 더 큰 성장을 할 수 있는 기회였습니다.

#### Indexed and Extensible Files

[완성본](https://github.com/eunsik-kim/pintos11/tree/eunsik/filesys/filesys/fat.c)

3주차 때 swap 영역을 bitmap을 이용해 swap table을 구현하였습니다. 마찬가지로 pintos의 basic file system은 free-map.c에서 bitmap으로 구현 되어있습니다. 다만 bitmap_filp_multiple을 사용하여 가용공간을 탐색하므로, 즉 disk의 가용공간을 사용하려면 인접한 영역들로만 사용할 수 있습니다. malloc병에 걸려있다면 외부단편화에 취약하다는 사실을 바로 알 수 있습니다.  그럼.. 고쳐야겠죠?  

`FAT`(file allocation table) 는 간단히 생각하면 4byte ptr를 가지는 단방향 연결 리스트입니다. disk의 영역인 sector와 fat 배열의 단위인 cluster는 1-1대응 관계가 성립하여 fat를 참조하여 disk공간을 할당해주면 됩니다.  

늘 했듯이 처음엔 (사용방법도 모른채로...) 다시고쳐야될 fat util함수를 만들면 됩니다. 아무것도 사용방법을 모른채, fat를 모른채 작업하는 방법은 늘 그랬듯이 적응이 되지 않아 힘들었습니다.
결국 vine test를 하며 debugging하기 전까지 늘 그랬듯이 틀린줄도 모르고 계속 사용했었습니다.

| fucntion          | list와 유사한 함수             
| :---------------- | :-------------------------------: |
| fat_create_chain   | list_insert  |
| fat_remove_chain | list_remove |
| fat_get | list_next |
| cluster_to_sector, sector_to_cluster | cluster와 sector를 호환하는 함수 |

위의 함수들은 stack형태의 list로 생각하면 이해하기가 쉽습니다. 하지만 cluster가 배열로 구성되어 있고, index 시작이 1부터되며, 미리 지정된 fat_sectors값에 의해 결정된다는 불편한점들이 있습니다.  

꼭 ASSERT를 넣어 debbuing을 하셔야 합니다. 저는 항상 cluster_to_sector에서 많이 에러가 발생했습니다. 이런 에러가 발생함을 인지는 할 수 있지만 그럼에도 어디부터 chain이 망가진건지 추측하기는 쉽지 않습니다. 

> fat와 free-map의 booting 과정은 같지만 일부 한가지가 다릅니다. 요약해보면 보면 fat_init() -> do_format -> fat_open 순서로 두과정 모두 동일하게 진행됩니다. 1.system을 초기화 한뒤 2. system을 create하고 3. close하면서 disk에 내용을 기록하고 4.마지막으로 다시 open하여 fat가 시작이 됩니다. 다만 중요한 다른점은 기존에 system은 root dir을 열어주지만 fat는 열어주지 않기에 따로 추가해줘야합니다. (이 내용이 gitbook에 없는점, root를 안열고 0으로 memset만 해주는 점은 이해가 되지 않습니다.)
{: .prompt-info }

그리고 file growth를 구현해야 합니다. 조건문을 해결하기 위해 10시간 이상을 debbuging에 했기 때문에 좀 이상하더라도 양해 부탁드립니다. 

[완성본](https://github.com/eunsik-kim/pintos11/tree/eunsik/filesys/filesys/inode.c/#563)

```
disk_sector_t file_growth (struct inode *inode, off_t size, off_t offset) {

	disk_sector_t sector, sector_idx;
	cluster_t clst, last_clst, off_clst;
	off_t create_cnt, add_length = offset + size - inode->data.length;
	off_t res = inode->data.length % DISK_SECTOR_SIZE;
	off_t temp = ((res + add_length) / DISK_SECTOR_SIZE - ((res + add_length) % DISK_SECTOR_SIZE == 0));

	// strange condition ....
	if ((inode->data.length == 0) || (!res && add_length > 0) || temp > 0) {
		lock_acquire(&inode->w_lock);		// file growth is atomic action
		if (inode->data.length == 0) {	// case 1
			clst = last_clst = sector_to_cluster(inode->data.start);	
			create_cnt = bytes_to_sectors(add_length);		
		} else {	// case 2
			if (!res && add_length > 0) {
				clst = last_clst = sector_to_cluster(byte_to_sector2(inode, inode->data.length));
				create_cnt = bytes_to_sectors(add_length);
			} else{
				clst = last_clst = sector_to_cluster(byte_to_sector(inode, inode->data.length));
				create_cnt = temp;		
```
{: file='filesys/inode.c'}

조건은 크게 5가지 입니다.
1. file length가 0인 file을 적을 때 늘려주기
2. file length가 512(sector_size)의 배수인 경우 늘려주기
3. sector_size와 offset+size가 512보다 커지는 경우 늘려주기
4. sector가 안늘어 나지만 file length가 늘어나는 경우
5. file length가 늘어나지 않는 경우

조금 더 좋은 조건을 생각하지 못했지만 위의 함수에서 res를 기준으로 생각하면 모든 경우가 포함됨을 알 수 있습니다. (포함 안되는 반례 찾으면 패배를 인정합니다...)

각 경우에 대해서 늘려준 만큼 chain을 추가하였고, 추가된 offset까지 0으로 채워줬습니다.  

그리고 동기화를 위해 file length가 바뀌는 경우 inode에 종속된 lock을 걸어 주었습니다.

| fucntion          | 변경된점             
| :---------------- | :-------------------------------: |
| inode_create   | length에 맞게 chain을 생성하여 sector를 할당 |
| byte_to_sector | chain을 통해 연결 list로 offset에 해당하는 sector를 찾기 |
| inode_close 	| 할당된 cluster chain을 삭제 |
| inode_read_at | cluster를 통해 연쇄적으로 읽기, lock 제거하고, length가 변동시 다시 읽기 |

특히 byte_to_sector를 작성하는것이 file-growth와 함께 시간이 많이 걸렸습니다. 계속 해서 수정하면서 생각하지 못하는 예외를 처리하는게 힘들었습니다.  

create한 만큼 offset을 할당해주면 오류가 발생하기 때문에 주의해야합니다. 그리고 sector와 cluster가 훼손되지 않았는지, inode를 잘못 덮어쓰지 않았는지 assert 구문으로 꼭 확인 해야 됩니다.

그리고 growth를 잘못 구현하면 모든 test가 통과하더라도 persistence가 안됩니다. 그렇기 때문에 의심 또 의심을 하면 (테스트에만?) 완벽한 함수를 찾을 수 있습니다. 여러분 힘내세요

##### syncronization

동시에 읽어 i/o 처리 효율 향상을 위해 block은 최대한 적게 하도록 노력했습니다. 구체적으로는 syn_read test와 syn-rw test를 통과하도록 inode write에서만 lock을 걸고 확인하도록 했습니다. file growth하는 경우에만 일관성을 보장받으면 되었습니다. 그러므로  filesys lock을 제거하고 inode에 의존하는 lock을 걸었습니다. 

gitbook에서 reader가 writer의 작성 후 정확한 변동사항을 읽을 수 있어야 한다고 했습니다. semaphore나 wrt_cnt같은 변수를 통해 진입점을 확인할 수 있다고 생각했습니다. 하지만 속도를 위해 단순히 file length 변동시 다시 읽도록 작성하였습니다. 그리고 fat에는 항상 lock이 걸려있고 disk에서 역시 lock이 걸려있기 때문에 함수에 동시에 진입하되 순차적으로 읽을 수 있다고 생각했습니다.

같은 directory혼자서 사용해야 됩니다. 그래서 directory에 종속적인 lock을 만들어 dir 내용을 확인하는 함수에 dir_readdir, dir_remove, dir_add에 lock을 걸었습니다.

#### Subdirectories and Soft Links

[완성본](https://github.com/eunsik-kim/pintos11/tree/eunsik/filesys/filesys/directory.c/#113)

```
/* 절대경로 or 상대경로를 받아와서 경로중 마지막 dir과 file_name을 반환하는 함수 */
struct dir *find_dir(char *origin_paths, char *file_name) {
	ASSERT (origin_paths != NULL);

	// save due to strtok_r
	paths = palloc_get_page(0);
	strlcpy(paths, origin_paths, strlen(origin_paths) + 1);

	if (cwd && paths[0] != '/')    // 상대경로
		next_dir = dir_reopen(cwd);
	else
		next_dir = dir_open_root();		

	// parse paths
	cur_path = strtok_r(paths, "/", &rest_path);
	if ((cwd && paths[0] != '/') && !cur_path) { // if relative path is null_path
		dir_close(next_dir);
		palloc_free_page(paths);
		return NULL;
	}

	next_path = strtok_r(NULL, "/", &rest_path);

	while (next_path) {
		if (strlen(next_path) > NAME_MAX)	// validate file name
			return NULL;

		if (!dir_lookup(next_dir, cur_path, &inode)) {	// find sub dir
			dir_close(next_dir);
			palloc_free_page(paths);
			return NULL;
		}
		dir_close(next_dir);
		next_dir = dir_open(inode);
		if (!next_dir->inode->data.isdir) { 	// wrong path
			dir_close(next_dir);
			palloc_free_page(paths);
			return NULL;
		}		

		cur_path = next_path;
		if (strstr(rest_path, "/")) {
			next_path = strtok_r(NULL, "/", &rest_path);
			if (!next_path)		// null indicates current directory
				cur_path =".";
		} else
			next_path = strtok_r(NULL, "/", &rest_path);
	}
	strlcpy(file_name, cur_path, strnlen(cur_path, NAME_MAX) + 1);
	palloc_free_page(paths);
	return next_dir;
}

```
{: file='filesys/directory.c'}

sub directory를 구현하는데 가장 고민했던 부분은 path를 parsing하여 효율적으로 다른 함수들을 활용 할 수 있는 방법을 찾아내는 것이였습니다. 그래서 그냥 parsing을 먼저 하다보니 위와 같은 함수를 만들 수 있었습니다. 가장 주의해야 할 부분은 overflow가 생기지 않도록 filename을 malloc을 해야합니다. 입력으로 주어지는 path는 strtok를 사용하면 변형이 되므로 복사하여 활용해야 합니다.

overflow를 조심 또 조심해야 합니다. 처음엔 인지못한다면 나중에 버그를 울면서 찾을 수도 있습니다. 위의 함수를 만들고 나면 filesys에서 file을 탐색하는 과정을 수정하고 그것들을 활용해 syscall의 함수들을 만들면 쉽게 할 수 있습니다. userporg에서 했던것처럼 하면됩니다.

##### current working directory

현재 사용자의 directory를 thread에 저장을 합니다. 저장된 cwd는 닫히지 않게 하도록 cwd_cnt를 만들어 directory를 함부로 닫지 못하게 하였습니다. 같은 방식으로 root도 제어 할 수 있습니다.

##### symlink

symlink는 바로가기를 생성하는것입니다. 제가 생각한 바로가기는 해당하는 file의 inode위치를 가르키도록 하는것입니다.   

하지만 sym_link test에선 아직 생성되지 않은 file까지 directing하고 있음을 알 수 있습니다.... 그래서 path을 저장하여 관리하였고 참조시 해당 파일을 찾아 바뀌도록 구현하였습니다. 그리고 참조가 끝나면 원래 link로 돌아오도록 구현하였습니다.   

하지만 한번더 놀랍게도 persistence test에서 link 파일을 file로 취급하지 않고 directory로 취급하라고 하였습니다... 그래서 참조하는 즉시 그 내용을 복사하여 저장하도록 수정하였습니다.

1. symlink 파일을 따로 생성하여 target file의 내용을 보도록 구현
2. symlink 파일에 path를 저장하여 참조시 확인하도록 구현
3. symlink 파일이 다른 파일에 의해 참조되는 경우 target파일의 inode를 저장하도록 구현
4. 요약하면 머리가 나쁘면 몸이 고생한다는 사실을 알 수 있었습니다.

### page_caches

[need more time]

### Summary

드디어 pintos가 끝이 났습니다. 밤과 낮을 많이 샌만큼 재미도 있었습니다. 무엇보다 c언어이긴 하지만 pintos code를 보면서 style과 design도 많이 배울 수 있어서 좋았습니다.

매주마다 조금씩 더 어려워지고, 자유도가 높아지면서 마지막 file system에선 정말 자유롭게 코딩을 하며 성장하는 느낌을 받을 수 있었습니다. 

특히 하면서 c언어의 모든 에러와 버그들을 다 경험힐 수 있어서 뿌듯했습니다.

kernel의 기능 일부분을 다양한 자료구조와 register(asm), 알고리즘들을 실제로 적용해보는 경험이었습니다. 이를 통해 cs지식도 얻고 경험도 얻을 수 있는 좋은 과제라고 생각했습니다.

열심히 했지만 마지막 extra는 기한내에 못해서 아쉽다고 생각합니다. 남은 과정들에선 이런일이 없도록 시간계획을 잘 설정해서 더 열심히, 끝까지 최선을 다해서 마무리 하도록 하겠습니다.