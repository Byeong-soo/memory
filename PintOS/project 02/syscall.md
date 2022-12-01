1 주차 [[1주차 thread (수정할것)]]


-   깃은 기본적으로 중요함. 정말 강조해도 부족하지 않음.
-   깃 덕분에 포기하지 않고 올수 있었음.
-   처음에 구현을 하기 위해서 잘못된 지식으로 구현을 많이했었음. 덕분에 배운 것도 많았지만 시간 소비가 컸음
-   디버깅 파일을 만들어서 임포트 해서 쓰면 좋을꺼 같음. 매번 비슷한 파일 만드는거 같아서 시간낭비 같음.
-   어쨌든 재밌었당 :)

 [깃북 번역본](https://www.notion.so/KAIST-PINTOS-ebdc8be9d02d4475a4675c7b920e3653)
 
## userprogram

pintos가 부팅되고, 주어진 명령인자로 실행하게 됨.

처음에 run testName으로 파일이 실행되게 되고 파일이 실행되면 exec로 빠지게 된다. 그동안 main 스레드는 wait()를 하게 된다. 핀토스는 단일 스레드체계(?) 이고 wait는 자기가 생성한 자식 스레드에서만 한정되게 되어있다. (원래 자식만 기다리는지 확인 필요)

exec에서 받은 filename이 실행가능한 파일인지 아닌지 확인을 먼저함. 헤더를 읽어서 ELF 파일인지 먼저 확인을 하고 그다음에 phdr 을 읽어 정보를 확인한다.

```c
/* Create a minimal stack by mapping a zeroed page at the USER_STACK */
static bool
setup_stack(struct intr_frame *if_)
{
	uint8_t *kpage;
	bool success = false;

	kpage = palloc_get_page(PAL_USER | PAL_ZERO);
	if (kpage != NULL)
	{
		success = install_page(((uint8_t *)USER_STACK) - PGSIZE, kpage, true);
		if (success)
			if_->rsp = USER_STACK;
		else
			palloc_free_page(kpage);
	}
	return success;
}
```

그런 다음 rsp(스택포인터)fmf USER_STACK 으로 맞춘다. rip를 설정하고, 실행파일을 포인터로 스레드의 구조체에 저장한다. ( 처음에는 리스트로 관리를 했지만, 실행파일은 한가지만 유지될 수있고, 파일을 새로 실행하게 된다면, process_clean 작업에서 모두 날아가게 된다. 그런다음 명령어로 받은 인자들을 스택에 넣어주고, 파일에서 실행될 인자를 rdi 과 rsi에 포인터를 담아준다. 그럼 스레드는 do_iret으로 가 컨텍스트 스위칭을 하게 된다.

```c
/* Use iretq to launch the thread */
void do_iret(struct intr_frame *tf)
{
	__asm __volatile(
		"movq %0, %%rsp\\n"
		"movq 0(%%rsp),%%r15\\n"
		"movq 8(%%rsp),%%r14\\n"
		"movq 16(%%rsp),%%r13\\n"
		"movq 24(%%rsp),%%r12\\n"
		"movq 32(%%rsp),%%r11\\n"
		"movq 40(%%rsp),%%r10\\n"
		"movq 48(%%rsp),%%r9\\n"
		"movq 56(%%rsp),%%r8\\n"
		"movq 64(%%rsp),%%rsi\\n"
		"movq 72(%%rsp),%%rdi\\n"
		"movq 80(%%rsp),%%rbp\\n"
		"movq 88(%%rsp),%%rdx\\n"
		"movq 96(%%rsp),%%rcx\\n"
		"movq 104(%%rsp),%%rbx\\n"
		"movq 112(%%rsp),%%rax\\n"
		"addq $120,%%rsp\\n"
		"movw 8(%%rsp),%%ds\\n"
		"movw (%%rsp),%%es\\n"
		"addq $32, %%rsp\\n"
		"iretq"
		:
		: "g"((uint64_t)tf)
		: "memory");
}
```

컨텍스트 스위칭을 하며 프로그램이 시작되는 부분으로 Rsp가 점프를 하게된다.프로그램이 실행되면서 필요한 syscall들을 호출하게 된다.

시스템 콜을 호출하게 되면,

```c
bool
create (const char *file, unsigned initial_size) {
	return syscall2 (SYS_CREATE, file, initial_size);
}
```

이런식으로, 호출에 맞는 syscall로 변환된다.

```c
__attribute__((always_inline))
static __inline int64_t syscall (uint64_t num_, uint64_t a1_, uint64_t a2_,
		uint64_t a3_, uint64_t a4_, uint64_t a5_, uint64_t a6_) {
	int64_t ret;
	register uint64_t *num asm ("rax") = (uint64_t *) num_;
	register uint64_t *a1 asm ("rdi") = (uint64_t *) a1_;
	register uint64_t *a2 asm ("rsi") = (uint64_t *) a2_;
	register uint64_t *a3 asm ("rdx") = (uint64_t *) a3_;
	register uint64_t *a4 asm ("r10") = (uint64_t *) a4_;
	register uint64_t *a5 asm ("r8") = (uint64_t *) a5_;
	register uint64_t *a6 asm ("r9") = (uint64_t *) a6_;

	__asm __volatile(
			"mov %1, %%rax\\n"
			"mov %2, %%rdi\\n"
			"mov %3, %%rsi\\n"
			"mov %4, %%rdx\\n"
			"mov %5, %%r10\\n"
			"mov %6, %%r8\\n"
			"mov %7, %%r9\\n"
			"syscall\\n"
			: "=a" (ret)
			: "g" (num), "g" (a1), "g" (a2), "g" (a3), "g" (a4), "g" (a5), "g" (a6)
			: "cc", "memory");
	return ret;
}
```

알맞는 시스템콜로 변환되어 레지스터에 넣은 값들이 차례대로 위에서부터 rax, rdi, rsi, rdx … 에 들어가게 된다.

그럼 pintos 기준 syscall-entry파일 로 넘어가게 된다.

```nasm
#include "threads/loader.h"

.text
.globl syscall_entry
.type syscall_entry, @function
syscall_entry:
	movq %rbx, temp1(%rip)
	movq %r12, temp2(%rip)     /* callee saved registers */
	movq %rsp, %rbx            /* Store userland rsp    */
	movabs $tss, %r12
	movq (%r12), %r12
	movq 4(%r12), %rsp         /* Read ring0 rsp from the tss */
	/* Now we are in the kernel stack */
	push $(SEL_UDSEG)      /* if->ss */
	push %rbx              /* if->rsp */
	push %r11              /* if->eflags */
	push $(SEL_UCSEG)      /* if->cs */
	push %rcx              /* if->rip */
	subq $16, %rsp         /* skip error_code, vec_no */
	push $(SEL_UDSEG)      /* if->ds */
	push $(SEL_UDSEG)      /* if->es */
	push %rax
	movq temp1(%rip), %rbx
	push %rbx
	pushq $0
	push %rdx
	push %rbp
	push %rdi
	push %rsi
	push %r8
	push %r9
	push %r10
	pushq $0 /* skip r11 */
	movq temp2(%rip), %r12
	push %r12
	push %r13
	push %r14
	push %r15
	movq %rsp, %rdi

check_intr:
	btsq $9, %r11          /* Check whether we recover the interrupt */
	jnb no_sti
	sti                    /* restore interrupt */
no_sti:
	movabs $syscall_handler, %r12
	call *%r12
	popq %r15
	popq %r14
	popq %r13
	popq %r12
	popq %r11
	popq %r10
	popq %r9
	popq %r8
	popq %rsi
	popq %rdi
	popq %rbp
	popq %rdx
	popq %rcx
	popq %rbx
	popq %rax
	addq $32, %rsp
	popq %rcx              /* if->rip */
	addq $8, %rsp
	popq %r11              /* if->eflags */
	popq %rsp              /* if->rsp */
	sysretq

.section .data
.globl temp1
temp1:
.quad	0
.globl temp2
temp2:
.quad	0
```

-   CPU 레지스터 값들을 커널 스택에 push : push 되는 순서는 마지막 들어간 데이터부터 interrupt frame 구조체 순서가 되도록 하는 순서로 push💡 이때 SEL_UDSEG는 유저 데이터 세그먼트, SEL_UCSEG는 유저 코드 세그먼트를 의미한다.
    
-   커널 스택에 인터럽트 프레임 구조체 형태의 데이터가 다 들어가 있고, rsp는 이 시작점을 가리키는 형태 → `syscall_handler()` 호출 시 인자로 rsp를 넘겨줌 (핸들러는 *if 를 받았다고 생각할 수 있음)
    
    ```c
    /* syscall-entry.S 부분 */
    
    movabs $syscall_handler, %r12
    	call *%r12
    ```
    

`syscall_handler (struct intr_frame *f)`: 인자로 들어온 f는 커널스택의 rsp.

-   rsp부터는 intr_frame 형태의 데이터들이 순차적으로 들어가 있기 때문에 이 자체를 인터럽트 프레임처럼 사용할 수 있는 것 → f의 rax 값을 통해 필요한 시스템 콜을 수행하는 함수를 호출한다.

### 0 : halt

```c
void syscall_halt(void){
	power_off();	
}
```

### gitbook

<aside> 🙂 power_off()를 호출해서 Pintos를 종료합니다. (power_off()는 src/include/threads/init.h에 선언되어 있음.) 이 함수는 웬만하면 사용되지 않아야 합니다. deadlock 상황에 대한 정보 등등 뭔가 조금 잃어 버릴지도 모릅니다.

</aside>

-   halt의 구현은 간단하다. power off를 호출해서 종료하면 끝.

### 1 : exit

```c
void syscall_exit(struct intr_frame *f){
	thread_current()->exit_code = f->R.rdi;
	thread_exit();
}
```

### gitbook

<aside> 🙂 현재 동작중인 유저 프로그램을 종료합니다. 커널에 상태를 리턴하면서 종료합니다. 만약 부모 프로세스가 현재 유저 프로그램의 종료를 기다리던 중이라면, 그 말은 종료되면서 리턴될 그 상태를 기다린다는 것 입니다. 관례적으로, 상태 = 0 은 성공을 뜻하고 0 이 아닌 값들은 에러를 뜻 합니다.

</aside>

-   프로그램이 return 해주거나 프로그램 안에서 exit를 호출하면 들어가는 함수.
-   exit를 통해서 현재 스래드의 상태코드를 반환하고 종료한다.
-   exit(n)을 통해 호출하면 n을 리턴하며 종료된다. 하지만 정상적인 종료가 아닌 비정상 적인 종료를 할 경우에는 return 값으로 -1을 리턴한다.
-   exit() call을 받아서 tread_exit()으로 간다면, syscall과 여러 작을 통해 생긴 자원(구조체) 들을 정리하는 작업을 거친다. 부모 스레드가 가지고 있는 ,자식 스레드의 tid와 exit_code를 업데이트 해준다. 초기값으로 -2가 잡혀있기 때문에 -2가 아니면 뒤에 나오는 wait의 while 문에서 루프를 타면서 sema_down을 하게 된다. 부모는 주어진 pid를 가진 자식이 죽을 때 까지 계속해서 잠들어 있어야 하기 때문!

```c
// 자식 스레드의 상태를 저장하는 구조체! 자식을 한번 기다렸거나, 자기자신(부모)가 종료될때 모두 삭제해주어야 한다.
struct child_info
{
	uint32_t tid;
	int exit_code;
	struct list_elem elem;
};
```

-   그다음 본인이 가지고 있는 children_list를 지운다. pintos에서는 리스트가 이중연결 리스트로 구현되어 제공되기 때문에 리스트가 empty가 될때 까지 순환하면서 제거를 하면 된다.
-   그다음 내가 가지고 있는 fd_list도 삭제해준다.

```c
//* eg) while 돌면서 fd_list 삭제. 순환하면서 삭제하는 부분은 모두 이렇게 구현했다.
// 하지만 remove를 쓰지않고 pop을 해서 쓰면 더욱 편하다. 코드 주석에도 그 방법을 추천함.

void clear_fd_list()
{

	struct list *fd_list;
	struct fd *delete_fd;

	fd_list = &thread_current()->fd_list;

	if (list_empty(fd_list))
		return;

	struct list_elem *cur;
	cur = list_begin(fd_list);
	while (cur != list_end(fd_list))
	{
		delete_fd = list_entry(cur, struct fd, elem);

		file_lock_acquire();
		file_close(delete_fd->file);
		file_lock_release();

		cur = list_remove(&delete_fd->elem);
		free(delete_fd);
	}
}
```

-   항상 file에 관한 작업을 할때는 lock을 걸어줘서 동기화를 걸어주자. 그렇지 않으면 테스트케이스가 성공했다가 실패했다가.. 운에 맡겨야 한다.
-   구조체를 선언해 줬다면 무조건 free!!! 메모리 누수가 생겨선 안된다.
-   그리고 스레드가 파일을 실행시키고 있었다면, 그 파일을 닫아준다. exec 부분에서 실행 중인 파일은 쓰기가 불가능 하기 때문에, load 과정에 있던 close를 삭제하고, file_deny를 써서 쓰기 권한을 없앴다. 그렇기 때문에 파일을 종료할때 close 해줘야 한다.
-   마지막으로 할당받은 메모리를 해제해준다.

```c
uint64_t *pml4;
	/* Destroy the current process's page directory and switch back
	 * to the kernel-only page directory. */
	pml4 = curr->pml4;
	if (pml4 != NULL)
	{
		/* Correct ordering here is crucial.  We must set
		 * cur->pagedir to NULL before switching page directories,
		 * so that a timer interrupt can't switch back to the
		 * process page directory.  We must activate the base page
		 * directory before destroying the process's page
		 * directory, or our active page directory will be one
		 * that's been freed (and cleared). */
		curr->pml4 = NULL;
		pml4_activate(NULL);
		pml4_destroy(pml4);
```

-   이 과정이 끝나면 스레드는 destruction_que에 들어가 victim으로 만들어져 삭제되게 된다.

### 2: fork
test : [[shell 스크립트 while문]]

```c
pid_t syscall_fork (struct intr_frame *f){

	char * thread_name = f->R.rdi;
	int return_value;
	return_value = process_fork(thread_name, f);
	f->R.rax = return_value;
}
```

### gitbook

<aside> 🙂 THREAD_NAME이라는 이름을 가진 현재 프로세스의 복제본인 새 프로세스를 만듭니다.

피호출자(callee) 저장 레지스터인 %RBX, %RSP, %RBP와 %R12 - %R15를 제외한 레지스터 값을 복제할 필요가 없습니다. 자식 프로세스의 pid를 반환해야 합니다. 그렇지 않으면 유효한 pid가 아닐 수 있습니다. 자식 프로세스에서 반환 값은 0이어야 합니다. 자식 프로세스에는 파일 식별자 및 가상 메모리 공간을 포함한 복제된 리소스가 있어야 합니다. 부모 프로세스는 자식 프로세스가 성공적으로 복제되었는지 여부를 알 때까지 fork에서 반환해서는 안 됩니다. 즉, 자식 프로세스가 리소스를 복제하지 못하면 부모의 fork() 호출이 TID_ERROR를 반환할 것입니다. 템플릿은 `threads/mmu.c`의 `pml4_for_each`를 사용하여 해당되는 페이지 테이블 구조를 포함한 전체 사용자 메모리 공간을 복사하지만, 전달된 `pte_for_each_func`의 누락된 부분을 채워야 합니다. ([가상 주소](https://casys-kaist.github.io/pintos-kaist/appendix/virtual_address.html)) 참조).

</aside>

-   fork는 기본적으로 시스템 콜을 받으면 process_fork를 호출하게 된다. 현재 intr_frame과 스레드의 name을 인자로 주게 된다.

```c
/* Clones the current process as `name`. Returns the new process's thread id, or
 * TID_ERROR if the thread cannot be created. */

tid_t process_fork(const char *name, struct intr_frame *if_)
{

	// 	/* Clone current thread to new thread.*/

	struct fork_info *fork_info = (struct fork_info *)malloc(sizeof(struct fork_info));
	fork_info->parent_t = thread_current();
	fork_info->if_ = if_;

	// 포크 하기 전에 스택정보(_if)를 미리 복사 떠놓는 중. 포크로 생긴 자식에게 전해주려고
	tid_t pid = thread_create(name, PRI_DEFAULT, __do_fork, fork_info);

	process_fork_sema_down();

	if (!thread_current()->make_child_success)
	{
		return -1;
	}
	return pid;
}
```

-   thread_creaate()에 여분으로 줄수 있는 인자는 1자리 뿐이다(void*). 하지만 넘겨줘야할 데이터는 2개이기 때문에 나는 구조체를 사용해서 넘겨줬다. 사실 스택에 쌓이기 때문에 배열로 넘겨주면 되지만, 부끄럽게도 아직까지 c언어의 배열을 사용해본적이 없다….!
-   여기서 주의할 점은 스레드가 생성되고나면, Pid를 반환해주고 error가 뜨면 -1를 리턴해주면 된다. 처음에는 당연히 새로생긴 자식 스레드가 __do_fork를 하는 과정에서도 실패를 하면 pid가 -1을 받을거라고 생각했지만 따로 처리를 해줘야 했었다. exit_code를 사용해서 -1이라면 -1리턴을 해주는 방식으로 했으면 됬는데, 당시에 생각이 잘 나지않아서 구조체 자체에 자식 스레드 생성의 성공여부를 판단하는 bool 값을 넣어서 판별했다.
-   부모가 create_thread를 끝내면 자식이 fork를 하는 과정에서 성공적으로 파일을 모두 복사했는지 확인해야 한다. 그래서 부모는 sema_down을 통해서 wait 상태가 된다. 자식이 fork를 끝내고 실패 여부에 따라서 -1을 리턴할지 아니면 pid를 리턴할지 나눠진다.

```c
/* A thread function that copies parent's execution context.
 fork할 때 부모프로세스의 context(유전자)를 복사하는 함수
 * Hint) parent->tf does not hold the userland context of the process.
 *       That is, you are required to pass second argument of process_fork to
 *       this function. */

static void
__do_fork(void *aux)
{
	struct intr_frame if_;
	struct fork_info *fork_info = (struct fork_info *)aux;
	struct thread *parent = fork_info->parent_t;
	struct thread *current = thread_current();
	/* TODO: somehow pass the parent_if. (i.e. process_fork()'s if_) */
	struct intr_frame *syscall_if;
	syscall_if = fork_info->if_;

	bool succ = true;
	/* 1. Read the cpu context to local stack. */
	memcpy(&if_, syscall_if, sizeof(struct intr_frame));

	/* 2. Duplicate PT */
	current->pml4 = pml4_create();
	if (current->pml4 == NULL)
		goto error;
	process_activate(current);

#ifdef VM
	supplemental_page_table_init(&current->spt);
	if (!supplemental_page_table_copy(&current->spt, &parent->spt))
		goto error;
#else

	if (!pml4_for_each(parent->pml4, duplicate_pte, fork_info))
	{
		goto error;
	}
#endif
	/* TODO: Your code goes here.
	 * TODO: Hint) To duplicate the file object, use `file_duplicate`
	 * TODO:       in include/filesys/file.h. Note that parent should not return
	 * TODO:       from the fork() until this function successfully duplicates
	 * TODO:       the resources of parent.*/

	copy_fd_list(parent, current);
	process_init();
	/* Finally, switch to the newly created process. */

	if (succ)
	{
		parent->make_child_success = true;
		free(fork_info);
		if_.R.rax = 0;
		process_fork_sema_up();
		thread_yield();
		do_iret(&if_);
	}
error:
	parent->make_child_success = false;
	free(fork_info);
	thread_current()->exit_code = -1;
	del_child_info();
	process_fork_sema_up();
	thread_exit();
}
```

-   자식이 fork에서 복사하는 과정이다.
-   전달받은 if를 자기자신에게 memcpy로 복사한 후, pml4를 create 해서 할당 받는다.
-   다 복사를 하고, fd_list를 복사 받는다. fd_list도 순회하면서 다 복사하는 형식.
-   중간중간 과정 전부다, 실패를 하게 된다면 error 부분으로 뛰어서 에러에 대한 처리를 해준다.
-   parent에 결과를 담는 bool 값을 false로 바꿔주고 그 외에 malloc으로 할당받은 부분들을 free 해준다. 상태도 -1로 바꾸고 tread_exit()으로 빠져나가게 된다.
-   성공한다면 자식의 리턴 값은 0이어야 하므로 복사한 if에 0을 담아서 리턴해준다. 성공으로 왔다면 succ가 된거기 때문에 sema up으로 부모를 꺠워주고 다시 제어권을 전달해준다. 부모가 wait에 걸리거나 종료되게 되어 자식이 일어나게 된다면 do_iret으로 분기에서 깨어나서 작업을 수행하게 된다.

```c
/* Duplicate the parent's address space by passing this function to the
 * pml4_for_each. This is only for the project 2. */
static bool
duplicate_pte(uint64_t *pte, void *va, void *aux)
{
	struct thread *current = thread_current();
	struct fork_info *fork_info = (struct fork_info *)aux;
	struct thread *parent = fork_info->parent_t;

	void *parent_page;
	void *newpage;
	bool writable;

	/* 1. TODO: If the parent_page is kernel page, then return immediately.*/
	if is_kernel_vaddr (va)
	{
		return true;
	}

	/* 2. Resolve VA from the parent's page map level 4.
	pml4_get_page는 "물리주소를 찾는 함수"임. 누구의 물리주소를 찾냐면? 유저영역 쪽에 있는 가상주소 va(부모스레드)의 물리주소를 찾는 것.
	pml4_get_page가 리턴하는 거는 "커널주소"를 리턴한다. 어떤 커널주소를 리턴하냐면? 찾은 물리주소와 연결된 커널의 주소를 리턴함.
	즉 va의 물리주소를 찾아서 그 물리주소와 연결 되어있는 커널주소를 반환하는 함수임. if)물리주소가 매핑 안되어있으면 NULL반환 */

	parent_page = pml4_get_page(parent->pml4, va);
	if (parent_page == NULL)
	{
		return false;
	}

	/* 3. TODO: Allocate new PAL_USER page for the child and set result to NEWPAGE.*/
	newpage = palloc_get_page(PAL_USER | PAL_ZERO);
	if (newpage == NULL)
	{
		palloc_free_page(newpage);
		return false;
	}

	/*페이지를 할당 받을 건데 PAL_USER플레그를 줌으로써 유저가 쓸 수 있는 메모리 pool에서 페이지를 가져올거고
	PAL_ZERO를 씀으로써 할당받은 페이지 메모리를 0으로 초기화 할 거임.
	<https://casys-kaist.github.io/pintos-kaist/appendix/memory_allocation.html> 참고*/

	/* 4. TODO: Duplicate parent's page to the new page and
	 *    TODO: check whether parent's page is writable or not (set WRITABLE
	 *    TODO: according to the result). */
	memcpy(newpage, parent_page, PGSIZE);
	writable = is_writable(pte);

	/* 5. Add new page to child's page table at address VA with WRITABLE
	 *    permission. */

	if (!pml4_set_page(current->pml4, va, newpage, writable))
	{
		/* 6. TODO: if fail to insert page, do error handling. */
		palloc_free_page(newpage);
		return false;
	}
	return true;
}
#endif
```

-   fork 안에 과정중에서 pte를 복사하는 과정이 있다.
-   커널 영역을 만나면 true를 반환해서 작업을 종료하게 한다.( 커널영역을 만난다는 얘기는 유저영역 복사를 끝냈다는 말)
-   parent_thread에게서 pml4를 가져온다. 그다음 palloc_get_page로 newpage를 받아와서 memcpy로 복사를 해준다. 물론 중간 작업에서 오류를 만난다면 할당받은 메모리들을 해제하면서 리턴해준다.

### 3 : exec

```c
// exec func parameter : const char *cmd_line
// int syscall_exec (const char *cmd_line){
int syscall_exec (struct intr_frame *f){
   //! 지금까지 exec 안되던 이유 = palloc_get_page함수 호출하면서 palloc.h 파일 include 안해서.. 
   //! 저부터 머리 박습니다.. - yj 
	char *file_name = f->R.rdi;
	char *fn_copy ;

	check_addr(file_name);
	fn_copy = palloc_get_page (0); 
	if (fn_copy == NULL)
	{
		syscall_abnormal_exit(EXIT_CODE_ERROR);
		palloc_free_page(fn_copy);
		f->R.rax = -1;
		return -1;
	}
	strlcpy (fn_copy, file_name, PGSIZE); // filename을 fn_copy로 복사 
	if (process_exec (fn_copy) < 0) {
		palloc_free_page(fn_copy);
		f->R.rax = -1;
		syscall_abnormal_exit(-1);
	}
}
```

### gitbook

<aside> 🙂 현재의 프로세스가 cmd_line에서 이름이 주어지는 실행가능한 프로세스로 변경됩니다. 이때 주어진 인자들을 전달합니다. 성공적으로 진행된다면 어떤 것도 반환하지 않습니다. 만약 프로그램이 이 프로세스를 로드하지 못하거나 다른 이유로 돌리지 못하게 되면 exit state -1을 반환하며 프로세스가 종료됩니다. 이 함수는 exec 함수를 호출한 쓰레드의 이름은 바꾸지 않습니다. file descriptor는 exec 함수 호출 시에 열린 상태로 있다는 것을 알아두세요.

</aside>

-   맨위 설명 동일

### 4 : wait

```c
// wait func parameter : pid_t pid
int syscall_wait (struct intr_frame *f){
	int pid = f->R.rdi;
	struct child_info * child_info = search_children_list(pid);

	if(child_info == NULL){
		f->R.rax = -1;
		return -1;
	}

	int return_value;

	if(child_info->exit_code == EXIT_CODE_DEFAULT){
		return_value = process_wait(pid);
		f->R.rax = return_value;
	}else{
		return_value = child_info->exit_code;
		f->R.rax = return_value;
	}

	list_remove(&child_info->elem);
	free(child_info);
	return return_value;
}
```

### gitbook

<aside> 🙂 자식 프로세스 (pid) 를 기다려서 자식의 종료 상태(exit status)를 가져옵니다. 만약 pid (자식 프로세스)가 아직 살아있으면, 종료 될 때 까지 기다립니다. 종료가 되면 그 프로세스가 exit 함수로 전달해준 상태(exit status)를 반환합니다.

만약 pid (자식 프로세스)가 exit() 함수를 호출하지 않고 커널에 의해서 종료된다면 (e.g exception에 의해서 죽는 경우), wait(pid) 는 -1을 반환해야 합니다.

부모 프로세스가 wait 함수를 호출한 시점에서 이미 종료되어버린 자식 프로세스를 기다리도록 하는 것은 완전히 합당합니다만, 커널은 부모 프로세스에게 자식의 종료 상태를 알려주든지, 커널에 의해 종료되었다는 사실을 알려주든지 해야 합니다.

</aside>

### fail, -1 반환 조건

<aside> 🙂 다음의 조건들 중 하나라도 참이면 `wait` 은 즉시 fail 하고 `-1` 을 반환합니다 `pid` 는 호출하는 프로세스의 직속 자식을 참조하지 않습니다. 오직 호출하는 프로세스가 `fork()` 호출 후 성공적으로 `pid`를 반환받은 경우에만, `pid` 는 호출하는 프로세스의 직속 자식입니다.

자식들은 상속되지 않는다는 점을 알아두세요 : 만약 A 가 자식 B를 낳고 B가 자식 프로세스 C를 낳는다면, A는 C를 기다릴 수 없습니다. 심지어 B가 죽은 경우에도요. 프로세스 A가 `wait(C)` 호출하는 것은 실패해야 합니다.

마찬가지로, 부모 프로세스가 먼저 종료되버리는 고아 프로세스들도 새로운 부모에게 할당되지 않습니다. `wait` 을 호출한 프로세스는 이미  `pid` 에서 `wait` 을 호출한 상태입니다. 즉, 프로세스는 최대 한 번 주어진 자식 프로세스를 기다려야합니다.

프로세스들은 자식을 얼마든지 낳을 수 있고 그 자식들을 어떤 순서로도 기다릴 (`wait`) 수 있습니다. 자식 몇개로부터의 신호는 기다리지 않고도 종료될 수 있습니다. (전부를 기다리지 않기도 합니다.)

여러분의 설계는 발생할 수 있는 기다림의 모든 경우를 고려해야합니다. 한 프로세스의 (그 프로세스의 `struct thread` 를 포함한) 자원들은 꼭 할당 해제되어야 합니다.

부모가 그 프로세스를 기다리든 아니든, 자식이 부모보다 먼저 종료되든 나중에 종료되든 상관없이 이뤄져야 합니다.

**최초의 process가 종료되기 전에 Pintos가 종료되지 않도록 하십시오.**

제공된 Pintos 코드는 `main()` (in `threads/init.c`)에서 `process_wait()` (in `userprog/process.c` ) 를 호출하여 Pintos가 최초의 process 보다 먼저 종료되는 것을 막으려고 시도합니다.

여러분은 함수 설명의 제일 위의 코멘트를 따라서 `process_wait()` 를 구현하고 `process_wait()` 의 방식으로 wait system call을 구현해야 할 겁니다.

</aside>

-   기다려야하는 pid를 가진 스레드를 부모가 가진 child_info_list에서 검색을 한다. 만약 null이라면 자식이 아니거나 이미 기다렸던 스레드이기 때문에 기다리지않고 -1을 리턴한다.
-   -2 와 같다면 존재는 하지만 아직 종료되지 않은 스레드이기 때문에 다시 sema_down으로 들어가게 된다. 그게 아니라면 현재 가지고 있는 child_info 구조체의 exit_code를 리턴해 준다. 조건문을 나오고 나면 remove를 통해서 삭제해주고 할당받은 메모리를 free 해준다.

```c
/* Waits for thread TID to die and returns its exit status.
 * If it was terminated by the kernel (i.e. killed due to an exception), returns -1.
 * If TID is invalid or if it was not a child of the calling process, or if process_wait()
 *  has already been successfully called for the given TID, returns -1 immediately, without waiting.
 * This function will be implemented in problem 2-2.  For now, it does nothing. */
int process_wait(tid_t child_tid)
{
	// 이 함수가 호출되는 init.c의 main 함수를 보면, process_wait() 다음 thread_exit으로 쓰레드를 종료시킴.
	// 따라서 이 함수가 child_tid가 종료되기를 기다리는 동안 무한루프를 써서 기다리게 한다.
	/* The pintos exit if process_wait (initd),
	  we recommend you to add infinite loop here before implementing the process_wait. */

	struct list *child_list = &thread_current()->children_list;
	struct semaphore *sema;

	sema = &thread_current()->wait_sema;

	while ((search_children_list(child_tid))->exit_code == EXIT_CODE_DEFAULT)
	{
		wait_sema_down(sema);
	}

	return search_children_list(child_tid)->exit_code;
}
```

아래 시스템콜들은 4주차에 제대로 구현할꺼기 때문에 복기를 위한 코드는 생략.

---

### 5 : create

-   특별한건 없고 create으로 파일을 생성해준다. intr을 받아서 실행하는데 filesys_create를 통해서(이미 랩핑되어있음, 4주차에 파일시스템구현이 있어 지금은 정신 뺏기지 않기위해) create을 하면 되는데 lock을 걸어 원자성을 확보해주어야한다.

### 6 : remove

-   동일하게 파일시스템 호출

### 7 : open

-   open시에 fd를 관리해줘야하는데 나는 구조체로 만들어서 관리를 해줬다. file_open 시에 NULL 이 넘어오면 실패한거기 때문에 -1 를 리턴해줬다.

### 8 : filesize

-   fd로 크기를 읽어오는 함수. filesys에 랩핑된 함수가 있지만, 처음에 못찾아서 inode를 참조해서 크기를 가져왔다.

### 9 : read

-   열린 파일 중에서 read함수로 안에 내용을 읽어온다.

### 10 : write

-   fd 가 1이면 putbuf를 사용해서 화면에 적어준다. 1이 아니라면 fd를 찾아와서 그 파일에 쓴다.

### 11 : seek

-   받은 인자만큼 fd의 Posfmf dhfarlsek.

### 12 : tell

-   다음 써지거나 읽혀질 바이트의 위치를 반환. 시작점 부터 표현된다.

### 13 : close

-   파일의 식별자를 닫아준다. fd목록도 같이 관리해줘야 하기때문에 free를 해준다.