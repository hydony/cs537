Author: 
	Xi Song
	Junhan Zhu

CS ID: 
	Xi
 	junhan

///////////////////////////////////////////////
Description:

1.Null pointer.
	I just move the beginning of code segmentation to the first page. And I modified the according Makefile in the user directory.
	Onething to mention is that, the function copyuvm() is also modified that it should look up the PTE from the first pageszie but not addr 0.

2.Move stack to the top of user vm
	Fisrt, I should use allocuvm() to allocate a PGSZIE of memory in use vm for the stack, which starts at UESRTOP-PGSIZE and ends at USERTOP.
	In addition, I should use to new variables to track the top of the user heap and user stack. Of course, there is no need using the heap to track additionally, of which I could use proc->sz instead.
        Furthermore, the copyuvm() should also be modified so as to copy the stack page directories accordingly.
	This is just one page size stack.
3.Grow heap
	Actually, this is done by growproc(), which already exits.
3.grow stack
	When the kernel detects the trap that tells us there is a bad stack ps out of stack, the kernel grow stack by using the function growstack(), which is added by me. Every time, it grows one page backward in the user mem.
4.One page between heap and stack
	As I already use the variables sz_heap and sz_stack to track of the top of both heap and stack, is easy to leave one page size space between them. Just dectect the top when growproc() and growstack()
5.bounds
	When the system calls, the kernel checks the fetchint and the fetchstr so that the bad arg will not be passed to the kernel and return -1.

---------------------
trap.c
	if((tf->trapno == 14))
    {
        cprintf("-trap:growstack proc->sz_stack(%d)\n",proc->sz_stack);
        if(growstack()== 0) break; //success return,else failed;
    }
----------------------
exec.c
	sz = PGSIZE;//PGSIZW = 4K
	...
	 int sz_stack= 0;
  if((sz_stack = allocuvm(pgdir, USERTOP-PGSIZE, USERTOP))==0)
        panic("allocuvm sz_stack failed\n");
  sp = sz_stack;
	...
	proc->sz = sz_stack;//current stack bottom
  proc->sz_stack = sz_stack-PGSIZE;//current stack top
  proc->sz_heap = sz;

---------------
vm.c
	 i = USERTOP - PGSIZE;
    for(;i>=proc->sz_stack;i-=PGSIZE){
    if((pte = walkpgdir(pgdir, (void*)i, 0)) == 0)
      panic("copyuvm: pte should exist");
    if(!(*pte & PTE_P))
      panic("copyuvm: page not present");
    pa = PTE_ADDR(*pte);
    if((mem = kalloc()) == 0)
      goto bad;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(d, (void*)i, PGSIZE, PADDR(mem), PTE_W|PTE_U) < 0)
      goto bad;
    }
	...
--------------------------
proc.c
	np->sz_stack = proc->sz_stack;//sx:good
  	np->sz_heap = proc->sz_heap;//s
	...
	sz = proc->sz_heap;
  	cprintf("-growproc...sz_heap(%dK)sz_stack(%d)sz+n(%dK)\n",sz/1024,proc->sz_stack/1024,(sz+n)/1024);//
  	//sx:if heap reach top, fail
  	if((sz+n)>(proc->sz_stack-PGSIZE))
  	{
         cprintf("Error:Heap reaches the top\n");
         return -1;
  	}
	...
	growstack(void)
	{
 	 uint sz = proc->sz_stack-PGSIZE;//
 	 cprintf("-(%d)growstack...sz_stack(%dK)\n",proc->pid,proc->sz_stack/1024);
 	 if(sz < proc->sz_heap+PGSIZE)
 	       return -1;
 	 if((sz = allocuvm(proc->pgdir, sz, sz + PGSIZE)) == 0){
 	       return -1;
  	}
  	proc->sz_stack = sz -PGSIZE;
  	cprintf("-growstack end,sz_stack(%dK)\n-switchuvm\n",proc->sz_stack/1024);
  	switchuvm(proc);
  	return 0;
	}
----------------------------
syscall.c
	int
	checkbounds(struct proc *p,int addr)
	{
        if(p->pid !=1){
                if(     addr<0x1000
                        ||(addr>=proc->sz_heap&&addr<proc->sz_stack)
                        ||addr>=USERTOP)
                        return -1;
        }
        return 0;
	}




